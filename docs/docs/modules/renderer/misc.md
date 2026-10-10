# Renderer: прочее (stats / env / fx / debug / 3DFluid)

Второй блок порциона 17: статистика и графики, окружение (skybox/облака), флэры и молнии, debug-рендер, экран загрузки, GIF-плеер и воксельный fluid (3DFluid). UI-ядро (`dxUIRender` и др.) — на странице [UI-обвязка](dx-bridges.md).

## Ответственность

| Класс / файл | Роль |
| --- | --- |
| `dxStatsRender` (`IStatsRender`) | Вывод статистики `RCache.stat` в консоль |
| `dxStatGraphRender` (`IStatGraphRender`) | Отрисовка `CStatGraph` (бары/кривые/markers) |
| `stats_manager` | Учёт GPU-памяти буферов/RT |
| `dxEnvironmentRender` + env-descriptors | Skybox, облака, mixer-lerp описаний окружения |
| `dxLensFlareRender` / `dxFlareRender` | Отрисовка лins-фляров `CLensFlare` |
| `dxThunderboltRender` / `dxThunderboltDescRender` | Молнии (detail-model + градиенты) |
| `dxDebugRender` / `RDebugRender` | Пакетная отрисовка debug-линий |
| `dxApplicationRender` | Экран загрузки (progress, фон, логотип уровня) |
| `dxUISequenceVideoItem` | Обёртка `CTexture` для UI-видео |
| `CGIFAnimationPlayer` | GIF-анимации для UI |
| `dxObjectSpaceRender` (**DEBUG only**) | Wireframe-отладка collision-boxes |
| `3DFluid` (`xrRenderDX10/3DFluid/`) | Воксельная симуляция дыма/огня (активная) |

## Место в архитектуре

- Все `I*Render`-реализации создаются `dxRenderFactory` ([factory.md](factory.md)), данные — из `CApplication`/`CEnvironment`/`CLensFlare`/`CStatGraph` (xrEngine, итерация 2; устройство `CEnvironment` — [environment](../xr-engine/environment.md)).
- `RCache`/`HW`/`StateManager` — [Устройство рендера](render-device.md); `shader.create` — [Shader Bus](shader-bus.md).
- 3DFluid — подсистема `xrRenderDX10`, живёт отдельно от `RCache` (свои 3D-RT, свои blenders), встраивается в сцену через визуалы (`FHierrarhyVisual`, [Визуалы](visuals.md)) и `CRender::Load3DFluid` ([R4: scene](r4-scene.md)).
- HOM (hierarchical occlusion) — поле `CRender::HOM` (`CHOM`), устройство — [Occlusion](occlusion.md), здесь не дублируется.

## Публичный API

### `dxStatsRender`

- `OutData1(F)` — `VERT/POLY/DIP` (`RCache.stat.verts/polys/calls`),
- `OutData2(F)` — **DEBUG only**: states/textures/matrices/constants, RT/PS/VS, DCL/VB/IB,
- `OutData3(F)` — xforms,
- `OutData4(F)` — разборка вертексов/dips: static / flora (+lods) / dynamic (sw/inst/1B..4B) / details (`RCache.stat.r.s_*`),
- `GuardVerts(F)` (>500k), `GuardDrawCalls(F)` (>1k) — предупреждения,
- `SetDrawParams(pRender)` — world=identity + `m_SelectionShader` + `tfactor=1` (подготовка к стат-отрисовке).

### `dxStatGraphRender`

- `OnDeviceCreate/Destroy` — `hGeomLine`/`hGeomTri` (`FVF::F_TL0uv`),
- `OnRender(CStatGraph&)` — см. «Внутреннее устройство».

### `stats_manager`

Функции `increment_stats_*`/`decrement_stats_*` для vertex/index/rtarget-буферов (по 4 пула: default/default2/rtarget/rtarget2). **Все операции no-op при `g_dedicated_server`**. `decrement` использует трюк `AddRef()` + `Release()` для проверки refcnt: декремент только если буфер реально уничтожается (refcnt==1). DX10/11: пул всегда `D3DPOOL_MANAGED` (понятия «пул» нет), размер — из `GetDesc`. `get_format_pixel_size(D3DFORMAT)` — большой switch (DX9); `get_format_pixel_size(DXGI_FORMAT)` — range-based (DX10/11). DEBUG: `m_buffers_list` отслеживает каждый буфер, деструктор `Msg` оставшиеся (`R_ASSERT` закомментирован); `R_ASSERT` на decrement при отсутствии буфера.

### `dxEnvironmentRender` / env-descriptors

```cpp
// IEnvironmentRender
void OnLoad/OnUnload(CEnvironment& env);      // tonemap = DEV->_CreateTexture("$user$tonemap")
void OnFrame(CEnvironment& env);
void RenderSky(CEnvironment& env, bool OnlyMV = false);
void RenderClouds(CEnvironment& env);

// IEnvDescriptorRender — sky_texture / sky_texture_env / clouds_texture
// IEnvDescriptorMixerRender — lerp(A,B): sky_r_textures={(0,A),(1,B)}, то же _env/clouds; Clear(); Destroy()
```

`CBlender_skybox` (внутренний `IBlender`): `Compile` — E[0] `r_Pass("sky2","sky2")` + `s_sky0/s_sky1 "$null"` + `s_tonemap "$user$tonemap"` (`//. hack`) + `PassSET_ZB(FALSE,FALSE)`; E[1] `ssfx_sky_mv` (DX10/11, motion vectors). Конструктор: `tsky0/tsky1 = DEV->_CreateTexture("$user$sky0/1")`.

### `dxLensFlareRender` / `dxFlareRender`

- `dxFlareRender` (`IFlareRender`) — держатель шейдера на фляр (`CreateShader/DestroyShader`, `hShader` public).
- `dxLensFlareRender::Render(CLensFlare&, bSun, bFlares, bGradient)` — один lock `MAX_Flares(24)×4` LIT-вертексов, затем per-квад draw.

### `dxThunderboltRender` / `dxThunderboltDescRender`

Desc — держатель `l_model` (`IRender_DetailModel*`); `CreateModel` грузит из `$game_meshes$` через `RImplementation.model_CreateDM` (`R_ASSERT "Empty 'lightning_model'"`), `DestroyModel` → `model_Delete`.

### `dxDebugRender` / `RDebugRender`

```cpp
void add_lines(Fvector const*, u32 vc, u16 const* pairs, u32 pc, u32 color, bool bHud = false);
void Render();                    // hud-линии (HUD-проекция) + world-линии
void SetShader(const debug_shader&);   // (dxUIShader*)&*shader → hShader
void SetDebugShader(dbgShaderHandle);  // статическая таблица: dbgShaderParams[0]={"hud\\default","ui\\ui_pop_up_active_back"}
void SetAmbient(u32);             // DX10/11: VERIFY(!"Not implemented for DX10")
void NextSceneMode();             // DX10/11: no-op
```

`RDebugRender` — наследник `dxDebugRender` + `pureRender`, регистрируется в `Device.seqRender` с приоритетом `REG_PRIORITY_LOW − 100`; глобал `rdebug_render` указывает на статический экземпляр.

### `dxApplicationRender`

`LoadBegin` (geom + `sh_progress` `"hud\\default","ui\\ui_actor_loadgame_screen"` + `hLevelLogo_Add` + Discord-флаги), `destroy_loading_shaders`, `setLevelLogo(name)`, `load_draw_internal(CApplication&)`, `KillHW()`.

### `dxUISequenceVideoItem`

Тривиальная обёртка над `CTexture*`: `CaptureTexture()` = `RCache.get_ActiveTexture(0)`, `video_*` пробрасывает в текстуру.

### `CGIFAnimationPlayer`

`Load(fname)` (через `CGIFResource` xrEngine), `UpdateFrame()`, `Play()`, `Stop()`, `IsValid/IsPlaying`, `GetActiveFrame`, `GetMemUsed`.

## Внутреннее устройство

### `dxStatGraphRender::OnRender`

`RCache.OnFrameEnd()` → `RenderBack` (задник-квад, рамка-linestrip, сетка) → подсчёт элементов в два прохода: `stBar` ×4 вертекса (TRIANGLELIST), `stCurve` ×2 / `stBarLine` ×4 (LINELIST; `stPoint` = 0, закомментирован), один lock/fill/unlock/draw per class → markers (`stVert`/`stHor`, mapping `elem_offs`/`elem_factor`).

Quirks:
- `RenderBack`: `Num_H_LinesDwn = (owner.grid.y < PNum_H_LinesUp) ? owner.grid.y : PNum_H_LinesDwn` — **копи-паста**: условие сравнивается с `PNum_H_LinesUp`, а не `PNum_H_LinesDwn`.
- Циклы `RenderLines`/`RenderBarLines`/`RenderMarkers`: `it != pelements->end() + 1` — **out-of-bounds-сравнение итераторов** (работает только потому, что цикл останавливается на `end()` раньше).
- `RenderBarLines` рисует 4 вертекса на сегмент (T-образный штрих).

### `dxEnvironmentRender::RenderSky`

Ленивое re-create при `env.bNeed_re_create_env` (bug-fix «sky box stretch fix»: верхи дублируются, чтобы верхняя текстура не растягивалась). `rmFar()`, `mSky = rotateY(sky_rotation) × translate(camPos)`; **half-box** 12 вертексов / 20 трианглов (статические таблицы `hbox_verts`/`hbox_faces`); цвет = `sky_color × 255`, alpha = `weight × 255`. `OnlyMV` → E[1] `ssfx_sky_mv` + `set_xform_world_prev(mSky_prev)` (motion vectors для TAA); иначе E[0] + `set_Textures(&mixRen.sky_r_textures)`. Далее `rmNormal()`; если `!OnlyMV`: **хак `set_Z(FALSE); set_Z(TRUE);`** («state may not be set by RCache») + `env.eff_LensFlare->Render(TRUE, FALSE, FALSE)` (солнечный фляр) и `set_Z(FALSE)` после.

**Активные вызовы в R4**: `phase_combine` → `RenderSky()` + `RenderClouds()` (до combine, без Z-test — «Igor: to avoid siluets», [R4: post-process](r4-postprocess.md)); MV-pass `RenderSky(true)` — в `r4_R_render.cpp:360`; skybox в split-scene ветке `if(0)` — **мёртв** (`r4_R_render.cpp:476`).

`RenderClouds`: `rmFar()`; `mXFORM = rotateY(sky_rotation) × mulB_43(scale(10,0.4,10)) × translate(camPos)`; направление ветра кодируется двумя HP-направлениями (45° и 67.5°) → `Fvector4` в `C0` (rgba 0..255); `C1` = `clouds_color` (alpha = weight); геометрия — `env.CloudsVerts/CloudsIndices` (hemisphere quality 2 — см. [environment](../xr-engine/environment.md)); шейдер `clouds`/"null"; `rmNormal()`.

### `dxLensFlareRender::Render`

Один lock `24×4` LIT-вертексов. `fDistance = FAR_DIST × 0.75`, где **`FAR_DIST` — макрос** = `g_pGamePersistent->Environment().CurrentEnv->far_plane` (не константа). `bSun`: source-квад в `owner.vecLight` (размер `fRadius × fDistance`, цвет = `LightColor` или белый при `ignore_color`, alpha × `m_StateBlend`). `bFlares` (при `fBlend >= EPS_L`): per-фляр — pos = `vecCenter + vecAxis × fPosition`, размер `fRadius × fDistance`, цвет × `fOpacity × fBlend × m_StateBlend`. `bGradient`: квад в `vecLight`, цвет × `fGradientValue × m_StateBlend`. Каждый квад пушит свой шейдер в `_2render[]`; один unlock, затем **per-квад draw** (`set_Shader` + `RCache.Render` 4 верт / 2 три) — до 25 draw calls.

### `dxThunderboltRender::Render`

Модель: `CULL_NONE`; `dv = lightning_phase × 0.5` (randomize `0..0.5` после половины — мерцание); `l_model->transfer(current_xform, v_ptr, 0xffffffff, i_ptr, 0, 0, dv)` (transfer detail-model с UV-offset `dv` — анимация прокрутки текстуры молнии); draw с `l_model->shader`; restore `CULL_CCW`. Затем два gradient-квада (top в `current_xform.c`, center в `lightning_center`), цвет top = `m_GradientTop->fOpacity × lightning_phase` серый — **quirk: center-квад тоже использует `m_GradientTop->fOpacity`** (не `m_GradientCenter->fOpacity` — копи-паста). **DX10/11: блок `set_Z(TRUE)+set_ZFunc(LESSEQUAL)` продублирован 4×** (копи-паста) вокруг двух рендеров.

### `dxDebugRender`

Линии батчатся per-цвет (4 карты: verts/inds × world/hud). `add_lines`: **u16-лимит** — если суммарные вертексы или индексы превысят `u16(-1)` → сначала `Render()` flush; index-remap `*I = vertices_size + *J`. `Render()`: сначала hud-линии — swap `Device.mFullTransform = mFullTransformHud` + `set_xform_project(mProjectHud)` + `rmNear()`, DX10/11: `m_WireShader` + `set_c("tfactor", rgba/255)`; `RCache.dbg_Draw(LINELIST)`; очистка; restore.

### `dxApplicationRender::load_draw_internal`

DX10/11: `rmNormal()` + `set_RT(pBaseRT)` + `set_ZB(pBaseZB)` + `ClearRenderTargetView` (чёрный); reshade-hook (`if (use_reshade) render_reshade_effects()`). Aspect: `b_ws = (w/h) > 1.34`, `b_16x9 = b_ws && (w/h) > 1.77`; `ws_k = 0.75 (16:9) / 0.8333 (16:10)`, `ws_w = 171 / 102.6`. База 1024×768 → scale `k`. Progress-бар: 40 сегментов, per-сегмент цвет = `calc_progress_color` — **сигмоида**: `f = 1/(exp((idx − kk)×0.5) + 1)`, `kk = (stage+1)/max_stage × total`, `color_argb_f(f,1,1,1)`. Фон (1024×768 через `draw_face`); widescreen: left+right side-panel из `hLevelLogo_Add` (tex 0..128 / 128..256) — **quirk: ветки 16:9 и 16:10 идентичны** (копи-паста). Заголовок — `pFontSystem` (color (103,103,103), alCenter, `ls_header`/`""`/`ls_tip_number`, затем `draw_multiline_text(pFontSystem, 600k.x×(ws?0.8:1), ls_tip)`); логотип уровня — `hLevelLogo` в (0,173)+offsets, размер 1024×399, tex 0..0.77926. `draw_multiline_text`/`parse_word` — free-функции, word-wrap по `SizeOf_` vs `fTargetWidth` (разделители: пробел/tab/CR/LF/`,`/`.`/`:`/`!`). `KillHW()` — `ZeroMemory(&HW, sizeof(CHW))` (аварийный сброс, без разбора COM-ссылок — quirk).

### `CGIFAnimationPlayer`

Автономный GIF-плейер (конкретный класс, не `IRender`-интерфейс; используется UI для анимированных иконок). `Load`: per-кадр DX10/11 — `CreateTexture2D` (`D3D_USAGE_IMMUTABLE`, `R8G8B8A8_UNORM`, 1 mip) + `CreateShaderResourceView` (SRV в `Frame::srv`); DX9 — `CreateTexture` MANAGED + LockRect-copy. `UpdateFrame`: time-аккумулятор на `Device.dwTimeContinual`, skip-ahead ограничен `maxSkipFrames = 10` (потом `m_TimeAccum = 0`). `Play` — reset в кадр 0; `Stop` — null active frame. Деструктор освобождает surfaces + SRVs.

### `dxObjectSpaceRender` (**только `#ifdef DEBUG`**)

Конструктор: `m_shDebug.create("debug\\wireframe", "$null")`. `dbgAddSphere(sphere, colour)` → вектор `dbg_S`. `dbgRender()`: `R_ASSERT(bDebug)`, `set_Shader(m_shDebug)`; по `q_debug.boxes` (`clQueryCollision`): `RCache.dbg_DrawOBB(X, halfsize, red)` + `dbg_DrawEllipse(R, blue)`, очистка; по `dbg_S`: `dbg_DrawEllipse(M=scale+translate, P.second)`, очистка. Комментарий `// MT: dangerous` на `q_debug`/`dbg_S` (не thread-safe).

### 3DFluid (активная подсистема)

Файлы: `src/Layers/xrRenderDX10/3DFluid/` — `dx103DFluid{Manager,Data,Volume,Renderer,Grid,Blenders,Emitters,Obstacles}`.

- **Жизненный цикл**: `CRender::create` → `FluidManager.Initialize(70,70,70)` + `SetScreenSize` ([r4.cpp](../../../src/Layers/xrRenderPC_R4/r4.cpp)); `CRender::destroy` → `FluidManager.Destroy()`. `CRender::Load3DFluid` ([r4_loader](../../../src/Layers/xrRenderPC_R4/r4_loader.cpp)) — gate `RImplementation.o.volumetricfog`, читает `$level$level.fog_vol` (version 3), создаёт `dx103DFluidVolume` на запись, вешает в `FHierrarhyVisual::children` сектора.
- **`dx103DFluidVolume::Render`**: строит debug-wire-box (24 LIT-верт) — **но вызовы `RCache.Render` закомментированы** (box никогда не рисуется); затем `fTimeStep = 2.0f` **HARDCODED** (реальная строка `Device.fTimeDelta*30*2.0f` закомментирована) → `FluidManager.Update(m_FluidData, fTimeStep)` + `FluidManager.RenderFluid(m_FluidData)` — оба активны. `Load`: OGF-ветка `dxRender_Visual::Load` закомментирована; грузит version-3 данные; stub-шейдер `("fluid3d_stub","water\\water_ryaska1")` для сортировки.
- **`FluidManager::Update`**: `PIX_EVENT(simulate_fluid)`; `AttachFluidData` (per-volume private RT `VP_VELOCITY0/VP_PRESSURE/VP_COLOR` подсоединяются к слотам `RENDER_TARGET_VELOCITY0+`); viewport = размер среза 3D-текстуры; `RCache.set_ZB(0)`; `UpdateObstacles`; хардкод confinement/decay (BFECC: fire 0.03/0.9995, fog 0.06/0.994; без BFECC: 0.12/0.9995), затем **переопределение из `VolumeSettings`**; `AdvectColorBFECC`/`AdvectColor` → `AdvectVelocity(gravity)` → `ApplyVorticityConfinement` → `ApplyExternalForces` → `ComputeVelocityDivergence` → `ComputePressure` (Jacobi) → `ProjectVelocity`; `DetachAndSwapFluidData` (swap COLOR-RT с глобальным); восстановление RT (msaa vs non-msaa) + `rmNormal()`.
- **`RenderFluid`**: bind `VP_COLOR` → `RENDER_TARGET_COLOR_IN`, `m_pRenderer->Draw` (CompRayData Back/Front → QuadDownSample → EdgeDetect → RaycastFog/Copy, RaycastFire/Copy), восстановление RT.
- **RT-карта менеджера**: `RENDER_TARGET_VELOCITY1/COLOR/OBSTACLES/OBSTVELOCITY/TEMPSCALAR/TEMPVECTOR` (собственные) + `VELOCITY0/PRESSURE/COLOR_IN` (из volume-данных).
- **`dx103DFluidData`**: per-volume private RT + `Settings{m_fHemi, m_fConfinementScale, m_fDecay, m_fGravityBuoyancy, m_SimulationType FOG/FIRE}`, obstacles (`xr_vector<Fmatrix>`), emitters (`CEmitter`: gaussian/draught, позиция/радиус в fluid-space, flow velocity, saturation/density, draught period/phase/amp).
- **`dx103DFluidBlenders`**: 8 внутренних blenders — `CBlender_fluid_advect/_advect_velocity/_simulate/_obst/_emitter/_obstdraw/_raydata/_raycast` («INTERNAL: 3dfluid maths»), `canBeDetailed/canBeLMAPped = FALSE`.
- **`dx103DFluidGrid`**: debug-геометрия (slice/boundary quads/lines).
- **DEBUG**: `RegisterFluidData`/`UpdateProfiles` — real-time config reload из INI (консольный хук — `xrRender_console.cpp`).

## Взаимодействие

- `CEnvironment` → `IEnvironmentRender` (`RenderSky/RenderClouds/OnFrame`) — вызовы в [R4: post-process](r4-postprocess.md) (`phase_combine`) и [R4: scene](r4-scene.md) (MV-pass); данные окружения — [environment](../xr-engine/environment.md).
- `CLensFlare` → `ILensFlareRender` (вызов — из `dxEnvironmentRender::RenderSky`); `CEffect_Thunderbolt` → `IThunderboltRender` ([Визуальные эффекты](../xr-engine/effects.md), итерация 2).
- `CApplication` → `IApplicationRender` (экран загрузки, xrEngine); `CStatGraph`/`CStats` → [stats](../xr-engine/stats.md).
- 3DFluid: `CRender` (create/destroy/Load3DFluid) → `FluidManager` → `dx103DFluidVolume::Render` (из dsgraph-traversal, [Пайплайн: секторы](sector.md)); emitters/obstacles — данные уровня (`fog_vol`).
- `dxDebugRender::Render` — из `RDebugRender::OnRender` (`Device.seqRender`, приоритет `REG_PRIORITY_LOW−100`); линии добавляются из xrGame/алифы через `rdebug_render`.
- `stats_manager` — вызывается из `CTexture`/buffer-кода при создании/уничтожении GPU-буферов ([Ресурсы](resources.md)).

## Потоки данных

```
CStatGraph::subgraphs  → OnRender → hGeomTri(stBar)/hGeomLine(stCurve,stBarLine,markers) → RCache.Render
CEnvironment (phase_combine) → RenderSky → sh_2sky E[0]/E[1] + sky_r_textures(mixer) → RCache.Render
CLensFlare → Render → 24×4 LIT lock → per-quad RCache.Render (до 25 calls)
CApplication → load_draw_internal → progress(40 сег) + back + sidepanels + font → RCache
fog_vol (v3) → dx103DFluidVolume → FluidManager.Update (10+ shader-пассов по 3D-RT) → RenderFluid (raycast)
```

## Конфигурация

- `level.fog_vol` (version 3) — данные 3DFluid-объёмов; gate — `o.volumetricfog` ([R4: scene](r4-scene.md)).
- DEBUG: `RegisterFluidData`/`UpdateProfiles` — per-volume секции INI (confinement/decay/gravity/hemi), hot-reload из консоли.
- Остальное — хардкодированные шейдерные имена (`"sky2"`, `"clouds"`, `"hud\\default"`, `"ui\\ui_actor_loadgame_screen"`, `"debug\\wireframe"`) и константы (см. выше).

## Известные ограничения-дебаг

- **3DFluid `fTimeStep = 2.0f` HARDCODED** (в `dx103DFluidVolume::Render`) — скорость симуляции не привязана к реальному времени; debug-wire-box никогда не рисуется (`RCache.Render` закомментирован).
- `dxApplicationRender::KillHW()` — `ZeroMemory(&HW)` без разбора COM-ссылок; widescreen 16:9/16:10 side-panel ветки идентичны (копи-паста).
- `dxStatGraphRender` — copy-paste в `Num_H_LinesDwn` и out-of-bounds `end()+1` в циклах (см. выше).
- `dxThunderboltRender` — center-gradient квад использует `m_GradientTop` (копи-паста); `set_Z`-блок продублирован 4×.
- `dxLensFlareRender` — до 25 draw calls на один `Render` (по одному на квад).
- `dxDebugRender::SetAmbient` — `VERIFY(!"Not implemented for DX10")`; `NextSceneMode` — no-op в DX10.
- `dxObjectSpaceRender` — только DEBUG; `q_debug`/`dbg_S` не thread-safe (`// MT: dangerous`).
- `dxEnvironmentRender::OnFrame` — блок env-параметров (fog color/start/end) **не реализован в DX10/11** (fog в R4 обрабатывается в postprocess/skybox-шейдере); R2-only hack (push tonemap) — legacy-ветка.
- `dxUISequenceVideoItem::video_Play` — `return m_texture->video_Play(...)` (вызов выполняется, результат отбрасывается — странная стилистика).
