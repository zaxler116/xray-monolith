# CAI_Stalker: класс, feel, события

> **Заглушка** — заполнится на итерации 6, порцион 12.

`CAI_Stalker` (`src/xrGame/ai/stalker/ai_stalker.h`, ~920 стр. + 13 файлов ~8200 стр.) —
`CCustomMonster + CObjectHandler + CAI_PhraseDialogManager + CStepManager`: поля
(`CStalkerAnimationManager*`, `CStalkerPlanner* m_brain`, `CSightManager*`,
`stalker_movement_manager_smart_cover*`, 6 cover-оценщиков, dispersion-параметры),
`Think()`, feel (`ai_stalker_feel.cpp`), события (`ai_stalker_events.cpp`),
`ai_stalker_alife.cpp` (включение в ALife).

## Взаимодействие

- Мост: `CCustomMonster` (порц. 9 ит. 5).
- Planner'ы — порц. 13; движение/покрытия — порц. 14; анимация/стрельба/торговля — порц. 15.
- `Feel::*` (порц. 27 ит. 5); `CSE_ALife*` — ит. 11/12.
