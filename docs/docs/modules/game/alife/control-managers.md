# Контроллеры CControl_*

> **Заглушка** — заполнится на итерации 6, порцион 21.

`src/xrGame/ai/monsters/`: `CControl_Manager`/`control_manager_custom`, base-классы
`control_animation_base` (+accel/load/update), `control_movement_base` (+
`control_path_builder_base`), `control_direction_base`; специализации:
`control_animation`, `control_movement`, `control_path_builder`, `control_direction`,
`control_jump`, `control_melee_jump`, `control_rotation_jump`, `control_run_attack`,
`control_sequencer`, `control_threaten`, `control_critical_wound`.

## Взаимодействие

- Вызов из `CBaseMonster` — порц. 3; abilities (`can_jump`, `run_attack`, …) — порц. 2.
- Пейтфиндинг/пути — ит. 7.
