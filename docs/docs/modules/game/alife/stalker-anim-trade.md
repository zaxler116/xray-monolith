# CAI_Stalker: анимация, стрельба, торговля

> Модуль: `src/xrGame/`
> Файлы: `stalker_animation_manager.{h,cpp}` (242+144), `stalker_animation_manager_update.cpp` (265), `stalker_animation_manager_inline.h` (121), `stalker_animation_manager_impl.h` (41), `stalker_animation_manager_debug.cpp` (252), `stalker_animation_pair.{h,cpp}` (114+325), `stalker_animation_pair_inline.h` (128), `stalker_animation_script.{h,cpp}` (33+99), `stalker_animation_names.{h,cpp}` (31+119), `stalker_animation_state.{h,cpp}` (37+36), `stalker_animation_data.{h,cpp}` (29+17), `stalker_animation_data_storage.{h,cpp}` (33+64), `stalker_animation_callbacks.cpp` (242), `stalker_animation_legs.cpp` (306), `stalker_animation_torso.cpp` (394), `stalker_animation_global.cpp` (96), `stalker_animation_head.cpp` (49), `smart_cover_animation_selector.{h,cpp}` (63+219), `ai/stalker/ai_stalker_fire.cpp` (1442), `ai_stalker_alife.cpp` (459), `stalker_combat_action_base.cpp` (379), `object_handler.{h,cpp}` (94+366), `object_handler_planner.{h,cpp}` (83+335), `object_handler_space.h` (105).
> Порцион 15 (итерация 6).

Три самодостаточных слоя `CAI_Stalker`: **анимация** (`CStalkerAnimationManager` — 5 независимых «пар» анимаций скелета), **стрельба/броски/получение урона** (`ai_stalker_fire.cpp` — dispersion, fire-queue, can_kill raycast, throw, critical wounds) и **торговля/выбор оружия** (`ai_stalker_alife.cpp` — `CTradeItem`, `choose_*`, `conflicted`).

## Чего не делает / границы

- **Не принимает решений**: выбор «стрелять/ждать/брать покрытие» — planner'ы (порц. 13, `CStalkerActionCombatBase::fire/aim_ready`); здесь — исполнение и данные.
- **`CObjectHandler::update`** — сам цикл исполнения целей (show/hide/reload/fire/throw) — порц. 26; здесь — `set_goal` + queue-параметры + planner-структура.
- **Данные smart-покрытий** (`smart_cover::cover`/`loophole`) — ит. 5 + порц. 24; **движение** — порц. 14; **память/враги/agent** — порц. 22–23; Lua API — ит. 14.

## 1. `CStalkerAnimationManager` — 5 «пар» анимаций

`stalker_animation_manager.{h,cpp}`. TODO в коде прямо говорит: «consists of 5 independent managers, they should be represented with the different classes». Реализовано как один класс с 5 полями типа `CStalkerAnimationPair`:

| Поле | Тип | Роль |
|---|---|---|
| `m_global` | pair | Полноростовые: critical hit, panic/escape, damage FX |
| `m_head` | pair | Голова (idle/talk), bone-part fake |
| `m_torso` | pair | Корпус: draw/attack/drop/holster/reload/pick/aim/walk/run/idle/prepare/strap/unstrap/look_back/escape |
| `m_legs` | pair | Ноги: walk/run/escape × fwd/back/left/right |
| `m_script` | pair | Script-анимации (Lua), hand_usage, movement controller |

Каждая пара — `(MotionID, CBlend*, flags, callbacks)` + `step_dependence` (для legs/global/script: синхронизация с `CStepManager` через `on_animation_start`), `global_animation` (для global/script: `play_global_animation` по всем bone parts). `USE_HEAD_BONE_PART_FAKE` (определён в `stalker_animation_pair.h:17`) — global/script-анимации играют на всех parts кроме head (маска `m_script_bone_part_mask = all_bone_parts ^ (1 << bone_part)`), чтобы head-анимация не конфликтовала.

Данные: `CStalkerAnimationData` (`stalker_animation_data.{h,cpp}`) — `m_part_animations` (`CAniCollection<CStalkerAnimationState, state_names>`, по `cr_`/`norm_`/`dmg_norm_`), `m_head_animations` (`head_idle_0`/`head_talk_0`), `m_global_animations` (`item_*`). Кэш — `CStalkerAnimationDataStorage` (`stalker_animation_data_storage.cpp`): вектор пар `(IKinematicsAnimated*, CStalkerAnimationData*)`, `data_predicate` сравнивает motion slots (не указатель!), `stalker_animation_data_storage().object(skeleton)` лениво создаёт.

## 2. `CStalkerAnimationPair` — парная анимация

`stalker_animation_pair.{h,cpp}`. Поля: `m_animation` (MotionID), `m_blend` (CBlend*), `m_actual` (true = «уже играет, не перезапускать»), `m_step_dependence`, `m_global_animation`, `m_array`/`m_array_animation` (кэш `select()`), `m_callbacks` (FastDelegate0, вызываются в `on_animation_end`), `m_target_matrix` (для animation movement controller), `m_callback_on_collision`, `m_just_started` (DEBUG).

`play(...)` — если `actual()` → `m_just_started=false`, return (не перезапускать). Иначе: если `!global_animation()` → `PlayCycle(animation, TRUE, callback, m_object)` (один part) + `step_dependence` → сохранение позиции (`fmod(timeCurrent, timeTotal)/timeTotal`, для standing → 0.5); если `global_animation()` → `play_global_animation` (цикл по `MAX_PARTS`, `LL_PlayCycle` для каждого, `use_animation_movement_control` → `create_anim_mov_ctrl(blend, m_target_matrix, local_animation)`, иначе `animation_movement()->stop()`). `m_step_dependence` → `m_object->CStepManager::on_animation_start(animation(), blend())`.

`select(array, weights)` — если `m_array == &array` → return cached `animation()`; иначе `select_animation`: без weights → `array[::Random.randI(array.size())]`; с weights → `Random.randF() * accumulator`, поиск первого `accumulator + weight >= chosen`. Кэширует `m_array`/`m_array_animation` (для `select()` на следующий кадр, чтобы не пере-рандомить).

`on_animation_end()` — `make_inactual()`, `m_blend = 0`, вызов всех `m_callbacks` (через `buffer_vector` в `_alloca`, чтобы callback мог модифицировать `m_callbacks`). `reset()` — invalidate animation, `m_blend = 0`, `m_actual = true` (чтобы следующий `play` не перезапустил). `synchronize(skeleton, other)` — если оба motion имеют `esmSyncPart` flag → `blend()->timeCurrent = other.blend()->timeCurrent` (для torso↔legs sync).

## 3. Update-цикл

`stalker_animation_manager_update.cpp` + вызовы из `ai_stalker.cpp`:

- `ai_stalker.cpp:1159` — `animation().play_delayed_callbacks()` в `UpdateCL` (client-side, после `update_object_handler`).
- `ai_stalker.cpp:1361` — `animation().update()` в `UpdateCL`.
- `smart_cover_animation_selector.cpp:59` — `m_object->animation().update()` в `initialize()` (для RayQuery, до `UpdateCL`).

`update_impl()`:
1. `if (!g_Alive()) return;`
2. `update_tracks()` — если `need_update()` → `m_skeleton_animated->UpdateTracks()`. `need_update()` = `script_callback()` (script_animations непусты и `object().callback(eScriptAnimation)` существует) ИЛИ `non_script_need_update()` (`m_global_selector`/`m_global_callback`/любая пара `need_update()`).
3. `play_delayed_callbacks()` — `m_call_script_callback` → `object().callback(eScriptAnimation)()`; `m_call_global_callback` → `m_global_callback()`.
4. `#ifdef HOLDERCUSTOM_NEW: if (object().Holder()) return;`
5. `if (play_script()) return;` — если `script_animations()` непусты.
6. `if (play_global()) return;` — если `assign_global_animation()` вернул valid.
7. `play_head(); play_torso(); play_legs();` — дефолтный случай (без script/global).
8. `torso().synchronize(m_skeleton_animated, m_legs)` — sync torso с legs.

`update()` обёрнут в `try/catch`: при ошибке `Msg("! error in stalker with visual %s", ...)`, reset всех пар, return (не rethrow) — «prevent game from crashing».

`play_script()` — `play_script_impl()`: `clear_unsafe_callbacks()`, `global/torso/legs/head().reset()`, `assign_script_animation()` (front `m_script_animations`), `script().animation(...)`, если `use_movement_controller` → `script().target_matrix(selected.transform(object))` + `m_start_new_script_animation` → `destroy_anim_mov_ctrl()`. `script().play(skeleton, script_play_callback, use_movement_controller, local_animation, false, m_script_bone_part_mask)`. Под `USE_HEAD_BONE_PART_FAKE`: head-анимация тоже играет (`head().animation(assign_head_animation()); head().play(...)`).

`play_global()` — `assign_global_animation(animation_movement_controller)`:
- `m_global_selector` (smart cover) → return его результат (см. §4).
- `eMentalStatePanic != mental_state` → `global_critical_hit()` (см. §7).
- `eMentalStatePanic == mental_state` + `fis_zero(speed)` → `MotionID()` (пусто).
- `eMovementTypeRun + eBodyStateStand` → `MotionID()` (пусто, sprint не использует global).
- Иначе `global().select(m_part_animations[body_state].m_global.A[1].A)` (escape/panic).

`play_global_impl(animation, controller)`: `torso/legs().reset()`, `global().animation(...)`, `global().play(skeleton, global_play_callback, controller, true, false, m_script_bone_part_mask, true)`, `m_global_modifier(global().blend())` (для smart cover: speed factor), `head().animation(assign_head_animation()); head().play(...)`.

`play_legs()` — `legs().animation(assign_legs_animation())`. Если animation сменился → `setup_speed_from_animation(speed)` (speed из blend: `speed = (target - previous) * blendAmount + previous`). Это **обратная связь**: анимация задаёт скорость движения (ноги «тянут» тело).

`play_delayed_callbacks()` — `m_call_script_callback`/`m_call_global_callback` — deferred, чтобы не вызывать Lua/global callback во время `UpdateTracks` (который может рекурсивно вызвать bone callbacks).

## 4. `smart_cover::animation_selector` — smart cover анимации

`smart_cover_animation_selector.{h,cpp}`. Устанавливается как `m_global_selector` в `stalker_movement_manager_smart_cover` (порц. 14). `animation_planner` — отдельный `CActionPlannerScript<CAI_Stalker>` («animation planner»), не связанный с `brain()` (порц. 13).

`select_animation(animation_movement_controller)`:
- `animation_movement_controller = true` (smart cover анимации всегда используют movement controller).
- `m_callback_called` (global-анимация закончилась) → `current_operator()->on_animation_end()`, `m_planner->update()`, `current_operator()->on_no_mark()`, `is_animated_action()` → `select_animation(m_animation)`. `cover()->can_fire()` → return `ID_Cycle(m_animation)`.
- Иначе (анимация играет): `blend = global().blend()`, `marks = motion_def->marks` (motion marks), `marks.size() < 3` → `on_no_mark()`, return. `previous_time = m_previous_time`, `time_current = blend->timeCurrent + .1f`, `clamp(previous_time, 0, time_current)`, `marks[2].is_mark_between(previous, current)` → `on_mark()`, return. (Первые 2 marks — footsteps, 3-й — «действие».)

`modify_animation(blend)` — `blend->speed = motion_def->Speed() * g_smart_cover_animation_speed_factor` (1.0, тюнинг).

`initialize()` — `m_planner->initialize()`, `animation().update()` (чтобы bone callbacks работали до `UpdateCL`). `save/load` — сериализация `m_planner`.

## 5. Legs/head/torso-анимации

**Legs** (`stalker_animation_legs.cpp`): `assign_legs_animation()`:
- `standing()` (speed < EPS_L ИЛИ `eMovementTypeStand`) → `legs_no_move_animation()`: `m_previous_speed = m_target_speed = 0`, `m_crouch_state` (из config или `Random.randI(2)` если -1), body yaw vs target yaw → `animation[0/1/2/3]` (idle/turn_right/turn_left по `left_angle`), `eMentalStateFree` → `animation[1/4/5]` (idle/turn по `in_place_names`), `eBodyStateCrouch` → `animation[m_crouch_state]`.
- Иначе → `legs_move_animation()`: `eMentalStateDanger != mental_state` → `m_target_speed = speed(Forward)`, `eMovementTypeRun + eMentalStatePanic` → `m_movement.A[2].A[4].A[0]` (sprint, Alun fix), иначе `m_movement.A[movement_type].A[Forward].A[1]`. `eMentalStateDanger` → `legs_process_direction(yaw)` (yaw из `sight().GetDirectionAngles`, `angle_normalize_signed(-yaw)`): `legs_switch_factor()` (1.0, кроме fwd↔back/left↔right = 0.0), `legs_assign_direction` (переключение `m_current_direction` с задержкой `direction_switch_interval = 500ms`), `movement.m_body.target.yaw = yaw + direction_angles[m_current_direction]`. `speed_direction` по `angle_difference(yaw, body_current)`, `test_angle_forward = PI_DIV_4`, `test_angle_backward = PI - PI_DIV_4`. `m_previous_speed_direction != speed_direction` → `m_change_direction_time = now`, `legs_switch_factor() == 0` → `m_previous_speed = m_target_speed = 0`. `m_target_speed = movement.speed(speed_direction)`, return `m_movement.A[body_state()].A[movement_type].A[speed_direction].A[0]`.

**Head** (`stalker_animation_head.cpp`): `assign_head_animation()` — `m_head_animations.A` = `[head_idle_0, head_talk_0]`. Если `TalkMenu->IsShown()` и `OthersInvOwner() == m_object` и `playing_sound()` → `animations[1]` (talk). Иначе `sound().active_sound_count(true)` (без mask) → 0 → `animations[0]` (idle); если нет `eStalkerSoundMaskMovingInDanger` → `animations[1]` (talk-подобный); иначе `animations[0]`.

**Torso** (`stalker_animation_torso.cpp`): `assign_torso_animation()`:
- `!inventory().ActiveItem()` → `no_object_animation(body_state)`.
- `fill_object_info()` (`m_weapon`/`m_missile` из `ActiveItem`).
- `m_weapon` + `!strapped()` → `weapon_animation(slot, body_state)`; `m_weapon` + `strapped()` → `no_object_animation`.
- `m_missile` → `missile_animation(slot, body_state)`.
- Иначе → `unknown_object_animation(slot, body_state)`.

`weapon_animation` — `CWeapon::GetState()`:
- `eReload` → `animation[4].A[0/1/2]` (begin/in/end).
- `eShowing` → `torso().select(animation[0].A)` (draw).
- `eHiding` → `torso().select(animation[3].A)` (holster).
- `eHidden` → `no_object_animation`.
- `eFire`/`eFire2` → `standing` → `animation[1].A[0]`; walk → `animation[1].A[0]` или `animation[13+looking_back-1].A[1]` (look_back, `slot == 2`); run → `animation[1].A[3]`.
- `eMentalStatePanic + eMovementTypeRun + eBodyStateStand + speed != 0` → `animation[15].A[0]` (escape, Alun sprint fix).
- Иначе → `unknown_object_animation`.

`unknown_object_animation` — по `planner().current_action_state_id()` (ObjectHandlerSpace): `eWorldOperatorFire1/Fire2/Aim1/Aim2/AimingReady1/AimingReady2/QueueWait1/QueueWait2` → `aim_animation(slot, animation, 0/2/3)` (standing/walk/run) или `animation[13+looking_back-1]` (look_back); `eWorldOperatorStrapping/Unstrapping/Strapping2Idle/Unstrapping2Idle` → `animation_stand[11/12].A[0/1]`. Иначе по `mental_state`/`standing`/`movement_type` → `aim_animation` или `animation[9].A[1]` (idle) / `animation[7+movement_type].A[1]` (walk/run).

`aim_animation(slot, animation, index)` — `!m_special_danger_move` → `animation[6].A[index]` (aim). `m_special_danger_move` + `slot != 2` → `animation[6].A[index]`. `slot == 2` (automatic) + `animation[6].A.size() < 7` → `animation[6].A[index]`. Иначе `switch(index)`: 0→`animation[6].A[4]`, 2→`A[5]`, 3→`A[6]` (special danger aim-варианты).

`missile_animation` — `CMissile::GetState()`: `eShowing` → `select(animation[0].A)`; `eHiding` → `select(animation[3].A)`; `eThrowStart` → `animation[1].A[0]`; `eReady`/`eBore` → `animation[1].A[1]`; `eThrow` → `animation[1].A[2]`; `eThrowEnd`/`eHidden` → `animation[6].A[0]`; `eIdle` → `standing` → `animation[6].A[0]`, walk → `A[2]`, run → `A[3]`.

`no_object_animation` — `eMentalStateFree` → `standing` → `animation[9].A[1]` (idle), иначе `animation[7+movement_type].A[1]` (walk/run). Иначе `standing` → `aim_animation(0, animation, 0)`, walk → `aim_animation(0, animation, 2)`, run → `aim_animation(0, animation, 3)`.

`torso_play_callback(blend)` — `pair.on_animation_end()`, если `m_looking_back` → `m_change_direction_time = now + need_look_back_time_interval (2000ms)`, `m_looking_back = 0`.

`need_look_back()` — `m_looking_back` → true; `m_previous_speed_direction != Backward` → false; `m_change_direction_time + 0 > now` → false; иначе `m_looking_back = Random.randI(2) + 1` (1=left, 2=right), true. (Сталкер, идущий назад, поворачивает голову.)

## 6. Bone callbacks (head/shoulder/spine)

`stalker_animation_callbacks.cpp`. Три bone'а: `bone_head`, `bone_shoulder`, `bone_spin` (из ini-секции). `callback_params` — `(Fmatrix const* rotation, CAI_Stalker const* object, CBlend const* const* blend, bool forward)`.

`callback_rotation(bone)` — `bone->mTransform.mulA_43(*parameter->m_rotation)` (применяет current head/shoulder/spine rotation из `sight()`), `position = bone->mTransform.c` (сохраняет позицию), `CWeaponShotEffector& effector = object->weapon_shot_effector()`, `!effector.IsActive()` → `bone->mTransform.c = position`, return. `effector.GetDeltaAngle(angles)`, `angle_normalize_signed` ×3, `movement().current_params().cover()` → `angles.mul(0.f)` (в cover — нет отдачи), иначе `angles.mul(.1f)`, `effector_transform.setXYZ(angles)`, `bone->mTransform.mulA_43(effector_transform)`, `bone->mTransform.c = position`.

`callback_rotation_blend(bone)` — аналогично, но `multiplier = blend->timeCurrent / blend->timeTotal`, `m_forward ? multiplier : (1 - multiplier)`, `Fquaternion result; result.slerp(identity, Fquaternion(rotation), multiplier)` (slerp, не linear), `bone->mTransform.mulA_43(rotation)`.

`assign_bone_callbacks()` — в `reload()`: `m_head_params.m_rotation = &object().sight().current_head_rotation()`, `m_shoulder_params.m_rotation = &current_shoulder_rotation()`, `m_spine_params.m_rotation = &current_spine_rotation()`, `m_blend = 0`, `m_forward = true`, `set_callback(bctCustom, &callback_rotation, &params)`.

`assign_bone_blend_callbacks(forward)` — `m_blend = &global().blend()`, `m_forward = forward_direction`, `set_callback(bctCustom, &callback_rotation_blend, &params)`. Вызывается из `stalker_movement_manager_smart_cover` (порц. 14) при входе/выходе из cover.

`clear_unsafe_callbacks()` — если `m_head_params.m_blend` (т.е. blend callbacks установлены) → `assign_bone_callbacks()` (возврат к «безопасным» rotation callbacks). Вызывается из `play_script_impl`/`play_global_impl` (перед сменой global-анимации, чтобы blend не стал stale).

`remove_bone_callbacks()` — `set_callback(bctCustom, 0, 0)` для всех трёх. `forward_blend_callbacks()`/`backward_blend_callbacks()` — `m_head_params.m_forward`.

## 7. Critical wounds

`stalker_animation_names.h` — `ECriticalWoundType`: `head=4, torso, hand_left, hand_right, leg_left, leg_right, dummy=-1`. `global_names`: `damage_` (0), `escape_` (1), `dead_stop_` (2), `hello_` (3), `1_critical_hit_head/torso/hend_left/hend_right/legs_left/legs_right_` (4–9), `2_*` (10–15), `3_*` (16–21), `panic_stand_` (22).

`CCustomMonster::update_critical_wounded(bone_id, power)` (`CustomMonster.cpp:1110`) — `VERIFY(m_critical_wound_type == -1)`, `m_critical_wound_threshold < 0` → false. `time_delta = (now - m_last_hit_time)/1000.f`, `m_critical_wound_accumulator += power - m_critical_wound_decrease_quant * time_delta`, `clamp(0, threshold)`. `m_last_hit_time = now`. `accumulator < threshold` → false. `accumulator = 0`, `m_last_hit_time = 0`. `critical_wound_external_conditions_suitable()`:
- `body_state() != eBodyStateStand` → false.
- `animation().non_script_need_update()` → false.
- `!inventory().ActiveItem()` (weapon) → false.
- `movement().in_smart_cover()` → false.
- `weapon->animation_slot()` не 1/2/3 → false.
- `!agent_manager().member().registered_in_combat(this)` → false.
- `animation().global().make_inactual()`, true.

`m_bones_body_parts.find(bone_id)` → `m_critical_wound_type = (*I).second`, `critical_wounded_state_start()`.

`critical_wounded_state_start()` (`ai_stalker_fire.cpp:1306`) — `brain().CStalkerPlanner::m_storage.set_property(eWorldPropertyCriticallyWounded, true)`, `animation().global().add_callback(CALLBACK_ID(this, &CAI_Stalker::remove_critical_hit))`.

`remove_critical_hit()` (`:1291`) — `set_property(eWorldPropertyCriticallyWounded, false)`, `animation().global().remove_callback(...)`. (Вызывается из `on_animation_end` глобальной анимации.)

`global_critical_hit()` (`stalker_animation_global.cpp:42`) — `!critically_wounded()` → `MotionID()`. `global().animation()` (уже играет) → return cached. `weapon = ActiveItem()`, `animation_slot` 1–3, `global().select(m_part_animations[eBodyStateStand].m_global.A[critical_wound_type + 6 * (slot - 1)].A, &object().critical_wound_weights())` (веса из ini `critical_wound_weights`, 6 значений на слот).

`critically_wounded()` (`ai_stalker_fire.cpp:1244`) — `critical_wound_type() == dummy` → false; `!brain().storage.property(eWorldPropertyCriticallyWounded)` → `critical_wounded_state_stop()`, false; true.

## 8. Стрельба: dispersion, fire-queue, can_kill

**Dispersion** (`ai_stalker_fire.cpp:75` `GetWeaponAccuracy()`):
- `base = PI/180.f * m_fRankDisperison` (rank-множитель).
- `!movement().path_completed()` (идёт по path):
  - `eMovementTypeWalk` + `eBodyStateStand` → `m_disp_walk_stand`; `eBodyStateCrouch` → `m_disp_walk_crouch`.
  - `eMovementTypeRun` + `eBodyStateStand` → `m_disp_run_stand`; `eBodyStateCrouch` → `m_disp_run_crouch`.
- `path_completed()` (на месте):
  - `eBodyStateStand`: `zoom_state() + hasScope` → `m_disp_stand_stand_zoom`, иначе `m_disp_stand_stand`.
  - `eBodyStateCrouch`: `zoom_state() + hasScope` → `m_disp_stand_crouch_zoom`, иначе `m_disp_stand_crouch`.
  - Иначе → `m_disp_run_stand` (worst, fallback).
- Формула: `base *= (m_disp_* * g_dispersion_factor + g_dispersion_base)` (1.0/1.0, тюнинг).
- Lua: `_g.CAI_Stalker__GetWeaponAccuracy(lua_obj, weapon, base, body_state, movement_type)` → override.

**`g_fireParams(pHudItem, P, D)`** (`:145`) — точка/направление выстрела:
- `!ActiveItem` → `P = Position()`, `D = (0,0,1)`.
- `CMissile` → `update_throw_params()`, `P = m_throw_position`, `D = m_throw_velocity.normalize()`.
- `!CWeapon` (еда и т.п.) → `P = eye_matrix.c`, `D = eye_matrix.k`, `weapon_shot_effector().IsActive()` → `D = weapon_shot_effector_direction(D)`.
- `!g_Alive()` → `P = weapon->get_LastFP()`, `D = weapon->get_LastFD()`.
- `animation().script_animations()` непусты ИЛИ `animation().global_selector()` (smart cover) → `P = weapon->get_LastFP()`, `sniper_fire_mode()` → `D.setHP(-head.target.yaw, -head.target.pitch)`, иначе `D = weapon->get_LastFD()`.
- `eBodyStateStand` + `eMovementTypeStand` → `P = eye_matrix.c`, `D = eye_matrix.k`, effector → `D = weapon_shot_effector_direction(D)`, `sniper_fire_mode()` → `D.setHP(-head.target.yaw, -head.target.pitch)`.
- `eBodyStateStand` + движение → `D.setHP(-head.current.yaw, -head.current.pitch)`, effector, `P = Center() + D * 0.5f`, `P.y += 0.50f`, `sniper_fire_mode()` → `D.setHP(-head.target.yaw, -head.target.pitch)`.
- `eBodyStateCrouch` → `P = eye_matrix.c`, `D = eye_matrix.k`, effector, `sniper_fire_mode()` → `D.setHP(-head.target.yaw, -head.target.pitch)`.

**`g_WeaponBones(L, R1, R2)`** (`:254`) — `CObjectHandler::weapon_bones(r_hand, r_finger2, l_finger1)`, `R1 = r_hand`, `R2 = r_finger2`, `!script_animations.empty() && front().hand_usage()` → `L = R2` (левая рука на ружьё), иначе `L = l_finger1`.

**Fire-queue** (`stalker_combat_action_base.cpp:55` `fire()`):
- `direction = enemy_position - object_position`, `angle_difference(-yaw, head.current.yaw) > g_ai_aim_fire_angle (PI_DIV_8)` → `aim_ready()`, return (не стрелять, если враг не в FOV).
- `distance = enemy.distance_to(object)`, `select_queue_params(distance, min_size, max_size, min_interval, max_interval)`.
- `CObjectHandler::set_goal(eObjectActionFire1, best_weapon(), min_size, max_size, min_interval, max_interval)`.

`select_queue_params` (`:94`) — по `ef_weapon_type()`: 5 (pistol) → `pstl_*`, 9 (shotgun) → `shtg_*`, 11/12 (sniper) → `snp_*`, 6/7/8/10 (machine gun) → `mchg_*`, иначе → `auto_*`. По distance: `> queue_fire_dist_far` → `_far`, `> _med` → `_medium`, иначе → `_close`. Per-NPC scale: `fire_queue_size_k()`/`fire_queue_interval_k()` (из `set_fire_queue_scale`, default 1.0) → `min_size = _max(1, round(min_size * size_k))`, `max_size = _max(min_size, round(max_size * size_k))`, `min_interval = round(min_interval * interval_k)`, `max_interval = _max(min_interval, round(max_interval * interval_k))`.

`aim_ready()` (`:76`) — `set_goal(eObjectActionAimReady1, best_weapon(), ...)`. `aim_ready_force_full()` (`:85`) — `set_goal(eObjectActionAimForceFull1, ...)`.

**`CObjectHandlerPlanner::set_goal`** (`object_handler_planner.cpp:53`):
- `object_property(object_action)` → `EWorldProperties` (Fire1 → `eWorldPropertyFiring1`, AimReady1 → `eWorldPropertyAimingReady1`, AimForceFull1 → `eWorldPropertyAimForceFull1`, и т.д.).
- `condition_id = uid(game_object->ID(), goal)` (16+16 bits).
- `CState condition; condition.add_condition(CWorldProperty(condition_id, true)); set_target_state(condition)` (action-planner target).
- Queue-параметры: если изменились или `m_next_time_change <= now` → `m_queue_size = Random.randI(min, max)` (или `min` если равны), `m_queue_interval = Random.randI(min, max)`, `m_next_time_change = now + m_queue_interval`, `weapon->SetQueueSize(m_queue_size)`, `action(uid(weapon_id, eWorldOperatorQueueWait1/2)).set_inertia_time(m_queue_interval ? m_queue_interval : 300)`.

**`fire_make_sense()`** (`ai_stalker_fire.cpp:894`):
- `!memory().enemy().selected()` → false.
- `(pick_distance() + g_ai_fire_range_extension (2.5)) < distance_to(enemy)` → false.
- `|Position.y - enemy.y| > g_ai_fire_max_height_diff (2.0)` → false.
- `pick_distance() < g_ai_fire_min_dist (2.5)` → false.
- `memory().visual().visible_right_now(enemy)` → true.
- `last_time_seen == -1` → false.
- `now > last_time_seen + g_ai_fire_make_sense_interval (10000ms)` → false.
- `!best_weapon()` → false.
- `best_weapon()->ef_weapon_type()` не 6/7/8/10 (machine gun) → false (только auto-оружие стреляет «по последнему виду»).
- true.

**`can_kill_entity`** (`:737`) — raycast из `g_fireParams` в `distance`, `ray_query_callback` (`:698`): `feel_vision_mtl_transp` (прозрачность), `!result.O` (пусто) → `m_power > threshold` → true (остановиться), иначе `m_pick_distance = range`, false. `CEntityAlive*` → `is_relation_enemy` → `m_can_kill_enemy = true`, иначе `m_can_kill_member = true`, `m_pick_distance = range`, false.

`can_kill_entity_from(position, direction, distance)` (`:753`) — 5 raycast'ов: центр, `pitch - PI_DIV_8 * 0.125` (down), `pitch + ...` (up), `yaw - ...` (left), `yaw + ...` (right). После каждого: `m_can_kill_member && m_can_kill_enemy` → return; `m_can_kill_member` (после pitch-проверок) → return (не стрелять, если союзник на пути); `m_can_kill_enemy` сохраняется.

`pick_distance()` (`:802`) — `!ActiveItem` → `start_pick_distance()` (`_max(50.f, distance_to(enemy) + 1.f)`); иначе `update_can_kill_info()` (раз в кадр, `m_pick_frame_id == Device.dwFrame` → skip), return `m_pick_distance`.

`can_kill_member()`/`can_kill_enemy()` (`:1404/1413`) — `update_can_kill_info()`, return `m_can_kill_member`/`m_can_kill_enemy`.

**Throw** (`ai_stalker_fire.cpp:1065`): `missile_throw_force()` — `update_throw_params()`, `m_throw_velocity.magnitude()`, Lua `_G.CAI_Stalker__GetMissileThrowForce(lua_obj, force)` → override (должен быть > 0).

`update_throw_params()` (`:1167`): `m_throw_actual` + `Position`/`Direction` не изменились → return. `m_throw_position = Position()`, `CMissile` → `third_person_throw_point_offset` из ini (или `y += 2.f` + `throw_point_offset()`). `distances[] = {30, 40, 50, 60}` по `g_SingleGameDifficulty`, `max_distance = distances[difficulty]`. `m_throw_velocity = m_throw_target_position - m_throw_position`, `magnitude > max_distance` → `m_throw_enabled = false`, return. `ThrowMinVelTime(velocity, gravity)`, `TransferenceToThrowVel(velocity, time, gravity)`, `check_throw_trajectory(time)` (raycast траектории, `m_throw_enabled = true` если нет collisions), `m_throw_velocity.mul(Random.randF(0.75, 1.25))` (рандом ±25%).

`compute_throw_miss(vertex_id)` (`:1085`) — `throw_miss_radius = Random.randF(2, 5)`, 6 попыток: `random_dir()`, `check_position = m_throw_target_position + direction * radius`, `level_graph.check_position_in_direction(vertex_id, target, check)`, valid → `m_throw_target_position = check_position`, return. (Пропуск точного попадания в гранате.)

**`Hit(SHit* pHDS)`** (`:266`) — получение урона:
- `HDS.add_wound = true`, `hit_power = HDS.power * m_fRankImmunity` (rank-иммунитет).
- `m_boneHitProtection && eHitTypeFireWound` → `BoneArmor = getBoneArmor(bone)`, `ap = HDS.armor_piercing`:
  - `ap > BoneArmor` → `d_hit_power = (ap - BoneArmor) / (ap * APScale)`, `clamp(d_hit_power, m_fHitFracNpc, 1.0)`, `hit_power *= d_hit_power`.
  - `ap <= BoneArmor` → `hit_power *= m_fHitFracNpc`.
  - `wounded()` (уже лежит) → `hit_power = 1000.f` (добивание).
- `g_Alive()`:
  - `!already_critically_wounded` + `!invulnerable()` + `cover` + `HDS.initiator() != ID()` + `!fis_zero(damage)` + `brain().affect_cover()` → `agent_manager().location().add(xr_new<CDangerCoverLocation>(cover, now, g_ai_cover_danger_time (120000ms), g_ai_cover_danger_radius (3.f)))` (покрытие «опасно», другие не сядут).
  - `is_relation_enemy(initiator)` + `!wounded()` → `sound().play(eStalkerSoundInjuring)`.
  - `!wounded()` + `!already_critically_wounded` → `update_critical_wounded(boneID, power)`:
    - `!became_critically_wounded` + `!script_animations.empty()` + `bone != BI_NONE` → `power_factor = m_power_fx_factor * damage / 100.f`, `clamp(0, 1)` (FX, но закомментирован).
    - `became_critically_wounded` → `stalker = smart_cast<CAI_Stalker*>(HDS.who)`, `stalker->on_critical_wound_initiator(this)` (звук «враг ранен»), Lua `_G.CAI_Stalker__OnCriticallyWounded(lua_obj, hit, boneID)`.
- `g_Alive()` + `(!m_hit_callback || m_hit_callback(&HDS))` → Lua `_G.CAI_Stalker__BeforeHitCallback` (может вернуть false → skip), `HDS.ApplyScriptHit(&tLuaHit)`, `damage_factor = invulnerable() ? 0 : 100.f`, `memory().hit().add(damage_factor * damage, direction, who, boneID)`.
- `inherited::Hit(&HDS)`.

**`wounded(value)`** (`:1006`) — `notify_on_wounded_or_killed()` (последний `memory().hit().last_hit_object_id()` → `on_enemy_wounded_or_killed`), `m_wounded = value`, `!m_wounded` → `CreateCharacterSafe()` (восстановление физики), `m_wounded` → `DestroyCharacter()` (убор физики), `!registered_in_combat` → skip, иначе `unregister_in_combat(this)`.

**`on_weapon_shot_start/update/stop`** (`:940/952/956`) — `weapon_shot_effector().SetRndSeed(m_weapon_shot_random_seed)`, `Shot(weapon)`, Lua `_G.CAI_Stalker__OnWeaponShotStart/Stop`. `on_weapon_shot_update` — пусто.

## 9. Торговля и выбор оружия

**`CTradeItem`** (`ai_stalker_alife.cpp:32`) — `(CInventoryItem*, u16 m_old_owner_id, u16 m_new_owner_id)`. `operator<` по `ID()`, `operator==` по `ID()`.

**`tradable_item(item, current_owner_id)`** (`:42`) — `!useful_for_NPC()` → false; `PDA` + `GetOriginalOwnerID() == current_owner_id` → false; `trade_parameters().enabled(action_sell(0), cNameSect())` → true/false.

**`fill_items(inventory, old_owner, new_owner_id)`** (`:63`) — для каждого `m_all`: `tradable_item` → `m_temp_items.push_back(CTradeItem(item, old_owner->ID(), new_owner_id))`, `result += Cost()`.

**`update_sell_info()`** (`:268`) — `m_sell_info_actuality = true`, `m_temp_items.clear()`, `m_current_trader = 0`, `m_total_money = get_money()`, `money_delta = fill_items(inventory(), this, -1)`, `m_total_money += money_delta`, `sort(m_temp_items)`, `select_items()`, для каждого `!tradable_item` → `m_temp_items.push_back(CTradeItem(item, ID(), ID()))` (нельзя продать → owner = self).

**`can_sell(item)`** (`:291`) — `is_trader` (ini) → `tradable_item(item, ID())`; иначе `update_sell_info()`, `find(m_temp_items, item->ID())`, `m_new_owner_id != ID()` (если owner = self → нельзя продать).

**`select_items()`** (`:253`) — `!m_can_select_items` → return. `choose_food()` (пусто — «cannot change food»), `choose_weapon(Knife/Secondary/Primary/Grenade)`, `choose_medikit()` (пусто), `choose_detector()`, `choose_equipment()` (пусто).

**`choose_weapon(weapon_priority_type)`** (`:139`) — для каждого `m_temp_items`: `m_total_money < Cost()` → skip; `m_pfPersonalWeaponType->dwfGetWeaponType()` по типу: Knife (1) → `m_pfItemValue`, Secondary (5) → `m_pfSmallWeaponValue`, Primary (6/7/8/9/11/12) → `m_pfMainWeaponValue`, Grenade (4) → `m_pfItemValue`. `eItemPlaceSlot` → `+10.0f` (бонус за уже в слоте). `current_value > best_value` → `best_weapon = &*I`. `best_weapon` → `buy_item_virtual(*best_weapon)`, `attach_available_ammo(weapon)`.

**`buy_item_virtual(item)`** (`:94`) — `item.m_new_owner_id = ID()`, `m_total_money -= Cost()`, `m_current_trader->set_money(get_money() + Cost(), true)`.

**`attach_available_ammo(weapon)`** (`:108`) — `!weapon || m_ammoTypes.empty()` → return. Для каждого `m_temp_items`: `m_total_money < Cost()` → skip; `!find(m_ammoTypes, cNameSect)` → skip; `buy_item_virtual`, `++count`, `count >= MAX_AMMO_ATTACH_COUNT (1)` → break.

**`choose_detector()`** (`:217`) — для каждого: `m_total_money < Cost()` → skip; `smart_cast<CCustomDetector>` → `m_pfDetectorType->ffGetValue()`, `current > best` → `best_detector`. `buy_item_virtual`.

**`conflicted(item, new_weapon, new_weapon_enough_ammo, new_weapon_rank)`** (`:342`) — `non_conflicted` (разный `ef_weapon_type` или `item == new_weapon`) → false (не конфликт). `enough_ammo(current)` + `!enough_ammo(new)` → true (не менять, у текущего есть патроны). `!enough_ammo(current)` + `enough_ammo(new)` → false (менять). `!fsimilar(Condition, 0.05)` → `current >= new` (менять только если новый лучше на 5%). `ef_weapon_type` разный → `current.Cost() >= new.Cost()`. `get_rank(current) != new_rank` → `current_rank >= new_rank`. true.

**`can_take(item)`** (`:372`) — `!new_weapon` → false. `new_weapon_enough_ammo = enough_ammo(new)`, `new_weapon_rank = get_rank(new)`. Для каждого в инвентаре: `conflicted` → false (не брать). true.

**`update_conflicted(item, new_weapon)`** (`:434`) — `non_conflicted` → return. `remove_personal_only_ammo(item)` (удалить патроны, которые нужны только для этого оружия, если другое оружие их не использует), `item->SetDropManual(TRUE)` (бросить).

**`on_after_take(object)`** (`:443`) — `!g_Alive()` → return. `!use_single_item_rule` (ini, default true) → return. `!new_weapon` → return. Для каждого: `update_conflicted`.

**`best item to kill`** (`ai_stalker_fire.cpp:462` `update_best_item_info`):
- `m_bTrading || !m_can_select_weapon` → return.
- Lua `_g.update_best_weapon(lua_obj, current)` → override.
- `ai().ef_storage().alife_evaluation(false)`.
- `m_item_actuality = true`, `member = this`, `enemy = memory().enemy().selected() ?: this`.
- Для каждого `m_all`: `can_kill()` → `member_item = item`, `value = enemy ? m_pfWeaponEffectiveness->ffGetValue() : Cost()`. `value > m_best_item_value` → `m_best_item_to_kill = item`. `fsimilar` + `Cost() <= m_best_item_to_kill->Cost()` → skip (дешевле).
- `m_best_item_to_kill` → `m_best_ammo = m_best_item_to_kill`, return.
- Для каждого `memory().item().objects()`: `!useful` → skip. `can_kill(&inventory())` → `member_item = item`, `value = m_pfWeaponEffectiveness`, `value > m_best_item_value` → `m_best_found_item_to_kill = item`, `m_best_ammo = item`. Иначе `can_make_killing(&inventory())` → `m_best_found_ammo = item`.
- `m_best_found_item_to_kill || m_best_found_ammo` → return.
- Для каждого `memory().item().objects()`: `can_kill(objects)` → `m_best_found_item_to_kill = item`, `m_best_ammo = item`.

`item_to_kill()`/`item_can_kill()`/`remember_item_to_kill()`/`remember_ammo()` — `update_best_item_info()`, return `!!m_best_*`. `ready_to_kill()` — `m_best_item_to_kill && ActiveItem() && ID() == m_best_item_to_kill->ID() && m_best_item_to_kill->ready_to_kill()`. `ready_to_detour()` — `ready_to_kill()` + `CWeapon` + `GetAmmoElapsed() > GetAmmoMagSize() * g_ai_reload_threshold (0.5f)` (мало патронов → detour за патронами).

**`OnItemTake/OnItemDrop`** (`ai_stalker_fire.cpp:434/441`) — `m_item_actuality = false`, `m_sell_info_actuality = false`. `OnItemDrop` + `!g_Alive()` → return; `!critically_wounded()` → return; `ActiveItem() != item` → return; `brain().storage.set_property(eWorldPropertyCriticallyWounded, false)` (бросил оружие → не ранен).

## Связи

```mermaid
graph TD
  S[CAI_Stalker] --> AM[CStalkerAnimationManager "animation()"]
  AM --> GP[CStalkerAnimationPair "global/head/torso/legs/script"]
  GP --> SK[IKinematicsAnimated "PlayCycle / LL_PlayCycle"]
  GP --> CB[bone callbacks "callback_rotation / blend"]
  CB --> SI[CSightManager "sight() — порц. 14 §8"]
  CB --> SE[CWeaponShotEffector "recoil"]
  AM --> DS[CStalkerAnimationDataStorage "кэш по visual"]
  AM --> SMV[smart_cover::animation_selector "m_global_selector — порц. 14"]
  SMV --> AP[animation_planner "CActionPlannerScript"]
  S --> FI[ai_stalker_fire.cpp]
  FI --> DH[CObjectHandler::set_goal "порц. 26"]
  DH --> OH[CObjectHandlerPlanner "action-planner"]
  OH --> WA[object_actions "Fire1/AimReady1/QueueWait1"]
  FI --> MW[CCustomMonster::update_critical_wounded]
  S --> TR[ai_stalker_alife.cpp "торговля"]
  TR --> EF[ef_storage "m_pfWeaponEffectiveness"]
  TR --> TP[trade_parameters "action_sell/buy"]
  P13["порц. 13 combat actions"] --> FI
  P13 --> TR
```

- Planner'ы (порц. 13) — `CStalkerActionCombatBase::fire/aim_ready` → `set_goal` (здесь); `can_fire_right_now`/`aiming_position` (порц. 14).
- `CObjectHandler::update` — исполнение целей (show/hide/reload/fire/throw) — порц. 26; здесь — `set_goal` + queue-параметры.
- `CSightManager` (порц. 14 §8) — `current_head/shoulder/spine_rotation` для bone callbacks; `sight().GetDirectionAngles` для legs.
- `CWeaponShotEffector` — recoil, `GetDeltaAngle` для bone callbacks + `g_fireParams`.
- `ef_storage`/`trade_parameters`/`agent_manager` — порц. 22–23; `CSE_ALife*` — ит. 11/12.

## Замечания к плану

1. **5 «пар» вместо 5 классов** — TODO в `stalker_animation_manager.cpp:18` прямо говорит «should be represented with the different classes», но реализовано как один класс с 5 полями `CStalkerAnimationPair`. Это технический долг, не баг.
2. **`USE_HEAD_BONE_PART_FAKE`** — global/script-анимации играют на всех bone parts кроме head (маска), чтобы head-анимация не конфликтовала с global. Без этого define — head reset'ится вместе с global (старое поведение).
3. **`select()` кэширует `m_array`** — если `select()` вызывается с тем же `array` на следующий кадр, return cached `animation()` (не пере-рандомить). Это важно для `global_critical_hit()` (6 анимаций на слот, 1 выбрана и держится).
4. **`play_legs` задаёт скорость** — `setup_speed_from_animation(speed)`: анимация ног «тянет» тело. `speed = (target - previous) * blendAmount + previous` — плавный переход. Это обратная связь: planner ставит `target_speed`, анимация её «сглаживает».
5. **`fire_make_sense` — только auto-оружие** — `ef_weapon_type` 6/7/8/10 (machine gun). Пистолеты/дробовики/снайперки не стреляют «по последнему виду», только видя врага.
6. **`g_ai_fire_make_sense_interval = 10000ms`** — после потери вида врага можно стрелять ещё 10 секунд (по последнему известному положению).
7. **`weapon_shot_effector_direction` закомментирован** (`#if 1 return current`) — отдача не влияет на направление выстрела (только на bone callbacks для визуала). `#else` — активная реализация.
8. **`+10.0f` за `eItemPlaceSlot`** в `choose_weapon` — бонус за оружие, уже в слоте (не нужно доставать). Это делает выбор оружия «липким» (не меняется, пока не найдется значительно лучше).
9. **`use_single_item_rule` (default true)** — один предмет на слот: при получении нового оружия старое того же типа бросается (`SetDropManual(TRUE)`), патроны, которые нужны только старому оружию, удаляются.
10. **Critical wounds — только в бою, stand, не в smart cover** — `critical_wound_external_conditions_suitable()`: `registered_in_combat`, `eBodyStateStand`, `!in_smart_cover`, weapon slot 1–3. Иначе `accumulator` копится, но `critical_wound_type` не ставится (до следующего попадания, когда условия подойдут).
11. **`CDangerCoverLocation`** — при получении урона в cover, cover помечается «опасным» на 120 секунд в радиусе 3 м (другие сталкеры не сядут в это покрытие, если враг стреляет оттуда).
12. **`m_fRankImmunity`/`m_fRankDisperison`** — rank-множители: новички получают больше урона и хуже целятся. `m_boneHitProtection` (SBoneProtections) — броня по костям, `armor_piercing` vs `BoneArmor`.
13. **`g_dispersion_base = 1.0`, `g_dispersion_factor = 1.0`** — глобальные cvar-подобные константы для тюнинга dispersion. Формула: `base *= (m_disp * factor + base)`.
14. **`fire_queue_size_k`/`fire_queue_interval_k`** — per-NPC scale (Lua `set_fire_queue_scale`), default 1.0. Применяется к band, выбранному по weapon type + distance (не заменяет, а масштабирует).
15. **`compute_throw_miss`** — гранаты не попадают точно: 6 попыток сместить цель на 2–5 м в случайном направлении (valid vertex). Это ванильное поведение (гранаты «разбрасываются»).
16. **`animation().update()` из `smart_cover_animation_selector::initialize`** — вызывается до `UpdateCL` (для RayQuery), чтобы bone callbacks работали. Без этого `VERIFY(*parameter->m_blend)` упало бы (blend не создан).

## Краткое резюме

`CStalkerAnimationManager` — 5 «пар» анимаций (global/head/torso/legs/script) на `CStalkerAnimationPair` (`MotionID`+`CBlend`+callbacks), данные в `CStalkerAnimationDataStorage` (кэш по visual slots), update-цикл: `play_script` → `play_global` → `play_head/torso/legs` + `synchronize`, bone callbacks (head/shoulder/spine: `callback_rotation`/`callback_rotation_blend` с `CSightManager` + `CWeaponShotEffector`), `smart_cover::animation_selector` (отдельный `animation_planner` на motion marks). Стрельба: `GetWeaponAccuracy` (8 dispersion-параметров + rank + Lua), `g_fireParams` (точка/направление выстрела по body_state + sniper + effector), fire-queue (`select_queue_params` по weapon type + distance + per-NPC scale → `CObjectHandler::set_goal`), `fire_make_sense` (range/height/visibility/auto-only), `can_kill_entity` (5 raycast'ов), throw (`update_throw_params` + `compute_throw_miss`), `Hit` (rank immunity + bone armor + AP + critical wounds + `CDangerCoverLocation`), `update_best_item_info` (ef_storage weapon effectiveness). Торговля: `CTradeItem` + `fill_items`/`select_items`/`choose_weapon`/`choose_detector`/`can_sell`/`conflicted`/`can_take`/`update_conflicted`/`on_after_take` (single item rule).

Порцион 15 (итерация 6). Далее: порц. 16 — `monsters-animals.md` — `CAI_Dog`/`CAI_PseudoDog`/`CCat`/`CAI_Boar`/`CTushkano`/`CZombie` ([monsters-animals.md](monsters-animals.md)).
