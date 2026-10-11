# CBaseMonster: каркас и abilities

> **Заглушка** — заполнится на итерации 6, порцион 2.

`CBaseMonster` (`src/xrGame/ai/monsters/basemonster/base_monster.h`) — `CCustomMonster +
CStepManager`, ядро всех монстров: класс, поля (EnemyMan/CorpseMan/EventMan/MeleeChecker/
Morale/CoverMan/Home/AnomalyDetector), abilities (`ability_invisibility`, `can_drag`,
`psi_attack`, `can_jump`, `distant_feel`, `run_attack`, `rotation_jump`, `jump_over_physics`),
`smonsterSettings` (`m_base`/`m_current_settings`).

## Взаимодействие

- Родитель: `CCustomMonster` (порц. 9 ит. 5) — `Think()` вызывается отсюда.
- FSM-цикл: порцион 3; контроллеры: порцион 21; память: порцион 22.
- Наследники: 15 видов монстров (порционы 5–9, 16–17); `CAI_Rat` — **не** наследник (порц. 10).
