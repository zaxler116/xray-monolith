# xrSound — Emitter

> Заглушка-контракт. Разберётся в **порционе 9 итерации 4**.
> Файлы: `SoundRender_Emitter.h/.cpp`, `SoundRender_Emitter_FSM.cpp`,
> `SoundRender_Emitter_StartStop.cpp`, `SoundRender_Emitter_streamer.cpp`.

Ожидается: `CSoundRender_Emitter` (реализация `CSound_emitter`:
`set_*`/`stop(bDeffered)`/`play_time`), state-машина (play/stop/pause,
fade-in/out по `psSoundCull`), `psSoundCull`-логика, стриминг с диска,
очистка `feedback` при stop, владение эмиттером у `ref_sound_data`
(включая quirk `play_no_feedback`).

См. [Интерфейс и точка входа](interface.md) (порц. 5), [Обзор](index.md).
