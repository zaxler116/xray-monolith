# CAI_Stalker: движение и покрытия

> **Заглушка** — заполнится на итерации 6, порцион 14.

`stalker_movement_manager_smart_cover` (+ `fov_range`, `loopholes`),
`stalker_movement_manager_base`/`obstacles`, `stalker_movement_params`,
`stalker_movement_restriction`, `stalker_velocity_collection/holder`,
`smart_cover_evaluators`, `cover_evaluators` (выборочно, inline ~199k стр.),
`danger_cover_location`, `CSightManager` (`sight_manager`).

## Взаимодействие

- Покрытия монстров — порц. 24; steering/steps — порц. 25.
- Пейтфиндинг — ит. 5 (переходная `objects.md`) / ит. 7.
