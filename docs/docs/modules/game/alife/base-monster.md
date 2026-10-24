# CBaseMonster: каркас и abilities

> Источники: `src/xrGame/ai/monsters/basemonster/` (12 файлов, 3 754 строки) — в основном
> `base_monster.h` (686), `base_monster.cpp` (1 111), `base_monster_startup.cpp` (552),
> `base_monster_feel.cpp` (362). Порцион 2, итерация 6.

`CBaseMonster` — **ядро всех «живых» монстров** в xrGame (кроме `CAI_Rat`, `CAI_Trader`,
`CAI_Crow`, `CPhantom`). Это не FSM и не планировщик — это **каркас (scaffold)**: набор полей
(память, контроллеры, ability-флаги, настройки), виртуальные «способности» (`ability_*`),
циклические точки входа (`Think`/`update_fsm`/`ProcessScripts`) и сервисы, которые конкретные
виды монстров (кровавый, бурер, шиммера…) переопределяют. Конкретный «характер» монстра
появляется на двух уровнях ниже: **FSM-состояния** (порц. 3) и **настройки** `SMonsterSettings`.

## 1. Место в иерархии

```cpp
class CBaseMonster : public CCustomMonster, public CStepManager   // base_monster.h:58
```

- `CCustomMonster` (ит. 5, порц. 9) — мост в общий `CObject`/`CSE_ALife_Object`-мир, `Think()`.
- `CStepManager` (`xrGame/step_manager.h`) — шаги, скорость, движение по сетке.
- Абстрактный: `virtual char* get_monster_class_name() = 0;` (base_monster.h:489) — каждый
  конкретный вид обязан вернуть своё имя класса.

`CBaseMonster` — **промежуточный**: под ним `CController`/`CBloodsucker`/`CBurer`/`CChimera`/
`CDog`/`CPseudodog`/`CPoltergeist`/`CCommunity`/… (15 видов, порц. 5–9, 16–17). `CAI_Rat`
наследуется **не** отсюда, а от `CCustomMonster` напрямую (порц. 10).

## 2. Что это НЕ делает

- **Не ведёт FSM.** Выбор/переключение состояний — в `update_fsm()`/`Think()` и
  `IStateManagerBase* StateMan` (порц. 3). Здесь только точка входа и указатель `StateMan`.
- **Не двигается сам.** Реальное движение — `CMovementManager`/`CControlPathBuilder`
  (порц. 21), здесь лишь фабрика `create_movement_manager()` и `control().install_path_manager`.
- **Не хранит данные мира.** Регистры ALife (`CSE_ALife*`) — ит. 11/12; здесь только
  «своя» память одного экземпляра.

## 3. Конструктор: какие подсистемы создаются

`CBaseMonster::CBaseMonster()` (base_monster.cpp:61) собирает каркас. Всё, что тут
`xr_new` — **внешние подсистемы**, живущие у каждого экземпляра монстра:

| Подсистема | Поле | Что делает |
|---|---|---|
| Физика тела | `m_pPhysics_support` (`CCharacterPhysicsSupport::etBitting`) | удары, «укусы», коллизии |
| **Контроллер** | `m_control_manager` (`CControl_Manager`) | оркестрация anim/move/path/dir (порц. 21) |
| Память врагов | `EnemyMemory` | «кто видел меня/врага» (порц. 22) |
| Память звуков | `SoundMemory` | слышанные звуки (порц. 22) |
| Память трупов | `CorpseMemory` | найденные/съеденные трупы (порц. 22) |
| Память ударов | `HitMemory` | кто меня бьёт (порц. 22) |
| Менеджёр врагов | `EnemyMan` (`CMonsterEnemyManager`) | текущий выбор врага |
| Менеджёр трупов | `CorpseMan` (`CMonsterCorpseManager`) | цель-труп |
| События | `EventMan` (`CMonsterEventManager`) | очередь событий |
| Ближний бой | `MeleeChecker` | проверка дистанции удара |
| Мораль | `Morale` (`CMonsterMorale`) | настроение, бегство |
| **Аномалии** | `m_anomaly_detector` (`CAnomalyDetector`) | избегает аномалий |
| Укрытия | `CoverMan` (`CMonsterCoverManager`) | поиск укрытий (порц. 24) |
| Дом | `Home` (`CMonsterHome`) | «своя» территория, возврат |
| Custom-менеджмент | `m_com_manager` (`CControlManagerCustom`) | ability: sequencer, triple-anim, critical-wound |

Кастомные ability в конструкторе:

```cpp
control().add(&m_com_manager, ControlCom::eControlCustom);
m_com_manager.add_ability(ControlCom::eControlSequencer);
m_com_manager.add_ability(ControlCom::eControlTripleAnimation);
com_man().add_ability(ControlCom::eComCriticalWound);
```

## 4. `_construct` и базовые контроллеры

`_construct()` (base_monster.cpp:776) — **фабрика базовых контроллеров**, виртуальный, чтобы
конкретный вид мог подменить анимацию/направление:

```cpp
void CBaseMonster::_construct() {
    create_base_controls();                 // 4 базовых контроллера (можно переопределить)
    control().add(m_anim_base, ControlCom::eControlAnimationBase);
    control().add(m_move_base, ControlCom::eControlMovementBase);
    control().add(m_path_base, ControlCom::eControlPathBase);
    control().add(m_dir_base,  ControlCom::eControlDirBase);
    control().set_base_controller(m_anim_base, ControlCom::eControlAnimation);
    control().set_base_controller(m_move_base, ControlCom::eControlMovement);
    control().set_base_controller(m_dir_base,  ControlCom::eControlDir);
    inherited::_construct();                // CCustomMonster
    CStepManager::_construct();
    return this;
}
```

`create_base_controls()` (base_monster.cpp:820) создаёт **пустые базы**: `CControlAnimationBase`,
`CControlMovementBase`, `CControlPathBuilderBase`, `CControlDirectionBase`. `CController`
переопределяет `create_base_controls()` (controller.cpp:692) и ставит свои `CControllerAnimation`/
`CControllerDirection` вместо анимации/направления. Доступ к ним — через геттеры
`anim()`/`move()`/`path()`/`dir()` (base_monster.h:440–443).

## 5. Abilities (виртуальные флаги «что умеет»)

Блок `ability_*` (base_monster.h:233–242) — **переопределяемые по умолчанию `false`**; конкретный
вид включает нужные. Настройки `settings_read` читают `RunAttack_*` **только если**
`ability_run_attack()` — т.е. ability управляет и конфигурацией.

| Метод | Дефолт | Суть |
|---|---|---|
| `ability_invisibility()` | `false` | невидимость (`state_invisible`) |
| `ability_can_drag()` | `false` | таскать трупы |
| `ability_psi_attack()` | `false` | пси-атака (`Hit_Psy`) |
| `ability_earthquake()` | `false` | «землетрясение» (всплеск силы) |
| `ability_can_jump()` | `false` | прыжки |
| `ability_distant_feel()` | `false` | «чувство» на расстоянии |
| `ability_run_attack()` | `false` | атака на бегу (AOM, §8) |
| `ability_rotation_jump()` | `false` | прыжок с поворотом |
| `ability_jump_over_physics()` | `false` | перепрыгнуть сквозь физику |
| `ability_pitch_correction()` | **`true`** | корректировка наклона (единственный `true`) |

## 6. Настройки: `SMonsterSettings` и base/current

- Структура `SMonsterSettings` (`ai_monster_shared_data.h:4`) — «карта поведения» вида:
  `m_fDamagedThreshold`, пороги/задержки звуков, `satiety_threshold`, `m_fEatFreq/Slice`,
  `m_legs_number`, `m_max_hear_dist`, `SAttackEffector m_attack_effector`, `m_run_attack_*`.
- Два слота: `ref_smem<SMonsterSettings> m_base_settings` и `m_current_settings`
  (base_monster.h:279–280), геттер `db()` → текущие.
  - `settings_load(section)` (base_monster_startup.cpp:478) — читает `m_base_settings` из `.ltx`.
  - `settings_overrides()` (base_monster_startup.cpp:489) — поверх base накатывает секцию
    `settings_overrides` **из `spawn_ini()`** (конкретный экземпляр/локация) и создаёт
    `m_current_settings`. Оба — по `crc32` (см. «ref_smem» — shared-память на CRC).
  - `settings_read()` (base_monster_startup.cpp:410) — общий читатель (макро `READ_SETTINGS`).

Итог: **вид задаёт базу, спавн-локация переопределяет** — один класс монстра ведёт себя
по-разному на разных картах без кода.

## 7. `Load` / `PostLoad` / `reload` / `reinit`

| Метод | Когда | Что |
|---|---|---|
| `Load(section)` (startup:53) | загрузка класса | bones head/eyes, cover-эвалуаторы, `MeleeChecker`/`Morale`, физика, `settings_load`, `control().load`, `anomaly_detector.load`, auras, protections, `rank` |
| `PostLoad(section)` (startup:151) | **конец** Load конкретного вида | AOM-параметры (`aom_*`), анти-аим (`anti_aim_ability`), анимации AOM |
| `reload(section)` (startup:220) | смена скина/режима | звуки (12 базовых `LOAD_SOUND`), `m_monster_type` (`universal/indoor/outdoor`), `Home->load("home")` |
| `reinit()` (startup:274) | переспаун | **сброс** всех memory/managers/флагов `m_bDamaged/m_bAggressive/...`, `StateMan->reinit`, `m_anomaly_detector->reinit`, `anim().clear_override_animation` |

`EMonsterType` (base_monster.h:333): `eMonsterTypeUniversal / Indoor / Outdoor` — влияет на
путь/поведение в помещениях.

## 8. Атака на бегу (Attack On Move, AOM)

`attack_on_move_params_t` (base_monster.h:562) + `can_attack_on_move()`/`get_attack_on_move_*()`.
Читается в `PostLoad` (§7): `aom_enabled`, `aom_far_radius`, `aom_attack_radius`,
`aom_update_side_period`, `aom_prediction_factor`, `aom_prepare_time/radius`, `aom_max_go_close_time`.
При включении добавляет анимации `eAnimAttackOnRunLeft/Right`. Геттеры идут через
`override_if_debug(...)` — в DEBUG-сборке их можно переопределить из консоли AI.

## 9. Auras (влияние на игрока)

4 инстанса `monster_aura` (base_monster.cpp:61): `m_psy_aura`, `m_radiation_aura`, `m_fire_aura`,
`m_base_aura`. Каждый (monster_aura.h) — радиус/коэффициенты/пост-процесс/детектор-звук:

```cpp
float CBaseMonster::get_psy_influence()      { return m_psy_aura.calculate(); }     // base_monster.cpp:1007
float CBaseMonster::get_radiation_influence(){ return m_radiation_aura.calculate(); }
float CBaseMonster::get_fire_influence()     { return m_fire_aura.calculate(); }
void   CBaseMonster::play_detector_sound()   { /* все три детектора */ }
```

Auras — «aura» монстра (крывосос — psi-аура): влияет на HUD/детектор игрока по дистанции.

## 10. Память: `UpdateMemory` и `on_kill_enemy`

`UpdateMemory()` (`base_monster_misc.cpp`) — **единый тик памяти**, каждый цикл:

```cpp
EnemyMemory.update(); SoundMemory.UpdateHearing(); CorpseMemory.update(); HitMemory.update();
EnemyMan.update();     CorpseMan.update();
hear_dangerous_sound = hear_interesting_sound = false;
if (SoundMemory.IsRememberSound()) { SoundMemory.GetSound(se, hear_dangerous_sound); ... }
m_bDamaged    = conditions().GetHealth() < db().m_fDamagedThreshold;
m_bAggressive = hear_dangerous_sound || EnemyMan.get_enemies_count() > 0 || HitMemory.is_hit();
```

Флаги `m_bDamaged`/`m_bAggressive`/`hear_dangerous/interesting_sound` (base_monster.h:314–315,
396–403) — **вход в FSM**: состояния читают их, решая «злиться/бежать/спать».

`on_kill_enemy(obj)` (base_monster.cpp:751) — после убийства: труп в `CorpseMemory`,
`HitMemory.remove_hit_info`, `SoundMemory.clear()`.

## 11. Аномалии: `CAnomalyDetector`

`CAnomalyDetector` (anomaly_detector.h/.cpp) — **избегание аномалий** через ограничения пути:
`load` читает `Anomaly_Detect_Radius` (15) и `Anomaly_Detect_Time_Remember` (30 000 мс).
`update_schedule()`: при `m_active` → `feel_touch_update(...)`; встреченные зоны-реестр-торы
(`on_contact`, только `CCustomZone` с `restrictor_type != None`) кладутся в `m_storage` и как
**in-restrictions** подаются в `path_builder().restrictions()` (монстр не идёт через аномалию),
по истечении таймера снимаются.

## 12. Хиты: `Hit_Psy` / `Hit_Wound`

`Hit_Psy`/`Hit_Wound` (base_monster_feel.cpp:297/314) — **монстр наносит урон объекту** через
обычный `GE_HIT`-пакет (`SHit`), `whoID=weaponID=свой ID`:

- `Hit_Psy` — `eHitTypeTelepatic`, dir `(0,1,0)` (крывосос «контроль»: controller.cpp:424
  `control_hit()` вызывает `Hit_Psy` + PP/камерные эффекты на игроке).
- `Hit_Wound` — `eHitTypeWound`, реальный dir/impulse, boneID из визуала.

Оба — мост в общий `Hit`/`u_EventSend` (ит. 5), т.е. урон монстра = тот же канал, что урон
оружия, только `hit_type` другой.

## 13. Связи

```mermaid
graph TD
    CM[CCustomMonster / CStepManager] --> BM[CBaseMonster]
    BM -->|StateMan| FSM[FSM: update_fsm / IStateManagerBase<br/>порц. 3]
    BM -->|control()/anim()/move()/path()/dir()| CTRL[CControl_Manager + базовые контроллеры<br/>порц. 21]
    BM -->|Enemy/Sound/Corpse/Hit Memory, Man| MEM[Память монстра<br/>порц. 22]
    BM -->|CoverMan| COVER[Укрытия, порц. 24]
    BM -->|ability_run_attack / aom_*| AOM[Атака на бегу]
    BM -->|m_anomaly_detector| ANOM[CAnomalyDetector]
    BM -->|SMonsterSettings base/current| SET[Настройки, порц. 4]
    BM --> KINDS[CController, CBloodsucker, CBurer, CChimera, CDog, CPseudodog, CPoltergeist, CCommunity ...<br/>порц. 5-9, 16-17]
    KINDS -->|Hit_Psy / Hit_Wound| HIT[GE_HIT пакет, ит. 5]
```

## 14. Замечания к плану

1. **`base_monster.cpp` — 1 111 строк**, это самая «толстая» часть каркаса (помимо `base_monster.h`
   686). План считает каталог `basemonster/` целиком (3 754) — корректно.
2. **`CBaseMonster` абстрактный** через `get_monster_class_name() = 0`; сам по себе не существует
   в рантайме — только конкретные виды.
3. `ability_pitch_correction()` — единственный ability с дефолтом `true` (все остальные `false`);
   при переопределении вида держим это в виду.
4. AOM-геттеры обёрнуты в `override_if_debug` — в DEBUG-сборках параметры можно менять из AI-консоли,
   что важно для отладки, но в release это фиксированные значения из `.ltx`.

## 15. Краткое резюме

`CBaseMonster` — **каркас монстра**: создаёт подсистемы (контроллер, 4 памяти, 2 менеджера,
ауры, детектор аномалий, укрытия, дом), задаёт abilities-флаги, настройки base/current,
циклические точки входа (`UpdateMemory` → флаги → FSM) и сервисы хита/ауры. Конкретный
«характер» вида дают FSM (порц. 3) и `SMonsterSettings` (порц. 4). Ниже по иерархии —
15 конкретных монстров; `CAI_Rat` в эту ветку не входит.

---

*Порцион 2 (итерация 6). Далее: порц. 3 — `CBaseMonster`: FSM-цикл (`base-monster-fsm.md`).*
