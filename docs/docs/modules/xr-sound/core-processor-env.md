# xrSound — Core: процессор и окружение

> Итерация 4, порцион 7. Файлы: `SoundRender_Core_Processor.cpp` (333 стр.),
> `SoundRender_Environment.h/.cpp` (196 + 283 стр.), `SoundRender_CoreA.cpp`
> (только EFX-вектор: `LoadEffect`/`load_reverb`/`commit`/`set_listener`/
> `get_listener`).

Страница закрывает кросс-обязательство ит.2 порц.8
(`CSound_environment`/`SoundEnvironment_LIB`) и EFX/EAX-связку среднего слоя
с устройством. Ядро среднего слоя — [Core — ядро](core.md) (порц. 6),
интерфейс — [Интерфейс и точка входа](interface.md) (порц. 5).

## 1. Ответственность

Эта страница — два контура, которые [Core — ядро](core.md) только описывает
сверху:

- **Процессорный контур** (`SoundRender_Core_Processor.cpp`) — то, что
  *делается* внутри кадрового тика `update` (порц. 6 §5): полный разбор
  шагов, EFX-сглаживание окружения, `update_events`, `statistic`, и
  геометрия окклюзии `get_occlusion`/`get_occlusion_to`.
- **Окружение** (`SoundRender_Environment.h/.cpp`) — модель EAX-реверберации:
  поля EFX/EAX, `load`/`save` с версиями, `lerp`/`clamp`/`set_default`/
  `set_identity`, библиотека `SoundEnvironment_LIB` (`sEnvironment.xr`),
  массив `reverbs[]` (113 EFX-пресетов) и применение их к OpenAL-эффекту
  в CoreA.

Не здесь: OpenAL-устройство/`switch_device`/`slot` — [Устройства и прочее]
(devices-misc.md) (порц. 10); Source/Target/Cache — [Source + Target]
(source-target.md) (порц. 8); Emitter FSM — [Emitter](emitter.md) (порц. 9).

## 2. `SoundRender_Core_Processor.cpp` — кадровый тик и события

### 2.1 `i_play(S, _loop, delay)`

```
VERIFY(S->_p->feedback == 0);
E = xr_new<CSoundRender_Emitter>();
S->_p->feedback = E;
E->start(S, _loop, delay);
s_emitters.push_back(E);
```

Эмиттер привязывается к `feedback` данных *до* `start` (порядок важен —
`start` может сразу запросить target через `i_start`). Internals `start` —
порц. 9.

### 2.2 `update(P, D, N)` — порядок шагов

Весь тик под `bLocked = TRUE`…`FALSE`. Схема и семантика шагов 1–8 —
в [Core — ядро §5](core.md#5-кадровый-тик-updatep-d-n) (мермайнд-карта);
здесь — детали, не вынесенные туда:

- **dt** — `fTimer_Delta = Timer.GetElapsed_sec() - fTimer_Value`
  (секунды, `fTimer_Value` обновляется). Отдельная переменная `dt_sec` —
  алиас. Закомментирована миллисекундная ветка `float dt = .../1000.f`.
- **Шаг 1 (r-emitters)**: `E->update` может остановить эмиттера, поэтому
  `T->get_emitter()` **перепроверяется после** `update`; priority обновляется
  из живого эмиттера или ставится `-1`.
- **Шаг 2 (остальные)**: `marker != s_emitters_u` — один `update` на эмиттер
  на кадр (r-emitters уже помечены шагом 1); `!isPlaying()` →
  `xr_delete` + `erase` (обратный индекс `it--`).
- **Шаг 3–4 (targets)**: `s_targets_pu++` (обратимый счётчик —
  «round-robin» закомментирован: `/*if (PU == it)*/`); уже рендерящиеся —
  `fill_parameters` + `update`, только что привязанные — в
  `s_targets_defer`; затем `std::unique` + `fill_parameters` (commit
  параметров) **до** запуска рендера.
- **Шаг 5 (EFX)** — см. §3.2.
- **Шаг 6** — `update_listener(P, D, N, dt_sec)` (виртуал; CoreA-реализация
  — порц. 10).
- **Шаг 7** — `s_targets_defer`: **down-cast `this` → `CSoundRender_CoreA*`**
  ради `slot`; при `m_is_supported` — `Ptr->SetSlot(Core->slot)`, затем
  `Ptr->render()`.
- **Шаг 8** — `update_events()` (см. ниже).

### 2.3 `update_events()`

```
static u32 g_saved_event_count = 0;   // статическая, живёт между кадрами
g_saved_event_count = s_events.size();
for (it) Handler(s_events[it].first, s_events[it].second);
s_events.clear_not_free();
```

`Handler` — callback-указатель (устанавливается xrEngine/xrGame; поле
`CSound_manager_interface`). `g_saved_event_count` — статик, потому что
`statistic` (§2.4) читает его в свой момент; `clear_not_free` — события
пересоздаются без освобождения буфера.

### 2.4 `statistic(dest, ext)`

- `dest` (`CSound_stats`): `_rendered` — targets с `get_emitter() &&
  get_Rendering()`; `_simulated = s_emitters.size()`; `_cache_hits`/
  `_cache_misses` из `cache`; `_events = g_saved_event_count`; затем
  `cache.stats_clear()` (окно — кадр).
- `ext` (`CSound_stats_ext`): по каждому эмиттеру `SItem`: `_3D = !b2D`,
  `_rendered = !!target`, `params = p_source`, `volume = smooth_volume`;
  с `owner_data` — `name = source()->fname`, `game_object`/`game_type`/
  `type`; без — `game_object = 0`, `game_type = 0`, `type = st_Effect`.

Интерфейс структур — порц. 5 (`CSound_stats`/`CSound_stats_ext`); вызов из
`Device.Statistic->Sound` (xrEngine, ит.2 порц.12) — [Интерфейс](
interface.md).

### 2.5 `get_occlusion(P, R, occ)` — трёхступенчатая

Радиус `R` — радиус источника (смещение `pos = P + random_dir()*R`);
`base = listener_position()`; `dir = (pos - base)` нормализован;
`range = |pos - base|`.

1. **Кэш треугольника** (только если `geom_MODEL`): `CDB::TestRayTri(base,
   dir, occ, _u, _v, _range, true)` — тест против **трёх вершин**,
   закэшированных предыдущим кадром в `occ[0..2]`. Hit в `(0, range)` →
   `occ_value = psSoundOcclusionScale`, полный тест пропускается.
2. **Полный тест** (`geom_MODEL`, `OPT_ONLYNEAREST`): ближайший tri →
   его 3 вершины **записываются в `occ[0..2]`** (кэш для шага 1 следующего
   кадра), `occ_value = psSoundOcclusionScale`.
3. **SOM** (`geom_SOM`, `OPT_CULL`) — **всегда**, независимо от 1–2:
   по всем hit `occ_value *= *(float*)&R->dummy` (float bit-cast из
   `dummy` — окклюзия, заданная `set_geometry_som`, порц. 6 §7).

`_EDITOR` — те же запросы через `ETOOLS::` (вместо `geom_DB`).

### 2.6 `get_occlusion_to(hear_pt, snd_pt, dispersion = 0.2f)`

Упрощённая версия для «слышимости между двумя точками» (кросс-обязательство
ит.3 порц.4): `pos = snd_pt + random_dir()*dispersion`, ray
`pos → hear_pt` (`dir.sub(pos, hear_pt)`), только `geom_SOM` `OPT_CULL`,
`occ_value *= *(float*)&R->dummy` по всем hit. Без model-теста и без кэша.

## 3. Oкружение: `CSoundRender_Environment` + `SoundEnvironment_LIB`

### 3.1 Поля (EAX-reverb)

`version` (`sdef_env_version`), `name`, `Environment` (id исходного
окружения — индекс в `reverbs[]`/библиотеке) и 19 EAX-полей, 1-1
соответствующих параметрам `AL_EAXREVERB_*`: `Room`/`RoomHF`/`RoomLF`/
`RoomRolloffFactor`, `DecayTime`/`DecayHFRatio`/`DecayLFRatio`/
`DecayHFLimit` (int), `Reflections`/`ReflectionsDelay`/`ReflectionsPan[3]`,
`EchoTime`/`EchoDepth`, `Reverb`/`ReverbDelay`/`ReverbPan[3]`,
`EnvironmentSize`/`EnvironmentDiffusion`, `AirAbsorptionHF`,
`ModulationTime`/`ModulationDepth`, `Density`, `HFReference`/`LFReference`.

### 3.2 `set_default` / `set_identity` / `lerp` / `clamp`

- **`set_default()`** — копирует из `reverbs[0]` (**Generic-пресет**,
  индекс 0 массива; вызов **без аргументов** — quirk, см. §6):
  `Room = 0.f` (закомментирован `reverbs->flGain`), остальные поля —
  из `reverbs->...`.
- **`set_identity()`** — `set_default()` + `clamp()`. Используется для
  `e_identity`/`e_current` (ctor Core, порц. 6 §2) и static `identity`
  в `get_environment`.
- **`lerp(A, B, f)`** — `fi = 1 - f`; линейно интерполируются **19 полей**
  (`Room`…`EchoDepth`, включая `DecayLFRatio`/`ModulationTime`/`Density`/
  `HFReference`/`LFReference`); `ReflectionsPan`/`ReverbPan`/
  `EnvironmentSize`… — **не интерполируются** (кроме тех, что в списке);
  завершается `clamp()`. Вызывается в EFX-блоке `update` (порц. 6 §5 шаг 5).
- **`clamp()`** — 20 `::clamp` на диапазоны `AL_EAXREVERB_MIN_*`/
  `AL_EAXREVERB_MAX_*` (включая `DecayHFLimit` как int).

### 3.3 `load(IReader*)` / `save(IWriter*)` — версии формата

`load`:

- `version = r_u32()`, `name = r_stringZ()`.
- **`version <= sdef_env_version` (4)**: `Room`/`RoomHF`/`Reverb`/
  `AirAbsorptionHF` читаются **в мБ** и конвертируются
  `mB_to_gain = powf(10.f, mb / 2000.f)` (миллидецибел → линейный gain).
- **`version > 4`**: эти же поля — уже линейные, читаются как есть.
- Общие поля: `RoomRolloffFactor`, `DecayTime`, `DecayHFRatio`,
  `Reflections`, `ReflectionsDelay`, `ReverbDelay`, `EnvironmentSize`,
  `EnvironmentDiffusion`, `Environment` (u32).
- **`version == sndenv_ver_extended` (5)**: добираются EFX/EAX-поля
  `RoomLF`, `ReflectionsPan[0]`, `ReverbPan[0]`, `DecayHFLimit` (u32),
  `EchoTime`, `EchoDepth`, **`ReverbDelay` (перечитывается второй раз)**,
  `DecayLFRatio`, `ModulationTime`, `ModulationDepth`, `HFReference`,
  `LFReference`, `Density`.

`save`: **всегда** пишет `sdef_env_version` (4) — только базовые поля
(`Room`…`Environment`), **без** extended-блока. Т.е. save → v4,
load — v4/v5.

### 3.4 `SoundEnvironment_LIB`

`Load(name)` — `R_ASSERT(library.empty())`, `reserve(256)`, по каждому
chunk'у файла — `xr_new<CSoundRender_Environment>()` + `load(C)` (chunk =
одно окружение); `Save(name)` — обратное (chunk per env); `Unload()` —
`xr_delete` всех; `GetID(name)` — **линейный** `stricmp` → индекс (или -1);
`Get(name)`/`Get(int id)`; `Append(parent)` — `xr_new` (копия `parent` или
новый); `Remove(name)`/`Remove(id)`; `Library()`.

Файл — `$game_data$/sEnvironment.xr` (`SNDENV_FILENAME`), загрузка —
`env_load` (порц. 6 §3). Названия окружений **кейс-независимы**
(`stricmp`).

### 3.5 `reverbs[]` — 113 EFX-пресетов

`constexpr EFXEAXREVERBPROPERTIES reverbs[]` — в `SoundRender_Environment.h`,
`#include <AL/efx-presets.h>`. Группы: Default (27: Generic…Psychotic),
Castle (9), Factory (9), Ice Palace (9), Space Station (9), Wooden Galleon
(9), Sports (7), Prefab (5), Dome & Pipe (5), Outdoors (5), Mood (3),
Driving (8), City (6), Misc (3) = **113** (комментарии в .h группируют;
фактически — массив пресетов `EFX_REVERB_PRESET_*` из OpenAL-SDK).
Индекс 0 = `EFX_REVERB_PRESET_GENERIC` — используется `set_default` и
`LoadEffect` (CoreA).

## 4. EFX-применение в CoreA (EFX/EAX-вектор)

Core (средний слой) **не трогает OpenAL** — применение окружения к
устройству делает `CSoundRender_CoreA` (интерфейс: порц. 10; здесь —
только EFX-связка):

- **`LoadEffect()`** — `alGenEffects(1, &effect)`,
  `load_reverb(effect, &reverbs[0])` (Generic-пресет как старт).
  Если `psSoundFlags.test(ss_EFX)` и нет ошибки → `m_is_supported = true`
  + `alGenAuxiliaryEffectSlots(1, &slot)`; иначе `m_is_supported = false`
  (эффект удаляется). Вызывается из `_initialize` и `switch_device`
  (порц. 10).
- **`load_reverb(effect, reverb)`** — `AL_EFFECT_EAXREVERB` + 22
  `alEffectf`/`alEffectfv`/`alEffecti` из `EFXEAXREVERBPROPERTIES`
  (Density…DecayHFLimit).
- **`commit()`** — `AL_EFFECTSLOT_GAIN = 1.f`,
  `AL_EFFECTSLOT_AUXILIARY_SEND_AUTO = false`,
  `AL_EFFECTSLOT_EFFECT = effect` (копирует свойства эффекта в слот;
  эффект можно менять/удалять после).
- **`set_listener(env)`** — два режима:
  - `reverb_overwrite == TRUE` — 22 `psReverb*` (cvar'ы, порц. 5)
    **вместо** `env`;
  - иначе — 10 полей из `env` (`Diffusion`, `Room`→GAIN, `RoomHF`,
    `DecayTime`, `DecayHFRatio`, `Reflections`, `ReflectionsDelay`,
    `Reverb`→LATE_REVERB_GAIN, `AirAbsorptionHF`, `RoomRolloffFactor`);
    при `env.version == sndenv_ver_extended` — ещё 10 (GAINLF,
    DECAY_HFLIMIT, ECHO_TIME/DEPTH, LATE_REVERB_DELAY, DECAY_LFRATIO,
    MODULATION, HF/LF REFERENCE, DENSITY). `REFLECTIONS_PAN`/
    `LATE_REVERB_PAN` — **закомментированы**.
- **`get_listener(env)`** — обратное чтение 22 параметров.

### 4.1 EFX-блок в `update` (связка)

Из [Core — ядро §5](core.md#5-кадровый-тик-updatep-d-n) (только
`m_is_supported`):

```
bListenerMoved? → e_target_ptr = get_environment(P) (или &e_identity)
alpha = 1 - exp(log(1 - 0.95) * dt_sec / snd_efx_environment_change_time)
clamp(alpha, 0, 1)
e_current.lerp(e_current, *e_target_ptr, alpha)
set_listener(e_current)
commit()
```

`get_environment(P)` — порц. 6 §7 (ray вниз `(0,-1,0)`, front/back по
`dir·n < 0`, miss → static identity). `bListenerMoved` ставится
`env_apply()` (порц. 6 §7) — т.е. пересчёт цели **только при изменении
окружения уровня**, не каждый кадр. `snd_efx_environment_change_time = 1.66`
(порц. 5).

## 5. Кросс-обязательства

| Откуда | Что | Статус |
| --- | --- | --- |
| ит.2 порц.8 | `CSound_environment`/`SoundEnvironment_LIB` | **закрыто** (§3) |
| ит.2 порц.12 | `CSound_stats_ext`/`statistic()` | реализация — §2.4 (интерфейс — порц. 5) |
| ит.3 порц.4 | `get_occlusion_to` | реализация — §2.6 |

## 6. Обнаружено (quirks)

1. **`set_default()` использует `reverbs->` (указательную арифметику)** —
   `reverbs->flGainHF` и т.д., хотя `reverbs` — массив (декларация
   `constexpr EFXEAXREVERBPROPERTIES reverbs[]`). В выражении `reverbs->x`
   имя массива вырождается в указатель на первый элемент, поэтому читается
   **всегда `reverbs[0]` (Generic-пресет)**, независимо от имени окружения.
   `Room = 0.f` (строка `reverbs->flGain` закомментирована) — Room не
   берётся из пресета.
2. **`save` всегда пишет v4** — extended-поля (v5) **теряются при
   сохранении**; load v5 работает, save v5 — нет.
3. **`ReverbDelay` в load v5 перечитывается дважды** (базовый блок +
   extended-блок) — формат v5 имеет два `ReverbDelay`.
4. **`lerp` не интерполирует** `ReflectionsPan`/`ReverbPan`/`Environment`/
   `RoomLF` и др. — только 19 полей; панорамирование окружения не
   сглаживается.
5. **`set_listener` (CoreA)** — `reverb_overwrite` **полностью
   переопределяет** `env` 22 cvar'ами `psReverb*`; при `FALSE` — 10+10
   полей из `env`, `REFLECTIONS_PAN`/`LATE_REVERB_PAN` закомментированы.
6. **`get_environment` — static `identity`** — локальный статик
   (`CSoundRender_Environment identity`), **не** `e_identity` поля Core;
   `set_identity()` вызывается при miss.
7. **`g_saved_event_count` — static** в `update_events` (не поле) —
   потому что `statistic` читает его в свой момент.
8. **`s_targets_pu`** — счётчик round-robin, но сама логика
   (`if (PU == it)`) **закомментирована** — targets обновляются
   последовательно.

## 7. Взаимодействие

- [Core — ядро](core.md) (порц. 6) — `update`-каркас, `get_environment`,
  `set_geometry_env`, `env_load`/`env_apply`.
- [Интерфейс и точка входа](interface.md) (порц. 5) — `CSound_stats`/
  `CSound_stats_ext`, `psReverb*`, `snd_efx_environment_change_time`,
  `sdef_env_version`/`sndenv_ver_extended`.
- [Устройства и прочее](devices-misc.md) (порц. 10) — internals CoreA:
  `switch_device`, `slot`, `update_listener`, `default_device_changed`.
- [Source + Target](source-target.md) (порц. 8) — `get_occlusion`/
  `get_occlusion_to` consumer'ы (device-side).
