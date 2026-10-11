# xrSound — Core: ядро

> Заглушка-контракт. Разберётся в **порционе 6 итерации 4**.
> Файлы: `SoundRender_Core.cpp` (реализация), `SoundRender_Core_StartStop.cpp`,
> `SoundRender_Core_SourceManager.cpp`.

Ожидается: контейнеры `s_sources`/`s_emitters`/`s_targets`, `create`/
`attach_tail`/`clone`/`destroy`/`play*`, `_create_data`/`_destroy_data`,
`set_geometry_env/som/occ`, `get_environment` (ray-вниз, front/back id),
`pause_emitters`, `i_*`-внутренний API, `_restart`.

См. [Интерфейс и точка входа](interface.md) (порц. 5), [Обзор](index.md).
