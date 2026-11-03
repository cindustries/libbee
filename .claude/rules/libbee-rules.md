# libbee House Rules

The shared house rules come from skilletor — bundle `getty-house@getty`: engineering
discipline, the delegation lock, the publish gate, karr coordination. They load as
`.claude/rules/.local.*.md`; if they are missing, run `skilletor sync` and publish nothing
until they load. This file adds what is true only here, and wins where it differs.

## What this repo is

Portable C implementation of the BeePack format: a CDB whose values are MsgPack, plus the
override directory and layered lookup the SunRiser firmware established. One C99 code base for
STM32 firmware (FatFS), 32-bit x86 without SSE, Android, iOS and the Perl `BeePack`
distribution (via XS). Read first: the firmware original (`~/dev/sunriser8/src/bee.c`) and the
Perl writer (`~/dev/p5-beepack`) define what "compatible" means; read them before changing a
read or write path.

## Agent team

| Task | Agent |
|---|---|
| Implement / refactor / debug `bee.c`, backends, Makefile, headers | `libbee-worker` (default) |
| Write or extend tests and fixtures under `t/` | `libbee-test-writer` |
| Read-only research, audit, gap analysis | `libbee-worker` ("read-only, change nothing") |

Behavior-relevant = every `.c`/`.h`, the Makefile, `t/`. Prose in README is not. The
orchestrator runs `make test` and edits README/CHANGELOG prose.

## Family

libbee is part of the CanLin family: a change a consumer needs arrives as a ticket tagged
`from:<repo>`; a change a consumer must make is a ticket on that repo's board (protocol: skill
`canlin-coordination` in `~/dev/canlin`).

## Release

`make`, `make test` are fine anytime. Tagging a release, pushing tags, or publishing an
Alien/CPAN distribution built from this repo needs the go-ahead.

## Deviations from the house rules

- **No release-manager.** There is none on this team, so the orchestrating main agent owns the
  commits; workers leave a commit-ready tree.

## Hazards specific to this repository

- **A test that only checks our writer against our reader** cannot fail when we drift from the
  format.
- **The host compiles and runs SSE code the targets do not have.** A green `make test` on the
  dev box says nothing about i686-no-SSE or STM32. Build with `-m32 -march=i686 -mno-sse` as
  well before calling anything done; "tests pass" is wrong if the no-SSE build was not run —
  say so.
- **gcc 4.9 is a consumer.** sunriser8 is pinned to `gcc-arm-none-eabi 4.9.3`; C11 features,
  `_Generic`, VLAs and newer builtins break that build silently here and loudly there. C99, no
  VLAs.

## Format and library rules — reference, don't restate

The on-disk format is skill `beepack-format`; the library's architecture, API names,
no-allocation and callback rules are skill `libbee-core` (force-loaded for `libbee-*` agents).
Do not duplicate that content here.
