# CAI_Bloodsucker

> **Заглушка** — заполнится на итерации 6, порцион 5.

Крупнейший наследник `CBaseMonster` (`src/xrGame/ai/monsters/bloodsucker/`, ~3700 стр.):
особые состояния (vampire, burer-attack-подгруппы `EMonsterState`), атаки, скрипты.

## Взаимодействие

- Базовый цикл — порц. 2–4; FSM — порц. 18–20.
- `CSE_ALife*` — ит. 11/12; Lua-скрипты — ит. 14.
