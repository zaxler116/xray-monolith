# Постобработка

Общий обзор post-processing в R4.

## Структура

Post-processing в R4 — это набор фаз `CRenderTarget::phase_*`, которые читают deferred-буферы (`rt_Color`, `rt_Position`, `rt_Accumulator`) и пишут финальное изображение или промежуточные ssfx-RT.

## Страницы

- [R4: post-process](r4-postprocess.md) — `r4_rendertarget_phase_*` (bloom/combine/hdao/hdr10_*/luminance/occq/PP/rain/smap_D/smap_S/ssao), `dx11HDAOCSBlender`, `dx11MinMaxSMBlender`, `ComputeShader`/`CSCompiler`.
- [R4: scene/lighting phase](r4-scene.md) — `CRender::Render` (вызовы `phase_*`), `render_lights`, `shader_compile`.
- [R4: deferred / накопление](r4-deferred.md) — `CRenderTarget` (RT-карта кадра), `phase_scene_*`, `accum_*`.

## Ключевые фазы

| Фаза                       | Назначение                                             |
| -------------------------- | ------------------------------------------------------ |
| `phase_combine`            | Основная фаза: сборка из deferred-буферов (800+ строк) |
| `phase_pp`                 | Final post-process (blur/gray/noise/duality/colormap)  |
| `phase_bloom`              | Legacy bloom (luminance→filter)                        |
| `phase_ssfx_bloom`         | Новый ssfx bloom (downsample→Kawase→upsample)          |
| `phase_luminance`          | 3 прохода (256→64→8→1) + tonemap-adaptation            |
| `phase_hdao`               | HD AO (compute shader)                                 |
| `phase_ssao`               | Legacy SSAO                                            |
| `phase_ssfx_ao`            | Новый AO (4 blur-фазы)                                 |
| `phase_ssfx_il`            | Новый IL (4 blur-фазы)                                 |
| `phase_hdr10_bloom`        | HDR10 bloom (Kawase)                                   |
| `phase_hdr10_lens_flare`   | HDR10 lens flare (Kawase)                              |
| `phase_smap_direct`        | SMAP direct (sun)                                      |
| `phase_smap_spot`          | SMAP spot                                              |
| `phase_rain`               | Rain setup                                             |
| `phase_ssfx_rain`          | SSFX rain                                              |
| `phase_occq`               | Occlusion query setup                                  |
| `phase_combine_volumetric` | Volumetric light combine                               |
| `phase_wallmarks`          | Wallmarks setup                                        |
| `DoAsyncScreenshot`        | Async screenshot                                       |
| `phase_downsamp`           | Downsample для SSAO                                    |

## Конфигурация

- `ps_pp_*` — post-process параметры (blur, gray, noise, duality, colormap, gasmask, nightvision, heatvision, fakescope).
- `ps_r__smaa` — SMAA.
- `ssfx_taa` — TAA.
- `ps_ssfx_*` — ssfx-эффекты (bloom, volumetric, motion blur, rain, fog scattering, sunshafts).
- `dx11_hdr10` — HDR10.
- `ssao_hdao` / `ssao_ultra` — HD AO.
- `advancedpp` — advanced post-processing.

## Известные ограничения

- `PP_Complex` — хардкод `TRUE` (HOLGER HACK), игнорирует `u_need_PP()`.
- `phase_luminance()` — 3 прохода (256→64→8→1), tonemap-adaptation EMA.
- `phase_hdao()` — compute shader, group 56, overlap 12 (static).
- `phase_ssfx_bloom()` — новый ssfx bloom (6 downsample + 5 upsample + lens + combine).
- `phase_hdr10_bloom()` / `phase_hdr10_lens_flare()` — Kawase blur (ping-pong).
- `DoAsyncScreenshot()` — TODO: won't work in DX11 with HDR (format incompat).
- `phase_smap_spot_tsh()` — не реализовано (`VERIFY`).
