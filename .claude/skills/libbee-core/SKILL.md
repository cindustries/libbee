---
name: libbee-core
description: "Use when implementing, refactoring, testing or reviewing libbee — the portable C99 BeePack library: the header-first API (bee_t, bee_stack_t, bee_io_t), the positional reader and the bee_fs callback seam, the CDB and override layers, the POSIX and FatFS backends, and the Perl-oracle fixture discipline."
user-invocable: false
allowed-tools: Read, Grep, Glob
model: sonnet
---

# libbee — Kern

libbee ist die portable C99-Umsetzung des BeePack-Formats: ein CDB, dessen Werte MsgPack
sind, ein Override-Verzeichnis und der geschichtete Lookup darüber. Sie ersetzt
`sunriser8/src/bee.c` und wird der Speicher-Layer von CanLin. **Es gibt noch keinen
Code** — `docs/superpowers/specs/2026-09-22-libbee-design.md` ist der Vertrag; das Format
steht in der Skill `beepack-format`.

## Konsumenten — jeder verbietet etwas anderes

| Konsument | Host | Folge für libbee |
|---|---|---|
| sunriser8 | STM32F1/F4, FatFS, gcc 4.9 | C99, keine VLA, kein Heap im Lesepfad, I/O per Callback |
| CanLin | i686 **ohne SSE**, musl, statisch; Android/iOS | kein SIMD, keine glibc-only Aufrufe, Pfade sind Backend-Sache |
| p5-beepack | Perl-XS (`CDB_File` → tinycdb) | schlanke C-API mit Fehlercodes, kein `exit`, kein `printf` |

## Aufbau

- **Kern** — `bee.c`, `bee_cdb.c`, `bee_override.c`: Wertenavigator, typisierte Reads,
  Stack-Auflösung, Override-Lesen. Kein `<stdio.h>`, kein `malloc`.
- **Backend** — `bee_posix.c`, `bee_fatfs.c`: nur I/O. Ein Konsument linkt genau eines
  oder bringt seinen eigenen Callback-Satz mit.
- **Writer** — `bee_make.c`: Host-Seite, darf allokieren, braucht die Firmware nie.
- **Codec** — `third_party/cmp/`: MsgPack, verbatim von upstream; ein lokaler Fix geht
  als Karte dorthin, nicht in die Datei.

## Die I/O-Naht

- **Positionales Lesen** (`read(ctx, off, buf, len)`) statt read+seek: der Leser bleibt
  zustandslos, das mmap-Backend implementiert `read` als `memcpy` und darf allokieren.
- **`write` hat create-always-Semantik.** POSIX schreibt `<pfad>.tmp` und benennt um;
  FatFS schreibt an Ort und Stelle, wie die Firmware heute.
- **Pfade baut der Kern** mit `/` und `%08X.MP`; das Backend darf die Trenner umschreiben
  (CanLin: app-privates Verzeichnis unter Android, später ein Browser-VFS).

`bee_io_t` = `{ctx, size, read, close}`, `bee_fs_t` = `{ctx, open, write, unlink, list}`.

## Vokabular

- **Layer** — ein Stack-Eintrag: `FILE` (eine `.bee`) oder `DIR` (ein Override-Verzeichnis).
- **Stack** — `bee_stack_t`, vom Aufrufer allokiert und initialisiert. `BEE_MAX_LAYERS`
  default 4 (Firmware 3: CONFIG, THEME, FIRMWARE), überschreibbar mit `-DBEE_MAX_LAYERS=`;
  `BEE_KEY_MAX` default 128.
- **Override-Datei** — `%08X.MP`: MsgPack-`str` des Keys, unmittelbar danach der Wert. Der
  Key steht in der Datei, damit der Leser prüft, ob es derselbe Key ist.
- **Maskierendes nil** — ein vorhandenes nil ist ein Treffer und verdeckt die tieferen
  Layer (Firmware-Semantik). Perl entscheidet das selbst (`nil_exists`, Default: nil
  existiert nicht).

## Der Weg einer Suche

`bee_seek(stack, key, klen)` läuft die Layer ab und stoppt beim ersten Treffer,
**spezifisch zuerst** (CONFIG → THEME → FIRMWARE, so die Firmware). Ein DIR-Layer hasht
den Key, öffnet `<dir>/<%08X>.MP`, liest das MsgPack-`str` und vergleicht **Länge und
Bytes**; eine Abweichung ist `BEE_EKEY` für diesen Layer, die Suche läuft weiter. Ein
FILE-Layer nutzt die CDB-Suche. Beim Treffer hält `cur` Kontext, Offset und Länge.

## API-Oberfläche

- **Strikt**: `bee_seek`, `bee_type`, `bee_get_bool` / `_u32` / `_i32` / `_bytes` /
  `_count`, `bee_next` (Container iterieren), `bee_raw` (Offset und Länge des Werts —
  große Blobs streamen ohne Kopie).
- **Bequem**: `bee_has`, `bee_bool`, `bee_u32`, `bee_i32`, `bee_bytes`, `bee_array`,
  `bee_map` — die Namen, die `sunriser8/src/bee.c` heute exportiert; der Port ist ein
  Rename.
- **Stack**: `bee_stack_init`, `bee_stack_push_file`, `bee_stack_push_dir`.
- **Fehler**: ein Enum — `BEE_OK`, `BEE_ENOENT`, `BEE_EIO`, `BEE_EFORMAT`, `BEE_ETYPE`,
  `BEE_ERANGE`, `BEE_EINVAL`, `BEE_EFULL`, `BEE_EKEY`; dazu `bee_strerror()`.
- **Overrides**: `bee_override_set*`, `_unset`, `_export`, `_import` (brauchen `s->fs`).
  **Writer**: `bee_make_begin` / `_add*` / `_finish` — baut eine `.bee` unter einem
  Temp-Pfad.

## Invarianten

- **Kein Heap im Kern.** Puffer kommen vom Aufrufer, Größen sind Parameter; ein Wert
  größer als der Puffer ist ein Fehler, keine Kürzung.
- **Keine Globals.** Jede geöffnete Datei ist ein `bee_t`, jeder Lookup ein `bee_stack_t`
  des Aufrufers; mehrere Stacks gleichzeitig sind normal (CanLin: einer pro Modul). Die
  Firmware hält `bee_obj` und `beepack_in` in Globals — das wird hier abgeschafft.
- **Fehler sind Rückgabecodes**, ein Enum, kein Logging im Bibliothekscode; der Aufrufer
  entscheidet, was ein fehlender Key bedeutet.
- **Nichts wird umgedeutet.** float, 64-Bit und ext liefern `BEE_ETYPE`. **C99**, keine
  VLA, kein `_Generic`; warnungsfrei unter `-Wall -Wextra -Werror` auf gcc und clang.

## Fallen

- **Der Host kompiliert SSE-Code, den die Ziele nicht haben**, und gcc 4.9 ist ein
  Konsument: ein grünes `make test` sagt nichts über i686-ohne-SSE oder STM32; C11,
  `_Generic` und VLAs brechen dort laut, hier still. Die Firmware nutzt selbst eine VLA
  (`hashn()`) — nicht übernehmen.
- **Der Override-Hash rechnet über signiertes `char`** (djb2 additiv, `h = h*33 + c`),
  die Firmware endet am ersten NUL-Byte, `Digest::DJB32` rechnet über unsigned — für Keys
  mit Bytes ≥ 0x80 verschiedene Dateinamen. libbee hasht die deklarierte `klen`.
- **Die alte Firmware verglich den Key mit `||`** (Länge *oder* Bytes) — ein Bug, den
  der Hash verdeckt; seit dem Fix gilt `&&`. Der Board-Eintrag „the firmware uses OR"
  ist überholt.
- **Die Firmware deutet float-Rohbits um** (ohne Log) und liest ein Zahlen-`str` („9")
  als Zahl — beides nicht in der Teilmenge, libbee liefert `BEE_ETYPE`.
- **Der Perl-Writer passt nicht ganz zur Teilmenge:** er erzeugt 64-Bit-Integer
  (`cf`/`d3`), die die Teilmenge ausschließt — ein strikter Leser lehnte legal erzeugte
  Werte ab — und emittiert nie `bin` (`utf8 => 1`), obwohl der Leser `bin` annehmen muss.
- **Nur `.bee` hat ein Perl-Orakel.** `p5-beepack` schreibt keine `.MP`-Dateien; für
  Override-Fixtures bleibt die Firmware die einzige Vorlage. Die `bee`-CLI liefert Bytes
  nur bei Skalaren, rohe Bytes über `BeePack::get_raw`.
- **Die Layer-Reihenfolge muss vor dem ersten Code feststehen:** die Spec sagt „most
  specific first", `bee_stack_push_file` „push least specific first" — beides stimmt nur,
  wenn `push` einfügt statt anhängt; falsch herum maskieren Overrides die tieferen Layer
  nicht.
- **CanLin braucht die Länge vor dem Lesen** (ein Wert über dem Limit wird gemeldet, ohne
  gelesen oder allokiert zu werden) und **Zero-Copy-Lebensdauer** (die Bytes müssen das
  Theme überleben) — beides verlangt `bee_raw` und einen offenen `bee_t`. Der Theme-Seam
  ist byte-orientiert; einen Renderer hat die API nicht.

## Kommandos

```bash
make                # baut die Objekte und libbee.a
make test           # startet die C-Testprogramme unter t/
make test CFLAGS="-m32 -march=i686 -mno-sse -std=c99 -Wall -Wextra -Werror"
make test32         # 32-Bit; braucht gcc-multilib (fehlt auf der Dev-Kiste)
make check-fatfs    # kompiliert bee_fatfs.c gegen ~/dev/sunriser8/inc/ff.h, ohne Link
```

Fixtures entstehen mit dem Perl-Werkzeug: `perl -I ~/dev/p5-beepack/lib
~/dev/p5-beepack/bin/bee <datei.bee> <key> <typ> <wert>` schreibt, `… bee <datei.bee>
<key>` liest zurück. Ein Writer ist fertig, wenn Perl die Datei gleich zurückliest.

## Datei-Karte

```
bee.h              öffentliche API — die einzige Headerdatei, die ein Konsument einbindet
bee.c              Kern: Wertenavigator, typisierte Reads, Stack-Auflösung, Override-Lesen
bee_cdb.c/.h       CDB-Leser und -Writer-Primitive
bee_override.c     Override set/unset/export/import
bee_make.c         .bee-Writer (Host-Seite, darf allokieren)
bee_posix.c/.h     POSIX-Backend: mmap oder pread, Temp+rename, unlink
bee_fatfs.c/.h     FatFS-Backend: FIL, f_read/f_lseek, create-always, f_unlink
third_party/cmp/   MsgPack-Codec, verbatim
t/                 C-Testprogramme, ein Programm pro Thema; Fixtures mit Erzeugerskript
```

## Grenzen

- Das **Format** gehört der Skill `beepack-format`; diese Skill wiederholt es nicht.
- **Perl-Writer** und `bee`-CLI gehören `p5-beepack`; die XS-Schicht dort wird später eine
  dünne Hülle **über libbee**.
- Der **Firmware-Port** (`sunriser8/src/bee.c` → libbee) ist ein sunriser8-Ticket.
- **Welche Layer** ein Stack hat, ist Konvention des Konsumenten, nicht des Formats;
  CanLin nutzt `local/` → `state.bee` → `module.bee` und bindet libbee als
  **git-Submodul** unter `third_party/` ein (Pin pro Release, nie darin editieren).
- **Lizenz**: Artistic-1.0/GPL-1.0-or-later; Canlins Runtime-Policy verbietet GPL —
  verlinken muss unter Artistic passieren, sonst ist MIT/BSD eine Entscheidung des Owners.
- **Nicht drin**: keine Thread-Sicherheit innerhalb eines Kontexts (Aufrufer
  serialisieren), kein In-place-Update einer `.bee` (CDB kann das nicht), kein Logging.

## Reihenfolge

Die Spec ist die Reihenfolge, das Board folgt ihr: 1 `bee.h` (gegen alle `bee_*`-Aufrufstellen
in `sunriser8/src` und die CanLin-Bedürfnisse geprüft), 2 cmp, POSIX-Backend, CDB-Leser,
typisierte Reads, 3 Stack mit DIR-Schichten und strikter Key-Prüfung, 4 Override
set/unset/export/import, 5 `bee_make` mit Perl-Read-back, 6 FatFS-Backend, 7 `test32`.
