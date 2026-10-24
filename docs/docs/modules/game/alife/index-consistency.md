# Связность: object-model + зависимости + index

> Итерация 6, порцион 33 (финальный).
> Обновление страниц: `docs/docs/architecture/object-model.md`, `docs/docs/architecture/dependencies.md`, `docs/docs/modules/game/index.md`, `docs/mkdocs.yml`.

## Чего не делает / границы

- **Разбор AI** — порционы 1–32 этой итерации.
- **Сеть** (`CSE_ALife*`, `net_Spawn`) — ит. 11/12.
- **Lua API AI** — ит. 14.

## 1. Обновление `object-model.md`

### 1.1. Добавление веток AI в клиентскую иерархию

В `docs/docs/architecture/object-model.md` (раздел «Иерархия (клиент)») добавлены:

```cpp
// После CCustomMonster:
CCustomMonster <|-- CBaseMonster : base_monster.h:42
CBaseMonster <|-- CAI_Bloodsucker : ai_bloodsucker.h:28
CBaseMonster <|-- CAI_Controller : ai_controller.h:30
CBaseMonster <|-- CBurer : ai_burer.h:29
CBaseMonster <|-- CPoltergeist : ai_poltergeist.h:28
CBaseMonster <|-- CChimera : ai_chimera.h:29
CBaseMonster <|-- CAI_Dog : ai_dog.h:27
CBaseMonster <|-- CAI_Boar : ai_boar.h:27
CBaseMonster <|-- CZombie : ai_zombie.h:28
// ... и другие 15 видов

// Отдельные ветки:
CCustomMonster <|-- CAI_Rat : ai_rat.h:25
CCustomMonster <|-- CAI_Stalker : ai_stalker.h:89
CEntityAlive <|-- CAI_Trader : ai_trader.h:22
CEntity <|-- CAI_Crow : ai_crow.h:18
CEntity <|-- CPhantom : phantom.h:15
```

### 1.2. Слой ALife (серверная иерархия)

В разделе «Серверная иерархия (CSE)» добавлено:

```
CSE_Abstract
  └── CSE_ALifeObject : CSE_Abstract, CRandom
        └── CSE_ALifeDynamicObject : CSE_ALifeObject
              ├── CSE_ALifeDynamicObjectVisual : CSE_ALifeDynamicObject, CSE_Visual
              │     ├── CSE_ALifeCreatureAbstract : CSE_ALifeDynamicObjectVisual
              │     │     ├── CSE_ALifeMonsterAbstract : + CSE_ALifeSchedulable
              │     │     │     ├── CSE_ALifeHumanAbstract : CSE_ALifeTraderAbstract + CSE_ALifeMonsterAbstract
              │     │     │     │     └── CSE_ALifeHumanStalker
              │     │     │     ├── CSE_ALifeMonsterBase / MonsterRat / MonsterZombie
              │     │     │     └── CSE_ALifePsyDogPhantom
              │     │     ├── CSE_ALifeCreatureActor (игрок)
              │     │     └── CSE_ALifeCreatureCrow / Phantom
              │     ├── CSE_ALifeItem : CSE_ALifeDynamicObjectVisual, CSE_ALifeInventoryItem
              │     │     ├── CSE_ALifeItemWeapon → …
              │     │     └── CSE_ALifeItemArtefact / Ammo / Detector / PDA / …
              │     ├── CSE_ALifeSpaceRestrictor : CSE_ALifeDynamicObject, CSE_Shape
              │     │     ├── CSE_ALifeCustomZone
              │     │     │     └── CSE_ALifeAnomalousZone → CSE_ALifeZoneVisual
              │     │     ├── CSE_ALifeSmartZone : + CSE_ALifeSchedulable
              │     │     └── CSE_ALifeTeamBaseZone / LevelChanger
              │     └── CSE_ALifeObjectPhysic / HangingLamp / Helicopter / Car / …
              └── CSE_ALifeOnlineOfflineGroup
```

### 1.3. Связи

Добавлены стрелки:
- `CCustomMonster → CBaseMonster` → 15 видов монстров (FSM).
- `CCustomMonster → CAI_Stalker` (planner'ы).
- `CCustomMonster → CAI_Rat` (отдельная ветка).
- `CEntityAlive → CAI_Trader`, `CEntity → CAI_Crow/CPhantom`.
- `CALifeSimulator` (композиция менеджеров) ↔ `CSE_ALife*` (регистры).

## 2. Обновление `dependencies.md`

### 2.1. Внутренние зависимости `xrGame`

В разделе «Прямые `#include` между `xrGame` и другими» добавлено:

```
xrGame (AI/ALife) тянет:
- xrEngine: ISheduled, CObject, CAI_ObjectLocation, ai_space
- xrServerEntities: CSE_ALife* (abstract-обёртки)
- xrSound: ai_sounds (типы звуков)
- xrCDB: CSE (сериализация)
```

### 2.2. Таблица зависимостей

Добавлена строка:

| Подмодуль | Тянет | Зачем |
|-----------|-------|-------|
| `xrGame/ai/` | `xrEngine`, `xrServerEntities` | AI-логика, FSM, planner'ы |
| `xrGame/alife_*` | `xrEngine`, `xrServerEntities`, `xrCDB` | Симулятор ALife, регистры, менеджеры |

## 3. Обновление `modules/game/index.md`

### 3.1. Строка ALIFE

В таблице порционов обновлена строка:

```markdown
| 48 | Связность: object-model + зависимости + `index.md` | [object-model.md](../../architecture/object-model.md), [dependencies.md](../../architecture/dependencies.md) | ✅ |
```

Добавлена ссылка на ALIFE:

```markdown
- **ALIFE** (ит. 6) — 33 страницы: [alife/index.md](alife/index.md).
```

### 3.2. Карта связей

В mermaid-диаграмме добавлена стрелка:

```mermaid
Mon -.Think / FSM / planner.-> AI[ai/ — итерация 6]
AI --> ALIFE[CALifeSimulator + CSE_ALife*]
```

## 4. Обновление `mkdocs.yml`

Заменена заглушка `alife.md` на подкаталог:

```yaml
- ALIFE (AI) (Ит. 6):
    - Обзор: modules/game/alife/index.md
    - "CBaseMonster: каркас и abilities": modules/game/alife/base-monster.md
    - "CBaseMonster: FSM-цикл, память, контроллеры": modules/game/alife/base-monster-fsm.md
    - "CBaseMonster: настройки, ауры, berserk": modules/game/alife/base-monster-settings.md
    - "CAI_Bloodsucker": modules/game/alife/bloodsucker.md
    - "CAI_Controller": modules/game/alife/controller.md
    - "CBurer": modules/game/alife/burer.md
    - "CPoltergeist": modules/game/alife/poltergeist.md
    - "CChimera": modules/game/alife/chimera.md
    - "CAI_Rat": modules/game/alife/rat.md
    - "CAI_Trader / CAI_Crow / CPhantom": modules/game/alife/trader-crow-phantom.md
    - "CAI_Stalker: класс, feel, события": modules/game/alife/stalker.md
    - "CAI_Stalker: planner'ы": modules/game/alife/stalker-planners.md
    - "CAI_Stalker: движение и покрытия": modules/game/alife/stalker-movement.md
    - "CAI_Stalker: анимация, стрельба, торговля": modules/game/alife/stalker-anim-trade.md
    - "Монстры-«животные»": modules/game/alife/monsters-animals.md
    - "Монстры редкие": modules/game/alife/monsters-rare.md
    - "FSM-фундамент": modules/game/alife/monster-fsm.md
    - "Состояния монстров (states/)": modules/game/alife/monster-states.md
    - "Групповые состояния (group_states/)": modules/game/alife/monster-group-states.md
    - "Контроллеры CControl_*": modules/game/alife/control-managers.md
    - Память: modules/game/alife/memory.md
    - Враги: modules/game/alife/enemy.md
    - Покрытия: modules/game/alife/cover.md
    - "Steering + CStepManager": modules/game/alife/steering-steps.md
    - "Пространства AI": modules/game/alife/ai-space.md
    - "MONSTER_COMMUNITY": modules/game/alife/community.md
    - "CALifeSimulator": modules/game/alife/alife-simulator.md
    - "Регистры ALife": modules/game/alife/alife-registries.md
    - "Abstract-обёртки ALife": modules/game/alife/alife-abstract.md
    - "Менеджеры ALife": modules/game/alife/alife-managers.md
    - "Combat/communication managers": modules/game/alife/alife-combat-communication.md
    - Связность: modules/game/alife/index-consistency.md
```

## 5. Итог итерации 6

### 5.1. Что сделано

- **33 страницы** в `docs/docs/modules/game/alife/`:
  - `index.md` — обзор архитектуры AI.
  - 15 страниц по видам монстров (bloodsucker, controller, burer, poltergeist, chimera, rat, trader/crow/phantom, сталкеры, животные, редкие).
  - 8 страниц по FSM/состояниям/контроллерам/памяти/врагам/покрытиям/steering.
  - 6 страниц по слою ALife (simulator, registries, abstract, managers, combat/communication).
  - `index-consistency.md` — эта страница.

- **Обновлены страницы архитектуры**:
  - `object-model.md` — ветки AI + слой ALife.
  - `dependencies.md` — зависимости AI/ALife.
  - `modules/game/index.md` — ссылка на ALIFE, стрелка в карте связей.

- **Обновлён `mkdocs.yml`** — подкаталог ALIFE с 33 пунктами.

### 5.2. Обещания из итераций 0–5 (выполнены)

| Обещание | Итерация | Статус |
|----------|----------|--------|
| `MONSTER_COMMUNITY` (полный разбор) | 6, порц. 27 | ✅ |
| `CCustomMonster`-мост (AI-внутренности) | 6, порц. 2–17 | ✅ |
| `CAI_ObjectLocation`/`ai_obstacle` | 6, порц. 26 | ✅ |
| `Feel::*` (реализации в AI) | 6, порц. 2/12 | ✅ |
| `CSpaceRestrictor ↔ AI` | 6, порц. 19/30 | ✅ |

### 5.3. Что осталось на другие итерации

| Тема | Итерация |
|------|----------|
| `CSE_ALife*` (сетевые обёртки, полная детализация) | 11/12 |
| Полный Lua API AI (`alife_*_brain_script.cpp`, `stalker_*_script.cpp`) | 14 |

## Связи

- **Порц. 1–32** — разбор AI/ALife.
- **Ит. 5** — `CCustomMonster` (порц. 9), `Feel::*` (порц. 41–42), `CAI_ObjectLocation` (порц. 3).
- **Ит. 11/12** — `CSE_ALife*` сетевые обёртки.
- **Ит. 14** — полный Lua API.

## Замечания к плану

1. **План называет ~54 000 строк в `ai/monsters/`** — фактически 457 файлов, ~43 600 строк.
2. **`ai/stalker/` — план ~8 200 строк** → факт 13 файлов, 6 985 строк.
3. **`CALifeSimulator` — не отдельный класс с логикой**, а композиция 7 менеджеров через множественное наследование.
4. **`CALifeHumanObjectHandler`** — все методы заглушки; реальная логика в `brain().objects()`.
5. **`cover_evaluators_inline.h`** (~199k строк) — читается выборочно: только API и структура.

## Краткое резюме

Итерация 6 завершена:
- **33 страницы** в `docs/docs/modules/game/alife/` (AI/ALife).
- **Обновлены** `object-model.md`, `dependencies.md`, `modules/game/index.md`, `mkdocs.yml`.
- **Выполнены** все обещания из итераций 0–5.
- **План** (`docs/ITERATION-6-PLAN.md`) удаляется.

---
Порцион 33 (итерация 6). **Итерация 6 завершена.**
