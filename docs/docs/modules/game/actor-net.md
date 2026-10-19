# CActor: сеть и интерполяция (порцион 28)

> **Модуль:** `xrGame` · **Файлы:** `Actor_Network.cpp` (2028), `Actor.cpp` (3128), `Actor.h` (922), `actor_defs.h` (154) · **Порцион 28, итерация 5**
>
> **Страница:** сетевой обмен и клиентская интерполяция: `net_Export/net_Import` (+ `_Base/_Physic`-ветки), `net_ExportDeadBody` (каркас — порц. 22), `net_Spawn/net_Destroy/net_Relevant/net_Relcase` (порц. 3–4), коррекция физики `PH_*_CrPr`, `CalculateInterpolationParams`/`make_Interpolation`, `save/load/net_Save`, `UpdateCL` (детали), `RPC_Update*`, видимость (`SetActorVisibility`/`GetVisibility`/…), hit-события (`SetHitInfo`/`OnHitHealthLoss`/`OnCritical*`).

## 1. Ответственность

Клиент-серверный обмен состоянием игрока: экспорт в сервер (`net_Export`), импорт с сервера (`net_Import`), хранение истории обновлений (`NET`, `NET_A`), механизм correction-prediction (`PH_B_CrPr`/`PH_I_CrPr`/`PH_A_CrPr`), клиентская интерполяция сплайнами (Безье/Эрмита/линейная), сериализация (`save/load/net_Save`), ежекадровое `UpdateCL` (pickup, HUD-дисперсия, Discord, LV-шейдеры), RPC-обновления Discord-инфо, система видимости актёра, hit-события статистики/килфидов.

## 2. Глобальные переменные и `Actor()` (`Actor_Network.cpp:54–67`)

- `g_cl_InterpolationType` (0=линейная, 1=Безье, 2=Эрмита), `g_cl_InterpolationMaxPoints` (лимит отладочных стеков), `g_dwInputUpdateDelta=20`, `net_cl_inputguaranteed`.
- `CActor* g_actor` + `Actor()` — геттер с `R_ASSERT2(GameID() == eGameIDSingle, ...)` (только SP).
- `actInterpType` (1228), `extern float g_cl_lvInterp` (1057) — настройки интерполяции.

## 3. Структуры данных (`actor_defs.h:107–153`, `Actor.h:588–641`)

- **`net_update`** (`actor_defs.h:107`): `dwTimeStamp` (серверный таймстемп), `o_model` (yaw модели), `o_torso` (`SRotation`), `p_pos/p_accel/p_velocity`, `mstate`, `weapon`, `fHealth`; метод `lerp(A,B,f)` — **весь закомментирован** (`Actor_Network.cpp:909–925`).
- **`net_update_A`** (`:137`): `dwTimeStamp` + `SPHNetState State` (состояние физики).
- **`InterpData`** (`:147`): `Pos/Vel/o_model/o_torso`.
- **Поля актёра** (`Actor.h:588–641`): `xr_deque<net_update> NET` (≤5 элементов), `NET_SavedAccel`, `NET_Last`, `NET_WasInterpolating`, `NET_Time`; `xr_deque<net_update_A> NET_A` (≤5); сплайны `SCoeff[3][4]` (Безье), `HCoeff[3][4]` (Эрмита), `IPosS/IPosH/IPosL`; состояния `LastState/RecalculatedState/PredictedState`, `IStart/IRec/IEnd`, флаги `m_bInInterpolation/m_bInterpolate`, тайминги `m_dwIStartTime/m_dwIEndTime/m_dwILastUpdateTime`; `PH_STATES m_States`, `m_u16NumBones`.

## 4. Экспорт

### 4.1 `net_Export(NET_Packet& P)` (88–160) — клиент → сервер

- `GetfHealth()` (float), `Level().timeServer()` (u32), `flags` (u8=0), позиция `Position()` (vec3).
- Углы: `angle_normalize(r_model_yaw)`, `unaffected_r_torso.yaw/pitch/roll` (по float вместо angle8).
- `g_Team()/g_Squad()/g_Group()` (u8).
- `mstate_real & 0xffff` (u16), `NET_SavedAccel` (sdir), `GetVelocity()` из `movement()` (sdir), `g_Radiation()` (float), `inventory().GetActiveSlot()` (u8).
- Далее физика: `NumItems = PHGetSyncItemsNumber()`; обнуляется если `H_Parent()`, SP, (`NumItems>1 && OnClient()`) или мёртв (`125–128`). При `NumItems==0` — выход после записи счётчика.
- Если жив: для sync-объекта 0 — `enabled`, `angular_vel`, `linear_vel`, `force`, `torque`, `position`, `quaternion(x,y,z,w)` (по float).
- Если мёртв → `net_ExportDeadBody(P)`.

### 4.2 `net_ExportDeadBody(NET_Packet& P)` (247–288) — каркас

Для мёртвого актёра: по всем `PHGetSyncItemsNumber()` костям собирает min/max-границы позиций (+`linear_vel/10`), пишет `P.w_u8(10)`, min, max, затем по костям `w_vec_q8(position)`, `w_qt_q8(quaternion)`, `w_vec_q8(px)` (квантование q8 внутри границ). Помощники `w_vec_q8/r_vec_q8/w_qt_q8/r_qt_q8/UpdateLimits` (162–245). **Детали — порц. 22.**

## 5. Импорт

### 5.1 `net_Import(NET_Packet& P)` (290–301)

`net_Import_Base(P)` → `m_u16NumBones = P.r_u16()` (0 → return) → `net_Import_Physic(P)`.

### 5.2 `net_Import_Base(NET_Packet& P)` (303–403)

- `health` (float) → при `OnClient()` — `SetfHealth`; `N.dwTimeStamp` (u32); `flags` (u8); `N.p_pos` (vec3); `o_model`, `o_torso.yaw/pitch/roll` (float; roll > PI → `-2π`).
- `id_Team/id_Squad/id_Group` (u8). В `Level().IsDemoPlay()` — копирование торса в `unaffected_r_torso` и камеру (`cam_Active()->yaw/pitch`).
- `mstate` (u16→u32), `p_accel`, `p_velocity` (sdir), радиация (при `OnClient()` → `SetfRadiation`), `ActiveSlot` (u8; при `OnClient()`: `NO_ACTIVE_SLOT` или `inventory().Activate` если отличается).
- **Буферизация**: `Local() && OnClient()` → return; `N.dwTimeStamp < NET.back()` → return (регресс); равенство → замена `NET.back()`, иначе `push_back` + `pop_front` при `size()>5`.
- → `net_Import_Base_proceed()` (405–417): если жив — `setVisible(TRUE)/setEnabled(TRUE)`; `Remote()` → return.

### 5.3 `net_Import_Physic(NET_Packet& P)` (419–502)

- `m_States.clear()`.
- **Если `m_u16NumBones != 1` (труп, много костей):** читает `u8`, min/max, для каждой кости `r_vec_q8(position)`, `r_qt_q8(quaternion)`, `r_vec_q8(linear_vel)` и восстанавливает скорость как `(px - position) * 10`; заполняет `m_States`.
- **Если 1 кость (живой):** `net_update_A N_A` — `enabled` (u8), `angular_vel`, `linear_vel`, `force`, `torque`, `position` (vec3), `quaternion` (4 float); `N_A.dwTimeStamp` берётся из `NET.back()` или `Level().timeServer()`; `previous_position/quaternion = current`. Условие `Local() && OnClient() || !g_Alive()` → return; далее буферизация по времени в `NET_A` (≤5, отбрасывание старых/замена по равенству; в DEBUG — `VERIFY(valid_pos(...))`); при непустом `NET_A` → `m_bInterpolate = true`.
- → `net_Import_Physic_proceed()` (504–509): `Level().AddObject_To_Objects4CrPr(this)`, `CrPr_SetActivated(false)`, `CrPr_SetActivationStep(0)` — постановка в очередь correction-prediction.

## 6. Жизненный цикл

- **`net_Spawn(CSE_Abstract*)` (511–715)** — см. порц. 3–4. Ключевое для сети: сброс `m_holder_id/feel_touch_characters/snd_noise`; `M_SPAWN_OBJECT_LOCAL` → `g_actor = this`; `m_pActorEffector = xr_new<CActorCameraManager>()`; `CInventoryOwner::net_Spawn` + `inherited::net_Spawn`; `mstate_wishful = E->mstate & (mcCrouch|mcAccel)`, `set_state_box`, `m_pPhysics_support->in_NetSpawn`; при `E->m_holderID != -1` — `DestroyCharacter()`; торс из `E->o_torso`; камера `cam_Set` (`AF_PSP`→`eacLookAt`, иначе `eacFirstEye`); `NET_SavedAccel.set(0)`, `NET_WasInterpolating = TRUE`; `setEnabled(M_SPAWN_OBJECT_LOCAL)`, `Engine.Sheduler.Register`; `m_bInInterpolation/m_bInterpolate = false`; мёртвый → `PlayCycle("death_init")`; callback `on_requested_spawn` в `client_spawn_manager().add(E->m_holderID, ID(), callback)`; `m_iLastHitterID` и др. = -1; SP → `MapManager().AddMapLocation("actor_location"/"actor_location_p")` + `m_statistic_manager`; `m_bNightVisionOn` из `eTraderFlagNightVisionActive`.
- **`net_Destroy()` (721–776):** `client_spawn_manager().remove(m_holder_id, ID())`, `delete_data(m_statistic_manager)`, `MapManager().OnObjectDestroyNotify`, `CInventoryOwner::net_Destroy`, `cam_UnsetLadder`, `movement()->DestroyCharacter`, удаление shell/effector'ов/pStatGraph/m_pActorEffector, `processing_deactivate`, `m_holder=NULL`, `SetDefaultVisualOutfit(NULL)`, `g_actor=NULL`, `Engine.Sheduler.Unregister`, `destroy_physics_shell(actor_camera_shell)` если принадлежит, `crash_saving::save_impl = nullptr`.
- **`net_Relcase(CObject* O)` (778–804):** очистка `m_pObjectWeLookingAt`/`m_pVehicleWeLookingAt`; если `O` — текущий `m_holder` → `detach_Actor()`, `m_holder=NULL`; `memory().remove_links(O)`, `m_pPhysics_support->in_NetRelcase(O)`, `HUD().net_Relcase(O)`.
- **`net_Relevant()` (806–816):** на сервере — `getSVU() | getLocal()`; на клиенте — `Local() & g_Alive()`.
- **`net_Save/net_SaveRelevant` (1773–1797):** `inherited::net_Save` + `m_pPhysics_support->in_NetSave` + `P.w_u16(m_holderID)`; `net_SaveRelevant → TRUE`.

## 7. Correction-prediction: `PH_B_CrPr` / `PH_I_CrPr` / `PH_A_CrPr` (931–1056)

Вызываются движком физики до/между/после шагов предсказания (`CrPr_*`-состояние, активация из `net_Import_Physic_proceed`).

- **`PH_B_CrPr` (931):** пропуск если активирован или `CrPr_GetActivationStep() > physics_world()->StepsNum()`. **Живой:** `CrPr_SetActivated(true)`, заполняет `IStart` (Pos/Vel/углы из `Position()`, `movement()->GetVelocity()`, `r_model_yaw`, `unaffected_r_torso`), запоминает `LastState` (get_State sync-объекта 0); `Local() && OnClient()` → `PHUnFreeze()` + `set_State(NET_A.back().State)`; иначе — `NET_Last = N`, камера `cam_Active()->Set(-yaw, pitch, 0)`; если `!N_A.State.enabled` → только `set_State`, иначе `PHUnFreeze()` + `set_State` + `g_Physics(N.p_accel, 0, 0)` + `Position().set(IStart.Pos)`. **Мёртвый:** если `PHGetSyncItemsNumber() == m_u16NumBones && !m_States.empty()` — `PHUnFreeze` и перенос состояний из `m_States` по костям.
- **`PH_I_CrPr` (1018):** если активен и жив — `pSyncObj->get_State(RecalculatedState)` (пересчитанное состояние сохраняется).
- **`PH_A_CrPr` (1035):** `get_State(PredictedState)` → `set_State(RecalculatedState)` (откат к пересчитанному); при `m_bInterpolate` — `mstate_wishful = mstate_real = NET_Last.mstate` и `CalculateInterpolationParams()`.

## 8. Интерполяция

### 8.1 `CalculateInterpolationParams()` (1059–1226)

- `IRec` ← `RecalculatedState` (Pos/Vel) + углы `NET_Last`; `IEnd` ← `PredictedState`, углы = IRec.
- **Касательные:** если уже в интерполяции — точка/производная текущего Безье-сплайна (`SCoeff/HCoeff` по `factor = (CurTime - m_dwIStartTime)/(m_dwIEndTime - m_dwIStartTime)`); иначе `HP1` = `(LastState.pos - LastState.previous_pos)/fixed_step` (или из `RecalculatedState` если скорость нулевая), `SP1 = SP0 + HP1`; `HP2 = (PredictedState.pos - previous)/fixed_step`, `SP2 = PredictedState.pos - HP2`.
- Ограничение касательных `TotalLen/3` (1182–1208).
- **Тайминг:** `ConstTime = (fixed_step - physics_world()->FrameTime())*1000 + Level().GetInterpolationSteps() * fixed_step*1000`; `m_dwIStartTime = m_dwILastUpdateTime`, `m_dwIEndTime = m_dwIStartTime + ConstTime`.
- **Коэффициенты:** `SCoeff[i][0..3]` (кубический Безье), `HCoeff[i][0..3]` (Эрмита) (1210–1221).
- `m_bInInterpolation = true`; `m_pPhysicsShell->NetInterpolationModeON()`.

### 8.2 `make_Interpolation()` (1230–1325)

Вызывается каждый кадр: `m_dwILastUpdateTime = Level().timeServer()`.
- **Истёк `m_dwIEndTime`:** `m_bInInterpolation=false`, `mstate_real/mstate_wishful = NET_Last.mstate`, `NET_SavedAccel = NET_Last.p_accel`, `set_State(PredictedState)`.
- **Иначе:** `factor`; `NewPos.lerp(IStart.Pos, IEnd.Pos, factor)` (линейный), торс `angle_lerp`; позиции по трём кривым: `IPosL` (линейная), `IPosS` (Безье `SCoeff`), `IPosH` (Эрмита `HCoeff`). Переключатель **`g_cl_InterpolationType`**: 0 → линейная позиция и скорость `(IEnd - IStart)/Δt`; 1 → Безье + производная; 2 → Эрмита + производная; иначе `R_ASSERT2(0, "Unknown interpolation curve type!")`. Далее `movement()->SetPosition/SetVelocity`, `cam_Active()->Set(-unaffected_r_torso.yaw, pitch, 0)`.
- В DEBUG — наполнение стеков `LastPosS/H/L` (лимит `g_cl_InterpolationMaxPoints`).
- `UpdatePosStack`/`FindMemoryPos` (1328–1370) — **закомментированы целиком**.

## 9. `save` / `load` (1372–1405)

Сериализация игры (не сетевого обмена): `inherited::save/load` + `CInventoryOwner`; `m_bOutBorder` (u8); 4 флага `CUITaskWnd` (Treasures/QuestNpcs/SecondaryTasks/PrimaryObjects); `g_quick_use_slots[0..3]` (stringZ).

## 10. `UpdateCL()` (`Actor.cpp:1128–1406`) — детали

Ежекадровый клиентский апдейт (каркас вызовов — порц. 19 §3.6):

1. **Pickup-режим** (1130–1148): если жив, `CurrentViewEntity() == this`, нет блокирующего `TopInputReceiver()->StopAnyMove()` и нет `m_holder` — при зажатом `kUSE` (`get_action_dik(kUSE, 0/1)` + `iGetAsyncKeyState`) и `AF_MULTI_ITEM_PICKUP` → `m_bPickupMode = true`; иначе false.
2. `UpdateInventoryOwner(dwTimeDelta)`; обновление боксов `feel_touch`-объектов (`UpdateObjectBox(PHCharacter())`) при `m_feel_touch_characters > 0`; `m_holder->UpdateEx(currentFOV())`.
3. `m_snd_noise -= 0.3f*fTimeDelta`; `inherited::UpdateCL()`; `m_pPhysics_support->in_UpdateCL()`.
4. `PickupModeUpdate()` (если жив) → `PickupModeUpdate_COD(pickup_result)` (порц. 24).
5. `SetZoomAimingMode(false)`; `cam_Update(dt, currentFOV())`; `m_legs_controller.update(this)`.
6. **HUD/дисперсия** (1188–1268): для текущего entity — флаги `HUD_CROSSHAIR_RT2/HUD_DRAW_RT`; активный `CWeapon`: при зуме — `CEffectorZoomInertion::SetParams(GetFireDispersion(true))` + `SetZoomAimingMode(true)`; прицел: `m_fdisp_controller.SetDispertion/GetCurrentDispertion`, `HUD().SetCrosshairDisp/ShowCrosshair`, `HUD_WEAPON_RT/HUD_CROSSHAIR_RT2/HUD_DRAW_RT` (учёт lookout в MP), `pWeapon->UpdateSecondVP()`, шейдер-константы `hud_params`/`hud_fov_params` (`GetZRotatingFactor/GetSecondVPZoomFactor/GetHudFov/CurrentZoomFactor/GetMinScopeZoomFactor`); для `CWeaponStatMgun` без зума — дефолтный прицел; без оружия — сброс констант и `Device.m_SecondViewport.SetSVPActive(false)`.
7. `UpdateDefferedMessages()`; `CStepManager::update(...)` (жив); `spatial.type |= STYPE_REACTTOSOUND`.
8. **`m_sndShockEffector`** (1277–1288): update только для `CurrentViewEntity`, удаление при отсутствии работы/смерти.
9. `Cameras().hud_camera_Matrix` → `g_player_hud->update(trans)` при `IsFocused()`; `AF_MULTI_ITEM_PICKUP` → `m_bPickupMode = false`.
10. **Discord** (1298–1395, при `rsDiscord`): godmode/health/time; при первом обновлении после лоадскрина — Lua `_g.IsHardcoreMode`, `ironman_manager.get_lives_left`, уровень, режимы (`_g.IsStoryMode/IsWarfare/IsAzazelMode/IsSurvivalMode`), `GameTaskManager().RPC_UpdateTaskName()`, `RPC_UpdateFaction/Rank/Reputation`.
11. **LV-шейдеры** (1397–1400): `g_pGamePersistent->actor_data.health/stamina/bleeding` (из `conditions()`); при скрытом HUD — `get_hud_states()->UpdateZones()`.

## 11. `RPC_UpdateFaction/Rank/Reputation` (`Actor.cpp:1421–1475`)

Lua-функторы `_g.get_actor_true_community`, `ranks.get_player_rank_name`, `ranks.get_player_reputation` + `alun_utils.get_reputation_name` → заполнение `discord_gameinfo.faction/faction_name/rank_name/reputation` (через `CStringTable().translate`). Вызов из `UpdateCL` (1389–1391), `InventoryOwner.cpp:452`, `level_script.cpp:1184`.

## 12. Видимость актёра (`Actor.cpp:2094–2182`)

- **Свободные функции** (только SP): `SetActorVisibility(u16 who_id, float value)` → `SetVisibilityFromObject`; `GetActorVisibility()` → `GetVisibility()` с clamp [0,1]. Вызывающие: `CustomMonster.cpp:740/886`, `poltergeist.cpp:228`, `visual_memory_manager.cpp:788/798`.
- **`CActor::GetVisibility()` (2122):** при `m_visibility_changed` — сортировка `m_npc_visibility` и выбор max (`back().value`), иначе 0; далее экспоненциальное сглаживание `m_visibility_cur` к `m_visibility_tgt` по `Device.fTimeDelta`.
- **`SetVisibilityValue(float)` (2151):** прямая установка `m_visibility_tgt` (вызывается из UI `UIMainIngameWnd.cpp:298`).
- **`SetVisibilityFromObject(u16 who_id, float)` (2156):** upsert/erase записи `{id, value}` в `m_npc_visibility` (0 → удаление), `m_visibility_changed = true`.
- Lua-доступ: `get_actor_visibility`/`get_actor_ui_luminosity` (`script_game_object_script3.cpp:639–640`, `CScriptGameObject::GetActorVisibility` 2639); UI `UIMotionIcon.cpp:81`.

## 13. Hit-события и статистика (`Actor_Network.cpp:1800–2028`)

| Метод | Строки | Назначение |
|---|---|---|
| `SetHitInfo(who, weapon, element, Pos, Dir)` | 1800–1809 | Сохранение `m_iLastHitterID/m_iLastHittingWeaponID/m_s16LastHittedElement/m_fLastHealth/m_bWasHitted/m_vLastHitDir/Pos` |
| `OnHitHealthLoss(float)` | 1811–1831 | MP+server: `GE_GAME_EVENT`/`GAME_EVENT_PLAYER_HITTED` (потеря здоровья от последнего бьющего) |
| `OnCriticalHitHealthLoss()` | 1834–1923 | MP+server: `GAME_EVENT_PLAYER_KILLED` (`KT_HIT`); определение `SpecialHit`: `SKT_KNIFEKILL` (нож), `SKT_HEADSHOT` (head-бон + `CWeaponMagazined` → ещё `GEG_PLAYER_PLAY_HEADSHOT_PARTICLE`), `SKT_EYESHOT`, обход родительских костей до головы, `SKT_BACKSTAB` (`m_bWasBackStabbed`); в MP — `m_WeaponUsageStatistic->OnBullet_Check_Result(true)` |
| `OnPlayHeadShotParticle(NET_Packet)` | 1925–1943 | Чтение element/dir/pos, инверсия dir, `CParticlesPlayer::MakeXFORM` + создание `m_sHeadShotParticle` → `GamePersistent().ps_needtoplay` (обработчик в `Actor_Events.cpp:340`) |
| `OnCriticalWoundHealthLoss()` | 1945–1963 | `GAME_EVENT_PLAYER_KILLED`/`KT_BLEEDING` (кровопотеря) |
| `OnCriticalRadiationHealthLoss()` | 1965–1979 | `GAME_EVENT_PLAYER_KILLED`/`KT_RADIATION` |
| `Check_for_BackStab_Bone(u16)` | 1981–1992 | true для `m_head/m_neck/m_spine2/m_l_clavicle/m_r_clavicle/m_spine1/m_spine` |
| `InventoryAllowSprint()` | 1994–2005 | false если активный предмет или вышмод `!IsSprintAllowed()` |
| `BonePassBullet(int)` | 2007–2020 | SP → `inherited`; MP: без вышмода — параметр кости `(param(3) > 0.5f)`, иначе `pOutfit->BonePassBullet` |
| `On_B_NotCurrentEntity()` | 2022–2028 | `inventory().Items_SetCurrentEntityHud(false)` |

## 14. DEBUG-рендер сети

`dbg_draw_piramid` (1410), `OnRender_Network()` (1474+, только `#ifdef DEBUG`): отрисовка `dbg_net_Draw_Flags`-флагов (автопикап-бокс `m_AutoPickUp_AABB`, скелетные формы `CCF_Skeleton` box/cylinder/sphere, пирамиды из `NET_Last.p_pos/p_velocity` 1555). Стеки `LastPosS/H/L` в `make_Interpolation`.

## 15. Связи

- **Сетевые события (порц. 29–30):** `CActorMP` переопределяет `net_Export/net_Import/net_Relevant` (свои state-холдеры).
- **`g_PerformDrop`-связь (порц. 27):** drop идёт локально через input (`kDROP`) — сетевой синхронизации не имеет.
- **Физика (ит. 7):** `PH_*_CrPr`, `AddObject_To_Objects4CrPr`, `g_Physics(N.p_accel, 0, 0)` (порц. 23), `fixed_step`, `physics_world()`.
- **Камера (порц. 32):** `cam_Active()->Set(...)` в `PH_B_CrPr`/`make_Interpolation`, `m_pActorEffector` (`CActorCameraManager`).
- **Holder (порц. 26):** `m_holderID` в `net_Save`/`net_Spawn`, `client_spawn_manager` + `on_requested_spawn`.
- **Смерть (порц. 22):** `net_ExportDeadBody`-квантование, `net_Spawn` при мёртвом (`death_init`).
- **Lua/Discord:** `_g.get_actor_true_community`, `ranks.*`, `_g.IsHardcoreMode` и др., `RPC_Update*`.
- **UI/HUD (порц. 32):** `CUITaskWnd` в save/load, `HUD().SetCrosshairDisp`, `UpdateZones`.
- **Условия (порц. 20):** `conditions().GetPower/BleedingSpeed` в LV-данные, `UpdateInventoryOwner`.

## 16. Краткое резюме

Сеть игрока: `net_Export` пишет health/time/позицию/углы/команду/mstate/скорость/радиацию/слот + физ. состояние (или квантованный труп через `net_ExportDeadBody`); `net_Import` раскладывает пакет на базовый (`NET`, ≤5 записей с проверкой таймстемпов) и физический (`NET_A` для живого / `m_States` для мёртвого) и ставит объект в очередь correction-prediction (`AddObject_To_Objects4CrPr`). `PH_B/I/A_CrPr` снимают/восстанавливают состояния вокруг шагов предсказания, `CalculateInterpolationParams` строит Безье/Эрмита-сплайны (`SCoeff/HCoeff`, окно `ConstTime`), `make_Interpolation` прогоняет их по `g_cl_InterpolationType` (0/1/2) и двигает персонажа/камеру. `UpdateCL` — клиентский такт (pickup, инвентарь, HUD-дисперсия и прицел, шаги, шок-эффектор, Discord/RPC, LV-шейдеры). Плюс save/load, видимость актёра (сглаживание по `m_npc_visibility`), hit-события килфидов/статистики (MP) и DEBUG-отладка сети.
