# Модуль: xrSound

`xrSound` — **модуль воспроизведения и 3D-симуляции звука** движка:
интерфейс `ref_sound`/`CSound_manager_interface`, ядро `CSoundRender_Core`
и OpenAL/EFX-бэкенд `CSoundRender_CoreA`. Отвечает на вопрос «как звук
звучит в 3D-пространстве: позиционирование, затухание по расстоянию,
окклюзия, реверберация окружения».

**Чего НЕ делает**: не описывает, *какие* звуки играют на события
(логика выбора файлов — в `CObject`/`CLevel`/Lua, xrGame/xrEngine),
не декодирует Ogg-файлы в плоских данных (декодер — внутри Source),
не делает музыкальный секвенсор (музыка — тот же `ref_sound` c
`st_Music`).

См. [Карта модулей](../../architecture/module-map.md), [Граф связей](../../architecture/dependencies.md).

## Структура

| Страница                                     | Что покрывает                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Интерфейс и точка входа](interface.md)      | `Sound.h` (`ref_sound`/`ref_sound_data`, `CSound_manager_interface`, `CSound_emitter`, `CSound_source`, `CSound_params`, `CSound_stats(_ext)`, флаги, `soundSmoothingParams`), `sound.cpp` (`_create`/`_destroy`), `SoundRender.h` (константы), глобалы `psSound*`/`psReverb*`                                                                                                                                                                |
| [Core — ядро](core.md)                        | `SoundRender_Core.cpp` (контейнеры sources/emitters/targets, `create/attach_tail/clone/destroy/play*`, `_create_data`, `set_geometry_*`, `get_environment`, `pause_emitters`), `SoundRender_Core_StartStop.cpp`, `SoundRender_Core_SourceManager.cpp`                                                                                                                                                                                                  |
| [Core — процессор и окружение](core-processor-env.md) | `SoundRender_Core_Processor.cpp` (полный разбор `update`-шагов, `update_events`, `statistic`, `get_occlusion`/`get_occlusion_to`), `SoundRender_Environment.h/.cpp` (`CSoundRender_Environment` — EAX-поля, `load`/`save` v4/v5, `lerp`/`clamp`, `SoundEnvironment_LIB` — `sEnvironment.xr`, `reverbs[]` 113 пресетов), EFX-применение в CoreA (`LoadEffect`/`load_reverb`/`commit`/`set_listener`/`get_listener`)                                                                                                                                                                                                                   |
| [Source + Target](source-target.md)          | `SoundRender_Source.h/.cpp` (загрузка WAV/Ogg 44100 Hz, метаданные из Ogg-комментариев v1–v3, `i_decompress_fr` 16-bit PCM, `ov_error`), `SoundRender_Source_loader.cpp` (ov-callback'ы, `LoadWave`, `load`/`unload` — `$level$`→`$game_sounds$`→`$no_sound.ogg`), `SoundRender_Target.h/.cpp` (абстрактный: ленивый `attach`/`dettach`, stub-API), `SoundRender_TargetA.h/.cpp` (OpenAL: 3 ALbuffer, `buf_block` 400ms, 3D-параметры с flip -z, gain/pitch с гистерезисом, underrun), `SoundRender_Cache.h/.cpp` (LRU: line + page-table, `initialize` удваивает память, `request` hit/miss/eviction), `fill_block`/`fill_data`/курсор/attached-переход (путь данных, FSM — порц. 9), consumer-side `get_occlusion_to` (ит.3 порц.4)                                                                                          |
| Emitter _(порцион 9)_                        | `SoundRender_Emitter.h/.cpp` (эмиттер = активный экземпляр звука), `SoundRender_Emitter_FSM.cpp` (state-машина play/stop/pause, fade-in/out), `SoundRender_Emitter_StartStop.cpp`, `SoundRender_Emitter_streamer.cpp` (стриминг с диска)                                                                                                                                                                                                                                                                                       |
| Устройства и прочее _(порцион 10)_           | `OpenALDeviceList.h/.cpp` (`ALDeviceList` — перечисление/выбор устройств), `NotificationClient.h/.cpp` (Core Audio device-change notifications), `SoundRender_CoreA.cpp` (инициализация устройства, hot-swap `switch_device`, EFX-слот), `cl_intersect.h` (дубль `xrEngine/cl_intersect.h`), статистика/`statistic`-интерфейс, `guids.cpp`                                                                                                                              |

## Карта связей

```mermaid
graph TD
    subgraph xrEngine
        LV[IGame_Level / CObject]
        SL[SoundProcessor — pureFrame]
    end
    subgraph xrSound
        IF[Sound.h: ref_sound + CSound_manager_interface]
        CORE[CSoundRender_Core]
        COREA[CSoundRender_CoreA]
        EM[Emitter FSM]
        SRC[Source / Streamer]
        TGT[Target / TargetA]
        ENV[Environment LIB — sEnvironment.xr]
        DEV[ALDeviceList + NotificationClient]
    end
    AL[OpenAL / EFX]

    LV -->|create / play / set_position| IF
    SL -->|Sound->update per frame| IF
    IF -->|delegate| CORE
    CORE --> COREA
    COREA -->|alc/al + EFX| AL
    COREA --> DEV
    CORE --> EM
    CORE --> SRC
    EM --> TGT
    SRC --> TGT
    CORE --> ENV
```

- **Потребители**: xrEngine (`IGame_Level::Load` → `Sound->set_geometry_occ`,
  `SoundProcessor::OnFrame` → `Sound->update`, `xr_ioc_cmd` →
  `snd_restart`/`sounddevice`), xrGame (`CObject`-звук: `ref_sound` в
  объектах, `Feel::Sound`, `SoundEvent_Register`).
- **Зависимости**: xrCore (`IReader`/`IWriter`, `xr_vector`, `shared_str`),
  xrCDB (`CDB::MODEL`/`COLLIDER` — `geom_DB`/`geom_SOM`/`geom_MODEL`/`geom_ENV`),
  OpenAL + EFX (efx-headers, `AL/efx-presets.h`), Windows Core Audio
  (COM `IMMDeviceEnumerator`, только в `NotificationClient`).
- **Публичный API**: глобал `CSound_manager_interface* Sound`
  (`Sound.h`) — единственная точка входа; фабрика-статик `_create(stage)`.
- `XRSOUND_API` (dllexport) **закомментирован** — модуль статический
  (аналог `PARTICLES_API` в [xrParticles](../xr-particles/index.md)).

## Статус порционов (итерация 4)

| #   | Порцион                           | Страницы                  | Статус    |
| --- | --------------------------------- | ------------------------- | --------- |
| 5   | Звук: интерфейс и точка входа     | `index.md`, `interface.md`| ✅ готово |
| 6   | Звук: Core — ядро                 | `core.md`                 | ✅ готово |
| 7   | Звук: Core — процессор и окружение| `core-processor-env.md`   | ✅ готово |
| 8   | Звук: Source + Target             | `source-target.md`        | ✅ готово |
| 9   | Звук: Emitter                     | `emitter.md`              | ⬜ план   |
| 10  | Звук: устройства и прочее         | `devices-misc.md`         | ⬜ план   |
