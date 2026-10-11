# Память: memory_manager, visual/sound/hit, memory_space

> **Заглушка** — заполнится на итерации 6, порцион 22.

Корень `src/xrGame/`: `memory_manager` (база), `agent_memory_manager`,
`visual_memory_manager` (~990 стр.), `sound_memory_manager`, `hit_memory_manager`,
`memory_space` (пространство памяти: что видно/слышно/где видели). Связь с
`CMonsterEnemyMemory`/`SoundMemory`/`CorpseMemory`/`HitMemory` (порц. 3) и
`actor_memory` (игрок).

## Взаимодействие

- Враги — порц. 23; `CAI_ObjectLocation`/`ai_obstacle` — порц. 26.
- Lua-скрипты памяти — ит. 14.
