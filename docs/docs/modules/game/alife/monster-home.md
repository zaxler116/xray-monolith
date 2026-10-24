# Дом монстра (monster_home)

Модуль: `src/xrGame/ai/monsters/monster_home.h` (48 стр.) + `monster_home.cpp` (549 стр.)
Порцион 21 (итерация 6).

## Чего не делает

- Не управляет FSM-состояниями — только предоставляет «географию» дома.
- Не содержит level graph / path builder — вызывает их методы.
- Не знает про Lua API (ит. 14) — только `script_game_object_use2.cpp` вызывает `Home->setup/remove`.
- Не управляет поведением — только отвечает на вопросы «где дом?», «внутри ли?», «куда идти?».

## §1. Роль в архитектуре

`CMonsterHome` — член `CBaseMonster` (`base_monster.h:342`, поле `Home`).
Каждый монстр имеет **один** дом, определяющий:

- **Центр** — либо одна вершина level graph (`m_level_vertex_id`),
  либо набор вершин патрульного пути (`m_path` → `CPatrolPath`).
- **Три радиуса**: `min`, `middle`, `max` — концентрические зоны.
- **Флаг агрессии** — `m_aggressive` (дом «горячий», монстр агрессивен у себя).

Все `group_states` (порц. 20) и одиночные состояния (порц. 19) опрашивают `Home`
для принятия решений: «вернуться ли домой», «убежать от звука», «куда тащить труп».

## §2. Поля

| Поле | Тип | Назначение |
|---|---|---|
| `m_object` | `CBaseMonster*` | Владелец |
| `m_path` | `const CPatrolPath*` | Патрульный путь (множество вершин) или `0` |
| `m_level_vertex_id` | `u32` | Одна вершина (если `m_path == 0`); `u32(-1)` = не задан |
| `m_radius_min` | `float` | Внутренняя зона (по умолчанию 20) |
| `m_radius_middle` | `float` | Средняя зона (по умолчанию 30) |
| `m_radius_max` | `float` | Внешняя зона (по умолчанию 40) |
| `min_move_dist` | `u32` | Мин. дистанция «блуждания» (по умолчанию 7) |
| `max_move_dist` | `u32` | Макс. дистанция «блуждания» (по умолчанию 10) |
| `m_aggressive` | `bool` | Дом агрессивный |

**Инвариант**: `min < middle < max` (гарантируется в `setup`/`load`).

## §3. Инициализация

### `load(line)` (`monster_home.cpp:45`)

Вызывается из `base_monster_startup.cpp:267` при старте монстра.
Читает секцию `line` из `spawn_ini()`:

| Ключ ini | Поле | Дефолт |
|---|---|---|
| `path` | `m_path` (через `ai().patrol_paths().path(name)`) | `0` |
| `radius_min` | `m_radius_min` | 20 |
| `radius_max` | `m_radius_max` | 40 |
| `radius_middle` | `m_radius_middle` | `(min+max)/2` |
| `min_move_dist` | `min_move_dist` | 7 |
| `max_move_dist` | `max_move_dist` | 10 |

Валидации:
- `VERIFY3(m_radius_max > m_radius_min, …)` — жёсткая.
- `radius_middle` вне `[min, max]` → `(min+max)/2`.
- `min_move_dist >= max_move_dist` → сброс к дефолтам (7, 10).
- `m_aggressive = false` всегда (в ini не читается).

В DEBUG: `check_path()` проверяет, что патрульный путь на том же уровне,
что и монстр (`game_graph` level_id сравнение).

### `setup(path_name, min, max, aggressive, middle)` (`:97`)

Программная настройка (из Lua: `script_game_object_use2.cpp:348`).
Аналогична `load`, но без ini, с явным `aggressive`.

### `setup(lv_ID, min, max, aggressive, middle)` (`:115`)

То же, но с одной вершиной вместо пути. `m_path = 0`.

### `remove_home()` (`:518`)

`m_path = 0`, `m_level_vertex_id = u32(-1)`, `m_aggressive = false`.
Вызывается из Lua (`script_game_object_use2.cpp:360`).

### `set_move_dists(min, max)` (`:525`)

Перезапись `min_move_dist`/`max_move_dist` (только если `max > min`).
Вызывается из `dog.cpp:229`.

## §4. Зоны и проверки «внутри»

```
        ┌─────────────────────────────────┐
        │         max_radius (40)         │
        │    ┌─────────────────────┐      │
        │    │   mid_radius (30)   │      │
        │    │    ┌───────────┐    │      │
        │    │    │  min (20) │    │      │
        │    │    │           │    │      │
        │    │    └───────────┘    │      │
        │    └─────────────────────┘      │
        └─────────────────────────────────┘
```

### `at_home(pos, radius)` (`:487`)

- Если `m_path == 0`:
  - `m_level_vertex_id` невалидна → `true` (дома нигде).
  - Иначе: `dist(pos, vertex_pos) < radius`.
- Если `m_path != 0`:
  - **Итерация по всем вершинам пути**: если хотя бы одна в радиусе → `true`.

### Обёртки

| Метод | Радиус |
|---|---|
| `at_home()` (`:466`) | `m_radius_max` (позиция объекта) |
| `at_home(pos)` (`:472`) | `m_radius_max` |
| `at_home(pos, radius)` (`:487`) | явный |
| `at_min_home(pos)` (`:477`) | `m_radius_min` |
| `at_mid_home(pos)` (`:482`) | `m_radius_middle` |

## §5. Точка дома

### `get_home_point()` (`:534`)

- `m_path == 0`: позиция `m_level_vertex_id`; если невалидна — `object->Position()`.
- `m_path != 0`: позиция **первой** вершины пути (`vertex(0)`); если пусто — `object->Position()`.

Используется как «якорь» для расчётов направлений
(например, `home2sound` в `group_state_hear_danger_sound`).

## §6. Выбор вершины в зоне

Все методы `get_place*` возвращают `u32` — level vertex id
(`u32(-1)` = не найдена).

Общий паттерн:
1. Определить `input_node` (центр поиска).
2. `path_builder().get_node_in_radius(input_node, r_min, r_max, attempts, result)`.
3. Если `result == u32(-1)` — фолбэк (см. ниже).

### `get_place_in_min_home()` (`:133`)

- `input_node`: случайная вершина пути ИЛИ `m_level_vertex_id` ИЛИ текущая позиция.
- `get_node_in_radius(input, 1, m_radius_min, 5, result)`.
- Фолбэк: если `input_node` недоступен → текущая позиция объекта; иначе → `input_node`.

### `get_place_in_mid_home()` (`:164`)

- Если объект **не в mid_home** или **в min_home**:
  - `input_node` как выше.
  - `get_node_in_radius(input, m_radius_min, m_radius_middle, 5, result)`.
- Если объект **в mid_home** (не в min):
  - **Блуждание**: `m_move_dist = min_move_dist + rand(max_move_dist - min_move_dist)`.
  - До 10 итераций: берёт текущий `Direction()`, добавляет случайный угол
    (`±PI/4`, после 5 итераций — `±PI/3`), вычисляет точку на `m_res_dist`,
    проверяет `valid_vertex_position`.
  - Если найдена — `vertex_id(point)`; иначе `get_node_in_radius(input, m_res_dist-1, m_res_dist, 5, result)`.
- Фолбэк: `input_node` если валидна и доступна; иначе текущая позиция.
- **Валидация**: если `result` невалидна или не в mid_home → рекурсия в `get_place_in_min_home()`.

### `get_place_in_max_home_to_direction(to_direction)` (`:246`)

Наиболее сложный метод. Находит точку **в max_home в заданном направлении**
(от центра дома). Используется для «убежать от звука/врага вглубь дома».

- `m_res_dist = m_radius_middle + (max - middle)/2 - (max - middle)/10`
  (т.е. ~70% от max, смещённое к middle).
- До 10 итераций: `to_direction` + случайный угол (`±PI/5`, после 5 — `±PI/4`),
  вычисляет точку, проверяет `valid_vertex_position`.
- Если найдена: `get_node_in_radius(point, 1, (max - middle)/2, 5, result)`.
- Если не найдена — **повтор с другой дистанцией**:
  `m_res_dist = min + (middle - min)/2 - (middle - min)/10` (ближе к min).
- Фолбэки: `input_node` → `get_place_in_min_home()` → `get_place_in_max_home()`.

### `get_place_in_max_home()` (`:355`)

- `input_node`: случайная вершина пути ИЛИ `m_level_vertex_id` ИЛИ текущая.
- `get_node_in_radius(input, m_radius_min, m_radius_max, 5, result)`.
- Фолбэк: как в min_home.

### `get_place()` (`:386`)

«Произвольная точка дома» (для патрулирования):
- `m_path == 0`:
  - `get_node_in_radius(m_level_vertex_id, min, min + (max-min)/2, 5, result)`.
  - Если не найдено: `get_node_in_radius(current, 5, 15, 5, result)`.
  - Если не найдено: `get_node_in_radius(current, 2, 3, 10, result)`.
  - Если не найдено: текущая вершина.
- `m_path != 0`:
  - Случайная вершина пути → `get_node_in_radius(vertex, min, min+(max-min)/2, 5, result)`.
  - Фолбэк: сама вершина (если доступна) или текущая.

### `get_place_in_cover()` (`:436`)

- `m_path == 0`: `CoverMan->find_cover(vertex_pos, vertex_pos, min, min+(max-min)/2)`.
- `m_path != 0`: случайная вершина пути → `CoverMan->find_cover(point, point, min, min+(max-min)/2)`.
- Возвращает `point->level_vertex_id()` или `u32(-1)`.

## §7. Методы доступа

| Метод | Возврат |
|---|---|
| `get_min_radius()` | `m_radius_min` |
| `get_mid_radius()` | `m_radius_middle` |
| `get_max_radius()` | `m_radius_max` |
| `has_home()` | `m_path != 0 && m_level_vertex_id != u32(-1)` |
| `is_aggressive()` | `m_aggressive` |

## §8. Использование в FSM

| Потребитель | Методы Home | Назначение |
|---|---|---|
| `group_state_attack` (§5, порц. 20) | `at_home`, `at_mid_home`, `is_aggressive`, `get_home_point` | Логика агрессии |
| `group_state_attack_run` (§6) | `at_home` | Обход (encirclement) |
| `group_state_rest` (§3) | `at_min_home` | Усыпление |
| `group_state_rest_idle` (§4) | `get_place_in_min_home`, `get_place_in_mid_home`, `get_home_point`, `get_min/mid_radius` | Прогулка |
| `group_state_eat_drag` (§11) | `get_place_in_min_home`, `get_home_point`, `get_min_radius` | Точка для трупа |
| `group_state_hear_danger_sound` (§14) | `get_home_point`, `get_place_in_max_home_to_direction` | Убежать от звука |
| `group_state_home_point_attack` (§8) | `at_home`, `get_home_point`, `get_place_in_max_home_to_direction`, `get_place_in_min_home` | Отход домой |
| `group_state_panic_run` (§13) | `get_home_point`, `get_place_in_max_home_to_direction` | Бег от врага |
| `group_state_squad_move_to_radius` (§7) | `get_home_point` | Направление дуги |
| `monster_state_rest_fun` | `at_home` (corpse) | Можно ли есть |
| `monster_state_hear_int_sound` | `has_home`, `at_home`, `get_place` | Реакция на звук |
| `monster_state_home_point_danger` | `at_home`, `get_place_in_cover`, `get_place` | Опасность у дома |
| `monster_state_home_point_rest` | `get_place_in_mid_home`, `is_aggressive` | Возврат домой |
| `monster_state_home_point_attack` | `get_place_in_cover`, `get_place` | Отход |
| `bloodsucker_attack_state` | `at_home` (enemy) | Атака у дома |
| `bloodsucker_attack_state_hide` | `has_home`, `get_place_in_cover`, `get_place` | Скрытие |
| `bloodsucker_predator` | `has_home`, `get_place_in_cover`, `get_place` | Скрытие |

## §9. Связи

```mermaid
graph TD
    MH[CMonsterHome]
    BM[CBaseMonster.Home]
    PP[CPatrolPath]
    LG[ai().level_graph()]
    PB[path_builder()]
    CM[CoverMan]

    BM -->|владеет| MH
    MH -->|m_path| PP
    MH -->|vertex_pos| LG
    MH -->|get_node_in_radius| PB
    MH -->|find_cover| CM

    subgraph "Потребители (FSM)"
        GA[CStateGroupAttack]
        GR[CStateGroupRest]
        GE[CStateGroupEat]
        GP[CStateGroupPanic]
        GHS[CStateGroupHearDangerousSound]
        GHA[CStateGroupAttackMoveToHomePoint]
    end

    GA --> MH
    GR --> MH
    GE --> MH
    GP --> MH
    GHS --> MH
    GHA --> MH
```

## Замечания к плану

- `CMonsterHome` — **не-template**, не-интерфейс: конкретный класс, один на монстра.
- `m_path` и `m_level_vertex_id` — взаимоисключающие: либо путь, либо одна вершина.
  `has_home()` проверяет **оба** (и `m_path != 0` И `m_level_vertex_id != -1`),
  что выглядит как баг: при `m_path != 0` вершина не используется, но `has_home()`
  требует её валидности. На практике `setup(path_name, …)` не заполняет
  `m_level_vertex_id`, поэтому `has_home()` может вернуть `false` для валидного дома-пути.
- `get_place_in_mid_home()` содержит **рекурсию** в `get_place_in_min_home()` —
  потенциально глубокая, но на практике одна итерация (min_home всегда найдётся).
- `min_move_dist` / `max_move_dist` используются **только** в `get_place_in_mid_home`
  для «блуждания» монстра, который уже в mid_home.
- `m_aggressive` **не читается из ini** — только через `setup(…, aggressive, …)`.
  В `load()` всегда `false`.

## Краткое резюме

`CMonsterHome` (549 стр.) — геометрическое ядро «дома» монстра:
3 концентрических радиуса (min/mid/max), центр (одна вершина или патрульный путь),
флаг агрессии. 6 методов выбора вершины (`get_place*`) с каскадными фолбэками.
15+ потребителей в FSM (group_states и states). Не содержит логики поведения —
только отвечает «где?», «внутри?», «куда идти?».

Порцион 21 (итерация 6). Далее: порц. 22 — `memory.md` — память монстров (enemy/sound/hit/corpse memory + enemy/corpse manager) ([memory.md](memory.md)).
