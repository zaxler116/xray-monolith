# Сборка

## Решения

| Файл | Назначение |
|---|---|
| `src/engine.sln` | Основное решение (старое) |
| `src/engine-vs2022.sln` | Решение для VS2022 (рекомендуется) |
| `src/batch_build.bat` | Batch-сборка (для CI) |

## Конфигурации

| Конфигурация | Описание |
|---|---|
| `Debug` | Полная отладка, ассерты, логирование |
| `Mixed` | Смешанная (debug-инструменты + release-оптимизация) |
| `Release` | Продакшен, оптимизация, без ассертов |
| `Master_Gold` | Релизная сборка (определяется в `xrCore.h`: `#ifndef DEBUG #define MASTER_GOLD`) |

## Defines

`src/build_config_defines.h` — ключевые `#define` для сборки:
- `XRCORE_EXPORTS` / `XRCORE_STATIC` — экспорт/статика `xrCore`
- `BENCHMARK_BUILD` — benchmark-режим (см. `xrCore.h`)
- `DEDICATED_SERVER` — сборка только сервера
- `_EDITOR` — сборка редактора (ингайм)

## Сборка

### Через VS2022

1. Открыть `src/engine-vs2022.sln`.
2. Выбрать конфигурацию (Release x64).
3. Build → Build Solution.

### Через batch

```bat
cd src
batch_build.bat
```

## Hot Reload (VS2022)

Для быстрого цикла «изменил код → увидел в игре»:

1. В VS2022: Debug → Start Debugging (F5) — игра запустится.
2. Изменить `.cpp` (не `.h` — изменение заголовков требует пересборки зависимостей).
3. Build → Build Project (Ctrl+Shift+B).
4. VS2022 предложит **Hot Reload** (заменить DLL в запущенном процессе).
5. Изменения вступают в силу без перезапуска игры.

> **Ограничения hot reload**:
> - Работает только для `.cpp`-изменений (не `.h`).
> - Не работает для изменений в `xrCore` (фундамент, перезагрузка DLL ломает состояние).
> - Не работает для статических/глобальных инициализаций.
> - Для `xrGame`/`xrEngine` — работает хорошо.

## Артефакты

После сборки:
- `gamedata/xr_3d.exe` — клиент
- `gamedata/xrServer.exe` — сервер (если собран `DEDICATED_SERVER`)
- `gamedata/xrCore.dll`, `xrEngine.dll`, `xrGame.dll`, ... — DLL-модули
- `gamedata/xrRenderPC_R4.dll` — рендер

## Частые проблемы сборки

См. [Диагностика](troubleshooting.md).
