# CActor: input и feel (порционы 24–25)

> **Модуль:** `xrGame` · **Файлы:** `ActorInput.cpp` (855), `actor_input_handler.cpp` (37), `Actor_Feel.cpp` (397), `Actor.h` (922), `Actor.cpp` (3128) · **Порционы 24–25, итерация 5**
>
> **Страница:** input (§1–3) и feel (§4). Порц. 24 — input, порц. 25 — feel (обвязка актёра Feel::Touch/Vision/Sound, порц. 41–42).

## 1. Ответственность (input)

`CActor` реализует `IInputReceiver` (Actor.h). Обрабатывает клавиатуру/мышь, управляет `mstate_wishful`, переключает камеру/freelook/toggle-режимы, взаимодействует с инвентарём и holder'ом, поддерживает pickup-режим, использует внешний `CActorInputHandler`.

## 2. Интерфейсы и поля

- **Интерфейс:** `CActor : ..., IInputReceiver, Feel::Touch, ..., Feel::Sound` (Actor.h:73–84)
- **Методы IInputReceiver (объявлены):** `IR_OnMouseMove(int x,int y)` (Actor.h:502), `IR_OnKeyboardPress(int dik)` (ActorInput.cpp:43), `IR_OnKeyboardRelease(int dik)`, `IR_OnKeyboardHold(int dik)`, `IR_OnMouseWheel(int direction)` (Actor.h:506), `IR_OnMousePress/Release/Hold` (реализации в ActorInput.cpp)
- **Внешний хэндлер:** `set_input_external_handler(CActorInputHandler*)` (Actor.h:690), `input_external_handler_installed() const` (Actor.h:691), `m_input_external_handler` (Actor.h:696), `m_time_lock_accel` (Actor.h:697). Реализация/класс — `actor_input_handler.cpp` (37)

## 3. Input (порцион 24)

### 3.1 `IR_OnKeyboardPress(int cmd)` (ActorInput.cpp:43–200+)

Блокировки: `hud_adj_mode && pInput->iGetAsyncKeyState(DIK_LSHIFT)` → return; `Remote()` → return; `IsTalking()` → return; `m_input_external_handler && !authorized(cmd)` → return.

- `kWPN_FIRE` (54–77): если `(mstate_wishful & mcLookout) && !IsGameTypeSingle()` → return. Lua `_G.CActor_Fire` (functor) — если возвращает false → return. Если `OnServer()` → отправка пакета `M_PLAYER_FIRE` с `ID()`.
- `if (!g_Alive()) return`.
- Если `m_holder && kUSE != cmd`: `m_holder->OnKeyboardPress(cmd)`; если `m_holder->allowWeapon()` и `inventory().Action((u16)cmd, CMD_START)` вернул true → return; иначе return. Иначе если `inventory().Action((u16)cmd, CMD_START)` вернул true → return.
- DEBUG `AF_NO_CLIP` → `NoClipFly(cmd)`, return.
- Состояния: `kJUMP` → `mstate_wishful |= mcJump`. `kSPRINT_TOGGLE` (при `AF_SPRINT_TOGGLE`): если `AF_WALK_TOGGLE` — `mstate_wishful &= ~mcAccel`; если `AF_CROUCH_TOGGLE` — `mstate_wishful &= ~mcCrouch`; инверт `mcSprint`. `kFREELOOK` (при `AF_FREELOOK_TOGGLE`): если `cam_freelook == eflDisabled && CanUseFreelook()` → `cam_SetFreelook()`; если `eflEnabled` → `cam_UnsetFreelook()`. `kCROUCH` (при `AF_CROUCH_TOGGLE`) → инверт `mcCrouch`. `kACCEL` (при `AF_WALK_TOGGLE`) → инверт `mcAccel`. `kL_LOOKOUT` (при `AF_LOOKOUT_TOGGLE`): `mstate_wishful &= ~mcRLookout`; инверт `mcLLookout`. `kR_LOOKOUT`: `mstate_wishful &= ~mcLLookout`; инверт `mcRLookout`.
- Камеры: `kCAM_1`→`cam_Set(eacFirstEye)`, `kCAM_2`→`cam_Set(eacLookAt)`, `kCAM_3`→`cam_Set(eacFreeLook)`.
- `kNIGHT_VISION`, `kTORCH` — закомментированы (управление через Lua).
- `kDETECTOR`: берёт `inventory().ItemFromSlot(DETECTOR_SLOT)`, если `CCustomDevice*` → `dev->ToggleDevice(g_player_hud->attached_item(0) != NULL)`.

### 3.2 Keyboard Release/Hold, Mouse (каркас)

`IR_OnKeyboardRelease/Hold`, `IR_OnMousePress/Release/Hold/Move`, `IR_OnMouseWheel` — обработка отпусканий/удержаний мыши/клавиатуры, делегирование в `inventory().Action(cmd, CMD_*)`, при наличии `m_holder` — делегирование `m_holder` (с проверкой `allowWeapon()`), учёт блокировок (`Remote/IsTalking/external handler`).

### 3.3 Pickup-режим

`PickupModeUpdate()` (Actor.h:431, вызывается в `UpdateCL` Actor.cpp:1174), `PickupModeUpdate_COD(...)` (Actor.h:433), `PickupInfoDraw(CObject*)` (Actor.h:432), `DrawPickupItems()` (Actor.h:436). `m_bPickupMode` (Actor.h:420), `m_bDelayDrawPickupItems` (Actor.h:437), `m_fPickupInfoRadius` (Actor.h:424). В `UpdateCL`: если жив, текущий view entity — актёр, нет блокирующего UI, нет `m_holder` — при удержании `kUSE` (action dik 0/1) и `AF_MULTI_ITEM_PICKUP` → `m_bPickupMode = true`, иначе false; затем `PickupModeUpdate()` + `PickupModeUpdate_COD()`.

### 3.4 Связи input

`IInputReceiver` (порц.2 ит.2), движение (порц.23) — модификация `mstate_wishful`, инвентарь (ит.8) — `inventory().Action`, holder (порц.26), камера (порц.32), Lua `_G.CActor_Fire`, pickup/UI (32/33), сеть (28–30) — отправка `M_PLAYER_FIRE` на сервере, `CActorInputHandler` (actor_input_handler.cpp 37).

## 4. Feel: Touch/Vision/Sound (порцион 25)

### 4.1 Ответственность

Обвязка feel-систем актёра: `Feel::Touch`, `Feel::Vision`, `Feel::Sound` (Actor.h:76,80). Реализация методов feel в `Actor_Feel.cpp` (397). Связь с `CActorCondition` (порц.20), обработка контактов/видимости/звука, feel_touch_* (PDA-контакты — порц.12), `feel_visible_isRelevant`.

### 4.2 Интерфейсы Feel

`CActor` наследует:
- `public Feel::Touch` (Actor.h:76)
- `public Feel::Sound` (Actor.h:80)

Методы (виртуальные, переопределения/реализации по коду):
- `virtual void feel_touch_new(CObject* O)` (Actor.h:394)
- `virtual void feel_touch_delete(CObject* O)` (Actor.h:395)
- `virtual bool feel_touch_contact(CObject* O)` (Actor.h:396)
- `virtual bool feel_touch_on_contact(CObject* O)` (Actor.h:397)
- `virtual void feel_sound_new(CObject* who, int type, CSound_UserDataPtr user_data, const Fvector& Position, float power)` (Actor.h:131)
- `virtual Feel::Sound* dcast_FeelSound() { return this; }` (Actor.h:133)

`m_feel_touch`, `m_feel_vision`, `m_feel_sound` — поля feel (группы полей каркаса Actor.h, см. порц.19 §3.3). `m_feel_touch_characters` (Actor.h:263) — счётчик объектов в feel_touch для обновления колижена с актёром.

### 4.3 Actor_Feel.cpp (397) — обвязка актёра

`Actor_Feel.cpp` содержит реализацию feel-методов актёра (Touch/Vision/Sound). По плану (порц.25): `Feel::Touch/Vision/Sound`-обвязка актёра, `feel_touch_*` (PDA-контакты — порц.12), `feel_vision_*`, `feel_sound_*`, `ActorCondition`-связь (порц.20), `OnH_*`-обработчики (порц.3), `feel_visible_isRelevant`.

Краткое назначение (по плану):
- `feel_touch_new/delete/on_contact/contact` — обработка новых/удалённых контактов touch (PDA-контакты, зоны, артефакты)
- `feel_vision_query/update/mtl_transp` — зрение (`CCustomMonster` использует, порц.9), `feel_visible_isRelevant`
- `feel_sound_new/update` — звук
- Связь с `CActorCondition` (изменение состояний при контактах/зрении/звуке)

> Детали реализации `Actor_Feel.cpp` (397 строк) фиксируются здесь как каркас feel-обвязки актёра (порц.25). Полное раскрытие Touch/Vision/Sound — порционы 41–42.

### 4.4 Связи feel

`Feel::Touch/Vision/Sound` (порц.25,41–42), `CActorCondition` (порц.20), PDA-контакты (порц.12), `CCustomMonster` (использует feel_vision, порц.9), `CScriptEntity::process_sound_callbacks` (порц.43).

## 5. Краткое резюме

Порц.24 (input): `IInputReceiver` — обработчики клавиатуры/мыши, модификация `mstate_wishful`, переключатели камеры/freelook/toggle, делегирование в `inventory()` и `m_holder`, Lua `_G.CActor_Fire`, pickup-режим (`PickupModeUpdate*`), внешний `CActorInputHandler`.

Порц.25 (feel): наследует `Feel::Touch` и `Feel::Sound`, методы `feel_touch_*`, `feel_sound_new/dcast_FeelSound`, `m_feel_touch_characters`, реализация в `Actor_Feel.cpp` (397) — обвязка feel-систем актёра (Touch/Vision/Sound), связь с `CActorCondition`, PDA-контакты, `feel_visible_isRelevant`. Полные детали Touch/Vision/Sound — порц.41–42.