# План документирования — Итерация 5: xrGame: Actor / GameObject / Entity — объект-модель

> Рабочий документ-план. Удаляется после завершения итерации 5 (лог — в §9 GENERATE-DOCS.md).

## Масштаб

Итерация 5 — «объектная модель» xrGame: всё, из чего состоит мир в игре и кто есть кто. Ядро:

- `CObject` (xrEngine) → `CGameObject` (C++-ядро + Lua-обёртка)
- `CGameObject` → `CEntity` → `CEntityAlive` → `CActor` (игрок) и `CCustomMonster` (мост в AI)
- `CGameObject` → `CInventoryItemObject` (вещи инвентаря; оружие → итерация 8)
- `CGameObject` → `CSpaceRestrictor` → `CCustomZone` (зоны)
- Lua-обёртка `CScriptGameObject` (скрипты видят все объекты мира)

Покрытие: ~45 классов/файлов в `xrGame/` (Actor, CGameObject, CEntity*, CInventoryItemObject, CCustomZone и др.) + `CScriptGameObject` (~1300 строк). Остальные ~650 файлов xrGame (AI, физика, оружие, UI, уровни) — итерации 6–14.

## Решения (фиксация)

1. **`CGO` (legacy) не документируется отдельно** — упомянуть в порционах 2 и 30 как исторический предок `CGameObject`.
2. **AI-внутренности (`ai/` папка) — итерация 6.** Порцион 9 (`CCustomMonster`) — только как мост: класс, `Think()`, интерфейс для AI.
3. **`CWeapon` и наследники — итерация 8.** Порцион 10 лишь отмечает место в дереве.
4. **`Level_*` / сетевой код — итерация 12.** В порционах 4/10/19 — только сигнатуры `save/load/net_*` как «здесь будет сеть».
5. **Полный Lua API `CScriptGameObject` — итерация 14.** Порцион 13 — только механика обёртки.
6. **`architecture/object-model.md` (ит. 0) ОБНОВЛЯЕТСЯ** после порциона 2 и порциона 30.

## Порционы

| #   | Порцион                                 | Страницы                                                               |
| --- | --------------------------------------- | ---------------------------------------------------------------------- |
| 1   | Фундамент: CObject (xrEngine)           | `index.md`, `xr-object.md`                                             |
| 2   | Фабрика и регистр классов               | `class-registry.md`                                                    |
| 3   | CGameObject: каркас                     | `game-object.md`                                                       |
| 4   | CGameObject: сеть и жизненный цикл      | `game-object-net.md`                                                   |
| 5   | CEntity: здоровье и урон                | `entity.md`                                                            |
| 6   | EntityCondition: статы                  | `entity-condition.md`                                                  |
| 7   | CEntityAlive: живое существо            | `entity-alive.md`                                                      |
| 8   | Травмы, раны, смерть                    | `wounds-death.md`                                                      |
| 9   | CCustomMonster: мост в AI               | `custom-monster.md`                                                    |
| 10  | CInventoryItemObject                    | `inventory-item.md`                                                    |
| 11  | Torch/Flashlight/CustomDevice/Detectors | `misc-items.md`                                                        |
| 12  | PDA/Dosimeter/Antirad/Medkit            | `medical-items.md`                                                     |
| 13  | Lua-обёртка CScriptGameObject           | `script-game-object.md`                                                |
| 14  | CActor: каркас и условия                | `actor.md`                                                             |
| 15  | CActor: камеры и эффекторы              | `actor-camera.md`                                                      |
| 16  | CActor: движение                        | `actor-movement.md`                                                    |
| 17  | CActor: input и feel                    | `actor-input.md`                                                       |
| 18  | CActor: оружие-обвязка                  | `actor-weapon.md`                                                      |
| 19  | CActor: сеть и интерполяция             | `actor-net.md`                                                         |
| 20  | CActor: анимация                        | `actor-animation.md`                                                   |
| 21  | CActor: HUD и рендер                    | `actor-hud.md`                                                         |
| 22  | CActor: сон, смерть, прочее             | `actor-misc.md`                                                        |
| 23  | CActor: MP-обвязка                      | `actor-mp.md`                                                          |
| 24  | Зоны: CSpaceRestrictor                  | `zones-restrictor.md`                                                  |
| 25  | Зоны: CCustomZone FSM                   | `zones-custom.md`                                                      |
| 26  | Зоны: аномалии и smart-террейн          | `zones-anomaly.md`                                                     |
| 27  | Feel: Touch/Vision/Sound                | `feel.md`                                                              |
| 28  | Script: script_entity + script_hit      | `script-entity.md`                                                     |
| 29  | Script: attachments и binder            | `script-attachment.md`                                                 |
| 30  | Связность: object-model + зависимости   | update `architecture/object-model.md` + `dependencies.md` + `index.md` |

## Статус

| #   | Статус |
| --- | ------ |
| 1   | ✅     |
| 2   | ✅     |
| 3   | ✅     |
| 4   | ✅     |
| 5   | ✅     |
| 6   | ⬜     |
| 7   | ✅     |
| 8   | ✅     |
| 9   | ✅     |
| 10  | ✅     |
| 11  | ✅     |
| 12  | ⬜     |
| 13  | ⬜     |
| 14  | ⬜     |
| 15  | ⬜     |
| 16  | ⬜     |
| 17  | ⬜     |
| 18  | ⬜     |
| 19  | ⬜     |
| 20  | ⬜     |
| 21  | ⬜     |
| 22  | ⬜     |
| 23  | ⬜     |
| 24  | ⬜     |
| 25  | ⬜     |
| 26  | ⬜     |
| 27  | ⬜     |
| 28  | ⬜     |
| 29  | ⬜     |
| 30  | ⬜     |

## Обещанное из итераций 0–4

- **`architecture/object-model.md`** (ит. 0) — обновляется в порционах 2, 30
- **`CObject`/`CGO`/`CSE_Abstract`** — порционы 1–2
- **`IKinematics`/`CBoneInstance`/`CVisual`** (ит. 3 порц. 8) — только использование из `CGameObject`/`CEntityAlive`
- **`CSE_ALife*`/`CSE_Object`** (ит. 11) — только `CSE_Abstract*` как аргумент `net_Spawn`
- **`CParticlesObject`** (ит. 4) — только как «соседний объект» в зонах/ранах
- **`CWeapon`-иерархия** (ит. 8) — только место в дереве (порцион 10)
- **AI-внутренности** (ит. 6) — только интерфейс `CCustomMonster` (порцион 9)
- **`CScriptGameObject` API** (ит. 14) — только механика обёртки (порцион 13)
- **`Level_*`/сетевой код** (ит. 12) — только сигнатуры (порционы 4/10/19)

## После итерации 5

- `docs/docs/modules/game/` — ~20 страниц (index, xr-object, game-object, entity*, actor*×10, zones*, items*, script*, feel)
- `docs/docs/architecture/object-model.md` — обновлена с реальной иерархией
- Лог итерации 5 → `GENERATE-DOCS.md` §9
- Этот файл удаляется
