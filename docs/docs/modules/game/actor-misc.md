# CActor: инвентарь, артефакты на поясе, сон/НВ/прочее (порционы 33–34)

> **Модуль:** `xrGame` · **Файлы:** `Actor.cpp` (2611–2668 инвентарь-события, 2670–2790 артефакты на поясе, 2857–2873 `can_attach`, 2896–2916 физ.-геттеры, 2916–3032 `GetRestoreSpeed`, 3049–3110 `SwitchNightVision`, 2094–2153 visibility), `Actor.h` (225–227, 760, 820, 862–879), `Actor_Sleep.cpp` (1), `ActorNightVision.cpp` (89), `ActorHelmet.cpp` (344), `ActorBackpack.cpp` (102), `Actor_Events.cpp` (378), `actor_defs.h` (154) · **Порционы 33–34, итерация 5**
>
> **Страница:** реакция актёра на перемещения предметов, постоянный апдейт эффектов артефактов/экипировки на поясе, защита хитами, прикрепление навесок, `OnDifficultyChanged` и физ.-геттеры (33); мёртвый код сна, ночное зрение, `CHelmet`/`CBackpack`, `OnEvent`-обработчики (34). Связи: `CInventoryOwner`-мост (порц. 10), `CInventory` (ит. 8), `CArtefact`/`CCustomOutfit`/`CHelmet`/`CBackpack` (ит. 10), `CActorCondition::Change*`/`LoadImmunities` (порц. 20), Hit-цепочка (порц. 21), Lua `_G.CActor__HitArtefactsOnBelt`/`_G.CInventory__eat` (порц. 18/21), `shedule_Update` (порц. 19), holder/weapon/зум (порц. 26–27), `GE_*`-каркас (порц. 4), `actor_statistic_defs.h`/`actor_communication.cpp` (порц. 30).

## 1. Ответственность

Мост между инвентарём и состоянием актёра: переопределения событий `CInventoryOwner` (снятие костюма/оружия, авто-гренада, пере-прикрепление навесок), ежесекундное применение свойств артефактов пояса и экипировки к `CActorCondition` (health/power/satiety/bleeding/radiation), вычитание иммунитетов артефактов при хитах, агрегированная защита для UI, правила прикрепления предметов-навесок.

## 2. Инвентарь-события (`Actor.cpp:2611–2668`)

| Метод | Строки | Логика |
|---|---|---|
| `OnItemTake(inventory_item)` | 2611–2615 | `CInventoryOwner::OnItemTake` → `if (OnClient()) return` (дальнейшей обработки нет — каркас под MP-ветки) |
| `OnItemDrop(item, just_before_destroy)` | 2617–2644 | `inherited`; **`CCustomOutfit`** из слота (`m_ItemCurrPlace.type == eItemPlaceSlot`) → `outfit->ApplySkinModel(this, false, false)` (снятие скина модели); **`CWeapon`** из слота → `OnZoomOut()`, при `GetRememberActorNVisnStatus()` → `EnableActorNVisnAfterZoom()` (восстановление NV после зума, §11); **граната**: если `!just_before_destroy`, `BaseSlot() == GRENADE_SLOT` и `ItemFromSlot(GRENADE_SLOT) == NULL` → `inventory().SameSlot(GRENADE_SLOT, item, true)` → `inventory().Slot(GRENADE_SLOT, grenade, true, true)` (автоподбор гранаты обратно в слот) |
| `OnItemDropUpdate()` | 2647–2657 | `inherited`; перебор `inventory().m_all`: `if (!(*I)->IsInvalid() && !attached(*I)) attach(*I)` — принудительное возвращение валидных навесок (вызывается при обновлении после дропа, `CInventoryItemObject` → порц. 10) |
| `OnItemRuck(item, previous_place)` | 2660–2663 | только `CInventoryOwner::OnItemRuck` (чистый мост) |
| `OnItemBelt(item, previous_place)` | 2665–2668 | только `CInventoryOwner::OnItemBelt` (чистый мост) |

## 3. `UpdateArtefactsOnBeltAndOutfit()` (`Actor.cpp:2672–2742`)

`virtual` (`Actor.h:225`). Вызывается из **`shedule_Update`** (`Actor.cpp:2089`, порц. 19). `#define ARTEFACTS_UPDATE_TIME 0.100f` (2670).

1. **Таймер** (2674–2687): static-накопитель `conditions().fdelta_time()`; до 0.1 с — `return`; иначе прошедшее время уходит в `f_update_time` и обнуляется.
2. **Артефакты пояса** (2689–2709): для каждого `CArtefact` из `inventory().m_belt` — `art_cond = GetCondition()`:
   - `ChangeBleeding/Health/Power/Satiety((m_f*RestoreSpeed * art_cond) * f_update_time)`;
   - **радиация** (2700–2707): при положительном `m_fRadiationRestoreSpeed` (артефакт-источник) вычитается `GetBoostRadiationImmunity()` с `clamp(val, 0, val)` (иммунитет гасит радиацию, но не лечит); при отрицательном (абсорбирующий) — применяется напрямую.
3. **Экипировка** (2710–2729): `CCustomOutfit* outfit = GetOutfit()` и `CHelmet` из `HELMET_SLOT`; при наличии любого — те же 5 `Change*` от **суммы** свойств костюма и шлема (отсутствующий → 0.f).
4. Закомментированный фрагмент (2730–2741): выключение NV факела, если нет шлема (неактивно).

## 4. Защита артефактов от хитов

### `HitArtefactsOnBelt(hit_power, hit_type)` (`Actor.cpp:2744–2774`)

1. **Lua-оверрайд** (2746–2759): функтор `_G.CActor__HitArtefactsOnBelt` получает таблицу `{override=false, hit_power}` и аргументы `(table, hit_power, hit_type)`; если результат — таблица с `override=true` → возврат `output["hit_power"]` (полная передача формулы скрипту).
2. Иначе — по всем `CArtefact` пояса: `hit_power -= artefact->m_ArtefactHitImmunities.AffectHit(1.0f, hit_type)` (2761–2770).
3. `clamp(hit_power, 0, flt_max)` → возврат.

**Вызовы** — в `CActor::Hit` (`Actor.cpp:520–764`): ветка `IsGameTypeSingle() && !GodMode()` (`693`) — после Lua-хука `_G.CActor__BeforeHitCallback` и `ApplyScriptHit`, до `inherited::Hit`; ветка non-single (`752`) — вместо 100000.0f в случае back-stab; закомментированная копия (709) — порц. 21.

### `GetProtection_ArtefactsOnBelt(hit_type)` (`Actor.cpp:2776–2790`)

Агрегированная защита: `sum += AffectHit(1.0f, hit_type) * GetCondition()` по всем артефактам пояса. **Вызовы (UI):** `UIActorStateInfo.cpp:195–257` (burn/radiation/chemical/telepatic/shock + сводная строка), `UIHudStatesWnd.cpp:704/854`.

## 5. `can_attach(const CInventoryItem*)` (`Actor.cpp:2857–2873`)

`const`-проверка перед прикреплением навески:

1. `smart_cast<const CAttachableItem*>` — не приводится или `!can_be_attached()` → `false`;
2. секция предмета не в белом списке `m_attach_item_sections` (`std::find`) → `false` (комментарий: «можно ли присоединять объекты такого типа»);
3. уже прикреплён такой же (`attached(cNameSect())`) → `false` (комментарий: «если уже есть присоединенный объект такого типа»);
4. иначе `true`.

## 6. Физ.-геттеры (`Actor.cpp:2896–2914`)

| Метод | Строки | Логика |
|---|---|---|
| `GetMass()` | 2896–2903 | жив → `character_physics_support()->movement()->GetMass()`; мёртв и есть `m_pPhysicsShell` → `shell->getMass()`; иначе `0` |
| `is_on_ground()` | 2905–2908 | `movement()->Environment() != CPHMovementControl::peInAir` |
| `is_ai_obstacle()` | 2911–2914 | всегда `false` (закомментирован `true`) — актёр не является AI-препятствием для обхода |

## 7. `GetRestoreSpeed(type)` (`Actor.cpp:2916–3032`)

> Примечание: переопределение `CInventoryOwner` — **не** `CInventory`/инвентарь-события, а подсчёт скоростей восстановления (UI).

Сводная скорость восстановления для UI (`Actor.h:820`; вызовы `UIActorStateInfo.cpp:242`). По `ALife::EConditionRestoreType`:

| Тип | База | Артефакты пояса | Экипировка |
|---|---|---|---|
| `eHealthRestoreSpeed` | `conditions().change_v().m_fV_HealthRestore` + `V_SatietyHealth() ± GetSatiety()` | `+ m_fHealthRestoreSpeed * condition` | `+ outfit->m_fHealthRestoreSpeed` |
| `eRadiationRestoreSpeed` | — | `+ m_fRadiationRestoreSpeed * condition` | `+ outfit->...` |
| `eSatietyRestoreSpeed` | `conditions().V_Satiety()` | `+ m_fSatietyRestoreSpeed * condition` | `+ outfit->...` |
| `ePowerRestoreSpeed` | `conditions().GetSatietyPower()` | `+ m_fPowerRestoreSpeed * condition` | `+ outfit->m_fPowerRestoreSpeed`, затем `VERIFY(m_fPowerLoss != 0)` и `/ m_fPowerLoss` (без костюма — `/0.5f`) |
| `eBleedingRestoreSpeed` | `change_v().m_fV_WoundIncarnation` | `+ m_fBleedingRestoreSpeed * condition` | `+ outfit->...` |

(Артефактная часть идентична §3, но без таймера — для показа мгновенных скоростей. Рюкзака в формуле нет — только `m_belt` + `OUTFIT_SLOT`.)

## 8. `OnDifficultyChanged()` (`Actor.cpp:2875–2889`)

Смена сложности игры (вызывается при загрузке/смене difficulty):

1. **Иммунитеты** (2878–2882): `VERIFY(g_SingleGameDifficulty в [egdNovice..egdMaster])`; имя секции `"actor_immunities_" + token` → `conditions().LoadImmunities(tmp, pSettings)` (порц. 20 — `CActorCondition::LoadImmunities`);
2. **Вероятность попадания** (2884–2885): `"hit_probability_" + token` → `m_hit_probability = r_float(cNameSect(), tmp)`;
3. **Двойное попадание** (2887–2888): `"actor_thd_" + token` → `conditions().LoadTwoHitsDeathParams(tmp)`.

## 9. Связи (порц. 33)

- **Ит. 8/10** — `CInventory::m_belt/m_all`, `SameSlot/Slot/BaseSlot`, `CInventoryOwner::attach/attached`, `CAttachableItem`, `CCustomOutfit::ApplySkinModel`, `CWeapon`-зум/NV, `CArtefact` (`m_f*RestoreSpeed`, `m_ArtefactHitImmunities`, `GetCondition`), `CHelmet`.
- **Порц. 20 (`CActorCondition`)** — все `Change*`/`GetBoostRadiationImmunity`/`fdelta_time`/`change_v()`; `LoadImmunities`/`LoadTwoHitsDeathParams` — вызовы из `OnDifficultyChanged` (§8).
- **Порц. 21 (Hit)** — `HitArtefactsOnBelt` вызывается в `CActor::Hit` (693/752, контекст — §4); Lua `_G.CActor__HitArtefactsOnBelt`.
- **Порц. 19 (`shedule_Update`)** — вызов `UpdateArtefactsOnBeltAndOutfit()` (2089).
- **UI** — `UIActorStateInfo`/`UIHudStatesWnd` — `GetProtection_ArtefactsOnBelt`, `GetRestoreSpeed`.
- **Ит. 7** — `CCharacterPhysicsSupport::movement` (`GetMass`, `Environment`), `CPhysicsShell::getMass` (§6).
- **Порц. 27/19** — из смежного диапазона: `SetZoomRndSeed`/`SetShotRndSeed`/`GetMissileOffset`/`SetMissileOffset` (2792–2812), `spawn_supplies`-заглушка (2814–2818) — задокументированы в порц. 19/27.
- **Ит. 10** — `CBackpack` (свойства/сеть, §14), `CAttachableItem::can_be_attached`, `CInventoryItem`-мосты.
- **Ит. 6/20** — `GetCarryWeightBoost` (`CActorCondition`), `MaxWalkWeight` (свойства), `SetMaxWalkWeight` (Lua, порц. 18).

## 10. Краткое резюме (порц. 33)

Инвентарь-события актёра — тонкие переопределения `CInventoryOwner`: при дропе снимается скин костюма, выключается зум оружия (с восстановлением NV) и автоподбирается граната в пустой слот; `OnItemDropUpdate` принудительно возвращает валидные навески. Главная функция — `UpdateArtefactsOnBeltAndOutfit` (таймер 0.1 с из `shedule_Update`): свойства артефактов пояса × `GetCondition` + сумма свойств костюма/шлема применяются к пяти параметрам `CActorCondition`, радиация артефакта-источника гасится `BoostRadiationImmunity`. `HitArtefactsOnBelt` сначала даёт полный Lua-оверрайд (`_G.CActor__HitArtefactsOnBelt`), затем вычитает `AffectHit`-иммунитеты пояса из силы хита (вызов из `CActor::Hit` — §4/порц. 21); `GetProtection_ArtefactsOnBelt`/`GetRestoreSpeed` дают агрегированные значения для UI (без рюкзака). `can_attach` — три проверки (тип/whitelist-секция/дубликат). `OnDifficultyChanged` — иммунитеты/вероятность/двойное попадание по секциям `actor_immunities_*`/`hit_probability_*`/`actor_thd_*`. Физ.-геттеры: `GetMass` (движение/оболочка/0), `is_on_ground`, `is_ai_obstacle = false`.

---

# Порцион 34: сон, ночное зрение, CHelmet/CBackpack, OnEvent

## 11. Сон — мёртвый код

- **`Actor_Sleep.cpp`** — фактически **1 строка**: `#include "stdafx.h"` (файл без реализации).
- `UpdateSleep()` — закомментированный вызов в конце `shedule_Update` (`Actor.cpp:2086`).
- `actor_defs.h:73–75` — `typedef const char* EActorSleep; extern EActorSleep easCanSleepResult; extern string32 g_quick_use_slots[4];`; старый `enum EActorSleep` (easCanSleep/easNotSolidGround/easEnemies) закомментирован (77–85).
- Вывод: сон либо вынесен в скрипты/Lua, либо мёртв; C++-реализации нет.

## 12. Ночное зрение

### `CNightVisionEffector` (`ActorNightVision.cpp`, 89 строк; `ActorNightVision.h`, 22)

| Метод | Строки | Логика |
|---|---|---|
| `CNightVisionEffector(section)` | 10–17 | `HUD_SOUND_COLLECTION_LAYERED`: загрузка 4 звуков из секции — `snd_night_vision_on/off/idle/broken` (`SOUND_TYPE_ITEM_USING`) |
| `Start(sect, pA, play_sound)` | 19–27 | `AddEffector(pA, effNightvision, sect)` (PP-эффектор); при звуке — `eStartSound` + `eIdleSound` |
| `Stop(factor, play_sound)` | 29–42 | `Cameras().GetPPEffector(effNightvision)` → `pp->Stop(factor)`; звук остановки; `StopSound("NightVisionIdleSnd")` |
| `IsActive()` | 44–50 | PP-эффектор `effNightvision` существует |
| `OnDisabled(pA, play_sound)` | 52–56 | только `eBrokeSound` (отключение при запрете/сломе) |
| `PlaySounds(which)` | 58–89 | 1-е лицо (`pActor->HUDview()`) → `bPlaySoundFirstPerson`; `eStart/Stop/Idle/BrokeSound` → `PlaySound` по позиции актёра (idle — с `looped=true`) |

### `CActor::SwitchNightVision(vision_on, use_sounds, send_event)` (`Actor.cpp:3049–3110`, `Actor.h:862`)

1. `m_bNightVisionOn = vision_on`; lazy-создание `m_night_vision = xr_new<CNightVisionEffector>(cNameSect())`; `bIsActiveNow = IsActive()`.
2. Источник секции NV: **приоритет шлем** (`HELMET_SLOT` → `CHelmet::m_NightVisionSect`), иначе **костюм** (`OUTFIT_SLOT` → `CCustomOutfit::m_NightVisionSect`):
   - `m_bNightVisionAllow` и вкл. и не активен → `m_night_vision->Start(sect, this, use_sounds)`;
   - не разрешено → `OnDisabled(this, use_sounds)` + `m_bNightVisionOn = false`.
3. Выключение при активном → `Stop(100000.0f, use_sounds)`.
4. **Alun** (`send_event`): `m_trader_flags.set(eTraderFlagNightVisionActive, ...)` + `u_EventGen(GE_TRADER_FLAGS)` → `u_EventSend`.

**Состояние:** инициализация `Actor.cpp:245–247` (`m_night_vision=NULL`, allow=true, on=false), деструктор `xr_delete` (288); **восстановление из флага** при спавне — `Actor_Network.cpp:711–712` (`eTraderFlagNightVisionActive`, комментарий: при спавне шлема/костюма в слот сработает `SwitchNightVision(true)`); геттеры `Actor.h:864–866` (`GetNightVisionStatus/SetNightVisionAllowed/GetNightVision`).

**Вызовы:** переключатель-обёртка `CActor::SwitchNightVision()` (`ActorInput.cpp:775–778`, бинд `kNIGHT_VISION` в DEBUG-режиме 827–829; игровой бинд закомментирован — «controlled via LUA scripts», `ActorInput.cpp:172`); `CHelmet::OnMoveToSlot/Ruck` (`ActorHelmet.cpp:117–131`), `CCustomOutfit` (285–286/355), `Weapon.cpp:1372–1398` (зум оружия → NV off→on), Lua `script_game_object_inventory_owner.cpp:1380/1392`.

## 13. `CHelmet` (`ActorHelmet.cpp`, 344 строки)

**Наследование:** `class CHelmet : public CInventoryItemObject` (`ActorHelmet.h:8`) — **не `CEquipment`** (см. «Замечания к плану»). `DECLARE_SCRIPT_REGISTER_FUNCTION`.

- **Ctor** (15–23): `FUsingCondition=TRUE`, `m_HitTypeProtection` на `eHitTypeMax` = 1.0f, `xr_new<SBoneProtections>`.
- **`Load`** (30–68): 8 защит `*_protection` (Burn/Strike/Shock/Wound/Radiation/Telepatic/ChemicalBurn/Explosion); `FireWound` = **0.0f жёстко** (параметр отключён), `LightBurn := Burn`; `hit_fraction_actor`, опциональные `fire_wound_param_1/2` (`std::optional`, демонизед-формула Юрко), `nightvision_sect`, 5 `*_restore_speed`, `power_loss` (clamp EPS..1), `bones_koeff_protection`, `nearest_enemies_show_dist`, `use_condition`.
- **Кости:** `ReloadBonesProtection` (70–78) — в SP parent = `Level().CurrentViewEntity()`, `m_boneProtection->reload(sect, IKinematics)`; `net_Spawn` (80–87) вызывает в SP; `AddBonesProtection` (219–227), `get_HitFracActor` (229–232).
- **Сеть:** `net_Export/Import` (89–101) — condition `w_float_q8/r_float_q8` (в отличие от `CBackpack`, см. §14).
- **НВ-мост:** `OnMoveToSlot` (109–121) — при выходе из слота и `GetNightVisionStatus()` → `SwitchNightVision(true, false)`; `OnMoveToRuck` (123–134) → `SwitchNightVision(false)`.
- **`Hit`** (136–141): `hit_power *= GetHitImmunity(hit_type)` → `ChangeCondition(-hit_power)` (при `IsUsingCondition`).
- **Защиты:** `GetDefHitTypeProtection` = `protection × GetCondition()` (143–146); `GetHitTypeProtection` = ещё × `getBoneProtection(element)` (148–153); `GetBoneArmor` (155–158).
- **Апгрейды** `install_upgrade_impl` (160–217): все `*_protection` + `nightvision_sect` + 5 restore + `power_loss` + `nearest_enemies_show_dist` + `bones_koeff_protection` (→ `ReloadBonesProtection`) + `bones_koeff_protection_add` (→ `AddBonesProtection`) + `hit_fraction_actor` (demonized).
- **`HitThroughArmor`** (234–344) — ключевой метод попаданий в голову:
  - **`eHitTypeFireWound`** с `m_fireWoundParam1/2` (альтернативная формула Юрко): при `ap ≤ BoneArmor/var2` — `d_hit = HitFracActor` (× `var1` если `ap ≤ BoneArmor*var2`); иначе `d_hit = (ap − BoneArmor*var2)/(ap*(1.1 − BoneArmor*var1))`, при `ap/2 > BoneArmor/var2` — ещё `/var1`; `clamp(d_hit, HitFracActor, 1)`; `NewHitPower *= d_hit`.
  - **`FireWound` без параметров** (vanilla): `ap ≤ BoneArmor` → `× HitFracActor` (коммент «пуля НЕ пробила бронь»); иначе `d_hit = (ap − BoneArmor)/(ap*APScale)`, clamp `HitFracActor..1`.
  - **Прочие хиты:** `one = 1.0f` для Strike/Wound/Wound_2/Explosion, иначе `0.1f`; `NewHitPower −= GetDefHitTypeProtection × one`, clamp ≥ 0.
  - Завершение: `Hit(hit_power, hit_type)` — **изношенность шлема растёт от полной силы удара** (338); везде отладка `-dbgbullet`.

## 14. `CBackpack` (`ActorBackpack.cpp`, 102 строки)

`class CBackpack : public CInventoryItemObject` (`ActorBackpack.h:5`).

- **Ctor** (6–9): `FUsingCondition=FALSE` (в отличие от шлема — по умолчанию без износа).
- **`Load`** (15–26): `additional_inventory_weight`/`_weight2`, `power_restore_speed` (def 0), `power_loss` (def 1, clamp EPS..1), `use_condition` (def **TRUE** — `READ_IF_EXISTS`).
- **Сеть** (33–45): `net_Export/Import` — **сериализация condition закомментирована** (в отличие от `CHelmet::net_Export`).
- **`OnMoveToSlot/Ruck`** (52–80): только `inherited`, хуки с `smart_cast<CActor*>` закомментированы.
- **`Hit`** (82–87): идентично шлему (immunity × condition, но условие по умолчанию выкл.).
- **Апгрейды** (89–102): power_restore_speed/power_loss + обе additional-weight.

> Роль в `GetRestoreSpeed`/`UpdateArtefactsOnBeltAndOutfit` у рюкзака **нет** — туда идут только пояс/костюм/шлем (§3, §7). Роль в весе — **есть**: `CActor::get_additional_weight()` (`Actor_Movement.cpp:710–733`, порц. 23) суммирует `m_additional_weight` рюкзака (вместе с костюмом и артефактами пояса) для `MaxCarryWeight`/`MaxWalkWeight`; `m_additional_weight2`/`m_additional_weight` читаются/пишутся Lua (`GetAdditionalMaxWeight`/`GetAdditionalMaxWalkWeight`/`SetAdditionalMaxWeight`/`SetAdditionalMaxWalkWeight` — `script_game_object_inventory_owner.cpp:2506–2572`, порц. 15/18).

## 15. `CActor::OnEvent` (`Actor_Events.cpp`, 378 строк)

`inherited::OnEvent` + `CInventoryOwner::OnEvent` (29–30), затем switch:

| Кейс | Строки | Логика |
|---|---|---|
| `GE_TRADE_BUY`/`GE_OWNERSHIP_TAKE` | 35–87 | `net_Find(id)` → VERIFY; MP-мёртвый → отказ; `inventory().CanTakeItem` → `H_SetParent` + `Take(GO,false,true)` + `SelectBestWeapon`; иначе SP → `GE_OWNERSHIP_REJECT`-ответ, MP → ошибка; MP_LOGGING-логи |
| `GE_TRADE_SELL`/`GE_OWNERSHIP_REJECT` | 88–159 | `just_before_destroy = r_u8()`, `dont_create_shell` (SELL или destroy), `SetTmpPreDestroy`; проверки `H_Parent` и владельца; `inventory().DropItem(...)` → `m_feel_deny.feel_touch_deny(Obj, 1000)`; расширенный пакет: `dropPosition` → `GO->MoveTo`; `SelectBestWeapon` (если не destroy) |
| `GE_INV_ACTION` | 160–185 | `cmd/flags/ZoomRndSeed/ShotRndSeed`; MP-мёртвый → break; `CMD_START` → `SetZoomRndSeed` (kWPN_ZOOM) / `SetShotRndSeed` (kWPN_FIRE) + `IR_OnKeyboardPress(cmd)`, иначе `IR_OnKeyboardRelease` (порц. 24/19) |
| `GEG_PLAYER_ITEM2SLOT/2BELT/2RUCK/EAT/ACTIVATEARTEFACT` | 186–266 | VERIFY объекта/нев destroy/MP-мёртвый; `ACTIVATEARTEFACT` → `CArtefact::ActivateArtefact`; `ITEM2SLOT` → `Slot(slot_id, iitem, bDoNotActivate=r_u8()==1)`; `2BELT` → `Belt`; `2RUCK` → `Ruck`; `EAT` → Lua `_G.CInventory__eat` (bool) → `inventory().Eat` (порц. 18) |
| `GEG_PLAYER_ACTIVATE_SLOT` | 267–274 | `inventory().Activate(slot_id)` |
| `GEG_PLAYER_DISABLE_SPRINT` | 276–288 | `m_block_sprint_counter += cmd (s8)`; `>0` → `mstate_wishful &= ~mcSprint`; иначе clamp 0 (счётчик блокировок) |
| `GEG_PLAYER_WEAPON_HIDE_STATE` | 290–296 | `inventory().SetSlotsBlocked(State, Set)` |
| `GE_MOVE_ACTOR` | 297–305 | `MoveActor(NewPos, NewRot)` |
| `GE_ACTOR_MAX_POWER` | 306–311 | `conditions().MaxPower()` + `ClearWounds()` |
| `GE_ACTOR_MAX_HEALTH` | 312–316 | `SetfHealth(GetMaxHealth())` |
| `GEG_PLAYER_ATTACH_HOLDER` | 317–330 | `net_Find` → `VERIFY(m_holder==NULL)` → `CHolderCustom::Engaged()` → `use_Holder(holder)` (порц. 26) |
| `GEG_PLAYER_DETACH_HOLDER` | 331–339 | `VERIFY(id==holder ID)` → `use_Holder(NULL)` |
| `GEG_PLAYER_PLAY_HEADSHOT_PARTICLE` | 340–344 | `OnPlayHeadShotParticle(P)` |
| `GE_ACTOR_JUMPING` | 345–358 | **мёртвый кейс** — тело закомментировано (NET_Jump/mcJump-заготовка) |

**`MoveActor(NewPos, NewDir)`** (362–378): `M.translate(NewPos)` → `r_model_yaw/r_torso.yaw = NewDir.y`, `r_torso.pitch = −NewDir.x`, `unaffected_r_torso` (roll=0), `r_torso_tgt_roll=0`, `cam_Active()->Set(...)`, `ForceTransform(M)`, `m_bInInterpolation = false` (телепорт-телепорт-перемещение без интерполяции; см. порц. 23/28).

## 16. Связи и смежное

- **`SetActorVisibility`/`GetActorVisibility`** (`Actor.cpp:2094–2120`) — обёртки над `Level().CurrentViewEntity()` (single-only) → `SetVisibilityFromObject`/`GetVisibility`; `CActor::GetVisibility` (2122–2149) — сортировка `m_npc_visibility`, сглаживание `m_visibility_cur` по `Device.fTimeDelta`; `SetVisibilityValue` (2151+) — порц. 28 (связь видимости).
- **`actor_defs.h`** (154) — `ACTOR_DEFS`: `EActorSleep`/`easCanSleepResult`/`g_quick_use_slots` (73–75, §11), `EMoveCommand`/`EActorAction`/`EFreelookStates` (порц. 23/24), `EActorCameras` (порц. 23), `net_input`/`net_update`/`net_update_A`/`InterpData` (89–153, структуры порц. 28).
- **`actor_statistic_defs.h` / `actor_communication.cpp` (221)** — детали в порц. 30 (см. `actor-mp.md` §20–22).
- **`CHelmet`-связки:** `UpdateArtefactsOnBeltAndOutfit`/`GetRestoreSpeed` используют `CHelmet` только через `HELMET_SLOT` (§3, §7); `HitThroughArmor` вызывается из Hit-пути `CEntityCondition` (порц. 21); `SBoneProtections` — ит. 3/7 (`BoneProtections.h`).
- **`Weapon.cpp:1366–1401`** — зум оружия запоминает/гасит NV актёра (`GetNightVisionStatus`/`SwitchNightVision(false)`/`EnableActorNVisnAfterZoom` из `OnItemDrop`, §2/§12).

## 17. Замечания к плану

1. **`CHelmet` — не наследник `CEquipment`**, а `CInventoryItemObject` (`ActorHelmet.h:8`); план предлагал «проверить в коде» — проверено.
2. **`SwitchNightVision` (3049–3113) находится в `Actor.cpp`, не в `ActorNightVision.cpp`** — файл `ActorNightVision.cpp` содержит только `CNightVisionEffector` (89 строк); номера строк в плане (3049–3113) фактически 3049–3110.
3. **`Actor_Sleep.cpp` подтверждён как 1 строка** (только include); смежный `UpdateSleep()`-вызов в `shedule_Update` закомментирован (`Actor.cpp:2086`).
4. Порцион 34 также фактически покрывает `OnEvent` целиком (`Actor_Events.cpp` 27–360) и `MoveActor` — в плане упомянут только «`OnEvent`-обработчики» и `GE_*`-обвязка.
5. **`GetRestoreSpeed`/`can_attach`/`GetMass`/`is_on_ground`/`is_ai_obstacle`** в составе порциона 33 (план) не упомянуты явно — фактический диапазон `Actor.cpp:2896–2914` (физ.-геттеры) и `2916–3032` (`GetRestoreSpeed`) входит в «прочее» плана; включены в эту страницу.
6. **`OnDifficultyChanged` (2875–2889)** в плане порциона 33 не упомянут — смежный с `LoadImmunities` блок, включён в §8.
7. **`CBackpack`-роль в весе** (`get_additional_weight`, `MaxCarryWeight`/`MaxWalkWeight` — `Actor_Movement.cpp:693–733`, порц. 23) и в Lua `GetAdditionalMaxWeight`/`SetAdditionalMaxWalkWeight` (порц. 15/18) не охвачены планом порциона 34 (только «`CBackpack` (102): `CActorBackpack`») — включена в §14/§16.

## 18. Краткое резюме (порц. 34)

Сон в C++ отсутствует (`Actor_Sleep.cpp` — 1 строка, `UpdateSleep()` закомментирован, `EActorSleep` — только typedef/extern в `actor_defs.h`). Ночное зрение: `CNightVisionEffector` (4 звука + PP-эффектор `effNightvision`), `CActor::SwitchNightVision` выбирает секцию NV из шлема, затем из костюма, уважает `m_bNightVisionAllow`, при `send_event` публикует `eTraderFlagNightVisionActive` через `GE_TRADER_FLAGS` (восстановление при спавне — `Actor_Network.cpp:711`). `CHelmet : CInventoryItemObject` (не `CEquipment`) — 8 типов защиты, костная броня `SBoneProtections`, `HitThroughArmor` с опциональной формулой Юрко для FireWound и изношенностью от полного удара; `CBackpack` — тот же класс без NV/костей, condition-сеть закомментирована, износ по умолчанию выключен, но `m_additional_weight`/`m_additional_weight2` реально участвуют в весе (`get_additional_weight`, порц. 23) и в Lua-обёртках. `CActor::OnEvent` — приём `GE_TRADE_*`/`GE_OWNERSHIP_*` (take/drop с `SelectBestWeapon` и `feel_touch_deny`), `GE_INV_ACTION` (зеркало ввода с RNG-сидами), `GEG_PLAYER_*` (слот/пояс/рюкак/еда через Lua, holder'ы, блок спринта, скрытие слотов), `GE_MOVE_ACTOR`→`MoveActor` (телепорт без интерполяции); кейсы `GE_ACTOR_JUMPING` мёртвые.

