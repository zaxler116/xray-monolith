# CAI_Stalker: класс, feel, события

> Модуль: `src/xrGame/ai/stalker/`
> Файлы: `ai_stalker.h` (918 стр.), `ai_stalker.cpp` (1794), `ai_stalker_inline.h` (594), `ai_stalker_feel.cpp` (115), `ai_stalker_events.cpp` (187), `ai_stalker_misc.cpp` (225), `ai_stalker_cover.cpp` (413), `ai_stalker_fire.cpp` (1442), `ai_stalker_script*.cpp` (348+193), `ai_stalker_debug.cpp` (1831)
> Порцион 12 (итерация 6).

`CAI_Stalker` — основной AI-сталкер. В отличие от монстров (`CBaseMonster`) ведёт себя как **человеческий NPC**: полный инвентарь (`CInventoryOwner`), оружие/патроны, торговля, planner'ы (`CActionPlannerScript`), покрытия, снайпинг, крит. раны.

## Чего не делает / границы

- **Не `CBaseMonster`** — нет `CStateManager`/abilities/`MonsterSpace`-FSM; «мозг» = `CStalkerPlanner` (planner'ы) — порц. 13.
- **Планы/анимации/стрельба**: planner'ы — порц. 13, движение/покрытия (`stalker_movement_manager_smart_cover`, 6 cover-оценщиков, `find_best_cover`) — порц. 14, анимация/стрельба/торговля (`CStalkerAnimationManager`, `ai_stalker_fire.cpp`) — порц. 15.
- `Feel::*`/`memory_manager`/`agent_manager` — ит. 5 и порц. 18–24; `CSE_ALifeHumanStalker` — ит. 11/12.

## 1. Иерархия и состав

`ai_stalker.h:89` `CAI_Stalker : CCustomMonster, CObjectHandler, CAI_PhraseDialogManager, CStepManager`.

`_construct` (`ai_stalker.cpp:1403`) создаёт 5 суб-менеджеров:
- `CCharacterPhysicsSupport* m_pPhysics_support` (etStalker);
- `CStalkerAnimationManager* m_animation_manager` — `animation()`;
- `CStalkerPlanner* m_brain` — `brain()` (мозг);
- `CSightManager* m_sight_manager` — `sight()`;
- `CWeaponShotEffector* m_weapon_shot_effector`.

Движение: `create_movement_manager()` → `stalker_movement_manager_smart_cover` (`movement()`); память: `create_memory_manager()` → `CMemoryManager` + `CStalkerSoundDataVisitor`.

Ключевые поля: 8 dispersion (`m_disp_walk_stand`…`m_disp_stand_crouch_zoom`); 6 cover-оценщиков (`m_ce_close/far/best/angle/safe/ambush`); ~60 fire-queue-параметров (pstl/shtg/snp/mchg/auto × far/medium/close × size/interval); best-item/ammunition (`m_best_item_to_kill`, `m_best_ammo`, …); `m_actor_relation_flags` (флаги поведения актёра); крит. раны (`m_critical_wound_weights`, `SBoneProtections* m_boneHitProtection`); пер-NPC aim (`m_aim_min_speed/min_angle/max_angle/predict_time`, отриц. = unset → глоб. cvar) и fire-queue-масштабы (`m_fire_queue_size_k/interval_k`).

## 2. Тик — `shedule_Update` + `Think`

`shedule_Update` (`ai_stalker.cpp:1129`), по порядку (только local, `g_Alive()`):
1. `update_object_handler()` если planner не инициализирован;
2. обрезка очереди `NET`;
3. `animation().play_delayed_callbacks()`; Lua `_G.CNPCBeforeLookAtActor` (LookAtActor);
4. `agent_manager().update()`;
5. `Exec_Visibility()` (зрение);
6. `process_enemies()` + `memory().update(dt)`;
7. `inherited::inherited::shedule_Update(DT)` (двойной `inherited` = `CCustomMonster`);
8. **`Think()`** (или `ProcessScripts()` при `GetScriptControl()`);
9. `feel_touch_update` + push `net_update` в `NET` (позиция/health/ yaw головы);
10. `UpdateInventoryOwner(DT)`; `m_pPhysics_support->in_shedule_Update(DT)`.

**`Think()`** (`:1292`) — весь «мозг» = 2 вызова:
```
brain().update(update_delta);      // planner'ы → цели/действия
if (!g_Alive()) return;
movement().update(update_delta);   // исполнение (движение + покрытия + оружие)
```
Это всё: нет switch-FSM — решения принимают planner'ы (порц. 13), исполнение — movement (порц. 14).

### 2.1 `UpdateCL` (client)
`UpdateCL` (`:1024`): `update_object_handler()` (или в `Device.seqParallel` при `mtObjectHandler`); звук «бежит в опасности»; `inherited::UpdateCL`; `m_pPhysics_support->in_UpdateCL`; `sight().update()` (с fallback `setup(eSightTypeCurrentDirection)` при исключ.) + `Exec_Look(fdelta)`; `CStepManager::update(false)`; `weapon_shot_effector().Update()`.

`update_object_handler` (`:966`) — оборачивает `CObjectHandler::update()` в try/catch: при любом исключении → `set_goal(eObjectActionIdle)` + `update()` (не роняет игру). Вызывается из `Device.seqParallel` и удаляется в `net_Destroy` (`:836`).

## 3. Feel

`ai_stalker_feel.cpp`. `feel_vision_isRelevant` — любой живой `CEntityAlive`/предмет. `feel_touch_new` (`ai_stalker_events.cpp:145`): игнорирует зоны при выключенном `g_ai_die_in_anomaly`/`m_enable_anomalies_pathfinding`; берёт предмет (`useful_for_NPC` && `can_take` && !wounded) → Lua `_G.CAI_Stalker__OnBeforeOwnershipTake` (может отказать) → `generate_take_event` (`GE_OWNERSHIP_TAKE`); иначе в `m_ignored_touched_objects`. `feel_touch_contact`/`on_contact` — аналогичные гейты по аномалиям.

## 4. События

`OnEvent` (`ai_stalker_events.cpp:26`):
- `GE_TRADE_BUY`/`GE_OWNERSHIP_TAKE` → `inventory().Take`; при отказе `GE_OWNERSHIP_REJECT`; `on_after_take`.
- `GE_TRADE_SELL`/`GE_OWNERSHIP_REJECT` → `on_ownership_reject` (`:97`): пересчёт костей, `inventory().DropItem`, `feel_touch_deny(O, 2000)`.

`useful`/`evaluate` (`ai_stalker_misc.cpp:41`): предметы — взрывные регистрируются в `agent_manager().location()/explosive()` как опасность; `CBolt` не берётся. `useful(CEnemyManager*)` — через `agent_manager().enemy().useful_enemy`.

`react_on_grenades` (`:109`) / `react_on_member_death` (`:156`) — звуковые реакции (грейнад-аларм, колокол/раненый) + Lua `_G.CAI_Stalker__OnMemberDeathReaction`. `process_enemies` (`:184`) — если врага нет, берёт врага видимого товарища.

## 5. Снап и сеть

`net_Spawn` (`ai_stalker.cpp:673`): `CSE_ALifeHumanStalker`; деньги; `animation().reload()`; yaw головы/тела из `o_torso`; game-вертекс (`m_tGraphID`) и `set_game_dest_vertex` (`m_tNextGraphID`); иммунити из model ini (`immunities`/`bone_protection`); **ранг** → `m_fRankImmunity/Visibility/Disperison` (интерполяция novice↔experienced); `sight().setup(...)`; bone-callback `bip01_head` → `BoneCallback`→`LookAtActor`.

`net_Export/Import` (`:870`/`:919`): health, `NET.back()` (поз/yaw головы), team/squad/group, game-вертекс (дублируется дважды — совместимость), `m_sStartDialog`. `save/load` (`:1459`): `inherited` + `CInventoryOwner` + **`brain().save/load`** (состояние planner'ов сериализуется).

## 6. Смерть

`Die` (`:603`): `movement().on_death()`, `notify_on_wounded_or_killed(who)`, `SelectAnimation`, звук смерти (`eStalkerSoundDie` / `eStalkerSoundDieInAnomaly` если `is_special_killer`); `m_hammer_is_clutched` (случайно сжимает молот); `inherited::Die`; `inventory().SetSlotsUseful(false)`; **уничтожает патроны** активного оружия (`GE_DESTROY`).

## 7. LookAtActor (голова на игрока)

`LookAtActor` (`:1671`, bone-callback): не для мёртвых/раненых; если далеко (`>NPCsLookAtActorMinDistance=3.5`), не видит актёра, cvar `NPCsLookAtActor=0` или Lua вернул false → `LookAtActorSoftReset`. Иначе строит матрицу на голову актёра, `getHPB`, clamp'ы, `AdjustHeadOrientation` (инерция через `angle_inertion`), `dTimeFSeen/dTimeNfSeen` — гасит ориентацию через 1с после ухода из зоны.

## 8. Замечания к плану

1. **Два «мозга»**: `CCustomMonster` даёт базовый `Think`, но у сталкера он переопределён целиком → `brain().update` + `movement().update`. Planner'ы — единственный источник решений.
2. `inherited::inherited::shedule_Update` — намеренный двойной skip (`CCustomMonster` → `CEntityAlive`), чтобы пропустить часть цепочки.
3. `aim_*`/`fire_queue_*` — отрицательное значение = «unset» → getter возвращает глоб. cvar/1.0 (vanilla-поведение). Пер-NPC тюнинг не ломает глобальные cvar-потребители.
4. `net_Export` дублирует game-вертекс и yaw (совместимость формата); `save/load` сериализует `brain()`.
5. `AlwaysTheCrow()` — legacy-название (взаимодействие с физикой), не про ворона.
6. `g_ai_unlimited_ammo` (default 1) гейтит бесконечные патроны; `g_ai_die_in_anomaly` — гейт аномалий.

## Связи

```mermaid
graph TD
  S[CAI_Stalker] --> CM[CCustomMonster]
  S --> OH[CObjectHandler]
  S --> SM[CStepManager]
  S --> AN[CStalkerAnimationManager]
  S --> BR[CStalkerPlanner "brain()"]
  S --> SI[CSightManager "sight()"]
  S --> MV[stalker_movement_manager_smart_cover "movement()"]
  S --> PH[CCharacterPhysicsSupport]
  BR --> P13["порц. 13 planner'ы"]
  MV --> P14["порц. 14 движение/покрытия"]
  AN --> P15["порц. 15 анимация/стрельба"]
  S --> EV["OnEvent / feel / react_*"]
  S --> LK["LookAtActor (bone-callback)"]
```

- Мост: `CCustomMonster` (порц. 9 ит. 5).
- Planner'ы — порц. 13; движение/покрытия — порц. 14; анимация/стрельба/торговля — порц. 15.
- `Feel::*` (порц. 27 ит. 5); `CSE_ALifeHumanStalker` — ит. 11/12.

## Краткое резюме

`CAI_Stalker` — человек-NPC на `CCustomMonster + CObjectHandler + CStepManager`: 5 суб-менеджеров (`animation`/`brain`/`sight`/`movement`/physics), полный инвентарь и торговля, ранг-зависимые иммунити/видимость/разброс, ~60 fire-queue-параметров и 6 cover-оценщиков. Тик: `shedule_Update` (зрение, память, agent, feel) → `Think` = **`brain().update` + `movement().update`** (решения planner'ов + исполнение). События — `OnEvent` (trade/ownership) и feel с гейтами по аномалиям; `LookAtActor` вращает голову на игрока через bone-callback; `save/load` сериализует `brain()`. Planner'ы/движение/стрельба раскрыты в порц. 13–15.

Порцион 12 (итерация 6). Далее: порц. 13 — `stalker-planners.md` — `CAI_Stalker`: planner'ы (combat/danger/search/death) ([stalker-planners.md](stalker-planners.md)).
