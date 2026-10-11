# xrGame

`xrGame` — **игровая логика** движка X-Ray (STALKER): всё, что стоит над ядром `xrEngine` и отвечает на вопрос «что происходит в мире игры». Ядро модуля — **объектная модель**: иерархия `CObject` (xrEngine) → `CGameObject` → `CEntity` → `CEntityAlive` → `CActor` / `CCustomMonster`, плюс ветки инвентаря (`CInventoryItemObject`) и зон (`CSpaceRestrictor` → `CCustomZone`), и Lua-обёртка `CScriptGameObject`, через которую скрипты видят все объекты мира.

**Чего НЕ делает**: не владеет рендером/физикой/звуком/частицами (это [xrEngine](../xr-engine/index.md), [xrRender](../renderer/index.md), [xrSound](../xr-sound/index.md), [xrParticles](../xr-particles/index.md)); не реализует AI-планирование (внутренности `ai/` — [ALIFE](alife.md), итерация 6); не реализует сетевой транспорт (сигнатуры `net_*` и `Level_*` — итерация 12).

См. [Карта модулей](../../architecture/module-map.md), [Модель объектов](../../architecture/object-model.md) (обновляется в порц. 2 и 30), [Граф связей](../../architecture/dependencies.md).

## Структура (итерация 5: объект-модель)

~45 классов/файлов в `src/xrGame/` (Actor, CGameObject, CEntity*, CInventoryItemObject, CCustomZone и др.) + `CScriptGameObject` (~1300 строк). Остальные ~650 файлов `xrGame` (AI, физика, оружие, UI, уровни) — итерации 6–14.

| # | Порцион | Страницы | Статус |
|---|---------|----------|--------|
| 1 | Фундамент: `CObject` (xrEngine) | `index.md`, [xr-object.md](../xr-engine/xr-object.md) | ✅ |
| 2 | Фабрика и регистр классов | [class-registry.md](class-registry.md) — `CObjectFactory`, `NEW_INSTANCE`-вход, CLSID'ы, `game.ltx` | ✅ |
| 3 | `CGameObject`: каркас | [game-object.md](game-object.md) — класс, down-cast'ы, жизненный цикл, AI-локации, аним. контроллер, attachments, visual-callback'и, Lua-связь, CrPr, эффекторы | ✅ |
| 4 | `CGameObject`: сеть и жизненный цикл | [game-object-net.md](game-object-net.md) — `net_Spawn`/`net_Destroy`, `OnEvent`, сериализация, `spawn_supplies()`, путь удаления | ✅ |
| 5 | `CEntity`: здоровье и урон | [entity.md](entity.md) | ⬜ |
| 6 | `EntityCondition`: статы | [entity-condition.md](entity-condition.md) | ⬜ |
| 7 | `CEntityAlive`: живое существо | [entity-alive.md](entity-alive.md) | ⬜ |
| 8 | Травмы, раны, смерть | [wounds-death.md](wounds-death.md) | ⬜ |
| 9 | `CCustomMonster`: мост в AI | [custom-monster.md](custom-monster.md) | ⬜ |
| 10 | `CInventoryItemObject` | [inventory-item.md](inventory-item.md) | ⬜ |
| 11 | Torch/Flashlight/CustomDevice/Detectors | [misc-items.md](misc-items.md) | ⬜ |
| 12 | PDA/Dosimeter/Antirad/Medkit | [medical-items.md](medical-items.md) | ⬜ |
| 13 | Lua-обёртка `CScriptGameObject` | [script-game-object.md](script-game-object.md) | ⬜ |
| 14 | `CActor`: каркас и условия | [actor.md](actor.md) | ⬜ |
| 15 | `CActor`: камеры и эффекторы | [actor-camera.md](actor-camera.md) | ⬜ |
| 16 | `CActor`: движение | [actor-movement.md](actor-movement.md) | ⬜ |
| 17 | `CActor`: input и feel | [actor-input.md](actor-input.md) | ⬜ |
| 18 | `CActor`: оружие-обвязка | [actor-weapon.md](actor-weapon.md) | ⬜ |
| 19 | `CActor`: сеть и интерполяция | [actor-net.md](actor-net.md) | ⬜ |
| 20 | `CActor`: анимация | [actor-animation.md](actor-animation.md) | ⬜ |
| 21 | `CActor`: HUD и рендер | [actor-hud.md](actor-hud.md) | ⬜ |
| 22 | `CActor`: сон, смерть, прочее | [actor-misc.md](actor-misc.md) | ⬜ |
| 23 | `CActor`: MP-обвязка | [actor-mp.md](actor-mp.md) | ⬜ |
| 24 | Зоны: `CSpaceRestrictor` | [zones-restrictor.md](zones-restrictor.md) | ⬜ |
| 25 | Зоны: `CCustomZone` FSM | [zones-custom.md](zones-custom.md) | ⬜ |
| 26 | Зоны: аномалии и smart-террейн | [zones-anomaly.md](zones-anomaly.md) | ⬜ |
| 27 | Feel: Touch/Vision/Sound | [feel.md](feel.md) | ⬜ |
| 28 | Script: script_entity + script_hit | [script-entity.md](script-entity.md) | ⬜ |
| 29 | Script: attachments и binder | [script-attachment.md](script-attachment.md) | ⬜ |
| 30 | Связность: object-model + зависимости | update [object-model.md](../../architecture/object-model.md) + [dependencies.md](../../architecture/dependencies.md) + `index.md` | ⬜ |

Остальные итерации: [Иерархия](objects.md) (переходная заглушка), [ALIFE](alife.md) (ит. 6), [Физика](physics.md) (ит. 7), [Оружие](weapons.md) (ит. 8), [Инвентарь](inventory.md) (ит. 8), [Зоны и эффекты](zones.md) (ит. 9), [UI](ui.md) (ит. 10).

## Карта связей

```mermaid
graph TD
    subgraph xrEngine
        CObject[CObject]
        ObjL[CObjectList]
    end
    subgraph xrGame
        GO[CGameObject]
        Ent[CEntity]
        Cond[EntityCondition]
        EA[CEntityAlive]
        Actor[CActor]
        Mon[CCustomMonster]
        Item[CInventoryItemObject]
        SR[CSpaceRestrictor]
        CZ[CCustomZone]
        SGO[CScriptGameObject Lua-обёртка]
        AI[ai/ — итерация 6]
    end
    subgraph Lua
        LAPI[Lua API — итерация 14]
    end

    CObject -->|H_SetParent / UpdateCL / net_Spawn| GO
    GO --> Ent
    GO --> Item
    GO --> SR
    Ent --> Cond
    Ent --> EA
    EA --> Actor
    EA --> Mon
    SR --> CZ
    Mon -.Think / интерфейс.-> AI
    SGO -.обёртка.-> GO
    LAPI -.полный API — ит. 14.-> SGO
```

- **`CObject`** (xrEngine) — каркас: пространство, crow, родитель, сетевой ID — [xr-object](../xr-engine/xr-object.md).
- **`CGameObject`** — «настоящий» игровой объект (C++-ядро + Lua-обёртка); исторический предок `CGO` (legacy) упоминается в порц. 2 и 30, отдельно не документируется.
- **`CEntity`/`CEntityAlive`** — здоровье, урон, статы, живые существа (порц. 5–8).
- **`CActor`** — игрок: 10 страниц (порц. 14–23).
- **`CCustomMonster`** — мост в AI: только класс, `Think()`, интерфейс (порц. 9); внутренности `ai/` — итерация 6.
- **`CScriptGameObject`** — обёртка для Lua: механика — порц. 13, полный API — итерация 14.
- **Зоны** — `CSpaceRestrictor`/`CCustomZone` (порц. 24–26); **вещи** — `CInventoryItemObject` (порц. 10–12, оружие — итерация 8).
