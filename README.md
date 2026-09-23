# libbee

Portable C implementation of the BeePack format: a CDB (constant database) whose values
are MsgPack, plus the override directory and layered lookup the SunRiser firmware
established. One C99 code base for STM32 firmware (FatFS), 32-bit x86 without SSE,
Android, iOS and the Perl `BeePack` distribution (via XS).

Format: `.claude/skills/beepack-format/SKILL.md`. Library rules:
`.claude/skills/libbee-core/SKILL.md`. Sibling projects: `p5-beepack` (Perl reference
writer and `bee` CLI), `sunriser8` (firmware consumer), `canlin` (runtime consumer).
