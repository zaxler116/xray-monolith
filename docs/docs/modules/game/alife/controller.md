# CAI_Controller

> **Заглушка** — заполнится на итерации 6, порцион 6.

`src/xrGame/ai/monsters/controller/` (~3600 стр.): контроль над другими объектами
(`controlled_actor`/`controlled_entity` в корне `monsters/`), особые состояния `Controlled`.

## Взаимодействие

- Базовый цикл — порц. 2–4; FSM — порц. 18–20; `controlled_entity` — порц. 26.
- `CSE_ALife*` — ит. 11/12; Lua-скрипты — ит. 14.
