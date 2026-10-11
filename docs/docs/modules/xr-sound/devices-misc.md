# xrSound — Устройства и прочее

> Заглушка-контракт. Разберётся в **порционе 10 итерации 4**.
> Файлы: `OpenALDeviceList.h/.cpp`, `NotificationClient.h/.cpp`,
> `SoundRender_CoreA.cpp` (устройство/EFX), `cl_intersect.h`,
> `SoundRender_CoreA.h` (listener/реверберация), `guids.cpp`,
> статистика (`statistic`/`statistic_ext` — кросс-обязательство ит.2 порц.12).

Ожидается: `ALDeviceList` (`Enumerate`/`SelectBestDevice`/
`GetDefaultDeviceName`, `ALDeviceDesc`), `CNotificationClient`
(Core Audio notifications → pending-флаги), `CSoundRender_CoreA`
(инициализация stage 1: device/context/EFX, 33 EFX-процедуры,
`switch_device` hot-swap, `set_listener`/`get_listener` (bug: GAINHF→RoomLF),
`update_listener` (Z-flip)), `cl_intersect.h` (почти дубль
`xrEngine/cl_intersect.h`), `statistic`/`statistic_ext`.

См. [Интерфейс и точка входа](interface.md) (порц. 5), [Обзор](index.md).
