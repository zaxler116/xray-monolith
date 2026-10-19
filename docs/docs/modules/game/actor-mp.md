# CActorMP: клиент, сервер, RPC (порционы 29–30)

> **Модуль:** `xrGame` · **Файлы (порц. 29):** `actor_mp_client.h` (42), `actor_mp_client.cpp` (89), `actor_mp_client_export.cpp` (127), `actor_mp_client_import.cpp` (166), `actor_mp_state.h` (53), `actor_mp_state_inline.h` (15), `actor_mp_state.cpp` (288), `mpactor_dump_impl.cpp` (27), `anticheat_dumpable_object.h` (9)
>
> **Файлы (порц. 30):** `actor_mp_server.h` (32), `actor_mp_server.cpp` (44), `actor_mp_server_export.cpp` (48), `actor_mp_server_import.cpp` (48), `actor_communication.cpp` (221), `actor_statistic_mgr.h` (20), `actor_statistic_mgr.cpp` (189), `actor_statistic_defs.h` (33), `actor_memory.h` (35), `actor_memory.cpp` (50) · **Порционы 29–30, итерация 5**
>
> **Страница:** MP-класс актёра `CActorMP` (клиентский синк) и серверная сущность `CSE_ActorMP`, RPC-обновления. Связи: `net_*`/интерполяция `CActor` (порц. 28), регистрация в `CObjectFactory` (порц. 2, `ADD_MP`), `CActor::Die` (порц. 22), `CActorCondition` (порц. 20), `GEG_PLAYER_USE_BOOSTER` (порц. 4, 34), `CSE_ALifeCreatureActor` (ит. 11/12), `CInventoryOwner`-коммуникации (порц. 10), Lua-функторы/`discord_gameinfo` (порц. 28 §10).

## 1. Ответственность

Мультиплеерный актёра: `CActorMP : public IAnticheatDumpable, public CActor` (`actor_mp_client.h:8–10`) — клиентская сторона; `CSE_ActorMP : public CSE_ALifeCreatureActor` (`actor_mp_server.h:7`) — серверная сторона (§15). Вместо полноценного `net_update`-синка SP-актёра (порц. 28) обе стороны используют компактный `actor_mp_state_holder` — одну структуру `actor_mp_state`, сериализуемую в бит-пакет (~27 байт вместо 138). Клиент: заполнение state из физ./логического состояния, релевантность, экспорт/импорт пакета, поддержание очередей `NET`/`NET_A` для интерполяции, MP-ограничения камеры, бустеры-предметы, отчёт для античита. Сервер: приём state управляемого игроком актёра в поля `CSE`, отдача состояния всем клиентам, отбраковка мёртвых. Дополнительно в порционы 29–30: `RPC_Update*` (Discord), SP-коммуникации, статистика актёра, зрительная память.

## 2. Классы и структуры

### 2.1 `IAnticheatDumpable` (`anticheat_dumpable_object.h:4–9`)

```cpp
class IAnticheatDumpable {
public:
  virtual void DumpActiveParams(shared_str const& section_name, CInifile& dst_ini) const = 0;
  virtual shared_str const GetAnticheatSectionName() const { return ""; }
};
```

Интерфейс выгрузки активных параметров объекта в ini для античита. Обход dumpable-объектов: `configs_dumper.cpp:64` (поиск по `GetAnticheatSectionName`), `mp_config_sections.cpp:98–112` (перебор `dumpable_obj` → `DumpActiveParams(obj_sect_name, dest_dumper)`).

### 2.2 `CActorMP` (`actor_mp_client.h:8–40`)

| Поля/методы | Строка | Назначение |
|---|---|---|
| `actor_mp_state_holder m_state_holder` | 16 | Текущее сериализуемое состояние |
| `float prev_cam_inert_value` | 18 | Сохранённое `psCamInert` до `On_SetEntity` |
| `static const float cam_inert_value = 0.7f` | 19 / client.cpp:14 | Инерция камеры для не-управляемого актёра |
| `fill_state(actor_mp_state&)` | 22 | State ← физика/логика (§5) |
| `process_packet(net_update&)` | 23 | Очередь `NET` (§9) |
| `postprocess_packet(net_update_A&)` | 24 | Очередь `NET_A` (§9) |
| `net_Export` / `net_Import` / `net_Relevant` | 28–30 | Сетевой обмен (§6–8) |
| `OnEvent` | 31 | Только `GEG_PLAYER_USE_BOOSTER` (§10) |
| `Die(CObject*)` | 32 | HP := 0 + базовый `Die` (§10) |
| `DumpActiveParams` / `GetAnticheatSectionName` | 33–34 | Античит-выгрузка, секция `"mp_actor"` (§11) |
| `On_SetEntity` / `On_LostEntity` | 35–36 | Инертность камеры (§10) |
| `cam_Set(EActorCameras)` | 38 | MP: только первый взгляд (§10) |
| `use_booster(NET_Packet&)` | 39 | Использование бустера (§10) |

## 3. `actor_mp_state` (`actor_mp_state.h:6–30`)

Структура состояния (POD, `ZeroMemory` + `physics_quaternion.z = 1.f` при создании — `actor_mp_state_inline.h:4–8`):

| Поле | Тип | Комментарий кода |
|---|---|---|
| `physics_quaternion` | `Fquaternion` | из `SPHNetState` |
| `physics_angular_velocity/linear_velocity/force/torque/position` | `Fvector` | физ. sync |
| `position` | `Fvector` | «should be removed in future(?)» — логическая позиция |
| `logic_acceleration` | `Fvector` | `NET_SavedAccel` |
| `model_yaw` | `float` | угол модели |
| `camera_yaw/pitch/roll` | `float` | «should be removed in future» |
| `time` | `u32` | «should be removed» — `Level().timeServer()` |
| `health` | `float` | 0..1 |
| `radiation` | `float` | 0..1 (доза / 100) |
| `inventory_active_slot : 4` | бит-филд | слот инвентаря |
| `body_state_flags : 15` | бит-филд | `mstate_real & 0xffff` |
| `physics_state_enabled : 1` | бит-филд | `State.enabled` |

`actor_mp_state_holder` (`actor_mp_state.h:32–49`): поле `m_state` + условное `u32 m_mask` (только `#ifdef USE_DIFFERENCES`), методы `check/write/read/relevant/state()`.

> **`USE_DIFFERENCES`** (`actor_mp_state.h:4`) — **закомментирован** → режим дифф-синка не активен: `check()` всегда `true` (`actor_mp_state.cpp:75–82`), `relevant()` всегда `true` (121–152), маска в пакет не пишется (154–159). `#define USE_LOGIC_ACCELERATION` (`actor_mp_state.cpp:4`) — всегда включён (бит `logic_acceleration_flag = 1<<15`, строка 24).

## 4. Сериализация (`actor_mp_state.cpp`)

### 4.1 Битовые флаги и биты (`actor_mp_state.cpp:6–47`)

Флаги `physics_linear_velocity_{x,y,z}` (0–2), `physics_position_{x,y,z}` (3–5), `model_yaw` (6), `camera_yaw/pitch/roll` (7–9), `inventory_active_slot` (10), `body_state_flags` (11), `health` (12), `radiation` (13), `physics_state_enabled` (14), `logic_acceleration` (15). Размеры: slot 4 бит, body_state 15, health 8, radiation 4, phys_enabled 1 (37–44) → ровно 32 бита битового блока. Диапазон компонентов скорости: ±32 (46–47).

### 4.2 Хелперы (`actor_mp_state.cpp:84–119`)

- `write(bit_count, value, current, output)` (84–89) / `read(bit_count, current, output)` (91–97) — упаковка полей в `u32` по битам.
- `pack(unpacked_value, bit_count)` (106–119) — нормализация 0..1 → целое, `iFloor(max*value+.5)`; при `bit_count>1` ненулевое значение никогда не пакуется в 0 (+1).
- `unpack(packed_value, bit_count)` (99–104) — обратно: `packed / (max + .0001f)`.

### 4.3 `write(NET_Packet&)` (`actor_mp_state.cpp:154–218`)

1. При `USE_DIFFERENCES`: `R_ASSERT(m_mask)` + `w_u32(m_mask)` (отключено).
2. `w_u32(m_state.time)`.
3. Скорость: clamp ±32 → 3 × `w_float_q8` (по компоненте).
4. Позиция: 3 × `w_float(physics_position.*)` (полная точность).
5. `model_yaw`: `w_float` (q8-вариант закомментирован, 180).
6. Камера: yaw/pitch/roll — 3 × `w_float_q8(0, PI_MUL_2)` (181–186).
7. `w_sdir(logic_acceleration)` (189).
8. Битовый блок (slot/health/radiation/phys_enabled/body_state) → 1–4 байта `w_u8` (194–217), запись прекращается по `current <= 8/16/24`.

### 4.4 `read(NET_Packet&)` (`actor_mp_state.cpp:220–288`)

Зеркально: маска (откл.), `r_u32(time)`, 3×`r_float_q8`-скорость, 3×`r_float`-позиция, yaw, 3×q8-камера, `r_sdir`. **`m_state.position = m_state.physics_position`** (255) — логическая позиция всегда равна физической после чтения. Битовый блок читается по `total_bit_count` (261–279), поля распаковываются `unpack` (health/radiation).

## 5. `fill_state` (`actor_mp_client_export.cpp:40–95`)

`CActorMP::fill_state(actor_mp_state& state)`:

- `SPHNetState State; PHGetSyncItem(0)->get_State(State)` (48–49) — единственный sync-item.
- `physics_*` ← из `State` (68–73); `state.position = Position()` (75) — логическая позиция; `logic_acceleration = NET_SavedAccel` (77).
- `model_yaw = angle_normalize(r_model_yaw)`, `camera_yaw/pitch/roll = angle_normalize(unaffected_r_torso.*)` (79–82).
- `time = Level().timeServer()` (84).
- `inventory_active_slot = inventory().GetActiveSlot()`, `body_state_flags = mstate_real & 0x0000ffff` (86–87).
- `health = GetfHealth()`; **`if (health < EPS) health = 0`** (88–91) — иначе после паковки в 8 бит получится положительное значение.
- `radiation = g_Radiation() / 100.0f` (93), `physics_state_enabled = State.enabled ? 1 : 0` (94).

В шапке файла — детальная сводка экономии байт (комментарии 8–38): 111 байт вырезано из 138 (осталось ~27): health 8 бит, убраны angular velocity/force/torque/quaternion/logical position/velocity/team/squad/group, квантование камеры и скоростей.

## 6. `net_Relevant()` (`actor_mp_client_export.cpp:97–115`)

```cpp
if (character_physics_support()->IsRemoved()) return FALSE;
actor_mp_state state; fill_state(state);
return m_state_holder.relevant(state);
```

Без `USE_DIFFERENCES` `relevant()` всегда возвращает `true` (кроме removed-физподдержки) — пакет отправляется каждый синк. Ветки проверки `g_Alive()`/`m_i_am_dead` закомментированы (101–106).

## 7. `net_Export(NET_Packet&)` (`actor_mp_client_export.cpp:117–127`)

`R_ASSERT2(valid_pos(m_state_holder.state().position), "trying to export bad position")` → `m_state_holder.write(packet)` — в пакет идёт state из `m_state_holder` (заполненный `fill_state` в `net_Relevant`/на сервере — см. §9 связи).

## 8. `net_Import(NET_Packet&)` (`actor_mp_client_import.cpp:11–106`)

1. `m_state_holder.read(P)`; `R_ASSERT2(valid_pos(...position), "imported bad position")` (15–16).
2. **Health** (OnClient, 22–42): если `GetfHealth() < new_health` — лечит всегда; иначе (урон) — `SetfHealth` только если `!ps || !ps->testFlag(GAME_PLAYER_FLAG_INVINCIBLE)`.
3. **Ранний выход при `PPhysicsShell() != NULL`** (44–47) — физ. тело отключает радиацию/слот/интерполяцию.
4. `SetfRadiation(radiation * 100.0f)` (OnClient, 50–51).
5. **Слот:** если `inventory().GetActiveSlot() != ActiveSlot` → `SetActiveSlot` (53–61).
6. Сборка `net_update N`: `mstate` ← `body_state_flags`, `dwTimeStamp` ← `time`, `p_pos` ← `position`, `o_model`, `o_torso` ← camera_*; **нормализация roll**: `if (roll > PI) roll -= PI_MUL_2` (73–74).
7. **Камера** (77–86): при `IsDemoPlay() || OnServer() || Remote()` — копирование в `unaffected_r_torso` и `cam_Active()->yaw = -o_torso.yaw`, `pitch = o_torso.pitch` (без roll).
8. `N.p_accel = logic_acceleration` → `process_packet(N)` (89–91).
9. `net_update_A N_A; m_States.clear();` — заполнение `N_A.State` из `physics_*` (93–102) → `postprocess_packet(N_A)` (105).

## 9. `process_packet` / `postprocess_packet` (`actor_mp_client_import.cpp:108–166`)

### `process_packet(net_update& N)` (142–166)

- `if (Local() && OnClient()) return` — локальный клиент не интерполирует себя.
- Отбраковка устаревших: `N.dwTimeStamp < NET.back().dwTimeStamp` → return.
- `g_Alive()` → `setVisible(TRUE)`, `setEnabled(TRUE)` (150–154).
- Совпадение таймстемпа → замена `NET.back() = N`; иначе `push_back`, лимит **5** (`pop_front`).

### `postprocess_packet(net_update_A& N_A)` (108–140)

- `N_A.dwTimeStamp` ← `NET.back().dwTimeStamp` или `Level().timeServer()` (110–113); `previous_position/previous_quaternion` ← текущие (115–116).
- `if (Local() && OnClient() || !g_Alive()) return` (118).
- Аналогичная отбраковка/замена/push в `NET_A`, лимит 5 (122–132); при `!NET_A.empty()` → `m_bInterpolate = true` (134).
- `Level().AddObject_To_Objects4CrPr(this)`, `CrPr_SetActivated(false)`, `CrPr_SetActivationStep(0)` (137–139) — постановка в очередь коррекции предсказания (см. `PH_*_CrPr` в порц. 28).

## 10. `CActorMP`: события и поведение (`actor_mp_client.cpp`)

| Метод | Строки | Логика |
|---|---|---|
| `CActorMP()` | 16–19 | Пусто (закомментирован `m_i_am_dead = false`) |
| `OnEvent(P, type)` | 21–29 | `GEG_PLAYER_USE_BOOSTER` → `use_booster(P)`; иначе `inherited::OnEvent` |
| `Die(killer)` | 31–37 | `conditions().SetHealth(0.f)` (закомментированный `m_i_am_dead`) → `inherited::Die` (порц. 22) |
| `cam_Set(style)` | 39–49 | **В релизе (`#ifndef DEBUG`) пропускает всё, кроме `eacFirstEye`**; `old_cam->OnDeactivate()` → `cam_Active()->OnActivate(old_cam)` |
| `use_booster(P)` | 51–74 | `OnServer()` → выход; `r_u16(id)` → `Objects.net_Find` → `smart_cast<CEatableItem*>` → `UseBy(this)` (с VERIFY2/Msg-ошибками) |
| `On_SetEntity()` | 76–84 | `prev_cam_inert_value = psCamInert`; `if (this != Level().CurrentControlEntity()) psCamInert = 0.7f` → `inherited::On_SetEntity` |
| `On_LostEntity()` | 86–89 | `psCamInert = prev_cam_inert_value` |

## 11. Античит-выгрузка (`mpactor_dump_impl.cpp:4–27`)

`CActorMP::DumpActiveParams(section_name, dst_ini)` — 16 × `w_float`: движение (`m_fWalkAccel`, `m_fJumpSpeed`, `m_fRunFactor`, `m_fRunBackFactor`, `m_fWalkBackFactor`, `m_fCrouchFactor`, `m_fClimbFactor`, `m_fSprintFactor`), стрейф (`m_fWalk_StrafeFactor`, `m_fRun_StrafeFactor`), дисперсия (`m_fDispBase`, `m_fDispAim`, `m_fDispVelFactor`, `m_fDispAccelFactor`, `m_fDispCrouchFactor`, `m_fDispCrouchNoAccelFactor`). Секция — `"mp_actor"` (`actor_mp_client.h:34`).

## 12. Регистрация

`src/xrServerEntities/object_factory_register.cpp`:
- L193: `#define ADD_MP(a,b,c,d,e,f) add(xr_new<CObjectItemClientServerSingleMp<a,b,c,d> >(e,f))` — макрос MP-регистрации (клиент+сервер+MP-серверная сущность).
- **L235:** `ADD_MP(CActor, CActorMP, CSE_ALifeCreatureActor, CSE_ActorMP, CLSID_OBJECT_ACTOR, "actor")` — **единственный MP-класс** в фабрике (порц. 2).

`CSE_ActorMP` (`actor_mp_server.h:7–30`) — серверная сущность с собственным `actor_mp_state_holder` и `m_ready_to_update` — §15–18.

## 13. Связи

- **Порц. 28 (сетевой каркас `CActor`)** — `NET`/`NET_A`/`m_States`/`m_bInterpolate`/`InterpData`, `RPC_Update*`, `AddObject_To_Objects4CrPr`/`CrPr_*` — общая инфраструктура, `CActorMP` подменяет только формат пакета.
- **Порц. 30 (§15–22: `CSE_ActorMP`, сервер, `actor_communication.cpp`, статистика, память)** — server-side `UPDATE_Read/Write`, `STATE_Read/Write`, `Net_Relevant`, `RPC_Update*` (Discord).
- **Порц. 2 (фабрика объектов)** — `ADD_MP` L235.
- **Порц. 22 (`Die`)** — `CActorMP::Die` добавляет `conditions().SetHealth(0)` и вызывает базовую смерть.
- **Порц. 20 (`CActorCondition`)** — `conditions().SetHealth`, `g_Radiation()` (источники health/radiation для state).
- **Порц. 4/34 (`OnEvent`, `GE_*`)** — `GEG_PLAYER_USE_BOOSTER`; `CEatableItem::UseBy` (бустеры, ит. 8/10).
- **Ит. 11/12** — `CSE_ActorMP : CSE_ALifeCreatureActor`, `CObjectItemClientServerSingleMp`.
- **Античит** — `configs_dumper.cpp`, `mp_config_sections.cpp` (обход `IAnticheatDumpable`).
- **Ит. 12 (`Level`)** — `IsDemoPlay()`, `timeServer()`, `CurrentControlEntity()`, очередь `Objects4CrPr`.

## 14. Краткое резюме (порц. 29)

`CActorMP` — MP-актёра с компактным state-синком: `actor_mp_state` (позиция/скорость/углы/health/radiation/слот/флаги, ~27 байт против 138) сериализуется через `actor_mp_state_holder::write/read` с бит-паковкой (`USE_DIFFERENCES` отключён → полный state каждый пакет). `fill_state` собирает состояние из `PHGetSyncItem(0)`, `net_Relevant` всегда true (кроме removed-физики), `net_Import` применяет health (с защитой INVINCIBLE при уроне), радиацию, слот, камеру и раскладывает state в `net_update`/`net_update_A` → `process_packet`/`postprocess_packet` (очереди ≤5, отбраковка по времени, `m_bInterpolate`, `Objects4CrPr`). Поведение: только `eacFirstEye` (релиз), инертность камеры 0.7 для не-управляемого актёра, `GEG_PLAYER_USE_BOOSTER`, античит-выгрузка 16 float в секцию `"mp_actor"`. Регистрация — `ADD_MP(CActor, CActorMP, ...)` (единственный MP-класс фабрики). Серверный side (`CSE_ActorMP`) и `actor_communication.cpp` — порц. 30.

---

## 15. Серверная сущность `CSE_ActorMP` (`actor_mp_server.h`, `actor_mp_server.cpp`)

```cpp
class CSE_ActorMP : public CSE_ALifeCreatureActor   // actor_mp_server.h:7
{
  actor_mp_state_holder m_state_holder;  // то же состояние, что у клиента
  bool m_ready_to_update;                // состояние обновлено после spawn/загрузки
  void fill_state(actor_mp_state& state);          // §16
public:
  CSE_ActorMP(LPCSTR section);
  virtual void UPDATE_Read(NET_Packet& packet);    // §17
  virtual void UPDATE_Write(NET_Packet& packet);   // §16
  virtual void STATE_Read(NET_Packet& packet, u16 size);   // §18
  virtual void STATE_Write(NET_Packet& packet);            // §18
  virtual BOOL Net_Relevant();                             // §18
#ifdef XRGAME_EXPORTS
  virtual void on_death(CSE_Abstract* killer);             // §18
#endif
};
```

Конструктор (`actor_mp_server.cpp:4–8`): `m_ready_to_update = false` — до первого `fill_state`/`UPDATE_Read`. Регистрируется парой `CActorMP` в фабрике через `ADD_MP(CActor, CActorMP, CSE_ALifeCreatureActor, CSE_ActorMP, ...)` (порц. 2, `object_factory_register.cpp:235`).

**Общая схема синка (сервер ↔ клиент):** обе стороны используют один `actor_mp_state` + `actor_mp_state_holder` (§3–4) и одну и ту же сериализацию; клиент заполняет state из физ. объекта (`PHGetSyncItem`, §5), сервер — из полей `CSE` (§16); `net_Import`/`UPDATE_Read` — зеркальные разборы.

## 16. Серверный экспорт: `fill_state` / `UPDATE_Write` (`actor_mp_server_export.cpp:8–48`)

### `fill_state(actor_mp_state&)` (8–35)

Источники полей — поля `CSE_ALifeCreatureActor`/`CSE_ALifeCreatureAbstract`:

| Поле state | Источник (CSE) | Строка |
|---|---|---|
| `physics_*` (quaternion/angular/linear/force/torque/position) | `m_AliveState.*` | 10–15 |
| `position` | `o_Position` | 17 |
| `logic_acceleration` | `accel` | 19 |
| `model_yaw`, `camera_yaw/pitch/roll` | `angle_normalize(o_model/o_torso.*)` | 21–24 |
| `time` | `timestamp` | 26 |
| `inventory_active_slot` | `weapon` | 28 |
| `body_state_flags` | `mstate` | 29 |
| `health` | `get_health()` | 30 |
| `radiation` | `fRadiation` (как есть; у клиента — `g_Radiation()/100`) | 31 |
| `physics_state_enabled` | `m_AliveState.enabled ? 1 : 0` | 32 |

В конце — `m_ready_to_update = true` (34).

### `UPDATE_Write(NET_Packet&)` (37–48)

1. `if (!m_ready_to_update)` → `fill_state` + `m_state_holder.relevant(state)` (39–44) — первый/восстановленный state после спавна или загрузки.
2. `R_ASSERT2(valid_pos(m_state_holder.state().position), "trying to write bad position")` (46).
3. `m_state_holder.write(packet)` (47).

## 17. Серверный импорт: `UPDATE_Read` (`actor_mp_server_import.cpp:7–48`)

Пакет от клиента (контролирует сервер — серверный `CSE` принимает state управляемого игроком актёра):

1. Сброс: `flags = 0`, `m_u16NumItems = 1`, `velocity.set(0,0,0)` (9–11).
2. **Мёртвые:** `if (get_health() <= 0)` → чтение во **временный** `actor_mp_state_holder` и `return` (13–18) — пакет поглощается без применения (мёртвые не синхронизируются, см. комментарии в export-шапке порц. 29 §5).
3. `m_state_holder.read(packet)` + `R_ASSERT2(valid_pos(...position), "read bad position")` (19–20).
4. Разбор state в поля `CSE` (зеркально §16): `m_AliveState.*` (22–27), `o_Position` (29), `accel` (31), `o_model`/`o_torso.*` (33–36), `timestamp` (38), `weapon` (40), `mstate` (41), **`set_health(state.health)`** (42), `fRadiation` (43), `m_AliveState.enabled` (44).
5. `m_ready_to_update = true` (46) — следующий `UPDATE_Write` не перезаполнит состояние.

## 18. `STATE_*`, `Net_Relevant`, `on_death` (`actor_mp_server.cpp:10–44`)

| Метод | Строки | Логика |
|---|---|---|
| `STATE_Read(packet, size)` | 10–18 | `inherited::STATE_Read` + DEBUG-лог health |
| `STATE_Write(packet)` | 20–27 | `inherited::STATE_Write` + DEBUG-лог health |
| `Net_Relevant()` | 29–33 | **`if (get_health() <= 0) return FALSE`** — мёртвый CSE не отправляется → `inherited::Net_Relevant` |
| `on_death(killer)` | 36–44 | `inherited::on_death` → `fill_state` + `m_state_holder.relevant(state)` — фиксация последнего состояния в holder (без `USE_DIFFERENCES` — просто обновление `m_state`) |

`STATE_*` — spawn-пакет (сохранение/загрузка мира), `UPDATE_*` — потоковый синк (см. контракты `CSE_Abstract`, ит. 11/12).

## 19. `RPC_Update*` (`Actor.h:875–877`, `Actor.cpp:1421–1472`)

> Несмотря на имя, это **не сетевой RPC**, а обновление **Discord Rich Presence** (`discord_gameinfo`) из Lua-функторов.

| Метод | Lua-функтор | Заполняет | Строка |
|---|---|---|---|
| `RPC_UpdateFaction()` | `_g.get_actor_true_community` | `discord_gameinfo.faction` (id), `faction_name` ← перевод `st_faction_%s` | 1421–1436 |
| `RPC_UpdateRank()` | `ranks.get_player_rank_name` | `discord_gameinfo.rank_name` ← перевод `st_rank_%s` | 1438–1452 |
| `RPC_UpdateReputation()` | `ranks.get_player_reputation` → `alun_utils.get_reputation_name(reputation)` | `discord_gameinfo.reputation` ← перевод | 1454–1472+ |

**Вызовы:** `UpdateCL` (`Actor.cpp:1389–1391`) внутри блока `discord_gameinfo.ex_update` (порц. 28 §10, там же `Level().GameTaskManager().RPC_UpdateTaskName()` на 1386), `InventoryOwner.cpp:452/468/490` (при изменении faction/rank/reputation), `level_script.cpp:1184–1186` (вызов из Lua). Все три пишут в глобальную `discord_gameinfo` — см. также её инициализацию в `UpdateCL` (порц. 28 §10).

## 20. `actor_communication.cpp` (221) — коммуникации актёра (SP)

| Метод | Строки | Назначение |
|---|---|---|
| `AddGameNews(GAME_NEWS_DATA&)` | 34–44 | `receive_time = Level().GetGameTime()`, push в `game_news_registry`, UI `UIMainIngameWnd->ReceiveNews` |
| `OnReceiveInfo(info_id)` | 47–70 | `CInventoryOwner::OnReceiveInfo` → `CInfoPortion::Load` → callback `eInventoryInfo` → обновление вопросов `TalkMenu` (SP) |
| `OnDisableInfo(info_id)` | 73–86 | Базовый + `TalkMenu->NeedUpdateQuestions` |
| `ReceivePhrase(phrase_dialog)` | 88–98 | `TalkMenu` → `CPhraseDialogManager::ReceivePhrase` |
| `UpdateAvailableDialogs(partner)` | 100–113 | Очистка `m_AvailableDialogs/m_CheckedDialogs`, добавление `CharacterInfo().ActorDialogs()` партнёра, `inherited` |
| `TryToTalk()` | 115–121 | `!IsTalking()` → `RunTalkDialog(m_pPersonWeLookingAt, false)` |
| `RunTalkDialog(talk_partner, disable_break)` | 123–136 | `OfferTalk(this)` → `StartTalk` → скрытие `TopInputReceiver()` → `CUIGameSP::StartTalk(bDisableBreakDialog)` |
| `StartTalk(talk_partner)` | 138–146 | `CInventoryOwner::StartTalk` + отключение free-look камеры (`cam_UnsetFreelook`) |
| `NewPdaContact(pInvOwner)` | 148–156 | SP-only: `AnimateContacts(alive)`, `MapManager().AddRelationLocation` |
| `LostPdaContact(pInvOwner)` | 158–170 | Удаление map-локаций всех типов отношений + `"deadbody_location"` |
| `AddGameNews_deffered(news_data, delay)` | 172–180 | Отложенная новость (`time = dwTimeGlobal + delay`), сортировка `m_defferedMessages` |
| `UpdateDefferedMessages()` | 182–196 | Вывод по времени (`back()` → `AddGameNews` → `xr_delete`) |
| `OnDialogSoundHandlerStart/Stop(inv_owner, phrase)` | 198–214 | `smart_cast<CAI_Trader*>` → `dialog_sound_start/stop` |
| `DumpTasks()` (DEBUG) | 217–220 | `Level().GameTaskManager().DumpTasks()` |

## 21. `actor_statistic_mgr.cpp` (189) — статистика актёра

**Структуры** (`actor_statistic_defs.h`): `SStatDetailBData` (`key`, `int_count`, `int_points`, `str_value`), `SStatSectionData` (`key` + `vStatDetailData`), `vStatSectionData`, `CActorStatisticRegistry = CALifeAbstractRegistry<u16, vStatSectionData>`. Токен `actor_stats_token` — закомментирован (`actor_statistic_mgr.cpp:6–17`).

- **Сериализация** — `SStatDetailBData::save/load` (18–34): `str_value` пишется только при версии alife-header `> 0x0002`; `SStatSectionData::save/load` (38–80): при версии ровно `0x0002` — миграция старых int-ключей токенов (100→`total`, 1→`stalkerkills`, 2→`monsterkills`, 3→`quests`, 4→`artefacts`, 5→`reputation`, 0→`foo`) и пропуск старого `total_points`.
- **`GetData(key)`** (82–97) — поиск detail в секции, при отсутствии — создание с нулевыми count/points.
- **`GetTotalPoints()`** (99–112) — `sum(int_count * int_points)`; если есть `str_value` → **−1** (строковые секции не суммируются).
- **`CActorStatisticMgr`** (114–148): ctor создаёт `CActorStatisticsWrapper` и `registry().init(0)`; `GetSection(key)` — поиск/создание секции в `GetStorage()`.
- **`AddPoints(key, detail_key, str_value)`** (150–155) — строковое значение; **`AddPoints(key, detail_key, cnt, pts)`** (157–163) — `int_count += cnt`, `int_points += cnt * pts`.
- **`GetSectionPoints(key)`** (165–189): для `"total"` — сумма `GetTotalPoints()` по всем секциям (−1-секции пропускаются, общий −1 если нет ни одной числовой), иначе — `GetSection(key).GetTotalPoints()`.

## 22. `actor_memory.cpp` (50) — память (зрение) актёра

`CActorMemory : public vision_client` (`actor_memory.h:15`), поле `CActor* m_actor`.

- **Ctor** (15–23): `inherited(actor, 100)` — vision-клиент с лимитом 100 объектов, `VERIFY(m_actor)`.
- **`feel_vision_isRelevant(O)`** (25–32): релевантны только `CEntityAlive` (остальные — false).
- **`camera(position, direction, normal, fov, aspect, near, far)`** (34–50): `m_actor->cam_Active()->Get(...)`, `fov = deg2rad(camera.f_fov)`, `aspect = camera.f_aspect`, `near = .1f`, `far = g_pGamePersistent->Environment().CurrentEnv->far_plane`.

Используется подсистемой `Feel::Vision` актёра (зрительная память/запоминание врагов — ит. 6).

## 23. Связи (порц. 30)

- **Порц. 28** — инфраструктура `net_*` SP-актёра и `discord_gameinfo`/`UpdateCL` (контекст `RPC_Update*`); `CActor::RPC_*` декларированы в `Actor.h:875–877`.
- **Порц. 29** — общая `actor_mp_state`/`actor_mp_state_holder`, клиентская пара `fill_state/net_Export/net_Import/process_packet/postprocess_packet`.
- **Порц. 2 (фабрика)** — `ADD_MP(CActor, CActorMP, CSE_ALifeCreatureActor, CSE_ActorMP, ...)` (`object_factory_register.cpp:193/235`).
- **Ит. 11/12** — `CSE_Abstract::STATE_*/UPDATE_*`, `CSE_ALifeCreatureActor` (`m_AliveState`, `o_Position`, `accel`, `o_model`, `o_torso`, `timestamp`, `weapon`, `mstate`, `get_health/set_health`, `fRadiation`), альфейные реестры (`CALifeAbstractRegistry`).
- **Порц. 10 (`CInventoryOwner`)** — `OnReceiveInfo/OnDisableInfo/StartTalk`, Lua-callback `eInventoryInfo`; инвентарь (`InventoryOwner.cpp:452/468/490`) — вызовы `RPC_Update*`.
- **Порц. 22/20** — `on_death`/`Net_Relevant` (health ≤ 0), `get_health`.
- **Lua/скрипты** — `_g.get_actor_true_community`, `ranks.*`, `alun_utils.get_reputation_name`, `level_script.cpp:1184–1186`; `GAME_NEWS_DATA`/PDA/диалоги (`PhraseDialog`, `RelationRegistry`, `MapManager`, `GameTaskManager`).
- **Зрение** — `vision_client`/`Feel::Vision` (ит. 6).

## 24. Краткое резюме (порционы 29–30)

MP-актёра — компактный state-синк через общую `actor_mp_state` (~27 байт вместо 138): клиент собирает state из физики (`fill_state` ← `PHGetSyncItem(0)`), сервер — из полей `CSE` (`fill_state` ← `m_AliveState/o_Position/accel/...`); обе стороны пишут/читают одинаково (`write/read`, бит-паковка health/radiation/slot, `USE_DIFFERENCES` отключён). Клиент: `net_Relevant` (всегда true, кроме removed-физики) → `net_Export`; `net_Import` → health-политика (INVINCIBLE-защита), радиация ×100, слот, камера → `process_packet`/`postprocess_packet` (очереди ≤5 + интерполяция, порц. 28). Сервер: `UPDATE_Write` (lazy `fill_state`), `UPDATE_Read` (при `health <= 0` пакет поглощается временным holder'ом), `Net_Relevant` мёртвых = false, `on_death` фиксирует state. Поведение `CActorMP`: только `eacFirstEye`, камера-инертность 0.7, `GEG_PLAYER_USE_BOOSTER`, античит-дамп `"mp_actor"` (16 float). `RPC_Update*` — на деле Discord-обновления faction/rank/reputation из Lua-функторов (вызовы из `UpdateCL`/`InventoryOwner`/Lua). Дополнительно в порцион: SP-коммуникации (`actor_communication.cpp`: новости, инфо-порции, диалоги, PDA-контакты), статистика (`actor_statistic_mgr`: реестр с миграцией версии `0x0002`), зрительная память (`actor_memory`: `vision_client`, только `CEntityAlive`). Регистрация — `ADD_MP(CActor, CActorMP, ...)` (единственный MP-класс фабрики).
