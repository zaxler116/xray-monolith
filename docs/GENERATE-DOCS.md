# GENERATE-DOCS — инструкция по генерации документации X-Ray Monolith

> Это живая инструкция. Она обновляется по мере продвижения итераций:
> план структуры (§3), порядок итераций (§5), список страниц (§7) — всё это
> корректируется по мере исследования движка. Не является финальным контрактом.

---

## 0. Параметры проекта

| Параметр     | Значение                                                                                      |
| ------------ | --------------------------------------------------------------------------------------------- |
| Движок       | X-Ray Monolith (fork X-Ray 1.6.0.8, S.T.A.L.K.E.R. Anomaly)                                   |
| Исходники    | `src/` (xrCore, xrEngine, Layers/xrRender*, xrGame, xrCDB, xrServerEntities, ...)             |
| Инструмент   | MkDocs Material                                                                               |
| Корень доков | `docs/` (в этом репозитории)                                                                  |
| Язык         | **Русский** (английский отложен, будет позже отдельным проходом)                              |
| Глубина      | **Полная**: каждый модуль — до последнего винтика. Не обзорная вики, а детальный разбор.      |
| Публикация   | Отдельный сайт на MkDocs (не в корне репозитория, а `docs/` → свой деплой)                    |
| Объём        | `.md`-файлов будет **много** — это осознанно, детализация требует разбивки на мелкие страницы |

---

## 1. Конфиг `docs/mkdocs.yml`

- `site_name: X-Ray Monolith Engine Documentation`
- Тема: **material** (search + mermaid + tabs + codehighlight)
- Плагины:
  - `search` — базовый поиск
  - `mkdocs-autorefs` — **критично**: авто-перекрёстные ссылки между страницами (решает «всё взаимосвязано»)
  - `mermaid2` — диаграммы (архитектура, frame-loop, sequence)
  - `tabs` — где уместно (варианты конфигурации, платформы)
- `locale: ru`, `docs_dir: docs`
- `strict: true` при сборке (ловит битые ссылки и страницы вне nav)
- `nav:` — ручная, полная. Генератор nav не используется.

---

## 2. Ключевое решение: как учесть взаимосвязи между модулями

Проблема: движок — плотный граф, модуль A использует классы из B, C, D.
Waterfall-разбор («сейчас модуль A, потом B») ломается: читая A, читатель
улетает в B, где B ещё не разобран.

**Решение — двухслойная модель.**

### Слой 1. «Карта движка» (строятся один раз, на итерации 0)

Страницы, которые задают **глобальный контекст** и уже содержат все связи.
Детальные страницы ссылаются на них, а не повторяют. Это и есть ответ на
«не прыгать по всему движку»: общий контекст один, прочитывается раз,
и применяется ко всем последующим страницам.

| Страница                             | Что даёт                                                                                                                                         | Как строить                                                             |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------- |
| `architecture/overview.md`           | Слои: xrCore → xrEngine → xrRender* → xrGame → Server. Mermaid-диаграмма зависимостей (кто кого `#include`'ит)                                   | Статический анализ: `grep -r '#include "xr'` по `src/` → граф → mermaid |
| `architecture/module-map.md`         | Таблица: каждая папка `src/*` ↔ ответственность ↔ ключевые классы ↔ зависящие модули                                                             | По `.vcxproj` + заголовкам                                              |
| `architecture/dependencies.md`       | **Явный граф связей**: модуль X → [A, B, C] + однолинейное «зачем». Ключевая страница-справочник                                                 | По include-графу + ручная корректировка                                 |
| `architecture/frame-loop.md`         | **Канонический порядок кадра**: `Device::tick` → `xrSheduler` → Lua-callback'ы → Render → Physics. «Скелет», к которому привязываются все модули | По `Engine.cpp` / `device.cpp`                                          |
| `architecture/object-model.md`       | Иерархия объектов: `xr_object` → `CGO`/`CGameObject` → `CActor`/`CWeapon`...; наследование, владение (smart_ptr)                                 | По `xr_object.h`, `GameObject.h`, `Actor.h`                             |
| `architecture/scripting-boundary.md` | Где C++ встречает Lua: `script_binder`, `ai_script_lua_extension`, все `*_script.cpp`. Полный список экспортируемых в Lua классов                | По luabind-обвязкам                                                     |

**Правило**: детальная страница модуля **не объясняет**, что такое `xr_object`
или как устроен кадр. Она **ссылается** на `architecture/object-model.md` /
`frame-loop.md`. Повторение глобального контекста запрещено.

### Слой 2. Детальные страницы (по итерациям)

Каждая итерация = **один модуль, целиком, до винтика**. Формат страницы модуля:

```
# Модуль: <имя>
## 1. Ответственность          (2 абзаца: что делает, чего НЕ делает)
## 2. Место в архитектуре     (ссылка на overview + dependencies)
## 3. Публичный API           (таблица: классы/функции + однолинейное назначение)
## 4. Внутреннее устройство   (по классам/функциям, до винтика)
## 5. Взаимодействие          (явный список: кто вызывает меня, кого вызываю я)
## 6. Потоки данных/управления(mermaid sequence/flow для ключевых сценариев)
## 7. Конфигурация            (console-команды, секции .ini, defines)
## 8. Известные ограничения/дебаг (флаги, debug-рендереры, логи)
```

**Пункт 5 — «взаимодействие» — и есть учёт связей.** Плюс `mkdocs-autorefs`
делает перекрёстные ссылки `[xrSheduler](../xr-engine/scheduler.md)`
кликабельными. Читатель не гадает, где объект.

### Механизм против waterfall: заглушки-контракты

На итерации 0 создаётся **каждая** детальная страница как заглушка:

- название + папка в `src/`
- 1–3 строки: что делает
- **публичный API** (таблица классов/ключевых функций) — извлечение, не разбор
- разделы-заглушки: `## 4. Внутреннее устройство — *будет разобран в итерации N*`
- раздел «Взаимодействие» заполняется сразу (список ссылок) — он статичен

Заглушка — это **не «недоделанная страница»**, а минимальный контракт.
Итерация N _дополняет_ заглушку (заполняет п. 4–8), не создаёт страницу с нуля.

**Итог**: nav с первого дня — рабочая карта. Детальные страницы растут
по итерациям, но граф ссылок существует сразу.

---

## 3. Структура `docs/docs/` (итерация 0)

> **Внимание**: структура — предварительная. Она будет меняться по мере
> исследования движка (§5). Это не контракт, это стартовая гипотеза.
> Разбивка `modules/game/*` на под-страницы может стать мельче или крупнее.

```
docs/docs/
├── index.md                          # перезаписать существующий
├── architecture/
│   ├── overview.md
│   ├── module-map.md
│   ├── dependencies.md               # ключевая: граф связей
│   ├── frame-loop.md
│   ├── object-model.md
│   └── scripting-boundary.md
├── getting-started/
│   ├── installation.md
│   ├── building.md
│   └── troubleshooting.md
├── modules/
│   ├── xr-core.md
│   ├── xr-engine/
│   │   ├── index.md
│   │   ├── device.md
│   │   ├── scheduler.md
│   │   ├── lua-binding.md
│   │   └── console.md
│   ├── renderer/
│   │   ├── index.md
│   │   ├── pipeline.md
│   │   ├── postprocessing.md
│   │   └── shader-bus.md             # мигрировать SHADER_BUS.md
│   ├── xr-particles.md
│   ├── xr-sound.md
│   ├── game/
│   │   ├── index.md
│   │   ├── objects.md                # Actor / GameObject / Entity
│   │   ├── alife.md                  # AI (крупнейший подмодуль)
│   │   ├── pathfinding.md
│   │   ├── physics.md
│   │   ├── weapons.md
│   │   ├── inventory.md
│   │   ├── zones.md
│   │   └── ui.md
│   ├── server.md
│   └── network.md
├── data/
│   ├── ltx-format.md
│   └── gamedata-layout.md
├── api/
│   └── lua-reference.md              # агрегирует всё, итерация 14
└── reference/
    ├── changelog.md
    └── patches.md
```

> **Примечание про объём**: `.md`-файлов будет **много**. Детализация
> до винтика не помещается в одну страницу на модуль. По ходу итераций
> страницы будут **разбиваться** на под-страницы (например, `modules/game/alife.md`
> распадётся на `alife/index.md`, `alife/simulator.md`, `alife/planner.md`,
> `alife/registry.md` и т.д.). Это нормально и осознанно. Nav обновляется
> по мере разбивки.

---

## 4. Порядок итераций (зависимостный)

> **Внимание**: порядок — стартовая гипотеза. Он может меняться по мере
> исследования. Критично не «порядок как таковой», а принцип: **фундамент → верхушка**,
> и каждая итерация **завершена** (модуль задокументирован целиком),
> прежде чем начинать следующую.

```
Итерация 0:  Каркас — заглушки всех страниц + architecture/* + getting-started/* + shader-bus
Итерация 1:  xrCore         — математика, FS, memory, log, string, compression
Итерация 2:  xrEngine (ядро) — device, scheduler, input, console, Lua-binding
Итерация 3:  xrRender/R4    — рендерер: pipeline, constants, postproc, shader bus
Итерация 4:  xrParticles + xrSound
Итерация 5:  xrGame: Actor / GameObject / Entity — объект-модель
Итерация 6:  xrGame: ALIFE (AI) — крупнейший подмодуль
Итерация 7:  xrGame: Physics (PH)
Итерация 8:  xrGame: Weapons / Inventory / Ammo
Итерация 9:  xrGame: Zones / Effects / Environment
Итерация 10: xrGame: UI
Итерация 11: xrCDB + xrServerEntities + xrNetServer — сервер
Итерация 12: Сеть (Level_network, xrServer_*, anticheat)
Итерация 13: Data formats (LTX / DXML, gamedata layout)
Итерация 14: Lua API reference — агрегирует всё
Итерация 15: Миграция README.md (changelog / patches / troubleshooting)
```

**Почему это не waterfall**: в итерации N создаётся/дополняется страница
модуля N, но её пункт 5 («взаимодействие») ссылается на **заглушки**
будущих страниц (N+1..N+k). Заглушка содержит публичный API + раздел
«взаимодействие» — статичные части. Так читатель итерации 5 (Actor)
может по ссылке дойти до AI-страницы и получить хотя бы API,
а не «страница не существует».

---

## 5. Что будет изменяться по мере исследования

- **Структура §3**: страницы будут разбиваться/объединяться. `modules/game/*`
  почти наверняка распадется на под-каталоги (ALIFE, Physics, Weapons —
  все крупные). `xr-engine/` тоже (device/scheduler/lua — по отдельности).
- **Порядок итераций §4**: если при разборе xrCore выяснится, что какой-то
  модуль логически ближе к xrEngine — он двигается.
- **Список страниц в §7**: навигация в `mkdocs.yml` правится по ходу.
- **`architecture/dependencies.md`**: уточняется после каждой итерации
  (детальный разбор модуля X даёт более точный граф, чем include-анализ).

**Что НЕ меняется**:

- Двухслойная модель (§2) — глобальный контекст + детальные страницы.
- Формат детальной страницы (§2, Слой 2) — 8 разделов.
- Принцип: каждая итерация завершена до начала следующей.
- Заглушки-контракты — существуют с итерации 0.

---

## 6. Инструменты автоматизации

| Инструмент                     | Назначение                                                                 | Статус                                                                     |
| ------------------------------ | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Include-граф (Python, разовый) | Парсит `#include "xr*.h"` по `src/` → JSON + mermaid для `dependencies.md` | TODO: написать на итерации 0, положить в `docs/tools/gen_include_graph.py` |
| `mkdocs build --strict`        | Ловит битые ссылки и страницы вне nav                                      | Использовать на каждой итерации                                            |
| Nav-генератор                  | **Не используется** — nav ~50 строк вручную в `mkdocs.yml`                 | —                                                                          |

---

## 7. Итерация 0 — план работ

1. Записать `docs/mkdocs.yml` (тема, плагины, locale, полная nav с заглушками).
2. Создать **все** каталоги и страницы-заглушки по структуре §3.
3. Заполнить `architecture/*` (6 страниц) — «карта», фундамент.
4. Заполнить `getting-started/*` из `README.md` (installation / building / troubleshooting — уже есть в README).
5. Перенести `SHADER_BUS.md` → `modules/renderer/shader-bus.md` (готовый текст).
6. Заполнить `index.md`.
7. Написать `docs/tools/gen_include_graph.py`, прогнать, результат в `architecture/dependencies.md`.
8. `mkdocs build --strict` — проверить валидность каркаса.

**Объём итерации 0**: ~25 страниц.

- `architecture/*`: 6 страниц, содержательные (100–300 строк каждая).
- `getting-started/*`: 3 страницы (миграция из README).
- Заглушки: ~15 страниц, короткие (30–60 строк каждая).
- `index.md`, `shader-bus.md`: 2 страницы.

---

## 8. Конвенции

- **Язык**: русский. Технические термины — как в коде (не транслитерируем).
- **Ссылки на исходники**: относительные от репозитория, `src/xrEngine/Engine.h`.
- **Код**: блоки с подсветкой (`cpp`, `lua`, `hlsl`, `ini`).
- **Диаграммы**: mermaid, тип `graph TD` для архитектуры, `sequenceDiagram`
  для потоков, `classDiagram` для object-model.
- **Названия файлов**: kebab-case (`shader-bus.md`), не snake_case.
- **Имена классов/функций**: как в коде, без адаптации (`xrSheduler`, `CGO`).
- **Запрещено**:
  - повторять глобальный контекст в детальных страницах (ссылаться на `architecture/*`)
  - «будет разобран позже» без указания итерации
  - страницы без раздела «Взаимодействие» (п. 5)

---

## 9. Логи итераций

> Обновляется после каждой итерации: что сделано, что изменилось в структуре,
> что обнаружено нового в движке, что перенесено/перенаправлено.

- **[Итерация 0]**: каркас + architecture (6) + getting-started (3) + shader-bus + index.
- **[Итерация 1]** (завершена): **xrCore** — 10 страниц:
  - `types-math.md` — типы, векторы, матрицы, кватернионы, FPU/CPU, треды.
  - `memory.md` — `xrMemory`, `operator new/delete`, `MEMPOOL`, PSO, debug-учёт.
  - `strings.md` — `shared_str` (interned, CRC), `xr_string`, `smem_container`, конкатенации.
  - `logging.md` — `Msg`/`Log`, `LogFile`, `LogCallback`, `FormatString`.
  - `filesystem.md` — `IWriter`/`IReader`/`IReaderBase`, `CMemoryWriter`, `CVirtualFileRW`, `CStreamReader`, `EFS_Utils`.
  - `inifile.md` — `CInifile`, DLTX-override, include, кэш, типизированное чтение/запись.
  - `compression.md` — LZO (1x/9x), LZHUF, PPMd (mt, trained), SubAlloc.
  - `locator.md` — `CLocatorAPI` (виртуальная ФС), auth, cache, rescan.
  - `system.md` — таймеры (`CTimer`), синхронизация (`xrCriticalSection`, `Lock`, `xrSRWLock`), CPU, `xrDebug`, `NET_Packet`, `CLASS_ID`, `intrusive_ptr`.
  - `index.md` — обзор xrCore.
  - Все заглушки для будущих итераций созданы (nav полный, `mkdocs build --strict` проходит).
- **[Итерация 2, порцион 1]** (завершён): **xrEngine — Ядро и API** — 2 страницы + обновлён `index.md`:
  - `engine.md` — `CEngine`, `PSGP` (xrCPU_Pipe), `WinMain`/`WinMain_impl`/`Startup`, `CApplication` (`pApp`, `KERNEL:*`, `LoadBegin/End`), `pure.h`/`CRegistrator<T>`, `pure_relcase`, `defines.h` (флаги, пути), `std_classes.h` (CLSID), `PROTECT_API`/`NO_SINGLE`, `mp_logging.h`.
  - `api.md` — `CEngineAPI` (рендерер R1..R4, `xrGame`, vTune, фабрика `NEW_INSTANCE`/`DEL_INSTANCE`, `DLL_Pure`), `CEventAPI`/`CEvent` (реф-счётчик, `Signal`/`Defer`, `OnFrame`, `Peek`), поток `KERNEL:start`.
  - `index.md` — карта связей xrEngine + таблица статусов 12 порционов.
  - `mkdocs.yml` — секция xrEngine расширена до 29 записей (все страницы-цели порционов 1–12).
  - **Обнаружено**: `PSGP` — это `xrDispatchTable` (скиннинг `skin1W..4W` + `PLC_calc3`), **не** general-purpose API. `CApplication::OnFrame` — первый участник `Device.seqFrame` (приоритет `REG_PRIORITY_HIGH + 1000`), поэтому `Engine.Event.OnFrame` (defer-события) выполняется до остальных. Рендерер выбирается **компиляцией** (`STATIC_RENDERER_R?`), а не рантайм-поиском.
- **[Итерация 2, порцион 2]** (завершён): **xrEngine — Цикл кадра и планировщик** — 2 страницы:
  - `frame-loop.md` — `Device.Run` (3 потока, `seqAppStart`/`seqAppEnd`), `message_loop` (кадр = «пробел» Win-очереди), `on_idle` (17 шагов: загрузка, `FrameMove`, матрицы, синхронизация `mt_csEnter/mt_csLeave`, `seqRender`, fallback `seqFrameMT` на main), `FrameMove` (два режима тайминга, EMA-сглаживание, `ShaderBus::frame_latch`, `seqFrame.Process`), `mt_Thread` (`seqParallel` + `seqFrameMT` + параллельный Lua-GC во время GPU-рендера), `mt_DiscordThread`, `mt_FreezeThread`, `Begin/End`, все registrator'ы `CRenderDevice`, `AddSeqFrame(f, mt)`.
  - `scheduler.md` — `ISheduled` (битовое поле `t_min/t_max/b_RT/b_locked`, `shedule_Scale/Needed/Update/Name`), `CSheduler`: контейнеры (`ItemsRT`, `Items` heap, `ItemsProcessed`, `Registration`), отложенная регистрация с парной отменой, ленивое удаление (NULL в heap), `EnsureOrder`, `Update()` (бюджет в QPC-циклах, RT-проход, `ProcessStep`, EMA `psShedulerCurrent`), `ProcessStep` (формула интервала `dwMin+floor((dwMax-dwMin)*scale)`, контроль бюджета каждый 3-й объект, `break` при превышении), глобалы `psShedulerCurrent/Target/Reaction`, `g_bSheduleInProgress`, DEBUG-проверки.
  - `index.md` — статус порциона 2 → ✅.
  - **Обнаружено**: (1) «кадр» привязан к **отсутствию** Windows-сообщений, а не к сна-циклу; (2) `seqFrameMT` исполняется на **вторичном потоке** параллельно GPU-рендеру, и есть **fallback на main-поток**, если поток не отметился (`mt_Thread_marker`); (3) `CSheduler` — адаптивный временной бюджет (`psShedulerCurrent` 3–66 мс), **не** фиксированный тик; (4) `ProcessStep` переносит недоделанные задачи на следующий кадр через `break`, а `psShedulerTarget` растёт на `reaction*3` при превышении бюджета; (5) `Register/Unregister` — deferred, применяются в 2 точках `Update()`; (6) `EnsureOrder` работает только для realtime-объектов. Не покрыто: `Device_wndproc` (порцион 3), механика `Begin/End` (порционы 3–4), `mt_FreezeThread` подробно (порцион 3).
- **[Итерация 2, порцион 3]** (завершён): **xrEngine — Устройство** — 2 страницы:
  - `device.md` — `CRenderDevice`/`CRenderDeviceData`/`CRenderDeviceBase` (наследование), `CRenderDeviceData` (все поля: размеры, флаги, тайм-поля, матрицы, регистраторы), `CRenderDevice` (дополнения: `dwPrecacheTotal`, `m_pRender`, `seqFrameMT`, `seqDeviceReset`, `seqParallel`, `mInv*`, `m_SecondViewport`, `mt_cs*`, `m_imgui`, `LuaGC*`), `Initialize` (окно, класс `_XRAY_1.5`, `m_imgui`), `Create` (GPU, `SetupGPU`, `m_pRender->Create`, `_Create`, `PreCache(0)`), `Destroy`, `Reset` (reshade, `bNeed_re_create_env`, `PreCache(20)`), `ChangeOutputMonitor` (живой перенос), `Pause`/`Paused` (`g_pauseMngr`, `snd_emitters_`), `Begin`/`End` (состояния GPU: `dsOK/dsLost/dsNeedReset`, `precache_light`, `updateGamma`, `ResourcesDestroyNecessaryTextures`, SASH), `PreCache` (вращение камеры, `precache_light`), `FrameMove` (два режима тайминга, EMA, `ShaderBus::frame_latch`, `seqFrame`), `on_idle` (матрицы, `SetCacheXform`, трава, ветер, `mInvFullTransform`), `CSecondVPParams` (SVP: `isActive`, `frameDelay`, `isCamReady`, `IsSVPFrame`), `CLoadScreenRenderer` (`pureRender`, `start/stop/OnRender`), `mt_FreezeThread` (`FreezeTimer`, 5/25 с, `FlushLog`), мониторы (`g_StartupMonitor`, `InitMonitor`, `GetMonitorResolution/Position/Refresh`), `hud_to_world`/`world_to_hud` (матричные преобразования), `GetPerceivedDist` (FOV), `CalcSSADynamic`, `SetNearer`, глобалы (`Device`, `g_bRendering`, `g_bLoaded`, `precache_light`, `psLua_ParallelGC*`, `g_svDedicateServerUpdateReate`, `g_loading_events`, `app_inactive_time*`, `bShowPauseString`, `FreezeTimer`, `refresh_rate`, `mt_Thread_marker`).
  - `device-window.md` — `WndProc` (статический обработчик), `on_message` (все `WM_*`: `SYSKEYDOWN`, `ACTIVATE`, `SETCURSOR`, `SYSCOMMAND`, `CLOSE` → `KERNEL:disconnect`/`KERNEL:quit`, `HOTKEY`, `SYSCHAR`, `CHAR` → `imgui().InputChar`, `INPUTLANGCHANGE`, `DISPLAYCHANGE` → `g_monitor_list_dirty`), `OnWM_Activate` (`rsAlwaysActive`, `b_is_Active`, `seqAppActivate/Deactivate`, курсор, `app_inactive_time`), `DumpFlags` (`rsClearBB/rsVSync/rsWireframe`), `overdrawBegin/End` (заглушки `VERIFY(0)`), `_d3d_extensions.h` (`Flight` ≈ `D3DLIGHT9`, `Fmaterial` ≈ `D3DMATERIAL9`, `VDeclarator` ≈ `D3DVERTEXELEMENT9`, устаревшие), `xrHemisphere` (3 качества: 26/91/196 вершин, `Build` — инверсия + нормализация + равномерная энергия, `Indices` — quality 3 закомментирован).
  - `index.md` — статус порциона 3 → ✅.
  - **Обнаружено**: (1) `WM_CLOSE` шлёт **два** defer-события: `KERNEL:disconnect` + `KERNEL:quit`; (2) `rsAlwaysActive` при `g_screenmode == 2` **не** действует; (3) `app_inactive_time` не накапливается при `rsAlwaysActive` (early return в `OnWM_Activate`); (4) `Begin` при `dsLost` — `Sleep(33)` + skip кадра; при `dsNeedReset` — `Reset()` прямо в `Begin`; (5) `precache_light` создаётся только если `g_pGameLevel` существует и `g_loading_events` пуст; (6) `mt_FreezeThread`: порог 5 с (25 с при загрузке), после срабатывания — `FlushLog` + пауза 5 с; (7) `xrHemisphere` quality 3 — индексы закомментированы; (8) `Flight`/`Fmaterial`/`VDeclarator` — устаревшие D3D9-структуры, современный бэкенд их не использует. Не покрыто: рендер-бэкенд `IRenderDeviceRender` (итерация 3), `CStats` (порцион 12), `g_pauseMngr` (порцион 12/6), `pInput` (порцион 12), `imgui` (порцион 11).
- **[Итерация 2, порцион 4]** (завершён): **xrEngine — Рендер-слой** — 3 страницы:
  - `render.md` — `IRender_interface` (полный API: feature level, loading, main, wallmarks, ROS, lighting, models, sun, screenshot, particles, render mode, occlusion, calculate/render), `IRender_Light` (LT enum, volumetric, hud_mode, playerlight), `IRender_Glow`, `IRender_ObjectSpecific` (ROS: TRACE_LIGHTS/SUN/HEMI), `IRender_Target` (post-processing: blur, gray, duality, noise, cm_*), `IRender_Portal`/`IRender_Sector` (маркерные), `CPS_Instance` (жизненный цикл: `ps_active`, `ps_destroy`, `PSI_destroy`, `PSI_internal_delete`, `shedule_Update`), `ShadersExternalData` (Lua-параметры: `m_script_params`, `hud_params`, `hud_fov_params`, `m_blender_mode`), `Shader_xrLC` + `Shader_xrLC_LIB` (бинарная библиотека, `Load/Save/GetID/Get/Append/Remove`), `vis_data` (bounding volume + метки кадров), `MbHelpers` (`mbhMulti2Wide` UTF-8→UTF-16, `MB_DUMB_CONVERSION`, line-breaking helpers), `imf_Process` (2D фильтрованное ресэмплирование: 7 фильтров, separable, anti-aliasing при уменьшении), `PAPI` (psystem.h: `pVector`, `Particle` 72B, `PDomainEnum`, `PActionEnum`, `IParticleManager`, `ParticleManager()`) .
  - `renderable.md` — `IRenderable` (структура `renderable`: `xform`, `visual`, `pROS`, `pROS_Allowed`), конструктор (`STYPE_RENDERABLE`), деструктор (`VERIFY(!g_bRendering)`, `model_Delete`, `ros_destroy`), `renderable_ROS()` (ленивое создание), `renderable_Render/ShadowGenerate/ShadowReceive` (чистые виртуальные), `GetHotness/GetTransparency/GetGlowing` (HeatVision/SilencerOverheat).
  - `fmesh.md` — `Fmesh.h` (формат OGF: `MT` enum, `OGF_Chuncks` enum, `ogf_header`, `ogf_desc` (Load/Save), `ogf_bbox`/`ogf_bsphere`, `FSlideWindow`/`FSlideWindowItem`, `OGF_SkeletonVertType`, `xrOGF_FormatVersion=4`, `xrOGF_SMParamsVersion=4`), `CFM_DynamicMesh` (наследование `CCF_Skeleton`, `_RayQuery` двухэтапный: грубый + `PickBone` уточнение), `SEnumVerticesCallback`.
  - `index.md` — статус порциона 4 → ✅.
  - **Обнаружено**: (1) `::Render` — глобальный указатель `IRender_interface*`, устанавливается `xrRender`, используется в деструкторах `IRender_Light`/`IRender_Glow`; (2) `CPS_Instance` — `m_iLifeTime = int_max` по умолчанию, `m_bAutoRemove = TRUE`, `pROS_Allowed = FALSE`; (3) `mbhMulti2Wide` — `MB_DUMB_CONVERSION` всегда включён (невалидный UTF-8 → as-is); (4) `imf_Process` — separable (2 прохода: горизонтальный + вертикальный), при уменьшении — нормализация весов (`/fscale`); (5) `CFM_DynamicMesh::_RayQuery` — двухэтапный: `CCF_Skeleton` (грубый, bounding volumes) + `IKinematics::PickBone` (точный, ray-triangle), ложные хиты отфильтровываются; (6) `Shader_xrLC_LIB::Load` — `FATAL` если файл не найден; (7) `PAPI::Particle` — 72 байта, plain data (можно сериализовать); (8) `vis_data` — `#pragma pack(push,4)`, размер зависит от `Fsphere`/`Fbox`. Не покрыто: рендер-бэкенд `IRenderDeviceRender` (итерация 3), `RenderVisual`/`Kinematics` (итерация 3), `CStats` (порцион 12), `g_pauseMngr` (порцион 12/6), `pInput` (порцион 12), `imgui` (порцион 11).
- **[Итерация 2, порцион 5]** (завершён): **xrEngine — Коллизии и физика (интерфейсы)** — 1 страница:
  - `collide-physics.md` — `ICollidable` (mixin: `collidable.model` + `STYPE_COLLIDEABLE`), `ICollisionForm` (база: `owner`, `bv_box`/`bv_sphere` local, виртуальный `_RayQuery`; `_BoxQuery` закомментирован), `CCF_Skeleton` (по костям `SBoneShape`: box/sphere/cylinder; `BuildTopLevel` + `BuildState` ленивое rebuild по кадру и `vis_mask`; `_RayQuery` двухфазный: world-bv-sphere → пер-кость тесты `RAYvsOBB/SPHERE/CYLINDER`; невалидная bone-matrix → `Msg("! ERROR: invalid bone xform")` + `elem_id = u16(-1)`), `CCF_EventBox` (1×1×1 бокс, 6 плоскостей, `Contact(O)`; `_RayQuery` — заглушка `FALSE`), `CCF_Shape` (произвольные шары/боксы: `add_sphere`/`add_box`, `ComputeBounds`, `_RayQuery` в локальных координатах, `Contact(O)` по 6 плоскостям бокса / пересечению сфер), `clQueryCollision` + `clQueryTri` (контейнер результатов + флаги `clGET_TRIS/BOXES/SPHERES`, `clQUERY_ONLYFIRST/TOPLEVEL/STATIC/DYNAMIC`, `clCOARSE`), `IObjectPhysicsCollision` (фасад: `physics_shell()`, `physics_character()` deprecated), `IPhysicsShell`/`IPhysicsElement`/`IPhysicsGeometry` (read-only фасады на `CPhysicsShell`/`CPHElement`/`CODEGeom` — **структурное совпадение**, не полиморфизм), `hdrCFORM` (в `xrLevel.h`, `CFORM_CURRENT_VERSION=4`), `CObjectSpace` (xrCDB: `RayTest`/`RayPick`/`RayQuery`/`BoxQuery`/`GetNearest`, `level.cform`), `CPhysicObject::create_collision_model` (`collide.mesh` → `CCF_DynamicMesh`, иначе `CCF_Skeleton`).
  - `index.md` — статус порциона 5 → ✅.
  - **Обнаружено**: (1) `IPhysicsShell`/`IPhysicsElement`/`IPhysicsGeometry` — **не наследуются** `xrPhysics`: сигнатуры совпадают, но полиморфизма нет; на границе `xrRender ↔ xrPhysics` идёт `static_cast`/`reinterpret_cast`; (2) `IObjectPhysicsCollision` реализован **только** в xrGame (`CPhysicsShellHolder`), в xrEngine — только forward-декларации; (3) `CObject::BoundingBox()` — из `renderable.visual->getVisData().box`, **не** из `CFORM()->getBBox()` (два разных bounding volume); (4) `CCF_Skeleton` rebuild только по кадру и по смене `vis_mask` — внутри кадра без смены mask форма устаревает; (5) `CCF_Shape::_RayQuery` трансформирует луч в **локальные** координаты владельца, `CCF_Skeleton` — в world; (6) `CPhysicsShellHolder::physics_shell()` const-версия возвращает shell **анимационной** коллизии (ленивое создание через `CCharacterPhysicsSupport`), если `m_pPhysicsShell == NULL`; non-const — только `m_pPhysicsShell`; (7) `CCF_EventBox` — **нигде** в xrGame не создаётся напрямую (в коде только `CCF_Shape` и `CCF_Skeleton`); (8) `hdrCFORM` — `#pragma pack(push,8)`, размер зависит от `u32`/`Fbox`. Не покрыто: `xrPhysics` целиком (итерация 7 — Physics), `CObjectSpace` подробно (итерация 7), `IKinematics` (итерация 3).
- **[Итерация 2, порцион 6]** (завершён): **xrEngine — Объекты и уровень** — 4 страницы:
  - `xr-object.md` — `CObject` (layout: `Props` union из битов, 3 имени, `renderable.xform`), crow-режим (`MakeMeCrow` CAS по `dwFrame_AsCrow`, `CROW_RADIUS=30`/`CROW_RADIUS2=60`), `spatial_update` (PositionStack 4 записи, `eps_P/eps_R`), `H_SetParent` (spatial_unregister при родителе), `net_Spawn`/`net_Destroy`, `CObjectList` (`objects_active/sleeping`, `m_crows[2]` по потокам, `Update` — crow + destroy_queue, `net_Export/Import` по `map_NETID[0xffff]`, `register_object_to_destroy` — рекурсия по детям), `CObjectAnimator` (`.anm/.anms`, `Play/Update/Stop`, `DrawPath` только `_EDITOR`).
  - `level.md` — `IGame_Level` (конструктор: `g_pGameLevel`, `m_pCameras`; `Load` — 14 шагов: `Level_Set`, `level.ltx`, `hdrLEVEL` version check, `ObjectSpace.Load`, `Sound->set_geometry_occ`, `g_hud`, `Render->level_Load`, `Environment().mods_load`, `Objects.Load`, `bReady`, `seqRender/seqFrame.Add`; `OnFrame` — `Objects.Update` + `g_hud->OnFrame` + `Sounds_Random` 10–20 с; `OnRender` — `Render->Calculate/Render` или `Sleep` для dedicated; `SetEntity`/`SetViewEntity` (`On_LostEntity/On_SetEntity`); `SoundEvent_Register` (deferred: `q_box STYPE_REACTTOSOUND`, `Power = (1-d/max)*vol*occlusion`, `snd_Events`), `SoundEvent_Dispatch` (в `CLevel::OnFrame`, не в базе), `SoundEvent_OnDestDestroy`; `LL_CheckTextures` (отчёт, не ошибка); `CServerInfo` (max 15, молча игнорирует переполнение); `xrLevel.h` (`fsL_Chunks`, `hdrLEVEL/CFORM/NODES`, `NodePosition` 5B, `NodeCompressed` 12B, `NodeCompressed6` 11B `AI_COMPILER`, `XRCL_PRODUCTION_VERSION=14`, `CFORM_CURRENT_VERSION=4`).
  - `object-pool.md` — `IGame_ObjectPool` (фабрика: `create` = `r_clsid` + `NEW_INSTANCE` + `Load`; `destroy` = `xr_delete`; **не пул** — старая реализация закомментирована L66–140; `prefetch` — из `prefetch_objects_<game_type>`, объекты **не** в `CObjectList`/`g_SpatialSpace` — только для кэширования ресурсов; `clear` при `OnGameEnd`).
  - `persistent.md` — `IGame_Persistent` (конструктор: 5 регистраторов `seqAppStart/End/Activate/Deactivate/Frame` приоритет `HIGH+1`, `PerlinNoise1D` seed случайный, `pEnvironment`, `ShadersExternalData`; `params` union 4×`string256` + `EGameIDs`, `parse_cmd_line` по `/`; `OnAppStart/End` (`Environment.load/unload`, `m_textures_prefetch_config`); `PreStart/Start` (смена типа → `OnGameEnd`/`OnGameStart` + `DEL_INSTANCE(g_hud)`); `OnGameStart` (`LoadTitle` + `Prefetch` если нет `-noprefetch`); `OnFrame` (`Environment().OnFrame`, `ps_needtoplay` → `Play`, `ps_destroy` → `PSI_internal_delete` с `Locked` break); `destroy_particles` (`all` vs `destroy_on_game_load`); `GrassBenders*` (16 слотов, `grass_shader_data`, `ps_ssfx_grass_interactive`); `CGamePersistent` (xrGame: ambient, DoF `m_dof[4]`, intro, `OnEvent`, `OnThunderboltSound/OnRainSound` → Lua); `IMainMenu` (4 чистых); `g_pGamePersistent` (в `x_ray.cpp`, не в конструкторе); `IsMainMenuActive()` inline).
  - `index.md` — строки порциона 6 обновлены + статус → ✅.
  - **Обнаружено**: (1) `CObjectList::map_NETID[0xffff]` — статический массив (не `xr_map`), `u16(-1)=0xffff` = «нет»; `net_Find(0xffff)` → `NULL`; (2) `CObjectList::Update` — при `destroy_queue` не пуст `net_Relcase` вызывается для **всех** активных+спящих (O(n²)); (3) `CObject::MakeMeCrow` — CAS по `dwFrame_AsCrow` (один раз за кадр); `m_crows[2]` — по потокам (owner vs secondary); (4) `IGame_Level::SetEntity` — устанавливает **оба** `pCurrentEntity` и `pCurrentViewEntity` (асимметрия с `SetViewEntity`); (5) `SoundEvent_Dispatch` — вызывается в `CLevel::OnFrame` (xrGame), **не** в `IGame_Level::OnFrame` (база не вызывает); (6) `IGame_ObjectPool` — **не пул**: старая реализация (закомментирована) возвращала объекты в `map_POOL`; сейчас `create` = всегда новый, `destroy` = `xr_delete`; (7) `IGame_Persistent::OnFrame` — приоритет `HIGH+1` (**до** `CApplication` `HIGH+1000`); (8) `IGame_Persistent` не `IEventReceiver` в базе — события в `CGamePersistent`; (9) `xrLevel.h` — `NodePosition`/`NodeCompressed` — для `CLevelGraph` (AI-навигация), **не** для рендера; `CLevelGraph` — в xrGame; (10) `fsL_Chunks` — пропуск `5` (между `PORTALS=4` и `LIGHT_DYNAMIC=6`) — исторический. Не покрыто: `CLevel` подробно (xrGame, итерация 3+), `CLevelGraph` (xrGame), `CEnvironment` подробно (порцион 8), `CPS_Instance` подробно (порцион 4 — `render.md`), `CGamePersistent` подробно (xrGame).
- **[Итерация 2, порцион 7]** (завершён): **xrEngine — Скелет и анимация** — 1 страница:
  - `skeleton-motion.md` — `bone.h/.cpp` (`CBoneInstance` (5 `Fmatrix` + callback + `param[4]`), `CBone` (editor: rest/mot-состояния, `IK_data`, `shape`, `mass`, `engine/editor_lo/hi_limit`), `CBoneData` (shared: `bind_transform`, `m2b_transform` (`CalculateM2B` рекурсивно), `shape`, `IK_data`, `child_faces`), `IBoneData` (read-only интерфейс, реализуют `CBone`+`CBoneData`), `SBoneShape` (box/sphere/cylinder + `sfNoPickable/sfNoPhysics/...`), `SJointIKData` (`EJointType`, `limits[3]`, `flBreakable`, `break_force/torque`, `Export/Import` с ODE-инверсией), `vertBoned1W..4W` (skin-вершины 60/64/70/76B, `#pragma pack(2)`)), `SkeletonMotionDefs.h` (`MAX_PARTS=4`, `SAMPLE_FPS=30`, `KEY_Quant=32767`, `fQuantizerRangeExt=1.5`), `SkeletonMotions.h/.cpp` (`CKey/CKeyQR/CKeyQT8/CKeyQT16` (`#pragma pack(2)`), `CMotion` (8-bit flags + 24-bit count, `ref_smem<CKeyQR/QT8/QT16>` — CRC-дедуп, `_initT/_sizeT`), `CMotionDef` (speed/power/accrue/falloff квантованные 0..65535, `Dequantize V/655.35`, `Accrue/Falloff ×1.5`, `falloff>=accrue`→`accrue-1`, `marks` `vers>=4`), `motion_marks` (intervals, `pick_mark`/`is_mark_between`/`time_to_next_mark`), `CPartDef`/`CPartition` (≤4, `load` из `<model>.ltx`), `motions_value` (`m_motion_map`/`m_cycle`/`m_fx`/`m_partition`/`m_motions`/`m_mdefs`/`m_dwReference`, `load` — `OGF_S_SMPARAMS`+`OGF_S_MOTIONS`, `VERIFY(dwCNT<0x3FFF)`), `shared_motions` (ref-counted, **не thread-safe**), `motions_container`/`g_pMotionsContainer` (глобал, `dock`/`clean`/`dump`)), `motion.h/.cpp` (`EChannelType` 6 каналов XYZ+HPB, `ESMFlags` 8 флагов, `CCustomMotion` (frame start/end/fps, `Save/Load`), `COMotion` (6 `CEnvelope*`, `EOBJ_OMOTION=0x1100`, версии 0x0003/0x0004/0x0005, `_Evaluate`), `CSMotion` (**только `_EDITOR`/`_MAX_EXPORT`/`_MAYA_EXPORT`**, `BoneMotionVec`, `EOBJ_SMOTION=0x1200`, версии 0x0004..0x0007, `SortBonesBySkeleton`/`WorldRotate`/`Optimize`), `SAnimParams` (`t_current/tmp/min_t/max_t`, `Update` — `bWrapped` + loop `t-=k*len`, `Play/Stop/Pause`)), `envelope.h/.cpp` (`SHAPE_TCB/HERM/BEZI/LINE/STEP/BEZ2`, `BEH_RESET/CONSTANT/REPEAT/OSCILLATE/OFFSET/LINEAR`, `st_Key` (`#pragma pack(1)`, `Save` stepped без параметров, `Load_1` shape u32&0xff, `Load_2` shape u8 + `float_q16`), `CEnvelope` (`keys: KeyVec<st_Key*>`, `behavior[2]`, `Evaluate` → `extern evalEnvelope` (`interp.cpp`, порцион 8), `InsertKey` SHAPE_TCB+BEH_CONSTANT, `Optimize` — все равные → front+back, `SaveA/LoadA` — LightWave ASCII)), `ObjectAnimator.h/.cpp` (`.anm` 1 / `.anms` N, `$level$`→`$game_anims$`, `Play` — `lower_bound` по name или `front()`, `Update` — `_Evaluate`+`SAnimParams::Update`+`m_XFORM`, `DrawPath` только `_EDITOR`), `cf_dynamic_mesh` (`CCF_DynamicMesh` — `CCF_Skeleton` + `_RayQuery` двухэтапный, см. [Fmesh](fmesh.md)/[Коллизии](collide-physics.md)).
  - `index.md` — строка `skeleton-motion.md` обновлена + статус порциона 7 → ✅.
  - **Обнаружено**: (1) `g_pMotionsContainer` — аллоцируется **`CModelPool` (xrRender)** в конструкторе, **не** `xrEngine`; в `xrEngine` — только `extern` + определение; (2) `CBone` (editor) наследует `CBoneInstance` (имеет `mTransform`+callback+`param`), `CBoneData` (shared, рантайм) — **нет**; `CBone` — для редактора/экспорта, `CBoneData` — для рантайма (`CKinematics::bones`); (3) `CBone::get_obb()` — `static const Fobb dummy = Fobb().identity()` (заглушка), `CBoneData::get_obb()` — реальный `obb`; (4) `CBone::engine_lo/hi_limit` — `-IK_data.limits[k].limit.y/.x` (инвертировано), `editor_lo/hi_limit` — прямые; `SJointIKData::Export` тоже инвертирует (комментарий: ODE vs X-Ray); `Import` — сырьё (без инверсии); (5) `CSMotion` — **только** `_EDITOR`/`_MAX_EXPORT`/`_MAYA_EXPORT`, в рантайме **не компилируется**; рантайм-анимация — `.omf` (`CMotion`) и `.anm/.anms` (`COMotion`); (6) `CMotion`-ключи — `ref_smem` (CRC-дедуп), `mem_usage()` делит на `ref_count()`; (7) `shared_motions` — ref-counted, **не thread-safe** (`m_dwReference++` без атома); `motions_container::clean` — без локов; (8) `CObjectAnimator::Play(NULL)` — `m_Motions.front()` (первый по алфавиту, `std::sort` по `name`); (9) `CEnvelope::Evaluate` — `extern float evalEnvelope` в `interp.cpp` (порцион 8); (10) `MotionID` — 2 bit slot + 14 bit index (`VERIFY(dwCNT < 0x3FFF)`); `SAMPLE_FPS=30` — частота квантованных движений. Не покрыто: `CKinematics`/`CKinematicsAnimated`/`CSkeletonX` (xrRender, итерация 3 — Skeleton: `PlayCycle`/`PlayFX`/blending `CBlend`/де-квантизация/skinning `skin1W..4W`), `interp.cpp`/`evalEnvelope` (порцион 8 — [Окружение](environment.md)), `CModelPool` (xrRender, итерация 3).
