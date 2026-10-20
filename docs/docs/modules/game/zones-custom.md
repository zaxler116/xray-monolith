# Зоны: CCustomZone — FSM, состояние, данные; свет, ветер, grass, частицы (порционы 37–38)

> **Модуль:** `xrGame` · **Файлы:** `CustomZone.h` (378), `CustomZone.cpp` (1764) · **Порционы 37–38, итерация 5**
>
> **Страница:** устройство игровой аномалии `CCustomZone` — базовый класс всех зон. Порцион 37: данные класса (конфиг, состояния, флаги), чтение секции настроек, сетевой слой, тактовое расписание `shedule_Update`/`UpdateWorkload` и конечный автомат состояний `EZoneState`, feel-touch-объекты в зоне, воздействие на объекты, сила/дистанция, сериализация (§1–12). Порцион 38: визуальные/смысловые эффекты — idle/blowout-свет, ветер, interactive grass, частицы, вкл/выкл-цикл, fast/slow-переключение (§13–22). Подклассы зон — порц. 39, аномалии — порц. 40, `Feel::Touch` — порц. 41.
>
> **Связи:** `CSpaceRestrictor` (→ `zones-restrictor.md`, порц. 35), `CZoneEffector`/`zone_effector.cpp` (порц. 40), `CCF_Shape` (xrEngine), `CSE_ALifeCustomZone` (ит. 11/12 — только сигнатуры), `CActor`/`CArtefact`/`CBolt`, Lua-колбэки (ит. 14).

## 1. Архитектура и данные (`CustomZone.h`)

```
CGameObject → CSpaceRestrictor + Feel::Touch
 └── CCustomZone                    — game-зона: FSM + объекты в зоне + эффекты
      ├── CMosquitoBald             — выбросы-«комары»        (порц. 39)
      ├── CVisualZone → CHairsZone  — визуальная зона         (порц. 39)
      ├── CZoneCampfire             — костёр (наследует CMosquitoBald) (порц. 39)
      └── CScriptZone / аномалии    — script/грави/радио и пр. (порц. 40)
```

- Заголовок включает `space_restrictor.h`, `../xrEngine/feel_touch.h`, `script_export_space.h` (3–5); константа `SMALL_OBJECT_RADIUS = 0.6f` (12) — порог «маленького объекта».
- Поля-конфиг (protected, 129–181): `m_owner_id`/`m_ttl` (владелец-спавнер и время жизни в мультиплеере), `m_zone_flags` (`Flags32`), сила `m_fMaxPower`/`m_fSecondaryHitPower`/`m_fAttenuation`/`m_fHitImpulseScale`/`m_fEffectiveRadius`, тип хита `m_eHitTypeBlowout`, `m_eZoneState`, таймеры состояния `m_iStateTime`/`m_iPreviousStateTime`, on/off-тайминги `m_TimeToDisable/Enable/Shift/StartTime`, volumetric-параметры, **Interactive Grass**-блок (`m_BendGrass_*`, `grassbender_id`/`grassbender_frame`, `GrassZoneUpdate()` 183).
- Таблица времён состояний: `StateTimeSVec m_StateTime` (185–189, `svector<int, eZoneStateMax>`): `0` — мгновенно, `-1` — бесконечность, `-2` — не вызывать.
- Частицы/звуки/свет: имена партиклов и `ref_sound` (257–284), `m_pIdleParticles` (287), idle-свет (292–302), blowout-свет (305–315), `OBJECT_INFO_VEC m_ObjectInfoMap` (317–319), `CreateHit` (321–328), `Hit` (331), `Play*`-семейство (335–347), `IsVisibleForZones()` = `false` (349, переопределяется подклассами), `OnMove`/`m_vPrevPos`/`m_dwLastTimeMoved` (351–354), `m_fDistanceToCurEntity` (357), `ef_anomaly_type`/`ef_weapon_type` (359–360).
- Публичное: `IsEnabled()` = `m_eZoneState != eZoneStateDisabled` (210), `ZoneEnable`/`ZoneDisable` (211–212), `ZoneState()` (215), `cast_custom_zone()` (86), `register_schedule() = true` (365), `AlwaysTheCrow`/`o_switch_2_*` (369–371), `light_in_slow_mode()` = `true` (375, private), `DECLARE_SCRIPT_REGISTER_FUNCTION` (377).

### 1.1 `SZoneObjectInfo` (15–34)

Один объект внутри зоны: `object` (`CGameObject*`), флаги `small_object` (`Radius() < 0.6f`), `nonalive_object`, **`zone_ignore`** (объект игнорируется зоной), `particles_vector` (прикреплённые партиклы), `dw_time_in_zone` (мс в зоне), `f_time_affected`. `operator==` сравнивает по `object` (33) — используется в `std::find`.

### 1.2 `EZoneState` (89–101)

| Значение | Комментарий из кода |
|---|---|
| `eZoneStateIdle = 0` | внутри нет активных объектов (зона «спит», но работает on/off) |
| `eZoneStateAwaking` | пробуждение (объект попал в зону) |
| `eZoneStateBlowout` | выброс |
| `eZoneStateAccumulate` | накапливание энергии после выброса |
| `eZoneStateDisabled` | выключена |
| `eZoneStateMax` | счётчик (граница `StateTimeSVec`) |

### 1.3 Флаги `EZoneFlags` (106–127)

`eIgnoreNonAlive`(0) `eIgnoreSmall`(1) `eIgnoreArtefact`(2) — фильтры объектов; `eZoneIsActive`(3) — есть неигнорируемые объекты (выставляется каждый тик); `eBlowoutWind`(4)/`eBlowoutLight`(5)/`eIdleLight`(6) — эффекты; `eBlowoutWindActive`(7) — ветер активен; `eUseOnOffTime`(8) — цикл вкл/выкл; `eIdleLightVolumetric`(9) `eIdleLightShadow`(10) — качество света; `eAlwaysFastmode`(11) — всегда fast (из spawn_ini `fast_mode/always_fast`); **`eFastMode`(12)** — зона обновляется каждый кадр (`UpdateCL`); `eIdleObjectParticlesDontStop`(13); `eAffectPickDOF`(14) — DOF-эффектор при входе; `eIdleLightR1`(15) — разрешить idle-свет на R1; `eBoltEntranceParticles`(16); `eUseSecondaryHit`(17); `eBlowoutDisableIdle`(18).

## 2. ctor / dtor (`CustomZone.cpp`)

- Глобалы (22–25): `WIND_RADIUS = 4*Radius()` (дистанция появления ветра), `FASTMODE_DISTANCE = 100.f` (переход fast/slow), `extern Fvector4 ps_ssfx_int_grass_params_1` (w-компонента — радиус grass-обновления).
- **ctor** (27–60): дефолты `m_fMaxPower=100`, `m_fAttenuation=1`, `m_fEffectiveRadius=1`, hit type `eHitTypeWound`, все свет/частицы = NULL; `m_StateTime.resize(eZoneStateMax)` + нули (43–45); `m_ef_weapon_type = m_owner_id = u32(-1)`; флаги: `eIdleObjectParticlesDontStop`/`eBlowoutWindActive` = FALSE, **`eFastMode` = TRUE** (57); `m_eZoneState = eZoneStateIdle`.
- **dtor** (62–72): `GrassBendersRemoveByIndex(grassbender_id)`, `destroy()` всех шести `ref_sound`, `xr_delete(m_actor_effector)`.

## 3. `Load(LPCSTR section)` (74–365)

Полное чтение секции настроек (после `inherited::Load` — `CSpaceRestrictor`):

| Блок | Строки | Ключи (секция настроек) |
|---|---|---|
| Тайминги игнора | 78–80 | `disable_time`, `disable_time_small`, `disable_idle_time` (s32, мс; `-1` — не гасить) |
| Сила/тип | 81–83 | `hit_impulse_scale`, `effective_radius`, `hit_type` (`ALife::g_tfString2HitType`) |
| Фильтры объектов | 85–87 | `ignore_nonalive`, `ignore_small`, `ignore_artefacts` → флаги |
| Времена FSM | 89–93 | `awaking_time`, `blowout_time`, `accamulate_time` (опечатка в имени — в коде); Idle = `-1` |
| Spatial | 96–97 | `spatial.type |= STYPE_COLLIDEABLE \| STYPE_SHAPE` |
| Interactive Grass (idle/active/blowout) | 102–152 | `bend_grass_idle_*` (anim/str/radius/speed), `bend_grass_whenactive_*` (anim/speed/str), `bend_grass_blowout_duration`/`bend_grass_blowout`/`bend_grass_blowout_speed`/`bend_grass_blowout_radius` — все через `line_exist` с дефолтами |
| Звуки | 155–189 | `idle_sound`, `accum_sound`, `awake_sound`, `blowout_sound`, `hit_sound`, `entrance_sound` → `ref_sound::create(st_Effect, sg_SourceType)` |
| Частицы зоны | 192–205 | `idle_particles`, `blowout_particles`, `blowout_once`, `accum_particles`, `awake_particles` |
| Частицы объектов | 208–225 | `entrance_small/big_particles`, `hit_small/big_particles`, `idle_small/big_particles` (см. замечание 6), `idle_particles_dont_stop` |
| Пост-эффект актора | 227–231 | `postprocess` → `xr_new<CZoneEffector>()->Load(...)` (`m_actor_effector`) |
| Болт | 234–238 | `bolt_entrance_particles` → флаг `eBoltEntranceParticles` |
| Задержки эффектов blowout | 240–294 | `blowout_particles_time`/`blowout_light_time`/`blowout_sound_time`/`blowout_explosion_time` — каждая обрезается по `m_StateTime[eZoneStateBlowout]` с `Msg("! ERROR: ...")` в не-MASTER_GOLD; дефолт 0 |
| Ветер | 296–315 | `blowout_wind` (флаг), `blowout_wind_time_start/peak/end` (`R_ASSERT` start<peak<end, end обрезается), `blowout_wind_power` |
| Прочее | 317–364 | `blowout_disable_idle`; blowout-свет: `blowout_light`, `light_color` (`sscanf "r,g,b"`), `light_range/time/height`, `blowout_light_anim` (`LALib.FindItem`); idle-свет: `idle_light`, `idle_light_range/anim/height`, `idle_light_volumetric*`, `idle_light_shadow`, `idle_light_r1`; `use_secondary_hit` + `secondary_hit_power` (READ_IF_EXISTS); `ef_anomaly_type`, `ef_weapon_type`; `pick_dof_effector` → `eAffectPickDOF` |

## 4. Сетевой слой

| Метод | Строки | Логика |
|---|---|---|
| `net_Spawn(CSE_Abstract*)` | 367–436 | `inherited::net_Spawn` (регистрация ристриктора, → порц. 35); `smart_cast<CSE_ALifeCustomZone*>`; `max_start_power`/`attenuation` из настроек; `m_owner_id` из сервера → при наличии `m_ttl = now + 40000` (40 с, MP-автоудаление), иначе `-1` (379–382); on/off-тайминги из `m_disabled_time/m_enabled_time/m_start_time_shift * 1000`, флаг `eUseOnOffTime` (384–388); создание `m_pIdleLight` (только при не-R2/R3/R4 **или** `eIdleLightR1`; volumetric-ветка 401–408) и `m_pLight` (shadow) (390–419); `setEnabled(TRUE)`, `PlayIdleParticles()`, сброс таймеров, инициализация `m_dwLastTimeMoved`/`m_vPrevPos`; чтение spawn_ini `fast_mode/always_fast` → `eAlwaysFastmode` |
| `net_Destroy()` | 438–459 | `StopIdleParticles`, `inherited::net_Destroy`, `StopWind`, `destroy()` обоих светов, `CParticlesObject::Destroy(m_pIdleParticles)`, `m_actor_effector->Stop()` (451–452), выход всех объектов через `exit_Zone` + `m_ObjectInfoMap.clear()` (454–458) |
| `net_Import`/`net_Export` | 461–469 | пустые делегаты `inherited` (сигнатуры, ит. 11/12) |
| `net_Relcase(CObject*)` | 1453–1468 | выход объекта из `m_ObjectInfoMap` (`exit_Zone` + erase); сброс `m_owner_id` при уходе владельца; `m_actor_effector->Stop()` при уходе актора-носителя эффектора; делегат `inherited` |

## 5. Расписание: `UpdateCL`, `shedule_Update`, `UpdateWorkload`

- **`UpdateCL`** (581–586): только в fast-mode → `UpdateWorkload(Device.dwTimeDelta)` (комментарий: «called only in fast-mode»).
- **`shedule_Update(dt)`** (589–689) — главный тик:
  1. сброс `eZoneIsActive` (591);
  2. если `IsEnabled()`: сфера `CFORM()->getSphere()` + мировой центр `P` → **`feel_touch_update(P, s.R)`** (600);
  3. обход `m_ObjectInfoMap` (604–639): накопление `dw_time_in_zone`; по `disable_time[_small]` для мёртвых → `zone_ignore = true` (614–620); по `disable_idle_time` → `StopObjectIdleParticles` (621–625); **Lua-хук** `_G.CCustomZone_BeforeActivateCallback(this, object)` — результат задаёт `zone_ignore` (627–633, ит. 14); любой неигнорируемый объект → `eZoneIsActive = TRUE` (637–638);
  4. в Idle → `CheckForAwaking()` (641–642);
  5. `inherited::shedule_Update` (644);
  6. **fast/slow-переключение** (647–651): дистанция до актора минус радиус > `FASTMODE_DISTANCE` и не `eAlwaysFastmode` → `o_switch_2_slow()`, иначе `o_switch_2_fast()`; в slow-mode `UpdateWorkload(dt)` вызывается здесь (653–654);
  7. grass: дистанция < `ps_ssfx_int_grass_params_1.w` → `GrassZoneUpdate()`, иначе fade-out/удаление grassbender (656–679) — детали порц. 38;
  8. `UpdateOnOffState()` (682) — цикл вкл/выкл, порц. 38;
  9. MP: `!IsGameTypeSingle() && Local()` и `Device.dwTimeGlobal > m_ttl` → `DestroyObject()` (684–688).
- **`UpdateWorkload(dt)`** (525–578) — логика состояния: сдвиг `m_iPreviousStateTime`/`m_iStateTime`; выключена → только `m_actor_effector->Stop()` и выход (530–535); `UpdateIdleLight()`; **диспетчер FSM** по `m_eZoneState` → `IdleState/AwakingState/BlowoutState/AccumulateState`, `eZoneStateDisabled` — no-op, `default: NODEFAULT` (539–556); дистанция до актора `CalcDistanceTo` (позиция актора `y -= 0.9f`) → `m_actor_effector->Update(dist, radius, m_eHitTypeBlowout)` (558–569); активный blowout-свет → `UpdateBlowoutLight()` (571–572); `eUseSecondaryHit` и состояние не Idle/Disabled → `UpdateSecondaryHit()` (574–577).

## 6. Конечный автомат состояний

| Состояние | Строки | Переходы |
|---|---|---|
| `IdleState()` | 471–476 | только `UpdateOnOffState()`; всегда `false` (будит `CheckForAwaking`) |
| `AwakingState()` | 478–486 | `m_iStateTime >= m_StateTime[eZoneStateAwaking]` → `SwitchZoneState(eZoneStateBlowout)` |
| `BlowoutState()` | 488–509 | по таймеру → `Accumulate`; при `eBlowoutDisableIdle` — пауза/возобновление idle-партиклов (493–506); `m_bBlowoutOnce` → `ZoneDisable()` (497–500) |
| `AccumulateState()` | 511–523 | по таймеру: `eZoneIsActive` → снова `Blowout`, иначе → `Idle` |
| `CheckForAwaking()` | 691–695 | `eZoneIsActive` && Idle → `SwitchZoneState(eZoneStateAwaking)` |

**Переключение** (двухступенчатая, через сетевое событие):

- **`SwitchZoneState(new_state)`** (1310–1322): только на сервере (`OnServer()`) шлёт `GE_ZONE_STATE_CHANGE` с `u8(state)`; локально лишь обнуляет `m_iPreviousStateTime`/`m_iStateTime`. Состояние меняется не здесь!
- **`OnEvent(P, type)`** (1278–1291): `GE_ZONE_STATE_CHANGE` → `r_u8` → **`OnStateSwitch(state)`**; иначе делегат `inherited` (каркас события — ит. 4).
- **`OnStateSwitch(new_state)`** (1293–1308): `Disabled` → `Disable()`, иначе → `Enable()` (свет/частицы, порц. 38); `Accumulate` → `PlayAccumParticles()`, `Awaking` → `PlayAwakingParticles()`; запись `m_eZoneState` и сброс таймеров. Оба метода виртуальны: `OnStateSwitch` переопределяет `CMincer` (Mincer.h:36), `SwitchZoneState` — `CVisualZone` (ZoneVisual.h:15) и `CAmebaZone` (AmebaZone.h:23) (порц. 39).
- **`save`/`load`** (1690–1706): `save` пишет `u8(m_eZoneState)`; `load` восстанавливает **только два** варианта: `eZoneStateDisabled` → Disabled, всё прочее → Idle (промежуточные состояния не сохраняются).

## 7. Объекты в зоне: `Feel::Touch`, `enter_Zone`/`exit_Zone`

- **`feel_touch_new`** (697–733): `smart_cast` на `CGameObject`/`CEntityAlive`/`CArtefact`; заполнение `SZoneObjectInfo` (alive-флаг, `Radius() < SMALL_OBJECT_RADIUS`); `zone_ignore` по `eIgnoreSmall`/`eIgnoreNonAlive`/`eIgnoreArtefact`; `enter_Zone(info)` → `push_back`; при `IsEnabled()` — `PlayEntranceParticles` + `PlayObjectIdleParticles`.
- **`feel_touch_delete`** (735–749): `StopObjectIdleParticles` (если объект не уничтожается), `std::find` → `exit_Zone` + `erase`.
- **`feel_touch_contact`** (751–768): отказы — другая `CCustomZone`, `CBreakableObject`, нет `IKinematics`-визуала, это сам объект, `!IsVisibleForZones()`; затем `((CCF_Shape*)CFORM())->Contact(O)` и делегат `object->feel_touch_on_contact(this)`.
- **`feel_touch_on_contact`** (1577–1583): требует `spatial.type & STYPE_VISIBLEFORAI`, иначе `false`; далее `inherited` (см. порц. 41).
- **`enter_Zone`** (1470–1477): при `eAffectPickDOF` и входе текущего актора → `GamePersistent().SetPickableEffectorDOF(true)`; **`exit_Zone`** (1479–1488): `StopObjectIdleParticles` + снятие DOF.
- **`net_Relcase`** (1453–1468): объект (в т.ч. владелец по `m_owner_id` или актор-носитель `m_actor_effector`) покидает мир → `exit_Zone` + `erase` из `m_ObjectInfoMap`, сброс `m_owner_id` и `Stop` эффектора; делегат `inherited`.

## 8. Воздействие: `AffectObjects`, `Hit`, `CreateHit`

- **`AffectObjects()`** (1180–1197): защита от повтора в том же кадре (`m_dwAffectFrameNum == Device.dwFrame`), пропуск на `Device.dwPrecacheFrame`; обход `m_ObjectInfoMap` → `Affect(&info)` для неуничтоженных. **`Affect(SZoneObjectInfo*)` — пустая виртуальная заглушка** (`.h:220–222`): реальный урон реализуют подклассы (`CMosquitoBald::Affect` и др., порц. 39/40). Вызывается из `UpdateBlowout()` по `m_dwBlowoutExplosionTime` (1221–1228, порц. 38).
- **`Hit(SHit*)`** (1112–1119): зона сама по себе не обрабатывает входящие хиты — только `PlayBulletParticles` в точке `p_in_bone_space` (мировая через `XFORM()`), т.е. визуальный ответ на попадание пули.
- **`CreateHit(...)`** (1428–1451): исходящий хит от зоны, **только на сервере**: при заданном `m_owner_id` `id_from` подменяется владельцем; собирается `SHit` (`GE_HIT`), `whoID = id_from`, `weaponID = this->ID()`, `u_EventSend`. Используется подклассами при `Affect`.
- **`UpdateSecondaryHit()`** — пустая виртуальная заглушка (`.h:200–202`), вызывается из `UpdateWorkload` при `eUseSecondaryHit`; реализация — в подклассах/эффекторах (порц. 39/40).

## 9. Сила, дистанция, ef-типы

| Метод | Строки | Формула/логика |
|---|---|---|
| `effective_radius(nearest_shape_radius)` | 778–781 | `nearest_shape_radius * m_fEffectiveRadius` (радиус ближайшей формы в процентах) |
| `RelativePower(dist, radius)` | 771–776 | `radius < dist` → 0; иначе `1 - attenuation * (dist/radius)²`, обрезка по 0 |
| `Power(dist, radius)` | 783–786 | `m_fMaxPower * RelativePower(...)` |
| `CalcDistanceTo(P, dist, radius)` | 1594–1666 | `R_ASSERT(CFORM()->Type()==cftShape)`; одна форма → дистанция до центра зоны + радиус сферы; иначе перебор всех `CCF_Shape::shape_def`: центр (сфера `P` / бокс `c`) через `XF.transform_tiny` → ближайшая форма; `dist` = до её центра, `radius` = `R` сферы или максимум длин осей бокса |
| `ef_anomaly_type()` / `ef_weapon_type()` | 1417–1426 | геттеры `m_ef_anomaly_type`/`m_ef_weapon_type` (`VERIFY != u32(-1)`); используются эф-системой ит. 3 |

## 10. Прочие методы порциона

- **`AlwaysTheCrow()`** (1585–1592): если зона **не** в Idle/Disabled (т.е. будится/выброс/накопление) либо `eAlwaysFastmode` && `IsEnabled()` → всегда `TRUE` (объект всегда «в кадре» для расписания); иначе делегат `inherited::AlwaysTheCrow()`.
- **`MoveScript(Fvector pos)`** (1271–1276): `XFORM().translate_over(pos)` + `OnMove()` — перемещение зоны из Lua (скрипты, ит. 14).
- **`OnMove()`** (1231–1269): первый вызов — только фиксация `m_dwLastTimeMoved`/`m_vPrevPos`; далее вычисление скорости `(P - P_prev)/dt` → `m_pIdleParticles->UpdateParent(XFORM(), vel)`, перестановка активных светов, обновление позиции grassbender (1262–1267).
- **`register_schedule() = true`** (`.h:365`) — зона в расписании (в отличие от голого `CSpaceRestrictor`).
- **`UpdateSecondaryHit()` — пустая виртуальная заглушка** (`.h:200–202`, тело `{}`), не чисто виртуальная: базовый класс конкретен, подклассы переопределяют её при необходимости (реализация — `CMosquitoBald::UpdateSecondaryHit`, MosquitoBald.cpp:82; порц. 39).
- **`UpdateBlowout()`** (`.h:235`, 1199–1229) — **в базовом классе нигде не вызывается**: её дёргают подклассы из `BlowoutState`/`UpdateWorkload` (`CMosquitoBald` MosquitoBald.cpp:30/35, `CBaseGraviZone` GraviZone.cpp:68, `CHairsZone` HairsZone.cpp:13, `CRadioactiveZone` RadioactiveZone.cpp:27, `CAmebaZone` AmebaZone.cpp:33), а `CVisualZone` **переопределяет** метод (`ZoneVisual.h:17`, cpp:77–79) и зовёт `inherited::UpdateBlowout()`. Для `CCustomZone` это мёртвая точка расширения (детали — порц. 38/39).

## 11. Связи

- **База:** `CSpaceRestrictor` (геометрия, регистрация в `space_restriction_manager` — `zones-restrictor.md`, порц. 35), `Feel::Touch` (обход объектов — порц. 41).
- **Сервер:** `CSE_ALifeCustomZone` (`m_owner_id`, `m_disabled_time/enabled_time/start_time_shift`) — ит. 11/12, порц. 40.
- **Подклассы:** `CMosquitoBald`/`CVisualZone`/`CZoneCampfire`/`CHairsZone` (переопределяют `Load`/`Affect`/`BlowoutState`/`UpdateSecondaryHit`/`SwitchZoneState`) — порц. 39; аномалии (`CGraviZone` и др.), `CScriptZone` — порц. 40.
- **Эффекторы:** `CZoneEffector` (`m_actor_effector`, пост-процесс актора; `Update`/`Stop` — порц. 40), `GamePersistent().SetPickableEffectorDOF` (DOF), `LALib` (анимации света).
- **Актор/объекты:** `CActor` (дистанция, effector), `CEntityAlive`/`CArtefact`/`CBolt`/`CBreakableObject`/`CPhysicsShellHolder`/`CParticlesPlayer` (фильтры и частицы).
- **Игровое окружение:** `g_pGamePersistent->grass_shader_data` / `GrassBenders*` (grass-сгибание, ит. 9), `g_pGamePersistent->Environment().wind_strength_factor` (ветер, порц. 38).
- **Lua:** `_G.CCustomZone_BeforeActivateCallback` (`shedule_Update:631`), `move_script` и прочие обёртки — `CustomZone_script.cpp` (порц. 40, ит. 14).
- **Сеть:** `GE_ZONE_STATE_CHANGE`, `GE_HIT` (ит. 4), `Level().CurrentControlEntity()`.

## 12. Замечания к плану

1. **`reload`/`reinit` в `CCustomZone` отсутствуют** — grep по `CCustomZone::reload|reinit` пуст; в плане упомянуты, фактически есть только `Load` (перезагрузки конфига у зоны нет; возможно, имелись в виду `Load` подклассов).
2. **`EZoneState` в плане описана как «eDisabled/eEnabled/eBlowout…»** — фактически: `Idle/Awaking/Blowout/Accumulate/Disabled/Max` (без «Enabled»).
3. **`generate_orthonormal_basis` — не метод `CCustomZone`**, а `Fvector::generate_orthonormal_basis`, вызывается в `PlayBoltEntranceParticles` (1020).
4. **Диапазон плана «1–900» не покрывает заявленные функции**: `AffectObjects` (1180), `OnMove` (1231), `OnEvent` (1278), `CreateHit` (1428), `net_Relcase` (1453), `enter_Zone`/`exit_Zone` (1470/1479), `feel_touch_on_contact` (1577), `AlwaysTheCrow` (1585), `CalcDistanceTo` (1594), `save`/`load` (1690/1696) — задокументирован весь файл (1–1764); функции света/ветра/grass/частиц отложены в порц. 38 по смыслу.
9. **`load`** (1696–1706) нормализует только два состояния, но **поле `m_eZoneState` устанавливается напрямую** (без сброса таймеров/`OnStateSwitch`) — после загрузки переменные состояния `m_iPreviousStateTime`/`m_iStateTime` остаются от предыдущего состояния.
5. **Кросс-ссылки в плане:** `Hit`/`CreateHit` и `AffectObjects` формально частично относятся к порц. 38 (`UpdateBlowout` вызывает `AffectObjects`); `feel_touch_*` целиком отданы порц. 41 — здесь описаны только реализации `CCustomZone`.
6. **Возможная особенность `Load` (218–222):** `line_exist(section, "idle_small_particles")` читает ключ `idle_big_particles` (и наоборот) — читается корректно только при наличии **обеих** строк; при наличии одной — обращение к несуществующему ключу.
7. **`accamulate_time`** — опечатка в имени ключа настроек, воспроизведена и в коде, и в `.ltx` (менять нельзя — совместимость).
8. `net_Export`/`net_Import` (461–469) — пустые; интересны только как сигнатуры (ит. 11/12).

## 13. Быстрый/медленный режим: `o_switch_2_*` (порцион 38)

- **`o_switch_2_fast()`** (1669–1676): если уже fast — выйти; иначе `eFastMode = TRUE`, `StartIdleLight()`, `PlayIdleParticles(true)`, **`processing_activate()`** — объект начинает получать `UpdateCL` каждый кадр.
- **`o_switch_2_slow()`** (1678–1688): если уже slow — выйти; иначе `eFastMode = FALSE`, при `!light_in_slow_mode()` → `StopIdleLight()`, `StopIdleParticles(true)`, **`processing_deactivate()`** — `UpdateCL` больше не вызывается. (База — `light_in_slow_mode() = true` (`.h:375`), т.е. idle-свет по умолчанию переживает slow; свет гасит только переопределение — `CTorridZone::light_in_slow_mode` (TorridZone.cpp:70), порц. 39.)
- Переключение происходит в `shedule_Update` (647–654): дистанция до актора (по сфере `CFORM`) `> FASTMODE_DISTANCE` (100 м) и нет `eAlwaysFastmode` → slow; иначе fast. В slow-mode `UpdateWorkload` выполняется разово из `shedule_Update`; в fast-mode — каждый кадр из `UpdateCL` (581–586). Плюс `eAlwaysFastmode` задаётся spawn_ini `fast_mode/always_fast` (431–434), а `AlwaysTheCrow()` (1585–1592, §10) форсирует присутствие в расписании при активной зоне.

## 14. Вкл/выкл-цикл: `Enable`/`Disable`, on/off-таймеры

| Метод | Строки | Логика |
|---|---|---|
| `UpdateOnOffState()` | 1520–1548 | только при `eUseOnOffTime`; фаза `t = (now - m_StartTime + m_TimeShift) % (Enable + Disable)`; `t < Enable` → вкл, в Disable-фазе → выкл; из `Disabled` и «пора включиться» → `GoEnabledState()`, из `Idle` и «пора выключиться» → `GoDisabledState()`. Вызывается из `IdleState` (473) и `shedule_Update` (682) |
| `GoEnabledState()` | 1568–1575 | шлёт `GE_ZONE_STATE_CHANGE` + `eZoneStateIdle` (мимо `SwitchZoneState`, см. §9) |
| `GoDisabledState()` | 1550–1566 | шлёт `GE_ZONE_STATE_CHANGE` + `eZoneStateDisabled`; **выгружает все объекты**: `exit_Zone` по каждому, `m_ObjectInfoMap.clear()`, `feel_touch.clear()` |
| `Enable()` | 1324–1340 | если уже включена → `false`; `o_switch_2_fast()`; по всем объектам в зоне `PlayEntranceParticles` + `PlayObjectIdleParticles`; `PlayIdleParticles()`; → `true` |
| `Disable()` | 1342–1360 | если уже выключена → `false`; `o_switch_2_slow()`; `StopObjectIdleParticles` по всем, `StopIdleParticles()`, `m_actor_effector->Stop()`; → `false` (всегда `false` — значение для подклассов) |
| `ZoneEnable()`/`ZoneDisable()` | 1362–1370 | публичные обёртки: `SwitchZoneState(eZoneStateIdle)` / `SwitchZoneState(eZoneStateDisabled)` (переключение через сеть, §9); `ZoneDisable` дёргает `BlowoutState` при `m_bBlowoutOnce` (497–500) |

Времена `m_TimeToDisable/Enable/Shift/StartTime` приходят из `CSE_ALifeCustomZone` в `net_Spawn` (384–388).

## 15. Idle-свет

| Метод | Строки | Логика |
|---|---|---|
| `StartIdleLight()` | 832–843 | только в fast-mode; если `m_pIdleLight`: `set_range(m_fIdleLightRange)`, позиция `Position() + y:m_fIdleLightHeight`, `set_active(true)` |
| `StopIdleLight()` | 845–849 | `set_active(false)` |
| `UpdateIdleLight()` | 851–872 | только fast-mode и активный свет; цвет по кадру `m_pIdleLAnim->CalculateBGR` (**формат BGR → перестановка компонент** в `Fcolor`, 861–863), джиттер радиуса `±0.25*rand`, позиция по `Position()+height` |

Создание света — `net_Spawn` (390–419): только для не-R2/R3/R4-рендера или при `eIdleLightR1`; volumetric-ветка (`set_volumetric`, distance/intensity/quality) при `eIdleLightVolumetric`; параметры из `Load` (339–354). Плейсхолдер анимации — `CLAItem* m_pIdleLAnim` (`LALib.FindItem`, 343–344).

## 16. Blowout-свет

| Метод | Строки | Логика |
|---|---|---|
| `StartBlowoutLight()` | 1121–1135 | fast-mode; `m_fLightTime <= 0` → выйти; `m_fLightTimeLeft = now + m_fLightTime*1000`; цвет/радиус/позиция из конфига, `set_active(true)` |
| `StopBlowoutLight()` | 1137–1141 | `m_fLightTimeLeft = 0`, `set_active(false)` |
| `UpdateBlowoutLight()` | 1143–1178 | пока `m_fLightTimeLeft > now`: остаток `time_k` → `scale = powf(time_k/(m_fLightTime*1000) + EPS_L, 0.15f)` (быстрое затухание); цвет — по `m_pBlowLAnim->CalculateBGR` либо `m_LightColor * scale`; радиус `m_fLightRange * scale`; позиция `Position()+height`. Иначе `StopBlowoutLight` |

Запуск — `UpdateBlowout` по `m_dwBlowoutLightTime` (1206–1208); включение/параметры — `Load` (324–336); создание — `net_Spawn` (413–417).

## 17. Ветер

- **`StartWind()`** (1372–1380): только если `m_fDistanceToCurEntity <= WIND_RADIUS` (`4*Radius()`); `eBlowoutWindActive = TRUE`; текущая сила ветра сохраняется в `m_fStoreWindPower`, `Environment().wind_strength_factor` ограничивается `[0,1]`.
- **`StopWind()`** (1382–1387): снятие флага, возврат `wind_strength_factor` из `m_fStoreWindPower`.
- **`UpdateWind()`** (1389–1415): выход → `StopWind`, если актор ушёл за `WIND_RADIUS` **или** `m_dwBlowoutWindTimeEnd < m_iStateTime`; линейная интерполяция «разгон» (`Start→Peak`, от `m_fBlowoutWindPowerMax` к сохранённой силе) и «затухание» (`Peak→End`), clamp `[0,1]`. Вызывается из `UpdateBlowout` (1218), запуск — по `m_dwBlowoutWindTimeStart` при `eBlowoutWind` (1214–1216); времена и `m_fBlowoutWindPowerMax` читаются в `Load` (296–315).

## 18. `UpdateBlowout()` — временная шкала выброса (1199–1229)

Только fast-mode. Каждый параметр срабатывает, если его момент попал в интервал `[m_iPreviousStateTime, m_iStateTime)` (событие «между тиками»):

| Момент | Действие |
|---|---|
| `m_dwBlowoutParticlesTime` | `PlayBlowoutParticles()` |
| `m_dwBlowoutLightTime` | `StartBlowoutLight()` |
| `m_dwBlowoutSoundTime` | `m_blowout_sound.play_at_pos(Position())` |
| `m_dwBlowoutWindTimeStart` (при `eBlowoutWind`) | `StartWind()` |
| постоянно | `UpdateWind()` |
| `m_dwBlowoutExplosionTime` | **`AffectObjects()`** (§8) + при `m_BendGrass_Blowout` → `GrassBendersAddExplosion(...)` |

Моменты задаются в `Load` (240–294) и обрезаются по `m_StateTime[eZoneStateBlowout]`. **Вызова из `CCustomZone` нет**: `UpdateBlowout()` дёргают подклассы внутри своих `BlowoutState`/`UpdateWorkload` (`CMosquitoBald` MosquitoBald.cpp:30/35, `CBaseGraviZone` GraviZone.cpp:68, `CHairsZone` HairsZone.cpp:13, `CRadioactiveZone` RadioactiveZone.cpp:27, `CAmebaZone` AmebaZone.cpp:33), а `CVisualZone` **переопределяет** его (`ZoneVisual.h:17`, cpp:77–79) и зовёт `inherited::UpdateBlowout()` — не «делегат», а override с пробросом (порц. 39/40). Для базового класса это мёртвая точка расширения (§10, §22).

## 19. Частицы зоны (собственные)

Все `Play*`-функции зоны не работают вне fast-mode (`eFastMode`) — экономия в slow.

| Метод | Строки | Логика |
|---|---|---|
| `ChangeIdleParticles(name, bIdleLight)` | 788–793 | `StopIdleParticles(true)` → смена `m_sIdleParticles` → `PlayIdleParticles(bIdleLight)` |
| `PlayIdleParticles(bIdleLight=true)` | 795–815 | `m_idle_sound.play_at_pos` (если нет feedback); при непустом имени — ленивое создание `m_pIdleParticles` (`CParticlesObject::Create(name, FALSE)`), `UpdateParent(XFORM(), zero_vel)`, `Play(false)`; `bIdleLight` → `StartIdleLight()` |
| `StopIdleParticles(bIdleLight=true)` | 817–829 | `m_idle_sound.stop()`, `Stop(FALSE)` + `CParticlesObject::Destroy(m_pIdleParticles)`, опционально `StopIdleLight()` |
| `PlayAccumParticles()` | 1490–1503 | разовый `CParticlesObject::Create(*m_sAccumParticles, TRUE)` + `m_accum_sound` |
| `PlayAwakingParticles()` | 1505–1518 | разовый Create + `m_awaking_sound` |
| `PlayBlowoutParticles()` | 875–886 | разовый Create; `m_fBlowoutTimeLeft = now + m_BendGrass_Blowout_time` (окно активного grass-сгибания) |

Разовый Create (`TRUE` = auto-destroy) — частицы не висят, в отличие от персистентного `m_pIdleParticles`. Диспетчеризация: `PlayAccumParticles`/`PlayAwakingParticles` — из `OnStateSwitch` (1300–1304, §9); `PlayBlowoutParticles` — из `UpdateBlowout` (§18); idle-частицы — `net_Spawn`/`Enable`/`o_switch_*`.

## 20. Частицы объектов внутри зоны

| Метод | Строки | Логика |
|---|---|---|
| `PlayObjectIdleParticles(obj)` | 1054–1081 | только `CParticlesPlayer`; выбор имени по `Radius() < SMALL_OBJECT_RADIUS` (`m_sIdleObjectParticlesSmall/Big`); `StopParticles(str, BI_NONE, true)` (reset) → `StartParticles(str, (0,1,0), ID())` (без кости); выключена зона → сразу `StopParticles`. Примечание: **не проверяет `eFastMode`** (в отличие от прочих `Play*`) |
| `StopObjectIdleParticles(obj)` | 1083–1110 | пропуск при `eIdleObjectParticlesDontStop` и не-акторе; объект должен быть в `m_ObjectInfoMap`; `StopParticles(str, BI_NONE, true)` |
| `PlayHitParticles(obj)` | 888–916 | `m_hit_sound`; выбор small/big-имени по радиусу; `PP->StartParticles(str, play_bone=GetRandomBone(), (0,1,0), ID())` |
| `PlayEntranceParticles(obj)` | 920–975 | `m_entrance_sound`; направление = нормализованный `PHGetLinearVell` (иначе `(0,1,0)`); `MakeXFORM` по случайной косточке → Create(TRUE) + `UpdateParent(xform, vel)`; при `eBoltEntranceParticles` и `CBolt` дополнительно `PlayBoltEntranceParticles` |
| `PlayBoltEntranceParticles()` | 977–1034 | только сферы `CCF_Shape` (box — `break`): по 10·R точек на сфере со случайными HP-углами, для каждой — Create + `Fvector::generate_orthonormal_basis` (1020) в плоскости скорости |
| `PlayBulletParticles(pos)` | 1036–1052 | `m_entrance_sound` + `Create(*m_sEntranceParticlesSmall, TRUE)` с матрицей `XFORM` и `c = pos` |

Вызовы: `feel_touch_new` (вход — 730–731), `feel_touch_delete`/`exit_Zone` (выход — 740/1481), `Enable` (1335–1338), `Hit` → `PlayBulletParticles` (1118), `net_Spawn` → `PlayIdleParticles`.

## 21. Interactive Grass: `GrassZoneUpdate()` (1708–1764)

Сгибание травы шейдером (`g_pGamePersistent->grass_shader_data`, слот `grassbender_id`).

- Выход, если не заданы ни `m_BendGrass_idle_anim`, ни `m_BendGrass_whenactive_anim` (1710–1711).
- **Окно активности**: если `m_BendGrass_Blowout_time <= -1` → активность = `m_eZoneState != eZoneStateIdle`; иначе `m_fBlowoutTimeLeft > now` (выставляется в `PlayBlowoutParticles`, 885).
- **Целевая анимация**: активна → `m_BendGrass_whenactive_anim` (или idle как fallback); нет → idle (1724–1727).
- Регистрация/обновление слота: `GrassBendersUpdate(ID(), grassbender_id, frame, Position(), idle_radius, ...)` либо `GrassBendersRemoveByIndex` (1730–1733); далее ранний выход при отсутствии слота (1736–1737).
- **Переход анимации**: интенсивность приглушается к 0 (`GrassBenderToValue(..., 7.5f)`), при `<= 0.05` — смена `GData.anim[slot]` (1740–1748).
- Применение параметров: активна → плавный докрут к `m_BendGrass_whenactive_speed/str` (скорость 10.0), иначе к idle-значениям (1751–1763).
- Вызывается из `shedule_Update` при `act_distance < ps_ssfx_int_grass_params_1.w` (656–657); при уходе за радиус — fade-out и удаление слота (659–679). Удаление слота — и в dtor (64). Настройки — `Load` (102–152).

## 22. Замечания к плану (порцион 38)

1. **Диапазон «900–1764» неточен**: `ChangeIdleParticles` (788), `PlayIdleParticles` (795), `StopIdleParticles` (817), `StartIdleLight` (832), `StopIdleLight` (845), `UpdateIdleLight` (851), `PlayBlowoutParticles` (875) начинаются раньше 900.
2. **`UpdateCL`/`UpdateWorkload`** заявлены в порц. 38, но описаны в §5 (порц. 37) как часть FSM-расписания; здесь они упоминаются только в контексте fast/slow (§13).
3. **«аномалии-эффекторы»** в названии порциона — на деле `CZoneEffector` (сам объект пост-эффекта) живёт в `zone_effector.cpp` и документируется в порц. 40; в этом порционе только вызовы `m_actor_effector->Update/Stop` (532–533, 565–568, 1356–1357, 1464–1465).
4. `UpdateBlowout()` в базовом классе **нигде не вызывается** — grep по `UpdateBlowout()` даёт вызовы только в подклассах (`CMosquitoBald`, `CGraviZone`, `CHairsZone`, `CRadioactiveZone`, `CAmebaZone`, `CVisualZone`) — диспетчеризация отдана подклассам (порц. 39); для `CCustomZone` это мёртвая точка расширения.
5. `GoDisabledState`/`GoEnabledState` шлют сетевое событие напрямую (мимо `SwitchZoneState`) — фактически это те же `SwitchZoneState(Disabled/Idle)`, дублирование кода.

## 23. Краткое резюме (порционы 37–38)

`CCustomZone` = `CSpaceRestrictor` (геометрия зоны) + `Feel::Touch` (кто внутри) + FSM `EZoneState` (Idle → Awaking → Blowout → Accumulate ⇄) и эффекты. `Load` целиком описывает поведение из секции настроек; `net_Spawn` поднимает свет/ttl/on-off-тайминги. `shedule_Update` собирает активность объектов (`eZoneIsActive`, Lua-хук игнорирования), будит зону, переключает fast/slow (`FASTMODE_DISTANCE = 100.f`; fast → `UpdateCL` каждый кадр через `o_switch_2_fast` + `processing_activate`) и обновляет grassbender'ы; `UpdateWorkload` диспетчеризует состояния и `CZoneEffector`. Переключение состояния — через сетевое `GE_ZONE_STATE_CHANGE` (`SwitchZoneState` шлёт, `OnStateSwitch` применяет + частицы состояния), `save/load` хранит только Idle/Disabled; цикл вкл/выкл (`UpdateOnOffState` → `GoEnabledState`/`GoDisabledState`) живёт в Idle. Сила удара — парабола `1 - att*(d/r)²` от `effective_radius`; `Affect`/`UpdateSecondaryHit` — виртуальные заглушки для подклассов. Эффекты (§13–22): два источника света с анимациями `CLAItem` и экспоненциальным затуханием blowout, ветер с интерполяцией по трём точкам времени, разовые частицы состояний и персистентные idle/объектные частицы (все — только в fast-mode), interactive grass со слотами и переходами анимаций, `Enable`/`Disable` с перезапуском визуала. Подклассы (порц. 39) добавляют `Affect`/`UpdateBlowout`/`UpdateSecondaryHit`; аномалии и script-зоны — порц. 40.

---

*Порционы 37–38 (итерация 5) · страница `zones-custom.md` завершена. Смежные: порц. 39 `zones-subclasses-2.md`, порц. 40 `zones-anomaly.md`, порц. 41–42 `feel.md`.*
