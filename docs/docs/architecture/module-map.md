# Карта модулей

Таблица всех модулей `src/`: ответственность, ключевые классы/файлы, от кого зависит, кто зависит от него.

> Зависимости помечены как **тянет** (этот модуль `#include`'ит) и **тянут** (этот модуль `#include`'ится).

## Фундамент

### `xrCore` — [разобрано](../modules/xr-core/index.md)

|                     |                                                                                                                                                                                                                                                                   |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ответственность** | Типы, математика (вектора/матрицы/квaternion), память (операторы `new`/`delete`, пулы), строки (`shared_str`), логирование, файловая система + виртуальные файлы, INI/конфиги, сжатие (LZO/PPMD/LzHuf), локализатор LTX-архивов, CRC32, таймеры, трассировка CPU. |
| **Ключевые классы** | `xrCore`, `IReader`/`IWriter`/`CMemoryWriter`, `CInifile`, `CLocatorAPI`, `shared_str`, `str_container`, `xrMemory`, `Memory`, `Fvector`/`Fmatrix`/`Fquaternion`, `CTimer`                                                                                        |
| **Ключевые файлы**  | `xrCore.h`, `_types.h`, `_vector3d.h`, `_matrix.h`, `_quaternion.h`, `xrMemory.h`, `xrstring.h`, `log.h`, `FS.h`, `FileSystem.h`, `xr_ini.h`, `rt_compressor.h`, `lzhuf.h`, `ppmd_compressor.h`, `LocatorAPI.h`, `FTimer.h`, `_math.h`, `cpuid.h`                 |
| **Тянет**           | ничего (внутри движка)                                                                                                                                                                                                                                            |
| **Тянут**           | все остальные модули                                                                                                                                                                                                                                              |

## Ядро

### `xrEngine` — [итерация 2](../modules/xr-engine/index.md)

|                     |                                                                                                                                                                                                                                                                                                                                                               |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Ответственность** | Цикл приложения: `CEngine`, `CSheduler` (планировщик кадров), `device` (окно, таймеры, регистраторы событий), ввод (`xr_input`, XInput), консоль (`xr_ioc_cmd`, `Text_Console`), Lua-биндинг (`ai_script_lua_extension`, `script_binder`), эффекты/постобработка (`Effector`, `EffectorPP`), статистика (`Stats`), `xr_object` (базовый рендерящийся объект). |
| **Ключевые классы** | `CEngine`, `CSheduler`, `ISheduled`, `CObject` (`xr_object`), `CEffector`, `CEffectorPP`, `Text_Console`, `xr_input`                                                                                                                                                                                                                                          |
| **Ключевые файлы**  | `Engine.h`/`Engine.cpp`, `xrSheduler.h`/`xrSheduler.cpp`, `device.h`/`device.cpp`, `xr_object.h`, `EventAPI.h`, `EngineAPI.h`, `ISheduled.h`, `ai_script_lua_extension.h`, `xr_ioc_cmd.h`                                                                                                                                                                     |
| **Тянет**           | `xrCore`, `xrCPU_Pipe`, `Layers/xrRender` (через `Include/`)                                                                                                                                                                                                                                                                                                  |
| **Тянут**           | `xrGame`, `xrRenderPC_R*`                                                                                                                                                                                                                                                                                                                                     |

### `xrCPU_Pipe`

|                     |                                                                                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Ответственность** | CPU-dispatch: набор оптимизаций (SIMD), выбираемых по `_processor_info`. Функции из `xrMath`/`_vector3d` имеют несколько версий (SSE/SSSE3/AVX). |
| **Тянет**           | `xrCore`                                                                                                                                         |
| **Тянут**           | `xrEngine`                                                                                                                                       |

## Рендер

### `Layers/xrAPI`

C-интерфейс рендера: `R_*` функции, `xrRender`-агностичный контракт между движком и бэкендом.

### `Layers/xrRender` — [итерация 3](../modules/renderer/index.md)

Бэкенд-агностичный pipeline: `CBlender_Compile`, константы (`R_constant`, `RCache`), `SetMapping` (привязка имён констант к GPU-регистрам), постобработка, `xrRender_console` (включая [shader bus](../modules/renderer/shader-bus.md)).

### `Layers/xrRenderDX9` / `xrRenderDX10`

Конкретные бэкенды D3D9 / D3D10.

### `Layers/xrRenderPC_R1..R4`

Версии бэкенда. **R4** — основной для Anomaly (D3D9, расширенный pipeline). R1–R3 — устаревшие/альтернативные.

## Игра

### `xrGame` — [итерации 5–10](../modules/game/index.md)

| Подсистема   | Ключевые файлы                                              | Страница                                      |
| ------------ | ----------------------------------------------------------- | --------------------------------------------- |
| Объекты      | `GameObject.h`, `Actor.h`, `Entity.h`, `moving_object.h`    | [objects](../modules/game/objects.md)         |
| AI (ALIFE)   | `alife_*`, `agent_manager*`, `stalker_*`, `smart_cover*`    | [alife](../modules/game/alife.md)             |
| Пейтфиндинг  | `level_graph.h`, `path_manager_*`, `a_star.h`, `dijkstra.h` | [pathfinding](../modules/game/pathfinding.md) |
| Физика (PH)  | `PH*`, `physics_*`, `PhysicObject.h`                        | [physics](../modules/game/physics.md)         |
| Оружие       | `Weapon*.cpp`                                               | [weapons](../modules/game/weapons.md)         |
| Инвентарь    | `Inventory*`, `inventory_*`                                 | [inventory](../modules/game/inventory.md)     |
| Зоны/эффекты | `Zone*`, `CustomZone.h`, `space_restriction*`               | [zones](../modules/game/zones.md)             |
| UI           | `ui_base.h`, `UIPanelsClassFactory.cpp`, `CustomHUD`        | [ui](../modules/game/ui.md)                   |

### `xrParticles`, `xrSound`, `xrPhysics`, `xrXMLParser`

Частицы ([xrParticles](../modules/xr-particles/index.md)), звук ([xrSound](../modules/xr-sound.md)), обвязка PhysX, XML-парсер.

## Сервер

### `xrCDB` — CSE

Серверные сущности: `CSE_Abstract` иерархия, сериализация состояния для сети.

### `xrServerEntities` — [итерация 11](../modules/server.md)

Мост CSE ↔ игровая логика, Lua-хелпер сервера (`script_lua_helper`).

### `xrNetServer` — [итерация 12](../modules/network.md)

TCP-сервер, протокол, валидация клиента (`svclient_validation`), DSA-подпись (`xr_dsa_signer`/`verifyer`).

## Вспомогательное

### `compressor`

Утилита сжатия LTX-архивов (используется `LzHuf`, `ppmd`, `lzo` из `xrCore`).

### `sdk/`

Бинарные SDK: заголовки (`include/`), библиотеки (`libraries/`), бинарники (`binaries/`) — PhysX, Theora и т.д.

### `gamedata/`

Игровые данные: `.ltx`-конфиги, скрипты, текстуры, модели. Не часть движка, но описывается в [Раскладка gamedata](../data/gamedata-layout.md).
