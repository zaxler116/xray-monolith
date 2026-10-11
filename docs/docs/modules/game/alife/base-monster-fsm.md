# CBaseMonster: FSM-цикл, память, контроллеры

> **Заглушка** — заполнится на итерации 6, порцион 3.

`update_fsm()` / `post_fsm_update()` / `Think()` / `ProcessScripts()`: как тик монстра
связывает FSM (`CMonsterStateManager`, порц. 18), память
(`CMonsterEnemyManager`/`CMonsterCorpseManager`/`CMonsterEventManager`/`CMonsterEnemyMemory`/
`SoundMemory`/`CorpseMemory`/`HitMemory`, порц. 22), `CMonsterCoverManager*`, `CMonsterHome*`,
`CAnomalyDetector` и контроллеры (`CControl_Manager*`, порц. 21).

## Взаимодействие

- FSM-фундамент — порц. 18; конкретные состояния — порц. 19–20.
- `steering_behaviour::manager` — порц. 25.
- Сеть: `CSE_ALife*` — ит. 11/12 (только `CSE_Abstract*` как аргумент `net_Spawn`).
