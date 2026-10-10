# План итерации 4 — `xrParticles` + `xrSound`

## Масштаб

| Модуль | .cpp | Объём |
|--------|------|-------|
| xrParticles | 8 | ~130 KB |
| xrSound | 23 | ~230 KB |
| **Итого** | **31** | **~360 KB** ≈ 1/2 xrEngine (ит.2) |

Ориентир: ит.3 (xrRender/R4) = 17 порционов; ит.4 — **10 порционов**.

## Решения

1. **Страницы**: подкаталоги `modules/xr-particles/` (3 файла) + `modules/xr-sound/` (7 файлов). Текущие заглушки `xr-particles.md` / `xr-sound.md` заменяются на `index.md` в подкаталогах.
2. **`particle_actions_collection.cpp` (55 KB)** — самый крупный файл. Полный разбор 30 подклассов действий в порционе 3 (не дублируя ит.3 порц.16, где описан редактор `ParticleEffectActions.h/.cpp`).
3. **`cl_intersect.h` в xrSound (25 KB)** — проверить дублирование с `xrEngine/cl_intersect.h`; если дубль — один абзац со ссылкой.
4. **`SoundRender_CoreA` (21 KB)** — OpenAL-аллокаторы; если legacy/закомм. — сократить.
5. **Граница с ит.3**: рантайм-частицы (`CParticleEffect`/`CParticleGroup`) уже покрыты в `renderer/particles-wallmarks.md`; здесь — только internals `PAPI::ParticleManager`.

## Порционы

### xrParticles (порционы 1–4)

| # | Страница | Файлы |
|---|----------|-------|
| 1 | PAPI и ядро симуляции — `xr-particles/index.md`, `core.md` | psystem.h, particle_core, particle_effect, particle_manager |
| 2 | Actions: ядро и домены — `xr-particles/actions.md` | particle_actions, particle_actions_collection (каркас) |
| 3 | Actions: 30 подклассов + IO — `xr-particles/actions-actions.md`, `actions-io.md` | подклассы действий, particle_io |
| 4 | Noise и связка с движком — `xr-particles/noise-integration.md` | particle_noise, particle_noise_*, интеграция с xrEngine |

### xrSound (порционы 5–10)

| # | Страница | Файлы |
|---|----------|-------|
| 5 | Звук: интерфейс и точка входа — `xr-sound/index.md`, `interface.md` | Sound.h/.cpp, cvar'ы, фабрика |
| 6 | Звук: Core — ядро — `xr-sound/core.md` | SoundRender_Core, StartStop, SourceManager |
| 7 | Звук: Core — процессор и окружение — `xr-sound/core-processor-env.md` | Core_Processor, Environment, EFX/EAX |
| 8 | Звук: Source + Target — `xr-sound/source-target.md` | Source, Source_loader, Target, TargetA, Cache LRU |
| 9 | Звук: Emitter — `xr-sound/emitter.md` | State-машина, FSM, StartStop, streamer |
| 10 | Звук: устройства и прочее — `xr-sound/devices-misc.md` | OpenALDeviceList, NotificationClient, cl_intersect.h, CoreA, statistic |

## Статус

- [ ] 1. PAPI и ядро симуляции
- [ ] 2. Actions: ядро и домены
- [ ] 3. Actions: 30 подклассов + IO
- [ ] 4. Noise и связка с движком
- [ ] 5. Звук: интерфейс и точка входа
- [ ] 6. Звук: Core — ядро
- [ ] 7. Звук: Core — процессор и окружение
- [ ] 8. Звук: Source + Target
- [ ] 9. Звук: Emitter
- [ ] 10. Звук: устройства и прочее

## Обещанное (кросс-обязательства, которые ит.4 обязана закрыть)

| Откуда | Что | Куда |
|--------|-----|------|
| ит.3 порц.16 | `ParticleManager`/`PAPI` internals | порционы 1–3 |
| ит.2 порц.12 | `CSound_stats_ext`/`statistic()` | порционы 5, 10 |
| ит.2 порц.4 | `CSound_manager_interface`/`CSound_emitter` | порцион 5 |
| ит.2 порц.6 | `Sound->set_geometry_occ` | порцион 6 |
| ит.3 порц.4 | `get_occlusion_to` | порцион 8 |
| ит.2 порц.8 | `CSound_environment`/`SoundEnvironment_LIB` | порцион 7 |

## После

- [ ] Обновить `architecture/dependencies.md` (xrParticles, xrSound в графе)
- [ ] Перенести лог итерации в `GENERATE-DOCS.md` §9
- [ ] Удалить этот файл
