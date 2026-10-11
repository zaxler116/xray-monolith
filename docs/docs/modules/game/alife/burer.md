# CBurer

> **Заглушка** — заполнится на итерации 6, порцион 7.

`src/xrGame/ai/monsters/burer/` (~2400 стр.): сканирование (`scanning_ability`),
состояния `BurerScanning`/`BurerAttack_*` (`EMonsterState`), телекинез.

## Взаимодействие

- Базовый цикл — порц. 2–4; FSM — порц. 18–20; `telekinesis`/`telekinetic_object` — порц. 26.
- `CSE_ALife*` — ит. 11/12; Lua-скрипты — ит. 14.
