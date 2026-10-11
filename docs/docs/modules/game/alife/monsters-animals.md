# Монстры-«животные»: Dog/PseudoDog/Cat/Boar/Tushkano/Zombie

> **Заглушка** — заполнится на итерации 6, порцион 16.

Наследники `CBaseMonster` из `src/xrGame/ai/monsters/`: `CAI_Dog` (~730 стр.),
`CAI_PseudoDog` (~1200), `CCat` (~300), `CAI_Boar` (~370), `CTushkano` (~230),
`CZombie` (~580): особые состояния, атаки, поведение стай (`ai_monster_squad*`).

## Взаимодействие

- Базовый цикл — порц. 2–4; FSM — порц. 18–20; squad — порц. 20.
- `CSE_ALife*` — ит. 11/12; Lua-скрипты — ит. 14.
