# Зоны: подклассы `CCustomZone` и скриптовые обёртки (порцион 39)

> **Модуль:** `xrGame` · **Файлы:** `MosquitoBald.h/.cpp` (26+123), `ZoneCampfire.h/.cpp` (32+191), `ZoneVisual.h/.cpp` (20+87), `HairsZone.h/.cpp` (23+77), `team_base_zone.h/.cpp` (39+184), `MosquitoBald_script.cpp` (30), `HairsZone_script.cpp` (20), `CustomOutfit_script.cpp` (70), `searchlight_script.cpp` (29) · **Порцион 39, итерация 5**
>
> **Страница:** конкретные игровые зоны поверх `CCustomZone` (→ `zones-custom.md`, порц. 37–38): «комар» `CMosquitoBald` с реальным `Affect`/secondary-hit, костёр `CZoneCampfire` с вкл/выкл из скрипта и плавным розжигом, визуальная зона `CVisualZone` с анимациями модели, «волосы» `CHairsZone`, пробуждающиеся по скорости; командная база `CTeamBaseZone` (не `CCustomZone` — отдельная ветка `CGameObject`+`Feel::Touch`); Lua-регистрации зон и смежных классов. Связи: `CCustomZone` FSM (порц. 37–38), аномалии `CTorridZone`/`CAmebaZone`/`CNoGravityZone` (порц. 40), `CCustomOutfit`/`CHelmet` (порц. 33–34), `CProjector` (свет, ит. 9), итерация 14 (Lua).
>
> **Иерархия:**
> ```
> CCustomZone
>  ├── CMosquitoBald            — база «ударных» зон: Affect + UpdateSecondaryHit
>  │    └── CZoneCampfire       — костёр: turn_on/turn_off, розжиг 3 c
>  ├── CVisualZone              — видимая зона с цикл. анимациями модели
>  │    └── CHairsZone          — будится по скорости внутри существ
>  ├── CScriptZone / аномалии   — порц. 40
>  └── …
> CGameObject + Feel::Touch
>  └── CTeamBaseZone            — MP-база команды (отдельная ветка)
> ```

## 1. `CMosquitoBald` (`MosquitoBald.h/.cpp`, 26+123)

«Комар» — самая распространённая «ударная» зона: наследует `CCustomZone`, реализует три виртуальных точки расширения.

- **Данные** (`.h:18–23`): `m_bLastBlowoutUpdate` — «blowout обновился один раз после переключения состояния»; ctor (`.cpp:8–12`) ставит `m_fHitImpulseScale = 1.f`. `Load` (18–21) — только делегат `inherited` (без новых ключей).
- **`BlowoutState()`** (24–39): вызывает `inherited::BlowoutState()`; пока состояние не переключилось — каждый тик `UpdateBlowout()` (моменты эффектов срабатывают, §18 `zones-custom.md`); на тике переключения (результат `true` и `m_bLastBlowoutUpdate == false`) — последний `UpdateBlowout()` и флаг в `true`. Т.е. `UpdateBlowout` гоняется ровно в Blowout-состоянии, включая кадр выхода.
- **`Affect(SZoneObjectInfo*)`** (50–80): только `CPhysicsShellHolder` и не `zone_ignore`; центр `P` — мировая центральная сфера; случайное `hit_dir` `(-0.5..0.5, 0..1, -0.5..0.5)` + normalize; `dist = distance_to(P) - Radius()` (обрезка по 0), **`power = Power(dist, Radius())`** (§9 `zones-custom.md`), `impulse = m_fHitImpulseScale * power * GetMass()`; при `power > 0.01` → `CreateHit(...)` (bone 0, `m_eHitTypeBlowout`) + `PlayHitParticles`.
- **`UpdateSecondaryHit()`** (82–123): защита по кадру (`m_dwAffectFrameNum`) и `dwPrecacheFrame` — тот же приём, что в `AffectObjects`; обход `m_ObjectInfoMap`, пропуск не-`CPhysicsShellHolder` и `zone_ignore`; **`power = m_fSecondaryHitPower * RelativePower(...)`** (постоянный урон по нахождению, без жёсткого порога, обрезка `< 0` → `return`), `CreateHit` без частиц. Включается флагом `eUseSecondaryHit` из конфига (`use_secondary_hit`/`secondary_hit_power`) — вызов идёт из `UpdateWorkload` каждый тик вне Idle/Disabled (`zones-custom.md` §5).
- Мёртвый закомментированный `SecondaryHitState()` (41–48).

## 2. `CZoneCampfire` (`ZoneCampfire.h/.cpp`, 32+191)

Костёр поверх `CMosquitoBald`: управляемое (Lua) включение/выключение с визуальным «розжигом».

- **Данные** (`.h:8–13`): `m_pEnablingParticles`/`m_pDisabledParticles` (персистентные, `FALSE`-Create), `m_disabled_sound`, `m_turned_on`, `m_turn_time` (абсолютный конец окна розжига `now + OVL_TIME`, `OVL_TIME = 3000` мс, `.cpp:71`). Ctor (18–22): `m_turned_on = true`, `m_turn_time = 0` (закомментированный глобал `g_zone`, 7–17). Dtor (24–29): `Destroy` обоих партиклов + `destroy` звука. `Load` (31–34) — делегат.
- **`GoEnabledState()`** (36–53): `inherited` (шлёт `GE_ZONE_STATE_CHANGE` → Idle); гасит/уничтожает `m_pDisabledParticles`, `stop+destroy` `m_disabled_sound`; создаёт персистентные `enabling_particles` (ключ секции) с `UpdateParent`+`Play`.
- **`GoDisabledState()`** (55–69): `inherited` (→ Disabled, выгружает объекты); `R_ASSERT(NULL == m_pDisabledParticles)`; создаёт `disabled_particles` + `disabled_sound` (`play_at_pos`, loop).
- **Скриптовое управление** (73–96): `turn_on_script`/`turn_off_script` — **только при `rsR2|rsR3|rsR4`** (в R1 костёр не переключается), ставят `m_turn_time = now + 3000`, флаг `m_turned_on`, вызывают `Go*State`; `is_on()` → `m_turned_on`. Lua: `turn_on`/`turn_off`/`is_on` (`MosquitoBald_script.cpp:26–28`).
- **`shedule_Update(dt)`** (98–113): если выключена и идёт окно розжига — принудительно `UpdateWorkload(dt)` (иначе выключенная зона не обновилась бы); idle-частицам передаётся скорость ветра `wind_blast_direction * wind_strength_factor` (огонь колышется); далее `inherited::shedule_Update`.
- **Плавный розжиг** — `UpdateWorkload(u32 dt)` (143–191): `inherited` + пока `m_turn_time > now` коэффициент `k = (m_turn_time - now)/3000`; включение → `k = 1-k`, `PlayIdleParticles(true)` + `StartIdleLight()`, выключение → `StopIdleParticles(false)`; цвет и радиус idle-света масштабируются на `k` (161–177, анимация цвета по `m_pIdleLAnim->CalculateBGR`); по окончании окна (`m_turn_time != 0`) — сброс в 0 и финальный Play/Stop (179–190).
- **`PlayIdleParticles`/`StopIdleParticles`** (116–133): `inherited` вызывается только когда розжиг почти завершён: Play — при `m_turn_time == 0` или остаток `< 3000-2000` (= прошло ≥1 с; заодно гасит `m_pEnablingParticles`), Stop — при остатке `< 3000-500` (прошло ≥2.5 с). Т.е. enabling-частицы живут первые ~1–2 с перехода.
- **`AlwaysTheCrow()`** (135–141): `TRUE`, пока идёт окно розжига (`m_turn_time != 0`), иначе делегат.

## 3. `CVisualZone` (`ZoneVisual.h/.cpp`, 20+87)

Зона с собственной моделью и циклическими анимациями.

- **Данные** (`.h:7–10`): `MotionID m_idle_animation`/`m_attack_animation`, `u32 m_dwAttackAnimaionStart/End` (опечатка в имени — в коде).
- **`net_Spawn`** (18–55): `inherited`; `CSE_ALifeZoneVisual` → `attack_animation`/`startup_animation` → `IKinematicsAnimated::ID_Cycle_Safe` с `VERIFY2` (сообщение об отсутствии анимации в модели); `PlayCycle(m_idle_animation)`; **`setVisible(TRUE)`** — зона видима (в отличие от голого ристриктора).
- **`SwitchZoneState(new_state)`** (57–66): при выходе из Blowout (`m_eZoneState == Blowout && new != Blowout`) → снова `PlayCycle(m_idle_animation)`; далее `inherited` (через сеть, §9 `zones-custom.md`).
- **`Load`** (68–75): `attack_animation_start`/`attack_animation_end` (u32, моменты внутри Blowout), `VERIFY2(start < end)`.
- **`UpdateBlowout()`** (77–87): `inherited` (свет/ветер/частицы/`AffectObjects`) + по интервалу `[m_iPreviousStateTime, m_iStateTime)` переключение на attack-цикл в `start` и возврат на idle-цикл в `end`. Вызывается из… самого себя не зовут — как и в базе, `UpdateBlowout` дёргает подкласс; у `CVisualZone` прямого вызова в `BlowoutState` нет (см. замечание 5).

## 4. `CHairsZone` (`HairsZone.h/.cpp`, 23+77)

«Волосы/щупальца» — `CVisualZone` + пробуждение по скорости существ.

- **Данные**: `float m_min_speed_to_react` (`.h:18`); `Load` (39–43) читает `min_speed_to_react`.
- **`CheckForAwaking()`** (18–37): вместо базового «есть неигнорируемый объект» — перебор `m_ObjectInfoMap`, для `CEntityAlive` берётся **фактическая скорость** `character_physics_support()->movement()->GetVelocityActual()`; при `sp > m_min_speed_to_react` → `SwitchZoneState(eZoneStateAwaking)` и выход. Зона будится только от движущихся существ.
- **`BlowoutState()`** (10–16): как у `CMosquitoBald`, но без флага-одноразовика: пока не переключилось — каждый тик `UpdateBlowout()`.
- **`Affect`** (45–77): калька `CMosquitoBald::Affect` с двумя отличиями: `P.y = pGameObject->Position().y` (64 — горизонтальное расстояние, без вертикальной разницы) и **`dist` не обрезается по 0**: `Power(distance_to(P), Radius())`; порог и `CreateHit`+`PlayHitParticles` те же.

## 5. `CTeamBaseZone` (`team_base_zone.h/.cpp`, 39+184)

**Не** `CCustomZone`: отдельная ветка `CGameObject` + `Feel::Touch` (шапка — Dmitriy Iassenev, 2004). MP-зона базы команды.

- **Данные/интерфейс** (`.h`): `u8 m_Team`, `reinit()` (33–36 — делегат), `Center`/`Radius` (38–46 — как у ристриктора: мировая центральная сфера/`CFORM()->getRadius()`), `GetZoneTeam()` (35), DEBUG `OnRender` (36–38).
- **`net_Spawn`** (48–96): `xr_new<CCF_Shape>` → `collidable.model`; `CSE_ALifeTeamBaseZone` (`R_ASSERT`); `feel_touch.clear()`; по `shapes` — `add_sphere`/`add_box`; `m_Team` из сервера; `inherited::net_Spawn` → `ComputeBounds`, `setEnabled(TRUE)`; **в MP** (`GameID() != eGameIDSingle && !g_dedicated_server`): `MapManager().AddMapLocation("mp_team_base_%d_location", ID())->EnablePointer()` (88–93).
- **`net_Destroy`** (98–104): `MapManager().OnObjectDestroyNotify(ID())` (кроме dedicated) + `inherited`.
- **`shedule_Update`** (106–114): `inherited` + `feel_touch_update(P, s.R)` по сфере (то же, что у зон).
- **`feel_touch_new`/`feel_touch_delete`** (116–141): на сервере при входе/выходе **актора** — `GE_GAME_EVENT` с `GAME_EVENT_PLAYER_ENTER_TEAM_BASE`/`GAME_EVENT_PLAYER_LEAVE_TEAM_BASE`, `w_u16(object ID)`, `w_u8(team)` (`u_EventSend`, `net_flags(TRUE,TRUE)`). Всё состояние базы живёт на сервере по этим событиям (ит. 4).
- **`feel_touch_contact`** (143–148): только `CActor` + `CCF_Shape::Contact(O)`.
- **DEBUG `OnRender`** (150–184): по флагу `dbg_draw_teamzone` — эллипсы сфер cyan и OBB боксов (полубокс `0.5`).

## 6. Скриптовые обёртки (Lua, итерация 14)

### 6.1 `MosquitoBald_script.cpp` (30)

| Регистрация | Экспорт в Lua |
|---|---|
| `CMosquitoBald::script_register` (9–18) | `CTorridZone(constructor)` и `CMosquitoBald(constructor)` (база — `CGameObject`) |
| `CZoneCampfire::script_register` (20–30) | `CZoneCampfire`: `constructor`, `turn_on` → `turn_on_script`, `turn_off` → `turn_off_script`, `is_on` |

### 6.2 `HairsZone_script.cpp` (20)

`CHairsZone::script_register` (9–20): `CHairsZone`, **`CAmebaZone`**, **`CNoGravityZone`** (обе — аномалии, порц. 40; в Lua выведены именно здесь).

### 6.3 `CustomOutfit_script.cpp` (70)

- Хелперы-мосты для enum-аргумента (7–25): `GetDefHitTypeProtection_CurrentOutfit/Helmet`, `GetHitTypeProtection_CurrentOutfit/Helmet` (приводят `int hit_type` → `ALife::EHitType`, строковый элемент → `s16`).
- **`CCustomOutfit::script_register`** (28–51): `CCustomOutfit(constructor)`; поля `def_readwrite`: `m_fPowerLoss`, `m_additional_weight/2`, `m_fHealthRestoreSpeed`, `m_fRadiationRestoreSpeed`, `m_fSatietyRestoreSpeed`, `m_fPowerRestoreSpeed`, `m_fBleedingRestoreSpeed`; `def_readonly`: `bIsHelmetAvaliable`, `bIsBackpackAvaliable`; методы: `BonePassBullet`, `get_HitFracActor`, `get_artefact_count`, `GetDefHitTypeProtection`/`GetHitTypeProtection` (через хелперы), `GetBoneArmor`.
- **`CHelmet::script_register`** (53–70): аналогично (без веса/артефактов): `m_fPowerLoss` + restore-скорости, `get_HitFracActor`, защита, `GetBoneArmor`.

### 6.4 `searchlight_script.cpp` (29)

Целиком под **`#ifdef PROJECTOR_NEW`** (3, определён в `build_config_defines.h:49` — «Upgrade CProjector», т.е. активен). `CProjector::script_register` (11–28): `CProjector(constructor)`, enum `projector_enum` (`eActive`, `eSwitch`, `eDesiredPos`, `eDesiredDir`, `eDesiredAng`), методы `Action`, `SetParam`. Файл про **проектор** (`searchlight.h`), а не про класс `CSearchlight`.

## 7. Связи

- **База:** `CCustomZone` FSM/эффекты → `zones-custom.md` (порц. 37–38): `Affect`/`UpdateSecondaryHit`/`UpdateBlowout`/`BlowoutState`/`CheckForAwaking`/`SwitchZoneState`/`Go*State`/`Play*` — виртуальные точки, переопределённые здесь.
- **Аномалии:** `CTorridZone` (MosquitoBald_script), `CAmebaZone`/`CNoGravityZone` (HairsZone_script) — реализации в `TorridZone.cpp`/`AmebaZone.cpp`/`NoGravityZone.cpp` → порц. 40.
- **Одежда:** `CCustomOutfit`/`CHelmet` → порц. 33–34 (`CHelmet` §12 в `actor-misc.md`); Lua-поля пересекаются с `GetRestoreSpeed` (порц. 33).
- **Сеть/события:** `GE_ZONE_STATE_CHANGE` (порц. 37), `GE_GAME_EVENT` + `GAME_EVENT_PLAYER_ENTER/LEAVE_TEAM_BASE` (ит. 4), `MapManager` (ит. 9), `CSE_ALifeZoneVisual`/`CSE_ALifeTeamBaseZone` (ит. 11/12).
- **Среда:** `GamePersistent().Environment().wind_blast_direction/wind_strength_factor` (ветер, порц. 38), `rsR2|rsR3|rsR4` (`psDeviceFlags`), `LALib` (idle-свет костра).
- **Lua-колбэки:** `_G.CCustomZone_BeforeActivateCallback` работает для всех подклассов (порц. 37).

## 8. Замечания к плану

1. **`mosquito_bald_script.cpp` + «ZoneCampfire-скрипт» = один файл** `MosquitoBald_script.cpp` (30 строк): обе регистрации (`CMosquitoBald`+`CTorridZone` и `CZoneCampfire`).
2. **Размеры:** `ZoneVisual.h` = 20 (план писал только «87» — cpp); `HairsZone_script.cpp` = 20; `searchlight_script.cpp` = **29** (не 28); `team_base_zone.h` = 39.
3. **`searchlight_script.cpp` регистрирует `CProjector`, не «searchlight»**, и весь включён только при `PROJECTOR_NEW` (defined, `build_config_defines.h:49`).
4. **`CVisualZone` и `CTeamBaseZone` не имеют Lua-регистрации** (grep по `script_register` пуст) — из Lua недоступны напрямую.
5. **`CVisualZone::UpdateBlowout` нигде не вызывается из её собственного кода**: в отличие от `CMosquitoBald`/`CHairsZone`, у `CVisualZone` нет переопределённого `BlowoutState`, а базовый `CCustomZone` `UpdateBlowout` не дёргает → анимации attack/возврата, возможно, не работают (или подключаются через иной путь — не найден). Фактически это потенциальный баг/мёртвый код.
6. **`CTeamBaseZone` не наследует `CCustomZone`** — в плане перечислен в том же блоке, но это независимая ветка (`CGameObject` + `Feel::Touch`), без FSM и настроек зон.
7. `CustomOutfit_script.cpp` — про экипировку, а не про зоны; в порционе 39 по плану (кросс-ссылка на порц. 33–34 сохранена).
8. В `CHairsZone::Affect` расстояние считается по горизонтали (`P.y` подменяется) и **не обрезается по 0** — отличие от `CMosquitoBald::Affect`, отмечено в §4.

## 9. Краткое резюме

Порцион 39 закрывает визуально-поведенческие подклассы `CCustomZone`: `CMosquitoBald` реализует боевую математику (`Affect` = одиночный blowout-хит по `Power`, `UpdateSecondaryHit` = постоянный `m_fSecondaryHitPower`) и гоняет `UpdateBlowout` строго в Blowout; `CZoneCampfire` добавляет Lua-переключение `turn_on/off` (только на R2–R4), 3-секундный розжиг со светом, масштабируемым на коэффициент `k`, и ветровые idle-частицы; `CVisualZone` оживляет зону `PlayCycle`-анимациями из `CSE_ALifeZoneVisual` (видимость, attack-окно внутри Blowout); `CHairsZone` будится по фактической скорости существ (`min_speed_to_react`). `CTeamBaseZone` — отдельная MP-ветка: `CCF_Shape`-геометрия, карта-метка команды и `GE_GAME_EVENT` ENTER/LEAVE_TEAM_BASE при входе актора. Lua-слой: конструкторы зон + `turn_on/off/is_on`, полный экспорт `CCustomOutfit`/`CHelmet` (поля восстановления, защиты) и `CProjector` (под `PROJECTOR_NEW`).

---

*Порцион 39 (итерация 5) · следующие: порц. 40 `zones-anomaly.md`, порц. 41–42 `feel.md`.*
