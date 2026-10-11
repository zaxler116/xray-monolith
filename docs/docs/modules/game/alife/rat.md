# CAI_Rat

> **Заглушка** — заполнится на итерации 6, порцион 10.

`src/xrGame/ai/monsters/rats/` (~3700 стр.): `CAI_Rat : CCustomMonster + CEatableItem` —
**не наследует `CBaseMonster`**; собственные состояния в `rats/`, поедание
(`CEatableItem` — ит. 5, инвентарь/еды).

## Взаимодействие

- Мост: `CCustomMonster` (порц. 9 ит. 5) — `Think()`.
- Еда/инвентарь — ит. 5 (порц. 10) и ит. 8.
- `CSE_ALife*` — ит. 11/12; Lua-скрипты — ит. 14.
