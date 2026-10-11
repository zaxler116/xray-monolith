# ALIFE (AI): обзор

> **Заглушка** — заполнится на итерации 6, порцион 1.

Архитектура AI в этом форке: два параллельных стека — монстры (`CCustomMonster` → `CBaseMonster` →
15 видов, FSM `CState`/`CMonsterStateManager`) и сталкеры (`CAI_Stalker` → planner'ы
`CActionPlannerScript`), плюс короткие ветки `CAI_Rat`/`CAI_Trader`/`CAI_Crow`/`CPhantom` и
слой ALife (`CALifeSimulator`, регистры, abstract-обёртки, менеджеры) в корне `src/xrGame/`.

## Публичный API (контракт)

- `CALifeSimulator::update` — тик ALife (online/offline-группы).
- `CBaseMonster::Think()` / `CAI_Stalker::Think()` — точки входа, вызываемые из `CCustomMonster`
  (порц. 9 ит. 5).
- `MONSTER_COMMUNITY` — relations между монстрами (ит. 5 порц. 5 «частично закрыто»).

## Взаимодействие

- **Ит. 5**: `CCustomMonster` — мост (порц. 9); `Feel::*` (порц. 27); `CAI_ObjectLocation`/
  `ai_obstacle` (порц. 3).
- **Ит. 7** (физика), **ит. 9** (зоны) — smart-террейн/аномалии как объекты окружения AI.
- **Ит. 11/12** — `CSE_ALife*` сетевые обёртки (здесь только `CSE_Abstract*` в `net_Spawn`).
- **Ит. 14** — полный Lua API (`alife_*_brain_script.cpp`, `stalker_*_script.cpp`).

## Связь с иерархией

```cpp
CEntityAlive → CCustomMonster → CBaseMonster → CAI_Bloodsucker, CAI_Boar, CBurer, CCat,
                            CChimera, CController, CAI_Dog, CAI_Flesh, CFracture,
                            CPoltergeist, CAI_PseudoDog, CPseudoGigant, CSnork,
                            CTushkano, CZombie
CEntityAlive → CCustomMonster → CAI_Rat        (напрямую, не CBaseMonster)
CEntityAlive → CCustomMonster → CAI_Stalker    (CObjectHandler + CStepManager)
CEntityAlive → CAI_Trader
CEntity      → CAI_Crow, CPhantom
```

Разбор — итерация 6 (порционы 1–33, см. план).
