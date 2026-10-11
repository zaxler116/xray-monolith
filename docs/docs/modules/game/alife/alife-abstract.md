# Abstract-обёртки ALife

> **Заглушка** — заполнится на итерации 6, порцион 30.

Корень `src/xrGame/`: `alife_object`, `alife_dynamic_object`, `alife_creature_abstract`,
`alife_monster_abstract`, `alife_monster_base`, `alife_human_abstract`,
`alife_trader_abstract`, `alife_group_abstract`; `alife_monster_patrol_path_manager`,
`alife_monster_detail_path_manager`, `alife_monster_movement_manager` (упрощённое
движение оффлайн-монстров); `alife_human_brain_save`/`alife_human_object_handler`.

## Взаимодействие

- Симулятор — порц. 28; регистры — порц. 29.
- Пейтфиндинг (patrol/detail path) — ит. 7.
- Сеть: `CSE_ALife*` — ит. 11/12.
