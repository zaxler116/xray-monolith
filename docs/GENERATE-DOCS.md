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
