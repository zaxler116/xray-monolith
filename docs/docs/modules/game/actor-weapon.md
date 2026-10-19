# CActor: holder (транспорт) и оружие-обвязка (порционы 26–27)

> **Модуль:** `xrGame` · **Файлы:** `Actor.cpp` (3128), `Actor_Weapon.cpp` (406), `ActorInput.cpp` (855), `Actor_Events.cpp` (378), `ActorAnimation.cpp` (783), `holder_custom.h` (80), `holder_custom.cpp` (37), `Actor.h` (922) · **Порционы 26–27, итерация 5**
>
> **Страница:** §1–11 — holder-объекты (транспорт, стационарные пулемёты, `HolderEntityObject`) (порц. 26); §12–19 — оружие-обвязка: снаряды/drop, точность, отдача камеры, артефакты на поясе, misc (порц. 27).

## 1. Ответственность

Посадка/высадка игрока из объектов-«держателей» (машины `CCar`, стационарные пулемёты `CWeaponStatMgun`, `CHolderEntityObject`). Актёр на время использования holder'а теряет собственную физику (`DestroyCharacter`) и делегирует ввод/камеру объекту. Секция описывает сам мост актёра; реализация конкретных holder-объектов — итерация 8, Lua-доступ — порц. 16 (`holder_*`).

## 2. Интерфейс `CHolderCustom` (`holder_custom.h` 80, `holder_custom.cpp` 37)

Базовый интерфейс holder-объекта:

- **Состояние:** `m_owner` (`CGameObject*`), `m_ownerActor` (`CActor*`), `m_bEnterLocked`, `m_bExitLocked`. `Owner()/OwnerActor()` (protected), `Engaged() { return m_owner != NULL; }`.
- **Ввод:** чистые виртуальные `OnMouseMove/OnKeyboardPress/OnKeyboardRelease/OnKeyboardHold` — принимают делегированный ввод актёра.
- **Камера:** `virtual void cam_Update(float dt, float fov = 90.0f) = 0`, `virtual CCameraBase* Camera() = 0`.
- **Посадка/высадка:** `Use(pos, dir, foot_pos)` (по умолчанию true — условие посадки), `attach_Actor(CGameObject*)`, `detach_Actor()`, `EnterLocked()/ExitLocked()`, `SetEnterLocked/SetExitLocked`, `ExitPosition()`, `ExitVelocity()` (по умолчанию нулевой вектор).
- **Прочее:** `allowWeapon() const = 0` (разрешено ли оружие внутри), `HUDView() const = 0` (HUD-режим), `GetInventory() = 0` (инвентарь holder'а, напр. машина), `Action(u16 id, u32 flags)`, `SetParam(int id, Fvector2/Fvector val)`, `m_sUseAction` (tip-строка использования), `cast_holder_custom()`, `UpdateEx(float fov)` (вызывается владельцем — см. `UpdateCL` Actor.cpp:1164–1165: `if (m_holder) m_holder->UpdateEx(currentFOV())`).
- `DECLARE_SCRIPT_REGISTER_FUNCTION` — регистрируется в Lua (`holder_custom_script.cpp`); под `HOLDERCUSTOM_NEW` есть `Owner_script()`.

## 3. Поля и методы актёра (`Actor.h`)

| Элемент | Строка | Назначение |
|---|---|---|
| `CHolderCustom* m_holder` | 288 | Текущий holder (`NULL` если вне транспорта) |
| `u16 m_holderID` | 289 | ID объекта-holder (`u16(-1)` если нет) |
| `bool use_Holder(CHolderCustom*)` | 290 | Переключение (protected) — см. §5 |
| `void ActorUse()` | 291 | USE-действие (protected) — см. §6 |
| `bool use_HolderEx(CHolderCustom*, bool bForce)` | 284 | Низкоуровневое посадка/высадка — см. §4 |
| `attach_Vehicle/detach_Vehicle/steer_Vehicle` | 275–283 | `#ifdef HOLDERCUSTOM_NEW` разные сигнатуры (`detach_Vehicle(bool)`, `attach_Vehicle(obj,bool)`) vs старые (`detach_Vehicle()`, `attach_Vehicle(obj)`) |
| `CHolderCustom* Holder() { return m_holder; }` | 111 | Геттер |
| `m_pVehicleWeLookingAt` | 406 | Объект-транспорт, на который смотрит актёр (ray-query в `shedule_Update`) |
| `ALife::_OBJECT_ID m_holder_id` | 814 | ID holder'а для отложенного спауна (`on_requested_spawn`) |

## 4. `use_HolderEx(CHolderCustom* object, bool bForce)` — две реализации (`Actor.cpp:1479–1720`)

### 4.1 `#ifdef HOLDERCUSTOM_NEW` (`Actor.cpp:1480–1488`)

Простая обёртка: `object` → `attach_Vehicle(object, bForce)`; иначе `detach_Vehicle(bForce)`, return true.

### 4.2 `#else` — единая функция (`Actor.cpp:1617–1719`)

**Если `m_holder` есть (высадка):** при `!ExitLocked() || bForce` и (`!object || m_holder == object`):
1. Для `CPhysicsShellHolder`-holder'а: `SplitterHolderDeactivate()` → `movement()->ActivateBoxDynamic(0)` (проверка, что можно создать персонажа; при неудаче — `SplitterHolderActivate()` и выход, ссадиться нельзя), затем обратно `SplitterHolderActivate()`.
2. `SetWeaponHideState(INV_STATE_BLOCK_ALL, false)`, Lua-callback `eDetachVehicle(go)`.
3. `m_holder->detach_Actor()`; `movement()->CreateCharacter()`, `SetPosition(m_holder->ExitPosition())`, `SetVelocity(m_holder->ExitVelocity())`.
4. Ориентация: `r_model_yaw = -m_holder->Camera()->yaw; r_torso.yaw = r_model_yaw; r_model_yaw_dest = r_model_yaw;` `cam_Active()->Direction().set(m_holder->Camera()->Direction())`.
5. `SetCallbacks()`, `m_holder = NULL`, `m_holderID = u16(-1)`.
6. Анимации: `PlayCycle(m_anims->m_normal.legs_idle)`, `PlayCycle(m_anims->m_normal.torso_idle)`; восстановление head-бон-callback: `set_callback(bctPhysics, HeadCallback, this)` (бон `bip01_head`).

**Если `m_holder == NULL` (посадка):** при `(!object->EnterLocked() || bForce)` и (`bForce || object->Use(...) && object->attach_Actor(this)`):
1. `inventory().SetActiveSlot(NO_ACTIVE_SLOT)`, `SetWeaponHideState(INV_STATE_BLOCK_ALL, true)`.
2. `movement()->DestroyCharacter()` — персонаж-физика уничтожается (актёр «сидит» в holder'е).
3. `m_holder = object`, `m_holderID = oHolder->ID()`.
4. Убирается cam-bobbing (`Cameras().RemoveCamEffector(eCEBobbing)`), уничтожается `actor_camera_shell`.
5. Head-бон-callback заменяется на `VehicleHeadCallback`.
6. Для `CCar`: по `car->DriverAnimationType()` берёт `SVehicleAnimCollection` из `m_vehicle_anims->m_vehicles_type_collections[anim_type]`, `PlayCycle(anims.idles[0])`, `CStepManager::on_animation_start`.
7. Lua-callback `eAttachVehicle(go)`, return true.

### 4.3 `#ifdef HOLDERCUSTOM_NEW`: `attach_Vehicle` / `detach_Vehicle` (`Actor.cpp:1490–1615`)

**`attach_Vehicle(object, bForce)` (1490–1553):** условия и шаги посадки те же, что §4.2; дополнительно:
- перед посадкой сохраняет `inventory().SetPrevActiveSlot(GetActiveSlot())` и `SetActiveSlot(NO_ACTIVE_SLOT)` (высадка вернёт предыдущий слот);
- сбрасывает bone- callbacks `bip01_spine/spine1/spine2`;
- для `CCar`: если `car->IsRemoteControl() == false` (`CAR_NEW`) — play idle + `CStepManager::on_animation_start`;
- для `CWeaponStatMgun` (`STATIONARYMGUN_NEW`): `stm->UpdateAnimation()`.

**`detach_Vehicle(bForce)` (1555–1615):** шаги высадки как §4.2 (SplitterHolder/ActivateBoxDynamic(0), `CreateCharacter`, `SetPosition/SetVelocity` от `ExitPosition/ExitVelocity` для `CCar` при `!IsRemoteControl()` и для `CWeaponStatMgun`), плюс:
- `SetWeaponHideState(INV_STATE_BLOCK_ALL, false)`;
- восстановление слотов: `inventory().SetActiveSlot(GetPrevActiveSlot())`, `SetPrevActiveSlot(NO_ACTIVE_SLOT)`;
- `cam_Active()->Direction().set(Fvector().setHP(GO->Direction().getH(), 0))`;
- `SetCallbacks()`, idle-анимации, Lua `eDetachVehicle`.

## 5. `use_Holder(CHolderCustom* holder)` (`ActorInput.cpp:485–523`)

Тоггл-обёртка:
- **Если уже в holder'е:** `use_HolderEx(0, false)` (высадка), затем `inventory().ActiveItem()` — `CHudItem::OnAnimationEnd(GetState())` (докрутка анимации активного предмета).
- **Иначе (посадка):** `use_HolderEx(holder, false)`; при успехе выключается факел: `CAttachmentOwner::attachedItem(CLSID_DEVICE_TORCH)` → `CTorch::Switch(false)`. Также `OnAnimationEnd` для активного HUD-предмета.

## 6. `ActorUse()` (`ActorInput.cpp:525–628`) — USE

- **Если в holder'е:** отправляет сетевой `GEG_PLAYER_DETACH_HOLDER` (`u_EventGen(P, GEG_PLAYER_DETACH_HOLDER, ID())`, `P.w_u16(GO->ID())`) и возвращается — высадка идёт через событие (см. §7).
- **Иначе:** если нет `AF_MULTI_ITEM_PICKUP` → `m_bPickupMode = true`; `PHCapture()` → `PHReleaseObject()` (отпускание захваченной физики).
- Если `m_pUsableObject && NULL == m_pObjectWeLookingAt->cast_inventory_item()` → `m_pUsableObject->use(this)`.
- Если `m_pInvBoxWeLookingAt` и `nonscript_usable()` → в SP `pGameSP->StartCarBody(this, inv_box)` (открытие контейнера), return.
- Если `!m_pUsableObject || nonscript_usable()`:
  - **LSHIFT + объект**: физический захват `PHCaptureObject(object, element)` из `HUD().GetRQ()`, с проверкой секции `ph_capture_visuals` и Lua `_G.CActor__OnBeforePHCapture(object, b_allow)`.
  - **Иначе если объект — `CHolderCustom`**: отправляет `GEG_PLAYER_ATTACH_HOLDER` (`P.w_u16(object->ID())`) и return — посадка идёт через событие.
  - Если `m_pPersonWeLookingAt`: в SP живому — `TryToTalk()`; мёртвому (после `GetLevelDeathTime() + 3000`) — `pGameSP->StartCarBody(...)` (обыск трупа).

## 7. Сетевые события (`Actor_Events.cpp:317–339`)

- **`GEG_PLAYER_ATTACH_HOLDER`**: `u16 id = P.r_u16()` → `Level().Objects.net_Find(id)` (ошибка, если не найден) → `VERIFY(m_holder==NULL)` → `use_Holder(holder)` только если `!holder->Engaged()` (holder не занят другим актёром).
- **`GEG_PLAYER_DETACH_HOLDER`**: `if (!m_holder) break` → `u16 id = P.r_u16()` → `VERIFY(id == GO->ID())` → `use_Holder(NULL)`.

События генерируются клиентом в `ActorUse()` (§6) и исполняются актёром — единый канал посадки/высадки.

## 8. `on_requested_spawn(CObject* object)` (`Actor.cpp:1722–1756`)

Отложенный вход в holder после спауна (holder появился раньше актёра):
- `smart_cast<CHolderCustom*>` — если не holder, возврат.
- `HOLDERCUSTOM_NEW`: `use_HolderEx(oHolder, true)` (форс).
- `#else`: сначала высадка из текущего (SplitterHolder/`ActivateBoxDynamic(0)`/`CreateCharacter`/`ExitPosition/ExitVelocity`), `m_holder = NULL`, затем `use_HolderEx(oHolder, true)`; в конце `r_torso.yaw` берётся из `object->XFORM().getXYZi()`.

Связано с `m_holder_id` (Actor.h:814) и `Level().client_spawn_manager().add/remove(m_holder_id, ID(), callback)` в `Actor_Network.cpp` (513, 571, 681–684, 725–727) — spawn-менеджер дожидается появления holder-объекта. Также в `net_Export/net_Import` сериализуется `m_holderID` (`P.w_u16(m_holderID)` — Actor_Network.cpp:1785, 1790).

## 9. Прочие точки интеграции

| Место | Что делает |
|---|---|
| `CActor::Die` (Actor.cpp:867–871) | При смерти форс-высадка: `use_HolderEx(NULL, true)` (`HOLDERCUSTOM_NEW`) |
| `UpdateCL` (Actor.cpp:1164–1165) | `if (m_holder) m_holder->UpdateEx(currentFOV())` |
| `IR_OnKeyboardPress/Release/Hold`, `IR_OnMouseMove` (ActorInput.cpp:86–93, 330–335, 359–362, 458–462) | Если есть `m_holder` и это не `kUSE` — ввод делегируется `m_holder->On*` (+ `allowWeapon()` и `inventory().Action`) |
| `HUDview()` (ActorInput.cpp:630–634) | HUD виден, если `IsFocused() && eacFirstEye && (!m_holder || (m_holder->allowWeapon() && m_holder->HUDView()))` |
| `canRenderLegs` (Actor.cpp:2235–2240) | Ноги не рендерятся при `m_holder` (и иных условиях) |
| Камеры (ActorCameras.cpp:237, 481) | ladder-камера блокируется при `m_holder`; `if (m_holder) return;` в ветке камеры |
| `steer_Vehicle(float angle)` (ActorAnimation.cpp:288–304) | Если holder — `CCar` (`ENABLE_CAR`): по `angle` проигрывает цикл `anims.idles[0]/steer_right/steer_left`; вызывается из `CarInput.cpp` (197, 200, 247, 250) |
| `CActor::can_attach` (Actor.cpp:2857–2873) | Присоединение `CAttachableItem` (секция в `m_attach_item_sections`, не дубликат) |
| `GEG_PLAYER_ATTACH_HOLDER` извне | `Car.cpp:359` (`OwnerActor()->use_HolderEx(nullptr, true)` при смерти водителя), `Car.cpp:2069`, `Entity.cpp:271`, `HolderEntityObject.cpp:90/95`, Lua `script_game_object3.cpp:1763–1793` (`actor->use_HolderEx(vehicle, bForce)` — порц. 16), `ai_stalker.cpp:1785` (`CAI_Stalker::use_HolderEx`) |

## 10. Связи

- **`CCharacterPhysicsSupport` (ит. 7):** `movement()->DestroyCharacter/CreateCharacter`, `ActivateBoxDynamic(0)`, `SetPosition/SetVelocity`, `PHCapture/PHReleaseObject`.
- **Holder-объекты (ит. 8):** `CCar`, `CWeaponStatMgun`, `CHolderEntityObject` реализуют `CHolderCustom`.
- **Инвентарь (ит. 8/порц. 33):** `SetActiveSlot/SetPrevActiveSlot`, `NO_ACTIVE_SLOT`, `SetWeaponHideState(INV_STATE_BLOCK_ALL, ...)`, `CHudItem::OnAnimationEnd`.
- **Ввод (порц. 24):** делегирование `m_holder->On*`, `ActorUse` (USE), `HUDview()`-логика.
- **Камера (порц. 32):** `cam_Active()->Direction()`, cam-bobbing (`eCEBobbing`), `actor_camera_shell`, блокировки в `ActorCameras.cpp`.
- **Lua:** `eAttachVehicle`/`eDetachVehicle` callback'и, `_G.CActor__OnBeforePHCapture`, `holder_*` API (порц. 16), `DECLARE_SCRIPT_REGISTER_FUNCTION` для `CHolderCustom`.
- **Сеть (порц. 28):** события `GEG_PLAYER_ATTACH_HOLDER/DETACH_HOLDER`, `m_holderID` в `net_Export/net_Import`, `client_spawn_manager` + `on_requested_spawn` (`m_holder_id`).
- **Анимация (порц. 31):** `m_vehicle_anims`/`SVehicleAnimCollection`, `steer_Vehicle`, idle-циклы, `CStepManager::on_animation_start`.
- **Смерть (порц. 22):** форс-высадка в `Die`.

## 11. Краткое резюме

Holder-режим: `CHolderCustom` задаёт интерфейс (ввод/камера/выход). Пункт USE → `ActorUse()` шлёт `GEG_PLAYER_ATTACH_HOLDER/DETACH_HOLDER`; события исполняются через `use_Holder()` → `use_HolderEx()` (2 реализации под `HOLDERCUSTOM_NEW`), где `attach_Vehicle` уничтожает физику персонажа, прячет оружие, ставит idle-анимацию машины, а `detach_Vehicle` наоборот восстанавливает персонажа (`CreateCharacter`) на `ExitPosition/ExitVelocity` и слот оружия. Всё время в holder'е ввод/мышь/камера/HUD делегируются объекту (`allowWeapon()/HUDView()`), рендер ног отключён, смерть и `on_requested_spawn` форсируют посадку/высадку, сеть синхронизирует `m_holderID` через spawn-менеджер.

---

# Оружие-обвязка (порцион 27)

## 12. Ответственность

`Actor_Weapon.cpp` (406) + блоки `Actor.cpp`: связка актёра с оружием и предметами — расчёт точности (`GetWeaponAccuracy`), параметры выстрела (`g_fireParams/GetPick/g_WeaponBones`), состояние (`g_State`), сетевые управляющие события (`SetCantRunState/SetWeaponHideState`), автоподбор оружия (`SelectBestWeapon`), индикация попаданий (`HitSector`), отдача камеры (`on_weapon_shot_*`, `weapon_recoil_*`), автоспавн патронов (`Spawn/RemoveAmmoForWeapon`), drop предметов (`g_PerformDrop`), бросок снарядов (`use_default_throw_force/missile_throw_force`), артефакты на поясе/вышмоде (`UpdateArtefactsOnBeltAndOutfit`, `HitArtefactsOnBelt`, `GetProtection_ArtefactsOnBelt`), misc (`Set/SetShotRndSeed`, `Get/SetMissileOffset`, `unlimited_ammo`, `SwitchNightVision`).

## 13. Точность и параметры выстрела (`Actor_Weapon.cpp`)

| Метод | Строки | Назначение |
|---|---|---|
| `GetWeaponAccuracy() const` | 33–101 | Текущий разброс (рад) с учётом движения. `m_fDispBase * GetWeaponParam(W, ...)` (PDM-множители базы/подавителя/прицела/гранатомёта), множители скорости: угловая (`fAVelocity`/`VEL_A_MAX=10.f`, опция `g_fix_avelocity_spread` → `fCurAVelocity`) и линейная (`state.fVelocity/VEL_MAX=10.f`), множитель ускорения/приседа (`m_fDispAccelFactor`, `m_fDispCrouchFactor`, `m_fDispCrouchNoAccelFactor`), buckShot. Прицеливание: `m_fDispAim` (опции `g_apply_pdm_to_ads`, `g_smooth_ads_transition` — `_lerp(m_fDispAim, 1, GetInertionAimFactor())`). Глобальные `g_fix_avelocity_spread/g_apply_pdm_to_ads/g_smooth_ads_transition` (29–31). |
| `GetPick()` | 103–112 | `SPickParam&` — pick-параметры от HUD-прикрепления `g_player_hud->attached_item(0/1)`, иначе `HUD().GetPick()`. |
| `g_fireParams(const CHudItem*, Fvector& fire_pos, Fvector& fire_dir)` | 114–128 | Заполняет позицию/направление выстрела: `item_0/1->m_parent_hud_item->g_fireParams(pp)`, результат из `pp.defs.start/dir`. |
| `g_WeaponBones(int& L, int& R1, int& R2)` | 130–135 | Кости оружия: `R1 = m_r_hand`, `R2 = m_r_finger2`, `L = m_l_finger1`. |
| `g_State(SEntityState& state) const` | 137–146 | Флаги `bJump/bCrouch/bFall/bSprint` из `mstate_real` (порц. 23), `fVelocity` из `movement()->GetVelocityActual()`, `fAVelocity = fCurAVelocity`. |
| `GetWeaponAccuracyStm()` (STATIONARYMGUN_NEW) | 397–405 | Если `Holder()` — `CWeaponStatMgun` с зумом камеры → `m_fDispAim`, иначе `m_fDispBase` (связь с порц. 26). |

## 14. Управляющие события (`SetCantRunState`, `SetWeaponHideState`)

- `SetCantRunState(bool bDisable)` (148–157): если жив и это текущий контролируемый объект — шлёт `GEG_PLAYER_DISABLE_SPRINT` (`P.w_s8(+1/-1)`).
- `SetWeaponHideState(u16 State, bool bSet)` (159–169): шлёт `GEG_PLAYER_WEAPON_HIDE_STATE` (`P.w_u16(State)`, `P.w_u8(bSet)`).
- **Обработчики** (`Actor_Events.cpp:276–296`): `GEG_PLAYER_DISABLE_SPRINT` → `m_block_sprint_counter += cmd`; при `>0` снимает `mstate_wishful &= ~mcSprint`, иначе обнуляет счётчик. `GEG_PLAYER_WEAPON_HIDE_STATE` → `inventory().SetSlotsBlocked(State, !!Set)`.
- Вызывающие: `SetWeaponHideState(INV_STATE_BLOCK_ALL, ...)` при посадке/высадке из holder'а (порц. 26), ladder (порц. 23); `GEG_PLAYER_WEAPON_HIDE_STATE` также шлётся из `controller_psy_hit.cpp` (107, 297).

## 15. Автоподбор и индикация попаданий

- `SelectBestWeapon(CObject* O)` (178–233): только для MP (`IsGameTypeSingle() → return`); если `O` — оружие/граната/артефакт (артефакт с `H_Parent()` пропускается), в режимах `eGameIDArtefactHunt`/`eGameIDCaptureTheArtefact` не выбирает, если в `BaseSlot` уже есть другое. Перебирает `BestWeaponSlots[] = {INV_SLOT_3, INV_SLOT_2, GRENADE_SLOT, KNIFE_SLOT}` (171–176), при `can_kill()` → `inventory().Activate(slot)`.
- `HitSector(CObject* who, CObject* weapon)` (238–263): если `g_bShowHitSectors`, жив, `who` — другой `CEntityAlive`, оружие не с подавителем (`IsSilencerAttached()`) → `Level().MapManager().AddMapLocation("mp_hit_sector_location", who->ID())` (MP-индикация, `ENEMY_HIT_SPOT` 235).

## 16. Отдача камеры (`on_weapon_shot_*`, `weapon_recoil_*`)

- `on_weapon_shot_start(CWeapon*)` (265–292): выбирает `weapon->zoom_cam_recoil` при `IsZoomAimingMode()`, иначе `cam_recoil`; находит/создаёт `CCameraShotEffector` (`eCEShot`), `Initialize/UpdateCameraRecoil` при смене оружия, `SetRndSeed(GetShotRndSeed())`, `SetActor(this)`, `Shot(weapon)`.
- `on_weapon_shot_update()` (294–301) → `update_camera(effector)`.
- `on_weapon_shot_remove(CWeapon*)` (303–306) → `RemoveCamEffector(eCEShot)`.
- `on_weapon_shot_stop()` (308–315) → `effector->StopShoting()` если активен.
- `on_weapon_hide(CWeapon*)` (317–322) → `effector->Reset()` если активен.
- `weapon_recoil_delta_angle()/weapon_recoil_last_delta()` (324–344) → `GetDeltaAngle/GetLastDelta` (или нулевой вектор).

## 17. Патроны и drop

- `SpawnAmmoForWeapon(CInventoryItem*)` (348–359): `OnClient → return`; если `CWeaponMagazined::AutoSpawnAmmo()` → `pWM->SpawnAmmo(0xffffffff, NULL, ID())`.
- `RemoveAmmoForWeapon(CInventoryItem*)` (361–394): `OnClient → return`; находит `CWeaponAmmo` по `m_ammoTypes[0]` (`inventory().GetAny`) → `pAmmo->DestroyObject()` (проверка «не нужен ли другим оружию» закомментирована).
- `g_PerformDrop()` (`Actor.cpp:2368–2381`): сбрасывает `b_DropActivated`; активный предмет, не `IsQuestItem()`, слот не `SlotIsPersistent(s)` → `pItem->SetDropManual(TRUE)`. Вызывается из input по `kDROP` при `GAME_PHASE_INPROGRESS` (`ActorInput.cpp:345`), а также монстрами (`burer.cpp:283`, `controller.cpp:759` — `Actor()->g_PerformDrop()`).
- `use_default_throw_force()` (`Actor.cpp:2383–2389`): `g_Alive()` (для мёртвого — false, сила по умолчанию не применяется).
- `missile_throw_force()` (`Actor.cpp:2391–2394`): `0.f` (у актёра своя сила не задана — дефолтная). Используются в `Missile.cpp:565–568` (`m_fake_missile->m_fThrowForce = inventory_owner->missile_throw_force()` при `use_default_throw_force()`).
- `OnItemDropUpdate()` (`Actor.cpp:2647–2657`): вызывается из `Inventory.cpp:908`; `CInventoryOwner::OnItemDropUpdate()` + прикрепляет к актёру невалидные/неприкреплённые предметы (`attach(*I)`).
- `OnItemRuck/OnItemBelt` (2660–2668) — делегируют `CInventoryOwner`.

## 18. Артефакты на поясе и вышмод (`Actor.cpp`)

- **`UpdateArtefactsOnBeltAndOutfit()` (2672–2742):** вызывается в конце `shedule_Update` (2089). Троттлинг `ARTEFACTS_UPDATE_TIME = 0.100f` по `conditions().fdelta_time()`. Для каждого `CArtefact` из `inventory().m_belt` (с учётом `GetCondition()`) применяет `conditions().Change*`: Bleeding/Health/Power/Satiety/`m_f*RestoreSpeed`; радиация — с вычетом `GetBoostRadiationImmunity()` при положительном `m_fRadiationRestoreSpeed` (`clamp(val, 0, val)`). Затем суммирует те же `m_f*RestoreSpeed` от `GetOutfit()` + шлема из `HELMET_SLOT` (`CHelmet`). Закомментированная ветка выключения NV у факела при отсутствии шлема (2730–2741).
- **`HitArtefactsOnBelt(float hit_power, ALife::EHitType)` (2744–2774):** сначала Lua `_G.CActor__HitArtefactsOnBelt(table, hit_power, hit_type)` — если возвращает таблицу с `override==true`, результат берётся из неё; иначе по `inventory().m_belt`: `hit_power -= artefact->m_ArtefactHitImmunities.AffectHit(1.0f, hit_type)`, затем `clamp(hit_power, 0, flt_max)`. Вызывается при обработке урона (`Actor.cpp:693, 752`); детали — порц. 21.
- **`GetProtection_ArtefactsOnBelt(ALife::EHitType)` (2776–2790):** сумма `AffectHit(1, hit_type) * GetCondition()` по артефактам пояса. Детали применения — порц. 21.

## 19. Misc (`Actor.cpp`)

| Метод | Строки | Назначение |
|---|---|---|
| `SetZoomRndSeed(s32 Seed)` | 2792–2796 | `m_ZoomRndSeed = Seed` или `Level().timeServer_Async()` если 0 (детали использования — порц. 19) |
| `SetShotRndSeed(s32 Seed)` | 2798–2802 | `m_ShotRndSeed` аналогично; используется `on_weapon_shot_start` → `effector->SetRndSeed` |
| `GetMissileOffset()/SetMissileOffset(const Fvector&)` | 2804–2812 | `m_vMissileOffset` — смещение снаряда (детали — порц. 19) |
| `unlimited_ammo()` | 3044–3047 | `!!psActorFlags.test(AF_UNLIMITEDAMMO)` |
| `SwitchNightVision(bool vision_on, bool use_sounds, bool send_event)` | 3049–3110 | Обвязка NV: `m_bNightVisionOn`; лениво `xr_new<CNightVisionEffector>(cNameSect())`; приоритет секции шлема (`HELMET_SLOT`, `m_NightVisionSect`), иначе вышмода (`OUTFIT_SLOT`); при `m_bNightVisionAllow` — `m_night_vision->Start(sect, this, use_sounds)`, иначе `OnDisabled` + сброс флага; выключение → `Stop(100000.f, use_sounds)`; при `send_event` обновляет `m_trader_flags.set(eTraderFlagNightVisionActive, ...)` и шлёт `GE_TRADER_FLAGS`. Тоггл-обёртка `SwitchNightVision()` (без аргументов) — `ActorInput.cpp:775–777`; вызовы из `ActorHelmet.cpp:118/131`, `CustomOutfit.cpp:286/355`, `Weapon.cpp:1375/1398`, Lua `script_game_object_inventory_owner.cpp:1380` (детали порц. 34) |

## 20. Связи оружие-обвязки

- **Оружие (ит. 8):** `CWeapon/CWeaponMagazined/CWeaponAmmo/CGrenade/CHudItem`, PDM-параметры (`Get_PDM_*`), `zoom_cam_recoil/cam_recoil`.
- **Прицеливание/движение:** `IsZoomAimingMode`, `mstate_real` (порц. 23), `character_physics_support()->movement()` (`GetVelocityActual`).
- **Сеть (порц. 28):** `GEG_PLAYER_DISABLE_SPRINT`/`GEG_PLAYER_WEAPON_HIDE_STATE` (обработчики `Actor_Events.cpp`, `xrServer_process_event.cpp:319–320` — relay), `OnClient/OnServer`-проверки в Spawn/RemoveAmmo.
- **Урон (порц. 21):** `HitArtefactsOnBelt/GetProtection_ArtefactsOnBelt` в `HDS.damage()`-цепочке.
- **Условия (порц. 20):** весь блок `UpdateArtefactsOnBeltAndOutfit` работает через `conditions().Change*/fdelta_time()/GetBoostRadiationImmunity()`.
- **HUD (порц. 32):** `GetPick`, `g_fireParams`, `g_player_hud->attached_item`.
- **Камера/эффекторы (порц. 32):** `CCameraShotEffector` (`eCEShot`), `update_camera`.
- **Holder (порц. 26):** `GetWeaponAccuracyStm` от `Holder()`; `SetWeaponHideState(INV_STATE_BLOCK_ALL)`.
- **Lua:** `_G.CActor__HitArtefactsOnBelt`.
- **Смерть (порц. 22):** `use_default_throw_force` возвращает false для мёртвого.

## 21. Краткое резюме порциона 27

Оружие-обвязка актёра: `Actor_Weapon.cpp` даёт точность (`GetWeaponAccuracy` — база × PDM × скорость × присед × прицел), параметры выстрела (`g_fireParams/GetPick/g_WeaponBones/g_State`), управляющие события (спринт-блок, скрытие слотов), автоподбор лучшего оружия (MP), карту попаданий, отдачу камеры (`on_weapon_shot_*` + `CCameraShotEffector`), автоспавн/удаление патронов. В `Actor.cpp` — drop активного предмета (`g_PerformDrop` через `kDROP`), сила броска снарядов, `OnItemDropUpdate` (прикрепление предметов), артефакты на поясе и вышмоде (`UpdateArtefactsOnBeltAndOutfit` по таймеру 0.1 c через `conditions()`, иммунитеты `HitArtefactsOnBelt` с Lua-override), misc: rnd-сиды, `m_vMissileOffset`, `unlimited_ammo`, `SwitchNightVision` (шлем → вышмод → `CNightVisionEffector`, сеть `GE_TRADER_FLAGS`).

