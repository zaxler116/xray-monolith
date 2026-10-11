# Монстры редкие: Flesh/Fracture/PseudoGigant/Snork

> **Заглушка** — заполнится на итерации 6, порцион 17.

Наследники `CBaseMonster` из `src/xrGame/ai/monsters/`: `CAI_Flesh` (~310 стр.),
`CFracture` (~220), `CPseudoGigant` (~690), `CSnork` (~700): особые состояния
(«Predator_*» подгруппы `EMonsterState`), атаки.

## Взаимодействие

- Базовый цикл — порц. 2–4; FSM — порц. 18–20.
- `CSE_ALife*` — ит. 11/12; Lua-скрипты — ит. 14.
