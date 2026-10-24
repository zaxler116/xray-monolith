# Память монстров (monster_*_memory + *_manager)

Модуль: `src/xrGame/ai/monsters/` — 6 пар h/cpp:
`monster_enemy_memory` (267+39), `monster_sound_memory` (231+130),
`monster_hit_memory` (190+40), `monster_corpse_memory` (154+37),
`monster_enemy_manager` (343+85), `monster_corpse_manager` (86+35).
Порцион 22 (итерация 6).

## Чего не делает

- Не содержит `memory_manager`/`visual_memory_manager`/`sound_memory_manager`
  (глобальные, корень `src/xrGame/` — это ит. 11/12, `CSE_ALife*`).
- Не содержит `actor_memory` (игрок — ит. 14).
- Не управляет FSM — только хранит и предоставляет данные.
- Не знает про Lua API (ит. 14).

## §1. Архитектура: два слоя

Каждый монстр имеет **6 членов-памятей** (`base_monster.h:300-306`):

| Член | Тип | Роль |
|---|---|---|
| `EnemyMemory` | `CMonsterEnemyMemory` | Множество врагов + опасность |
| `SoundMemory` | `CMonsterSoundMemory` | Множество звуков + опасность |
| `CorpseMemory` | `CMonsterCorpseMemory` | Множество трупов |
| `HitMemory` | `CMonsterHitMemory` | Множество хитов |
| `EnemyMan` | `CMonsterEnemyManager` | **Один** выбранный враг + флаги |
| `CorpseMan` | `CMonsterCorpseManager` | **Один** выбранный труп |

**Два слоя**:
- **Memory** (множественный): хранит множество объектов, обновляет danger,
  удаляет устаревшие. `update()` вызывается из `Think()`.
- **Manager** (ед. число): выбирает **один** лучший объект из memory,
  хранит его позицию/вершину/время, вычисляет флаги. `update()` тоже из `Think()`.

```
Memory (множество)          Manager (один)
  EnemyMemory ────────→     EnemyMan
  SoundMemory ────────────→  (чтение звука для позиции)
  CorpseMemory ───────→     CorpseMan
  HitMemory ───────────────── (не имеет manager, читается напрямую)
```

## §2. Данные (ai_monster_defs.h)

### `SMonsterEnemy` (`:513`)

| Поле | Тип | Назначение |
|---|---|---|
| `position` | `Fvector` | Последняя известная позиция |
| `vertex` | `u32` | Вершина level graph |
| `time` | `TTime` | Время последнего обновления |
| `danger` | `float` | Оценка опасности (пересчитывается) |

`ENEMIES_MAP` = `xr_map<const CEntityAlive*, SMonsterEnemy>`.

### `SMonsterCorpse` (`:525`)

Аналогично, но без `danger`. `CORPSE_MAP`.

### `SMonsterHit` (`:535`)

| Поле | Тип | Назначение |
|---|---|---|
| `object` | `CObject*` | Кто нанёс хит |
| `time` | `TTime` | Время хита |
| `side` | `EHitSide` | Сторона (front/back/left/right) |
| `position` | `Fvector` | Позиция монстра при хите |

`MONSTER_HIT_VECTOR` = `xr_vector<SMonsterHit>`.

### Флаги врага (`ai_monster_defs.h:49-61`)

13 битовых флагов: `FLAG_ENEMY_DIE`, `LOST_SIGHT`, `GO_CLOSER/FARTHER`,
`GO_CLOSER/FARTHER_FAST`, `STANDING`, `HIDING`, `RUN_AWAY`,
`DOESNT_KNOW_ABOUT_ME`, `GO_OFFLINE`, `DOESNT_SEE_ME`, `STATS_NOT_READY`.

## §3. `CMonsterEnemyMemory` (monster_enemy_memory*)

Поля: `monster`, `time_memory` (дефолт 15000 мс), `m_objects` (ENEMIES_MAP).

### `update()` (`:33`)

Вызывается каждый тик из `Think()`. Цепочка:

1. **Hit-реакция** (`:43`): если `HitMemory.is_hit()` и хит в последние 1000 мс —
   добавляет объект-источник хита как врага (если `useful` и в пределах
   `get_feel_enemy_who_just_hit_max_distance()`). Для собаки — `squad->set_home_in_danger()`.
2. **Sound-реакция** (`:64`): если `SoundMemory` не пуст и актёр видит монстра —
   берёт самый опасный звук; если dangerous и в последние 2000 мс —
   добавляет `sound.who` как врага (если `useful`, y_dist < 10, xz_dist <
   `get_feel_enemy_who_made_sound_max_distance()`).
3. **Обход memory().enemy().objects()** (`:94`): для каждого объекта —
   если `feel_enemy` (дистанция < `get_feel_enemy_max_distance()`)
   или `visible_now` → `add_enemy`.
4. **Актёр** (`:108`): отдельно проверяет `g_actor` — если в пределах
   `feel_enemy_max_distance`, y_dist < 10, `is_useful`, и актёр видит монстра
   → `add_enemy(g_actor)`.
5. **`remove_non_actual()`** (`:176`): удаляет если:
   - объект `NULL`, мёртв, или `getDestroy()`,
   - `time + time_memory < now` (старый),
   - `g_Team() == monster->g_Team()` (своя команда),
   - `!memory().enemy().is_useful()` (не полезен).
6. **Пересчёт danger** (`:126`): для каждого врага:
   `danger = (1 + relation³) / (1 + dist)`.
   `relation` = `tfGetRelationType()` (0–4, 4 = худший враг).

### `add_enemy` (два overload)

- `add_enemy(enemy)` (`:134`): обновляет position/vertex/time, `danger = 0`.
- `add_enemy(enemy, pos, vertex, time)` (`:155`): то же, но принимает явные
  параметры; обновляет только если `time` новее (для `transfer_enemy`).

### `find_best_enemy()` (`:218`)

**Двухпроходный поиск**:
1. Первый проход: только враги **в home** (`Home->at_home(position)`),
   выбирает максимальный `danger`.
2. Если не найдено: второй проход по всем, максимальный `danger`.

### `get_enemy()` / `get_enemy_info()`

Возвращают `it->first` / `it->second` от `find_best_enemy()`.

## §4. `CMonsterSoundMemory` (monster_sound_memory*)

Поля: `time_memory`, `Sounds` (`xr_vector<SoundElem>`, reserve 20),
`monster`, `m_time_help_sound`, `m_help_node`.

### `SoundElem` (struct в .h, `:41`)

| Поле | Назначение |
|---|---|
| `who` | Кто издал звук |
| `type` | `TSoundDangerValue` (19 типов, см. ниже) |
| `position` | Позиция звука (не объекта!) |
| `power` | Мощность (громкость) |
| `time` | Время обнаружения |
| `value` | Оценка опасности (пересчитывается) |

### `TSoundDangerValue` (`:8-32`)

19 значений: `WEAPON_SHOOTING`, `MONSTER_ATTACKING`, `WEAPON_BULLET_RICOCHET`,
`WEAPON_RECHARGING`, `WEAPON_TAKING`, `WEAPON_HIDING`, `WEAPON_CHANGING`,
`WEAPON_EMPTY_CLICKING`, `MONSTER_DYING`, `MONSTER_INJURING`, `MONSTER_WALKING`,
`MONSTER_JUMPING`, `MONSTER_FALLING`, `MONSTER_TALKING`, `DOOR_OPENING`,
`DOOR_CLOSING`, `OBJECT_BREAKING`, `OBJECT_FALLING`, `NONE_DANGEROUS_SOUND`.

Порядок = приоритет (меньше индекс = опаснее).

### `CalcValue(cur_time, cur_pos)` (`:77`)

```
value = 8 * (NONE_DANGEROUS_SOUND - type)       // тип: чем раньше в enum, тем опаснее
      - 1 * floor(dist)                          // дистанция: чем ближе, тем опаснее
      - 2 * floor((cur_time - time) / 1000)      // свежесть: чем новее, тем опаснее
      + 50 * floor(power)                        // громкость: чем громче, тем опаснее
```

`operator<` — по `value` (для `max_element`).

### `ConvertSoundType(ESoundTypes)` (`:9`)

13 `CHECK_SOUND_TYPE` макросов: битовая маска `SOUND_TYPE_*` → `TSoundDangerValue`.
Если не совпала ни одна → `NONE_DANGEROUS_SOUND`.

### `HearSound` (два overload)

- `HearSound(SoundElem&)` (`:55`): отбрасывает `NONE_DANGEROUS`, `DOOR_*`,
  `MONSTER_WALKING` без `who`. Ищет в массиве: если есть тот же `who` + `type`
  и `time >= existing.time` → заменяет; иначе → push_back.
- `HearSound(who, eType, pos, power, time)` (`:80`): обёртка — создаёт `SoundElem`,
  `SetConvert`, `CalcValue`, вызывает первый overload.

### `UpdateHearing()` (`:144`)

1. Удаляет: `who->getDestroy()`, `time < now - time_memory`, `who` мёртв.
2. Пересчитывает `CalcValue` для всех.
3. Сбрасывает `m_time_help_sound` если прошло > `time_help_sound_remember` (10000 мс).

### Get-методы

| Метод | Возврат |
|---|---|
| `GetSound(s, bDangerous)` (`:98`) | Самый опасный (`max_element` по `value`); `bDangerous = (type <= WEAPON_EMPTY_CLICKING)` |
| `GetFirstSound(s, bDangerous)` (`:90`) | Первый в массиве |
| `GetSound()` (`:109`) | `SoundElem&` самого опасного |
| `get_sound_from_object(who, s)` (`:174`) | Поиск по `who` |

### Help sounds

- `hear_help_sound()` (`:215`): `true` если `m_time_help_sound` задан и не истёк.
- `check_help_sound(eType, node)` (`:221`): если звук содержит
  `MONSTER_ATTACKING` + `MONSTER_INJURING` + `MONSTER_DYING` (три маски сразу)
  и help не задан → запоминает время и вершину.
- `hear_help_sound_node()` — вершина, где слышен help.

## §5. `CMonsterHitMemory` (monster_hit_memory*)

Поля: `monster`, `time_memory` (дефолт 10000 мс), `m_hits` (MONSTER_HIT_VECTOR).

### `update()` (`:22`)

`remove_non_actual()` — удаляет если `time + time_memory < now`
или `object` мёртв.

### `add_hit(who, side)` (`:33`)

Создаёт `SMonsterHit` (position = позиция монстра). Если `who` уже есть —
обновляет; иначе push_back.

### `get_last_hit_dir()` (`:85`)

Находит хит с максимальным `time`. Вращает `monster->Direction()` по `side`:
back → +PI, left → +PI/2, right → -PI/2.

### Другие get-методы

| Метод | Возврат |
|---|---|
| `is_hit()` | `!m_hits.empty()` |
| `is_hit(pO)` | `who == pO` в массиве |
| `get_num_hits()` | `m_hits.size()` |
| `get_last_hit_time()` | Макс `time` |
| `get_last_hit_object()` | `object` последнего хита |
| `get_last_hit_position()` | `position` последнего хита |

## §6. `CMonsterCorpseMemory` (monster_corpse_memory*)

Поля: `monster`, `time_memory` (дефолт 10000 мс), `m_objects` (CORPSE_MAP).

### `update()` (`:26`)

Обход `memory().item().objects()`: если `visible_now` и объект —
`CEntityAlive` и **мёртв** (`!g_Alive()`) → `add_corpse`.
Затем `remove_non_actual()`.

### `add_corpse(corpse)` (`:46`)

Если `corpse` `NULL` или `is_locked_corpse()` → ignore.
Иначе: обновляет/добавляет в map.

### `remove_non_actual()` (`:73`)

Удаляет если: `NULL`, **жив** (`g_Alive()`), `getDestroy()`,
`time + time_memory < now`, `m_fFood < 1` (съеден), `is_locked_corpse()`.

### `find_best_corpse()` (`:127`)

Простой: минимальная дистанция до `monster->Position()`.

### `get_corpse()` (`:104`)

Возвращает лучший труп, если не `is_locked_corpse()`, иначе `NULL`.

## §7. `CMonsterEnemyManager` (monster_enemy_manager*)

Поля: `monster`, `enemy` (один!), `position`, `vertex`, `time_last_seen`,
`flags` (Flags32), `forced`, `expediency`, `prev_enemy`, `prev_enemy_position`,
`enemy_see_me`, `danger_type`, `my_vertex_enemy_last_seen`,
`enemy_vertex_enemy_last_seen`, `m_time_updated`, `m_time_start_see_enemy`,
`m_script_enemy`.

### `update()` (`:40`)

1. **Script enemy**: если `m_script_enemy` мёртв/уничтожен → сбрасывает.
2. **Forced**: если `forced` — проверяет валидность `enemy`; если мёртв → `enemy = 0`.
3. **Не-forced**:
   - `m_script_enemy` → `enemy = m_script_enemy`.
   - Иначе → `enemy = EnemyMemory.get_enemy()` (лучший из памяти).
   - Обновляет `position/vertex/time_last_seen` из `EnemyMemory`.
4. **Sound-позиция** (`:81`): если в `SoundMemory` есть звук от `enemy`
   и он новее `time_last_seen` → обновляет `position` из звука
   (вершина сбрасывается на `u32(-1)`).
5. **`enemy_see_me`** = `is_faced(enemy, monster)`.
6. **`danger_type`** через `dwfChooseAction` (0 → `eWeak`, 1–4 → `eStrong`).
7. **Флаги** (`:114`):
   - `LOST_SIGHT`: тот же враг, но не виден сейчас.
   - `DIE`: предыдущий враг мёртв.
   - `DOESNT_SEE_ME`: враг не видит монстра.
   - `STANDING` / `GO_CLOSER` / `GO_FARTHER` (+`_FAST` если delta < 1.2):
     сравнение `dist_now` vs `dist_prev`.
   - `DOESNT_KNOW_ABOUT_ME`: `STANDING` + `DOESNT_SEE_ME`.
   - `STATS_NOT_READY`: новый враг (не `prev_enemy`).
8. **Вершины** (`:150`): если `see_enemy_now()` — запоминает
   `my_vertex_enemy_last_seen`, `enemy_vertex_enemy_last_seen`,
   `m_time_start_see_enemy`.

### `see_enemy_now()` / `see_enemy_recently()`

- `see_enemy_now()` (`:223`): `memory().visual().visible_right_now(enemy)`.
- `see_enemy_recently()` (`:233`): `memory().visual().visible_now(enemy)`
  (включает «недавно видел»).

### `enemy_see_me_now()` (`:243`)

Если враг = актёр: `Actor()->memory().visual().visible_right_now(monster)`.
Если враг = монстр: `cast_custom_monster()->memory().visual().visible_right_now(monster)`.

### `is_faced(obj0, obj1)` (`:261`)

Проверка «смотрит ли obj0 на obj1»:
- Дистанция < `ffGetRange()`.
- Расчёт FOV: `fYawFov` расширен на `atan(1/dist)` (для близких целей).
- Углы yaw/pitch в пределах FOV.

### `force_enemy` / `unforce_enemy`

- `force_enemy` (`:163`): устанавливает `enemy`, `forced = true`, вызывает `update()`.
- `unforce_enemy` (`:175`): берёт из `EnemyMemory`, `forced = false`.

### `transfer_enemy(friend)` (`:301`)

Копирует врага от друга (для squad): `EnemyMemory.add_enemy(friend.enemy, pos, vertex, time)`.

### `script_enemy()`

- Без аргумента: сбрасывает `m_script_enemy = 0`.
- С аргументом: устанавливает `m_script_enemy` (приоритет над EnemyMemory).

## §8. `CMonsterCorpseManager` (monster_corpse_manager*)

Поля: `monster`, `corpse`, `position`, `vertex`, `time_last_seen`, `forced`.

### `update()` (`:24`)

- **Forced**: если `corpse->m_fFood < 1` → `corpse = 0`.
- **Не-forced**: `corpse = CorpseMemory.get_corpse()`; обновляет позицию.

### `force_corpse` / `unforce_corpse`

Аналогично enemy manager.

## §9. Взаимодействие со FSM

| Потребитель | Методы | Назначение |
|---|---|---|
| `group_state_attack` | `EnemyMan.get_enemy()`, `get_enemy_position()`, `get_flags()` | Целевой враг, флаги для подкрадывания |
| `group_state_attack_run` | `EnemyMan.get_enemy()`, `get_enemy_position()` | Предсказание траектории |
| `group_state_panic` | `EnemyMan.get_enemy_time_last_seen()`, `HitMemory.get_last_hit_time()` | `check_force_state` |
| `group_state_rest` | `SoundMemory.GetSound()` | `hear_dangerous_sound` |
| `group_state_eat` | `CorpseMan.get_corpse()` | Труп для поедания |
| `group_state_eat_drag` | `CorpseMan`, `EatedCorpse` | Захват трупа |
| `group_state_hear_danger_sound` | `SoundMemory.GetSound().position` | Направление звука |
| `monster_state_attack_on_run` | `EnemyMemory.get_memory()` | Перебор всех врагов |
| `monster_state_steal` | `EnemyMan.get_flags()` | `GO_FARTHER_FAST`, `DOESNT_KNOW_ABOUT_ME` |
| `bloodsucker_*` | `EnemyMan.see_enemy_now()`, `enemy_see_me_now()` | Взаимная видимость |
| `chimera_*` | `HitMemory.is_hit()` | Реакция на хит |
| `controller_*` | `EnemyMan.see_enemy_now()`, `HitMemory.is_hit()` | Стрельба/укрытие |
| `burer_*` | `EnemyMan.see_enemy_now()`, `enemy_see_me_now()` | Телепорт/щит |

## §10. Связи

```mermaid
graph TD
    subgraph "Memory (множество)"
        EM[CMonsterEnemyMemory]
        SM[CMonsterSoundMemory]
        CM[CMonsterCorpseMemory]
        HM[CMonsterHitMemory]
    end

    subgraph "Manager (один)"
        EMan[CMonsterEnemyManager]
        CMan[CMonsterCorpseManager]
    end

    BM[CBaseMonster]
    MM[memory().enemy()]
    VM[memory().visual()]
    AM[memory().item()]
    AC[g_actor]

    BM --> EM
    BM --> SM
    BM --> CM
    BM --> HM
    BM --> EMan
    BM --> CMan

    EM -->|update читает| MM
    EM -->|visible_now| VM
    EM -->|g_actor| AC
    CM -->|visible_now| VM
    CM -->|objects| AM
    EMan -->|get_enemy| EM
    EMan -->|get_sound| SM
    EMan -->|visible_right_now| VM
    CMan -->|get_corpse| CM

    subgraph "FSM (порц. 18-20)"
        GA[CStateGroupAttack]
        GP[CStateGroupPanic]
        GR[CStateGroupRest]
        GE[CStateGroupEat]
    end

    GA --> EMan
    GP --> HM
    GP --> EMan
    GR --> SM
    GE --> CMan
```

## Замечания к плану

- `CMonsterEnemyMemory::update()` — **самый сложный** метод: 4 источника
  врагов (hit, sound, enemy memory, actor) + пересчёт danger.
- `CMonsterEnemyManager::update()` — **второй по сложности**: 8 блоков,
  включая флаги, danger_type, вершины.
- `HitMemory` **не имеет** отдельного manager — читается напрямую
  (`monster->HitMemory.is_hit()`, `get_last_hit_time()`, `get_last_hit_dir()`).
- `CMonsterSoundMemory` не обновляется в `update()` — `UpdateHearing()`
  вызывается отдельно (из `Think()` или `shedule_Update`).
- `m_script_enemy` в `CMonsterEnemyManager` — Lua-механизм: скрипт
  может принудительно задать врага. Приоритет над `EnemyMemory`.
- `find_best_enemy()` — **двухпроходный**: сначала ищет в home,
  потом по всем. Это значит, что враг у дома всегда приоритетнее
  (даже если danger ниже).
- `SoundElem::CalcValue` — формула с 4 коэффициентами
  (FACTOR_SOUND_TYPE=8, DISTANCE=1, DELTA_TIME=2, POWER=50).
  Громкость доминирует (×50).

## Краткое резюме

6 классов (4 memory + 2 manager), ~1400 стр. кода.
**Memory** хранит множества (враги/звуки/трупы/хиты), обновляет danger,
удаляет устаревшие. **Manager** выбирает один лучший объект,
хранит его позицию/вершину, вычисляет 13 флагов и danger_type.
`EnemyMan` — самый потребляемый: `get_enemy()`, `get_enemy_position()`,
`see_enemy_now()`, `enemy_see_me_now()`, `get_flags()`.
Ключевые механики: двухпроходный поиск врага (home first),
4 источника обнаружения (hit/sound/visual/actor),
формула danger с кубом relation.

Порцион 22 (итерация 6). Далее: порц. 23 — `enemy.md` — враги: `CEnemyManager`, `CAgentEnemyManager`, `CMemberEnemy` ([enemy.md](enemy.md)).
