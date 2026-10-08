# Установка

X-Ray Monolith — Windows-only (D3D9/10, PhysX, BASS). Сборка — Visual Studio 2022.

## Требования

| Компонент | Версия | Примечание |
|---|---|---|
| OS | Windows 10/11 x64 | 32-бит не поддерживается |
| Компилятор | Visual Studio 2022 (v143) | `engine-vs2022.sln` |
| DirectX SDK | September 2010 | D3D9/10 заголовки |
| PhysX SDK | 3.x | в `sdk/` |
| BASS/FMOD | в `sdk/` | звук |
| Theora | в `sdk/` | видео |
| Git | любой | submodules |

## Клонирование

```bash
git clone --recurse-submodules <url> xray-monolith
cd xray-monolith
```

`--recurse-submodules` обязателен: `src/3rd party/` (Lua, luabind, PhysX и т.д.) — подмодули.

## Структура

```
xray-monolith/
├── src/                  # исходники движка
│   ├── engine.sln        # решение VS2022
│   ├── engine-vs2022.sln # альтернативное решение
│   ├── batch_build.bat   # batch-сборка
│   ├── xrCore/           # фундамент
│   ├── xrEngine/         # ядро
│   ├── Layers/           # рендер
│   ├── xrGame/           # игра
│   ├── xrCDB/            # серверные сущности
│   ├── xrServerEntities/ # мост сервера
│   ├── xrNetServer/      # сетевой сервер
│   ├── xrParticles/      # частицы
│   ├── xrSound/          # звук
│   ├── xrPhysics/        # физика
│   ├── xrXMLParser/      # XML
│   ├── xrCPU_Pipe/       # CPU-оптимизации
│   └── 3rd party/        # зависимости (submodules)
├── sdk/                  # бинарные SDK (PhysX, Theora, ...)
│   ├── include/
│   ├── libraries/
│   └── binaries/
├── gamedata/             # игровые данные
├── compressor/           # утилита сжатия LTX
├── docs/                 # эта вики (MkDocs)
└── .gitmodules
```

## Быстрый старт

1. Установить VS2022 + DirectX SDK 2010.
2. `git clone --recurse-submodules`.
3. Открыть `src/engine-vs2022.sln`.
4. Build → Solution (Release x64).
5. Запустить `gamedata/xr_3d.exe` (или из `gamedata/` — зависит от версии).

## Горячая перезагрузка (hot reload)

Для быстрого цикла разработки — hot reload через VS2022 (см. [Сборка](building.md) → «Hot Reload»).

## Диагностика

Проблемы со сборкой/запуском — [Диагностика](troubleshooting.md).
