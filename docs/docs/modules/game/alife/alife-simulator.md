# CALifeSimulator + alife_simulator_base

> **Заглушка** — заполнится на итерации 6, порцион 28.

Корень `src/xrGame/`: `alife_simulator` (~1400 стр.) + `alife_simulator_base`
(online/offline-группы, `alife_online_offline_group` + brain), `alife_level_registry`-связь,
`alife_anomalous_zone`. Тик ALife: кто в «онлайне» (полный AI) vs «оффлайне» (упрощённый
brain-скрипт).

## Взаимодействие

- Регистры — порц. 29; abstract — порц. 30; менеджеры — порц. 31–32.
- Сеть: `CSE_ALife*` — ит. 11/12 (здесь только `CSE_Abstract*` как аргумент `net_Spawn`).
- Lua brain-скрипты (`alife_*_brain_script.cpp`) — ит. 14.
