# xrGame

`xrGame` — **игровая логика** движка X-Ray (STALKER): всё, что стоит над ядром `xrEngine` и отвечает на вопрос «что происходит в мире игры». Ядро модуля — **объектная модель**: иерархия `CObject` (xrEngine) → `CGameObject` → `CPhysicsShellHolder` → `CEntity` → `CEntityAlive` → `CActor` / `CCustomMonster`, плюс ветки инвентаря (`CPhysicItem` → `CInventoryItemObject`), зон (`CSpaceRestrictor` → `CCustomZone`) и Lua-обёртка `CScriptGameObject`, через которую скрипты видят все объекты мира.

**Чего НЕ делает**: не владеет рендером/физикой/звуком/частицами (это [xrEngine](../xr-engine/index.md), [xrRender](../renderer/index.md), [xrSound](../xr-sound/index.md), [xrParticles](../xr-particles/index.md)); не реализует AI-планирование (внутренности `ai/` — [ALIFE](alife/index.md), итерация 6); не реализует сетевой транспорт (сигнатуры `net_*` и `Level_*` — итерация 12).

См. [Карта модулей](../../architecture/module-map.md), [Модель объектов](../../architecture/object-model.md) (обновлена в порц. 2 и 48), [Граф связей](../../architecture/dependencies.md).

## Структура (итерация 5: объект-модель)

~45 классов/файлов в `src/xrGame/` (Actor, CGameObject, CEntity*, CInventoryItemObject, CCustomZone и др.) + `CScriptGameObject` (~1300 строк). Остальные ~650 файлов `xrGame` (AI, физика, оружие, UI, уровни) — итерации 6–14.

| #   | Порцион                                              | Страницы                                                        | Статус |
| --- | ---------------------------------------------------- | --------------------------------------------------------------- | ------ |
| 1   | Фундамент: `CObject` (xrEngine)                      | `index.md`, [xr-object.md](../xr-engine/xr-object.md)           | ✅     |
| 2   | Фабрика и регистр классов                            | [class-registry.md](class-registry.md)                          | ✅     |
| 3   | `CGameObject`: каркас                                | [game-object.md](game-object.md)                                | ✅     |
| 4   | `CGameObject`: сеть и жизненный цикл                 | [game-object-net.md](game-object-net.md)                        | ✅     |
| 5   | `CEntity`: здоровье и урон                           | [entity.md](entity.md)                                          | ✅     |
| 6   | `CEntityCondition`: статы                            | [entity-condition.md](entity-condition.md)                      | ✅     |
| 7   | `CEntityAlive`: живое существо                       | [entity-alive.md](entity-alive.md)                              | ✅     |
| 8   | Травмы, раны, смерть                                 | [wounds-death.md](wounds-death.md)                              | ✅     |
| 9   | `CCustomMonster`: мост в AI                          | [custom-monster.md](custom-monster.md)                          | ✅     |
| 10  | `CInventoryItemObject`                               | [inventory-item.md](inventory-item.md)                          | ✅     |
| 11  | Torch/Flashlight/CustomDevice/Detectors              | [misc-items.md](misc-items.md)                                  | ✅     |
| 12  | PDA/Dosimeter/Antirad/Medkit                         | [medical-items.md](medical-items.md)                            | ✅     |
| 13  | `CScriptGameObject`: механика обёртки                | [script-game-object.md](script-game-object.md)                  | ✅     |
| 14  | `CScriptGameObject`: базовый API                     | [script-game-object.md](script-game-object.md)                  | ✅     |
| 15  | `CScriptGameObject`: инвентарь и предметы            | [script-game-object.md](script-game-object.md)                  | ✅     |
| 16  | `CScriptGameObject`: оружие, holder, снайпинг        | [script-game-object.md](script-game-object.md)                  | ✅     |
| 17  | `CScriptGameObject`: мир, эффекты, звук              | [script-game-object.md](script-game-object.md)                  | ✅     |
| 18  | `CScriptGameObject`: Lua-регистрация (luabind)       | [script-game-object.md](script-game-object.md)                  | ✅     |
| 19  | `CActor`: каркас                                     | [actor.md](actor.md)                                            | ✅     |
| 20  | `CActor`: условия и статы (`CActorCondition`)        | [actor.md](actor.md)                                            | ✅     |
| 21  | `CActor`: Hit/PHHit и урон                           | [actor.md](actor.md)                                            | ✅     |
| 22  | `CActor`: смерть                                     | [actor-death.md](actor-death.md)                                | ✅     |
| 23  | `CActor`: движение и физика                          | [actor-movement.md](actor-movement.md)                          | ✅     |
| 24  | `CActor`: input                                      | [actor-input.md](actor-input.md)                                | ✅     |
| 25  | `CActor`: feel                                       | [actor-input.md](actor-input.md)                                | ✅     |
| 26  | `CActor`: holder (транспорт)                         | [actor-weapon.md](actor-weapon.md)                              | ✅     |
| 27  | `CActor`: оружие-обвязка                             | [actor-weapon.md](actor-weapon.md)                              | ✅     |
| 28  | `CActor`: сеть и интерполяция                        | [actor-net.md](actor-net.md)                                    | ✅     |
| 29  | `CActorMP`: клиент, state, импорт/экспорт            | [actor-mp.md](actor-mp.md)                                      | ✅     |
| 30  | `CActorMP`: сервер и RPC                             | [actor-mp.md](actor-mp.md)                                      | ✅     |
| 31  | `CActor`: анимация                                   | [actor-animation.md](actor-animation.md)                        | ✅     |
| 32  | `CActor`: HUD и рендер                               | [actor-hud.md](actor-hud.md)                                    | ✅     |
| 33  | `CActor`: инвентарь-события, артефакты на поясе      | [actor-misc.md](actor-misc.md)                                  | ✅     |
| 34  | `CActor`: сон, ночное зрение, прочее                 | [actor-misc.md](actor-misc.md)                                  | ✅     |
| 35  | Зоны: `CSpaceRestrictor` + space_restriction         | [zones-restrictor.md](zones-restrictor.md)                      | ✅     |
| 36  | Зоны: composition/manager/abstract + обвязка         | [zones-restrictor.md](zones-restrictor.md)                      | ✅     |
| 37  | Зоны: `CCustomZone` FSM и частицы                    | [zones-custom.md](zones-custom.md)                              | ✅     |
| 38  | Зоны: свет, ветер, grass, аномалии-эффекторы         | [zones-custom.md](zones-custom.md)                              | ✅     |
| 39  | Зоны: подклассы (MosquitoBald, Campfire, …)          | [zones-subclasses-2.md](zones-subclasses-2.md)                  | ✅     |
| 40  | Зоны: аномалии и smart-террейн                       | [zones-anomaly.md](zones-anomaly.md)                            | ✅     |
| 41  | Feel: Touch (xrEngine + GlobalFeelTouch)             | [feel.md](feel.md)                                              | ✅     |
| 42  | Feel: Vision и Sound                                 | [feel.md](feel.md)                                              | ✅     |
| 43  | Script: `CScriptEntity` (control, actions)           | [script-entity.md](script-entity.md)                            | ✅     |
| 44  | Script: `CScriptHit` + monster hit info + действия   | [script-entity.md](script-entity.md)                            | ✅     |
| 45  | Script: attachments                                  | [script-attachment.md](script-attachment.md)                    | ✅     |
| 46  | Script: `CScriptBinder`                              | [script-attachment.md](script-attachment.md)                    | ✅     |
| 47  | Script: `CScriptObject` и прочее (actions/sound/…)   | [script-object-api-2.md](script-object-api-2.md)                | ✅     |
| 48  | Связность: object-model + зависимости + `index.md`   | [object-model.md](../../architecture/object-model.md), [dependencies.md](../../architecture/dependencies.md) | ✅     |

Остальные итерации: [Иерархия](objects.md) (переходная заглушка), [ALIFE](alife/index.md) (ит. 6, **завершена**), [Физика](physics.md) (ит. 7), [Оружие](weapons.md) (ит. 8), [Инвентарь](inventory.md) (ит. 8), [Зоны и эффекты](zones.md) (ит. 9), [UI](ui.md) (ит. 10). Сетевой код (`Level_*`, `net_*` детали) — итерация 12; полный Lua API — итерация 14.

## Карта связей

```mermaid
graph TD
    subgraph xrEngine
        CObject[CObject]
        ObjL[CObjectList]
    end
    subgraph xrGame
        GO[CGameObject]
        PSH[CPhysicsShellHolder]
        Ent[CEntity]
        Cond[CEntityCondition]
        EA[CEntityAlive]
        Actor[CActor]
    Mon[CCustomMonster]
    BM[CBaseMonster]
    M15[15 видов монстров]
    ST[CAI_Stalker]
    RAT[CAI_Rat]
    TRD[CAI_Trader]
    CRW[CAI_Crow]
    PH[CPhantom]
    PI[CPhysicItem]
    Item[CInventoryItemObject]
    SR[CSpaceRestrictor]
    CZ[CCustomZone]
    SGO[CScriptGameObject Lua-обёртка]
    AI[ai/ — итерация 6 ✅]
    ALIFE[CALifeSimulator + CSE_ALife*]
    end
    subgraph Lua
        LAPI[Lua API — итерация 14]
    end

    CObject -->|H_SetParent / UpdateCL / net_Spawn| GO
    GO --> PSH
    PSH --> Ent
    PSH --> PI
    Ent --> Cond
    Ent --> EA
    EA --> Actor
    EA --> Mon
    PI --> Item
    GO --> SR
    SR --> CZ
    Mon --> BM
    BM --> M15
    Mon --> ST
    Mon --> RAT
    EA --> TRD
    Ent --> CRW
    Ent --> PH
    Mon -.Think / FSM / planner.-> AI
    AI --> ALIFE
    SGO -.обёртка.-> GO
    LAPI -.полный API — ит. 14.-> SGO
    ```

- **`CObject`** (xrEngine) — каркас: пространство, crow, родитель, сетевой ID — [xr-object](../xr-engine/xr-object.md).
- **`CGameObject`** — «настоящий» игровой объект (C++-ядро + Lua-обёртка); исторический предок `CGO` (legacy) упоминается в порц. 2 и 48, отдельно не документируется.
- **`CEntity`/`CEntityAlive`** — здоровье, урон, статы, живые существа (порц. 5–8). Ветка идёт через `CPhysicsShellHolder` (порц. 3).
- **`CActor`** — игрок: 10 страниц (порц. 19–34).
- **`CCustomMonster`** — мост в AI: только класс, `Think()`, интерфейс (порц. 9); внутренности `ai/` — итерация 6 (**завершена**, [alife/index.md](alife/index.md)).
- **`CScriptGameObject`** — обёртка для Lua: механика — порц. 13, полный API — итерация 14.
- **Зоны** — `CSpaceRestrictor`/`CCustomZone` (порц. 35–40); **вещи** — `CInventoryItemObject` (порц. 10–12, оружие — итерация 8).
