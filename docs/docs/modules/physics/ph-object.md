# CPHObject — базовый ODE-объект

> Итерация 7, порцион 1. Файлы: `PHObject.h` (148 строк), `PHObject.cpp` (293 строк), `PHUpdateObject.h` (25), `PHDefs.h` (20), `PHItemList.h` (109), `PHIsland.h` (286), `phvalide.h` (17), `ph_valid_ode.h` (45), `ode_redefine.h` (33), `ode_include.h` (9), `SpaceUtils.h` (20), `matrix_utils.h` (105), `MathUtils.h/cpp` (467+), `params.h/cpp`, `PhysicsCommon.h` (61).

`CPHObject` — базовый класс для всех физических объектов (shell'ы, персонажи). Наследники: `CPHShell` (порц. 3), `CPHCharacter` (порц. 5).

## Наследование

```
CPHObject : ISpatial (xrCDB)
CPHUpdateObject (без ISpatial — только PhDataUpdate/PhTune)
```

`ISpatial` — пространственный индекс (AABB + sphere) для broad-phase коллизий. `STYPE_PHYSIC` флаг в `spatial.type`.

## Поля

| Поле | Тип | Назначение |
|------|-----|------------|
| `m_flags` | `Flags8` | `st_activated`, `st_freezed`, `st_dirty`, `st_net_interpolation`, `fl_ray_motions`, `st_recently_deactivated`, `is_deadbody` |
| `m_island` | `CPHIsland` | ODE-остров (dxWorld) — тела и суставы этого объекта |
| `m_collide_bits` | `CLBits` (u32) | Маска классов коллизий (какой класс с каким) |
| `m_collide_class_bits` | `_flags<CLClassBits>` | Классы, с которыми объект коллидирует |
| `m_check_count` | `u8` | Счётчик для recently_deactivated |
| `AABB` | `Fvector` (half-size) | Полуразмеры AABB (для broad-phase) |

### Флаги `m_flags`

| Флаг | Значение |
|------|----------|
| `st_activated` | Объект активен (в `m_objects` мира) |
| `st_freezed` | Объект заморожен |
| `st_dirty` | AABB изменился (нужна коллизия) |
| `st_net_interpolation` | Включена сетевая интерполяция |
| `fl_ray_motions` | Объект участвует в ray-motions (пули) |
| `st_recently_deactivated` | Недавно деактивирован (отложенная чистка) |
| `is_deadbody` | Мёртвое тело (специальная коллизия) |

## Виртуальные методы

| Метод | Назначение |
|-------|------------|
| `dSpacedGeom()` | dGeomID объекта (для spatial query) |
| `get_spatial_params()` | Обновить AABB/sphere для ISpatial |
| `spatial_register()` | Регистрация в spatial DB |
| `spatial_move()` | Перемещение в spatial DB (после PhDataUpdate) |
| `PhDataUpdate(dReal step)` | Обновление данных после ODE step |
| `PhTune(dReal step)` | Корректировка перед ODE step |
| `InitContact(dContact*, bool&, u16, u16)` | Инициализация контакта (материалы) |
| `Collide()` | Бродкаст коллизий |
| `near_callback(CPHObject*)` | Колбэк при близости |
| `RMotionsQuery(qResultVec&)` | Query для ray-motions |
| `MoveStorage()` | `CPHMoveStorage*` (tracers) |
| `CastType()` | `tpNotDefinite` / `tpShell` / `tpCharacter` / `tpStaticShell` |
| `get_elements_number()` | Число элементов |
| `get_element_sync(u16)` | `CPHSynchronize*` для элемента (сеть) |
| `FreezeContent()` / `UnFreezeContent()` | Freeze/unfreeze контента |
| `vis_update_activate()` / `vis_update_deactivate()` | Визуальное обновление при активации |

## Активация / Деактивация

### `activate()`

1. Если уже активен — return.
2. Если frozen — `UnFreeze()`, return.
3. Если recently_deactivated — `remove_from_recently_deactivated()`.
4. `ph_world->AddObject(this)` — добавление в мир.
5. `vis_update_activate()`.
6. `st_activated = true`.

### `deactivate()`

1. Если не активен — return.
2. `VERIFY(!m_island.IsActive())` — нельзя во время processing.
3. `ph_world->RemoveObject(...)`.
4. `vis_update_deactivate()`.
5. `st_activated = false`.

### Recently deactivated

- `put_in_recently_deactivated()` — добавляет в `m_recently_disabled_objects`, `m_check_count = ph_console::ph_tri_clear_disable_count`.
- `check_recently_deactivated()` — вызывается в `CPHWorld::Step()`: decrement счётчик, при 0 → `ClearRecentlyDeactivated()` + `remove_from_recently_deactivated()`.
- Назначение: отложенная чистка триангуляций (tri-clear) после деактивации.

## Коллизии

### `Collide()`

1. **Ray-motions** (если `fl_ray_motions`):
   - Для каждого трассера в `MoveStorage()`:
     - `q_ray` в `g_SpatialSpacePhysic` (spatial DB).
     - Для каждого результата: `dGeomRayMotionSetGeom`, `dGeomRayMotionsSet`, `NearCallback(this, obj2, motion_ray, obj2->dSpacedGeom())`.
2. **CollideDynamics()**:
   - `q_box` в `g_SpatialSpacePhysic` (AABB query).
   - Для каждого результата (не self, `st_dirty`):
     - Если `is_deadbody` — специальная логика (CPHCharacter).
     - `NearCallback(this, obj2, dSpacedGeom(), obj2->dSpacedGeom())`.
3. **Static collision** (если `CPHCollideValidator::DoCollideStatic(*this)`):
   - `CollideStatic(dSpacedGeom(), this)` — коллизия с mesh уровня.
4. `st_dirty = false`.

### `NearCallback` / `CollideStatic`

Объявлены в `Physics.h`, реализованы в `Physics.cpp`:
- `NearCallback(CPHObject* obj1, CPHObject* obj2, dGeomID o1, dGeomID o2)` — ODE near-callback: создаёт контакты, вызывает `InitContact`, добавляет в `ContactGroup`.
- `CollideStatic(dGeomID o2, CPHObject* obj2)` — коллизия с статической геометрией (mesh).

## CPHIsland

Каждый `CPHObject` имеет собственный `CPHIsland` (наследует `dxWorld` — ODE world). Это позволяет:
- Группировать тела/суставы по объектам.
- Мержить острова при создании суставов (Merge/Unmerge).
- Вызывать `dWorldQuickStep` на каждом острове отдельно.

### Ключевые методы CPHIsland

| Метод | Назначение |
|-------|------------|
| `Init()` | Сброс: nj=nb=0, пустые списки |
| `AddBody/RemoveBody` | Добавление/удаление тела |
| `AddJoint/RemoveJoint` | Добавление/удаление сустава |
| `ConnectJoint/DisconnectJoint` | Подключение/отключение сустава (без изменения счётчиков) |
| `ConnectBody/DisconnectBody` | Подключение/отключение тела |
| `Merge(CPHIsland*)` | Слияние двух островов (при создании сустава) |
| `Unmerge()` | Разделение (после Step) |
| `Step(dReal step)` | `dWorldQuickStep` |
| `Enable()` | Включение острова |
| `Repair()` | Ремонт (валидация) |
| `CheckSize()` | nj < 1500 && nb < 500 |
| `CanMerge(CPHIsland*, int&)` | Можно ли мержить |

### Лимиты

- `JOINTS_LIMIT = 1500` — максимум суставов в острове.
- `BODIES_LIMIT = 500` — максимум тел в острове.

### CPHIslandFlags

- `stActive` — остров активен.
- `flPrefereExactIntegration` — предпочитать точную интеграцию.
- `flForceExactIntegration` — принудительная точная интеграция (перекрывает бюджет суставов).

## CPHUpdateObject

Базовый класс для объектов, которые только обновляют данные (без body/geom в ODE).

```cpp
class CPHUpdateObject {
    bool b_activated;
    virtual void PhDataUpdate(float step) = 0;
    virtual void PhTune(float step) = 0;
    virtual void NetRelcase(CPhysicsShell* s);
};
```

Хранится в `CPHWorld::m_update_objects`. Тикает в `Step()`: PhTune + PhDataUpdate.

## PHDefs.h — typedef'ы

```cpp
typedef std::pair<CPhysicsShell*, u16> shell_root;
DEFINE_VECTOR(CPHElement*, ELEMENT_STORAGE, ELEMENT_I)
DEFINE_VECTOR(CPHJoint*, JOINT_STORAGE, JOINT_I)
DEFINE_VECTOR(shell_root, PHSHELL_PAIR_VECTOR, SHELL_PAIR_I)
```

## PHItemList.h — linked list

`CPHItemList<T>` — intrusive linked list (без выделений):
- `DECLARE_PHLIST_ITEM(class)` — добавляет `next`/`tome` в класс.
- `push_back`, `erase`, `move_items`, `empty`, `begin/end`.
- `CPHItemStack<T>` — наследует, добавляет `stack_pos`.

Используется для `m_objects`, `m_freezed_objects`, `m_update_objects` в `CPHWorld`.

## phvalide.h — валидация позиций

```cpp
XRPHYSICS_API bool valid_pos(const Fvector& P);
XRPHYSICS_API const Fbox& ph_boundaries();
#define VERIFY_BOUNDARIES(pos, bounds, obj)  // DEBUG: проверка что позиция в границах
```

`valid_pos(P, B)` — проверка что точка в расширенном AABB (grow 100000).

## ph_valid_ode.h — валидация ODE-состояний

```cpp
IC BOOL dV_valid(const dReal* v)      // все 3 компонента finite
IC BOOL dM_valid(const dReal* m)      // 3x3 матрица finite
IC BOOL dV4_valid(const dReal* v4)    // 4 компонента finite
IC BOOL dQ_valid(const dReal* q)      // quaternion finite
IC BOOL dMass_valide(const dMass* m)  // mass + center + inertia finite
IC BOOL dBodyStateValide(const dBodyID body)  // полный body state
```

## ode_include.h / ode_redefine.h

- `ode_include.h`: `#define dSINGLE` (float, не double), `#include "ode/ode.h"`.
- `ode_redefine.h`: переопределение `dSqrt`, `dRecipSqrt`, `dSin`, `dCos`, `dFabs` на `float`-версии (только при `XRPHYSICS_EXPORTS`).

## SpaceUtils.h

```cpp
IC void spatialParsFromDGeom(dGeomID d_space, Fvector& center, Fvector& AABB, float& radius)
```

Извлекает AABB из `dGeomID` → центр, полуразмеры, радиус сферы.

## matrix_utils.h

Утилиты для матриц/квaternion:
- `clamp_rotation(Fquaternion&, float)` — ограничение угла поворота.
- `clamp_rotation(Fmatrix&, float)` — то же для матрицы.
- `clamp_change(Fmatrix&, const Fmatrix&, float ml, float ma, float tl, float ta)` — проверка/ограничение изменения матрицы.
- `cmp_matrix(...)` — сравнение двух матриц (линейное + угловое).
- `matrix_diff(...)` — разница как линейная + угловая скорость.

## MathUtils.h

Утилиты:
- `cast_fp` / `cast_fv` — каст Fvector ↔ float*.
- `dXZMag`, `dXZDot`, `dXZDotNormalized` — XZ-плоскость.
- `dVector*` — операции с dReal* (Set, Add, Sub, Mul, Interpolate, Deviation).
- `dMatrixSmallDeviation` — малое отклонение матрицы.
- `twoq_2w` — угловая скорость из двух quaternion.
- `to_mag_and_dir` — нормализация.
- `prg_pos_on_axis/plane/normal` — проекция.
- `TransferenceToThrowVel`, `ThrowMinVelTime`, `TransferenceAndThrowVelToTgA` — баллистика (бросок).
- `SInertVal` — инерционное сглаживание.
- `DET(Fmatrix&)` — определитель.
- `valid_pos(P, B)` — проверка позиции в AABB.
- `VERIFY_RMATRIX(M)` — DEBUG: проверка что матрица ортогональная (det ≈ 1).

## params.h / params.cpp

```cpp
extern float object_damage_factor;
void LoadParams();  // читает params.ini
```

## PhysicsCommon.h

Константы и формулы ODE solver:
- `default_l_limit`, `default_w_limit` — лимиты скоростей.
- `base_fixed_step`, `base_erp`, `base_cfm` — базовые параметры.
- `fixed_step` — текущий шаг (устанавливается `SetStep`).
- `world_cfm`, `world_erp`, `world_spring`, `world_damping` — параметры мира.
- `phTimefactor` — множитель скорости.
- `phIterations` — итерации solver.
- `default_world_gravity` — гравитация.
- Формулы: `ERP_S`, `CFM_S`, `SPRING_S`, `DAMPING` — пересчёт ERP/CFM/spring/damping от `k_p`, `k_d`, `step`.

## Связи

- **CPHWorld** → `CPHObject`: `AddObject`/`RemoveObject`, `Collide`, `PhTune`, `IslandStep`, `PhDataUpdate`, `spatial_move`.
- **CPHShell** (порц. 3) → `CPHObject`: наследует, реализует `dSpacedGeom`, `PhDataUpdate`, `PhTune`, `InitContact`.
- **CPHCharacter** (порц. 5) → `CPHObject`: наследует, добавляет body, interpolation, mass.
- **ISpatial** (xrCDB): `spatial_register`, `spatial_move`, `STYPE_PHYSIC`.
- **CPHCollideValidator** (порц. 6): `InitObject`, `DoCollideStatic`.

---

Порцион 1 (итерация 7). Далее: порц. 2 — C++ API и Lua-скрипты ([shell-api.md](shell-api.md)).
