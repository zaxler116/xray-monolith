# Пространства AI: ai_space, CAI_ObjectLocation, ai_obstacle, member_*

> **Заглушка** — заполнится на итерации 6, порцион 26.

Корень `src/xrGame/`: `ai_space` (пространство AI-объектов), `CAI_ObjectLocation`
(+impl/inline), `ai_obstacle`, `agent_location_manager`, `object_handler` (+
`object_handler_planner` weapon/missile), `object_actions`, `member_corpse`/
`member_order`/`member_enemy` (squad-состав), `controlled_entity`/`controlled_actor`.

## Взаимодействие

- Закрывает обещанное порц. 3 ит. 5 (`CAI_ObjectLocation`/`ai_obstacle`).
- Squad — порц. 20; planner'ы — порц. 13.
