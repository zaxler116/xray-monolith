# CAI_Stalker: движение и покрытия

> Модуль: `src/xrGame/`
> Файлы: `stalker_movement_manager_base.{h,cpp}` (151+863), `stalker_movement_manager_obstacles.{h,cpp}` (108+317), `stalker_movement_manager_obstacles_path.cpp` (178), `stalker_movement_manager_smart_cover.{h,cpp}` (209+608), `stalker_movement_manager_smart_cover_loopholes.cpp` (788), `stalker_movement_manager_smart_cover_fov_range.cpp` (112), `stalker_movement_manager_smart_cover_inline.h` (77), `stalker_movement_manager_base_inline.h` (156), `stalker_movement_manager_obstacles_inline.h` (18), `stalker_movement_manager_space.h` (72), `stalker_movement_params.{h,cpp}` (126+197), `stalker_movement_params_inline.h` (147), `stalker_movement_restriction.{h,inline}` (32+34), `stalker_velocity_collection.{h,cpp}` (30+56), `stalker_velocity_holder.{h,cpp}` (33+30), `cover_evaluators.{h,cpp}` (196+405), `smart_cover_evaluators.{h,cpp}` (230+343), `sight_manager.{h,cpp}` (137+931), `smart_cover_animation_planner.{h,cpp}` (93+~600), `smart_cover_animation_selector.{h,cpp}` (63+~300), `smart_cover_planner_target_selector.{h,cpp}` (51+~150), `stalker_movement_manager_obstacles_inline.h` (18).
> Порцион 14 (итерация 6).

Исполнительная часть `CAI_Stalker`: «ноги и голова» — уровень 2 стека движения поверх `CMovementManager` и **управление smart-покрытиями** (loopholes, анимации, FOV). Planner'ы (порц. 13) ставят `m_target` (куда/в какое покрытие/в какую позу), movement — приводит `m_current` к нему каждый кадр и управляет level-graph path, detail-path, обходом препятствий, дверями и анимацией перехода.

## Чего не делает / границы

- **Не принимает решений**: весь «мозг» = `brain().update` (порц. 13) → `target_params()`; здесь — только исполнение.
- **Покрытия монстров** (`CCoverPoint` + `CBaseMonster` FSM) — порц. 24; **steering/steps** (`CStepManager`, `detail_path_manager`) — порц. 25.
- **Анимация/стрельба** (`CStalkerAnimationManager`, `ai_stalker_fire.cpp`) — порц. 15; **`smart_cover::cover`/`loophole`/`description`** (данные из `.ltx`, `best_loophole`) — ит. 5 `cover-manager` + порц. 24.
- `Feel::*`/память/agent — ит. 5; `CSE_ALifeHumanStalker` — ит. 11/12.

## 1. Иерархия менеджеров

`ai_stalker.cpp:1388` `create_movement_manager()` → `xr_new<stalker_movement_manager_smart_cover>(this)`. Цепочка наследования:

| Класс | Файл | Роль |
|---|---|---|
| `CMovementManager` | `movement_manager.h` (ит. 5) | Базовый: level+detail path, `move_along_path`, `restrictions()`, `m_body`/`m_head` (SBoneRotation) |
| `stalker_movement_manager_base` | `stalker_movement_manager_base.{h,cpp}` | Stalker-параметры (`m_current`/`m_target`), velocity-маски, bad-path, `is_object_on_the_way` |
| `stalker_movement_manager_obstacles` | `stalker_movement_manager_obstacles.{h,cpp}` | Static+dynamic obstacle avoider, doors, `build_level_path` с симуляцией, `is_going_through` |
| `stalker_movement_manager_smart_cover` | `stalker_movement_manager_smart_cover.{h,cpp}` + `_loopholes` + `_fov_range` | Smart-покрытия: loopholes, переходы, FOV, анимации, target_selector |

`stalker_movement_manager_smart_cover` — единственный, реально создаваемый; два нижних уровня — `protected`/`private` API, доступные через `inherited::`.

Ключевые поля smart_cover: `m_path`/`m_temp_loophole_path` (LoopholePath = `xr_vector<shared_str>`), `m_target_selector` (`smart_cover::target_selector*`), `m_animation_selector` (`smart_cover::animation_selector*`), `m_current_transition`/`m_current_transition_animation`, `m_property_storage`, `m_apply_loophole_direction_distance` (4.0), `m_enter_animation`, `m_enter_cover_id`/`m_enter_loophole_id`, `m_ray_query_storage`, `m_entering_smart_cover_with_animation`, `m_non_animated_loophole_change`, `m_default_behaviour`, `m_check_can_kill_enemy`, `m_combat_behaviour`.

## 2. `stalker_movement_params` — состояние «куда»

`stalker_movement_params.{h,cpp}`. Держит 5 enum-полей + указатели на цели:

| Поле | Тип | Назначение |
|---|---|---|
| `m_body_state` | `MonsterSpace::EBodyState` | Stand/Crouch |
| `m_movement_type` | `MonsterSpace::EMovementType` | Stand/Walk/Run |
| `m_mental_state` | `MonsterSpace::EMentalState` | Free/Danger/Panic |
| `m_path_type` | `MovementManager::EPathType` | LevelPath/GamePath/PatrolPath/NoPath |
| `m_detail_path_type` | `DetailPathManager::EDetailPathType` | Smooth/Patrol/… |
| `m_desired_position` / `m_desired_direction` | `Fvector const*` (impl внутри) | Целевая точка/направление (null = «по path») |
| `m_cover_id` / `m_cover` | `shared_str` / `smart_cover::cover const*` | Текущее/целевое покрытие (lookup в `ai().cover_manager().smart_cover`) |
| `m_cover_loophole_id` / `m_cover_loophole` | `shared_str` / `loophole const*` | Зафиксированный loophole; `actualize_loophole()` lazy-вычисляет через `best_loophole(position, …, true, current==cover)` при пустом id (кэш 2 с, `time_before_selection`) |
| `m_cover_fire_object` / `m_cover_fire_position` | `CGameObject const*` / `Fvector const*` | Что стрелять (исключительные — объект и точка одновременно) |

`operator=` копирует impl'ы и переключает указатели на impl'ы (no aliasing). `equal_to_target` сравнивает все поля (для оптимизации «ничего не изменилось»). `construct(manager)` — биндинг к smart_cover-менеджеру (нужен для `position_to_cover_from()`).

Семантика: **`m_current`** — «что сейчас делаем», **`m_target`** — «куда должны прийти» (ставится planner'ами через `set_body_state`/`set_movement_type`/`set_mental_state`/`set_path_type`/`set_detail_path_type`/`set_desired_position`/`set_desired_direction` и через `cover_id`/`cover_loophole_id`/`cover_fire_object`/`cover_fire_position`). Каждый кадр `base::update` копирует `m_current = m_target`, затем `update(m_current)` приводит реальность к целевому состоянию.

## 3. Velocity-система

`stalker_movement_manager_space.h` — `eVelocities` битовые маски: движение (Standing/Walk/Run), поза (Stand/Crouch), ментальное (Danger/Free/Panic), направление (Positive/Negative). Комбинации дают 24 пресета (`eVelocityStandingFreeStand`, `eVelocityRunDangerCrouchPositive`, …).

`CStalkerVelocityCollection` (`stalker_velocity_collection.cpp`) читает из ini-секции: `m_danger[2][2][4]` (body_state × movement_type × direction), `m_free[2]` (walk/run), `m_panic` (run). `CStalkerVelocityHolder` — глобальный `associative_vector<shared_str, COLLECTION*>`, ленивый кэш по секции; `stalker_velocity_holder()` — singleton.

`base::init_velocity_masks` (`stalker_movement_manager_base.cpp:102`) — после `reload` регистрирует 16 пресетов в `detail().add_velocity(mask, STravelParams(linear, compute_angular, angular))`:
- Standing (Free/Panic/Danger) — linear=0, угловые скорости разные (Free: `PI/2*0.75` compute, `PI*2` real; Danger/Panic: `PI*2`).
- Walk/Run (Danger/Panic/Free) — линейные из `m_velocities->velocity(...)`, угловые `100*PI`/`PI`/`PI/2` (Danger быстрее поворачивается).
- Crouch-варианты — linear=0.

`parse_velocity_mask` (`:424`) — каждый кадр: если stand/пусто/path завершён → `m_fCurSpeed=0`, `m_body.speed=PI/2` (Free) или `PI*2` (Danger). Иначе берёт `point.velocity` (маска текущего detail-пункта), `detail().velocity(mask)` → `current_velocity`; если linear=0 и Free → `setup_body_orientation` + глушит sight (поворот на месте). Для Danger/Panic при крутом повороте (угол > порога) гасит линейную, ставит `PI` угловую. Из маски извлекает body/mental/movement state обратно в `movement_params` (обратная связь detail-path → параметры).

## 4. `stalker_movement_manager_base` — цикл обновления

`update(u32)` (`:624`): `VERIFY(mental != Free || body != Crouch)`; если `!enabled()` → return. `m_current = m_target`; `update(m_current)`.

`update(params)` (`:635`):
1. `if (m_force_update || movement_type != Stand) setup_movement_params(params)` — пересчёт level/detail dest по desired_position/direction + restrictions.
2. `if (script_control()) return;` — Lua-контроль скорости (`GetCurrentAction()->m_tMovementAction.m_fSpeed` → `set_desirable_speed`).
3. `setup_velocities` — маска в `detail()` (Danger: desirable+velocity = mask|Standing; иначе try_min_time, desirable|Standing, velocity|Walk|Standing).
4. `update_path` — rebuild level+detail при необходимости.
5. `parse_velocity_mask` — извлечение скоростей (п. 3).
6. `check_for_bad_path` — если Run+Danger: проверяет ближайшие 2 сегмента detail-path на резкий поворот (`>BAD_PATH_ANGLE = PI/2 - PI/8`) в пределах `BAD_PATH_DISTANCE_CHECK = 2.0` → понижает до Walk (не давит на анимацию).

`setup_movement_params` (`:270`): для GamePath/PatrolPath сбрасывает desired_position; `detail().set_path_type`, `level_path().set_evaluator(base_level_params())`; если desired_position — через `restrictions().accessible_nearest` (иначе прямой); если нет — берёт `level_path().dest_vertex_id()` и его позицию (с тем же accessible-гейтом). `desired_direction` → `detail().set_dest_direction` + `set_use_dest_orientation(true)`.

`set_nearest_accessible_position` (`:586`) — projection на level-graph vertex + plane_y, затем `restrictions().accessible_nearest` дважды (для позиции и вертекса). Используется в `initialize` и `on_restrictions_change`.

`is_object_on_the_way(object, distance)` (`:694`) — кэшированный query (объект, позиция, distance): проходит detail-path от текущего travel-point, `distance_to_line(p0, p1, p2)` (free func, проекция точки на отрезок) < 1.0 → true. Применяется planner'ами (порц. 13) для «враг на пути».

`body_state_combat_override(wo, body_state)` (`base_inline.h:51`) — Lua-хук `_G.CAI_Stalker__CombatSetBodyState(lua_game_object, wo, body_state)`; используется planner'ами для переопределения позы.

## 5. `stalker_movement_manager_obstacles` — препятствия и двери

`stalker_movement_manager_obstacles.{h,cpp}`. Поля: `m_static_obstacles`/`m_dynamic_obstacles` (`static_obstacles_avoider`/`dynamic_obstacles_avoider`), `m_doors_actor` (`doors::actor*`), `m_restricted_object` (`CRestrictedObjectObstacle*`), `m_level_path`/`m_detail_path`/`m_detail_current_index`/`m_detail_last_patrol_point` (сохранённое состояние для rollback), `m_saved_current_iteration` (обход), `m_last_dest_vertex_id`, `m_last_fail_time` (1 с дебаунс), `m_failed_to_build_path`.

`create_restricted_object` (`:52`) → `CRestrictedObjectObstacle(&object(), static.active_query(), dynamic.active_query())` — расширенная `CRestrictedObject` с учётом препятствий.

`move_along_path(movement_control, dest_position, time_delta)` (`:175`):
1. `ai().doors().actualize_doors_state(*m_doors_actor, old_desirable_speed())` — если дверь на пути не открыта → `set_desirable_speed(0)`, `inherited::move_along_path`, restore (сталкер ждёт).
2. Дебаунс: если `Device.dwTimeGlobal < m_last_fail_time + 1000` → `inherited::move_along_path` без обхода.
3. `if (!move_along_path())` (базовый level-path не построен) → `inherited::`.
4. `move_along_path_impl` (`:141`): `dynamic.update()` (если `!movement_enabled()` → стоп); `static.update()`; если `need_path_to_rebuild()` → `rebuild_path()` (level + detail, `set_build_path_at_once`).

`build_level_path` (`stalker_movement_manager_obstacles_path.cpp:108`) — основной rebuild:
- Если dest-вертекс изменился → `remove_query_objects(position, 5.0)` (очистка препятствий вокруг).
- `save_current_state` (level+detail+key_points+last_patrol+iteration).
- `static.inactive_query().copy(active).update_objects(position, 10000)`.
- Цикл: `inherited::build_level_path()` → если failed, одна попытка `pure_search` (очистка static, `invalidate_failed_info`, повтор) → `simulate_path_navigation()`.
- `simulate_path_navigation` (`:28`) — прогон detail-path с `predict_position(1.0, …)` и `static.query` на каждом шаге; при конфликте → `m_last_fail_time = now`, `restore_current_state`, return false.

`apply_border`/`remove_border`/`can_build_restricted_path` (`:77`/`:101`/`:111`) — временный «забор» вокруг query-области (level-graph `set_mask_no_check` + `restricted_object.add_border`), для проверки «построится ли path через ограничения».

`is_going_through(matrix, vector, max_distance)` (`:278`) — расстояние от отрезка (raycast) до detail-path: `get_distance` (free func, `LevelGraph::intersect` с safe_distance 0.35) по каждому сегменту, min. Используется для проверки «стреляет ли сквозь покрытие».

`on_death` (`:312`) — пересоздаёт `doors::actor` (ссылка на объект может устареть после смерти).

## 6. `stalker_movement_manager_smart_cover` — тик

`update(u32)` (`stalker_movement_manager_smart_cover.cpp:70`):
```
if (!m_current.cover()) {
    if (!m_target.cover()) { inherited::update(dt); return; }   // обычный режим
    enter_smart_cover(dt); return;                              // вход в покрытие
}
if (m_non_animated_loophole_change) non_animated_change_loophole();
if (!m_current.cover()) { inherited::update(dt); return; }
// синхронизация fire-цели
if (m_target.cover() && m_current.cover_loophole() == m_target.cover_loophole()) {
    m_current.cover_fire_object(m_target.cover_fire_object());
    m_current.cover_fire_position(m_target.cover_fire_position());
}
m_target_selector->update();
```

`enter_smart_cover(u32)` (`:312`) → `reach_enter_location(dt)` (`:186`):
- `m_current` = path_type/detail/mental/body/movement из target; `loophole = target.cover_loophole() ? (enterable ? target : nearest_enterable) : nearest_enterable`.
- Позиция = `current_transition().animation().position()` в world-space; projection на level-graph vertex + `restrictions().accessible_nearest`.
- `desired_position` + `desired_direction = cover->enter_direction(loophole)`; если `target_approached(m_apply_loophole_direction_distance=4)` → `sight().setup(eSightTypeDirection, direction, true)`.
- `inherited::update(m_current)` — построение path.
- Если path завершён и sight достиг: если `target_params().cover()->can_fire()` — проверяет `inventory().ActiveItem()` (если нет или не `INV_SLOT_3` → `set_goal(eObjectActionIdle, best_weapon)` и return).
- `animation().global().target_matrix(position, direction)`.
- Если `!current_transition().animation().has_animation()` → `enter_smart_cover()` (мгновенный вход).
- Иначе: `sight().setup(eSightTypeAnimationDirection, true, false)`, `on_smart_cover_enter` (`set_use_hit_anims(false)`), `m_entering_smart_cover_with_animation = true`, `m_enter_cover_id`/`m_enter_loophole_id`, `m_enter_animation = ID_Cycle(animation_id)`, биндинг `animation.global_selector(this, select_animation)` + `global_callback(this, on_animation_end)` + `global_modifier(this, modify_animation)`.

`select_animation` (`:140`) — callback анимационного менеджера: возвращает `m_enter_animation` (или 0 если не alive). `on_animation_end` (`:150`) — если `m_entering_smart_cover_with_animation`: сброс флага, если target.cover() → `enter_smart_cover()` (завершение входа). `modify_animation` (`:174`) — масштабирует `blend->speed` на `g_smart_cover_animation_speed_factor`.

`on_smart_cover_exit` (`:328`) — `set_use_hit_anims(true)`, сброс transition/animation, `animation_selector->finalize`, `unbind_global_selector`, `m_current.cover_id("")`, `inherited::update(m_current)`.

`non_animated_change_loophole` (`_loopholes.cpp:612`) — смена loophole без анимации: `setup_movement_params` (desired = `cover.fov_position(loophole)`, `m_movement_type=Run`, body_state из transition), `inherited::update(m_current)`, если path завершён и sight достиг → `go_next_loophole()`, `animation_selector->on_animation_end` + `planner().update`.

`setup_movement_params` (`_loopholes.cpp:653`) — для non-animated change: если path.size()==1 → сброс флага; иначе `loophole_id = m_path[1]`, `m_movement_type=Run`, body_state из `current_transition_animation().body_state()`, desired = `cover.fov_position(loophole)`, level_vertex = `cover.level_vertex_id(loophole)`.

`go_next_loophole` (`_loopholes.cpp:561`) — `try_actualize_path`, если `m_path.size()==1` → return; если `m_path[0]` = enter-vertex → `m_current.cover_id/loophole_id` из target; если `m_path[1]` = exit-vertex → `m_current.cover_id("")` + `on_smart_cover_exit`; иначе `m_current.cover_loophole_id(m_path[1])`.

`try_actualize_path` (`_loopholes.cpp:499`) — lazy rebuild `m_path`: если пуст → `actualize_path`; если `m_path.front() != current_loophole_id` → rebuild; если `m_path.back() != target_loophole_id` (или cover-переход) → rebuild. `actualize_path` (`:446`) — 4 ветки: enter (`build_enter_path`), exit (`build_exit_path`), cover→cover (`build_exit_path_to_cover`), внутри одного cover (`loophole_path`).

### 6.1 Построение loophole-path

`loophole_path(cover, source, target, path)` (`_loopholes.cpp:367`) — `ai().graph_engine().search(cover.description()->transitions(), source, target, &path, CBaseParameters(-1,-1,-1))` — BFS/Dijkstra по графу переходов покрытия.

`enter_path(result, position, level_vertex_id, cover, target_loophole_id)` (`:34`) — перебор всех enterable loopholes: `loophole_path(enterable → target)`, `new_value = distance(fov_position, position) + g()`, min. `result->insert(begin, transform_vertex("", true))` (enter-vertex). Возвращает вес.

`build_enter_path` (`:80`) — `target_loophole_id = transform_vertex(m_target.cover_loophole()->id() or "", false)`; `enter_path(&m_path, position, level_vertex_id, target_cover, target_loophole_id)`. `m_current_transition = &action(cover, m_path[0], m_path[1])` (или 0).

`build_exit_path` (`:279`) — перебор exitable loopholes: `loophole_path(current → exitable)`, `new_value = g() + exit_edge_weight + exit_path_weight(exit_vertex → target_position)`. `nearest_action(cover, exitable, exit_vertex, target_position, …, &m_target.m_body_state)` — выбор ближайшей animation-точки с учётом body_state. `m_path.push_back(transform_vertex("", false))` (exit-vertex).

`build_exit_path_to_cover` (`:352`) — переход между покрытиями: перебор exitable current → `nearest_action` → `enter_path(0, exit_position, exit_vertex, target_cover, target_loophole)` (без result — только вес). `selected_action` запоминается.

`nearest_action(cover, id0, id1, position, result_position, result_vertex_id, target_body_state)` (`:168`) — перебор `edge->data()` (ActionsList) → `animations()` → `transform_tiny(position)`, `distance_to_sqr`, `test_pick` (raycast через `collide::ray_defs` + `feel_vision_mtl_transp` — прозрачность), `body_state` match. Возвращает ближайший `transition_action`.

`action(cover, id0, id1)` (`:255`) — первый `applicable()` action на edge.

### 6.2 FOV и range

`_fov_range.cpp`:
- `fill_enemy_position(position)` — `m_current.cover_fire_position` → memory enemy position.
- `enemy_in_fov()` — `best_loophole(position, value, false, true)` или `is_position_in_fov` + `is_position_in_range` текущего loophole.
- `in_fov(cover_id, loophole_id, position)` / `in_range(...)` / `in_current_loophole_fov(position)` / `in_current_loophole_range(position)` / `in_min_acceptable_range(...)` — прямые делегаты на `smart_cover::cover` (с lookup по id).

`position_to_cover_from()` (`_loopholes.cpp:763`) — для `best_loophole`: `cover_fire_position` → `cover_fire_object` (через memory, если есть visual/sound/hit info) → `memory().enemy().selected()` → `object().Position()`.

### 6.3 Target selector и animation

`m_target_selector` (`smart_cover::target_selector`) — `CActionPlannerAction<animation_planner>`: принимает `EWorldProperties` target (`eWorldPropertyLoopholeIdle`/`Lookout`/`Fire`/`FireNoLookout`) через `target_idle()`/`target_lookout()`/`target_fire()`/`target_fire_no_lookout()` (`_smart_cover.cpp:490`/`:505`/`:520`/`:540`); Lua-callback через `target_selector(callback)`. `update()` — выбор целевого действия.

`m_animation_selector` (`smart_cover::animation_selector`) — обёртка над `animation_planner`: `select_animation`/`on_animation_end`/`modify_animation` (callbacks в `CStalkerAnimationManager`), `planner()` → `animation_planner&`.

`smart_cover::animation_planner` (`smart_cover_animation_planner.h`) — `CActionPlannerScript<CAI_Stalker>` (как root planner'ы порц. 13): `CWorldState m_target`, `m_time_object_hit`, `m_loophole_value`, `m_last_transition_time`, `m_default_idle_interval`/`m_default_lookout_interval`, `m_head_speed`, `m_idle_min_time`/`m_idle_max_time`, `m_lookout_min_time`/`m_lookout_max_time`, `m_stay_idle`, `m_last_idle_time`/`m_last_lookout_time`. `setup(object, storage)` — инициализация; `initialize`/`finalize` — вход/выход из покрытия; `target(EWorldProperties)` — смена цели; `hit_callback(SHit const*)` — обработка попадания (сброс idle/lookout).

`smart_cover::target_selector` (`smart_cover_planner_target_selector.h`) — `CActionPlannerAction<animation_planner>`: `callback_type m_script_callback` (Lua), `CRandom m_random`; `update()` — выбор target через evaluators.

### 6.4 Smart-cover evaluators

`smart_cover_evaluators.{h,cpp}` — 12 evaluators (все `CStalkerPropertyEvaluator` или `CPropertyEvaluator<animation_planner>`):

| Evaluator | Evaluate |
|---|---|
| `in_cover_evaluator` | `!!movement().current_params().cover()` |
| `cover_actual_evaluator` | `current.cover() == target.cover()` (VERIFY current.cover()) |
| `cover_entered_evaluator` | `!!current.cover()` (дублирует in_cover) |
| `loophole_actual_evaluator` | `current.cover == target.cover && current.loophole == target.loophole` |
| `loophole_hit_long_ago_evaluator` | `(time_object_hit + time_to_wait) < now` |
| `is_action_available_evaluator` | `current.cover_loophole()->is_action_available(action_id)` |
| `loophole_planner_const_evaluator` | const bool |
| `loophole_exitable_evaluator` | `current.cover_loophole()->exitable()` |
| `can_exit_loophole_with_animation` | `current_transition().animation().has_animation()` (при смене cover/loophole) |
| `default_behaviour_evaluator` | `default_behaviour() || combat_behaviour()` |
| `can_fire_at_enemy_evaluator` | `!default_behaviour || cover_fire_position || enemy_in_fov()` |
| `idle_time_interval_passed_evaluator` | `stay_idle && now <= last_idle_time + interval` (иначе сброс) |
| `lookout_time_interval_passed_evaluator` | `!stay_idle && now <= last_lookout_time + interval` (иначе переключение) |

## 7. `cover_evaluators` — выбор покрытия (6 оценщиков)

`cover_evaluators.{h,cpp}`. `CCoverEvaluatorBase` — базовый: `m_selected` (`CCoverPoint const*`), `m_loophole` (`smart_cover::loophole*`), `m_best_value`, `m_best_loophole_value`, `m_start_position`, `m_object` (`CRestrictedObject*`), `m_stalker` (`CAI_Stalker*`), `m_use_smart_covers_only`, `m_can_use_smart_covers`, `m_inertia_time`. `evaluate(cover_point, weight)` — если `!m_is_smart_cover` → `evaluate_cover`; иначе `evaluate_smart_cover` (только если `is_combat_cover()` и cvar `aiUseSmartCovers`). `inertia(position, radius)` — «остаться в текущем»: radius не уменьшился + время < inertia + текущий loophole в danger_fov + `can_fire_right_now()`.

6 наследников (создаются в `ai_stalker.cpp` как `m_ce_*`):

| Evaluator | `evaluate_cover` | `evaluate_smart_cover` |
|---|---|---|
| `CCoverEvaluatorCloseToEnemy` | min `enemy_distance` в [min, max] с deviation | пусто |
| `CCoverEvaluatorFarFromEnemy` | max `enemy_distance` (value = -distance) | пусто |
| `CCoverEvaluatorBest` | `threat_on_the_way` (угол < 30° и проекция < 1.5×) → reject; `cover_value = min(high, low)` + `10` если neighbor в direction; **`+1000` если товарищ в радиусе 1.5 м** (agent_manager); value /= weight | `best_loophole(enemy_position, value, false, current==cover)`, `value *= g_smart_cover_factor` |
| `CCoverEvaluatorAngle` | `m_best_direction` (предвычислен в `initialize` по 360 углам `compute_high/low_square`); max `cos(direction, best_direction)` | пусто |
| `CCoverEvaluatorSafe` | min `min(high, low)` vertex cover value, `distance > min_distance` | пусто |
| `CCoverEvaluatorAmbush` | `cover_from_enemy / cover_from_myself` (min из high/low в обе стороны), `my_distance > min_enemy_distance` | пусто |

`g_smart_cover_factor = 1.0` (global, тюнинг веса smart-покрытий).

## 8. `CSightManager` — поворот головы и тела

`sight_manager.{h,cpp}`. `CSetupManager<CSightControlAction, CAI_Stalker, u32>` — очередь `CSightAction` (eSightTypeCurrentDirection/PathDirection/Direction/AnimationDirection/…). Поля: `m_current` (bone_parameters: spine/shoulder/head — rotation+factor), `m_target` (base: rotation), `m_animation_id`, `m_aiming_type` (none/weapon/head), `m_animation_frame`, `m_max_left_angle`/`m_max_right_angle` (90°/60° из ini), `m_enabled`, `m_turning_in_place`.

`Exec_Look(dt)` (`:103`) — вызывается из `UpdateCL` (порц. 12 §2.1): нормализация yaw/pitch (body+head), `body_speed`/`head_speed` (из `current_action().change_body_speed/head_speed`), `vfValidateAngleDependency` (запрет пересечения head/body), `angle_lerp_bounds` × 4 (body.yaw/pitch, head.yaw/pitch) с `select_speed(distance, speed, aim_min_speed, aim_min_angle, aim_max_angle)` — плавное уменьшение скорости при малых углах. `compute_aiming(dt, head_speed)` + `current_action().on_frame()`. Если `!animation_movement_controlled` и нет Holder — обновление `m_object->XFORM()` из `body.current.yaw`.

`compute_aiming` (`:684`) — switch `m_aiming_type`:
- `aiming_none` → `process_action(dt)`: lerp `m_current.m_*.m_factor` к `s_danger_factors = {0, 0.5, 0.5}` (torso look) или `s_free_factors = {0.25, 0.25, 0.5}` (head look) со скоростью `s_factor_lerp_speed = 1.0`; `m_current.m_*.m_rotation = angles × factor` (angles = head − body, нормализованные).
- `aiming_weapon` — если `eSightTypeAnimationDirection` → identity; иначе `aimers::weapon aimer(…)` (bone_spin/shoulder/weapon_bone0/2 из ini + `aiming_position()`) → `m_target.m_spine/shoulder = aimer.get_bone(0/1)`, `m_target.m_head = Fidentity`.
- `aiming_head` — аналогично через `aimers::bone`.

`update()` (`:245`) — `CSetupManager::update` (обработка очереди) + turning-in-place: если `movement().speed() == 0` и угол head−body > max_left/right → `m_turning_in_place = true`, `body.target.yaw = head.current.yaw` (тело догоняет голову на месте).

`setup(CSightAction)` (`:233`) — очистка очереди, добавление нового action. `aiming_position()`/`object_position()` — позиции для aimers. `bone_aiming(animation_id, frame, type)` — настройка `m_aiming_type`/`m_animation_id`/`m_animation_frame`.

## 9. `CStalkerMovementRestrictor` — фильтр покрытий

`stalker_movement_restriction.{h,inline}`. Функтор: `operator()(cover)` → `agent_manager().location().suitable(object, cover, use_enemy_info)`; `weight(cover)` → `location().danger(cover, object)`; `finalize(cover)` → `location().make_suitable(object, cover)`. Используется planner'ами (порц. 13) при `find_best_cover` для фильтрации по agent-location.

## Связи

```mermaid
graph TD
  S[CAI_Stalker] --> MV[stalker_movement_manager_smart_cover "movement()"]
  MV --> OB[stalker_movement_manager_obstacles]
  OB --> BA[stalker_movement_manager_base]
  BA --> MM[CMovementManager "ит. 5"]
  MV --> TP[stalker_movement_params "m_current / m_target"]
  TP --> CP[smart_cover::cover / loophole "ит. 5 cover-manager"]
  MV --> TS[target_selector]
  MV --> AS[animation_selector]
  AS --> AP[animation_planner "CActionPlannerScript"]
  MV --> SE[smart_cover_evaluators "12 eval"]
  MV --> CE[cover_evaluators "6 eval"]
  S --> SI[CSightManager "sight()"]
  SI --> SA[CSightControlAction "eSightType*"]
  BA --> VC[CStalkerVelocityCollection / Holder]
  BA --> RP[CStalkerMovementRestrictor]
  OB --> DO[doors::actor]
  OB --> SA2[static/dynamic_obstacles_avoider]
  P13["порц. 13 planner'ы"] --> TP
  P13 --> CE
  P15["порц. 15 анимация/стрельба"] --> AS
```

- Мост: `CMovementManager` + level/detail path — ит. 5; `smart_cover::cover`/`loophole`/`description` — ит. 5 `cover-manager` + порц. 24.
- Planner'ы (порц. 13) ставят `target_params` и вызывают `find_best_cover` (через `cover_evaluators` + `CStalkerMovementRestrictor`).
- `CStalkerAnimationManager` (порц. 15) — потребитель `animation_selector`/`animation_planner`; `ai_stalker_fire.cpp` (порц. 15) — `can_fire_right_now`, `aiming_position`.
- `Feel::*`/память/agent — ит. 5; `CSE_ALifeHumanStalker` — ит. 11/12.

## Замечания к плану

1. **Два уровня «куда»**: `m_target` (planner'ы) vs `m_current` (исполнение). `base::update` копирует target → current каждый кадр, затем приводит реальность (path, velocity, body/head orientation). Smart-покрытия добавляют третий уровень — `m_path` (loophole-путь), который lazy-ребилдится через `try_actualize_path`.
2. **`desired_position` vs cover**: установка `desired_position`/`desired_direction` сбрасывает `cover_id` (и наоборот). Mutually exclusive — planner'ы должны выбрать одно.
3. **Non-animated loophole change** — «быстрая» смена loophole внутри покрытия без анимации (flag `m_non_animated_loophole_change`); используется при `target_fire_no_lookout` и т. п.
4. **`check_can_kill_enemy`** — flag, устанавливаемый planner'ами; используется в `enemy_in_fov`/`can_fire_at_enemy_evaluator` для гейта «можно ли стрелять из покрытия».
5. **`combat_behaviour`** — flag (в отличие от `default_behaviour`), включающий «боевое» поведение в coverage (не ждать idle/lookout).
6. **`g_smart_cover_factor`** (1.0) и **`g_smart_cover_animation_speed_factor`** — глобальные cvar-подобные константы для тюнинга веса smart-покрытий и скорости анимаций перехода.
7. **`+1000` за товарища в радиусе 1.5 м** в `CCoverEvaluatorBest` — анти-кластеризация: сталкеры разбредаются по разным покрытиям (vanilla-поведение, не в оригинале S.T.A.L.K.E.R.).
8. **`CSightManager` не в `CAI_Stalker::Think`** — вызывается из `UpdateCL` (client-side), а не из server-side тика. Это осознанное разделение: поворот головы — визуальный эффект, не влияет на логику.
9. **`animation_planner` (smart cover)** — отдельный `CActionPlannerScript`, не связанный с `brain()` (root planner'ы порц. 13). Два независимых planner-стека: `brain()` (решения) + `animation_planner` (поведение в покрытии).

## Краткое резюме

`stalker_movement_manager_smart_cover` — исполнительная часть `CAI_Stalker` (уровень 2 стека движения): 3-уровневая иерархия (base → obstacles → smart_cover), `stalker_movement_params` (m_current/m_target: 5 enum + desired position/direction + cover/loophole/fire), velocity-маски (24 пресета из ini), bad-path check (Run→Walk при резком повороте), static+dynamic obstacle avoider с симуляцией path, doors-ожидание, smart-покрытия (loophole-path через `graph_engine` на графе переходов, enter/exit/cover-to-cover, `nearest_action` с raycast-прозрачностью, FOV/range делегаты, non-animated change), 12 smart-cover evaluators, 6 cover evaluators (Best с анти-кластеризацией +1000), `CSightManager` (angle_lerp с aim-speed, turning-in-place, bone-aiming weapon/head), `CStalkerMovementRestrictor` (фильтр по agent-location). Planner'ы (порц. 13) ставят `target_params` и вызывают `find_best_cover`; исполнение — здесь.

Порцион 14 (итерация 6). Далее: порц. 15 — `stalker-anim-trade.md` — `CAI_Stalker`: анимация, стрельба, торговля ([stalker-anim-trade.md](stalker-anim-trade.md)).
