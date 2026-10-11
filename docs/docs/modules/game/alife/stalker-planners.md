# CAI_Stalker: planner'ы

> **Заглушка** — заполнится на итерации 6, порцион 13.

`CStalkerPlanner` (CActionPlannerScript) и под-planner'ы в корне `src/xrGame/`:
`stalker_combat_planner`/`stalker_combat_actions` (~1700 стр.), `stalker_danger_planner` +
danger-под-planner'ы (sound/grenade/in-direction/unknown), `stalker_search_planner`,
`stalker_death_planner`, `stalker_get_distance_planner`, `stalker_kill_wounded_planner`,
`stalker_low_cover_planner`, `stalker_anomaly_planner`; property-eval'ы
(`stalker_property_evaluators`, `object_property_evaluators`).

## Взаимодействие

- Класс-хозяин — порц. 12; `CObjectHandler`/`object_actions` — порц. 26.
- Lua-скрипты planner'ов — ит. 14.
