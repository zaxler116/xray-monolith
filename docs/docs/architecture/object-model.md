# Модель объектов

Иерархия объектов X-Ray Monolith. Это «дерево классов», к которому сводится всё игровое: персонажи, оружие, зоны, частицы — всё наследуется от этой иерархии.

## Иерархия (клиент)

Ниже — **фактическая** цепочка наследования, снятая с объявлений классов (файл:строка указаны у каждого ребра). Ключевой факт: `CEntity` наследует не `CGameObject` напрямую, а `CPhysicsShellHolder`, а предметы идут через `CPhysicItem`/`CInventoryItemObject`, а не через `CEntity`.

```mermaid
classDiagram
    class CObject {
        +xr_name m_name
        +Fvector m_position
        +Fmatrix m_G
        +u32 m_flags
        +virtual OnUpdate()
        +virtual OnFrame()
        +virtual Render()
    }
    class CGameObject {
        +CAI_ObjectLocation* m_ai_location
        +ALife::_STORY_ID m_story_id
        +animation_movement_controller* m_anim_mov_ctrl
        +virtual cast_actor()
        +virtual cast_weapon()
        +virtual cast_entity()
    }
    class CPhysicsShellHolder
    class CEntity {
        +ALife::_TIME_ID m_dwBodyRemoveTime
        +CEntityConditionSimple* m_entity_condition
        +float m_fFood
    }
    class CEntityAlive {
        +CEntityCondition* m_entity_condition
        +MONSTER_COMMUNITY* monster_community
    }
    class CActor {
        +CCameraBase* cameras[]
        +SStatisticManager* m_statistic_manager
        +CActorMemory* m_memory
    }
    class CCustomMonster {
        +Feel::Vision / Feel::Sound / Feel::Touch
        +CMemoryManager* m_memory_manager
    }
    class CAI_Stalker {
        +CObjectHandler / CStepManager
    }
    class CPhysicItem
    class CInventoryItemObject
    class CHudItemObject
    class CWeapon {
        +Fvector m_fire_position
        +CShootingObject
    }
    class CArtefact {
        +CPHUpdateObject
    }
    class CSpaceRestrictor
    class CCustomZone {
        +Feel::Touch
    }

    CObject <|-- CGameObject : GameObject.h:53
    CGameObject <|-- CPhysicsShellHolder : PhysicsShellHolder.h:26
    CPhysicsShellHolder <|-- CEntity : Entity.h:17
    CEntity <|-- CEntityAlive : entity_alive.h:18
    CEntityAlive <|-- CActor : Actor.cpp:120
    CEntityAlive <|-- CCustomMonster : CustomMonster.h:34
    CCustomMonster <|-- CAI_Stalker : ai_stalker.h:89
    CPhysicsShellHolder <|-- CPhysicItem : physic_item.h:15
    CPhysicItem <|-- CInventoryItemObject : inventory_item_object.h:14
    CInventoryItemObject <|-- CHudItemObject : hud_item_object.h:6
    CHudItemObject <|-- CWeapon : Weapon.h:49
    CHudItemObject <|-- CArtefact : Artefact.h:12
    CGameObject <|-- CSpaceRestrictor : space_restrictor.h:15
    CSpaceRestrictor <|-- CCustomZone : CustomZone.h:37
```

> **Примечание о множественном наследовании** (на диаграмме показана одна «главная» ветка на класс):
> - `CGameObject` = `CObject` + `CUsableScriptObject` + `CScriptBinder` (`GameObject.h:53`)
> - `CPhysicsShellHolder` = `CGameObject` + `CDamageReceiver` и др. (`PhysicsShellHolder.h:26`)
> - `CEntity` = `CPhysicsShellHolder` + `CDamageManager` (`Entity.h:17`)
> - `CCustomMonster` = `CEntityAlive` + `CScriptEntity` + `Feel::Vision` + `Feel::Sound` + `Feel::Touch` (`CustomMonster.h:34`)
> - `CAI_Stalker` = `CCustomMonster` + `CObjectHandler` + `CAI_PhraseDialogManager` + `CStepManager` (`ai_stalker.h:89`)
> - `CPhysicItem` = `CPhysicsShellHolder` + `CPHShellSimpleCreator` (`physic_item.h:15`)
> - `CInventoryItem` = `CAttachableItem` + `CHitImmunity` (`inventory_item.h:63`)
> - `CInventoryItemObject` = `CInventoryItem` + `CPhysicItem` (`inventory_item_object.h:14`)
> - `CHudItemObject` = `CInventoryItemObject` + `CHudItem` (`hud_item_object.h:6`)
> - `CWeapon` = `CHudItemObject` + `CShootingObject` (`Weapon.h:49`)
> - `CArtefact` = `CHudItemObject` + `CPHUpdateObject` (`Artefact.h:12`)
> - `CCustomZone` = `CSpaceRestrictor` + `Feel::Touch` (`CustomZone.h:37`)
>
> То есть ветка живых существ («персонажи») и ветка предметов («инвентарь») расходятся сразу под `CPhysicsShellHolder`: живые — через `CEntity`, предметы — через `CPhysicItem`.

## Базовые классы

### `CObject` — `xrEngine/xr_object.h`

Базовый **рендерящийся** объект движка. Хранит:
- `xr_name m_name` — имя
- `Fvector m_position`, `Fmatrix m_G` — трансформ
- `u32 m_flags` — флаги состояния
- Реализует `ISheduled` (`OnUpdate`, `OnFrame`) и `IRenderable` (`Render`)

**Не** знает про игровую логику — только про «я объект в мире, у меня трансформ и рендер».

### `CGameObject` — `xrGame/GameObject.h`

Базовый **игровой** объект. Наследует `CObject` + скриптовые миксы. Хранит:
- `CAI_ObjectLocation* m_ai_location` — связь с AI
- `ALife::_STORY_ID m_story_id` — ID в ALIFE
- `animation_movement_controller* m_anim_mov_ctrl` — анимация
- Набор `cast_*()` — «умные» downcast'ы без `smart_cast` (см. `cast_actor()`, `cast_weapon()`, `cast_entity()`)

> **`cast_*()` вместо `smart_cast`**: X-Ray использует виртуальные `cast_*()` для избежания дорогого `dynamic_cast`/`smart_cast` в hot-path. Каждый подкласс переопределяет свой `cast_*`.

### `CEntity` / `CEntityAlive` — `xrGame/Entity.h`, `xrGame/entity_alive.h`

`CEntity : CPhysicsShellHolder, CDamageManager` — объект с «состоянием» (`CEntityConditionSimple`) и едой (`m_fFood`), временем вывоза тела (`m_dwBodyRemoveTime`). `CEntityAlive : CEntity` — **живой** объект: фабрика условий `create_entity_condition` (см. [Условия сущности](../modules/game/entity-condition.md)), монстр-комьюнити, память о зрении. Наследники: `CActor` (игрок, `Actor.cpp:120`), `CCustomMonster` (монстр, `CustomMonster.h:34`). `CAI_Stalker` — уже потомок монстра (`ai_stalker.h:89`).

### `CWeapon` / `CArtefact` — предметы, а не живые объекты

Оба наследуют **`CHudItemObject`** (`CInventoryItemObject` + `CHudItem`), а не `CEntityAlive` и не `CGameObject` напрямую:
- `CWeapon : CHudItemObject, CShootingObject` — оружие (`Weapon.h:49`). Подклассы: `CWeaponMagazined`, `CWeaponPistol`, `CWeaponShotgun`, `CWeaponAK74`, `CWeaponSVD` и т.п. (см. [Оружие](../modules/game/weapons.md)).
- `CArtefact : CHudItemObject, CPHUpdateObject` — артефакт (`Artefact.h:12`).

Ветка предметов: `CPhysicsShellHolder → CPhysicItem → CInventoryItemObject → CHudItemObject → {CWeapon, CArtefact}`.

### `CSpaceRestrictor` / `CCustomZone`

`CSpaceRestrictor : CGameObject` (`space_restrictor.h:15`) — объём-ограничитель (набор сфер/параллелепипедов). `CCustomZone : CSpaceRestrictor, Feel::Touch` (`CustomZone.h:37`) — аномальная/радиационная зона. Более узкие зоны — см. [Зоны](../modules/game/zones.md).

## Серверная иерархия (CSE)

Сервер ведёт **параллельную** иерархию — **CSE** (Client-Server Entity), в `src/xrServerEntities/`. Классы объявляются макросом `SERVER_ENTITY_DECLARE_BEGIN*(Class, Base...)` (второй и третий родители — mix-in'ы):

```
CSE_Abstract                                     (SERVER_ENTITY_DECLARE_BEGIN, xrServer_Objects.h)
  ├── CSE_ALifeObject     : CSE_Abstract, CRandom
  │     └── CSE_ALifeDynamicObject     : CSE_ALifeObject
  │           ├── CSE_ALifeDynamicObjectVisual : CSE_ALifeDynamicObject, CSE_Visual
  │           │     ├── CSE_ALifeCreatureAbstract : CSE_ALifeDynamicObjectVisual
  │           │     │     ├── CSE_ALifeMonsterAbstract : + CSE_ALifeSchedulable
  │           │     │     │     ├── CSE_ALifeHumanAbstract : CSE_ALifeTraderAbstract + CSE_ALifeMonsterAbstract
  │           │     │     │     ├── CSE_ALifeMonsterBase / MonsterRat / MonsterZombie
  │           │     │     │     └── …
  │           │     │     ├── CSE_ALifeCreatureActor   (игрок, + CSE_ALifeTraderAbstract, CSE_PHSkeleton)
  │           │     │     └── CSE_ALifeCreatureCrow / Phantom
  │           │     ├── CSE_ALifeItem     : CSE_ALifeDynamicObjectVisual, CSE_ALifeInventoryItem
  │           │     │     ├── CSE_ALifeItemWeapon → …Magazined / ShotGun / AutoShotGun
  │           │     │     ├── CSE_ALifeItemArtefact / ItemAmmo / ItemDetector / ItemPDA / …
  │           │     ├── CSE_ALifeSpaceRestrictor : CSE_ALifeDynamicObject, CSE_Shape
  │           │     │     ├── CSE_ALifeCustomZone
  │           │     │     │     └── CSE_ALifeAnomalousZone → CSE_ALifeZoneVisual
  │           │     │     ├── CSE_ALifeSmartZone : + CSE_ALifeSchedulable
  │           │     │     └── CSE_ALifeTeamBaseZone / LevelChanger
  │           │     └── CSE_ALifeObjectPhysic / HangingLamp / Helicopter / Car / …
  │           └── CSE_ALifeOnlineOfflineGroup
  ├── CSE_ALifeGraphPoint
  └── CSE_Spectator / CSE_Temporary / CSE_AbstractVisual
```

> `CSE_ALifeTrader` — это `CSE_ALifeDynamicObjectVisual` + `CSE_ALifeTraderAbstract` (а не наследник `CSE_ALifeObject`). Игрок на сервере — `CSE_ALifeCreatureActor`, отдельного `CSE_Paragon` нет.

CSE — **сериализуемые** структуры, состояние которых передаётся по сети. Клиентский объект и серверная CSE связаны через `xrServerEntities` (мост): см. [Граница скриптинга](scripting-boundary.md) и [Сеть](../modules/network.md).

## Связи

- **`CObject` ↔ Renderer**: `CObject::Render()` → `xrRender` (через `xrAPI`).
- **`CGameObject` ↔ AI**: `m_ai_location` → `CAI_ObjectLocation` (ALIFE). См. [ALIFE](../modules/game/alife/index.md).
- **`CPhysicsShellHolder` ↔ Physics**: `CPhysicsShellHolder` → `CPhysicsShell` (PH). См. [Физика](../modules/game/physics.md).
- **`CGameObject` ↔ Lua**: `CScriptBinder` + `CUsableScriptObject` → `script_binder`. См. [Граница скриптинга](scripting-boundary.md).
- **`CGameObject` ↔ CSE**: синхронизация состояния через сеть. См. [Сеть](../modules/network.md).

## Ключевые свойства

- **Одна иерархия, два мира**: клиент — `CGameObject`, сервер — CSE. Синхронизация — через сеть.
- **`cast_*()` вместо `smart_cast`**: performance-решение, каждый подкласс знает свой тип.
- **Множественное наследование по всей цепочке**: `CGameObject` = `CObject` + `CUsableScriptObject` + `CScriptBinder`; `CEntity` = `CPhysicsShellHolder` + `CDamageManager`; `CInventoryItemObject` = `CInventoryItem` + `CPhysicItem`, и т.д. Каждый mix-in — независимая ответственность (рендер, физика, скриптинг, биндинг, иммунитет к хитам).
- **Ветвление под `CPhysicsShellHolder`**: живые существа (`CEntity → CEntityAlive`) и предметы (`CPhysicItem → CInventoryItemObject`) — разные ветки.
- **`ISheduled`**: все объекты — `ISheduled`, т.е. все участвуют в [цикле кадра](frame-loop.md).

## Связанные страницы

- [Цикл кадра](frame-loop.md) — как объекты исполняются.
- [Граница скриптинга](scripting-boundary.md) — как объекты экспортируются в Lua.
- [xrGame: Объекты](../modules/game/objects.md) — детальный разбор `CGameObject`/`CActor`.
- [ALIFE](../modules/game/alife/index.md) — как AI использует объект-модель.
