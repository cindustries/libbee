# CLAUDE.md — libbee

Portable C99 implementation of the BeePack format (CDB container, MsgPack values,
override directory, layered lookup). Consumers: `sunriser8` (STM32, FatFS, gcc 4.9),
`canlin` (i686 without SSE, Android, iOS), `p5-beepack` (XS). No code yet — the first
tickets on the karr board define the shape.

Build and test: `make`, `make test`; always also
`make test CFLAGS="-m32 -march=i686 -mno-sse -std=c99 -Wall -Wextra -Werror"`.

## Delegation

Delegate behavior-relevant code to the right agent instead of touching it yourself —
principle and lane are in `.claude/rules/libbee-rules.md`.

| Task | Agent |
|---|---|
| Implement / refactor / debug `bee.c`, backends, headers, Makefile | `libbee-worker` (default) |
| Write or extend tests and fixtures under `t/` | `libbee-test-writer` |

The agents carry their skills via `briefing.skills` (see `.claude/agents/`); the main
agent delegates rather than loading them. Skill sources live under `.claude/skills/` —
`beepack-format` (the on-disk format, owned here, hardlinked into `canlin`) and
`libbee-core` (library architecture and API). Work is tracked on the local `karr`
board; cross-repo protocol: skill `canlin-coordination` in `~/dev/canlin`.
