# CActor: смерть (порцион 22)

> **Модуль:** `xrGame` · **Файлы:** `Actor.h` (922), `Actor.cpp` (3128), `Actor_Events.cpp` (378) · **Порцион 22, итерация 5**
>
> **Страница:** смерть актёра. Связи: `Die(CObject*)` (§2), коллбэки перед смертью (порц. 5), `on_requested_spawn` (§3), `net_ExportDeadBody` (§4), события `OnEvent` (порц. 4, 34), `kill_type`/`SPECIAL_KILL_TYPE`/`killer_victim_angle` (порц. 8), `CCharacterPhysicsSupport::in_Die` (ит. 7), `OnDeath`-callback (порц. 13).

## 1. Ответственность

Смерть игрока (`CActor`): обработка `Die(CObject* who)`, перенос предметов из слотов/пояса в рюкзак (SP/MP), остановка звуков, переключение камеры, запуск туториала `game_over` (SP), обработка `on_requested_spawn` (респавн/смена состояния после смерти), экспорт мёртвого тела в сеть (`net_ExportDeadBody`).

## 2. `Die(CObject* who)` (`Actor.cpp:867–1011`)

`CActor::Die(CObject* who)` — переопределение `CEntityAlive::Die` (порц. 7).

**Порядок (основные шаги):**

1. **Детач от holder'а (транспорт):** `#ifdef HOLDERCUSTOM_NEW` → `use_HolderEx(NULL, true)` (`Actor.cpp:869–871`).
2. **Отладка:** `Msg("--- Actor [%s] dies !", this->Name())` (`Actor.cpp:873–876`).
3. **Базовая смерть:** `inherited::Die(who)` (`Actor.cpp:876`).
4. **Обработка инвентаря на сервере (`OnServer()`):** (`Actor.cpp:878–948`)
   - Перебор всех слотов `inventory().FirstSlot()` → `LastSlot()` (`Actor.cpp:880–918`):
     - Если текущий активный слот (`inventory().GetActiveSlot()`): 
       - SP — если граната `CGrenade*` → `grenade->DropGrenade()`; иначе `item_in_slot->SetDropManual(TRUE)` (`Actor.cpp:890–897`);
       - MP — закомментированная логика (не применяется) (`Actor.cpp:898–907`);
     - Иначе (неактивный слот): если `CCustomOutfit*` — **пропускает** (оставляет на теле), иначе — переносит в рюкзак `inventory().Ruck(item_in_slot)` (`Actor.cpp:911–917`).
   - **Очистка пояса:** `while (!l_blist.empty()) inventory().Ruck(l_blist.front())` (весь пояс → в рюкзак) (`Actor.cpp:921–923`).
   - **MP-особенность (ArtefactHunt / players bag):** если `!IsGameTypeSingle()` — проход по рюкзаку: для `eGameIDArtefactHunt` артефакты помечаются `SetDropManual(TRUE)`; предметы с `CLSID_OBJECT_PLAYERS_BAG` помечаются `SetDropManual(TRUE)` (`Actor.cpp:925–947`).
5. **Звуки (клиент):** если `!g_dedicated_server` — случайный звук смерти `sndDie[Random.randI(SND_DIE_COUNT)]` играет в позиции актёра; останавливаются `m_HeavyBreathSnd`, `m_BloodSnd`, `m_DangerSnd` (`Actor.cpp:950–957`).
6. **Камера и UI (SP/MP):** (`Actor.cpp:959–1006`)
   - **SP (`IsGameTypeSingle()`):**
     - **First Person Death:** если `firstPersonDeath` (глобальная переменная `extern BOOL firstPersonDeath`) — переключает камеру `cam_Set(eacFirstEye)`, `initFPCam()`, выставляет позицию/HPB камеры из `Device.vCameraPosition/Direction`, настраивает `m_FPCam` (`m_Position`, `m_HPB`, `m_Camera`, `m_customSmoothing=0`) (`Actor.cpp:962–978`).
     - Иначе — `cam_Set(eacFreeLook)` (`Actor.cpp:978`).
     - Скрывает открытые диалоги `CurrentGameUI()->HideShownDialogs()` (`Actor.cpp:980`).
     - Запускает туториал `start_tutorial("game_over")` (`Actor.cpp:1000`).
   - **MP:** `cam_Set(eacFixedLookAt)` (`Actor.cpp:1004`).
7. **Блокировка движения:** `mstate_wishful &= ~mcAnyMove`, `mstate_real &= ~mcAnyMove` (`Actor.cpp:1007–1008`).
8. **Очистка эффектора шока:** `xr_delete(m_sndShockEffector)` (`Actor.cpp:1010`).

**Связи Die:**
- `CCharacterPhysicsSupport::in_Die` (ит. 7) — вызывается из `CEntityAlive::Die`/цепочки смерти (упоминается в плане итерации 5 как связь).
- Скриптовый коллбэк перед смертью: `_G.CActor__BeforeHitCallback` проверяется в `Hit` (порц. 21), но непосредственно перед `inherited::Die` в коде `Die` нет явного глобального форка `ACTOR_BEFORE_DEATH_CALLBACK` — **в соответствии с планом (порц. 5)** фиксируем эту связь как «форк/коллбэк перед смертью» (может быть обработан в `CEntityAlive::Die` или через Lua-callback `eDeath`).
- `OnDeath`-callback (порц. 13): вызывается через `callback(eDeath)(...)` в наследниках/цепочке (обычно в `CEntityAlive::Die` или при обработке смерти) — фиксируем как связь с Lua-обёрткой `CScriptGameObject`.
- `kill_type`/`SPECIAL_KILL_TYPE`, `killer_victim_angle` (порц. 8) — передаются/формируются при генерации события смерти/в `SHit` (порц. 8, `wounds-death.md`).

## 3. `on_requested_spawn(CObject *object)` (`Actor.cpp:1722–1756`, `Actor.h:581`)

Вызывается при запросе спавна/после смерти (респавн актёра).

**Логика:**
1. `CHolderCustom* oHolder = smart_cast<CHolderCustom*>(object)`. Если нет — выход (`Actor.cpp:1724–1725`).
2. `#ifdef HOLDERCUSTOM_NEW`:
   - `use_HolderEx(oHolder, true)` — прикрепляет/обрабатывает holder (транспорт) при респавне (`Actor.cpp:1728`).
3. `#else` (ветка без HOLDERCUSTOM_NEW):
   - Получает `CGameObject* go`, `CPhysicsShellHolder* pholder` (`Actor.cpp:1730–1732`).
   - Если `pholder`: `pholder->PPhysicsShell()->SplitterHolderDeactivate()`; пытается `character_physics_support()->movement()->ActivateBoxDynamic(0)` — при неудаче реактивирует SplitterHolder и выходит; при успехе реактивирует SplitterHolder (`Actor.cpp:1734–1741`).
   - Создаёт/пересоздаёт character physics: `character_physics_support()->movement()->CreateCharacter()` (`Actor.cpp:1743`).
   - Устанавливает позицию/скорость выхода из holder'а: `SetPosition(oHolder->ExitPosition())`, `SetVelocity(oHolder->ExitVelocity())` (`Actor.cpp:1744–1745`).
   - Сбрасывает holder: `m_holder = NULL`, `m_holderID = (u16)(-1)` (`Actor.cpp:1747–1748`).
   - `use_HolderEx(oHolder, true)` (`Actor.cpp:1750`).
   - Копирует Y-ориентацию holder'а в `r_torso.yaw = xyz.y` (`Actor.cpp:1752–1755`).

**Назначение (каркас):** обработка респавна/возврата в мир после смерти (детали — ит. 12, MP/SP респавн).

## 4. `net_ExportDeadBody(NET_Packet&)` (`Actor.h:642`)

Метод экспорта мёртвого тела в сеть (каркас). Реализация — в `Actor_Network.cpp` (порц. 28). Здесь фиксируется только сигнатура и назначение: экспорт состояния мёртвого тела при сетевой смерти (MP).

## 5. События смерти (`OnEvent`)

`CActor::OnEvent(NET_Packet& P, u16 type)` (`Actor.h:125`, `Actor.cpp` — реализация в соответствующих файлах, включая `Actor_Events.cpp` 378). Обработка сетевых/игровых событий, связанных со смертью/спавном — каркас (порц. 4, 34).

## 6. Константы, тайминги респавна

`Actor.cpp:97–99`:
- `const u32 patch_frames = 50;`
- `const float respawn_delay = 1.f;` (задержка респавна)
- `const float respawn_auto = 7.f;` (авто-респавн)

Используются при логике респавна/смерти (SP) — каркас.

## 7. `TimePassedAfterDeath()` (`Actor.h:219`, `Actor.cpp:2602–2608`)

```cpp
ALife::_TIME_ID CActor::TimePassedAfterDeath() const
{
  if (!g_Alive()) return Level().timeServer() - GetLevelDeathTime();
  else return 0;
}
```
Используется в `NeedToDestroyObject()` (`Actor.cpp:2584–2600`, порц. 19) для определения времени удаления трупа (MP/SP).

## 8. Связи

- **`CEntityAlive::Die` (порц. 7)** — базовый вызов `inherited::Die(who)`.
- **`CCharacterPhysicsSupport::in_Die` (ит. 7)** — вызывается в цепочке смерти (через физическую поддержку).
- **Lua/callbacks (порц. 13)** — `callback(eDeath)(lua_game_object(), ...)` может вызываться в базовом `Die`/обработчиках смерти (OnDeath-callback). `_G.CActor__BeforeHitCallback` — коллбэк перед смертью через Hit (порц. 21).
- **`CActorCondition` (порц. 20)** — death-effector `m_death_effector`, условия/критические состояния влияют на смерть.
- **Hit/урон (порц. 21)** — источник смерти (`who`, `SHit`), `kill_type`/`SPECIAL_KILL_TYPE`/`killer_victim_angle` формируются при попадании (порц. 8).
- **Holder/транспорт (порц. 26)** — `use_HolderEx(NULL,true)` при смерти, `on_requested_spawn` работает с `CHolderCustom`.
- **Сеть (порц. 28–30)** — `net_ExportDeadBody` (экспорт мёртвого тела), события смерти в `OnEvent`.
- **Туториал/UI** — `start_tutorial("game_over")` (SP), `CurrentGameUI()->HideShownDialogs()`.
- **Звуки** — `sndDie[...]`, остановка дыхания/крови/опасности.

## 9. Краткое резюме

Смерть актёра: детач от транспорта → `CEntityAlive::Die` → обработка инвентаря (слоты → рюкзак, пояс → рюкзак, MP-особенности ArtefactHunt/players bag) → звуки → камера/UI (FPDeath/free look vs fixed look), `game_over` (SP), блокировка движения, удаление `m_sndShockEffector`. Респавн-обработка — `on_requested_spawn` (работа с holder'ом, позиционирование, физика). Сетевой экспорт мёртвого тела — `net_ExportDeadBody`. Тайминги респавна — `respawn_delay/respawn_auto`, `patch_frames`.