# xrSound — Core: процессор и окружение

> Заглушка-контракт. Разберётся в **порционе 7 итерации 4**.
> Файлы: `SoundRender_Core_Processor.cpp`, `SoundRender_Environment.h/.cpp`.

Ожидается: `update`/`commit`, логика слушателя, `CSoundRender_Environment`
(поля EFX/EAX, `load`/`save`/`lerp`/`clamp`), `SoundEnvironment_LIB`
(`sEnvironment.xr`, `GetID`/`Get`/`Append`), массив `reverbs[]`
(98 EFX-пресетов), версии окружений (`sdef_env_version=4`,
`sndenv_ver_extended=5`). Кросс-обязательство ит.2 порц.8.

См. [Интерфейс и точка входа](interface.md) (порц. 5), [Обзор](index.md).
