# CChimera

> **Истник**: `src/xrGame/ai/monsters/chimera/` (21 файл, ~1 900 строк) — ключевые
> `chimera.h` (46), `chimera.cpp` (175), `chimera_attack_state.h` (70),
> `chimera_attack_state_inline.h` (519), `chimera_state_manager.cpp` (80),
> `chimera_state_threaten*.h`, `chimera_state_hunting*.h`. Порцион 9, итерация 6.

`CChimera` (химера) — **самый «простой» крупный монстр**: наследует **только
`CBaseMonster`** (без `CControlledActor`/`CTelekinesis`/`CEnergyHolder`). Вся
специфика — в **FSM** (`ChimeraAttackState`): химера **прыгает** на врага
(`com_man().jump_if_possible`), атакуя в прыжке (`HitEntityInJump`). Атаки
«подготовка → прыжок» по таймерам и `attack_radius`, вращение (`eControlJump`),
`threaten`/`hunting`-состояния. Каркас — порц. 2–4; FSM-цикл — порц. 18–20.

## 1. Место в иерархии

```cpp
class CChimera : public CBaseMonster   // chimera.h:5
```

- `CBaseMonster` (порц. 2–4) — весь каркас. **Нет** дополнительных баз (сравнить:
  bloodsucker `+CControlledActor` порц. 5, controller `+CControlledActor` порц. 6,
  burer `+CTelekinesis` порц. 7, poltergeist `+CTelekinesis+CEnergyHolder` порц. 8).
- **`StateMan`** (конструктор, chimera.cpp:17): `xr_new<CStateManagerChimera>`.
- **Ability** (конструктор, chimera.cpp:18): `com_man().add_ability(ControlCom::eControlJump)` —
  прыжок как ability (управляется `CControlManagerCustom`, порц. 21).

`get_monster_class_name()` — `"chimera"` (chimera.h:20).

## 2. Abilities

Переопределённых abilities **нет** (только base). `CustomVelocityIndex2Action`
(chimera.cpp:119) мапит `eChimeraVelocityParameterJumpGround`/`Prepare` → `ACT_RUN`.
`eControlJump` — ability, включаемая в `ChimeraAttackState` через
`check_control_start_conditions` (m_allow_jump).

## 3. `Load`: анимации + `attack_params`

`Load` (chimera.cpp:26) — компактный (сравнить с «толстыми» порц. 5–8):
- `accel_load` + 4 `accel_chain_add`; `AddReplacedAnim(&m_bDamaged/…)`;
- `m_velocity_rotate` (`Velocity_Rotate`), `m_velocity_jump_start` (`Velocity_JumpStart`);
- ~20 `AddAnim` (stand/walk/run/lie/eat/attack/steal/prepare/upper-attack/threaten),
  `eAnimUpperAttack` = `jump_attack_` (прыжок-атака, `m_velocity_jump_start`);
  `LinkAction`.
- **`m_attack_params`** (chimera.cpp:105-111, `attack_params`, h:28-37):
  `attack_radius` (10), `prepare_jump_timeout` (2000), `attack_jump_timeout` (1000),
  `stealth_timeout` (2000), `force_attack_distance` (8), `num_attack_jumps` (4),
  `num_prepare_jumps` (2). Все `READ_IF_EXISTS`.
- `PostLoad` (chimera.cpp:116).

`reinit` (chimera.cpp:130): `move().load_velocity(…, "Velocity_JumpGround",
eChimeraVelocityParameterJumpGround)` + `com_man().load_jump_data(…, "jump_attack_1",
"jump_attack_2", …, eChimeraVelocityParameterJumpGround, 0)` — данные прыжка.

## 4. Прыжок-атака: `jump` / `HitEntityInJump`

- `jump(position, factor)` (chimera.cpp:166): `com_man().script_jump(position, factor)`
  + звук `eMonsterSoundAggressive`. Вызывается из FSM.
- `HitEntityInJump(pEntity)` (chimera.cpp:159): `AA_GetParams("jump_attack_1")`
  → `HitEntity(pEntity, params.hit_power, params.impulse, params.impulse_dir)`
  (удар в прыжке по данным анимации).
- `check_if_jump_possible` (inline:100): `com_man().check_if_jump_possible(target)` +
  `level_graph().valid_vertex_position` + `jump_intersect_geometry` (не сквозь врага).
- `jump(target, attack_jump)` (inline:118): проверка yaw (`angle_difference < 1`),
  `m_allow_jump=true`, `com_man().jump_if_possible(target, enemy, attack_jump, true, true)`.

## 5. FSM: `CStateManagerChimera` + `ChimeraAttackState`

`StateMan` = `CStateManagerChimera` (state_manager.cpp:18). Подсостояния:
`Rest`, `Panic`, `Attack` (→ `ChimeraAttackState`), `Eat`, `HearInt/DangerSound`.
(`Hitted`/`Threaten`/`Custom` — **закомменчены**, cpp:25-27.)

`execute` (cpp:33-67): enemy → по `get_danger_type` (`eStrong`→`Panic`,
`eWeak`→`Attack`); dangerous/interesting sound; `can_eat`→`Eat`; иначе `Rest`.
(Блок `eStateThreaten` — закомменчен.)

`ChimeraAttackState` (attack_state.h, inline 519 строк) — **вся логика атаки**:
- Состояния `state_undefined/state_rotate/state_prepare_jump` (h:53-54).
- `calculate_min_run_distance` (inline:21) — из `attack_radius` + `jump_max_distance`
  (геометрия дуги, `num_scan_points=10`).
- `select_target_for_move`/`select_target_for_jump`/`select_target_for_attack_jump` —
  выбор цели (вокруг врага, `behind_point`).
- `check_if_jump_possible`/`jump` (inline:100/118), `correct_jump_pos`,
  `set_turn_animation`.
- `check_control_start_conditions` (inline:87): `eControlJump` только при `m_allow_jump`.
- Таймеры: `m_last_jump_time`, `m_stealth_end_tick`, `m_state_end_tick`,
  `m_num_attack_jumps`/`m_num_prepare_jumps`.

Доп. состояния (закомменчены в менеджере, но файлы есть): `CStateChimeraThreaten`
(inline 117, `roar`/`steal`/`walk` подсостояния), `CStateChimeraHunting`
(inline 42, `come_out`/`move_to_cover`).

## 6. Связи

```mermaid
graph TD
    BM[CBaseMonster, порц. 2-4] --> CH[CChimera]
    CH -->|StateMan = CStateManagerChimera| SM[FSM, порц. 18-20]
    SM --> ATK[ChimeraAttackState: rotate/prepare_jump/attack_jump]
    CH -->|com_man().add_ability eControlJump| COM[CControlManagerCustom, порц. 21]
    ATK -->|jump_if_possible, check_if_jump_possible| JUMP[Jump control: com_man]
    ATK -->|HitEntityInJump: AA_GetParams jump_attack_1| HIT[HitEntity]
    ATK -->|select_target_for_move, level_graph| LG[ai level graph, порц. 26]
    CH -.->|закомменч.| TH[CStateChimeraThreaten / CStateChimeraHunting]
```

## 7. Замечания к плану

1. **Только `CBaseMonster`** — единственный «крупный» монстр без дополнительной базы
   (bloodsucker/controller `+CControlledActor`, burer/poltergeist `+CTelekinesis`,
   poltergeist `+CEnergyHolder`). Вся специфика — в FSM + ability `eControlJump`.
2. **Атака = прыжок** (`com_man().jump_if_possible`), удар — `HitEntityInJump`
   (данные `AA_GetParams("jump_attack_1")`). Нет «обычного» melee-удара.
3. **`eControlJump` — ability** (конструктор), управляется `m_allow_jump` в
   `ChimeraAttackState` (`check_control_start_conditions`).
4. `m_attack_params` (7 полей) — вся настройка атаки; `reinit` грузит
   `Velocity_JumpGround` + `load_jump_data("jump_attack_1/2")`.
5. `Threaten`/`Hitted`/`Hunting`-состояния **закомменчены** в `CStateManagerChimera`
   (файлы `chimera_state_threaten*.h`/`hunting*.h` существуют, но не добавлены).
6. `Load` компактный (по сравнению с «толстыми» порц. 5–8) — химера не имеет
   вампиризма/невидимости/контроля/телекинеза.
7. `calculate_min_run_distance` — геометрия дуги по `attack_radius` + `jump_max_distance`.

## 8. Краткое резюме

`CChimera` — «простой» наследник `CBaseMonster` (без доп. баз): вся специфика — в
FSM (`ChimeraAttackState`: rotate/prepare_jump/attack_jump), ability `eControlJump`,
`m_attack_params` (7 полей), прыжок-атака (`jump_if_possible` + `HitEntityInJump`),
`threaten`/`hunting`-состояния закомменчены. Каркас — порц. 2–4, FSM — порц. 18–20,
`eControlJump` — порц. 21, `level_graph` — порц. 26.

---

*Порцион 9 (итерация 6). Далее: порц. 10 — `CAI_Rat` ([rat.md](rat.md)).*
