# xrEngine (ядро)

`xrEngine` — ядро движка: `CEngine`, `CSheduler`, `CRenderDevice`, событийная шина, консоль, Lua-биндинг, камера/HUD, окружение, эффекты, ввод.

См. [Карта модулей](../../architecture/module-map.md), [Цикл кадра](../../architecture/frame-loop.md), [Модель объектов](../../architecture/object-model.md).

## Структура

| Страница                                             | Что покрывает                                                                                                                                                                                                                                                                                                                                                                                      |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Ядро (CEngine) и точка входа](engine.md)            | `CEngine`, `PSGP`, `WinMain`/`WinMain_impl`/`Startup`, `CApplication`, `pure.h`/`CRegistrator`, `defines.h`, CLSID                                                                                                                                                                                                                                                                                 |
| [API и события](api.md)                              | `CEngineAPI` (рендерер, игра, vTune, фабрика), `CEventAPI`/`CEvent`, `KERNEL:*`                                                                                                                                                                                                                                                                                                                    |
| [Цикл кадра](frame-loop.md)                          | `Device.Run`, `message_loop`, `on_idle`, `FrameMove`, `mt_Thread`, `seqFrame`/`seqRender`/`seqFrameMT`                                                                                                                                                                                                                                                                                             |
| [Sheduler](scheduler.md)                             | `CSheduler`, `ISheduled`, `Register`/`Unregister`/`EnsureOrder`, адаптивный бюджет                                                                                                                                                                                                                                                                                                                 |
| [Device](device.md)                                  | _(порцион 3)_ — `CRenderDevice`, `CRenderDeviceData`, `Device_create/Initialize/destroy`, таймеры, матрицы                                                                                                                                                                                                                                                                                         |
| [Окно и WndProc](device-window.md)                   | _(порцион 3)_ — `Device_wndproc`, `Device_Misc`, `Device_overdraw`, `xrHemisphere`                                                                                                                                                                                                                                                                                                                 |
| [Рендер-слой](render.md)                             | `Render.h/.cpp`, `IRender_Light/Glow/Target`, `CPS_Instance`, `ShadersExternalData`, `Shader_xrLC`, `vis_common`, `MbHelpers`, `imf_Process`, `PAPI`                                                                                                                                                                                                                                               |
| [IRenderable](renderable.md)                         | `IRenderable` — интерфейс рендеруемого объекта, `renderable_ROS`, lifecycle                                                                                                                                                                                                                                                                                                                        |
| [Fmesh](fmesh.md)                                    | `Fmesh.h` (OGF формат), `CFM_DynamicMesh`, `EnnumerateVertices`, `ogf_desc`                                                                                                                                                                                                                                                                                                                        |
| [Коллизии и физика (интерфейсы)](collide-physics.md) | `ICollidable`, `ICollisionForm`/`CCF_Skeleton`/`CCF_EventBox`/`CCF_Shape`, `clQueryCollision`, `IObjectPhysicsCollision`, `IPhysicsShell/Element/Geometry`                                                                                                                                                                                                                                         |
| [Объект (xr_object)](xr-object.md)                   | `CObject` (каркас: трансформ, `H_SetParent`, crow, `net_*`), `CObjectList` (active/sleeping/crow/destroy, `net_Export/Import`), `CObjectAnimator` (.anm/.anms)                                                                                                                                                                                                                                     |
| [Уровень](level.md)                                  | `IGame_Level` (контейнер + жизненный цикл: `Load`, `OnFrame/OnRender`, `SetEntity/ViewEntity`, `SoundEvent_*`), `CServerInfo`, `xrLevel.h` (бинарные структуры)                                                                                                                                                                                                                                    |
| [Пул объектов](object-pool.md)                       | `IGame_ObjectPool` (фабрика + prefetch; не пул — закомментирован)                                                                                                                                                                                                                                                                                                                                  |
| [Persistent](persistent.md)                          | `IGame_Persistent` (персистентный слой: `CEnvironment`, `ObjectPool`, партиклы `CPS_Instance`, `GrassBenders`, `params`, `g_pGamePersistent`)                                                                                                                                                                                                                                                      |
| [Скелет и анимация](skeleton-motion.md)              | `bone.h/.cpp` (`CBoneInstance`/`CBone`/`CBoneData`/`IBoneData`, `SBoneShape`, `SJointIKData`, `vertBoned1W..4W`), `SkeletonMotions.h/.cpp` (`CMotion`/`CMotionDef`/`CPartition`/`motion_marks`, `shared_motions`/`motions_container`), `motion.h/.cpp` (`CCustomMotion`/`COMotion`/`CSMotion`/`SAnimParams`), `envelope.h/.cpp` (`CEnvelope`/`st_Key`), `ObjectAnimator.h/.cpp`, `cf_dynamic_mesh` |
| [Окружение](environment.md)                          | `CEnvironment` (циклы погоды, WFX, миксер, модификаторы, амбиенты, солнце, ветер), `CEnvDescriptor`/`CEnvDescriptorMixer`, `CEnvModifier`, `CEnvAmbient`, `CPerlinNoise1D`, `envelope`/`interp` (LightWave-огибающие)                                                                                                                                                                              |
| [Эффекторы](effector.md)                             | `SBaseEffector`, `CEffectorCam` (камерные), `CEffectorPP` (пост-эффекты), типы из xrGame, `CCameraManager` (управление)                                                                                                                                                                                                                                                                            |
| [Feels](feel.md)                                     | _(порцион 9)_ — `Feel_Sound/Touch/Vision`                                                                                                                                                                                                                                                                                                                                                          |
| [Визуальные эффекты](effects.md)                     | _(порцион 10)_ — `perlin`, `thunderbolt`, `Rain`, `LightAnimLibrary`, `xr_efflensflare`                                                                                                                                                                                                                                                                                                            |
| [Камера](camera.md)                                  | _(порцион 11)_ — `CameraBase`, `CameraManager`                                                                                                                                                                                                                                                                                                                                                     |
| [HUD](hud.md)                                        | _(порцион 11)_ — `CustomHUD`, `GameFont`, `GIFResource`                                                                                                                                                                                                                                                                                                                                            |
| [UI-примитивы](ui-primitives.md)                     | _(порцион 11)_ — `imgui_base`, `line_edit_control`, `xrTheora_*`, `xrSASH`, `tntQAVI`                                                                                                                                                                                                                                                                                                              |
| [Ввод](input.md)                                     | _(порцион 12)_ — `xr_input`, `Xr_input`, `xr_input_xinput`                                                                                                                                                                                                                                                                                                                                         |
| [Консоль](console.md)                                | _(порцион 12)_ — `CConsole`/`CTextConsole`, `XR_IOConsole`, `xr_ioc_cmd`                                                                                                                                                                                                                                                                                                                           |
| [Lua-биндинг](lua-binding.md)                        | _(порцион 12)_ — `_scripting`, `ai_script_space`, `ai_script_lua_*`                                                                                                                                                                                                                                                                                                                                |
| [Демо](demo.md)                                      | _(порцион 12)_ — `FDemoPlay`, `FDemoRecord`                                                                                                                                                                                                                                                                                                                                                        |
| [Статистика](stats.md)                               | _(порцион 12)_ — `CStats`, `StatGraph`                                                                                                                                                                                                                                                                                                                                                             |
| [Прочее](misc.md)                                    | _(порцион 12)_ — `mailSlot`, `trivial_encryptor`, `cl_intersect`                                                                                                                                                                                                                                                                                                                                   |

> `shader_bus.h/.cpp` **не входит** в xrEngine: мигрирует в [Renderer → Shader Bus](../renderer/shader-bus.md) (итерация 3) вместе с `SHADER_BUS.md` из корня репо.

## Карта связей

```mermaid
graph TD
    subgraph xrEngine
        Engine[CEngine]
        Event[CEventAPI]
        Sched[CSheduler]
        Device[CRenderDevice]
        App[CApplication pApp]
        Ext[CEngineAPI]
    end
    subgraph расширения
        Render[xrRender R?]
        Game[xrGame]
        Server[xrServer]
    end
    Engine --> Event
    Engine --> Sched
    Engine --> Ext
    Ext --> Render
    Ext --> Game
    Game --> Server
    App --> Event
    App --> Device
    Device --> Render
    Sched --> App
    Sched --> Game
```

- **`CEngine`** — корень. Владельцы: `Engine`, `pApp`.
- **`CEventAPI`** — шина `KERNEL:*` между `xrGame`/`xrServer`/консолью и `CApplication`/рендерером.
- **`CSheduler`** — планировщик `ISheduled` (уровень, ALIFE, звук, …). Не путать с `CRegistrator` (упорядоченный список сообщений).
- **`CRenderDevice`** — устройство + окно + матрицы + таймеры. Поднимается через `Device.Create/Initialize/Run`.
- **`CApplication`** — «приложение» на уровне процесса: уровень, шрифт, загрузка, Discord.

## Статус порционов (итерация 2)

| #   | Порцион                                           | Страницы                                                                     | Статус    |
| --- | ------------------------------------------------- | ---------------------------------------------------------------------------- | --------- |
| 1   | Ядро и API                                        | `engine.md`, `api.md`                                                        | ✅ готово |
| 2   | Цикл кадра и планировщик                          | `frame-loop.md`, `scheduler.md`                                              | ✅ готово |
| 3   | Устройство                                        | `device.md`, `device-window.md`                                              | ✅ готово |
| 4   | Рендер-слой                                       | `render.md`, `renderable.md`, `fmesh.md`                                     | ✅ готово |
| 5   | Коллизии/физика                                   | `collide-physics.md`                                                         | ✅ готово |
| 6   | Объекты и уровень                                 | `xr-object.md`, `level.md`, `object-pool.md`, `persistent.md`                | ✅ готово |
| 7   | Скелет и анимация                                 | `skeleton-motion.md`                                                         | ✅ готово |
| 8   | Окружение и эффекторы                             | `environment.md`, `effector.md`                                              | ✅ готово |
| 9   | Feels                                             | `feel.md`                                                                    | ⬜        |
| 10  | Визуальные эффекты                                | `effects.md`                                                                 | ⬜        |
| 11  | Камера / HUD / UI                                 | `camera.md`, `hud.md`, `ui-primitives.md`                                    | ⬜        |
| 12  | Ввод / Консоль / Lua / Демо / Статистика / Прочее | `input.md`, `console.md`, `lua-binding.md`, `demo.md`, `stats.md`, `misc.md` | ⬜        |
