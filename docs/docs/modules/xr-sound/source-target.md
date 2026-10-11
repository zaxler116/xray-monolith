# xrSound — Source + Target

> Заглушка-контракт. Разберётся в **порционе 8 итерации 4**.
> Файлы: `SoundRender_Source.h/.cpp`, `SoundRender_Source_loader.cpp`,
> `SoundRender_Target.h/.cpp`, `SoundRender_TargetA.h/.cpp`,
> `SoundRender_Cache.h/.cpp` (LRU).

Ожидается: `CSoundRender_Source` (аллокатор/стример, Ogg-декодер,
`length_sec`/`channels_num`), loader (метаданные без полного декода),
`CSoundRender_Target`/`TargetA` (OpenAL-источник: buffer/source,
`sq_*`-качества, пул `psSoundTargets`), `CSoundRender_Cache`
(LRU, `cache_bytes_per_line`), `get_occlusion_to`/`get_occlusion`
(кросс-обязательство ит.3 порц.4).

См. [Интерфейс и точка входа](interface.md) (порц. 5), [Обзор](index.md).
