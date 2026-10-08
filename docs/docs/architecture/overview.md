# Обзор архитектуры

X-Ray Monolith — клиент-серверный игровой движок. Код разбит на **DLL-модули** (`src/*`), каждый со своей ответственностью и чёткой границей `#include`. Зависимости строго **снизу вверх**: нижние слои не знают про верхние.

## Слои

```mermaid
graph TD
    subgraph "Фундамент (без зависимостей внутри движка)"
        CORE[xrCore<br/>типы, математика, память,<br/>строки, лог, FS, INI, сжатие]
    end

    subgraph "Ядро движка"
        ENGINE[xrEngine<br/>device, sheduler, input,<br/>console, Lua-биндинг, эффекты]
        CPU[xrCPU_Pipe<br/>оптимизации под CPU<br/>(SIMD-dispatch)]
    end

    subgraph "Рендер (Layers/)"
        API[xrAPI<br/>C-интерфейс]
        RND[xrRender<br/>бэкенд-агностичный<br/>pipeline, константы]
        D9[xrRenderDX9 / D3D9]
        D10[xrRenderDX10 / D3D10]
        R1[xrRenderPC_R1..R4<br/>конкретные бэкенды<br/>R4 = D3D9 основной]
    end

    subgraph "Игра (клиент)"
        GAME[xrGame<br/>объекты, AI/ALIFE, физика PH,<br/>оружие, UI, MP-клиент]
        PART[xrParticles]
        SND[xrSound]
        PHY[xrPhysics<br/>обвязка PhysX]
        XML[xrXMLParser]
    end

    subgraph "Сервер"
        CDB[xrCDB<br/>CSE — серверные сущности]
        SE[xrServerEntities]
        NS[xrNetServer<br/>TCP-сервер, протокол]
    end

    CORE --> ENGINE
    CORE --> RND
    ENGINE --> RND
    CPU --> ENGINE
    API --> RND
    D9 --> RND
    D10 --> RND
    R1 --> D9
    R1 --> RND
    RND --> GAME
    ENGINE --> GAME
    PART --> GAME
    SND --> GAME
    PHY --> GAME
    XML --> GAME
    CDB --> SE
    SE --> GAME
    NS --> SE
```

> **Чтение диаграммы**: стрелка `A --> B` означает «A зависит от B» (A `#include`'ит B).
> `xrCore` — единственный модуль без внутренних зависимостей; от него начинается всё.

## Что делает каждый слой

| Слой | Модули | Одна строка |
|---|---|---|
| Фундамент | `xrCore` | Всё, что нужно, чтобы компилировать и запускать C++: типы, память, строки, лог, файловая система, конфиги, сжатие, архивы. |
| Ядро | `xrEngine`, `xrCPU_Pipe` | Цикл приложения: создание/разрушение устройства, планировщик кадров (`CSheduler`), ввод, консоль, мост в Lua, CPU-оптимизации. |
| Рендер | `xrAPI`, `xrRender`, `xrRenderDX9/10`, `xrRenderPC_R*` | Абстракция GPU: компиляция шейдеров, константы, постобработка, выбор бэкенда (R4 = D3D9 — основной в Anomaly). |
| Игра | `xrGame`, `xrParticles`, `xrSound`, `xrPhysics`, `xrXMLParser` | Всё игровое: иерархия объектов, AI (ALIFE), физика, оружие, инвентарь, зоны, UI, мультиплеер-клиент. |
| Сервер | `xrCDB`, `xrServerEntities`, `xrNetServer` | Авторитетная серверная часть: серверные сущности (CSE), валидация, сетевой протокол, DSA-подпись. |

## Ключевые принципы

- **Слои не перепрыгиваются**. `xrGame` не `#include`'ит `xrRenderDX9` напрямую — только через `xrRender`/`xrAPI`. Рендерный бэкенд заменяем, игра не замечает.
- **`xrCore` — без зависимостей**. Это единственная DLL, которую можно линковать куда угодно. Все остальные модули тянут его.
- **Игра и сервер разделяются на сущности**: клиент ведёт `CGameObject` (наследуется от `CObject`), сервер — CSE (`xrCDB`). Состояние синхронизируется через сеть (см. [Сеть](../modules/network.md)).
- **Lua — не отдельный модуль, а пограничный слой**. C++ экспортирует объекты в Lua через `script_binder` и `ai_script_lua_extension`; обратные вызовы идут через `*_script.cpp`. См. [Граница скриптинга](scripting-boundary.md).

## Куда смотреть дальше

- [Карта модулей](module-map.md) — таблица: папка ↔ ответственность ↔ ключевые классы ↔ зависимости.
- [Зависимости](dependencies.md) — полный граф `#include` между модулями.
- [Цикл кадра](frame-loop.md) — канонический порядок: device → sheduler → Lua → render → physics.
- [Модель объектов](object-model.md) — иерархия `CObject` → `CGameObject` → `CActor`/`CWeapon`.
- [Граница скриптинга](scripting-boundary.md) — где C++ встречает Lua.

## Связанные модули (страницы)

Все страницы модулей: [xrCore](../modules/xr-core/index.md) · [xrEngine](../modules/xr-engine/index.md) · [Renderer](../modules/renderer/index.md) · [xrGame](../modules/game/index.md) · [Сервер](../modules/server.md) · [Сеть](../modules/network.md).
