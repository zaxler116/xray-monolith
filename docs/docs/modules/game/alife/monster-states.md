# Состояния монстров (states/)

> **Заглушка** — заполнится на итерации 6, порцион 19.

`src/xrGame/ai/monsters/states/` (101 файл, ~6700 стр.) — ~20 классов `CStateMonster*`:
attack (+camp/stealout/melee/on_run/run/run_attack), controlled (+attack/follow), eat
(+drag/eat), find_enemy (+angry/look/run/walk), hear_danger/int_sound, help_sound,
hitted (+hide/moveout), home_point_*, panic (+run), rest (+fun/sleep/idle/walk_graph),
smart_terrain_task (+graph_walk), squad_rest (+follow), steal, custom_action, look_point,
look_unprotected_area, move_around_point, move_to_point, move_to_restrictor, hide_from_point,
hit_object, test.

## Взаимодействие

- FSM-фундамент — порц. 18; вызов из `CBaseMonster` — порц. 3.
- `state_move_to_restrictor` ↔ `CSpaceRestrictor` (ит. 5 порц. 24).
- Скрипт-состояния (`force_script_state`) — ит. 14.
