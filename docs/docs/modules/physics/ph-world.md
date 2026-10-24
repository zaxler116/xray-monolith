# CPHWorld — физический мир

> Итерация 7, порцион 1. Файлы: `PHWorld.h` (173 строк), `PHWorld.cpp` (709 строк), `Physics.h` (53), `Physics.cpp`, `params.h/cpp`, `PhysicsCommon.h`.

`CPHWorld` — singleton, управляющий ODE-миром: тела, геометрии, суставы, шаги симуляции. Тикает через `pureFrame::OnFrame()` → `FrameStep()`.

## Наследование

```
CPHWorld : pureFrame (ITickable), IPHWorld, cphysics_scripted, [DEBUG: pureRender]
```

- `pureFrame` — `OnFrame()` вызывается каждый кадр из `CSheduler` (ит. 2).
- `IPHWorld` — facade для `xrGame` (создание/уничтожение, gravity, freeze).
- `cphysics_scripted` — Lua-биндинг (порц. 2).

## Глобалы и точка входа

```cpp
extern CPHWorld* ph_world;          // PHWorld.h:169
IPHWorld* __stdcall physics_world() // PHWorld.cpp:42 — accessor
void __stdcall create_physics_world(bool mt, CObjectSpace*, CObjectList*, CRenderDeviceBase*)
void __stdcall destroy_physics_world()
```

`create_physics_world` вызывается из `xrGame` при загрузке уровня. `mt` — multi-thread flag (регистрация в `seqFrameMT` vs `seqFrame`).

## Поля

| Поле | Тип | Назначение |
|------|-----|------------|
| `m_objects` | `PH_OBJECT_STORAGE` (CPHItemList<CPHObject>) | Активные объекты (shell'ы, персонажи) |
| `m_freezed_objects` | `PH_OBJECT_STORAGE` | Объекты во время freeze |
| `m_recently_disabled_objects` | `PH_OBJECT_STORAGE` | Недавно деактивированные (отложенная чистка) |
| `m_update_objects` | `PH_UPDATE_OBJECT_STORAGE` | `CPHUpdateObject` (update-без-body) |
| `m_freezed_update_objects` | `PH_UPDATE_OBJECT_STORAGE` | Update-объекты во время freeze |
| `Mesh` | `CPHMesh` | TriList-геометрия уровня (статическая коллизия) |
| `m_motion_ray` | `dGeomID` | RayMotions-геометрия (пули, трассеры) |
| `ContactGroup` | `dJointGroupID` (static) | Группа контактов текущего шага |
| `phBoundaries` | `Fbox` (static) | Границы уровня (AABB, y1 −= 30) |
| `m_gravity` | `float` | Гравитация (default: `default_world_gravity`) |
| `m_steps_num` | `u64` | Счётчик физических шагов |
| `m_frame_time` / `m_previous_frame_time` | `dReal` | Остаток кадра (для fixed-step) |
| `b_frame_mark` | `bool` | Флип-фlop для FrameTime(bool) |
| `b_world_freezed` | `bool` | Мир заморожен |
| `b_processing` | `bool` | Идёт Step() |
| `m_update_callback` | `IPHWorldUpdateCallbck*` | Колбэк на каждый step (xrGame) |
| `m_object_space` | `CObjectSpace*` | Статическая геометрия уровня (xrCDB) |
| `m_level_objects` | `CObjectList*` | Список объектов уровня |
| `m_device` | `CRenderDeviceBase*` | Указатель на устройство |
| `m_default_contact_shotmark` | `ContactCallbackFun*` | Колбэк контакта (wallmark) |
| `m_default_character_contact_shotmark` | `ContactCallbackFun*` | Колбэк контакта персонажа |
| `physics_step_time_callback` | `PhysicsStepTimeCallback*` | Колбэк времени шага |

## Создание и уничтожение

### `Create(bool mt, CObjectSpace*, CObjectList*, CRenderDeviceBase*)`

1. `LoadParams()` — читает `params.ini` (fixed_step, erp, cfm, gravity, iterations).
2. `Device().AddSeqFrame(this, mt)` — регистрация в планировщике кадров.
3. `dWorldSetAutoEnableDepthSF1(phWorld, 1e8)` — ODE solver params.
4. `ContactGroup = dJointGroupCreate(0)` — группа контактов.
5. `dWorldSetGravity(phWorld, 0, -Gravity(), 0)`.
6. `Mesh.Create(0, phWorld)` — `dCreateTriList` для статической коллизии.
7. `m_motion_ray = dCreateRayMotions(0)` — геометрия для трассеров.
8. `phBoundaries` = AABB уровня, `y1 -= 30`.
9. `CPHCollideValidator::Init()` — инициализация маски коллизий.
10. `StepNumIterations(phIterations)` — `dWorldSetQuickStepNumIterations`.
11. `SetStep(ph_console::ph_step_time)` — устанавливает fixed_step и пересчитывает ERP/CFM/spring/damping.

### `Destroy()`

1. `r_spatial.clear()`.
2. `Mesh.Destroy()` — `dGeomDestroy`.
3. `dGeomDestroy(m_motion_ray)`.
4. `dJointGroupEmpty/Destroy(ContactGroup)`.
5. `ContactFeedBacks.clear()`, `ContactEffectors.clear()`.
6. `dCloseODE()` — закрытие ODE.
7. `dCylinderClassUser = -1`, `dRayMotionsClassUser = -1` — сброс классов.
8. `Device().RemoveSeqFrame(this)`.

## Цикл обновления

### `OnFrame()` (pureFrame)

```
OnFrame() → FrameStep(Device().fTimeDelta)
```

Замеряется через `Device().StatPhysics()->Physics.Begin()/End()`.

### `FrameStep(dReal step)`

1. Если `IsFreezed()` — return.
2. `step *= phTimefactor` — множитель скорости физики (консольная переменная).
3. Аккумулирует `m_frame_time += step`.
4. Если `m_frame_time >= fixed_step`:
   - `it_number = floor(m_frame_time / fixed_step)`
   - `m_frame_time -= it_number * fixed_step` (остаток)
   - `b_frame_mark = !b_frame_mark` (флип для FrameTime(bool))
5. Цикл: `for (i = 0; i < it_number; ++i) Step()`.

**Fixed-step:** физика тикает с фиксированным шагом (`fixed_step`, по умолчанию 0.025с = 40 Гц), независимо от FPS. Остаток кадра аккумулируется в `m_frame_time`.

### `Step()` — один физический шаг

Порядок операций (критичен):

| # | Операция | Что делает |
|---|----------|------------|
| 1 | `disable_count` logic | Периодическая чистка `m_recently_disabled_objects` (каждые `L2frames` шагов) |
| 2 | `m_steps_num++`, `m_steps_short_num++` | Счётчики шагов |
| 3 | **Collide pass**: `for obj in m_objects: obj->Collide()` | Бродкаст коллизий: ray-motions (пули) + AABB query + static collision |
| 4 | **PhTune pass (objects)**: `for obj in m_objects: obj->PhTune(fixed_step)` | Корректировка перед решателем (скорости, силы) |
| 5 | **PhTune pass (update objects)**: `for obj in m_update_objects: obj->PhTune(fixed_step)` | То же для update-объектов |
| 6 | **m_update_callback->update_step()** | Колбэк xrGame (CPHCommander) |
| 7 | **IslandStep pass**: `for obj in m_objects: obj->IslandStep(fixed_step)` | ODE solver: `dWorldQuickStep` на остров |
| 8 | **IslandReinit + PhDataUpdate + spatial_move**: `for obj in m_objects: obj->IslandReinit(); obj->PhDataUpdate(fixed_step); obj->spatial_move()` | Разъединение островов, обновление данных, перемещение в spatial DB |
| 9 | **PhDataUpdate (update objects)**: `for obj in m_update_objects: obj->PhDataUpdate(fixed_step)` | Обновление update-объектов |
| 10 | `dJointGroupEmpty(ContactGroup)` | Очистка контактов (ПОСЛЕ PhDataUpdate — порядок критичен) |
| 11 | `ContactFeedBacks.empty()`, `ContactEffectors.empty()` | Очистка буферов контактов |
| 12 | `physics_step_time_callback` | Колбэк времени (для сети) |

### `StepTouch()` — облегчённый шаг

Только `Collide()` + `Island().Enable()` + `IslandReinit()` + `spatial_move()`. Без ODE solver. Используется для "touch" — обновление коллизий без физики.

## Freeze / UnFreeze

- `Freeze()`: переносит все объекты из `m_objects` → `m_freezed_objects`, вызывает `FreezeContent()` на каждом. `b_world_freezed = true`.
- `UnFreeze()`: обратная операция. `b_world_freezed = false`.
- Во время freeze `FrameStep()` сразу return.

## Gravity

- `SetGravity(float g)` — устанавливает `m_gravity` и `dWorldSetGravity(phWorld, 0, -g, 0)`.
- `Gravity()` — getter.
- Default: `default_world_gravity` (из `params.ini`).

## SetStep

`SetStep(float s)` устанавливает `fixed_step = s` и пересчитывает:
- `world_cfm = CFM(SPRING_S(...), DAMPING(...))`
- `world_erp = ERP(SPRING_S(...), DAMPING(...))`
- `world_spring = SPRING(world_cfm, world_erp)`
- `world_damping = DAMPING(world_cfm, world_erp)`

Формулы ERP/CFM/spring/damping — см. `PhysicsCommon.h` (§ ниже).

## Связи

- **xrGame** → `CPHWorld`: `create_physics_world`, `ph_world->AddObject`, `SetGravity`, `Freeze/UnFreeze`, `GetState` (сеть).
- **CPHObject** → `CPHWorld`: `activate()` → `AddObject`, `deactivate()` → `RemoveObject`.
- **CPHIsland** → `CPHWorld`: каждый объект имеет свой island (dxWorld), `IslandStep` вызывает `dWorldQuickStep`.
- **CPHCollideValidator** → `CPHWorld`: `Init()` при создании мира.
- **ODE** → `CPHWorld`: `dWorldSetGravity`, `dWorldSetERP/CFM`, `dJointGroupCreate/Empty/Destroy`, `dCloseODE`.

## Замечания к плану

- `CPHMesh` — wrapper вокруг `dCreateTriList`, хранит `dGeomID`. Используется для статической коллизии уровня.
- `m_motion_ray` — `dCreateRayMotions`, используется в `CPHObject::Collide()` для ray-motions (пули).
- `ContactFeedBacks` / `ContactEffectors` — static `CBlockAllocator` (128 элементов), буферы контактов.
- `phTimefactor` — множитель скорости физики (консольная переменная, `params.cpp`).
- `phIterations` — число итераций ODE solver (`dWorldSetQuickStepNumIterations`).

---

Порцион 1 (итерация 7). Далее: [ph-object.md](ph-object.md).
