# MONSTER_COMMUNITY и relations

> **Заглушка** — заполнится на итерации 6, порцион 27.

`src/xrGame/monster_community.*`: `MONSTER_COMMUNITY : CIni_IdToIndex` — `set(id/index)`,
`id()`, `index()`, `team()`, статич. `relation(from, to)`. Таблица отношений
монстров (союзники/враги) — закрывает «частично закрытое» обещание порц. 5 ит. 5.

## Взаимодействие

- Враги — порц. 23; community-менеджеры ALife — порц. 31–32.
- Lua-скрипты — ит. 14.
