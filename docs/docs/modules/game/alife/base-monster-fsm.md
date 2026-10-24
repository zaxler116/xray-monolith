# CBaseMonster: FSM-цикл, память, контроллеры

> Источники: `base_monster_think.cpp` (126), `base_monster.cpp` (1 111), `base_monster_script.cpp`
> (602), `state_manager.h` (30). Порцион 3, итерация 6.

Порцион 2 описал **каркас**; этот — **как каркас «тикает»**: три цикла обновления (логика/
расписание/рендер), FSM-тик через `IStateManagerBase* StateMan`, связка с памятью и
контроллерами, и режим скриптового управления. Конкретные **состояния** FSM разбираются
позже (порц. 18–20); здесь — только «конвейер», который их запускает.

## 1. Три цикла: Think / shedule_Update / UpdateCL

`CBaseMonster` переопределяет **три** точки, которые движок зовёт на разных фазах кадра
(общий цикл `CObject` — ит. 5):

| Метод | Фаза | Что делает |
|---|---|---|
| `Think()` | **логика** (ALife, поредленно) | `InitThink` → `anim().ScheduledInit` → `UpdateMemory` → `monster_squad().update` → `update_fsm()` |
| `shedule_Update(dt)` | **расписание** (каждый тик) | `update_eyes_visibility`, `m_anti_aim->update_schedule`, 4× `aura.update_schedule`, `control().update_schedule`, `Morale.update_schedule`, `m_anomaly_detector->update_schedule` |
| `UpdateCL()` | **рендер** (каждый кадр) | `inherited::UpdateCL`; если жив: `update_enemy_accessible_and_at_home_info`, `CStepManager::update(false)`, `update_pos_by_grouping_behaviour`; затем `control().update_frame`, `m_pPhysics_support->in_UpdateCL` |

Ключевой принцип: **«мышление» (Think) — редкое**, а «дыхание» (`shedule_Update`/`UpdateCL`) —
**ежекадровое**. FSM принимает решение редко; анимация/движение/ауры интерполируются часто.
`Think()` сразу выход при `!g_Alive() || getDestroy()` (base_monster_think.cpp:18).

## 2. `Think()` — логический тик

```cpp
void CBaseMonster::Think() {                       // base_monster_think.cpp:13
    if (!g_Alive() || getDestroy()) return;
    InitThink();                    // виртуальный (по умолчанию пуст); CController заполняет EnemyMemory
    anim().ScheduledInit();
    UpdateMemory();                 // память + флаги m_bDamaged/m_bAggressive (порц. 2, §10)
    monster_squad().update(this);   // сквад/группировка
    update_fsm();                   // ← FSM-тик
}
```

`InitThink()` — **виртуальный, по умолчанию пустой** (base_monster.h:114). `CController::InitThink`
(controller.cpp:356) использует его: прогоняет подконтрольных монстров и записывает их врагов
в `EnemyMemory` контроллера.

## 3. `update_fsm()` / `post_fsm_update()`

```cpp
void CBaseMonster::update_fsm() {                  // base_monster_think.cpp:45
    StateMan->update();                 // ← вся логика FSM (порц. 18-20)
    post_fsm_update();                  // донастроить «смотреть вперёд при беге»
    TranslateActionToPathParams();      // action → скорости/путь (см. §5)
    squad_notify();                     // сообщить скваду свою цель
    debug_fsm();                        // DEBUG
}
```

`StateMan` — **указатель на `IStateManagerBase`**, создаётся конкретным видом (здесь только
`StateMan = 0` в конструкторе; реальный объект — `CMonsterStateManager`/`CStateManagerDog`/
`CStateManagerCat`/…). Контракт `IStateManagerBase` (state_manager.h):

```cpp
virtual void reinit() = 0;
virtual void update() = 0;                       // главный тик FSM
virtual void force_script_state(EMonsterState) = 0;   // принудить состояние из скрипта
virtual void execute_script_state() = 0;
virtual void critical_finalize() = 0;           // «чистый» финал при переходе под скрипт
virtual void remove_links(CObject* O) = 0;
virtual EMonsterState get_state_type() = 0;
virtual bool check_control_start_conditions(ControlCom::EControlType) = 0;
```

`post_fsm_update()` (think:62) — мелкая коррекция: если `eStateAttack` + двигаемся по пути,
сравнивает yaw пути с направлением на врага; при расхождении `PI/3..5PI/6` ставит
`m_bRunTurnLeft/Right` (плавный поворот головы во время погони/атаки на бегу).

## 4. `TranslateActionToPathParams()` — мост «действие → движение»

`base_monster.cpp:637`. Читает текущий `EAction` из `anim().m_tAction` (FSM только что выбрал
анимацию) и переводит его в **маски скоростей** пути (`path().set_velocity_mask/
set_desirable_mask`) + включать/выключать путь:

- `ACT_STAND_IDLE/SIT/LIE/EAT/SLEEP/REST/LOOK_AROUND` → `bEnablePath=false` (`path().disable_path`).
- `ACT_ATTACK` → если AOM выключен (`m_attack_on_move_params.enabled`), путь **выключен**;
  если включён — маски беговые (`RunNormal`/`RunDamaged` по `m_bDamaged`).
- `ACT_WALK_FWD/RUN/DRAG/STEAL/HOME_WALK_*` → соответствующие маски (с учётом `m_bDamaged`).
- `state_invisible` → маски `Invisible`; `m_force_real_speed` → `vel_mask = des_mask`.

Итог: **FSM выбирает анимацию-действие, а этот метод решает, как монстр физически движется**
(скорость и «желательность» на пути). Это точка, где «мозг» превращается в «ноги» (порц. 21).

## 5. `ProcessScripts()` — режим скриптового управления

Помимо «автономного» `Think()`, монстр может быть **под контролем Lua** (через
`CCustomMonster`/`CScriptEntity`, ит. 5). `CBaseMonster::ProcessScripts()` (script:476) —
переопределение, которое **вклинивает FSM в скриптовый конвейер**:

```cpp
void CBaseMonster::ProcessScripts() {              // base_monster_script.cpp:476
    if (!g_Alive() || m_script_processing_active) return;
    m_script_processing_active = true;
    m_script_state_must_execute = false;
    inherited::ProcessScripts();          // CScriptEntity: очередь CScriptEntityAction (bfAssignWatch/Animation/Sound/Object/Movement/Monster)
    UpdateMemory();
    anim().accel_deactivate();
    if (m_script_state_must_execute)      // если скрипт выбрал «универсальное» действие
        StateMan->execute_script_state();
    TranslateActionToPathParams();
    if (m_script_state_must_execute) { EnemyMan.unforce_enemy(); CorpseMan.unforce_corpse(); }
    m_script_processing_active = false;
}
```

Мост «скрипт → FSM» — `bfAssignMonsterAction` (script:429): `CScriptMonsterAction` с
`eGA_Rest/Eat/Attack/Panic` → `EnemyMan.force_enemy(pE)` / `CorpseMan.force_corpse(pE)` +
`StateMan->force_script_state(eStateAttack/eStateEat/...)`, и флаг `m_script_state_must_execute`.
После выполнения — `unforce_enemy/Corpse` (принудительные цели сбрасываются, монстр
«возвращается» к автономии).

`SetScriptControl` (script:566) — **вход под скрипт**: `StateMan->critical_finalize()` +
`patrol().make_inactual()` (если путь не GamePath — и detail). Т.е. при захвате монстра скриптом
FSM корректно «закрывается».

## 6. Где память и контроллеры

`CBaseMonster` **не владеет** логикой памяти/контроллеров — только **владельцем полей** и
**точки вызова**:

- Память (4 `*Memory` + `EnemyMan`/`CorpseMan`, порц. 22): обновляется в `UpdateMemory()`
  (порц. 2 §10), которая зовётся из `Think()` **и** из `ProcessScripts()`.
- Контроллеры (`CControl_Manager`, порц. 21): `control().update_schedule()` (shedule),
  `control().update_frame()` (UpdateCL), `TranslateActionToPathParams` (Think/ProcessScripts).
- Аномалии/ауры/мораль: `shedule_Update` (§1).
- Сквад (`CMonsterSquad`): `Think()` → `monster_squad().update` + `squad_notify()` (think:95)
  транслирует текущее состояние FSM в «цель сквада» (`MG_AttackEnemy`/`MG_Rest`/`MG_WalkGraph`).

## 7. Связи

```mermaid
graph TD
    SCHED[shedule_Update: control/aura/morale/anomaly update_schedule] --> CTRL[CControl_Manager, порц. 21]
    CL[UpdateCL: CStepManager::update, control().update_frame] --> STEPS[CStepManager + путь]
    THINK[Think: InitThink, UpdateMemory, squad] --> MEM[Память, порц. 22]
    THINK --> UF[update_fsm]
    UF --> SM[StateMan->update: FSM-состояния, порц. 18-20]
    UF --> TF[TranslateActionToPathParams: action→скорости]
    TF --> CTRL
    UF --> SQ[squad_notify]
    SCRIPT[ProcessScripts: bfAssignMonsterAction→force_script_state] --> SM
    SCRIPT --> MEM
    SCRIPT --> TF
```

## 8. Замечания к плану

1. **Три цикла, а не один** — `Think` (логика, редкий) отделён от `shedule_Update`/`UpdateCL`
   (каждый кадр). План в заголовке «FSM-цикл» — корректно, но важно: FSM-тик (`StateMan->update`)
   вызывается **только** из `update_fsm()` (в `Think`) и из `execute_script_state()` (в
   `ProcessScripts`). `shedule_Update`/`UpdateCL` FSM **не зовут**.
2. `StateMan` — **указатель, не объект**: `CBaseMonster` не создаёт FSM; конкретный вид ставит
   свой `IStateManagerBase`. Поэтому `Think()` в `CBaseMonster` без `StateMan` упадёт — вид обязан
   её установить (в `Load`/`_construct`).
3. `CAI_Rat` **не идёт** через этот конвейер: у него собственный `Think()` с явным `switch` по
   `aiRatDie/FreeActive/...` (ai_rat_fsm.cpp:50) — нет `StateMan`, нет `update_fsm`. Подтверждает
   порц. 2: крыса — отдельная ветка, не `CBaseMonster`.
4. `CAI_Stalker` — тоже **не** через `StateMan`: `Think()` у него `brain().update` +
   `movement().update` (ai_stalker.cpp:1292), планировщики (порц. 15). Третий вариант «мозга».

## 9. Краткое резюме

`CBaseMonster` тикает в **трёх фазах**: `Think()` (редкая логика: память → сквад → `update_fsm`),
`shedule_Update` (ежекадровое «дыхание»: контроллеры/ауры/мораль/аномалии) и `UpdateCL` (рендер:
шаги/путь/физика). FSM — за `IStateManagerBase* StateMan` (создаётся видом, порц. 18);
`TranslateActionToPathParams` превращает выбранный `EAction` в скорости пути. Альтернативный
режим — `ProcessScripts()` (Lua-захват через `force_script_state`). Крыса и сталкер используют
**свои** `Think()` и этот конвейер не ходят.

---

*Порцион 3 (итерация 6). Далее: порц. 4 — `CBaseMonster`: настройки `SMonsterSettings` ([base-monster-settings.md](base-monster-settings.md)).*
