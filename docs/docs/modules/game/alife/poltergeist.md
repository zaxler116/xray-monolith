# CPoltergeist

> **Заглушка** — заполнится на итерации 6, порцион 8.

`src/xrGame/ai/monsters/poltergeist/` (~2300 стр.): манипуляция объектами (телекинез),
особые состояния, `anti_aim_ability`.

## Взаимодействие

- Базовый цикл — порц. 2–4; FSM — порц. 18–20; `telekinesis`/`telekinetic_object` — порц. 26.
- `CSE_ALife*` — ит. 11/12; Lua-скрипты — ит. 14.
