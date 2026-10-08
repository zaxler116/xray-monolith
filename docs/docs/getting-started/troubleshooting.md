# Диагностика

Типовые проблемы при сборке/запуске X-Ray Monolith и их решения.

## Сборка

### Ошибка: `Cannot find D3DX9_43.lib` / `d3d9.h`

**Причина**: не установлен DirectX SDK September 2010.

**Решение**:

1. Скачать DirectX SDK 2010 (архив, Microsoft больше не раздаёт).
2. Установить (по умолчанию `C:\Program Files (x86)\Microsoft DirectXTech SDK (March 2010)`).
3. В VS2022: Project → Properties → VC++ Directories → Include/Library — добавить пути SDK.

### Ошибка: `Cannot find PhysX headers`

**Причина**: не инициализированы git submodules.

**Решение**:

```bash
git submodule update --init --recursive
```

### Ошибка: `LNK2019: unresolved external symbol`

**Причина**: неверный порядок линковки или пропущенная `.lib`.

**Решение**:

- Проверить `Additional Dependencies` в `.vcxproj`.
- Убедиться, что `xrCore` линкуется первым (фундамент).
- Для `xrGame` — проверить, что `xrEngine` и `xrRender` в зависимостях.

### Ошибка: `C1083: Cannot open include file: 'fastdelegate.h'`

**Причина**: не склонированы submodules (`src/3rd party/`).

**Решение**: `git submodule update --init --recursive`.

## Запуск

### Игра не запускается, в логе: `Failed to load xrRenderPC_R4.dll`

**Причина**: DLL не в `gamedata/` или зависимость (D3D9) не найдена.

**Решение**:

1. Убедиться, что `gamedata/xrRenderPC_R4.dll` существует.
2. Проверить, что DirectX 9 End-User Runtime установлен.
3. Запустить с `-v` для подробного лога.

### Игра крашится при старте: `R_ASSERT failed`

**Причина**: разные версии DLL (например, `xrCore.dll` из другого билда).

**Решение**:

1. Полностью очистить `gamedata/` (удалить все `.dll`).
2. Пересобрать с нуля.
3. Скопировать свежий `gamedata/`.

### Игра крашится: `Access violation in xrGame.dll`

**Причина**: несовместимость Lua-скриптов с версией движка, или битый `gamedata/`.

**Решение**:

1. Проверить, что `gamedata/scripts/` соответствует версии движка.
2. Запустить с `-no_script` (если поддерживается) для исключения Lua.
3. Проверить `gamedata/system.ltx` на правильные пути.

### Сервер не стартует: `Cannot bind port`

**Причина**: порт занят или права.

**Решение**:

1. Проверить `gamedata/server.ltx` → `port`.
2. Сменить порт (например, 27015 → 27016).
3. Запускать от администратора (если за Firewall'ом).

## Логирование

- **Основной лог**: `gamedata/logs/` (или `gamedata/*.log` — зависит от версии).
- **Повышенная диагностика**: запуск с `-dbg` (детальный лог, см. [Логирование](../modules/xr-core/logging.md)).
- **Профилировщик**: `xrCore/profiler.h` — `PROFILE_START`/`PROFILE_END` (включается `#define USE_PROFILER`).

## Консоль

В игре — консоль (по умолчанию `~` или `Tab`):

- `help` — список команд
- `bus_list` — все shader bus lanes (см. [Shader Bus](../modules/renderer/shader-bus.md))
- `stat` — статистика кадров
- `r_...` — рендерные переменные
- `ai_...` — AI-переменные

## Как сообщить о проблеме

При репорте бага включать:

1. Версию движка (commit hash).
2. Конфигурацию сборки (Debug/Release).
3. Полный лог из `gamedata/logs/`.
4. Шаги воспроизведения.
5. Системные требования (OS, GPU, DirectX).
