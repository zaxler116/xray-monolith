# CAI_Stalker: planner'ы (мозг)

> Модуль: корень `src/xrGame/` (planner'ы, не `ai/stalker/`)
> Файлы: `stalker_planner.{h,cpp}` (51+19/605), `stalker_combat_planner.{h,cpp}` (49/605), `stalker_combat_action_base.{h,cpp}` (41/379), `stalker_combat_actions.{h,cpp}` (337/1668), `stalker_danger_planner.{h,cpp}` (32/111), `stalker_danger_unknown_planner.cpp` (89), `stalker_danger_in_direction_planner.cpp` (104), `stalker_danger_grenade_planner.cpp` (102), `stalker_danger_by_sound_planner.cpp` (60), `stalker_search_planner.{h,cpp}` (33/90), `stalker_death_planner.{h,cpp}` (26/60), `stalker_anomaly_planner.{h,cpp}` (27/68), `stalker_kill_wounded_planner.cpp` (137), `stalker_get_distance_planner.cpp` (79), `stalker_low_cover_planner.cpp` (104), `stalker_property_evaluators.{h,cpp}` (353/598), `stalker_danger_property_evaluators.{h,cpp}` (149/278), `stalker_decision_space.h` (228), `stalker_base_action.{h,cpp}` (36/40); actions: `stalker_combat/search/alife/death/anomaly/kill_wounded/get_distance/low_cover/danger_*_actions.{h,cpp}`
> Порцион 13 (итерация 6).

«Мозг» `CAI_Stalker`: `CStalkerPlanner` (`brain()`), дерево под-planner'ов, property-eval'ы и actions. Решения принимает **action-planner** (граф состояний: property'ы + eval'аторы + operators/actions), а не switch-FSM.

## Чего не делает / границы

- **Исполнение** — `stalker_movement_manager_smart_cover` (движение, покрытия, `find_best_cover`) — порц. 14; анимация/стрельба/торговля — порц. 15; класс-хозяин — порц. 12.
- `CObjectHandler`/`object_actions` (`set_goal`) — порц. 26; `memory_manager`/`agent_manager` — порц. 22–23; Lua-скрипты planner'ов — ит. 14.

## 1. Архитектура: `CActionPlannerScript`

`CStalkerPlanner : CActionPlannerScript<CAI_Stalker>` (`stalker_planner.h:24`). Это **граф решаемых состояний**, не FSM:

- **Property** (`EWorldProperties`, `stalker_decision_space.h:13`) — булевы факты мира (~85 штук: `eWorldPropertyEnemy`, `…InCover`, `…LookedOut`, `…PositionHolded`, `…Danger`, `…Anomaly`, …).
- **Evaluator** — вычисляет value property'а из памяти/позиции/инвентаря (`CStalkerPropertyEvaluator*`).
- **Action** (`CStalkerActionBase : CActionScriptBase<CAI_Stalker>`) — узел: `add_condition(action, prop, val)` (когда выбрать) + `add_effect(action, prop, val)` (что изменит) + `add_operator(op, action)` (в граф).
- **Operator** (`EWorldOperators`, `stalker_decision_space.h:104`) — рёбра/узлы графа (~90).

Планировщик на каждом `update()` строит путь от текущего состояния к «решённой» задаче и активирует `initialize → execute* → finalize` выбранного action'а. Action'ы **не принимают решения** — они исполняют шаг графа; выбор делает planner по property/effect'ам.

## 2. `CStalkerPlanner` (root)

`stalker_planner.cpp:35` `setup(object, storage)` → `inherited::setup` → `clear()` → `add_evaluators()` → `add_actions()` → `CWorldState target`. Поле `m_affect_cover` (bool, getter/setter `affect_cover()`) — включает/выключает влияние cover'а на решение (действия combat/low-cover сами выставляют его: `object().brain().affect_cover(...)`).

Root регистрирует под-planner'ы как operators (глобальные): `eWorldOperatorDeathPlanner`, `…ALifePlanner`, `…CombatPlanner`, `…AnomalyPlanner`, `…DangerPlanner` (`stalker_decision_space.h:211`). Приоритет: death > anomaly > danger > combat > alife. `update(u32 dt)` тикает активный под-planner; `save/load` сериализует состояние (сериализуется через `CAI_Stalker::save`, порц. 12).

## 3. Property-eval'ы (общие)

`stalker_property_evaluators.cpp`. Все `: CStalkerPropertyEvaluator` (wrapper `CAI_Stalker` + `CPropertyEvaluator`). Ключевые:

| Evaluator | Вычисляет |
|-----------|-----------|
| `ALife` | `!!ai().get_alife()` — есть ли ALife-обёртка |
| `Alive` | `g_Alive()` |
| `Enemies` | враг выбран **или** в инерции (`last_enemy_time + time_to_wait`); `m_dont_wait` отключает инерцию |
| `SeeEnemy` / `EnemySeeMe` | вижу врага / враг видит меня (через `visual_memory`) |
| `ItemToKill` / `ItemCanKill` / `FoundItemToKill` / `FoundAmmo` | предметы для убийства/патроны в памяти |
| `ReadyToKill` | `ready_to_kill()` + проверка патронов (`min_ammo_count`, magazine) |
| `ReadyToKillSmartCover` | + если в cover'е и `!can_fire()` → true |
| `Anomaly` / `InsideAnomaly` | аномалия рядом / внутри (с `m_anomaly_detect_suppress_until` и `dwfChooseAction` гейтом по panic) |
| `Panic` | `dwfChooseAction(2000, panic_threshold, …)` |
| `SmartTerrainTask` | `alife->brain().select_task()` + `m_smart_terrain_id != 0xffff` |
| `EnemyReached` | расстояние до раненого врага ≤ 3 м и я — `wounded_processor` |
| `PlayerOnThePath` | актёр-враг на пути, виден |
| `EnemyCriticallyWounded` | враг-сталкер в крит. ране |
| `ShouldThrowGrenade` | граната в слоте, нет видимости врага, дальность > 10 м, `can_throw_grenade`, + Lua-хук `_G.CAI_Stalker__ShouldThrow` |
| `TooFarToKillEnemy` | `too_far_to_kill_enemy(position)` |
| `LowCover` | **жёстко `false`** (legacy-ветка отключена) |

`CPropertyEvaluatorConst`/`CPropertyEvaluatorMember` — примитивы: const-value и «прочитай property из storage» (с optional-флагом).

## 4. Combat planner + actions

`CStalkerCombatPlanner : CActionPlannerActionScript<CAI_Stalker>` (`stalker_combat_planner.h:24`): поля `m_last_level_time`, `m_last_enemy_id`, `m_last_wounded`, константа `POST_COMBAT_WAIT_INTERVAL=3000`, `on_best_cover_changed(old,new)`, `allow_action_switch`. Реализует основной боевой цикл: cover → look out → hold position → detour → search, plus kill/aim/ready.

Базовый action: `CStalkerActionBase` (`initialize/execute/finalize`, `object()`); боевой базовый: `CStalkerActionCombatBase` (`stalker_combat_action_base.cpp`) — `fire()` (угол цели < `g_ai_aim_fire_angle=PI/8`, иначе `aim_ready`), `aim_ready`/`aim_ready_force_full`, **`select_queue_params(distance, …)`** — выбор fire-queue (min/max size, interval) по типу оружия (pistol/shotgun/sniper/machinegun/auto) и дальности (far/medium/close) из ~60 параметров NPC (порц. 12), с пер-NPC `fire_queue_size_k/interval_k`. Звуки: `play_panic/attack/start_search/enemy_lost_sound`. `setup_cover(cover)` — smart-cover id или level-vertex.

Ключевые actions (`stalker_combat_actions.cpp`):

| Action | Роль |
|--------|------|
| `GetItemToKill` / `MakeItemKilling` | идти к найденному предмету/патронам |
| `RetreatFromEnemy` | отступить (run), far-cover, panic-звук; `weight()=100` |
| `GetReadyToKill` | подойти, `aim_ready`, выставлять `affect_cover`, блокировать смену врага |
| `KillEnemy` | stand, `fire()` если виден, `aim_ready`+sight-на-позицию если нет |
| `TakeCover` | идти к best cover (fallback на позицию товарища), `affect_cover`, fire/aim; Lua-хук `_G.CAI_Stalker__OnTakeCoverDestination` |
| `LookOut` | заглянуть из cover'а, случайный crouch (`g_ai_crouch_look_out_delta`), `current_cover()` рейкаст-обход |
| `HoldPosition` | держаться, инерция `g_ai_hold_position_inertia_*`, fire при `cover_detouring` |
| `DetourEnemy` | обойти, angle-cover, `CDangerCoverLocation` |
| `SearchEnemy` | искать врага по памяти |
| `PostCombatWait` | пауза после боя (3 с) |
| `HideFromGrenade` | укрыться от гранаты |
| `SuddenAttack` | внезапная атака: run/walk/crouch по дистанции (15/8/6/4 м) |
| `KillEnemyIfPlayerOnThePath` | убить если актёр на пути |
| `CriticalHit` | реакция на крит. хит (stand, idle, звук) |
| `ThrowGrenade` | бросок: цель, `set_goal(eObjectActionFire1, grenade)`, проверка угла головы |
| `SmartCover` | smart-cover поведение, `check_can_kill_enemy`, `combat_behaviour` |

## 5. Danger planner + 4 под-planner'а

`CStalkerDangerPlanner` (`stalker_danger_planner.cpp`): `finalize` — если враг выбран → пишет `memory().danger().time_line(now)`. Распределение по типу опасности (`CDangerObject::type()`):

- **`DangerUnknown`** (ricochet/смерть/свежий труп — `CStalkerPropertyEvaluatorDangerUnknown`): take cover → look around (crouch, 15 с) → search. Cover-выбор через `CStalkerPropertyEvaluatorDangerUnknownCoverActual` (двухпроходной `best_cover` 10 м → 30 м, `m_cover_selection_position`).
- **`DangerInDirection`** (attack sound/attacked/enemy sound): take cover → look out → hold position → detour → search.
- **`DangerGrenade`** (`eDangerTypeGrenade`): take cover → wait for explosion (`GrenadeExploded` = `!dependent_object()`) → take cover again → look around → search.
- **`DangerBySound`** — **заглушка**: единственный action `TakeCover` с `eWorldPropertyDangerUnknown=Const(false)`; все 5 actions (`ListenTo/Check/TakeCover/LookOut/LookAround`) — тривиальные stand/idle.

Общий eval' `CStalkerPropertyEvaluatorDangers` = `!!memory().danger().selected()`.

## 6. Search planner

`CStalkerSearchPlanner` (`stalker_search_planner.cpp`): `ENEMY_INERTIA_TIME_SEARCH=15000`. `initialize` — `cover(0)`, сброс `EnemyLocationReached`/`AmbushLocationReached`. 3 eval'а: `PureEnemy=Const(true)`, `EnemyLocationReached`/`AmbushLocationReached` (member). Actions: `ReachEnemyLocation` → `ReachAmbushLocation` (ambush-cover `m_ce_ambush`) → `HoldAmbushLocation` (crouch, `CoverLookOver`, инерция 15 с; если враг не виден > 60 с → `memory().enable(enemy, false)`).

## 7. Прочие planner'ы

- **`Death`** (`stalker_death_planner.cpp`): `PuzzleSolved=Const(false)`, `Dead` (member). Actions: `Dead` (умираю) → `dead` (лежу, `PuzzleSolved=true`). Action `CStalkerActionDead` (`stalker_death_actions.cpp`): при смерти `fire()` (выбросить/упасть оружие, `kWPN_FIRE`, ruck-инвентарь), `enable_movement(false)`, дроп предметов (`SetDropManual` для active).
- **`Anomaly`** (`stalker_anomaly_planner.cpp`): eval'ы `InsideAnomaly`/`Anomaly` (с suppress-гейтом). Actions: `GetOutOfAnomaly` (path-из-зоны, `add_restrictions` на `m_dynamic_in_restrictions`) → `DetectAnomaly` (бросок болта `BOLT_SLOT`, окно `m_anomaly_detect_start_time + 5000+rand(3000)`, cooldown 60 с). `update()` синхронизирует `eWorldPropertyAnomaly` между planner- и action-storage.
- **`KillWounded`** (`stalker_kill_wounded_planner.cpp`): догонеть/прицелиться/подготовиться/закончить раненого. `should_process` — я ли `wounded_processor`. `CStalkerActionKillWounded.execute` — виртуальный `GE_HIT` (power=1, `eHitTypeWound`), если враг невидим. `finalize` — если враг ещё выбран → `KilledWounded=false` + `eMentalStateDanger`.
- **`GetDistance`** (`stalker_get_distance_planner.cpp`): `InCover` (member) + `TooFarToKillEnemy`. Actions: `RunToCover` (close-cover) → `WaitInCover` (stand, crouch, инерция 1–3 с). `setup` вызывает `best_cover_invalidate()`.
- **`LowCover`** (`stalker_low_cover_planner.cpp`): `update` — `best_cover(position врага)`; `initialize` — stand, `eMentalStateDanger`, `InCover=true`. Eval'ы `LowCover=Const(true)`, `ReadyToKill`, `SeeEnemy`. Actions: `GetReadyToKillLowCover` (crouch, `aim_ready_force_full`) → `KillEnemyLowCover` (stand, fire) / `HoldPositionLowCover` (stand, aim, `NeedBackup` при detour'е).

## 8. Decision space (property'ы и operators)

`stalker_death_space.h` → нет, `stalker_decision_space.h`:
- `EWorldProperties` (`:13`) — ~85 property'ов: alive/dead, alife, smart-terrain, items/enemy/danger, kill-wounded (reached/aimed/prepared/paused/killed), too-far, enemy/ambush-location, danger (unknown/in-direction/grenade/by-sound), cover (actual/reached/looked-around), anomaly (inside/undetected), low-cover, smart-cover (~15), script.
- `EWorldOperators` (`:104`) — ~90 operators: death (2), alife (3), tasks (5), anomaly (2), combat (~25: aim/ready/kill/take-cover/look-out/hold/detour/search/hide/sudden/wounded/throw/smart-cover), danger (12: 4 planner'а + действия), global (5: Death/ALife/Combat/Anomaly/Danger planner'ы), script.
- `ESightActionType` (`:222`) — `WatchItem`/`WatchEnemy`.

## 9. Замечания к плану

1. **Planner ≠ FSM.** Это action-planner (граф): property'ы = факты, eval'аторы = их вычисление, actions = узлы с conditions/effects. Решения — по графу, не по `switch`.
2. **Action'ы исполняют, не решают.** `CStalkerActionBase::execute` тикается пока planner держит узел; `initialize`/`finalize` — вход/выход. `set_inertia_time` — задержка переключения.
3. **`affect_cover`** — мост action ↔ root-planner: combat/low-cover action'ы сами включают/выключают влияние cover'а на общее решение.
4. **`LowCover`-ветка** — частично legacy: `CStalkerPropertyEvaluatorLowCover::evaluate` жёстко `return false` (реальный low-cover работает через `LowCoverPlanner` + `m_ce_*`).
5. **`DangerBySound`** — заглушка: все 5 actions тривиальны, planner выбирает только `TakeCover` с fake-условием.
6. **`KillWounded.execute`** шлёт виртуальный `GE_HIT` (`power=1`, `eHitTypeWound`), если враг невидим — обход «враг в анимации внутри объекта».
7. **`select_queue_params`** — единственный источник fire-queue-параметров (порц. 12): тип оружия × дальность → ~60 параметров NPC, затем пер-NPC `size_k/interval_k`.
8. **Lua-хуки** в actions: `_G.CAI_Stalker__OnTakeCoverDestination`, `_G.CAI_Stalker__ShouldThrow` — точки расширения скриптами (ит. 14).
9. **`save/load`** сериализует состояние planner'ов через `CAI_Stalker::save` (порц. 12).

## Связи

```mermaid
graph TD
  S[CAI_Stalker] -->|brain()| P[CStalkerPlanner "root, CActionPlannerScript"]
  P --> D[DeathPlanner]
  P --> A[AnomalyPlanner]
  P --> Da[DangerPlanner]
  P --> C[CombatPlanner]
  P --> AL[ALifePlanner]
  P --> K[KillWoundedPlanner]
  P --> G[GetDistancePlanner]
  P --> L[LowCoverPlanner]
  P --> SE[SearchPlanner]
  Da --> DU[Unknown]
  Da --> DI[InDirection]
  P --> PE["property-eval'ы (stalker_property_evaluators)"]
  P --> CA["combat actions (1668 стр.)"]
  CA --> MV["movement() — порц. 14"]
  CA --> OH["CObjectHandler.set_goal — порц. 26"]
  CA --> MEM["memory/agent — порц. 22–23"]
```

- Класс-хозяин `CAI_Stalker` — порц. 12 (`Think()` = `brain().update` + `movement().update`).
- Исполнение: `stalker_movement_manager_smart_cover` — порц. 14; анимация/стрельба/торговля — порц. 15.
- `CObjectHandler`/`object_actions` — порц. 26; `memory_manager`/`agent_manager` — порц. 22–23; `cover_evaluators`/`smart_cover` — порц. 24; Lua-скрипты planner'ов — ит. 14.

## Краткое резюме

«Мозг» `CAI_Stalker` — `CStalkerPlanner` (`CActionPlannerScript`), граф состояний: ~85 property'ов + eval'аторы + ~90 operators/actions, а не FSM. Root раскладывает приоритеты death > anomaly > danger > combat > alife. Combat — основной цикл (cover/look-out/hold/detour/search/kill/aim/throw/smart-cover) с `select_queue_params` (типа оружия × дальности). Danger — 4 под-planner'а по типу опасности (unknown/in-direction/grenade/by-sound, последний — заглушка). Search — ambush-цикл. KillWounded — добивание (с виртуальным `GE_HIT`). Death — выброс инвентаря. Anomaly — выход из зоны + бросок болта. `affect_cover` — мост action ↔ root; Lua-хуки — точки расширения. Исполнение — `movement()` (порц. 14), стрельба/анимация — порц. 15, `set_goal` — порц. 26.

Порцион 13 (итерация 6). Далее: порц. 14 — `stalker-movement.md` — `CAI_Stalker`: движение и покрытия ([stalker-movement.md](stalker-movement.md)).
