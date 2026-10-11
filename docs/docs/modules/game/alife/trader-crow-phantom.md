# CAI_Trader / CAI_Crow / CPhantom

> **Заглушка** — заполнится на итерации 6, порцион 11.

Короткие ветки, не использующие `CBaseMonster`/`CActionPlannerScript`:
- `CAI_Trader` (`src/xrGame/ai/trader/`, ~750 стр.) — `CEntityAlive`; торговля, анимации.
- `CAI_Crow` (`src/xrGame/ai/crow/`, ~730 стр.) — `CEntity`; стаи, перелёты.
- `CPhantom` (`src/xrGame/ai/phantom/`, ~540 стр.) — `CEntity`; частицы
  (`CParticlesObject` — ит. 4), призрачный вид.

## Взаимодействие

- Иерархия: `CEntityAlive`/`CEntity` (порц. 5–7 ит. 5).
- `CSE_ALife*` — ит. 11/12; Lua-скрипты (`ai_crow_script.cpp`) — ит. 14.
