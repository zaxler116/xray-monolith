# Менеджеры ALife

> **Заглушка** — заполнится на итерации 6, порцион 31.

Корень `src/xrGame/`: `alife_time_manager`, `alife_update_manager` (~630 стр.),
`alife_interaction_manager`, `alife_storage_manager`, `alife_surge_manager`,
`alife_switch_manager` — что управляют и когда вызываются (связка с симулятором, порц. 28).

## Взаимодействие

- Симулятор — порц. 28; combat/communication — порц. 32.
- Цикл кадра — ит. 1/2 (`frame-loop`).
