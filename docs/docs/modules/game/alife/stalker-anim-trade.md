# CAI_Stalker: анимация, стрельба, торговля

> **Заглушка** — заполнится на итерации 6, порцион 15.

`CStalkerAnimationManager` (`stalker_animation_*`), `stalker_combat_actions` (стрельба,
`ai_stalker_fire.cpp`), торговля (`CTradeItem`, `choose_*`), броски гранат,
critical wounds, per-NPC aim params.

## Взаимодействие

- Оружие — ит. 8; звук — `xrSound` (ит. 2); анимация/скелет — ит. 3 (`IKinematics`).
- Сеть: `net_*`-сигнатуры — ит. 12.
