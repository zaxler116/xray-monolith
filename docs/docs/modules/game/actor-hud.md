# CActor: HUD и рендер (порцион 32)

> **Модуль:** `xrGame` · **Файлы:** `Actor.cpp` (1100–1124 FOV, 1128–1406 `UpdateCL` HUD-часть, 2184–2365 рендер, 2401–2540 HUD/индикаторы, 2542–2566 физ.-позиция), `Actor.h` (337–338, 678–684), `player_hud_legs.cpp` (12), консольные команды (`console_commands.cpp:193/136/3189/3273`) · **Порцион 32, итерация 5**
>
> **Страница:** рендер тела актёра (первый/третий взгляд, ноги, тени), HUD прицела/перекрести, скриптовые камера-вложения, мировые индикаторы/текст над головой. Связи: `UpdateCL` (порц. 28 — каркас движения/сети), `SetPhPosition`/`ForceTransform` (порц. 23), `CHudItem`/`CHUDManager`/`HUD()` (ит. 10), `g_player_hud`/`m_legs_controller` (порц. 31), `CWeapon`-зум/дисперсия (ит. 9), `script_attachment` (Lua), `CharacterPhysicsSupport` (ит. 7).

## 1. Ответственность

Все визуальные аспекты актёра вне костей (порц. 31): выбор FOV прицела, ежекадровые HUD-обновления (перекрестие, дисперсия, SecondVP/шейдер-константы, Discord, данные для LV-шейдеров), рендер тела в первом лице (ноги/показ тела/тени), рендер прикреплённых к камере скриптовых объектов, индикаторы и имена игроков над головой (MP).

## 2. `g_fov` и `currentFOV()` (`Actor.cpp:1100–1124`)

`float g_fov = 55.0f;` (1100), `extern float g_ironsights_factor` (1101).

```cpp
float CActor::currentFOV()
```

- Если выключены флаги `HUD_WEAPON | HUD_WEAPON_RT | HUD_WEAPON_RT2` → `g_fov`.
- `eacFirstEye` + активное зум-оружие (+ условие по `ZoomTexture`/`IsRotatingToZoom`): `GetZoomFactor() == 0` → **iron sights**: `atan(tan(g_fov/2) / g_ironsights_factor)` (в радианах-градусах формулы), иначе `GetZoomFactor() * 0.75`.
- Во всех остальных случаях — `g_fov`.

Используется в `UpdateCL` → `m_holder->UpdateEx(currentFOV())` (1165) и `cam_Update(dt, currentFOV())` (1181).

## 3. `UpdateCL`: HUD-часть (`Actor.cpp:1128–1406`)

Общий каркас (pickup по `kUSE`-dik 1130–1148, feel-touch боксы 1152–1163, `PickupModeUpdate` 1172–1176, `inherited::UpdateCL`/физика 1169–1170) — порц. 23/28. **HUD-часть** (1178–1405):

1. **Зум и камера** (1178–1204): `SetZoomAimingMode(false)`; при `pWeapon->IsZoomed()` — `CEffectorZoomInertion::SetParams(GetFireDispersion(true))` (effector `eCEZoom`) и `SetZoomAimingMode(true)`.
2. **Перекрестие и дисперсия** (1188–1229, только для `Level().CurrentEntity`):
   - флаги `HUD_CROSSHAIR_RT2`/`HUD_DRAW_RT` (1188–1192);
   - `m_fdisp_controller.SetDispertion(GetFireDispersion(true,true))` → `GetCurrentDispertion()` → `HUD().SetCrosshairDisp(disp, 0.02f)` (не в кадре SecondVP) и `HUD().ShowCrosshair(use_crosshair())` (1208–1216); DEBUG — `SetFirstBulletCrosshairDisp`;
   - `HUD_WEAPON_RT`/`HUD_CROSSHAIR_RT2`/`HUD_DRAW_RT` по `mcLookout` (MP), `show_crosshair`, `show_indicators` (1221–1229).
3. **SecondVP и шейдер-константы** (1231–1241): `pWeapon->UpdateSecondVP()`; `hud_params = (GetZRotatingFactor, GetSecondVPZoomFactor, GetHudFov, IsSVPFrame)`; `hud_fov_params = (CurrentZoomFactor, GetMinScopeZoomFactor)`.
4. **Ветки без оружия** (1244–1268): `CWeaponStatMgun` (fixed crosshair 0.05, обнуление `hud_params`/`m_blender_mode`) либо базовая — `SetCrosshairDisp(0)`, `ShowCrosshair(CROSSHAIR_SHOW_ALWAYS)`, полная очистка `hud_params/hud_fov_params/m_blender_mode`, `Device.m_SecondViewport.SetSVPActive(false)`.
5. **Прочее** (1270–1296): `UpdateDefferedMessages()` (порц. 30 §20), `CStepManager::update` (шаги), `spatial.type |= STYPE_REACTTOSOUND`, `m_sndShockEffector->Update`/удаление (1277–1288), `g_player_hud->update(hud_camera_Matrix)` при `IsFocused()` (1289–1293), сброс `m_bPickupMode` при `AF_MULTI_ITEM_PICKUP`.
6. **Discord** (1298–1395, `psDeviceFlags2.test(rsDiscord)`): godmode/health (`roundf(GetfHealth()*100)`), игровое время, после лоадскрина (`ex_update`): ironman (`_g.IsHardcoreMode` + `ironman_manager.get_lives_left`), уровень (`level_name`/случайный `level_icon_index` 1–3), режим истории/warfare/azazel (`azazel_mode.get_possessed_lives`)/survival/freeplay → `gamemode`, `RPC_UpdateTaskName` + **`RPC_UpdateFaction/Rank/Reputation`** (порц. 30 §19), `ex_update = false`.
7. **LV-шейдеры и зоны** (1397–1405): `g_pGamePersistent->actor_data.{health,stamina,bleeding}` ← `GetfHealth()/conditions().GetPower()/BleedingSpeed()`; если HUD выключен (`!HUD_DRAW`) → `UIMainIngameWnd->get_hud_states()->UpdateZones()` (радиация среды).

## 4. `RenderCamAttached()` (`Actor.cpp:2184–2209`)

Только `cam_active == eacFirstEye` и `::Render->active_phase() == 0`, если есть `m_script_attachments` (`CGameObject::GetAttachments()`, GameObject.h:350):

- `Render->set_HUD(TRUE)`, `set_CamAttached(TRUE)`, `set_Object(this)`;
- матрица камеры `Cameras().camera_Matrix(cam)` → для каждого `script_attachment` с типом **`eSA_CamAttached`** — `att->Render(nullptr, &cam)`;
- сброс `set_CamAttached(FALSE)`, `set_HUD(FALSE)`.

Рисует Lua-вложения, привязанные к камере (прицелы/навески, `level_script.cpp:2374 AddAttachment`, `script_attachment_manager.cpp`); аналогичные обходы есть в `HUDManager.cpp:218/237` и `player_hud.cpp:943/978`.

## 5. `AllowActorShadow()` (`Actor.cpp:2215–2227`)

`bool CActor::AllowActorShadow()` — **false**, если:

1. `!r__actor_shadow_in_demo_record && !pDemoRecords.empty()` (тень отключена при демо-записи);
2. `!r__actor_shadow_in_demo_record && m_FPCam` (FP-камера);
3. `!ps_actor_shadow_flags.test(1)` (глобальный флаг теней актёра);
4. `Render->get_generation() != GENERATION_R2` (только R2);
5. активен фонарь-детектор: `CFlashlight` в `DETECTOR_SLOT` с `torch_active()`.

Иначе `true`. Глобалы: `r__actor_shadow_in_demo_record = TRUE` (2214), `ps_actor_shadow_flags` (2213).

## 6. `canRenderLegs(CActor*, CHolderCustom*)` (`Actor.cpp:2235–2244`)

Свободная функция (`noexcept`), условия показа ног в первом лице:

```cpp
g_legs_enabled                                    // консоль g_legs (player_hud_legs.cpp:12)
&& (legs_in_low_crouch || !(mcCrouch && mcAccel)) // не низкий присед с ускорением
&& g_player_hud && !m_holder                      // HUD-руки есть, не в транспорте
&& (legs_in_demo_record || pDemoRecords.empty())  // не демо-запись
&& showActorBody == 0                             // консоль show_actor_body (Actor.cpp:114)
&& !actor->m_FPCam                                // не FP-камера
```

Флаги: `legs_in_demo_record = FALSE`, `legs_in_low_crouch = FALSE`, `legs_render_attachments_shadow = TRUE` (2230–2232), `g_legs_enabled` extern (2233).

## 7. `renderable_Render()` (`Actor.cpp:2246–2354`)

`VERIFY(_valid(XFORM()))`. `validRendererForShadow` = R2 и **не DX9** (`dx_level != 0x00090000`) (2251).

### 7.1 Первый взгляд (`eacFirstEye`), фаза 0 (2253–2274)

- `canRenderLegs(...)` → `m_legs_controller.render()` (2257–2260);
- `showActorBody == 1 || 2` → `inherited::renderable_Render()`; при `showActorBody == 2` дополнительно `CInventoryOwner::renderable_Render()` (предметы) (2262–2270).

### 7.2 Первый взгляд, остальные фазы — тень (2275–2345)

`else if (AllowActorShadow())`, только при `validRendererForShadow`:

- **Кадро-адаптация** (2279–2285): `static u32 renderFrame`; `needAdjust` один раз за кадр — вычисляется вектор `diff = XFORMShadow.c - XFORM().c` (нормализованный, `magnitude` хранится в `m`) (2289–2293) для сдвига объектов в теневую позицию.
- **При `canRenderLegs`** (2287–2333): `m_legs_controller.update(this, true)` + `render()` (полное тело без сокрытия костей — корректность тени); активный предмет — `XFORM().c.mad(diff, m)` + `renderable_Render()` (2301–2308); `m_attached_objects` (фонарь и пр.) — при `legs_render_attachments_shadow` со сдвигом (2311–2320); **болт** в `BOLT_SLOT` — только сдвиг `XFORM().c` (2323–2332).
- **Иначе / невалидный рендерер** (2334–2344): `inherited::renderable_Render()` + `CInventoryOwner::renderable_Render()`.

### 7.3 Третий взгляд (2349–2353)

`inherited::renderable_Render()` + `CInventoryOwner::renderable_Render()` (тело + предметы инвентаря).

## 8. `renderable_ShadowGenerate()` (`Actor.cpp:2356–2365`)

- `m_holder` (транспорт) → `FALSE` (генерация тени отключена — тень рисует транспорт);
- `GENERATION_R1 && eacFirstEye && canRenderLegs(...)` → `FALSE` (в R1 от первого лица ноги тени не генерируют);
- иначе — `inherited::renderable_ShadowGenerate()`.

## 9. `OnHUDDraw(CCustomHUD*)` (`Actor.cpp:2401–2438`)

`R_ASSERT(IsFocused())`. HUD-модель рисуется, если **не** (`mcLookout` в MP) и (`!m_FPCam || m_FPCam->hudEnabled`) → `g_player_hud->render_hud()` (2405–2406). Далее — отключённый (`#if 0`) отладочный блок: позиция/скорость/окружение физики и ввод (`ICoincidenced/IReceived`). Декларация — `Actor.h:337`.

## 10. `RenderIndicator` (`Actor.cpp:2440–2488`)

`virtual` (`Actor.h:683`). При `g_Alive()`:

- `UIRender->StartPrimitive(4, ptTriStrip, pttLIT)` — биллборд-квад у головы: `CalculateBones()` → позиция `M.c = XFORM() * LL_GetTransform(m_head) + dpos`;
- полуоси из `Device.vCameraTop/Right` × `r1/r2` → 4 точки (UV 0..1), `CacheSetXformWorld(Fidentity)`, шейдер `IndShader`, `FlushPrimitive()`.

**Вызовы (MP):** `game_cl_deathmatch.cpp:975` (индикатор неуязвимости `InvincibleShader`), `game_cl_capture_the_artefact.cpp:1664` (`IndicatorShader`), `game_cl_teamdeathmatch.cpp:580`.

## 11. `RenderText` (`Actor.cpp:2494–2540`)

`virtual` (`Actor.h:684`). При `g_Alive()` — текст над головой (имя игрока):

- позиция головы, вертикальный сдвиг: `size` = экранный размер (проекция `M.c` и `M.c + vCameraTop`), константы `mid_size = 0.097f`, `upsize = 0.33f`, `fontsize = 15.0f` (2490–2492): `delta_up = upsize` при `size < mid_size`, иначе `upsize * (mid_size/size)` (2515–2518);
- отсечение по NDC (`z/w < 0`, за экраном — return) (2527–2528);
- `pFontArial14`, `alCenter`, `SetColor(color)`, `Out(x, y, Text)`, возврат `*pdup = delta_up` (сдвиг для следующей строки).

**Вызовы:** `game_cl_capture_the_artefact.cpp:1659`, `game_cl_teamdeathmatch.cpp:571` (`PLAYER_NAME_COLOR`).

## 12. `SetPhPosition` / `ForceTransform` (`Actor.cpp:2542–2566`)

- **`SetPhPosition(transform)`** (2542–2549): если нет `m_pPhysicsShell` → `character_physics_support()->movement()->SetPosition(transform.c)` — синк позиции физики (порт сетевых позиций, **порц. 23**).
- **`ForceTransform(m)`** (2551–2566): `character_physics_support()->ForceTransform(m)`; в MP (`!IsGameTypeSingle()`) — `movement()->BlockDamageSet(u64(2.f / fixed_step))` (2 секунды защиты от урона при телепорте/пересадке).

## 13. Связи

- **Порц. 28** — `UpdateCL`-каркас (pickup/сети/Discord `discord_gameinfo`, `RPC_*` → порц. 30 §19), `m_sndShockEffector`.
- **Порц. 23** — `SetPhPosition`/`ForceTransform`, `cam_Update`, `mstate_real`/`mcLookout`.
- **Порц. 31** — `m_legs_controller` (IK-ноги, `g_legs_enabled`), `g_player_hud` (FP-руки), шаги `CStepManager`.
- **Ит. 10** — `HUD()` (`CCustomHUD`), `CHUDManager` (вызов `OnHUDDraw`), `CInventoryOwner::renderable_Render` (предметы), `CHudItem`-флаги HUD_WEAPON.
- **Ит. 9** — `CWeapon`: `IsZoomed/GetZoomFactor/GetFireDispersion/UpdateSecondVP/ZoomTexture/GetHudFov/show_crosshair/use_crosshair/show_indicators`, `CWeaponStatMgun`.
- **Ит. 7** — `CharacterPhysicsSupport::movement` (`SetPosition`/`BlockDamageSet`), `CPhysicsShell`.
- **Скриптовые вложения** — `CGameObject::m_script_attachments`/`add_attachment` (GameObject.h:67/185), `script_attachment::Render` (script_attachment_manager.cpp:169), Lua `AddAttachment` (level_script.cpp:2374), обходы в `HUDManager`/`player_hud`.
- **Ит. 12 (рендер)** — `IRender->set_HUD/set_CamAttached/set_Object`, `GENERATION_R1/R2`, `XFORMShadow`, `active_phase()`.
- **Консольные команды** — `g_legs` (0..1), `show_actor_body` (0..2), `r__actor_shadow_in_demo_record`, `ps_actor_shadow_flags`.

## 14. Краткое резюме

HUD и рендер актёра: `currentFOV()` (55° база + iron sights/zoom-фактор оружия) и HUD-часть `UpdateCL` — перекрестие/дисперсия (`m_fdisp_controller`), зум-эффектер, SecondVP и шейдер-константы `hud_params/hud_fov_params`, Discord-rich-presence, данные для LV-шейдеров и радиационных зон. Рендер тела: первое лицо — ноги через `m_legs_controller` (глобалы `g_legs`/`show_actor_body`), тени в R2 (не DX9) со сдвигом предметов/вложений к `XFORMShadow`, `AllowActorShadow` (запросы: демо/FP-камера/флаг/фонарь), транспорт отключает `renderable_ShadowGenerate`; третье лицо — тело+предметы. `RenderCamAttached` рисует Lua-вложения типа `eSA_CamAttached` в фазе 0. MP-мировые индикаторы/имена (`RenderIndicator`/`RenderText` — биллборд у головы, вызовы из deathmatch/CTA/TDM) и `SetPhPosition`/`ForceTransform` (порц. 23) завершают обзор.
