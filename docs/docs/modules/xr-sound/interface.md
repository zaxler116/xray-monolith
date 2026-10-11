# xrSound — Интерфейс и точка входа

> Порцион 5 итерации 4. Файлы: `Sound.h`, `sound.cpp`, `SoundRender.h`,
> `SoundRender_Core.h` (объявления), `NotificationClient.h` (объявления),
> глобалы `psSound*`/`psReverb*`/`soundSmoothingParams`.

## 1. Ответственность

Слой API модуля: типы, видимые игроку/игре (`ref_sound`,
`CSound_emitter`, `CSound_params`), интерфейс менеджера
`CSound_manager_interface`, точка входа (`_create`/`_destroy`) и все
глобальные cvar'ы звука. **Вся реализация** — в
[Core — ядро](core.md) (порцион 6), [Core — процессор и окружение](core-processor-env.md)
(7), [Source + Target](source-target.md) (8), [Emitter](emitter.md) (9),
[Устройства и прочее](devices-misc.md) (10).

## 2. Точка входа

Глобал `CSound_manager_interface* Sound` (`Sound.h`) — единственная
точка входа. Фабрика — статические методы в `sound.cpp`:

- **`CSound_manager_interface::_create(stage)`**
  - `stage == 0`: `SoundRenderA = xr_new<CSoundRender_CoreA>()`;
    `SoundRender = SoundRenderA; Sound = SoundRender;`. Если
    `-nosound` в `Core.Params` → `bPresent = FALSE` и **return**
    (модуль существует, но звук выключен). Иначе `bPresent = TRUE`.
  - Любая стадия, если `bPresent`: `Sound->_initialize(stage)`.
- **`CSound_manager_interface::_destroy()`**: `Sound->_clear()`,
  `xr_delete(SoundRender)`, `Sound = 0`.

Вызывается из xrEngine (`x_ray.cpp`): `InitSound1()` → `_create(0)`,
`InitSound2()` → `_create(1)`, `destroySound()` → `_destroy()`.
Кадровый тик — `SoundProcessor : pureFrame` (там же):
`OnFrame` → `Sound->update(vCameraPosition, vCameraDirection,
vCameraTop)` (обёрнуто в `Device.Statistic->Sound.Begin/End`).

Консоль: `snd_restart` → `Sound->_restart()` (`xr_ioc_cmd.cpp`),
`sounddevice <name>` → `Sound->switch_device(args)`.

`-nosound` — единственный путь, при котором `Sound` ненулевой, но
звук молчит (`bPresent = FALSE`; `create`/`play` — early-return).

## 3. Типы для игры (`Sound.h`)

### `CSound_UserData` / `CSound_UserDataPtr`

Пустой маркерный класс (ref-counted) — «visitor»-данные, привязываемые
к звуку через `ref_sound_data::g_userdata` (игровой код кладёт туда
свои указатели; `CSound_UserDataPtr` — `intrusive_ptr`-обёртка).

### `ref_sound_data` (xr_resource)

Внутренние данные одного «звука» (ресурс + активный эмиттер):

| Поле                 | Тип                  | Назначение                                                                                   |
| -------------------- | -------------------- | -------------------------------------------------------------------------------------------- |
| `handle`             | `CSound_source*`     | Ресурс-источник (метаданные, стример)                                                         |
| `feedback`           | `CSound_emitter*`    | Активный эмиттер; **очистится сам** при `stop` (см. [Emitter](emitter.md))                    |
| `s_type`             | `esound_type`        | `st_Effect = 0` / `st_Music = 1` — влияет на громкость (см. ниже) и на cull-порог             |
| `g_type`             | int                  | Игровой тип (для `statistic`/`statistic_ext`), или тип ресурса                               |
| `g_object`           | `CObject*`           | Владелец-объект; обнуляется `object_relcase` при уничтожении объекта                          |
| `g_userdata`         | `CSound_UserDataPtr` | Visitor-данные                                                                               |
| `fn_attached[2]`     | `shared_str`         | До 2 «хвостов» (продолжений) — `attach_tail`; 3-й — DEBUG `Msg` + skip                        |
| `dwBytesTotal`       | DWORD                | Размер в байтах (для кэша/статистики)                                                         |
| `fTimeTotal`         | float                | Длительность, сек.                                                                             |

Конструктор вызывает `::Sound->_create_data(...)` — регистрация
ресурса в менеджере (реализация — [core.md](core.md)).

### `ref_sound`

Публичный handle (обычный struct, **не** xr_resource): все методы
`VERIFY(!::Sound->i_locked())` и делегируют либо в менеджер
(`create`/`attach_tail`/`clone`/`destroy`/`play*`), либо в
`_feedback()` (`set_position`/`set_frequency`/`set_range`/`set_volume`/
`set_priority`/`stop`/`stop_deffered`/`get_params`/`set_params`).
`set_position` дополнительно `VERIFY(_feedback())` (позиция — только
для играющего звука), остальные — `if (_feedback())` (тихий no-op).

`play_no_feedback` — «разовый» звук: создаёт **временный**
`ref_sound_data`, играет на нём, затем восстанавливает `S._p` —
оригинальный `ref_sound` сохраняет свой feedback (quirk: эмиттер
принадлежит временному data, семантика владения — в
[emitter.md](emitter.md)).

### `CSound_source` (абстрактный)

Метаданные ресурса: `length_sec`, `game_type`, `file_name`,
`channels_num`, `bytes_total`. Реализация — [source-target.md](source-target.md).

### `CSound_environment`

**Пустой маркерный класс** (forward-тип для `_EDITOR`-методов
`set_user_env`/`refresh_env_library` в интерфейсе). Реальный тип —
`CSoundRender_Environment` — в [core-processor-env.md](core-processor-env.md)
(порцион 7).

### `CSound_params`

Состояние эмиттера (читается/пишется `get_params`/`set_params`):
`set` (флаг «заполнен»), `position`/`velocity`/`curVelocity`/
`prevVelocity`/`accVelocity`, `base_volume`/`volume`/`freq`/
`min_distance`/`max_distance`/`max_ai_distance`, `pitch_variation`
(fork: `0.02 * randF(-1, 1) * pitchVariationPower` — по умолчанию
`pitchVariationPower = 0`, т.е. выключено). `update_position`
(вычисляет `curVelocity` как дельту), `update_velocity` (скользящее
среднее через `soundSmoothingParams::getSmoothedValue`; после —
`velocity = accVelocity`).

### `CSound_emitter` (абстрактный)

12 чистых виртуальных: `is_2D`, `switch_to_2D/Intro/3D`,
`set_position/frequency/range/volume/priority`, `stop(bDeffered)`,
`get_params`, `play_time`. Реализация — [emitter.md](emitter.md).

### `CSound_stream_interface`

**Пустой маркер** (зарезервирован, не используется).

### `CSound_stats` / `CSound_stats_ext`

`CSound_stats` — 5 счётчиков (`dwPlay`, `dwAlloc` и др. —
используются в `CStats::Show`, [stats](../xr-engine/stats.md)).
`CSound_stats_ext` — `xr_vector<SItem>`: `name`/`params`/`volume`/
`type`/`game_type`/`game_object` + 1-bit флаги `_3D`/`_rendered`
(используется `CStats::OnRender` DEBUG-overlay; internals —
порционы 5/10). Кросс-обязательство ит.2 порц.12 закрыто на уровне
интерфейса; реализация `statistic()` — [devices-misc.md](devices-misc.md).

## 4. `CSound_manager_interface`

Чистый интерфейс (~35 виртуальных), реализует
`CSoundRender_Core` ([core.md](core.md)). Закрыто кросс-обязательство
ит.2 порц.4 (`CSound_manager_interface`/`CSound_emitter`):

| Метод                                    | Назначение (реализация — порц. 6–8)                                                                                                  |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `_initialize(stage)` / `_clear` (prot.)  | Двухстадийный init (0 — аллокация, 1 — устройство/EFX) / полный сброс                                                                 |
| `_create_data` / `_destroy_data` (prot.) | Регистрация/снятие `ref_sound_data`                                                                                                   |
| `is_present` / `is_ready` / `i_locked`   | Флаги `bPresent`/`bReady`/`bLocked`                                                                                                   |
| `create` / `attach_tail` / `clone` / `destroy` | Жизненный цикл `ref_sound_data`                                                                                                 |
| `play` / `play_at_pos` / `play_no_feedback` | Запуск (флаги `sm_Looped`/`sm_2D`/`sm_Intro` + delay)                                                                              |
| `stop_emitters` / `pause_emitters`       | Глобальные stop/pause (pause — счётчик, quirk: старт с 1)                                                                             |
| `set_master_volume`                      | `alListenerf(AL_GAIN, f)` (CoreA)                                                                                                     |
| `set_geometry_env/som/occ`               | Геометрия окружения/окклюзии (ит.2 порц.6 — `set_geometry_occ`; тела — [core.md](core.md); `get_occlusion_to` — порц. 8)              |
| `set_handler` / `update` / `commit`      | Callback на события / кадровый тик / flush-коммит (реверб-слот)                                                                       |
| `statistic` / `statistic_ext`            | `CSound_stats` / `CSound_stats_ext` (порц. 5/10)                                                                                      |
| `get_occlusion_to` / `get_occlusion`     | Октлюзия (ит.3 порц.4 — порц. 8)                                                                                                      |
| `object_relcase(O)`                      | Обнуление `g_object` у всех эмиттеров объекта                                                                                          |
| `listener_position`                      | Позиция слушателя (камера)                                                                                                            |
| `get_environment(P)`                     | Окружение в точке P (ray-вниз, front/back face id) — реализация [core.md](core.md), данные [core-processor-env.md](core-processor-env.md) |
| `_EDITOR`: `set_user_env` / `refresh_env_library` / `refresh_sources` / `set_environment_size` / `set_environment` | Только редактор; `set_environment_size`/`set_environment` — пустые заглушки                          |

## 5. Флаги и константы

- **`ss_Hardware (1<<1)` / `ss_EFX (1<<2)`** — `psSoundFlags`
  (def `ss_Hardware|ss_EFX`).
- **`sq_DEFAULT/NOVIRT/LIGHT/HIGH`** — качество стриминга.
- **`sm_Looped` / `sm_2D` / `sm_Intro`** — флаги `play`.
- **`esound_type`**: `st_Effect = 0`, `st_Music = 1`.
- **`sg_Undefined = 0`**, `sg_SourceType = -1` (g_type: «брать из ресурса»).
- `SNDENV_FILENAME "sEnvironment.xr"` (путь к библиотеке окружений,
  `$game_data$`), `OGG_COMMENT_VERSION 0x0003`.
- `SoundRender.h`: `sdef_target_count = 3`, `sdef_target_block = 400` мс,
  `sdef_target_size = 1200` мс, `s_f_def_source_footer = 0.0f`,
  `sdef_env_version = 4`, `sndenv_ver_extended = 5` (EFX/EAX-поля для
  IX-Ray), `sdef_level_version = 1`, `s_f_def_event_pulse = 0.5f`.

## 6. Глобалы (cvar'ы)

Определены в `SoundRender_Core.cpp` (если не указано). Консольные
обёртки — `xr_ioc_cmd.cpp` (ит.2 порц.12):

| Глобал                              | Def        | Назначение                                                            |
| ----------------------------------- | ---------- | --------------------------------------------------------------------- |
| `psSpeedOfSound`                    | 1          | Множитель скорости звука (для Доплера)                                 |
| `psSoundTargets`                    | 1024       | Пул OpenAL-target'ов (pre-create на stage 1)                           |
| `psSoundFlags`                      | HW\|EFX    | `Flags32`: hardware/efx                                                |
| `psSoundOcclusionScale`             | 0.5        | Масштаб окклюзии                                                        |
| `psSoundCull`                       | 0.01       | Порог громкости cull'а (объявление — `extern` в `SoundRender_Emitter_FSM.cpp`, определение в Core.cpp) |
| `psSoundRolloff`                    | 0.75       | Rolloff-фактор                                                          |
| `psSoundModel`                      | 0          | Модель затухания                                                         |
| `psSoundVEffects` / `psSoundVFactor`| 1.0 / 1.0  | Громкость эффектов / общий factor                                        |
| `psSoundVMusic` / `psSoundVMusicFactor` | 1 / 1    | Громкость музыки / factor                                              |
| `psSoundCacheSizeMB`                | 256        | Размер LRU-кэша стриминга (MB)                                          |
| `snd_efx_environment_change_time`   | 1.66       | Время перехода EFX-окружения (сек)                                      |

### `psReverb*` (22 глобал)

Все 22 свойства EAX-реверберации (`AL_EAXREVERB_*`) — объявлены в
`Sound.h`, определены в `SoundRender_CoreA.cpp` с дефолтами
`AL_EAXREVERB_DEFAULT_*`; `reverb_overwrite` (BOOL) — если TRUE,
`set_listener` применяет именно эти глобалы вместо `env` (отладочный
override). Применяются/читаются в `set_listener`/`get_listener`
(CoreA, порц. 10).

### `soundSmoothingParams`

Namespace в `SoundRender_CoreA.cpp` (объявление в `Sound.h`):

| Поле                     | Значение | Назначение                                             |
| ------------------------ | -------- | ------------------------------------------------------ |
| `distanceBasedDelayPower`| 1        | (не используется активно)                               |
| `distanceBasedDelayMinDistance` | 50   | (не используется активно)                               |
| `pitchVariationPower`    | **0**    | Сила pitch-вариации — **выключено по умолчанию**        |
| `power`                  | 1.8      | Экспонента сглаживания скорости                          |
| `steps`                  | 15       | Число шагов сглаживания                                  |
| `alpha`                  | `2/(steps+1)` | Коэффициент экспоненциального сглаживания           |

`getSmoothedValue` — экспоненциальное сглаживание (moving average).
`getTimeDeltaSmoothing` — возвращает просто `alpha` (dt-версия
закомментирована).

## 7. `SoundRender_Core` — объявления (порядок слоёв)

`SoundRender.h` + `SoundRender_Core.h`: абстрактный средний слой
(реализует `CSound_manager_interface`), наследуется `CSoundRender_CoreA`.
Ключевые поля (детали — [core.md](core.md)):

- Флаги: `bLocked` (volatile), `bPresent`, `bUserEnvironment`, `bReady`,
  `m_is_supported` (EFX-расширение есть), `bListenerMoved`,
  `bPendingDefaultDeviceSwitch`/`bPendingDeviceListRefresh` (volatile,
  ставятся `CNotificationClient`, читаются на звуковом потоке —
  [devices-misc.md](devices-misc.md)).
- Таймер: `Timer`/`fTimer_Value`/`fTimer_Delta`.
- Окружения: `e_current`, `e_identity`, `e_target_ptr` (init `&e_identity`).
- Очередь событий: `s_events` (`xr_vector<pair<ref_sound_data_ptr, float>>`).
- Геометрия: `geom_DB` (`CDB::COLLIDER*`, только non-editor),
  `geom_SOM`/`geom_MODEL`/`geom_ENV` (`CDB::MODEL*`).
- Контейнеры: `s_sources` (unordered_map имя→source), `s_emitters`,
  `s_targets`/`s_targets_defer`/`s_targets_pu`, `s_environment`
  (`SoundEnvironment_LIB*`), `s_user_environment`.
- Кэш: `CSoundRender_Cache cache`, `cache_bytes_per_line`.
- `m_iPauseCounter` (init **1** — quirk: не 0).
- Чистые виртуальные: `_initialize`/`_clear` (реализует A),
  `set_master_volume`, `update_listener`, `set_listener`/`get_listener`,
  `commit`.

`extern CSoundRender_Core* SoundRender;` — глобал среднего слоя
(определён в `SoundRender_Core.cpp`, обе глобалы стартуют nullptr).

## 8. `CNotificationClient` (объявление)

`NotificationClient.h`: `: IMMNotificationClient` (Windows Core Audio) —
COM-клиент с `IMMDeviceEnumerator*`, `m_bComInitialized`. Реализация
(`.cpp`, порц. 10): `Start()` — `CoInitialize` +
`CoCreateInstance(MMDeviceEnumerator)` +
`RegisterEndpointNotificationCallback`; `OnDefaultDeviceChanged`
(только `eRender`+`eConsole`) → `SoundRenderA->bPendingDefaultDeviceSwitch
= TRUE`; `OnDeviceRemoved/StateChanged/PropertyValueChanged` →
`bPendingDeviceListRefresh = TRUE`. Флаги читаются на звуковом потоке
— hot-swap без блокировок (реализация — `switch_device`/`refresh_devices`,
CoreA).

## 9. Взаимодействие

- **xrEngine**: точка входа (`InitSound1/2`, `destroySound`,
  `SoundProcessor::OnFrame` — `update`), геометрия (`IGame_Level::Load`
  → `set_geometry_occ`), консоль (`snd_restart`, `sounddevice`),
  статистика (`CStats` → `statistic`/`statistic_ext`).
- **xrGame**: `ref_sound` в объектах, `Feel::Sound`
  (ит.2 порц.9), `SoundEvent_Register` (ит.2 порц.6), Lua-звук
  (ит.2 порц.12).
- **Далее в итерации 4**: internals `CSoundRender_Core` —
  [core.md](core.md) (порц. 6); процессор + окружение —
  [core-processor-env.md](core-processor-env.md) (7); Source/Target/
  Cache/окклюзия — [source-target.md](source-target.md) (8); Emitter
  FSM — [emitter.md](emitter.md) (9); устройства/`cl_intersect.h`/
  статистика — [devices-misc.md](devices-misc.md) (10).
