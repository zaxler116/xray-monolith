# CActor: input (порцион 24)

> **Модуль:** `xrGame` · **Файлы:** `ActorInput.cpp` (855), `actor_input_handler.cpp` (37), `Actor.h` (922), `Actor.cpp` (3128) · **Порцион 24, итерация 5**
>
> **Страница:** ввод актёра. `CActor` реализует `IInputReceiver` (Actor.h). Обработчики клавиатуры/мыши: `IR_OnKeyboardPress/Release/Hold`, `IR_OnMousePress/Release/Hold/Move`, `IR_OnMouseWheel`. Маппинг `key_*`/`mouse_*`, `g_cl_CheckControls` (детали порц.23), `set_input_external_handler`, `PickupModeUpdate/PickupModeUpdate_COD/PickupInfoDraw/DrawPickupItems`, `GetDefaultActionForObject` (детали), обработчики инвентаря/оружия.

## 1. Ответственность

Ввод игрока: обработка нажатий/удержаний/отпусканий клавиш и мыши через `IInputReceiver`. Управление wishful-состояниями движения (`mstate_wishful`), активация предметов/инвентаря, переключение режимов камеры (1st/3rd/freelook), использование holder/транспорта, pickup-режим, взаимодействие с `CInventory`, внешние input-хэндлеры (`CActorInputHandler`).

## 2. Место в архитектуре

- **Интерфейс:** `CActor` наследует/реализует `IInputReceiver` (Actor.h:75).
- **Обработчики:** `IR_OnKeyboardPress(int cmd)` (ActorInput.cpp:43), `IR_OnKeyboardRelease(int cmd)` (ActorInput.cpp ~), `IR_OnKeyboardHold(int cmd)` (ActorInput.cpp ~), `IR_OnMouseMove(int x,int y)` (Actor.h:502), `IR_OnMousePress(int btn)` (ActorInput.cpp ~), `IR_OnMouseRelease(int btn)` (ActorInput.cpp ~), `IR_OnMouseHold(int btn)` (ActorInput.cpp ~), `IR_OnMouseWheel(int direction)` (Actor.h:506).
- **Внешний хэндлер:** `CActorInputHandler* m_input_external_handler` (Actor.h:696), `set_input_external_handler(CActorInputHandler*)` (Actor.h:690), `input_external_handler_installed()` const (Actor.h:691). Файл `actor_input_handler.cpp` (37).
- **Движение:** wishful states модифицируются в input, валидируются/применяются через `g_cl_CheckControls/g_cl_ValidateMState` (порц.23).

## 3. Клавиатура

### `IR_OnKeyboardPress(int cmd)` (`ActorInput.cpp:43–200+`)

Основной обработчик нажатий:

- Если `hud_adj_mode && pInput->iGetAsyncKeyState(DIK_LSHIFT)` — возврат (`45`).
- `if (Remote()) return;` (`47`), `if (IsTalking()) return;` (`49`).
- Внешний хэндлер: `if (m_input_external_handler && !m_input_external_handler->authorized(cmd)) return;` (`50`).
- Обработка `cmd`:
  - `kWPN_FIRE` (`54–77`): если `(mstate_wishful & mcLookout) && !IsGameTypeSingle()` — возврат. Lua-функтор `_G.CActor_Fire` — если возвращает false — возврат (`59–66`). Если `OnServer()` — отправляет `M_PLAYER_FIRE` пакет с `ID()` (`69–76`).
- `if (!g_Alive()) return;` (`84`).
- Если `m_holder && kUSE != cmd`: делегирует `m_holder->OnKeyboardPress(cmd)`, если `m_holder->allowWeapon()` и `inventory().Action((u16)cmd, CMD_START)` вернуло true — return; иначе return (`86–91`). Иначе если `inventory().Action((u16)cmd, CMD_START)` вернуло true — return (`92–93`).
- DEBUG: `if(psActorFlags.test(AF_NO_CLIP)) { NoClipFly(cmd); return; }` (`95–101`).
- Переключатели состояний (по `cmd`):
  - `kJUMP` → `mstate_wishful |= mcJump` (`104–108`)
  - `kSPRINT_TOGGLE` (`109–118`): если `psActorFlags.test(AF_SPRINT_TOGGLE)`: если `AF_WALK_TOGGLE` — снимает `mcAccel`; если `AF_CROUCH_TOGGLE` — снимает `mcCrouch`; инвертирует `mcSprint` (`111–117`)
  - `kFREELOOK` (`119–133`): если `AF_FREELOOK_TOGGLE` — если `cam_freelook == eflDisabled && CanUseFreelook()` → `cam_SetFreelook()`; если `eflEnabled` → `cam_UnsetFreelook()` (`121–132`)
  - `kCROUCH` (`134–139`): если `AF_CROUCH_TOGGLE` → инвертирует `mcCrouch` (`136–138`)
  - `kACCEL` (`140–145`): если `AF_WALK_TOGGLE` → инвертирует `mcAccel` (`142–144`)
  - `kL_LOOKOUT` (`146–154`): если `AF_LOOKOUT_TOGGLE` — снимает `mcRLookout`, инвертирует `mcLLookout` (`148–153`)
  - `kR_LOOKOUT` (`155–164`): аналогично, снимает `mcLLookout`, инвертирует `mcRLookout` (`157–163`)
- Переключение камеры: `kCAM_1` → `cam_Set(eacFirstEye)` (`164`), `kCAM_2` → `cam_Set(eacLookAt)` (`165`), `kCAM_3` → `cam_Set(eacFreeLook)` (`166`)
- `kNIGHT_VISION` (`170–175`) — закомментировано (управляется Lua)
- `kTORCH` (`175–180`) — закомментировано (управляется Lua)
- `kDETECTOR` (`181–191`): берёт `inventory().ItemFromSlot(DETECTOR_SLOT)` → если `CCustomDevice* dev` → `dev->ToggleDevice(g_player_hud->attached_item(0) != NULL)` (`183–190`)

### `IR_OnKeyboardRelease(int cmd)` (`ActorInput.cpp` ~)
Обработка отпускания клавиш: сбрасывает соответствующие wishful-флаги (`mcJump`, lookouts и др.), может вызывать `inventory().Action(cmd, CMD_STOP)`, делегировать `m_holder` при наличии.

### `IR_OnKeyboardHold(int cmd)` (`ActorInput.cpp` ~)
Удержание клавиши: может вызывать `inventory().Action(cmd, CMD_HOLD)`, делегировать `m_holder`, обрабатывать специфичные действия.

### Внешний хэндлер (`actor_input_handler.cpp` 37)
`CActorInputHandler` предоставляет проверку `authorized(cmd)` и обработку ввода извне (каркас). `CActor` хранит указатель `m_input_external_handler`, методы `set_input_external_handler`/`input_external_handler_installed()`.

## 4. Мышь

### `IR_OnMouseMove(int x, int y)` (`Actor.h:502`, реализация в `ActorInput.cpp`)
Обработка движения мыши: поворот камеры/торса в зависимости от режима (1st/3rd/freelook), учёт `m_holder` (делегирование), ограничение при разговоре/внешнем хэндлере. Детали — в `ActorInput.cpp`.

### `IR_OnMousePress/Release/Hold(int btn)` (`ActorInput.cpp`)
Нажатие/отпускание/удержание кнопок мыши: `inventory().Action((u16)btn, CMD_START/STOP/HOLD)`, делегирование `m_holder` если есть и `allowWeapon()`/условия, обработка `kWPN_FIRE` при нажатии (аналог клавиатурного), учёт `Remote()/IsTalking()`.

### `IR_OnMouseWheel(int direction)` (`Actor.h:506`, реализация в `ActorInput.cpp`)
Прокрутка колеса: смена слотов оружия/предметов через инвентарь (`inventory().Action` с соответствующим cmd), делегирование `m_holder`.

## 5. Pickup-режим

| Метод | Файл/строка | Назначение |
|---|---|---|
| `PickupModeUpdate()` | `Actor.h:431` (protected), вызов в `UpdateCL` (Actor.cpp:1174) | Обновление состояния pickup-режима: проверка видимых предметов, дистанции, флагов, определение что можно подобрать |
| `PickupModeUpdate_COD(pickup_result_t pickup_state)` | `Actor.h:433` | COD-подобное обновление pickup (вариант) |
| `PickupInfoDraw(CObject* object)` | `Actor.h:432` | Отрисовка инфо о подбираемом предмете |
| `DrawPickupItems()` | `Actor.h:436` | Отрисовка списка подбираемых предметов |
| `m_bPickupMode` | `Actor.h:420` | Флаг pickup-режима (true при удержании USE) |
| `m_bDelayDrawPickupItems` | `Actor.h:437` | Флаг задержки отрисовки |
| `m_fPickupInfoRadius` | `Actor.h:424` | Радиус показа pickup-инфо (загружается в `Load`) |

В `UpdateCL` (Actor.cpp:1128–1149): если живой и текущий view entity — актёр, и нет внешнего блокирующего UI и нет `m_holder` — при удержании `kUSE` (action dik 0 или 1) с флагом `AF_MULTI_ITEM_PICKUP` выставляет `m_bPickupMode = true`, иначе `false` (`1132–1148`). Затем вызывается `PickupModeUpdate()` и `PickupModeUpdate_COD()` (`1174–1177`).

## 6. Инвентарь/действия

- `inventory().Action((u16)cmd, CMD_START/STOP/HOLD)` вызывается из клавиатурных/мышиных обработчиков. Если возвращает `true` — ввод поглощается (не обрабатывается дальше) (`92–93`, `89`).
- При наличии `m_holder`: сначала `m_holder->OnKeyboardPress(cmd)`; если `m_holder->allowWeapon()` и `inventory().Action(...)` вернуло true — return; иначе return (`86–91`).
- `GetDefaultActionForObject()` (Actor.h:401) — возвращает `*m_sDefaultObjAction` (tip-текст для объекта под прицелом), заполняется в `shedule_Update` при ray-query (`m_pUsableObject/m_pPersonWeLookingAt/m_pVehicleWeLookingAt/m_pObjectWeLookingAt/m_pInvBoxWeLookingAt`).

## 7. Камера/моды

- Переключение `kCAM_1/2/3` → `cam_Set(eacFirstEye/eacLookAt/eacFreeLook)` (`164–169`).
- `kFREELOOK`: toggle freelook по `AF_FREELOOK_TOGGLE` с проверкой `CanUseFreelook()` (`119–133`).
- `cam_freelook` (Actor.h:360), `freelook_cam_control` (Actor.h:361), `CanUseFreelook()` (Actor.h:374), `cam_SetFreelook()`/`cam_UnsetFreelook()` (Actor.h:372–373) — см. порц.23.

## 8. Условия/флаги ввода

- `psActorFlags` влияет на toggle-режимы: `AF_SPRINT_TOGGLE`, `AF_WALK_TOGGLE`, `AF_CROUCH_TOGGLE`, `AF_LOOKOUT_TOGGLE`, `AF_FREELOOK_TOGGLE`, `AF_MULTI_ITEM_PICKUP`, `AF_NO_CLIP` и др. (`ActorInput.cpp` использует `psActorFlags.test(...)`).
- Блокировки: `Remote()`, `IsTalking()`, `hud_adj_mode && LSHIFT`, `m_input_external_handler && !authorized(cmd)`, `!g_Alive()`.

## 9. Связи

- **IInputReceiver (порц.2 ит.2 контекст)** — интерфейс ввода.
- **Движение (порц.23)** — модификация `mstate_wishful` (`mcJump/mcSprint/mcAccel/mcCrouch/mcLLookout/mcRLookout`), валидируется в `g_cl_ValidateMState`.
- **Инвентарь (ит.8)** — `inventory().Action(...)` управляет сменой слотов/оружия/предметов.
- **Holder/транспорт (порц.26)** — делегирование ввода `m_holder` (`OnKeyboardPress`, `allowWeapon()`).
- **Камера (порц.32)** — `cam_Set`, `cam_SetFreelook/cam_UnsetFreelook`, `CanUseFreelook()`.
- **Lua** — `_G.CActor_Fire` functor в `IR_OnKeyboardPress` для `kWPN_FIRE` (`59–66`).
- **Pickup/UI (порц.32/33)** — `PickupModeUpdate*`, `DrawPickupItems`, `GetDefaultActionForObject` заполняется в `shedule_Update`.
- **Сеть (порц.28–30)** — отправка `M_PLAYER_FIRE` пакета при `OnServer()` и `kWPN_FIRE` (`69–76`).
- **Внешний хэндлер** — `CActorInputHandler` (actor_input_handler.cpp 37), методы `authorized`, `set_input_external_handler`.

## 10. Краткое резюме

Ввод актёра реализован через `IInputReceiver`: клавиатурные/мышиные обработчики модифицируют `mstate_wishful` (движение/lookout/crouch/sprint/accel), делегируют ввод в `inventory()` и при наличии `m_holder`, управляют камерой/freelook, обрабатывают `kWPN_FIRE` с Lua-проверкой и сетевой отправкой, поддерживают pickup-режим, используют `psActorFlags` для toggle-режимов. Внешний ввод через `CActorInputHandler`.