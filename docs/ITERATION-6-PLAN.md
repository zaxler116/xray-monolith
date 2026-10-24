# План документирования — Итерация 6: xrGame: ALIFE (AI)

> Рабочий документ-план. Удаляется после завершения итерации 6 (лог — в §9 GENERATE-DOCS.md).

## Масштаб

Итерация 6 — «ALIFE (AI)»: всё, что стоит за мостом `CCustomMonster` (порц. 9 ит. 5) и за
симулятором ALife. Архитектура AI в этом форке **не классическая**: нет `CAI_Base`/`CAI_Monster`/
`CAI_Manager`/`ai_memory`/`ai_planner`. Вместо этого два параллельных стека:

- **Монстры**: `CCustomMonster` → `CBaseMonster` (`ai/monsters/basemonster/`, ~4600 стр.) → 15 видов
  монстров; FSM `CState`/`CMonsterStateManager`/`EMonsterState` + ~20 классов в `states/` (101 файл)
  и `group_states/` (27 файлов); память/враги/покрытия `CMonster*Manager`/`*Memory`;
  контроллеры `CControl_*`.
- **Сталкеры (NPC-люди)**: `CAI_Stalker` (`ai/stalker/`, 13 файлов ~8200 стр.) — `CCustomMonster +
  CObjectHandler + CStepManager`, planner'ы `CStalker*Planner` (CActionPlannerScript),
  `CStalkerAnimationManager`, `CSightManager`, `stalker_movement_manager_smart_cover`,
  торговля, стрельба, feel.
- **Отдельные ветки**: `CAI_Rat` (`CCustomMonster + CEatableItem`), `CAI_Trader` (`CEntityAlive`),
  `CAI_Crow` (`CEntity`), `CPhantom` (`CEntity`).
- **Слой ALife** (корень `xrGame/`, ~420 файлов): `CALifeSimulator` + `alife_simulator_base`,
  регистры `alife_*_registry` (spawn/schedule/abstract/group/graph/level/story/object),
  abstract-обёртки `alife_*_abstract`, менеджеры ALife (time/update/interaction/storage/surge/
  switch/combat/communication), `MONSTER_COMMUNITY` (relations), память агентов
  (`memory_manager`, `agent_memory_manager`, `visual/sound/hit_memory_manager`, `memory_space`),
  enemy-менеджеры (`enemy_manager`, `agent_enemy_manager`), cover-eval'ы (`cover_evaluators` —
  inline-файл ~199k строк, читается выборочно), steering, `CStepManager`,
  `CAI_ObjectLocation`/`ai_obstacle`/`ai_space`.

Покрытие: ~54 000 строк в `ai/monsters/` + ~8 200 в `ai/stalker/` + ~2 000 в
`ai/{crow,trader,phantom}` + ~420 файлов AI/ALife-слоя в корне `xrGame/`. Сетевой код
`CSE_ALife*` — итерация 11/12 (здесь только `CSE_Abstract*` как аргумент `net_Spawn`).

## Решения (фиксация)

1. **`CAI_Rat` не наследует `CBaseMonster`** — напрямую `CCustomMonster + CEatableItem`
   (состояния в `ai/monsters/rats/`). Документируется в порционе 10 как отдельная ветка.
2. **`CAI_Trader` = `CEntityAlive`, `CAI_Crow` = `CEntity`, `CPhantom` = `CEntity`** — отдельные
   короткие ветки, объединяются в один порцион 11.
3. **`CAI_Stalker` — отдельный большой блок** (порционы 13–16): класс + feel, planner'ы
   (combat/danger/search/…), движение/покрытия, анимация + trade.
4. **`ai/monsters/` — по видам, 4 порциона** (12, 15, 16, 17): крупные/уникальные (bloodsucker,
   controller, burer, poltergeist, chimera), «животные» (dog, pseudodog, cat, boar, tushkano,
   zombie), редкие (flesh, fracture, pseudogigant, snork).
5. **FSM-фундамент отдельный порцион** (14): `CState` (`state.h`), `EMonsterState`
   (`state_defs.h`), `CMonsterStateManager`, `CStateMove`; конкретные `CStateMonster*` —
   порцион 19 (states) и 20 (group_states).
6. **Контроллеры `CControl_*` — отдельный порцион** (18): `CControl_Manager`, base-классы
   animation/movement/path-builder/direction, jump/melee/run-attack/sequencer/threaten/critical-wound.
7. **Память и враги — общие порционы** (21–22): `memory_manager`-иерархия + `memory_space`;
   `enemy_manager`/`agent_enemy_manager` + `monster_enemy_manager`/memory.
8. **`cover_evaluators_inline.h` (~199k строк) читается выборочно** — только API и структура,
   не построчно.
9. **`CSE_ALife*` (сетевые обёртки) — итерация 11/12.** Здесь только упоминание
   `CSE_Abstract*`/`CSE_ALife_Abstract` как аргумента `net_Spawn` в порционе 28.
10. **`CScriptGameObject`-скрипты AI** (`alife_*_brain_script.cpp`, `stalker_*_script.cpp`) —
    только сигнатуры; полный Lua API — итерация 14.

## Порционы

| # | Порцион | Страницы |
|---|---------|----------|
| 1 | Обзор ALIFE: архитектура, карта веток, цикл обновления | `index.md` |
| 2 | `CBaseMonster`: каркас и abilities | `base-monster.md` |
| 3 | `CBaseMonster`: FSM-цикл, память, контроллеры (обзор) | `base-monster-fsm.md` |
| 4 | `CBaseMonster`: настройки, ауры, berserk, крит. раны | `base-monster-settings.md` |
| 5 | `CAI_Bloodsucker` | `bloodsucker.md` |
| 6 | `CAI_Controller` | `controller.md` |
| 7 | `CBurer` | `burer.md` |
| 8 | `CPoltergeist` | `poltergeist.md` |
| 9 | `CChimera` | `chimera.md` |
| 10 | `CAI_Rat` (отдельная ветка, не `CBaseMonster`) | `rat.md` |
| 11 | `CAI_Trader` / `CAI_Crow` / `CPhantom` (короткие ветки) | `trader-crow-phantom.md` |
| 12 | `CAI_Stalker`: класс, feel, события | `stalker.md` |
| 13 | `CAI_Stalker`: planner'ы (combat/danger/search/death) | `stalker-planners.md` |
| 14 | `CAI_Stalker`: движение и покрытия | `stalker-movement.md` |
| 15 | `CAI_Stalker`: анимация, стрельба, торговля | `stalker-anim-trade.md` |
| 16 | `CAI_Dog`/`CAI_PseudoDog`/`CCat`/`CAI_Boar`/`CTushkano`/`CZombie` | `monsters-animals.md` |
| 17 | `CAI_Flesh`/`CFracture`/`CPseudoGigant`/`CSnork` | `monsters-rare.md` |
| 18 | FSM-фундамент: `CState`, `EMonsterState`, `CMonsterStateManager` | `monster-fsm.md` |
| 19 | Состояния монстров (`states/`: attack, find_enemy, hitted, panic, rest, eat, …) | `monster-states.md` |
| 20 | Групповые состояния (`group_states/`) | `monster-group-states.md` |
| 21 | Контроллеры `CControl_*` | `control-managers.md` |
| 22 | Память: `memory_manager` + visual/sound/hit, `memory_space` | `memory.md` |
| 23 | Враги: `enemy_manager`, `agent_enemy_manager`, `monster_enemy_manager` | `enemy.md` |
| 24 | Покрытия: `cover_evaluators`, `smart_cover_evaluators`, `monster_cover_manager` | `cover.md` |
| 25 | Steering + `CStepManager` | `steering-steps.md` |
| 26 | Пространства AI: `ai_space`, `CAI_ObjectLocation`, `ai_obstacle`, `member_*` | `ai-space.md` |
| 27 | `MONSTER_COMMUNITY` + relations | `community.md` |
| 28 | `CALifeSimulator` + `alife_simulator_base` | `alife-simulator.md` |
| 29 | Регистры ALife (`alife_*_registry`) + `alife_registry_container` | `alife-registries.md` |
| 30 | Abstract-обёртки ALife (`alife_*_abstract`, `alife_object`) | `alife-abstract.md` |
| 31 | Менеджеры ALife (time/update/interaction/storage/surge/switch) | `alife-managers.md` |
| 32 | Combat/communication managers + `alife_human_object_handler` | `alife-combat-communication.md` |
| 33 | Связность: object-model + зависимости + index | update `architecture/object-model.md` + `dependencies.md` + `index.md` |

## Статус

| # | Статус |
|---|--------|
| 1 | ✅ |
| 2 | ✅ |
| 3 | ✅ |
| 4 | ✅ |
| 5 | ✅ |
| 6 | ✅ |
| 7 | ✅ |
| 8 | ✅ |
| 9 | ✅ |
| 10 | ✅ |
| 11 | ✅ |
| 12 | ✅ |
| 13 | ⬜ |
| 14 | ⬜ |
| 15 | ⬜ |
| 16 | ⬜ |
| 17 | ⬜ |
| 18 | ⬜ |
| 19 | ⬜ |
| 20 | ⬜ |
| 21 | ⬜ |
| 22 | ⬜ |
| 23 | ⬜ |
| 24 | ⬜ |
| 25 | ⬜ |
| 26 | ⬜ |
| 27 | ⬜ |
| 28 | ⬜ |
| 29 | ⬜ |
| 30 | ⬜ |
| 31 | ⬜ |
| 32 | ⬜ |
| 33 | ⬜ |

## Обещанное из итераций 0–5

- **`MONSTER_COMMUNITY`** (ит. 6, «частично закрыто» порц. 5 ит. 5) — полный разбор в порционе 27.
- **`CCustomMonster`-мост** (порц. 9 ит. 5 ⬜) — AI-внутренности закрываются здесь (порционы 2–17);
  порц. 9 ит. 5 остаётся как «каркас + `Think()`», ссылки ведут в `alife/`.
- **`CAI_ObjectLocation`/`ai_obstacle`** (порц. 3 ит. 5) — закрываются порционы 26.
- **`Feel::*`** (порц. 27 ит. 5 ⬜) — реализации `Feel::Vision/Sound/Touch` в
  `CCustomMonster`/`CAI_Stalker` (`ai_stalker_feel.cpp`) упоминаются в порционах 2/12;
  общий разбор `Feel` — ит. 5 порц. 27.
- **`CSpaceRestrictor` ↔ AI** (порц. 24–26 ит. 5 ⬜) — `state_move_to_restrictor`,
  `alife_anomalous_zone` — упоминания в порционах 19/30.
- **`CSE_ALife*`** (ит. 11/12) — только `CSE_Abstract*` как аргумент `net_Spawn` (порц. 28).
- **`CScriptGameObject` API** (ит. 14) — только скрипт-сигнатуры AI (решение 10).

## После итерации 6

- `docs/docs/modules/game/alife/` — 33 страницы (index + 31 по порционам).
- `docs/docs/architecture/object-model.md` — обновлена: ветки `CCustomMonster` →
  `CBaseMonster`/`CAI_Stalker`/`CAI_Rat`, слой ALife (порцион 33).
- `docs/docs/modules/game/index.md` — строка ALIFE → `alife/index.md`, стрелка `ai/` в карте связей.
- `docs/mkdocs.yml` — заглушка `alife.md` заменена на подкаталог `ALIFE (AI) (Ит. 6):`.
- Лог итерации 6 → `GENERATE-DOCS.md` §9.
- Этот файл удаляется.
