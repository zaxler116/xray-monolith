# CBaseMonster: настройки, ауры, berserk, крит. раны

> **Заглушка** — заполнится на итерации 6, порцион 4.

`SMonsterSettings m_base/m_current_settings` (`monster_settings`), auras
(`monster_aura`: psy/radiation/fire, `psy_aura`), berserk, panic threshold, critical wound
(`control_critical_wound`), `attack_on_move_params_t`, `CMeleeChecker` (ближний бой).

## Взаимодействие

- Раны/смерть — ит. 5 порц. 8 (`wounds-death.md`).
- `Feel::Vision/Sound/Touch` — ит. 5 порц. 27; реализации в монстре — порц. 2.
- Ауры ↔ зоны (порц. 24–26 ит. 5) — smart-террейн/аномалии.
