# FSM-фундамент: CState, EMonsterState, CMonsterStateManager

> **Заглушка** — заполнится на итерации 6, порцион 18.

- `CState<_Object>` (`src/xrGame/ai/monsters/state.h`) — шаблон: `reinit`/`initialize`/
  `execute`/`finalize`/`check_start_conditions`/`select_state`/substates
  (`SubStates xr_map<u32, CSState*>`).
- `EMonsterState` (`state_defs.h`) — битовые подгруппы: Rest/Eat/Attack(+Camp/Psy/HomePoint)/
  Panic/Hitted/HearDangerousSound/HearInterestingSound(+HelpSound)/Controlled/Threaten/
  FindEnemy/Squad/Custom (BurerScanning, SmartTerrainTask, Vampire, …).
- `CMonsterStateManager<_Object>` (`monster_state_manager.h`) — `IStateManagerBase + CState`,
  `update()`, `force_script_state`, `execute_script_state`, `can_eat`.
- `CStateMove` — `object->path().prepare_builder()`.

## Взаимодействие

- Конкретные состояния — порц. 19; групповые — порц. 20.
- Использование из `CBaseMonster` — порц. 3; из `CAI_Stalker` — порц. 12.
