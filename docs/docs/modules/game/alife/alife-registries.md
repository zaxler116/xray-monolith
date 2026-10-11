# Регистры ALife

> **Заглушка** — заполнится на итерации 6, порцион 29.

Корень `src/xrGame/`: `alife_spawn_registry` (~420 стр.), `alife_schedule_registry`,
`alife_abstract_registry`, `alife_group_registry`, `alife_graph_registry`,
`alife_object_registry`, `alife_level_registry`, `alife_registry_container`,
`alife_registry_wrapper(s)`. Что регистрируется и как спавн/расписание уровня
запускаются через них.

## Взаимодействие

- Симулятор — порц. 28; abstract-обёртки — порц. 30.
- Уровни/spawn — ит. 12; Lua — ит. 14.
