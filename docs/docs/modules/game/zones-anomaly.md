# Зоны: аномалии, smart-террейн и Lua-зоны (порцион 40)

> **Модуль:** `xrGame` · **Файлы:** `alife_anomalous_zone.cpp` (139), `alife_smart_zone.cpp` (35), `smart_zone.h` (19), `GraviZone.h/.cpp` (86+268), `RadioactiveZone.h/.cpp` (23+161), `AmebaZone.h/.cpp` (26+111), `NoGravityZone.h/.cpp` (15+64), `TorridZone.h/.cpp` (25+78), `FryupZone.h/.cpp` (16+16), `script_zone.h/.cpp` (40+140), `zone_effector.h/.cpp` (28+86), `script_zone_script.cpp` (32), `CustomZone_script.cpp` (15) · **Порцион 40, итерация 5**
>
> **Страница:** серверные (ALife) классы аномалий и smart-террейна, клиентские подклассы `CCustomZone` — грави/радио/амёба/невесомость/раскалённая зона, Lua-зона `CScriptZone`, мост пост-эффекта `CZoneEffector` и экспорт в Lua. Связи: FSM/эффекты → `zones-custom.md` (порц. 37–38), подклассы → `zones-subclasses-2.md` (порц. 39: `CTorridZone`/`CAmebaZone` регистрируются в его скриптовых файлах), ристрикторы → `zones-restrictor.md` (порц. 35), серверные объекты `CSE_ALife*` — ит. 11/12, Lua-колбэки — ит. 14.
>
> **Иерархия:**
> ```
> CSE_Abstract → CSE_ALifeDynamicObject → CSE_ALifeCustomZone
>  ├── CSE_ALifeAnomalousZone        — сервер аномалии (alife_anomalous_zone.cpp)
>  └── CSE_ALifeSmartZone            — сервер smart-террейна (alife_smart_zone.cpp)
>
> CSpaceRestrictor
>  ├── CSmartZone                    — клиент smart-террейна (19 строк, только schedule)
>  └── CScriptZone + Feel::Touch     — Lua-зона: eZoneEnter/eZoneExit
>
> CCustomZone
>  ├── CBaseGraviZone → CGraviZone   — затягивание + телекинез (CTelekinesis)
>  ├── CRadioactiveZone              — постоянный радиационный урон
>  ├── CNoGravityZone                — отключение гравитации
>  └── CMosquitoBald → CTorridZone   — движущаяся по траектории зона
> CVisualZone → CAmebaZone (+ CPHUpdateObject) — замедление существ
>
> CScriptObject → CFryupZone         — пустышка (не зона)
> ```

## 1. Серверные классы

### 1.1 `CSE_ALifeAnomalousZone` (`alife_anomalous_zone.cpp`, 139)

Реализация виртуалей серверного объекта зоны (объявление — `xrServer_Objects_ALife_Monsters.h`, ит. 11/12):

| Метод | Строки | Логика |
|---|---|---|
| `tpfGetBestWeapon(hitType&, hitPower&)` | 22–29 | обнуляет `m_tpCurrentBestWeapon`, фиксирует `m_tTimeID = game_time()`, `fHitPower = 1.0f`, `tHitType = m_tHitType`; **всегда возвращает NULL** (зона не воюет) |
| `tfGetActionType(...)` | 31–35 | всегда `ALife::eMeetActionTypeAttack` (при встрече с зоной — атака) |
| `bfActive()` | 37–40 | `false || !interactive()` → активна только пока неинтерактивна (`interactive()` — из `CSE_ALifeCustomZone`, ит. 12) |
| `tpfGetBestDetector()` | 42–49 | `VERIFY2(false, "This function shouldn't be called")` + `NODEFAULT` — детектор зоне не нужен |
| `on_spawn()` | 130–134 | только делегат (тело `spawn_artefacts()` закомментировано) |
| `keep_saved_data_anyway()` | 136–139 | `true` — данные зоны не выгружаются из памяти |

**Мёртвый код:** `spawn_artefacts()` (51–128) — целиком закомментирована; генерировала артефакты в точке спавна по весовым таблицам `artefacts` (`name/min_start_power/max_start_power/min..max_artefact_count`), расставляла `m_fAnomalyValue = m_maxPower*(1 - d/offline_interactive_radius)` через `spawns().assign_artefact_position`.

### 1.2 `CSE_ALifeSmartZone` (`alife_smart_zone.cpp`, 35)

| Метод | Строки | Логика |
|---|---|---|
| `tpfGetBestWeapon` | 12–16 | всегда NULL |
| `tfGetActionType` | 18–24 | объект на **том же graph-узле**, что и зона → `ALife::eMeetActionSmartTerrain`, иначе `eMeetActionTypeIgnore` — определение «своих» для smart-террейна |
| `bfActive()` | 26–29 | всегда `true` (террейн в расписании постоянно) |
| `tpfGetBestDetector` | 31–35 | `VERIFY2(false, ...)` — недопустимый вызов |

### 1.3 `CSmartZone` (`smart_zone.h`, 19)

Пустой клиентский маркер: `class CSmartZone : public CSpaceRestrictor` + `register_schedule() = true` (17) + `DECLARE_SCRIPT_REGISTER_FUNCTION` (18). Вся логика smart-террейна — в ALife/brain-слое (порц. 15–16, ит. 12), не здесь. Lua: `ce_smart_zone` (§11).

## 2. `CScriptZone` (`script_zone.h/.cpp`, 40+140)

Lua-зона: `CSpaceRestrictor` + `Feel::Touch`, **без FSM** — только колбэки входа/выхода.

- **`net_Spawn`** (35–43): `feel_touch.clear()` + делегат; **`shedule_Update`** (50–58): делегат + `feel_touch_update` по центральной сфере (схема как у зон).
- **`feel_touch_new`** (60–67): `callback(GameObject::eZoneEnter)(this, object)` — Lua-событие «вошёл в зону»; **`feel_touch_delete`** (69–77): `eZoneExit` (пропуск уничтожаемых); **`net_Relcase`** (79–90): если объект был в `feel_touch` → `eZoneExit` (закрытие при удалении объекта).
- **`feel_touch_contact`** (92–95): чистая геометрия `CCF_Shape::Contact(O)`; `IsVisibleForZones() = false` (`.h:34`).
- **`active_contact(u16 id)`** (132–140): есть ли объект с таким ID в `feel_touch` — используется Lua («объект сейчас в зоне?»).
- **`reinit`/`net_Destroy`** — делегаты (30–33, 45–48); **DEBUG `OnRender`** (97–130): cyan-эллипсы сфер и OBB боксов (как у `CTeamBaseZone`, порц. 39).

## 3. Гравитационные зоны (`GraviZone.h/.cpp`, 86+268)

Два класса в одном заголовке: абстрактный `CBaseGraviZone : CCustomZone` (16–70) с чистым виртуалом `virtual CTelekinesis& Telekinesis() = 0` (46) и конечный `CGraviZone` (72–86) с полем `CTelekinesis m_telekinesis`. По замыслу (шапка .h, 1–9): «две зоны в одной» — одна затягивает, другая взрывает и поднимает неживое в воздух.

- **Данные** (48–69): `m_fThrowInImpulse`/`m_fThrowInImpulseAlive` (импульс затягивания), `m_fThrowInAtten`, `m_fBlowoutRadiusPercent` (радиус выброса в %), телекинез `m_fTeleHeight`/`m_dwTimeToTele`/`m_dwTelePause`/`m_dwTeleTime`, партиклы `m_sTeleParticlesBig/Small`.
- **`Load`** (22–44): `throw_in_impulse`, `throw_in_impulse_alive`, `throw_in_atten`, `blowout_radius_percent`, `tele_height`, `time_to_tele`, `tele_pause`, `tele_particles_big/small` (`line_exist`).
- **Жизненный цикл:** `net_Spawn` — делегат (46–49); **`net_Destroy`** (51–55): `Telekinesis().deactivate()` + делегат; **`shedule_Update`** (57–61): делегат + `Telekinesis().schedule_update()`; **`net_Relcase`** (263–268): `Telekinesis().remove_links(O)`.
- **`BlowoutState()`** (64–72): делегат + **всегда** `UpdateBlowout()` + `AffectObjects()` (не дожидаясь, как в базе, отдельных моментов — здесь свой график).
- **`IdleState()`** (75–116): цикл телекинеза: `m_dwTeleTime += dt`; при `> m_dwTimeToTele` — деактивация всех поднятых объектов + `StopTeleParticles`; при `> TimeToTele + TelePause` — сброс счётчика и активация остальных: `Telekinesis().activate(GO, 0.1f, m_fTeleHeight, m_dwTimeToTele)` + `PlayTeleParticles`; в не-Idle → `Telekinesis().deactivate()`.
- **`Affect(O)`** (123–167): направление `throw_in_dir` = к центру зоны (`ThrowInCenter` → `Center`, 169–172), `dist_to_radius = dist/Radius()`; `CanApplyPhisImpulse = GO->Local()`. Ветка затягивания: `CheckAffectField(GO, dist_to_radius)` (`dist_to_radius > m_fBlowoutRadiusPercent` через виртуальный `BlowoutRadiusPercent(GO)`, `.h:44`) **и** `CanApplyPhisImpulse` → `AffectPull` (объект снаружи радиуса выброса и локальный — притягивается). Иначе — ветка выброса: если момент `m_dwBlowoutExplosionTime` ещё **не** попал в `[prev, state)` → `AffectPull` к центру на дистанцию `BlowoutRadiusPercent(GO) * Radius()` (втягивает внутрь радиуса для будущего удара); если попал → `AffectThrow` (стандартный blowout-удар).
- **`AffectPull`** (174–185): живой → `AffectPullAlife`, мёртвый с физ.шеллом → `AffectPullDead`.
  - **`AffectPullAlife`** (187–196): `throw_power = m_fThrowInImpulseAlive * rel_power⁵` (пятикратная степень — резкий рост к центру), `movement()->AddControlVel(dir * power)`.
  - **`AffectPullDead`** (198–201): `PPhysicsShell()->applyImpulse(dir, dist * m_fThrowInImpulse * mass / 100)`.
- **`AffectThrow`** (203–216): стандартный blowout-удар: `power = Power(dist, Radius())`, при `> 0.01` → `CreateHit` + `PlayHitParticles`.
- **`PlayTeleParticles`/`StopTeleParticles`** (219–261): выбор small/big по `SMALL_OBJECT_RADIUS`, `CParticlesPlayer::Start/StopParticles`.

## 4. `CRadioactiveZone` (`RadioactiveZone.h/.cpp`, 23+161)

Постоянный фоновый радиационный урон; **только акторы** и с оговорками для MP.

- **`BlowoutState`** (24–29): делегат + `UpdateBlowout()` пока не переключилось (как `CMosquitoBald`, но без флага).
- **`Affect(O)`** (31–78): квант `one = 0.1f` с; пропуск, если прошло `< 0.1` с с последнего удара (`f_time_affected`); `clamp(f_time_affected, tg-0.3, tg)` (догон); `power = Power(dist, nearest_shape_radius(O))`; при `power < EPS` — только отметка времени; иначе **`send_power = power * 0.1`** и цикл `while (f_time_affected + 0.1 < now)` — догоняет пропущенные кванты, шлёт `CreateHit` с `BI_NONE` и нулевым импульсом (радиация — не толчок). Итого урон линейно пропорционален времени в зоне.
- **`feel_touch_new`** (80–91): в **MP** при входе актора — сразу `CreateHit(power=0)` (комментарий `// ALife::eHitTypeRadiation` — тип берётся из `m_eHitTypeBlowout` секции).
- **`feel_touch_contact`** (95–105): только `CActor` + `CCF_Shape::Contact` + `A->feel_touch_on_contact(this)`.
- **`UpdateWorkload(dt)`** (107–145): в **MP** для каждого актора в зоне собирает локальный `SHit` (`GE_HIT`, `power = Power * dt/1000`, `impulse = 0`) и вызывает **`object->OnEvent(l_P, GE_HIT)` напрямую** (без сервера, локальный апдейт); далее делегат. Комментарий в .h: «related to fast-mode optimizations».
- **`nearest_shape_radius(O)`** (147–161): одна форма → `Radius()`, иначе радиус **первой** сферы `Shapes()[0]` (упрощение — не ближайшей фигуры).

## 5. `CAmebaZone` (`AmebaZone.h/.cpp`, 26+111)

`class CAmebaZone : public CVisualZone, public CPHUpdateObject` — «амёба»: замедляет существ внутри, пока зона активна.

- **`Load`** (23–27): `max_velocity_in_zone` → `m_fVelocityLimit` (дефолт 1.0, ctor 14–17).
- **`SwitchZoneState`** (90–102): вход в Blowout → `CPHUpdateObject::Activate()`, выход → `Deactivate()` (PH-таймер включается только на выброс) + делегат.
- **`BlowoutState()`** (29–39): делегат; пока не переключилось — `UpdateBlowout()`; **плюс всегда `Affect` по каждому объекту** (35–36) — непрерывный урон в Blowout.
- **`Affect(O)`** (41–70): как `CMosquitoBald`, но `power = Power(distance_to_center(O), m_fEffectiveRadius)` (радиус — эффективный, не геометрический).
- **`PhTune(step)`** (72–88): для `CEntityAlive` в пределах `effective_radius(m_fEffectiveRadius)` → `movement()->SetVelocityLimit(m_fVelocityLimit)` (существо ползёт не быстрее лимита); `PhDataUpdate` — пустой (`.h:21`).
- **`distance_to_center(O)`** (104–111): `_sqrt((P.x-OP.x)² + (P.x-OP.x)²)` — **опечатка: X считается дважды, Y/Z не учитывается** (см. замечание 7).

## 6. `CNoGravityZone` (`NoGravityZone.h/.cpp`, 15+64)

`CCustomZone`, выключающий гравитацию у объектов внутри.

- **`enter_Zone`** (11–15): делегат + `switchGravity(io, false)`; **`exit_Zone`** (17–21): `switchGravity(io, true)` + делегат.
- **`UpdateWorkload(dt)`** (23–27): **переопределён без делегата** — каждый тик принудительно `switchGravity(*i, false)` по всем объектам (гравитация удерживается, пока зона fast-mode работает).
- **`switchGravity(io, val)`** (29–64): для активного `CPhysicsShell` → `shell->set_ApplyByGravity(val)`; если выключаем и гравитация осталась («застряла») — толчок случайному элементу: `applyImpulseTrace(random_point, random_dir, mass * Gravity() * fixed_step)` (40–45); для живых без шелла → `movement()->SetApplyGravity(val)` + `SetForcedPhysicsControl(!val)`, при выключении на земле — импульс вдоль `GroundNormal` (57–62).

## 7. `CTorridZone` (`TorridZone.h/.cpp`, 25+78)

`CMosquitoBald`, **двигающаяся по анимированной траектории** (взрывы на маршруте).

- Ctor/dtor (6–14): `xr_new<CObjectAnimator>()`; **`net_Spawn`** (16–29): `CSE_ALifeTorridZone::get_motion()` → `m_animator->Load` + `Play(true)` (зацикленно).
- **`UpdateWorkload(dt)`** (31–37): делегат + `m_animator->Update(dt/1000)` → **`XFORM().set(m_animator->XFORM())`** + `OnMove()` (свет/частицы следуют, §10 `zones-custom.md`).
- **`shedule_Update`** (39–47): делегат + репозиционирование активных звуков на `XFORM().c` (idle/blowout/hit/entrance).
- **`Enable`/`Disable`** (49–67): при включении — `Stop()+Play(true)` аниматора, при выключении — `Stop()`.
- **`IsVisibleForZones() = true`** (`.h:18`) — зона видима для touch-обхода других зон; **`light_in_slow_mode() = false`** (69–73, «Lain: added») — свет гаснет в slow; **`AlwaysTheCrow() = true`** (75–78) — всегда в расписании.
- Lua: `CTorridZone(constructor)` — `MosquitoBald_script.cpp:13` (порц. 39).

## 8. `CFryupZone` (`FryupZone.h/.cpp`, 16+16)

**Не зона:** `class CFryupZone : public CScriptObject` — только ctor/dtor и пустой `OnRender` (`#ifdef DEBUG`). Ничего не делает; в Lua/игре не используется (grep — только объявление). Реликвия (см. замечание 3).

## 9. `CZoneEffector` (`zone_effector.h/.cpp`, 28+86)

Мост «зона → пост-процесс камеры актора» (`m_actor_effector` в `CCustomZone`, §3 `zones-custom.md`).

- **Данные** (`.h:9–19`): `r_min_perc`/`r_max_perc` (проценты радиуса зоны), `m_radius`, `m_factor`, `CPostprocessAnimatorLerp* m_pp_effector`, `m_pp_fname`, публичный `CActor* m_pActor`.
- **`Load(section)`** (23–30): `pp_eff_name` (VERIFY2-есть), `radius_min`, `radius_max` (`VERIFY(min <= max)`).
- **`Activate()`** (32–42): актор = `Level().CurrentEntity()`; создаёт `CPostprocessAnimatorLerp`: `SetType(EEffectorPPType(u32(u64(this) & u32(-1))))` (тип = указатель на effector — уникален на инстанс), `SetCyclic(true)`, **`SetFactorFunc(GET_KOEFF_FUNC(this, &GetFactor))`** (вес вызывается каждый кадр), `Load(pp_fname)`, `Cameras().AddPPEffector`.
- **`Stop()`** (44–51): `RemovePPEffector` по тому же типу, обнуление указателей.
- **`Update(dist, r, hit_type)`** (53–81): `min_r = r*r_min_perc`, `max_r = r*r_max_perc`; выключение при `dist > max_r` / камера не на акторе / актор мёртв; включение при `dist < max_r` и камере на акторе; **`m_factor = (max_r - dist)/(max_r - min_r) - protection`**, где `protection = outfit->GetDefHitTypeProtection(hit_type)` (69–75, броня костюма гасит эффект), clamp `[0.01, 1.0]`.
- **`GetFactor()`** (83–86) → `m_factor`. Вызывается из `CCustomZone::UpdateWorkload` (`m_actor_effector->Update(...)`, §5 `zones-custom.md`); `Stop` — в `Disable`/`net_Destroy`/`net_Relcase`.

## 10. Экспорт в Lua (итерация 14)

| Файл | Регистрация | Lua-имя / члены |
|---|---|---|
| `CustomZone_script.cpp` (15) | `CCustomZone::script_register` (7–15) | `CCustomZone(constructor)` — только конструктор, методов нет (весь функционал — у подклассов, §11.2) |
| `script_zone_script.cpp` (32) | `CScriptZone::script_register` (16–23) | `ce_script_zone(constructor)` |
| там же | `CSmartZone::script_register` (25–32) | `ce_smart_zone(constructor)` |
| — | колбэки `CScriptZone` | `GameObject:eZoneEnter/eZoneExit` (script_zone.cpp:66/76/88) |
| `MosquitoBald_script.cpp` (30, порц. 39) | `CTorridZone`, `CMosquitoBald`, `CZoneCampfire` | конструкторы + `turn_on/off/is_on` |
| `HairsZone_script.cpp` (20, порц. 39) | `CAmebaZone`, `CNoGravityZone` (+`CHairsZone`) | конструкторы |

## 11. Связи

- **База:** `CCustomZone` (FSM/эффекты — порц. 37–38): `BlowoutState`/`IdleState`/`UpdateWorkload`/`UpdateBlowout`/`AffectObjects`/`Affect`/`enter_Zone`/`exit_Zone`/`Enable`/`Disable`/`SwitchZoneState`/`feel_touch_*` — все переопределённые точки; `CSpaceRestrictor` (порц. 35).
- **Подклассы/порц. 39:** `CTorridZone : CMosquitoBald`, `CAmebaZone : CVisualZone` — Lua-регистрация в `MosquitoBald_script.cpp`/`HairsZone_script.cpp`.
- **Сервер (ит. 11/12):** `CSE_ALifeAnomalousZone`/`CSE_ALifeSmartZone`/`CSE_ALifeTorridZone`/`CSE_ALifeCustomZone` (объявления в `xrServer_Objects_ALife_Monsters.h`/`xrServer_Objects_ALife.h`), `interactive()`, `m_owner_id`.
- **Физика:** `CTelekinesis` (`ai/monsters/telekinesis.h`), `CPHUpdateObject` (xrphysics), `CPhysicsShell`/`CPHMovementControl`, `physics_world()->Gravity()`.
- **Эффекты:** `CPostprocessAnimatorLerp`/`Cameras().AddPPEffector` (xrEngine, порц. по эффекторам — ит. 9), `CCustomOutfit::GetDefHitTypeProtection` (порц. 33).
- **События:** `GE_HIT` (ит. 4; локальный `OnEvent` в `CRadioactiveZone::UpdateWorkload`), `Level().CurrentEntity()`.
- **Lua:** `eZoneEnter`/`eZoneExit` (ит. 14, `game_object_space.h`).

## 12. Замечания к плану

1. **`CustomZone_script.cpp` = 15 строк** (план: 14); экспортирует только конструктор `CCustomZone`.
2. **`script_zone_script.cpp` (32) содержит обе регистрации** — `ce_script_zone` и `ce_smart_zone`; в плане упомянут отдельно от `smart_zone.h`, фактически Lua-выход `CSmartZone` именно здесь.
3. **`FryupZone` — не зона**: `CFryupZone : CScriptObject`, пустышка 16+16 (только ctor/dtor/пустой OnRender), в плане перечислена среди аномалий ошибочно.
4. **`GraviZone.h` (86 строк) содержит два класса** — `CBaseGraviZone` + `CGraviZone`; в `GraviZone.cpp` реализован только базовый (план писал только «268»).
5. **Наследование:** `CAmebaZone : CVisualZone + CPHUpdateObject` (не напрямую от `CCustomZone`), `CTorridZone : CMosquitoBald`.
6. **Класс называется `CZoneEffector`** (план — «ZoneEffector»); Lua-имена зон — со смещением `ce_` (`ce_script_zone`, `ce_smart_zone`).
7. **Баг `CAmebaZone::distance_to_center`** (`AmebaZone.cpp:110`): `_sqrt((P.x-OP.x)² + (P.x-OP.x)²)` — вместо Y повторён X; расстояние неверно (по сути, только dx, масштабированный √2).
8. `CRadioactiveZone::nearest_shape_radius` при нескольких формах возвращает радиус `Shapes()[0]`, а не ближайшей (в отличие от `CCustomZone::CalcDistanceTo`).
9. `spawn_artefacts()` в `CSE_ALifeAnomalousZone` — закомментированный мёртвый код (51–128); генерация артефактов из аномалий давно перенесена на уровень спавн/скриптов.
10. **`CTorridZone` переопределяет `typedef CCustomZone inherited`** (`TorridZone.h:9`), хотя сам наследует `CMosquitoBald` (h:6). Из-за этого все `inherited::…`-вызовы в `TorridZone.cpp` идут напрямую в `CCustomZone`, минуя `CMosquitoBald::BlowoutState`/`Affect`/`UpdateSecondaryHit` — вероятно, намеренно (движущаяся зона не должна использовать баллистику «комара»), но формально это «прыжок» через родителя.

## 13. Краткое резюме

Порцион 40 завершает семейство зон. Серверный слой: `CSE_ALifeAnomalousZone` — почти заглушка (attack-встречи, `bfActive = !interactive()`, мёртвый `spawn_artefacts`), `CSE_ALifeSmartZone` — определяет принадлежность объекта к террейну по graph-узлу (`eMeetActionSmartTerrain`), клиентский `CSmartZone` — пустой маркер в расписании. `CScriptZone` — тонкий Lua-мост `CSpaceRestrictor`+`Feel::Touch` с колбэками `eZoneEnter`/`eZoneExit` и `active_contact`. Аномалии: `CBaseGraviZone`/`CGraviZone` (затягивание с `rel_power⁵`, телекинез-цикл Idle с паузами, `AffectPull/AffectThrow`), `CRadioactiveZone` (квант 0.1 с, догон времени, локальный `GE_HIT` в MP), `CAmebaZone` (лимит скорости через `PhTune`, PH-активация только в Blowout), `CNoGravityZone` (`set_ApplyByGravity` + толчки), `CTorridZone` (зонa на `CObjectAnimator`, всегда в расписании). `CZoneEffector` — пост-процесс камеры с весом `(max_r-d)/(max_r-min_r) - защита костюма`. Экспорт в Lua — конструкторы + `turn_on/off`, всё через `*_script.cpp` (порц. 39–40).

---

*Порцион 40 (итерация 5) · следующие: порц. 41–42 `feel.md`.*
