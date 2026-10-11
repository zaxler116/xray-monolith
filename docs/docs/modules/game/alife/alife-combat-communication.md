# Combat/communication managers + alife_human_object_handler

> **Заглушка** — заполнится на итерации 6, порцион 32.

Корень `src/xrGame/`: `alife_combat_manager`, `alife_communication_manager` (~900 стр.)
(+ `alife_communication_space`), `alife_human_object_handler` (обработка событий для
человеческих NPC), `ai_sounds`.

## Взаимодействие

- Менеджеры — порц. 31; враги — порц. 23; память — порц. 22.
- Звук — `xrSound` (ит. 2); сеть — ит. 12.
