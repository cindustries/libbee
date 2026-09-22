# libbee — design spec

Status: draft for review. Owner: `libbee`. Consumers: `sunriser8`, `gnx`, `p5-beepack`.

## 1. Purpose

libbee is the portable C implementation of the BeePack format (skill `beepack-format`):
a CDB file whose values are MsgPack, an override directory of hashed `.MP` files, and a
layered lookup across both. It replaces `sunriser8/src/bee.c`, becomes the storage
layer of gnx, and later the engine of `p5-beepack` through XS.

Goals, in priority order:

1. **Bit-for-bit interoperable** with files the Perl `bee` tool writes and the SunRiser
   firmware reads, including existing `CONFIG/` directories on SD cards in the field.
2. **Runs on the weakest host**: STM32 with gcc 4.9, FatFS, no heap in the read path.
3. **One code base**, backends swapped by linking one file.
4. **Boring to bind**: plain functions, integer error codes, caller-owned memory, so an
   XS layer and a Lua binding are thin.

Non-goals: several values per key, in-place update of a `.bee`, thread safety inside
one context (callers serialize; separate contexts are independent), MsgPack float,
64-bit and ext types (rejected on read, never written — a format extension is its own
ticket), any logging.

## 2. Layout

```
bee.h              public API, the only header a consumer includes
bee.c              core: value cursor, typed reads, stack resolution, override read
bee_cdb.c/.h       CDB reader and writer primitives
bee_make.c         .bee writer (host side, may allocate)
bee_override.c     override set / unset / export / import
bee_posix.c/.h     POSIX backend: pread or mmap, temp+rename writes, unlink
bee_fatfs.c/.h     FatFS backend: FIL, f_read/f_lseek, f_open create-always, f_unlink
third_party/cmp/   vendored MsgPack codec, unchanged upstream
t/                 C test programs; t/fixtures/ with generating scripts
Makefile           make, make test, make test32, make check-fatfs
```

The consumer compiles `bee.c bee_cdb.c bee_override.c third_party/cmp/cmp.c` plus one
backend, and optionally `bee_make.c`. No install step is required; `make` also produces
`libbee.a` for convenience.

Language: C99. No VLAs, no `_Generic`, no anonymous unions, no `<stdio.h>` in the core
files (backends may use it). Warnings clean under `-Wall -Wextra -Werror` on gcc and
clang.

## 3. I/O abstraction

Everything the core reads goes through a positional reader; everything it writes goes
through a small file-system interface. Both are structs of function pointers the backend
fills in.

```c
typedef struct bee_io {
    void    *ctx;
    uint32_t size;                                   /* bytes in the source */
    int    (*read)(void *ctx, uint32_t off, void *buf, uint32_t len); /* all or error */
    void   (*close)(void *ctx);
} bee_io_t;

typedef struct bee_fs {
    void *ctx;
    int (*open)(void *ctx, const char *path, bee_io_t *out);            /* read */
    int (*write)(void *ctx, const char *path, const void *data, uint32_t len);
    int (*unlink)(void *ctx, const char *path);
    int (*list)(void *ctx, const char *dir,
                int (*cb)(void *arg, const char *name), void *arg);     /* export */
} bee_fs_t;
```

Positional read instead of read+seek keeps the reader stateless and lets the mmap
backend implement `read` as `memcpy`. `write` has create-always semantics; the POSIX
backend writes to `<path>.tmp` and renames, FatFS writes in place (as the firmware does
today). Paths are built by the core with `/` and a `%08X.MP` name; the backend may
rewrite separators.

## 4. Contexts

```c
typedef struct bee {           /* one open source of MsgPack values */
    bee_io_t   io;
    uint8_t    kind;           /* BEE_KIND_CDB or BEE_KIND_MP */
    uint32_t   pos;            /* cursor for cmp reads */
    cmp_ctx_t  cmp;
    cmp_object_t obj;          /* last object parsed */
    uint32_t   val_off, val_len;  /* current value's MsgPack bytes */
} bee_t;

typedef struct bee_layer {
    uint8_t     type;          /* BEE_LAYER_FILE or BEE_LAYER_DIR */
    bee_t      *file;          /* FILE: an opened .bee */
    const char *dir;           /* DIR: override directory path */
} bee_layer_t;

typedef struct bee_stack {
    bee_fs_t    *fs;           /* needed by DIR layers and writers, may be NULL */
    bee_layer_t  layers[BEE_MAX_LAYERS];   /* most specific first */
    uint8_t      n;
    bee_t        scratch;      /* holds an opened override .MP during a lookup */
    bee_t       *cur;          /* the context the current value lives in */
} bee_stack_t;
```

All structs are caller-allocated (static in firmware, on the stack or heap on hosts).
`BEE_MAX_LAYERS` defaults to 4 (firmware uses 3: CONFIG, THEME, FIRMWARE; gnx uses 3:
local, state, module) and is overridable with `-DBEE_MAX_LAYERS=`. `BEE_KEY_MAX`
defaults to 128 bytes, the firmware's limit, and bounds the key buffer used when
verifying an override file.

## 5. Lookup semantics

`bee_seek(stack, key, klen)` walks the layers in order and stops at the first that
holds the key. A DIR layer is checked by hashing the key (djb2 over the key bytes taken
as **signed** `char`, exactly like the firmware, so existing SD cards resolve), opening
`<dir>/<%08X>.MP`, reading the MsgPack `str` at the start and comparing **length and
bytes** with the requested key. A mismatch is `BEE_EKEY` for that layer and the walk
continues. A CDB layer is checked with the CDB hash lookup. On a hit `cur` points at the
context, the value's offset and length are recorded, and the first MsgPack object of the
value is parsed into `cur->obj`.

A present **nil** is a hit: it masks lower layers and reads as nil (firmware
semantics). `bee_has` is true for it. Perl's `nil_exists` default is the Perl side's
concern; the C side never invents "absent" from "nil".

Typed reads coerce like the firmware (`beepack-format`, Coercion): integers read as
bool by non-zero, bool reads as 0/1, nil reads as the zero value. Types outside the
portable subset make every typed read return `BEE_ETYPE`; nothing reinterprets bit
patterns.

## 6. Public API

Errors: `BEE_OK 0`, `BEE_ENOENT` (no layer has the key), `BEE_EIO`, `BEE_EFORMAT`
(corrupt CDB or MsgPack), `BEE_ETYPE` (value type not what was asked / not portable),
`BEE_ERANGE` (buffer too small, value out of range for the C type), `BEE_EINVAL`,
`BEE_EFULL` (stack full), `BEE_EKEY` (override file holds a different key).
`const char *bee_strerror(int)`.

Open and stack:

```c
int  bee_open(bee_t *b, const bee_io_t *io, uint8_t kind);
void bee_close(bee_t *b);
void bee_stack_init(bee_stack_t *s, bee_fs_t *fs);
int  bee_stack_push_file(bee_stack_t *s, bee_t *b);      /* push least specific first */
int  bee_stack_push_dir(bee_stack_t *s, const char *dir);
```

Lookup and current value (strict, error-returning — the binding layer's API):

```c
int      bee_seek(bee_stack_t *s, const void *key, uint32_t klen);
uint8_t  bee_type(bee_stack_t *s);                 /* BEE_T_NIL, BOOL, UINT, SINT, BYTES, ARRAY, MAP, OTHER */
int      bee_get_bool(bee_stack_t *s, bool *out);
int      bee_get_u32 (bee_stack_t *s, uint32_t *out);
int      bee_get_i32 (bee_stack_t *s, int32_t *out);
int      bee_get_bytes(bee_stack_t *s, void *buf, uint32_t max, uint32_t *len);
int      bee_get_count(bee_stack_t *s, uint32_t *count);   /* array or map size */
int      bee_next(bee_stack_t *s);                 /* parse next object of the current container */
int      bee_raw(bee_stack_t *s, uint32_t *off, uint32_t *len, bee_t **src); /* zero-copy blobs */
```

Convenience reads with firmware semantics (zero value on any error, one call):

```c
bool     bee_has(bee_stack_t *s, const void *key, uint32_t klen);
bool     bee_bool(bee_stack_t *s, const void *key, uint32_t klen);
uint32_t bee_u32(bee_stack_t *s, const void *key, uint32_t klen);
int32_t  bee_i32(bee_stack_t *s, const void *key, uint32_t klen);
uint32_t bee_bytes(bee_stack_t *s, const void *key, uint32_t klen, void *buf, uint32_t max);
uint32_t bee_array(bee_stack_t *s, const void *key, uint32_t klen);  /* count, then bee_next */
uint32_t bee_map(bee_stack_t *s, const void *key, uint32_t klen);
```

These are the names the firmware port renames onto; `bee_array`/`bee_map` followed by
`bee_next` + `bee_get_*` is the firmware's `bee_array` + `bee_next_in` +
`bee_unsigned_in` pattern with the globals moved into the stack.

Override writes (need `s->fs`):

```c
int bee_override_set (bee_stack_t *s, const char *dir, const void *key, uint32_t klen,
                      const void *msgpack, uint32_t mlen);
int bee_override_set_u32 (…, uint32_t v);   int bee_override_set_i32 (…, int32_t v);
int bee_override_set_bool(…, bool v);       int bee_override_set_nil (…);
int bee_override_set_bytes(…, const void *p, uint32_t len);   /* MsgPack bin */
int bee_override_set_str  (…, const char *p, uint32_t len);   /* MsgPack str */
int bee_override_unset(bee_stack_t *s, const char *dir, const void *key, uint32_t klen);
int bee_override_export(bee_stack_t *s, const char *dir, const char *backup_path);
int bee_override_import(bee_stack_t *s, const char *dir, const char *backup_path);
```

`export` produces `BACKUP.MP` (map header with count, then key/value pairs), `import`
splits one back into files. Both are ports of the firmware's backup and restore.

`.bee` writer (host side, `bee_make.c`, allocates):

```c
int bee_make_begin (bee_make_t *m, bee_fs_t *fs, const char *tmp_path);
int bee_make_add   (bee_make_t *m, const void *key, uint32_t klen, const void *msgpack, uint32_t mlen);
int bee_make_add_u32(…); … /* same typed helpers as override_set */
int bee_make_finish(bee_make_t *m);   /* sorts keys, writes CDB, closes; caller renames */
```

Entries are sorted by key bytes at finish, so the output is content-identical to what
Perl writes for the same data; byte-identity is asserted by a test, not promised by the
API. Typed helpers refuse values outside the portable subset with `BEE_ETYPE`.

Backends:

```c
int bee_posix_open(bee_io_t *io, const char *path);     /* mmap if possible, else pread */
void bee_posix_fs(bee_fs_t *fs);                         /* fills a POSIX fs */
int bee_fatfs_open(bee_io_t *io, FIL *fil);              /* caller-owned FIL */
void bee_fatfs_fs(bee_fs_t *fs);
```

## 7. CDB

Reader: standard cdb — 2048-byte header of 256 (pos, len) pairs, records
(klen, dlen, key, data), hash `h = 5381; h = ((h << 5) + h) ^ c`, table `h & 255`,
slot `(h >> 8) % len`, linear probing. Implemented in `bee_cdb.c` on the positional
reader with a small stack buffer; no allocation. Sanity checks (offsets inside
`io.size`, `klen + dlen` bounded) turn a corrupt file into `BEE_EFORMAT`, never a wild
read. Writer: entries buffered, tables built at finish, written through `bee_fs.write`
in one go on POSIX.

## 8. Testing

- `t/*.c`, one program per topic, each returning non-zero on failure; `make test` runs
  them all with a summary line per program.
- Fixtures under `t/fixtures/` are produced by `t/fixtures/generate.sh` with
  `perl -I ~/dev/p5-beepack/lib ~/dev/p5-beepack/bin/bee`; the script is checked in
  and re-run when p5-beepack changes. Fixtures: every portable type, a 2 MB blob, a
  three-layer stack with an override directory containing one masking nil, one
  override file with a wrong stored key, a truncated CDB.
- Writer tests: `bee_make` output is read back by Perl (`bee out.bee key`, run from the
  test through `system` when Perl and p5-beepack are present, skipped with a visible
  SKIP line otherwise).
- `make test32`: the same suite built with `-m32 -march=i686 -mno-sse`. Requires
  `gcc-multilib`; not installed on the dev box today, a ticket for gnx-os covers a musl
  cross toolchain that can also serve this.
- `make check-fatfs`: compiles `bee_fatfs.c` against `~/dev/sunriser8/inc/ff.h` with
  `-std=c99 -Wall -Werror`, no link. Proves the backend still fits the firmware's FatFS.

## 9. Implementation order

1. `bee.h` complete, reviewed against every `bee_*` call site in `sunriser8/src` and
   the gnx storage needs. Nothing else until the header is agreed.
2. cmp vendored; POSIX backend; CDB reader; `bee_seek` on a single FILE layer; fixtures
   for every type; typed reads.
3. Stack with DIR layers; override read with strict key check; masking-nil test.
4. Override set/unset; export/import.
5. `bee_make` and the Perl read-back test.
6. FatFS backend and `check-fatfs`; ticket on the sunriser8 board for the port.
7. `test32` once a 32-bit toolchain exists.
