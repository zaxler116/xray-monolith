# Покрытия: cover_evaluators, smart_cover_evaluators, monster_cover_manager

> **Заглушка** — заполнится на итерации 6, порцион 24.

Корень `src/xrGame/`: `cover_evaluators` (inline ~199k строк — **только API и структура**),
`smart_cover_evaluators`; `ai/monsters/`: `CMonsterCoverManager*`, `corpse_cover`,
`danger_cover_location`. 6 cover-оценщиков `CAI_Stalker` — порц. 14.

## Взаимодействие

- Движение сталкера — порц. 14; `CBaseMonster` — порц. 3.
- Smart-объекты уровня — ит. 12 (`Level_*`).
