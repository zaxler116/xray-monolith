# Групповые состояния (group_states/)

> **Заглушка** — заполнится на итерации 6, порцион 20.

`src/xrGame/ai/monsters/group_states/` (27 файлов, ~3100 стр.): `group_state_attack`
(+run), `eat` (+drag/eat), `home_point_attack`, `hear_danger_sound`, `panic` (+run),
`rest` (+idle), `squad_move_to_radius`, `custom`, `state_adapter`. Связь со
`ai_monster_squad`/`ai_monster_squad_manager` (корень `monsters/`).

## Взаимодействие

- FSM-фундамент — порц. 18; монстровые состояния — порц. 19.
- Squad-менеджмент — порц. 26 (`member_*`).
