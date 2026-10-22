# ALIFE (AI): обзор

> **Модуль:** `xrGame` (AI/ALife-слой) · **Файлы:** `ai/` (`monsters/` 457 файлов ~43 600 стр.,
> `stalker/` 13×6 985, `crow/`/`trader/`/`phantom/`), корень `xrGame/` — `alife_*.h/.cpp`
> (~140 файлов, `alife_simulator.*`, `alife_*_manager.*`, `alife_*_registry.*`), `ai_space.*`,
> `ai_object_location.*`, `ai_obstacle.*`, `CustomMonster.*`, `monster_community.*`,
> `relation_registry.*` · **Порцион 1, итерация 6**

Архитектура AI в этом форке **не классическая** (нет `CAI_Base`/`CAI_Monster`/`CAI_Manager`/
`ai_memory`/`ai_planner` из оригинального X-Ray). Вместо единого дерева — два параллельных стека
(монстры и сталкеры-люди), несколько коротких отдельных веток и общий слой симулятора ALife в корне
`xrGame/`. Эта страница — карта иерархии, веток и цикла обновления; детали — в порционах 2–32.

## 1. Область и «Чего не делает»

**Документируется:** `ai/monsters/*` (база, 15 видов, FSM, состояния, контроллеры, память, покрытия),
`ai/stalker/*`, `ai/{crow,trader,phantom}/*`, слой ALife (`CALifeSimulator` и менеджеры, регистры,
abstract-обёртки, `ai_space`, `CAI_ObjectLocation`, `ai_obstacle`), `MONSTER_COMMUNITY`/relations.

**Не документируется здесь:**
- **Сетевые обёртки `CSE_ALife*`** — итерация 11/12; тут только `CSE_Abstract*`/`CSE_ALifeObject` как
  аргумент `net_Spawn`/типы регистров.
- **Полный Lua API AI** (`alife_*_brain_script.cpp`, `stalker_*_script.cpp`) — итерация 14; здесь
  только сигнатуры публичных скриптовых методов.
- **`CCustomMonster` как мост** — класс разобран в ит. 5 (порц. 9, [custom-monster.md](../custom-monster.md));
  здесь только то, что за ним (`Think()`, `CBaseMonster`, `CAI_Stalker`).
- **`feel`-фундамент** — ит. 5 (порц. 41–42, [feel.md](../feel.md)); здесь реализации `Feel::*` в
  `CCustomMonster`/`CAI_Stalker` лишь упоминаются.
- **`cover_evaluators_inline.h`** (~199k строк) — читается выборочно: только API и структура.

## 2. Два параллельных стека

| Стек | База | Потомки | Где |
| --- | --- | --- | --- |
| **Монстры** | `CCustomMonster` → `CBaseMonster` | 15 видов (`CAI_Bloodsucker`, `CController`, `CBurer`, `CPoltergeist`, `CChimera`, `CAI_Boar`, `CAI_Dog`, `CAI_PseudoDog`, `CCat`, `CTushkano`, `CZombie`, `CAI_Flesh`, `CFracture`, `CPseudoGigant`, `CSnork`) | `ai/monsters/` |
| **Сталкеры (NPC-люди)** | `CAI_Stalker` | planner'ы, движение/покрытия, анимация, торговля, стрельба | `ai/stalker/` |

Ключевое отличие от классики: у монстров нет общего `CAI_Monster`-планировщика — их поведение
реализовано **FSM** (`CState`/`CMonsterStateManager`/`EMonsterState`), тогда как у сталкеров —
**планировщик действий** (`CStalker*Planner` на базе `CActionPlannerScript`). То есть внутри одного
«AI» сосуществуют две разные парадигмы принятия решений.

```cpp
// ai/monsters/basemonster/base_monster.h
class CBaseMonster : public CCustomMonster, public CStepManager;   // порц. 2
    
// ai/stalker/ai_stalker.h:89
class CAI_Stalker : public CCustomMonster, public CObjectHandler,
                    public CAI_PhraseDialogManager, public CStepManager;
```

## 3. Отдельные ветки

| Ветка | База | Особенность | Порцион |
| --- | --- | --- | --- |
| `CAI_Rat` | `CCustomMonster + CEatableItem` | **не** `CBaseMonster`; состояния в `ai/monsters/rats/` | 10 |
| `CAI_Trader` | `CEntityAlive` | торговец; сам не-CustomMonster | 11 |
| `CAI_Crow` | `CEntity` | ворон, простая логика | 11 |
| `CPhantom` | `CEntity` | фантом | 11 |

`CAI_Rat`/`CAI_Trader`/`CAI_Crow`/`CPhantom` не проходят через `CCustomMonster`-мост `Think()` в том же
виде, что монстры/сталкеры, — это отдельные короткие ветки.

## 4. Слой ALife (симулятор мира)

Параллельно клиентскому AI живёт **симулятор ALife** — глобальная модель мира (`online`/`offline`
объекты), ядро которого в корне `xrGame/`.

### 4.1. Иерархия `CALifeSimulator`

`CALifeSimulator` собран множественным наследованием из менеджеров (композиция через базы):

```cpp
// alife_simulator.h:18
class CALifeSimulator : public CALifeUpdateManager,
                        public CALifeInteractionManager;

// alife_update_manager.h:20
class CALifeUpdateManager : public CALifeSwitchManager,
                            public CALifeSurgeManager,
                            public CALifeStorageManager,
                            public ISheduled;

// alife_interaction_manager.h:15
class CALifeInteractionManager : public CALifeCombatManager,
                                 public CALifeCommunicationManager;

// все ветки сходятся в: alife_simulator_base.h:42
class CALifeSimulatorBase : public IPureDestroyableObject;
```

Итого цепочка: `CALifeSimulator` → {`CALifeUpdateManager`, `CALifeInteractionManager`} → … →
`CALifeSimulatorBase` (+ `ISheduled`).

### 4.2. Менеджеры

| Менеджер | Ответственность | Файл |
| --- | --- | --- |
| `CALifeUpdateManager` | шедулер ALife, `update()`, `new_game`/`load` | `alife_update_manager.cpp` (628) |
| `CALifeSwitchManager` | `online`/`offline`-переключение объектов | `alife_switch_manager.cpp` (247) |
| `CALifeSurgeManager` | спавн новых объектов (`spawn_new_objects`) | `alife_surge_manager.cpp` (70) |
| `CALifeStorageManager` | save/load мира | `alife_storage_manager.cpp` (249) |
| `CALifeTimeManager` | игровое время | `alife_time_manager.cpp` (52) |
| `CALifeInteractionManager` | торговля/общение offline-NPC | `alife_interaction_manager.cpp` |
| `CALifeCombatManager` | offline-бои (`vfPerformAttackAction`, `kill_entity`) | `alife_combat_manager.cpp` (477) |
| `CALifeCommunicationManager` | offline-торговля (`vfPerformTrading`, `vfPerformCommunication`) | `alife_communication_manager.cpp` (905) |

### 4.3. Регистры и abstract-обёртки (порционы 29–30)

`CALifeSimulatorBase` держит набор регистров: `CALifeSpawnRegistry`, `CALifeObjectRegistry`,
`CALifeGraphRegistry`, `CALifeScheduleRegistry`, `CALifeStoryRegistry`,
`CALifeSmartTerrainRegistry`, `CALifeGroupRegistry`, `CALifeRegistryContainer`. Каждый объект мира —
`CSE_ALife*`-abstract-обёртка (порц. 30; см. также ит. 5 [object-model.md](../../../architecture/object-model.md)).

## 5. Цикл обновления

Точка входа — `CALifeUpdateManager::shedule_Update(u32 dt)` (шедулер xrEngine). Логика:

```cpp
void CALifeUpdateManager::shedule_Update(u32 dt)
{
    ISheduled::shedule_Update(dt);
    if (!initialized()) return;
    if (!m_first_time && g_mt_config.test(mtALife)) {          // MT-режим
        Device.seqParallel.push_back(FastDelegate0<>(this, &CALifeUpdateManager::update));
        return;
    }
    m_first_time = false;
    update();                                                  // sync-режим
}

void CALifeUpdateManager::update()                             // alife_update_manager.cpp:113
{
    update_switch();          // online/offline по дистанции (CSwitchPredicate)
    update_scheduled(false);  // тики живых online-объектов (scheduled().update())
}
```

Два под-этапа каждого тика:
1. **`update_switch()`** — `graph().level().update(CSwitchPredicate(this), …)`: проходит объекты уровня
   и переключает их `online`/`offline` (`CALifeSwitchManager::switch_object`).
2. **`update_scheduled()`** — `scheduled().update()`: обновляет объекты, поставленные в расписание
   (у которых `bfActive()`).

При `mtALife` (многопоточность) `update()` уезжает в `Device.seqParallel`, иначе выполняется в шедулере.

## 6. Карта каталогов

```
src/xrGame/
├── ai/
│   ├── monsters/          (457 файлов, ~43 600 стр.)
│   │   ├── basemonster/   CBaseMonster (12 файлов, 3 754)
│   │   ├── states/        CStateMonster* (101 файл, 5 314)
│   │   ├── group_states/  групповые состояния (27 файлов, 2 606)
│   │   ├── bloodsucker/ controller/ burer/ poltergeist/ chimera/   (крупные виды)
│   │   ├── boar/ dog/ pseudodog/ cat/ tushkano/ zombie/            (животные)
│   │   ├── flesh/ fracture/ pseudogigant/ snork/                   (редкие)
│   │   └── rats/          CAI_Rat (отдельная ветка)
│   ├── stalker/           CAI_Stalker (13 файлов, 6 985)
│   ├── crow/ trader/ phantom/   (короткие ветки)
│   └── ai_monsters_misc.* position_prediction.h weighted_random.*
├── alife_*.*              симулятор: simulator/base, *manager, *registry, *abstract
├── ai_space.*             CAI_Space: game_graph/level_graph/cross_table/cover_manager/…
├── ai_object_location.*   CAI_ObjectLocation
├── ai_obstacle.*          ai_obstacle
├── CustomMonster.*        мост (ит. 5, порц. 9)
├── monster_community.*    MONSTER_COMMUNITY (relations)
└── relation_registry.*    RELATION_REGISTRY (отношения)
```

## 7. Точки входа (контракт для остального движка)

| Точка входа | Кто вызывает | Что делает |
| --- | --- | --- |
| `CCustomMonster::Think()` | из `Shedule`/`UpdateCL` (ит. 5, порц. 9) | вызывается при `g_Alive()`, мост в AI |
| `CBaseMonster::Think()` / `CAI_Stalker::Think()` | переопределяют | FSM-тик / planner-тик |
| `CALifeSimulator::shedule_Update` | шедулер xrEngine | тик симулятора (`update_switch`+`update_scheduled`) |
| `ai()`, `ai().alife()`, `ai().level_graph()` | весь движок | доступ к `CAI_Space` (порц. 26) |
| `MONSTER_COMMUNITY::relation(a,b)` | `RELATION_REGISTRY` (порц. 27) | отношения между монстрами |

## 8. Связи

```mermaid
graph TD
    subgraph xrEngine
        IShed[ISheduled]
        COb[CObject]
    end
    subgraph bridge
        CM[CCustomMonster<br/>ит.5 порц.9]
    end
    subgraph monsters
        BM[CBaseMonster]
        M15[15 видов монстров]
        FSM[CState / CMonsterStateManager]
        RAT[CAI_Rat<br/>CCustomMonster + CEatableItem]
    end
    subgraph stalker
        ST[CAI_Stalker]
        PLN[CStalker*Planner]
    end
    subgraph short-branches
        TRD[CAI_Trader]
        CRW[CAI_Crow]
        PH[СPhantom]
    end
    subgraph alife-layer
        SIM[CALifeSimulator]
        MGR[Update/Switch/Surge/Storage/<br/>Interaction/Combat/Comm managers]
        REG[alife_*_registry]
        ABS[CSE_ALife* abstract]
        SPACE[CAI_Space / ai_space]
    end

    COb --> CM
    CM --> BM
    CM --> RAT
    CM --> ST
    BM --> M15
    BM --> FSM
    ST --> PLN
    CEntity2[CEntityAlive] --> TRD
    CEntity3[CEntity] --> CRW
    CEntity3 --> PH
    SIM --> MGR
    SIM --> REG
    REG --> ABS
    SIM --> SPACE
    SPACE -.ai().alife().-> SIM
    CM -.Think().-> FSM
    CM -.Think().-> PLN
```

- **Ит. 5:** `CCustomMonster` — мост (порц. 9, [custom-monster.md](../custom-monster.md)); `Feel::*`
  (порц. 41–42, [feel.md](../feel.md)); `CAI_ObjectLocation`/`ai_obstacle` (порц. 3 ит. 5 — закрываются
  здесь, порц. 26); `MONSTER_COMMUNITY` (ит. 5 порц. 5 «частично закрыто» — полностью порц. 27).
- **Ит. 7** (физика), **ит. 9** (зоны) — smart-террейн/аномалии как объекты окружения AI.
- **Ит. 11/12** — `CSE_ALife*` сетевые обёртки (здесь только `CSE_Abstract*` в `net_Spawn`).
- **Ит. 14** — полный Lua API (`alife_*_brain_script.cpp`, `stalker_*_script.cpp`).

## 9. Иерархия AI (сводка)

```cpp
CEntityAlive → CCustomMonster → CBaseMonster → CAI_Bloodsucker, CAI_Boar, CBurer, CCat,
                            CChimera, CController, CAI_Dog, CAI_Flesh, CFracture,
                            CPoltergeist, CAI_PseudoDog, CPseudoGigant, CSnork,
                            CTushkano, CZombie
CEntityAlive → CCustomMonster → CAI_Rat        (напрямую + CEatableItem, не CBaseMonster)
CEntityAlive → CCustomMonster → CAI_Stalker    (+ CObjectHandler + CAI_PhraseDialogManager + CStepManager)
CEntityAlive → CAI_Trader
CEntity      → CAI_Crow, CPhantom
```

## 10. Замечания к плану

1. **План называет ~54 000 строк в `ai/monsters/`** — фактически **457 файлов, ~43 600 строк**
   (`Measure-Object -Line`; точный обход). Порядок величины тот же, но цифру стоит держать реальной.
2. **`ai/stalker/` — план ~8 200 строк** → факт **13 файлов, 6 985 строк**.
3. **`CAI_Trader` = `CEntityAlive`** (подтверждено), но это не «одиночный» класс: `ai/trader/` — 5 файлов
   (602 строки). В план-таблице (порц. 11) он идёт вместе с `CAI_Crow`/`CPhantom`.
4. **`CPhantom : CEntity`** — подтверждено (план верен).
5. **`cover_evaluators_inline.h`** действительно огромен, но в `ai/monsters/` его нет — он в корне
   `xrGame/` (`cover_evaluators_inline.h`); план (решение 8) относит его к порц. 24.
6. **`CALifeSimulator` — не отдельный класс с логикой**, а композиция 7 менеджеров через множественное
   наследование (см. §4.1); вся логика — в менеджерах. Учитывать при разборе порц. 31–32.
7. **`ai/monsters/states/` содержит 101 файл** (план: «101 файл» — верно), `group_states/` — **27** (план
   «27 файлов» — верно).
8. **Публичного API у AI почти нет** — `Think()` защищённый; внешние точки входа — шедулер ALife и
   `CCustomMonster`-мост. Контракт «Публичный API» реалистично ограничен шедулером и `ai()`.

## 11. Краткое резюме

Итерация 6 разбирает AI-слой xrGame: **два стека** (монстры через FSM `CBaseMonster`, сталкеры через
планировщики `CAI_Stalker`), **короткие ветки** (`CAI_Rat`, `CAI_Trader`, `CAI_Crow`, `CPhantom`) и
**симулятор ALife** (`CALifeSimulator` = композиция менеджеров + регистры + `CSE_ALife*` abstract-обёртки),
живущий в корне `xrGame/`. Цикл ALife: `shedule_Update` → `update()` → `update_switch()` +
`update_scheduled()`. Точка стыка с остальным движком — мост `CCustomMonster` (ит. 5, порц. 9) и
`ai()` (`CAI_Space`). Разбор — порционы 2–32 этой итерации.

---

*Порцион 1 (итерация 6). Далее: порц. 2 — `CBaseMonster`: каркас и abilities ([base-monster.md](base-monster.md)).*
