# xrCore — Фундамент

`xrCore` — самый нижний слой движка. **Без внутренних зависимостей**: не `#include`'ит ни один другой модуль. От него начинается всё: типы, память, строки, лог, файловая система, конфиги, сжатие, архивы, математика.

> **Статус**: ✅ разобрано (итерация 1).

## Ответственность

`xrCore` отвечает за **всё, что нужно, чтобы компилировать и запускать C++-код движка**, но не знает про игру, рендер или сеть:

- **Типы и математика**: `s8/u8/.../u64`, `Fvector`/`Fmatrix`/`Fquaternion`, `FPU`, `CPU`-таймеры.
- **Память**: переопределённые `operator new`/`delete`, пулы, `xr_alloc`/`xr_free`, `vminfo`.
- **Строки**: `shared_str` (ref-counted, CRC-based), `str_container`, `xr_string` (std::string), `string_N` буферы.
- **Логирование**: `Msg`, `Log`, `FormatString`, `LogCallback`.
- **Файловая система**: `IReader`/`IWriter`/`CMemoryWriter`, виртуальные файлы, `FileSystem`.
- **Конфиги**: `CInifile` (LTX/INI формат с DLTX-override'ами).
- **Сжатие**: LZO (`rt_compressor`), PPMD, LzHuf.
- **Локализатор**: `CLocatorAPI` — работа с LTX-архивами (папка → виртуальный файл).
- **CRC32**: `crc32`, `path_crc32`.
- **Синхронизация**: `xrCriticalSection`, `Lock`, `ScopeLock`, `xrSyncronize`.
- **Время**: `CTimer`, `FTimer`, `CPU::QPC`, `CPU::GetCLK`.

## Чего xrCore НЕ делает

- Не знает про `CObject`, `CGameObject`, рендер, AI, физику, сеть.
- Не содержит игровых классов.
- Не содержит UI.
- Не содержит Lua.

## Структура модуля

| Подмодуль | Файлы | Страница |
|---|---|---|
| Типы и математика | `_types.h`, `_vector3d.h`, `_matrix.h`, `_quaternion.h`, `_sphere.h`, `_fbox.h`, `_plane.h`, `_cylinder.h`, `_math.h`, `cpuid.h` | [types-math](types-math.md) |
| Память | `xrMemory.h`/`.cpp`, `xrMemory_pso.h`, `xrMemory_POOL.h`, `xrMEMORY_POOL.h`, `memory_monitor.h`, `SubAlloc.hpp` | [memory](memory.md) |
| Строки | `xrstring.h`/`.cpp`, `string_concatenations.h`, `xr_trims.h` | [strings](strings.md) |
| Логирование | `log.h`/`.cpp` | [logging](logging.md) |
| Файловая система | `FS.h`/`.cpp`, `FS_impl.h`, `FileSystem.h`/`.cpp`, `file_stream_reader.h`, `stream_reader.h` | [filesystem](filesystem.md) |
| INI / конфиги | `xr_ini.h`/`.cpp`, `ini_id_loader.h`, `ini_table_loader.h` | [inifile](inifile.md) |
| Сжатие | `rt_compressor.h`/`.cpp`, `rt_lzo1x*.cpp`, `ppmd_compressor.h`/`.cpp`, `lzhuf.h`/`.cpp`, `LzHuf.cpp`, `compression_ppmd_stream.h` | [compression](compression.md) |
| Локализатор | `LocatorAPI.h`/`.cpp`, `LocatorAPI_defs.h`, `LocatorAPI_auth.cpp`, `LocatorAPI_Notifications.cpp`, `ELocatorAPI.h` | [locator](locator.md) |
| Временны́е/системные | `FTimer.h`/`.cpp`, `_math.h` (CPU/QPC), `xrSyncronize.h`/`.cpp`, `Lock.hpp`, `ScopeLock.hpp`, `clsid.h`, `crc32.cpp`, `cpuid.h`/`.cpp`, `net_utils.h`/`.cpp`, `intrusive_ptr.h`, `profiler.h` | [system](system.md) |

## Ключевые глобалы

| Глобал | Тип | Назначение |
|---|---|---|
| `Core` | `xrCore` | Главный объект: `ApplicationName`, `ApplicationPath`, `UserName`, `dwFrame` |
| `Memory` | `xrMemory` | Менеджер памяти |
| `g_pStringContainer` | `str_container*` | Контейнер строк (для `shared_str`) |
| `LogFile` | `xr_vector<xr_string>` | Буфер логов |
| `CPU::ID` | `_processor_info` | Информация о процессоре (CPUID) |
| `CPU::clk_per_second` | `u64` | Такты CPU в секунду |

## Инициализация

`xrCore::_initialize(ApplicationName, LogCallback, init_fs, fs_fname)` — вызывается при старте:
1. Инициализирует `Memory`.
2. Инициализирует `g_pStringContainer`.
3. Инициализирует лог (`CreateLog`).
4. Инициализирует CPU (`_initialize_cpu`).
5. Если `init_fs` — инициализирует `CLocatorAPI` (архивы).

`xrCore::_destroy()` — обратный порядок.

## Связи

- **Тянет**: ничего (внутри движка).
- **Тянут**: все остальные модули (`xrEngine`, `xrRender`, `xrGame`, ...).

## Связанные страницы

- [Архитектура: Обзор](../../architecture/overview.md) — где `xrCore` в слоях.
- [Зависимости](../../architecture/dependencies.md) — полный граф.
