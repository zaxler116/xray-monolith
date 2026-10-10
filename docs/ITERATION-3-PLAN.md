# Итерация 3: xrRender/R4 — план порционов

> Рабочий документ-план. После завершения итерации 3 содержание
> переносится в `GENERATE-DOCS.md` §9 (лог итераций) и файл удаляется.

## Масштаб

**Калибрка (итерация 2, xrEngine)**: 12 порционов, ~130 .cpp, 26 страниц.

**xrRender/R4** — по `xrRender_R4.vcxproj` (реальный состав скомпилированного R4,
`STATIC_RENDERER_R4` в `xrEngine.vcxproj`):

| Корзина                               | .cpp          | Что                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `xrRender/` (бэкенд-агностичный слой) | ~95           | R_Backend (7), r__sector/traversal/occlusion/dsgraph/pixel_calculator/screenshot (8), визуалы (FVisual/Basic/LOD/Skinned/Progressive/Hierrarhy/Tree), скелеты (SkeletonX/Rigid/Animated/Custom + Animation), ResourceManager (4), ModelPool, SH_* (5), Shader, Texture/ETextureParams/TextureDescrManager, Detail* (7), Light* (5), Particle* (5), blenders (13 + blenders/), dx*Render (17: device/UI/debug/console/font/imgui/env/lensflare/rain/thunderbolt×2/objectspace/statgraph/stats/wallmark/uishader/sequencevideo), R_DStreams, occRasterizer (2), stats_manager, tga, tss, du_* (5), NvTriStrip (2), VertexCache, WallmarksEngine, HOM, HW/HWCaps, uber_deffer, gifPlayer, xrRender_console, xr_effgamma, xrStripify |
| `xrRenderDX10/`                       | ~12           | dx10HW, dx10r_constants(+cache), dx10SH_RT, dx10SH_Texture, dx10ResourceManager_Resources/Scripting, dx10DetailManager_VS, dx10Texture/TextureUtils/BufferUtils/ConstantBuffer/StateUtils, StateManager (4: State/StateCache/StateManager/SamplerStateCache), dx10EventWrapper                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `xrRenderPC_R4/`                      | ~96           | r4 (4: core/loader/rendertarget/R_render), rendertarget phase_* (15), rendertarget accum_* (7), blender_* (24), light_* (5: GI/smapvis/vis/Light_Render_Direct/ComputeXFS), r2_* (6: blenders/R_calculate/R_lights/R_sun/sector_detect/test_hw), R_Backend_LOD, CSCompiler/ComputeShader, dx11HDAOCSBlender, dx11MinMaxSMBlender, SMAP_Allocator, r4_R_rain, r4_R_sun_support                                                                                                                                                                                                                                                                                                                                                    |
| Итого                                 | **~200 .cpp** | ≈ 1.5× xrEngine                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |

**Вывод: 17 порционов.**

## Решения (фиксация)

1. **R1/R2/R3 не документируем.** Активный бэкенд — R4 (`STATIC_RENDERER_R4`
   в `xrEngine.vcxproj`). В `index.md` одной таблицей: R1 = D3D9 legacy,
   R2 = D3D9 + advanced, R3 = DX10, R4 = DX11 — и «дальше только R4».
   Файлы R1/R2/R3 не трогаем.
2. **`xrRenderDX9/`** — не в проекте R4 (нет в sln/R4-proj), не трогаем.
3. **Исторические `r2_*` файлы** (r2_R_calculate/lights/sun/sector_detect/test_hw/blenders)
   — это D3D9-эпоха, но **компилируются в R4** (в `xrRender_R4.vcxproj`) —
   документируем в порционах 3, 11, 12 с пометкой
   «историческое ядро, переиспользуемое в R4».
4. **`3DFluid` (9 файлов)** — в R4. Если по ходу порциона 17 окажется, что это
   legacy/закомм., — сократим до абзаца.
5. **`shader-bus.md`** уже готов (итерация 0) — в порционе 14 только поправим
   перекрёстные ссылки, не перепишем.

## Порционы

| #   | Порцион                        | Страницы                                               | Содержание                                                                                                                                                                                                                                                                                                                                                   |
| --- | ------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Точка входа и фабрика          | `index.md`, `factory.md`                               | `DllMainXrRenderR4`, `RImplementation`, `RenderFactoryImpl` (`dxRenderFactory` — создание `IRenderTarget`/`IFontRender`/`IImGuiRender` и т.д.), `xrRender_test_hw` (`r2_test_hw`), `xrRender_initconsole`, глобалы (`::Render`, `::RenderFactory`, `::DU`, `UIRender`, `DRender`)                                                                            |
| 2   | Устройство рендера и константы | `render-device.md`, `constants.md`                     | `dxRenderDeviceRender`, `CHW`/`HWCaps`, `xrD3DDefs`, `dx10HW`, `dx10StateUtils`/StateManager (4), `r_constants`/`dx10r_constants`/`r_constants_cache`/`dx10r_constants_cache`, `SH_*` (5: Atomic/Constant/Matrix/RT/Texture + `dx10SH_RT`/`dx10SH_Texture`), `R_DStreams`, `xr_effgamma`                                                                     |
| 3   | Пайплайн: секторы и traversal  | `sector.md`                                            | `r__sector`, `r__sector_traversal`, `r2_sector_detect`, `r__pixel_calculator`, `QueryHelper`                                                                                                                                                                                                                                                                 |
| 4   | Occlusion                      | `occlusion.md`                                         | `r__occlusion`, `occRasterizer` (+core), `light_vis`                                                                                                                                                                                                                                                                                                         |
| 5   | Динамическая сцена (dsgraph)   | `dsgraph.md`                                           | `r__dsgraph_structure/build/render/render_lods`, `R_Backend_LOD`                                                                                                                                                                                                                                                                                             |
| 6   | VSM / SMAP                     | `vsm-smap.md`                                          | `R_Backend_xform`, `r_sun_cascades`, `light_smapvis`, `SMAP_Allocator`, `r2_R_sun`, `r4_R_sun_support`, `R_Backend_hemi`                                                                                                                                                                                                                                     |
| 7   | Ресурсы и модели               | `resources.md`                                         | `ResourceManager` (4: core/Loader/Reset/Scripting + `dx10ResourceManager_*`), `ModelPool`, `Texture`/`ETextureParams`/`TextureDescrManager`/`dx10Texture*`/`dx10BufferUtils`/`dx10ConstantBuffer`, `ColorMapManager`, `PSLibrary`, `tga`, `R_Backend_Runtime`                                                                                                |
| 8   | Визуалы                        | `visuals.md`                                           | `FVisual`, `FBasicVisual`, `FHierrarhyVisual`, `FLOD`, `FProgressive`, `FSkinned`, `FTreeVisual`, `xrStripify`, `NvTriStrip` (2), `VertexCache`                                                                                                                                                                                                              |
| 9   | Скелеты и анимация             | `kinematics.md`                                        | `SkeletonX`, `SkeletonRigid`, `SkeletonAnimated`, `SkeletonCustom`, `Animation`, `AnimationKeyCalculate`, `KinematicAnimatedDefs`                                                                                                                                                                                                                            |
| 10  | Детали (grass/крюки)           | `detail.md`                                            | `DetailManager` (5: core/CACHE/Decompress/VS/soft + `dx10DetailManager_VS`), `DetailModel`, `DetailFormat`, `Blender_detail_still`, `Blender_tree`, `R_Backend_tree`                                                                                                                                                                                         |
| 11  | Свет                           | `lights.md`                                            | `light` (ILight), `LightTrack`, `Light_DB`, `Light_Package`, `light_GI`, `r2_R_lights`, `r2_R_calculate`                                                                                                                                                                                                                                                     |
| 12  | R4: scene/lighting phase       | `r4-scene.md`                                          | `r4`, `r4_loader`, `r4_R_render`, `r2_blenders`, `Light_Render_Direct` (+ComputeXFS), `blender_light_direct`, `blender_light_point`, `blender_light_spot`, `blender_light_reflected`, `blender_light_mask`, `blender_luminance`                                                                                                                              |
| 13  | R4: deferred / накопление      | `r4-deferred.md`                                       | `r4_rendertarget`, `r4_rendertarget_accum_*` (7), `r4_rendertarget_phase_accumulator/scene`, `uber_deffer`, `blender_deffer_aref/flat/model`                                                                                                                                                                                                                 |
| 14  | R4: post-process               | `r4-postprocess.md` (+ обновление `postprocessing.md`) | `r4_rendertarget_phase_*` (bloom/combine/hdao/hdr10_*/luminance/occq/PP/rain/smap_D/smap_S/ssao), `dx11HDAOCSBlender`, `dx11MinMaxSMBlender`, `ComputeShader`/`CSCompiler`, `SMAP_Allocator`                                                                                                                                                                 |
| 15  | Blenders (библиотека)          | `blenders.md`                                          | `xrRender/blenders/` (Blender, Blender_Palette, Blender_Recorder) + `Blender_BmmD`, `Blender_Model_EbB`, `Blender_Lm(EbB)`, `Blender_Particle`, `Blender_Screen_SET`, `Blender_Editor_*` (2), `Blender_Recorder_*` (2), `r4 blender_*` (blur/combine/dof/gasmask__/hdr10__/lut/nightvision/pp_bloom/smaa/ss_sunshafts/ssao)                                  |
| 16  | Частицы и wallmarks            | `particles-wallmarks.md`                               | `ParticleEffect` (3: core/Actions/Def), `ParticleGroup`, `dxParticleCustom`, `WallmarksEngine`, `dxWallMarkArray`, `r4_rendertarget_wallmarks`, `r4_R_rain`, `r4_rendertarget_draw_rain/phase_rain`, `dxRainRender`                                                                                                                                          |
| 17  | UI-обвязка и прочее            | `dx-bridges.md`, `misc.md`                             | `dxUIRender`, `dxUIShader`, `dxFontRender`, `dxImGuiRender`, `dxConsoleRender`, `dxStatsRender`/`dxStatGraphRender`, `stats_manager`, `dxEnvironmentRender`, `dxLensFlareRender`, `dxThunderboltRender`/`dxThunderboltDescRender`, `dxDebugRender`, `dxObjectSpaceRender`, `dxApplicationRender`, `dxUISequenceVideoItem`, `gifPlayer`, `HOM`, `3DFluid` (9) |

## Статус

| #   | Порцион                        | Статус |
| --- | ------------------------------ | ------ |
| 1   | Точка входа и фабрика          | ✅     |
| 2   | Устройство рендера и константы | ✅     |
| 3   | Пайплайн: секторы и traversal  | ✅     |
| 4   | Occlusion                      | ✅     |
| 5   | Динамическая сцена (dsgraph)   | ✅     |
| 6   | VSM / SMAP                     | ✅     |
| 7   | Ресурсы и модели               | ✅     |
| 8   | Визуалы                        | ✅     |
| 9   | Скелеты и анимация             | ✅     |
| 10  | Детали (grass/крюки)           | ✅     |
| 11  | Свет                           | ✅     |
| 12  | R4: scene/lighting phase       | ✅     |
| 13  | R4: deferred / накопление      | ✅     |
| 14  | R4: post-process               | ⬜     |
| 15  | Blenders (библиотека)          | ⬜     |
| 16  | Частицы и wallmarks            | ⬜     |
| 17  | UI-обвязка и прочее            | ⬜     |

## Обещанное из итерации 2 (кросс-ссылки, которые итерация 3 обязана закрыть)

- рендер-бэкенд `IRenderDeviceRender` — порционы 1–2
- `RenderVisual`/`Kinematics` — `PlayCycle`/`PlayFX`, blending `CBlend`,
  де-квантизация, skinning `skin1W..4W` — порцион 9
- `CModelPool` — порцион 7
- `IEnvironmentRender`/`IEnvDescriptorRender`/`IEnvDescriptorMixerRender`
  — порцион 17 (`dxEnvironmentRender`)
- `dxRainRender`/`dxThunderboltRender`/`dxLensFlareRender`/`IFlareRender`/
  `IThunderboltDescRender` — порционы 16, 17
- `IFontRender`/`dxFontRender`/`IImGuiRender` — порцион 17
- `IStatsRender`/`IStatGraphRender`/`IUIShader`/`dxStatGraphRender` — порцион 17
- `SPPInfo`-применение `IRender_Target` — порционы 12–14
- `CStats` — уже покрыто в порционе 12 итерации 2, не дублируем

## После итерации 3

- обновить `architecture/dependencies.md` (детальный граф вместо include-анализа)
- перенести лог в `GENERATE-DOCS.md` §9
- удалить этот файл
