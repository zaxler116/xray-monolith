# CBaseMonster: настройки, ауры, berserk, крит. раны

> Источники: `base_monster_startup.cpp` (552), `base_monster_feel.cpp` (362),
> `base_monster_inline.h` (11), `monster_aura.h` (45), `monster_morale.*`, `melee_checker.*`.
> Порцион 4, итерация 6.

Порц. 2 дал **каркас**, порц. 3 — **циклы**. Этот — **данные и «телесные» реакции** монстра:
что читается из `.ltx` (настройки base/current, protections, auras, berserk, panic threshold),
и как монстр реагирует на урон (крит. раны), ведёт себя по морали и проверяет ближний бой.
Это «настройки + физиология», а не мозг (FSM, порц. 18).

## 1. `SMonsterSettings`: base vs current

Структура `SMonsterSettings` (ai_monster_shared_data.h:4) — «характер» вида. Два слота:

- `m_base_settings` — **заполняется один раз** при загрузке класса (`settings_load`, порц. 2 §6).
- `m_current_settings` — **накатывается на спавне**: `settings_overrides()`
  (startup:489) берёт base и поверх читает секцию `settings_overrides` **из `spawn_ini()`**
  (локация/конкретный экземпляр), создаёт `m_current_settings` по `crc32`.

Оба — `ref_smem` (shared-память на CRC). Точка вызова `settings_overrides()` — **в `net_Spawn`**
(startup:334, сразу после `register_member`). Итог: один класс — разное поведение на разных
картах без кода. `db()` → текущие настройки (геттер).

## 2. `net_Spawn` / `net_Destroy` (точка «оживания»)

`net_Spawn` (startup:334) — что реально происходит, когда ALife «создаёт» монстра:

```cpp
if (!inherited::net_Spawn(DC)) return FALSE;
// R_ASSERT: есть AI-Map / level graph / cross table
monster_squad().register_member(g_Team(), g_Squad(), g_Group(), this);
settings_overrides();                       // ← m_current_settings (§1)
if (GetScriptControl()) { control().animation().reset_data(); ProcessScripts(); }
m_pPhysics_support->in_NetSpawn(e);
control().update_frame();
control().update_schedule();
```

`net_Destroy` (startup:388): `m_controlled->on_destroy`, `StateMan->critical_finalize()`,
`inherited::net_Destroy`, `m_pPhysics_support->in_NetDestroy`, `monster_squad().remove_member`.

## 3. Protections (защита)

В `Load` (startup:139–146): `m_fSkinArmor` (0), `m_fHitFracMonster` (0.1); если секция
`protections_sect` — читает `skin_armor` и `hit_fraction_monster`. Это «броня кожи» и
доля урона, идущая на монстра (остальное — на физобъект). Используется в `Hit`/`HitSignal`
(ит. 5, wounds-death).

## 4. Auras (влияние на игрока)

4 инстанса `monster_aura` (base_monster.cpp:61): `m_psy_aura`, `m_radiation_aura`, `m_fire_aura`,
`m_base_aura` (имена `"psy"/"radiation"/"fire"/"base"`). Каждый (monster_aura.h) —
линейный/квадратный коэффициент, `m_max_power`, `m_max_distance`, `m_pp_*` (пост-процесс),
детектор-звук. Читается в `Load` (startup:131–134) через `load_from_ini`.

```cpp
float CBaseMonster::get_psy_influence()      { return m_psy_aura.calculate(); }   // base_monster.cpp:1007
float CBaseMonster::get_radiation_influence(){ return m_radiation_aura.calculate(); }
float CBaseMonster::get_fire_influence()     { return m_fire_aura.calculate(); }
void   CBaseMonster::play_detector_sound()   { /* все три детектора */ }
```

Жизненный цикл: `update_schedule()` — в `shedule_Update` (порц. 3 §1) **каждый кадр** (ауры
интерполируются часто); `on_monster_death()` — гашение. Это «aura» кривососа/радиации/огня —
влияет на HUD-детектор игрока по дистанции.

## 5. Berserk и panic threshold

- **Berserk** (base_monster.h:367–369): `u32 time_berserk_start` + `set_berserk()`
  (`time_berserk_start = time()`) + `bool berserk_always`. `berserk_always` читается из
  `Load` (`berserk_always`, startup:91). Точки вызова `set_berserk` — в Lua
  (`CScriptGameObject::berserk`) и в FSM-состояниях вида (bloodsucker `reselect_state`/
  `check_force_state`/`check_completion`). FSM читает `time_berserk_start`/`berserk_always`,
  решая «в ярости или нет».
- **Panic threshold** (base_monster_inline.h:3–11): `m_default_panic_threshold`
  фиксируется в `reload` (startup:270, = `m_panic_threshold` вида).
  - `set_custom_panic_threshold(value)` — `m_panic_threshold = value` (из Lua).
  - `set_default_panic_threshold()` — вернуть к дефолту вида.

Оба — **мост Lua → FSM**: Lua меняет порог паники/ярость, а FSM (порц. 18) их интерпретирует.

## 6. Критические раны

Типы (base_monster.h:469): `critical_wound_type_head / torso / legs`. Механика — в
`CCustomMonster` (ит. 5, wounds-death) + `CBaseMonster`:

- `load_critical_wound_bones()` (startup:513): из секций `critical_wound_anim_head/torso/legs`
  → `fill_bones_body_parts(...)` кладёт `bone_id → wound_type` в `m_bones_body_parts` и
  хранит имя анимации. `fill_bones_body_parts` (startup:535) читает секцию с костьми и
  `kinematics->LL_BoneID(...)`.
- `update_critical_wounded(bone_id, power)` (в `CCustomMonster`) решает, прилетел ли крит.
- `critical_wound_external_conditions_suitable()` (feel:332) — у монстра:
  **`check_start_conditions(eControlSequencer)` + `anim().IsStandCurAnim()`** (только стоя,
  контроллер-секвенсер свободен).
- `critical_wounded_state_start()` (feel:342): по `m_critical_wound_type` берёт анимацию
  (`m_critical_wound_anim_head/torso/legs`) и `com_man().critical_wound(anim)` — запускает
  через custom-менеджмент (порц. 21).

(Сравнение: у `CAI_Stalker` `load_critical_wound_bones` — 6 частей head/torso/hands/legs, а
`critical_wounded_state_start` ставит свойство `eWorldPropertyCriticallyWounded` в планировщик,
не анимацию — порц. 15.)

## 7. Мораль (`CMonsterMorale`)

`CMonsterMorale` (monster_morale.h) — «настройка боевого духа», поле `Morale` (конструктор).
Три состояния `eStable/eTakeHeart/eDespondent`, `m_morale ∈ [0,1]`:

- `on_hit()` → `change(-m_hit_quant)` (падает дух); `on_attack_success()` → `change(+m_attack_success_quant)`.
- `update_schedule(dt)` (monster_morale.cpp:35) — зовётся в `shedule_Update` (порц. 3):
  `m_morale += cur_v * dt/1000` (скорость зависит от состояния), `clamp(0,1)`.
- Параметры из `load`: `Morale_Hit_Quant`, `Morale_Attack_Success_Quant`,
  `Morale_Take_Heart_Speed`, `Morale_Despondent_Speed`, `Morale_Stable_Speed`,
  `Morale_Despondent_Threashold`. `reinit` → `eStable`, `m_morale=1`.

FSM читает `Morale.get_morale()`/`is_despondent()`, чтобы «паниковать/отступить».

## 8. Ближний бой (`CMeleeChecker`)

`CMeleeChecker` (melee_checker.h) — поле `MeleeChecker`, `load` в `Load`, `init_attack` в
`reinit` (startup:312). Логика:

- Параметры: `m_min_attack_distance`, `m_max_attack_distance`, `m_as_min_dist`, `m_as_step`
  («as» — aim step), `m_hit_stack[HIT_STACK_SIZE]` (буфер промахов/попаданий).
- `can_start_melee(enemy)` / `should_stop_melee(enemy)` — FSM спрашивает, можно ли начать/
  продолжать ближнюю атаку.
- `distance_to_enemy(enemy)` — RayQuery от `fire_bone` до центра врага (реальная дистанция
  с учётом рельефа), `on_hit_attempt(bool)` — учитывает попадания для подбора дистанции.

Связь с FSM: состояние `eStateAttack` (порц. 19) опрашивает `MeleeChecker`, а
`TranslateActionToPathParams` (порц. 3 §4) — с `ACT_*`.

## 9. `CControlManagerCustom` — ability-менеджер (мост)

`m_com_manager` (порц. 2) — «_ability»-слой. В `update_schedule`
(control_manager_custom.cpp:168) каждый кадр проверяет, не пора ли **самовольно** запустить
движение-ability:

```cpp
if (m_threaten)      check_threaten();
if (m_jump)          check_attack_jump();     // прыжок на врага (eControlJump)
if (m_rotation_jump) check_rotation_jump();   // прыжок с поворотом (случайный из m_rot_jump_data)
if (m_run_attack)    check_run_attack();      // атака на бегу
if (m_melee_jump)    check_melee_jump();      // прыжок в ближний бой
```

Каждый `check_*` → `m_man->capture(this, type)` + `m_man->activate(type)` (через
`CControl_Manager`, порц. 21). Т.е. ability-флаги (порц. 2 §5) здесь **проявляются**:
FSM не обязан явно запускать прыжок/атаку-на-бегу — менеджер сам срабатывает по условиям
(враг виден, контроллер свободен). Это ключевая точка, где «настройка» превращается в
«поведение» без прямого кода FSM.

## 10. Связи

```mermaid
graph TD
    SPAWN[net_Spawn: register_member, settings_overrides] --> SET[SMonsterSettings base→current]
    LTX[Load: protections, auras, berserk_always, MeleeChecker.load] --> DATA[Настройки]
    SCHED[shedule_Update: 4× aura.update_schedule, Morale.update_schedule, com_man.update_schedule] --> AURA[monster_aura]
    SCHED --> MOR[CMonsterMorale]
    SCHED --> COM[CControlManagerCustom: check_threaten/jump/rotation/run/melee]
    COM --> CTRL[CControl_Manager, порц. 21]
    HIT[Hit/HitSignal: protections, critical_wound] --> CRIT[com_man().critical_wound(anim)]
    MELEE[CMeleeChecker: can_start/should_stop_melee] --> FSM[FSM eStateAttack, порц. 19]
    BERSK[set_berserk / panic threshold: Lua→FSM] --> FSM
```

## 11. Замечания к плану

1. **`settings_overrides()` вызывается в `net_Spawn`, не в `Load`** — важно: `m_current_settings`
   появляется только при «оживании» конкретного экземпляра, а не при загрузке класса.
2. `SMonsterSettings` — `ref_smem` по `crc32`; переопределение идёт **по целому блоку**
   (секция `settings_overrides` из `spawn_ini`), а не поле-в-поле.
3. Крит. раны у монстра **только 3 типа** (head/torso/legs) и срабатывают только в стойке
   (`IsStandCurAnim` + `eControlSequencer` свободен); у сталкера — 6 частей и через
   планировщик. Не переносить модель сталкера на монстров.
4. `CControlManagerCustom::update_schedule` — **самостоятельные** ability (прыжок/атака-на-бегу/
   рывок), не требующие явного вызова из FSM. Это «скрытая» часть поведения — держать в виду
   при разборе конкретных видов (порц. 5–9).

## 12. Краткое резюме

Порц. 4 — «физиология и настройки» `CBaseMonster`: `SMonsterSettings` base/current
(override в `net_Spawn`), protections, 4 ауры (каждый кадр), berserk/panic threshold (Lua→FSM),
крит. раны (3 типа, через `com_man()`), мораль `CMonsterMorale` (3 состояния) и
`CMeleeChecker` (ближний бой). Плюс `CControlManagerCustom` — ability-менеджер, который сам
запускает прыжок/атаку-на-бегу по условиям. Всё это питает FSM (порц. 18) и контроллеры
(порц. 21).

---

*Порцион 4 (итерация 6). Далее: порц. 5 — конкретные монстры, первый вид ([bloodsucker.md](bloodsucker.md)).*
