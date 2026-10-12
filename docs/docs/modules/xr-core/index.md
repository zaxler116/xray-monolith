# xrCore — Фундамент

`xrCore` — самый нижний слой движка. **Без внутренних зависимостей**: не `#include`'ит ни один другой модуль. От него начинается всё: типы, память, строки, лог, файловая система, конфиги, сжатие, архивы, математика.

Что это значит на практике: «нижний слой» — это не про важность, а про **то, что видно в вашем `#include`**. Если ваш код включает `xrMemory.h`, `xrstring.h`, `xr_ini.h` или `LocatorAPI.h` — вы уже работаете в `xrCore`, даже если пишете «игровой» код. Практически любой файл движка включает хотя бы один заголовок `xrCore` прямо или косвенно (через `xrCore.h`), поэтому правила этого слоя (единый аллокатор, `shared_str` вместо `std::string`, `Msg` вместо `printf`) действуют на весь проект.

## Как читать xrCore

Страницы модуля расположены **по порядку слоёв**: каждый следующий слой строится на предыдущем, так что чтение в этом порядке — кратчайший путь к пониманию.

1. [Типы и математика](types-math.md) — с этого начинается любой код движка: `s8`–`u64`, `Fvector`/`Fmatrix`, CPU-таймеры. Всё остальное использует эти типы.
2. [Память](memory.md) — единый аллокатор и `xr_alloc`/`xr_free`. Строки, FS и конфиги ниже выделяются память именно так.
3. [Строки](strings.md) — `shared_str` строится на памяти и CRC из типов; это базовая единица данных для логгера, FS и INI.
4. [Логирование](logging.md) — `Msg`/`Log` работают до инициализации всего остального и пишут через строки.
5. [Файловая система](filesystem.md) — `IReader`/`IWriter` и виртуальные файлы; поверх памяти и строк.
6. [INI / конфиги](inifile.md) — `CInifile` читает LTX-файлы через файловую систему.
7. [Сжатие](compression.md) — LZO/PPMd применяются к данным, которые читает ФС и упаковывает локализатор.
8. [Локализатор](locator.md) — LTX-архивы: объединяет сжатие, FS и INI в виртуальную файловую систему, через которую игра открывает файлы.
9. [Временны́е/системные](system.md) — инфраструктура (таймеры, синхронизация, CRC, профайлер), которая опирается на все предыдущие слои.

Если времени мало: прочитайте хотя бы 1–3 (типы → память → строки) — на них опирается 90% кода движка.

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

| Подмодуль | Одним предложением | Файлы | Страница |
|---|---|---|
| Типы и математика | Базовые типы и математика, на которых построен весь движок: `s8`–`u64`, `Fvector`/`Fmatrix`/`Fquaternion`, CPU-таймеры | `_types.h`, `_vector3d.h`, `_matrix.h`, `_quaternion.h`, `_sphere.h`, `_fbox.h`, `_plane.h`, `_cylinder.h`, `_math.h`, `cpuid.h` | [types-math](types-math.md) |
| Память | Единый аллокатор вместо разрозненных `malloc`'ов: все `new`/`delete` движка уходят в `Memory`, плюс пулы и `xr_alloc` | `xrMemory.h`/`.cpp`, `xrMemory_pso.h`, `xrMemory_POOL.h`, `xrMEMORY_POOL.h`, `memory_monitor.h`, `SubAlloc.hpp` | [memory](memory.md) |
| Строки | `shared_str` — интернированные строки: одна копия в памяти, сравнение по CRC, а не побайтово | `xrstring.h`/`.cpp`, `string_concatenations.h`, `xr_trims.h` | [strings](strings.md) |
| Логирование | `Msg`/`Log` — единственный канал вывода в консоль и файл; работает ещё до инициализации FS | `log.h`/`.cpp` | [logging](logging.md) |
| Файловая система | Абстракция чтения/записи (`IReader`/`IWriter`) и виртуальные пути: код не знает, файл ли это или запись в LTX-архиве | `FS.h`/`.cpp`, `FS_impl.h`, `FileSystem.h`/`.cpp`, `file_stream_reader.h`, `stream_reader.h` | [filesystem](filesystem.md) |
| INI / конфиги | `CInifile` — чтение LTX/INI-конфигов (`settings.ltx`, секции уровней) с поддержкой override'ов | `xr_ini.h`/`.cpp`, `ini_id_loader.h`, `ini_table_loader.h` | [inifile](inifile.md) |
| Сжатие | LZO/PPMd/LzHuf — кодеки, которыми упакованы LTX-архивы и данные в них | `rt_compressor.h`/`.cpp`, `rt_lzo1x*.cpp`, `ppmd_compressor.h`/`.cpp`, `lzhuf.h`/`.cpp`, `LzHuf.cpp`, `compression_ppmd_stream.h` | [compression](compression.md) |
| Локализатор | `CLocatorAPI` — виртуальная ФС над LTX-архивами: `FS.r_open("levels/test.ltx")` ищет файл и внутри архива | `LocatorAPI.h`/`.cpp`, `LocatorAPI_defs.h`, `LocatorAPI_auth.cpp`, `LocatorAPI_Notifications.cpp`, `ELocatorAPI.h` | [locator](locator.md) |
| Временны́е/системные | Инфраструктура: таймеры (`CTimer`/`CPU::QPC`), критические секции (`Lock`/`ScopeLock`), CRC32, CPUID, профайлер | `FTimer.h`/`.cpp`, `_math.h` (CPU/QPC), `xrSyncronize.h`/`.cpp`, `Lock.hpp`, `ScopeLock.hpp`, `clsid.h`, `crc32.cpp`, `cpuid.h`/`.cpp`, `net_utils.h`/`.cpp`, `intrusive_ptr.h`, `profiler.h` | [system](system.md) |

## Ключевые глобалы

| Глобал                | Тип                    | Назначение                                                                  |
| --------------------- | ---------------------- | --------------------------------------------------------------------------- |
| `Core`                | `xrCore`               | Главный объект: `ApplicationName`, `ApplicationPath`, `UserName`, `dwFrame` |
| `Memory`              | `xrMemory`             | Менеджер памяти                                                             |
| `g_pStringContainer`  | `str_container*`       | Контейнер строк (для `shared_str`)                                          |
| `LogFile`             | `xr_vector<xr_string>` | Буфер логов                                                                 |
| `CPU::ID`             | `_processor_info`      | Информация о процессоре (CPUID)                                             |
| `CPU::clk_per_second` | `u64`                  | Такты CPU в секунду                                                         |

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

## Частые вопросы

**Почему `shared_str`, а не `std::string`?**
Строки движка живут всю сессию и сравниваются постоянно (пути, имена секций INI, ключи). `shared_str` интернирует строки: одна физическая копия в `str_container`, сравнение — по 32-битному CRC и refcount, а не побайтовый `memcmp`. Это быстрее и экономит память. Подробности — [Строки](strings.md).

**Что такое LTX-архив?**
LTX-архив — папка, упакованная в один файл с каталогом (сжатие LZO/PPMd). Игровые данные (`gamedata/levels/...`, `settings.ltx`) лежат внутри таких архивов. Для кода движка это прозрачно: `FS.r_open("levels/test.ltx")` находит файл и внутри архива. Подробности — [Локализатор](locator.md) и [Сжатие](compression.md).

**Почему один аллокатор для всего движка?**
Все `new`/`delete` переопределены и уходят в `Memory.mem_alloc`. Это даёт единый учёт выделений (`mem_usage`, debug-имена), пулы для частых мелких объектов и возможность компактизировать память. `malloc` из CRT в движке не используется. Подробности — [Память](memory.md).

**Почему `Msg`, а не `printf`?**
`Msg` пишет одновременно в файл логирования и в callback (консоль/отладчик) и доступен **до** инициализации файловой системы — с него начинается отладка при старте. Подробности — [Логирование](logging.md).

**С чего начать читать код?**
С [Типов и математики](types-math.md), затем [Память](memory.md) и [Строки](strings.md) — на них опирается всё остальное. Порядок и причины — в разделе «Как читать xrCore» выше.

**Где в коде находится «старт» движка?**
В `xrCore::_initialize` (см. раздел «Инициализация» выше) — он последовательно поднимает память, лог, CPU и FS. Всё остальное в движке запускается поверх этого.

## Связанные страницы

- [Архитектура: Обзор](../../architecture/overview.md) — где `xrCore` в слоях.
- [Зависимости](../../architecture/dependencies.md) — полный граф.
