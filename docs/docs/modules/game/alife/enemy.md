# Враги: enemy_manager, agent_enemy_manager, monster_enemy_manager

> **Заглушка** — заполнится на итерации 6, порцион 23.

Корень `src/xrGame/`: `enemy_manager` (база), `agent_enemy_manager` (~780 стр. +
inline ~280), `member_enemy`; `ai/monsters/monsters/`: `CMonsterEnemyManager`/
`CMonsterEnemyMemory` (порц. 3). `MONSTER_COMMUNITY`-relations — порц. 27.

## Взаимодействие

- Память — порц. 22; combat-менеджер ALife — порц. 32.
- Сеть: `CSE_ALife*` — ит. 11/12.
