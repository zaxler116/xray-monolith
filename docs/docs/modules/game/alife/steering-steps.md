# Steering + CStepManager

> **Заглушка** — заполнится на итерации 6, порцион 25.

Корень `src/xrGame/`: `steering_behaviour` (manager, base, separation/alignment/cohesion,
target_provider), `step_manager` (+ `step_manager_defs`), `monster_velocity_space`.
Поведенческое управление движением и по-step анимация.

## Взаимодействие

- `CBaseMonster`/`CAI_Stalker` (оба используют `CStepManager`) — порц. 2/12.
- Анимация — ит. 3 (`IKinematics`); движение монстров — порц. 21.
