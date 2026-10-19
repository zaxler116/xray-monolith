# CActor: движение и физика (порцион 23)

> **Модуль:** `xrGame` · **Файлы:** `Actor_Movement.cpp` (733), `Actor.h` (922), `Actor.cpp` (3128) · **Порцион 23, итерация 5**
>
> **Страница:** движение актёра (control states), валидация состояний (`g_cl_ValidateMState`), проверка контролей (`g_cl_CheckControls`), ориентация (`g_Orientate`/`g_cl_Orientate`/`g_sv_Orientate`/`g_LadderOrient`), ladder/freelook, `set_state_box`, `g_Physics`, `CanMove/CanJump/CanSprint/CanRun/CanAccelerate`, `StopAnyMove/StopSprint`, `AnyAction/AnyMove`, `is_jump`, `GetMovementSpeed`, `GetFireDispertion`, `MaxWalkWeight/get_additional_weight/GetWeaponAccuracyStm`, `ConvState`, `CameraHeight`, `SetPhPosition`, `ForceTransform`, `set_input_external_handler/input_external_handler_installed`.

## 1. Ответственность

Движение актёра (игрока): обработка wishful states (`mc*`), валидация/переходы реальных состояний (`g_cl_ValidateMState`), генерация желаемого ускорения (`g_cl_CheckControls`), ориентация модели/камеры (`g_Orientate`, ladder, freelook), работа с коробками состояний (`set_state_box`), физика движения (`g_Physics`), проверки возможностей движения, остановка движения, доступ к скорости, весовым ограничениям, установка позиции/трансформации, внешний input-хэндлер.

## 2. Control states (mstate)

Актёр использует битовые флаги движения (`actor_defs.h`/`entity_alive.h` контекст): `mcFwd/mcBack/mcLStrafe/mcRStrafe/mcJump/mcCrouch/mcSprint/mcLookout/mcLLookout/mcRLookout/mcClimb/mcFall/mcLanding/mcLanding2/mcAccel/mcAnyMove/mcAnyAction` и др. (упоминаются в коде `Actor_Movement.cpp`).

Поля:
- `mstate_wishful` — желаемое состояние от ввода (`Actor.h:470`)
- `mstate_old` — предыдущее реальное (`Actor.h:471`)
- `mstate_real` — текущее реальное (`Actor.h:472`)

## 3. `g_cl_ValidateMState(float dt, u32 mstate_wf)` (`Actor_Movement.cpp:41–163`)

Валидирует переходы между состояниями:

- **Lookout:** если одновременно `mcLLookout` и `mcRLookout` в `mstate_wf` или в `mstate_real` — снимает `mcLookout` из `mstate_real`. Если только `mcLookout` в `mstate_wf` — копирует флаг в `mstate_real`. Иначе снимает `mcLookout` (`41–59`).
- **Запрет lookout при прыжке/падении/приземлении:** если `mstate_real & (mcJump|mcFall|mcLanding|mcLanding2)` — снимает `mcLookout` (`60–62`).
- **Приземление (Landing/Landing2):** уменьшает `m_fLandingTime` по `dt`; по истечении снимает `mcLanding|mcLanding2` и `mcFall|mcJump` (`64–72`).
- **Падение/контакт с землёй:** если `character_physics_support()->movement()->gcontact_Was`:
  - если был `mcFall` и `GetContactSpeed() > 4.f`: если `fis_zero(GetContactSpeed() health lost?)` — `m_fLandingTime = s_fLandingTime1`, `mstate_real |= mcLanding`; иначе `m_fLandingTime = s_fLandingTime2`, `mstate_real |= mcLanding2`. Вызывает Lua `"_G.CActor_on_land"` с `GetContactSpeed()` (`74–96`).
  - сбрасывает `m_bJumpKeyPressed = TRUE`, `m_fJumpTime = s_fJumpTime`, снимает `mcFall|mcJump` (`96–99`).
- **Снятие прыжка по отпусканию:** если `(mstate_wf & mcJump)==0` → `m_bJumpKeyPressed = FALSE` (`100–102`).
- **Блокировка движения:** если ((скорость < 0.2 и не в `mcFall|mcJump|mcLanding|mcLanding2` ИЛИ (не в `mcClimb` и `Environment()==peAtWall`))) ИЛИ `movement()->bSleep` — снимает `mcAnyMove` (`103–109`).
- **Снятие прыжка на земле/у стены:** если `Environment()==peOnGround` или `peAtWall` и `((s_fJumpTime - m_fJumpTime)>s_fJumpGroundTime) && (mstate_real & mcJump)` — снимает `mcJump`, сбрасывает `m_fJumpTime = s_fJumpTime` (`109–119`).
- **Лестница (climb):** если `Environment()==peAtWall` и нет `mcClimb` — выставляет `mcClimb`, снимает `mcSprint`, вызывает `cam_SetLadder()` (`120–127`). Иначе если был `mcClimb` — вызывает `cam_UnsetLadder()` и снимает `mcClimb` (`128–136`).
- **Переход crouch<->stand:** если `mstate_wf != mstate_real` и был `mcCrouch`, а в wishful нет `mcCrouch` или сейчас `mcClimb` — пытается динамически активировать box 0 через `character_physics_support()->movement()->ActivateBoxDynamic(0)`, при успехе снимает `mcCrouch` (`137–147`).
- **Accel-флаг:** если `!CanAccelerate()` и `isActorAccelerated(mstate_real, IsZoomAimingMode())` — инвертирует `mcAccel` (`148–152`).
- **Скрытие оружия при лазании:** если текущий entity — актёр и состояние `mcClimb` изменилось — вызывает `SetWeaponHideState(INV_STATE_BLOCK_ALL, bOnClimbNow)` (`153–163`).

## 4. `g_cl_CheckControls(u32 mstate_wf, Fvector& vControlAccel, float& Jump, float dt)` (`Actor_Movement.cpp:165–250+`)

Генерирует желаемое ускорение/прыжок от ввода:

- Копирует `mstate_old = mstate_real`, обнуляет `vControlAccel` (`167–169`).
- **Падение:** если нет `mcFall` и `Environment()==peInAir` — `m_fFallTime -= dt`, по истечении `m_fFallTime = s_fFallTime`, выставляет `mcFall`, снимает `mcJump` (`171–182`).
- **Блокировка по CanMove:** если `!CanMove()` и `mstate_wf & mcAnyMove` — `StopAnyMove()`, чистит `mcAnyMove/mcJump` из `mstate_wf` (`183–192`).
- **Направления:** `mcFwd` → `vControlAccel.z += 1`, `mcBack` → `z -= 1`, `mcLStrafe` → `x += (-0.5 если sprint else -1)`, `mcRStrafe` → `x += (+0.5 если sprint else +1)` (`193–197`).
- **Прыжок:** если `curr_env == peOnGround || peAtWall`:
  - если `!m_bJumpKeyPressed` и `mstate_wf & mcJump` и `CanJump()` — `Jump = m_fJumpSpeed`, `m_bJumpKeyPressed = TRUE` (`199–210`)
  - иначе если `m_bJumpKeyPressed` — `Jump = 0` (`211–214`)
- **Климб/лазаем:** если `curr_env == peAtWall && mstate_real & mcClimb` — обрабатывает движение по стене/лестнице (устанавливает вертикальное ускорение в зависимости от `mstate_wf & mcFwd/mcBack`, ограничивает по `fLadder_cam_limit`) (`216–240`).
- **Ориентация:** вызывает `g_cl_Orientate(mstate_wf, dt)` (`246`).
- **Физика:** вызывает `g_Physics(vControlAccel, Jump, dt)` (`247`).
- **Валидация:** вызывает `g_cl_ValidateMState(dt, mstate_wf)` (`248`).

## 5. Ориентация: `g_Orientate`, `g_cl_Orientate`, `g_sv_Orientate`, `g_LadderOrient` (`Actor_Movement.cpp:246, ~250+`, `Actor.h:445–448`)

- `g_Orientate(u32 mstate_rl, float dt)` — общая ориентация актёра (`Actor.h:447`, реализация в `Actor_Movement.cpp`).
- `g_cl_Orientate(u32 mstate_rl, float dt)` — клиентская ориентация (`Actor.h:445`).
- `g_sv_Orientate(u32 mstate_rl, float dt)` — серверная ориентация (`Actor.h:446`).
- `g_LadderOrient()` — ориентация при лазании по лестнице/стене (`Actor.h:448`, `Actor_Movement.cpp`).

Эти методы управляют поворотом торса/модели актёра, учётом камеры, lookout'ов, ограничений при climb.

## 6. `set_state_box(u32 mstate)` (`Actor.h:835`, `Actor.cpp:1760–1773` по плану)

Устанавливает физический бокс в зависимости от состояния:
- если `mstate & mcCrouch`: если `isActorAccelerated(mstate_real, IsZoomAimingMode())` → активирует box 1 (`ActivateBox(1,true)`), иначе box 2 (`ActivateBox(2,true)`) (`Actor.cpp:1762–1768` по плану).
- иначе активирует box 0 (`ActivateBox(0,true)`) (`Actor.cpp:1770–1771`).

## 7. `g_Physics(Fvector& accel, float jump, float dt)` (`Actor.h:658`, `Actor_Movement.cpp:1027–1100` по плану)

Обрабатывает физику движения актёра: передаёт желаемое ускорение/прыжок в `CCharacterPhysicsSupport` (`movement()`), применяет ограничения по условиям (`CActorCondition` — порц. 20), обрабатывает climb, crouch, sprint, влияет на камеру/эффекторы. Детали реализации — в `Actor_Movement.cpp` (блок ~1027–1100).

## 8. Проверки возможностей движения

| Метод | Возврат | Файл/строка (план/код) | Назначение |
|---|---|---|---|
| `CanMove()` | `bool` | `Actor.h:453`, `Actor_Movement.cpp` | Может ли актёр двигаться в текущих условиях |
| `CanJump()` | `bool` | `Actor.h:452`, `Actor_Movement.cpp` | Может ли прыгать |
| `CanSprint()` | `bool` | `Actor.h:459`, `Actor_Movement.cpp` | Может ли спринтовать |
| `CanRun()` | `bool` | `Actor.h:460`, `Actor_Movement.cpp` | Может ли бежать |
| `CanAccelerate()` | `bool` | `Actor.h:451`, `Actor_Movement.cpp` | Может ли ускоряться |
| `StopAnyMove()` | `void` | `Actor.h:461`, `Actor_Movement.cpp` | Остановить любое движение (сброс wishful/реальных флагов движения) |
| `StopSprint()` | `void` | `Actor.h:462` (`mstate_wishful &= ~mcSprint`) | Остановить спринт |

## 9. Флаги состояния движения

| Метод | Возврат | Файл/строка | Назначение |
|---|---|---|---|
| `AnyAction()` | `bool` | `Actor.h:464` (`(mstate_real & mcAnyAction)!=0`) | В любом действии (по флагу) |
| `AnyMove()` | `bool` | `Actor.h:465` (`(mstate_real & mcAnyMove)!=0`) | В движении (по флагу) |
| `is_jump()` | `bool` | `Actor.h:467` | Проверка прыжка |
| `MovingState()` const | `u32` | `Actor.h:468` | Возвращает `mstate_real` |

## 10. Скорость, точность, вес

| Метод | Возврат | Файл/строка | Назначение |
|---|---|---|---|
| `GetMovementSpeed()` | `Fvector` | `Actor.h:497` (`return NET_SavedAccel`) | Скорость движения (сетевое сохранённое ускорение/скорость) |
| `GetFireDispertion()` const | `float` | `Actor.h:518` (`m_fdisp_controller.GetCurrentDispertion()`) | Текущая дисперсия стрельбы |
| `MaxWalkWeight()` const | `float` | `Actor.h:521` | Максимальный вес для ходьбы (доп. к `CActorCondition::MaxWalkWeight()` — см. порц. 20) |
| `get_additional_weight()` const | `float` | `Actor.h:522` | Дополнительный вес |
| `GetWeaponAccuracyStm()` | `float` | `Actor.h:525–526` (ifdef STATIONARYMGUN_NEW) | Точность для стационарного пулемёта |
| `CameraHeight()` | `float` | `Actor.h:454` | Высота камеры |

## 11. Конвертация состояния / ориентация

| Метод | Возврат | Файл/строка | Назначение |
|---|---|---|---|
| `ConvState(u32 mstate_rl, string128* buf)` | `void` | `Actor.h:573` | Конвертация состояния движения в строку (отладка/инфо) |
| `g_cl_Orientate(u32 mstate_rl, float dt)` | `void` | `Actor.h:445` | Клиентская ориентация |
| `g_sv_Orientate(u32 mstate_rl, float dt)` | `void` | `Actor.h:446` | Серверная ориентация |
| `g_Orientate(u32 mstate_rl, float dt)` | `void` | `Actor.h:447` | Общая ориентация |
| `g_LadderOrient()` | `bool` | `Actor.h:448` | Ориентация при лазании |

## 12. Позиционирование/трансформация

| Метод | Файл/строка | Назначение |
|---|---|---|
| `SetPhPosition(const Fmatrix& transform)` | `Actor.h:660`, `Actor.cpp:2542–2549` | Установка физической позиции (если нет physics shell — передаёт в `character_physics_support()->movement()->SetPosition(transform.c)`) |
| `ForceTransform(const Fmatrix& m)` | `Actor.h:659`, `Actor.cpp:2551–2566` | Принудительная трансформация: делегирует `character_physics_support()->ForceTransform(m)`, при MP блокирует урон на короткое время через `BlockDamageSet` (2 секунды / fixed_step) |

## 13. Внешний input-хэндлер

| Метод | Файл/строка | Назначение |
|---|---|---|
| `set_input_external_handler(CActorInputHandler* handler)` | `Actor.h:690` | Установка внешнего обработчика ввода |
| `input_external_handler_installed()` const | `bool` | `Actor.h:691` (`m_input_external_handler != 0`) | Проверка наличия внешнего хэндлера |
| `m_input_external_handler` | `CActorInputHandler*` | `Actor.h:696` | Поле хэндлера |
| `m_time_lock_accel` | `u32` | `Actor.h:697` | Блокировка ускорения по времени |

## 14. Связи

- **`CCharacterPhysicsSupport` (ит. 7):** `movement()`, `gcontact_Was`, `GetContactSpeed()`, `Environment()`, `ActivateBox/ActivateBoxDynamic`, `SetPosition`, `ForceTransform`, `BlockDamageSet`, `in_Hit` используется косвенно (PHHit).
- **`CActorCondition` (порц. 20):** влияет на `CanMove/CanSprint/CanJump/CanAccelerate` через условия (limping/cant_walk/cant_sprint, перегруз).
- **`CActorCameraManager`/камера (порц. 32):** `cam_SetLadder()`, `cam_UnsetLadder()`, `cam_Update`, `freelook` (cam_freelook).
- **Lua:** `"_G.CActor_on_land"` коллбэк в `g_cl_ValidateMState` (`Actor_Movement.cpp:93–95`).
- **Input (порц. 24):** `g_cl_CheckControls/g_cl_ValidateMState` работают с `mstate_wf` от ввода.
- **Inventory/оружие (порц. 33,27):** `SetWeaponHideState` при лазании, `GetFireDispertion`, `GetWeaponAccuracyStm`.
- **Network (порц. 28–30):** `NET_SavedAccel` в `GetMovementSpeed()`.
- **HUD (порц. 32):** взаимодействие с камерой/HUD при движении.