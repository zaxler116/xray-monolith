# CActor: анимация (порцион 31)

> **Модуль:** `xrGame` · **Файлы:** `ActorAnimation.cpp` (783), `ActorAnimation.h` (47), `actor_anim_defs.h` (104), `Actor.h` (295–335, 678), `Actor.cpp` (218, 2821–2825), `Actor_Network.cpp` (818–871), `step_manager.cpp` (117) · **Порцион 31, итерация 5**
>
> **Страница:** анимация актёра: выбор анимаций по `mstate` (ноги/торс/голова), костные колбэки ориентации, инициализация `SActorMotions`, транспортные анимации. Связи: `animation_movement_controlled`/`create/destroy_anim_mov_ctrl` (порц. 3), `select_animation` монстров/сталкеров (порц. 9, ит. 6), `mstate_rl`/`isActorAccelerated` (порц. 23), `CStepManager` (шаги), `player_hud` (HUD-руки), `CScriptEntityAction` (порц. 44), `OnChangeVisual`/`net_Spawn` (порц. 28), `steer_Vehicle` (порц. 26).

## 1. Ответственность

Каждый кадр преобразует битовую маску движения `mstate_rl` в три `MotionID` — ноги, торс, голова — и запускает их (`g_SetAnimation`, §7). Плюс: костные колбэки поворота корпуса/головы (§5), выбор спринт-анимаций (§6), инициализация наборов анимаций из листинга модели (§4), анимации руля транспорта (§8). Точка входа — `CActor::UpdateAnimation()` → `g_SetAnimation(mstate_real)` (`Actor.h:678`, вызывается базовым `Update`-циклом).

## 2. Макросы состояний (`ActorAnimation.h:7–45`)

Алиасы комбинаций `mc*` из `object_flags`/`mstate`: базовые (`_Fwd`, `_Back`, `_LStr`, `_RStr` + 4 диагонали), с ускорением (`_AFwd` … `_ABackRStr` — `mcAccel|...`), присед (`_Crch*` — `mcCrouch|...`, включая `_ACrchFwdLStr`-комбинации), `_Jump` (`mcJump`). Используются при разборе `mstate_rl` и в `#ifdef DEBUG`-выводе.

## 3. Структуры данных (`actor_anim_defs.h`)

| Структура | Строки | Содержимое |
|---|---|---|
| `SAnimState` | 5–12 | 4 `MotionID` ног: `legs_fwd/back/ls/rs`; `Create(K, base0, base1)` |
| `STorsoWpn` | 14–39 | `moving[eTotal]` (eIdle/eWalk/eRun/eSprint), `zoom/holster/draw/drop/reload{,_1,_2}/attack/attack_zoom/fire_idle/fire_end`, `all_attack_0..2` (атака всего тела из стойки), `safemode`; 13 слотов под типы оружия |
| `_total_anim_slots_` | 41 | `13` — размер `m_torso[]` |
| `SActorState` | 43–60 | `legs_idle/jump_begin/jump_idle/landing[2]/legs_turn/death`, `m_walk`/`m_run` (`SAnimState`), `m_torso[13]`, `m_torso_idle`, `m_head_idle`, `m_damage[DAMAGE_FX_COUNT]`; `Create(K, base)`, `CreateClimb(K)` |
| `SActorSprintState` | 62–74 | спринт-ноги + jump-варианты (`legs_jump_*`) |
| `SActorMotions` | 76–84 | `m_dead_stop`, `m_normal` (base `"norm"`), `m_crouch` (`"cr"`), `m_climb`, `m_sprint` |
| `SVehicleAnimCollection` | 87–96 | `MAX_IDLES=3`, `idles[]`, `steer_left/right`; `Create(K, num)` |
| `SActorVehicleAnims` | 98–104 | `TYPES_NUMBER=2` коллекции |

**Поля актёра** (`Actor.h:310–320`): `SActorMotions* m_anims`, `SActorVehicleAnims* m_vehicle_anims` (`#ifdef ENABLE_CAR`), `m_current_legs_blend/m_current_torso_blend/m_current_jump_blend` (`CBlend*`), `m_current_legs/torso/head` (`MotionID`), `m_bAnimTorsoPlayed` (294).

## 4. Инициализация и колбэки костей

- **Ctor:** `m_anims = xr_new<SActorMotions>()` (`Actor.cpp:218`), `m_vehicle_anims = xr_new<SActorVehicleAnims>()` (221), удаление — 284/`xr_delete`.
- **`OnChangeVisual()`** (`Actor_Network.cpp:846–871`): при смене модели/визуала — `CStepManager::reload`, `SetCallbacks()`, **`m_anims->Create(V)`**, **`m_vehicle_anims->Create(V)`**, `CDamageManager::reload`, кеширование ID костей (`m_head`, `m_eye_left/right`, `m_r_hand`, `m_l_finger1`, `m_r_finger2` из `weapon_bone0/1/2`). Вызывается и из `net_Spawn`-цепочки (порц. 28).
- **`SetCallbacks()` / `ResetCallbacks()`** (`Actor_Network.cpp:818–844`): `set_callback(bctCustom, ...)` на `bip01_spine` → `Spin0Callback`, `bip01_spine1` → `Spin1Callback`, `bip01_spine2` → `ShoulderCallback`, `bip01_head` → `HeadCallback`; `ResetCallbacks` — `reset_callback()` на те же 4 кости.

### `Create`-функции (разбор имён анимаций)

- `SActorMotions::Create` (`ActorAnimation.cpp:245–254`): `m_dead_stop = "norm_dead_stop_0"`, `m_normal.Create("norm")`, `m_crouch.Create("cr")`, `m_climb.CreateClimb()`, `m_sprint.Create()`.
- `SActorState::Create` (198–231): `<base>_turn`, `<base>_idle_0`, `<base>_death_0`, walk/run, торс `<base>_1.._13`, `<base>_torso_0_aim_0`, `head_idle_0`, jump/landing, 12 × `<base>_damage_N` (`ID_FX`).
- `SActorState::CreateClimb` (156–195): для climb — base `"cl"` (idle/torso/walk/run), далее base `"norm"` (turn/death/торса/jump/landing/damage); `m_head_idle.invalidate()` (закомментирован `ID_Cycle("head_idle_0")`).
- `SAnimState::Create` (146–153): `<base0><base1>_fwd_0/back_0/ls_0/rs_0`.
- `STorsoWpn::Create` (122–144): `<base0>_torso<base1>_aim_1/2/3` (idle/walk/run), `_escape_0` (sprint), `_aim_0` (zoom), `_holster_0`, `_draw_0`, `_reload_0/1/2`, `_drop_0`, `_attack_1/0/1/2` (attack/attack_zoom/fire_idle/fire_end), `<base0>_all<base1>_attack_0/1/2`, `_idle_1` (safemode). `ID_Cycle_Safe` — отсутствующие анимации дают невалидный `MotionID`.
- `SActorSprintState::Create` (233–243): `norm_escape_00/ls_00/rs_00` + `*_jump_00`-варианты.
- Vehicle (265–286): `steering_idle_<num>_ls/rs` (руль), `steering_idle_<num><i>` (idles, подсчёт `idles_num`).

## 5. Костные колбэки ориентации (`ActorAnimation.cpp:26–120`)

Все пять — статические, получают `CActor*` из `B->callback_param()`; строят матрицу поворота из `r_torso`/`r_model_yaw` и **сохраняют позицию кости** (`B->mTransform.c = c`), т.е. вращают только без трансляции.

| Колбэк | Факторы yaw/pitch/roll | Особенность |
|---|---|---|
| `Spin0Callback` (42–55) | 0.0 / 0.0 / 0.3 | только roll |
| `Spin1Callback` (57–70) | 0.4 / 0.2 / 0.3 | |
| `ShoulderCallback` (72–84) | 0.4 / 0.7 / 0.2 | основной поворот корпуса |
| `HeadCallback` (88–106) | 0.2 / 0.1 / 0.2 | **при `cam_freelook != eflDisabled`** yaw берётся из `-cam_FirstEye()->yaw` × 0.35 (95–98) |
| `VehicleHeadCallback` (108–120) | 0.75 / 0.75 / 0.2 | `spin.setHPB`, roll со знаком минус |

yaw-компоненты: `angle_normalize_signed(r_torso.yaw - r_model_yaw - r_model_yaw_delta)` (у головы при freelook — иначе без торса). Факторы — файловые `static const float` (26–37).

**Анимационные колбэки:**
- `legs_play_callback(CBlend*)` (306–311) — по окончании ног: `object->m_current_legs.invalidate()`.
- `CActor::AnimTorsoPlayCallBack(CBlend* B)` (`Actor.cpp:2821–2825`, декларация `Actor.h:295`) — `actor->m_bAnimTorsoPlayed = FALSE` — сброс «торс занят одноразовой анимацией» (drop/бросок/кинжал).

## 6. `g_SetSprintAnimation` (`ActorAnimation.cpp:313–326`)

`jump = mcFall | mcLanding | mcLanding2 | mcJump`; по направлению (`mcFwd`/`mcLStrafe`/`mcRStrafe`) выбирает `sprint.legs_fwd/ls/rs` либо `legs_jump_*` при прыжке. `head/torso` не заполняются (только `legs`). Вызывается из `g_SetAnimation` при `mcSprint` (414–418).

## 7. `g_SetAnimation(u32 mstate_rl)` (`ActorAnimation.cpp:344–783`)

Главный селектор. Вызывается каждый кадр через `UpdateAnimation()` (`Actor.h:678`).

### 7.1 Мёртвый актёр (346–361)

При `!g_Alive()`: если были текущие анимации — выбор `SActorState` (crouch/normal) сбрасывается, `mstate_real = 0`, `m_current_legs/torso.invalidate()`; выход (без запуска `m_dead_stop` — строка закомментирована, 357).

### 7.2 Выбор набора (362–388)

- `ST`: `mcCrouch` → `m_crouch`; `mcClimb` → `m_climb`; иначе `m_normal`.
- `bAccelerated = isActorAccelerated(mstate_rl, IsZoomAimingMode())` (порц. 23) → `AS = m_run / m_walk`; `moving_idx = eRun/eWalk` при `mcAnyMove`, иначе `eIdle`.

### 7.3 Ноги (397–418)

Приоритет: `mcLanding` → `landing[0]`; `mcLanding2` → `landing[1]`; `mcTurn && !mcClimb` → `legs_turn`; `mcFall` → `jump_idle`; `mcJump` → `jump_begin`; далее `mcFwd/Back/LStrafe/RStrafe` → `AS->legs_*`; иначе `is_standing = true`. **Исключение** (410–412): если `g_player_hud && is_standing && IsFocused() && m_legs_controller.is_active()` — `is_standing = false` (IK-контроль ног задаёт анимацию сам). Затем `mcSprint` → `g_SetSprintAnimation` + `moving_idx = eSprint` (414–418).

### 7.4 Уведомления HUD-рук (420–440)

Только при `this == Level().CurrentViewEntity()`: при переходах `mcSprint`/`mcCrouch`/`mcAccel` (без зума) → `g_player_hud->OnMovementChanged(...)`; отдельно — переход `mcAnyMove` вкл/выкл.

### 7.5 Торс (444–642)

- **Climb** (444–450): `M_torso` = те же `AS->legs_*` (торс следует за направлением).
- Иначе: `CHudItem* H` из `inventory().ActiveItem()`, слот `ST->m_torso[H->animation_slot() - 1]` (VERIFY ≤ 13) (454–460).
- **Drop** (461–470): `!b_DropActivated && !fis_zero(f_DropPower)` → `TW->drop` (фолбэк `m_torso_idle` + Msg при отсутствии), `m_bAnimTorsoPlayed = TRUE`.
- **`!m_bAnimTorsoPlayed`** (473–632) — разбор по типу предмета:
  - **`CWeapon`** (480–555): отдельно `CWeaponKnife` — `eFire/eFire2` при `is_standing` → `all_attack_0/1` (торс+ноги+голова вместе), иначе `attack_zoom/fire_idle`; базовый `CWeapon` — `eIdle` → `zoom` (при `IsZoomed()`) / `safemode` (при `m_bSafemode && !eSprint`) / `moving[moving_idx]`; `eReload` → `reload` либо (при `IsTriStateReload`) `reload/reload_1/reload_2` по `GetReloadState()`; `eShowing/eHiding` → `draw/holster`. Фолбэк при невалидном `M_torso` — `ST->m_torso[4]` (fix для бинокля, 554).
  - **`CMissile`** (556–602): стойка — `all_attack_0/1/2` для `eThrowStart/eReady/eThrow|eThrowEnd`; в движении — `attack_zoom/fire_idle/fire_end`.
  - **`CArtefact`** (603–617): `eActivating` → `zoom`, иначе moving/draw/holster.
  - **`CPda`** (618–631): `eIdle` → `zoom` при `m_bZoomed`, при спринте — `m_torso[0]`, иначе `m_torso[4]`.
  - **Без предмета** (635–641): спринт → `m_torso[0]`, иначе `m_torso[4]`.
- Мёртвая строка: `MotionID mid = ID_Cycle("norm_idle_0")` (643) — далее не используется.

### 7.6 Фолбэки и запуск (645–709)

- Ноги: при приседе без ускорения — жёстко `ID_Cycle("cr_idle_1")`, иначе `ST->legs_idle` (645–653); голова → `ST->m_head_idle` (654–655); торс → `m_current_torso` (если `m_bAnimTorsoPlayed`) либо `ST->m_torso_idle` (657–663).
- **Торс** (666–675): при смене — `PlayCycle(M_torso)`; при `m_bAnimTorsoPlayed` — `PlayCycle(..., TRUE, AnimTorsoPlayCallBack, this)`.
- **Голова** (677–683): `PlayCycle(M_head)` при смене.
- **Ноги** (685–709): сохранение фазы перехода `pos = fmod(timeCurrent, timeTotal)/timeTotal` (при движении в обоих кадрах), запуск через **`PlayMotionByParts(ka, M_legs, TRUE, legs_play_callback, this)`** (695, декларация 39–40), при старте движения `pos = 0.5f`, установка `timeCurrent = timeTotal * pos`, затем **`CStepManager::on_animation_start(M_legs, m_current_legs_blend)`** (708) — звук шагов.

### 7.7 DEBUG-вывод (712–760)

`g_ShowAnimationInfo` (335): имя текущего `moving_idx` (`mov_state[]`, 337–342), имена torso/head/legs (`LL_MotionDefName_dbg`), строка `MSTATE:` со всеми флагами (`Accel/Crouch/Fwd/.../+Jumping`).

### 7.8 Фазовый синк торса и ног (762–782)

Если `motion0` торса и `motion1` ног имеют флаг **`esmSyncPart`** — фаза торса подгоняется под ноги: `m_current_torso_blend->timeCurrent = legs.timeCurrent / legs.timeTotal * torso.timeTotal` (781–782).

## 8. Транспорт (`ActorAnimation.cpp:288–304`)

`CActor::steer_Vehicle(float angle)` (порц. 26): при `ENABLE_CAR` и `m_holder` как `CCar` — по `DriverAnimationType()` берёт коллекцию `m_vehicle_anims->m_vehicles_type_collections[anim_type]`; `angle == 0` → `PlayCycle(idles[0])`, `> 0` → `steer_right`, `< 0` → `steer_left`. Коллекция создаётся в `net_Spawn` (`Actor_Network.cpp:862`) и в `m_vehicle_anims->Create` (§4). Сопутствующее: `FindMotionKeys` (328–332) — `LL_GetRootMotion` для `MotionID` (используется внешним кодом движения).

## 9. Связи

- **Порц. 3** — `animation_movement_controlled()` (`CGameObject`, GameObject.cpp:1179), `create_anim_mov_ctrl`/`destroy_anim_mov_ctrl` (цепочка `CGameObject` → `CEntityAlive` → `CCustomMonster` → `CAI_Stalker`; у актёра — через `CEntityAlive`, entity_alive.cpp:756–770): анимация, управляющая движением (защёлкивание позы/перемещение), отключает физ.-контроль (`CharacterPhysicsSupport.cpp:217/240/1099`). IK-контроль ног (`m_legs_controller.is_active()`, §7.3) — та же подсистема.
- **Порц. 9 / ит. 6** — `select_animation` у актёра **не используется**: это `CControlAnimationBase::select_animation` (монстры, control_animation_base.cpp:202) и `CStalkerAnimationPair::select_animation` (stalker_animation_pair.cpp:223) / `AnimationManager` (`CAI_Stalker`) — см. план «select_animation (CCustomMonster)».
- **Порц. 23** — `mstate_rl`, `isActorAccelerated`, `mc*`-флаги, `mstate_old/mstate_real`.
- **Порц. 26** — `m_holder`/`CHolderCustom`, `steer_Vehicle` как callback транспорта.
- **Порц. 28** — `net_Spawn`/`OnChangeVisual` (инициализация анимаций и колбэков), `UpdateCL`-контекст.
- **Порц. 44** — `CScriptEntityAction`: скриптовые действия запускают анимации и `m_object->create_anim_mov_ctrl(blend, 0, true)` (script_entity.cpp:693).
- **Шаги** — `CStepManager::on_animation_start` (step_manager.cpp:117); также вызывается при посадке/высадке из транспорта (`Actor.cpp:1533/1707`).
- **HUD** — `g_player_hud->OnMovementChanged` (player_hud, руки/оружие в FP-виде).
- **Оружие** — `CHudItem::animation_slot()`, `CWeapon::GetState/IsTriStateReload/GetReloadState`, `CMissile::GetState`, `CArtefact::GetState`, `CPda::m_bZoomed` (ит. 8–10).

## 10. Замечания к плану

- `on_animation_stop`/`on_animation_switch` из плана **в коде отсутствуют** (поиск по всему `src/` — 0 совпадений); фактические события — запуск `CStepManager::on_animation_start` и колбэки-остановки `legs_play_callback`/`AnimTorsoPlayCallBack`.
- Класса **`CActorAnimation` в коде нет** (заголовок `ActorAnimation.h` содержит только макросы §2) — под «`CActorAnimation`-обвязка (порц. 19)» плана следует понимать `SActorMotions`/`actor_anim_defs.h` и `UpdateAnimation`-цепочку.
- В плане указано «`AnimTorsoPlayCallBack` (2821–2832)» — реализация занимает `Actor.cpp:2821–2825`, далее идут `ph_destroyable`/`create_entity_condition`.

## 11. Краткое резюме

Анимация актёра — два файла: `actor_anim_defs.h` (иерархия `SActorMotions` → normal/crouch/climb/sprint + 13 слотов торса под тип предмета + транспорт) и `ActorAnimation.cpp` (разбор имён анимаций, 5 костных колбэков ориентации, `g_SetAnimation`). Каждый кадр `g_SetAnimation(mstate_real)` выбирает набор (присед/лазание/обычный), определяет walk/run/sprint-индекс, по приоритетам собирает ноги (landing/turn/jump/направления), по типу активного предмета и его состояния — торс (zoom/safemode/reload-стадии/удары/броски/`all_attack_*` из стойки), запускает `PlayCycle`/`PlayMotionByParts` с сохранением фазы перехода, оповещает `player_hud`, уведомляет `CStepManager` (шаги) и синхронизирует фазу торса с ногами по `esmSyncPart`. Колбэки костей вращают spine/shoulder/head по `r_torso` без трансляции (голова учитывает freelook-камеру). Инициализация — `OnChangeVisual` → `m_anims->Create` + `SetCallbacks`; транспорт — `steer_Vehicle` по `DriverAnimationType`.
