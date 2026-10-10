# xrParticles — Actions: ядро и домены

> Порцион 2 итерации 4. Файлы: `particle_actions.h/.cpp`,
> `particle_actions_collection.h`, `particle_actions_collection.cpp` (каркас,
> ~1910 строк), `particle_actions_collection_io.cpp` (~507 строк).
> Полные разборы 29 подклассов — `actions-actions.md` + `actions-io.md`
> (страницы появятся в порционе 3).

## 1. Ответственность

Система действий (actions) — «физика» частиц. Эффект — это список
действий, каждое из которых на каждый шаг симуляции либо меняет состояние
частиц (`vel`, `pos`, `age`, цвет, размер), либо рождает/убивает частицы.
Страница покрывает:

- базовый класс `ParticleAction` и контейнер `ParticleActions`;
- **каркас** коллекции: общий шаблон 29 подклассов (макро `_METHODS`),
  контракт `Execute`/`Transform`/`Load`/`Save`, dual-domain паттерн
  (локальное/мировое пространство), закон сил, правила удаления;
- схему IO (общая для всех действий);
- специфичное: `PATurbulence` (SSE vs скалярный путь, мёртвый streamer).

**НЕ делает**: выбор и редактирование действий (редакторная библиотека
`EPA*`/`actions_token[]` — xrRender, [Частицы и wallmarks](../renderer/particles-wallmarks.md),
ит.3 порц.16), фабрика `CreateAction` (порцион 1, [PAPI и ядро](core.md)),
noise-функции (порцион 4, `noise-integration.md`).

## 2. Место в архитектуре

Часть модуля `xrParticles` (статическая библиотека, dllexport
`PARTICLES_API` закомментирован — [PAPI и ядро](core.md), §8). Действия
живут в списках `ParticleActions`, которые `CParticleManager` создаёт через
`CreateActionList` и заполняет через `CreateAction` (фабрика по
`PActionEnum`) или `LoadActions` (чтение из `.pe`). Вызываются только из
`CParticleManager::Update`/`Transform` (порцион 1).

## 3. Публичный API

| Символ                                  | Назначение                                                                                                                      |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `ParticleAction`                        | Базовый struct: `m_Flags` (Flags32), `type` (PActionEnum), 4 чистых виртуальных.                                                |
| `ParticleAction::ALLOW_ROTATE = (1<<1)` | Единственный флаг. `CParticleManager::Transform` при нём передаёт полный xform, иначе — только трансляцию.                      |
| `PAVec`                                 | `DEFINE_VECTOR(ParticleAction*, PAVec, PAVecIt)` — список действий (сырые указатели).                                           |
| `ParticleActions`                       | Контейнер: `clear`/`append`/`resize` (R_ASSERT `!m_bLocked`), `lock`/`unlock`, итераторы, `copy` (**без определения**, см. §8). |
| `PAAvoid` … `PATurbulence`              | **29 подклассов** действий (план ит.4 говорит «30» — в `particle_actions_collection.h` реально 29 struct).                      |
| `ParticleAction::Load/Save` (по умолч.) | Читают/пишут `m_Flags` (u32) + `type` (u32). Все подклассы сначала вызывают их.                                                 |

## 4. Внутреннее устройство

### `particle_actions.h` — базовый класс и контейнер

```cpp
struct PARTICLES_API ParticleAction
{
    enum { ALLOW_ROTATE = (1 << 1) };   // бит 0 не используется
    Flags32 m_Flags;                    // zero в ctor
    PActionEnum type;
    // чистые виртуальные:
    //   Execute(ParticleEffect*, const float dt, float& m_max)
    //   Transform(const Fmatrix&)
    //   Load(IReader&) / Save(IWriter&)
};
```

`ParticleActions` — «владельческий» вектор: `clear()` делает `xr_delete` на
каждом указателе. `m_bLocked` — мьютекс-флаг (не mutex): `lock()`/`unlock()`
с R_ASSERT состояния; `clear/append/resize` запрещены пока список заперт.
Запирает `CParticleManager` на время `Update`/`Transform`/`SaveActions`.

**`copy(ParticleActions* src)`** объявлен (particle_actions.h L67), но
**определения нет нигде в `src/`** — см. §8.

### `particle_actions.cpp`

5 строк (включает только `particle_actions.h`). Файл-заглушка: даже
`ParticleAction::Load/Save` в нём **нет** — их реализация лежит в
`particle_actions_collection_io.cpp` L7–17 (quasi-quirk, см. §8).

### Каркас `particle_actions_collection`

Макро в `particle_actions_collection.h`:

```cpp
#define _METHODS  virtual void Load(IReader& F); \
                  virtual void Save(IWriter& F); \
                  virtual void Execute(ParticleEffect* pe, const float dt, float& m_max); \
                  virtual void Transform(const Fmatrix& m);
```

Все 29 подклассов — `struct PARTICLES_API PA* : public ParticleAction { поля;
_METHODS; }`. Поля — только данные (константы действия), вся логика — в
`Execute`. Полный разбор каждого — порцион 3, здесь — общий шаблон.

**Контракт `Execute(effect, dt, tm_max)`**:

- прямой доступ `effect->p_count` / `effect->particles[i]` (layout `Particle`
  76 байт — [PAPI и ядро](core.md), §4);
- `tm_max` — длительность эффекта (в `CParticleManager::Update` передаётся
  `kill_old_time = 1.0f`); используют **только** `PAKillOld` (присваивает
  `tm_max = age_limit` — побочный эффект) и `PATargetColor`
  (окно `timeFrom*tm_max … timeTo*tm_max`);
- большинство действий меняют только `vel` (интегрирует `PAMove`).

**`PAMove` — интегратор**:

```cpp
m.age += dt;
m.posB = m.pos;          // m.velB = m.vel;  // закомментировано
m.pos  += m.vel * dt;
```

Порядок в списке действий задаёт семантику (например, `PAMove` после сил —
полупослеовательный Эйлер; `PACopyVertexB` перед `PAMove` фиксирует `posB`).

**Двойной домен (L/world)** — ключевой паттерн. Каждое геометрическое
действие хранит две копии домена/точки/направления: `xxxL` (локальное
пространство эффекта) и world `xxx`.

- **`L` — хранимые данные**; world-поля — производные;
- `Load` читает только world-версию и инициализирует `L = world`
  (`PAAvoid::Load`: `F.r(&position, sizeof(pDomain)); …; positionL = position;`);
- `Transform(m)` пересчитывает world из `L`: точки —
  `m.transform_tiny(w, L)`, направления/скорости — `m.transform_dir(d, L)`,
  домены — `d.transform(dL, m)` / `d.transform_dir(dL, m)`.

Мировое поле используется в `Execute` — так действия «следуют» за
перемещаемым эффектом (вызов `Transform` — из
`CParticleManager::Transform`, порцион 1). Примеры:
`PAAvoid::Transform` → `position.transform(positionL, m)`;
`PASource::Transform` → `position.transform` + `velocity.transform_dir`;
`PAJet` → `center` (tiny) + `acc` (transform_dir).

11 подклассов с `L`-полями: `PAAvoid, PABounce, PAExplosion, PAJet,
PAOrbitLine, PAOrbitPoint, PARandomAccel, PARandomDisplace,
PARandomVelocity, PASink, PASinkVelocity, PASource, PAVortex,
PATargetVelocity` (точнее — все, кроме действий без геометрии; пустой
`Transform(const Fmatrix&) { ; }` — у 13: `PACopyVertexB, PADamping,
PAFollow, PAGravitate, PAGravity, PAKillOld, PAMatchVelocity, PAMove,
PARestore, PASpeedLimit, PATargetColor, PATargetSize, PATargetRotate`).
`PATurbulence::Transform` — тоже пустой (поле `offset` не трансформируется).

**Закон сил**. Центральные действия:

```
vel += dir * (magdt / (rSqr + epsilon))                  // 1/r²
vel += dir * (magdt / (_sqrt(rSqr) * (rSqr + epsilon)))  // 1/r³ («normalize by 1/r»)
```

(`PAGravitate`, `PAMatchVelocity` — **O(n²) по парам** с третьим законом
Ньютона: `m.vel += acc; mj.vel -= acc`).

**Ограничение радиуса**: `max_radius == P_MAXFLOAT` (1e16, порцион 1) =
«безлимитное». Проверка `if (max_radiusSqr < P_MAXFLOAT)` разводит
ограниченный/безлимитный циклы — **почти идентичные копии веток**
(фосилия: в `PAOrbitPoint` комментарий «Avoids pipeline stalls» стоит в
ветке, а в `PAOrbitLine` — «Removed because it causes pipeline stalls» —
одинаковый код, разные комментарии).

**Правило удаления**. Действия, убивающие частицы (`PAKillOld`, `PASink`,
`PASinkVelocity`), обходят список **обратно** (`for i = p_count-1 … 0`),
потому что `ParticleEffect::Remove` — swap-with-back (порцион 1: «не менять
правило удаления»).

### `PATurbulence` — особый случай

Единственное действие, «раздвоенное» компиляцией
(`particle_actions_collection.cpp` L1677–1904):

- **`#ifndef _EDITOR`** (игровой путь): SSE-версия. Векторные хелперы
  `_mm_load_fvector`/`_mm_store_fvector` (ручная упаковка 3 float'а в
  `__m128` через `_mm_load_ss/_mm_unpacklo_ps/_mm_movelh_ps`), длина
  вектора — руками (`_mm_mul_ps/_mm_add_ss/_mm_sqrt_ss`), шум
  `fractalsum3` — скалярный (4 вызова на частицу),
  `#include "../xrCPU_Pipe/ttapi.h"` + `#pragma comment(lib,"xrCPU_Pipe.lib")`,
  profiling-скоуп `TAL_SCOPED_TASK_NAMED` при `_GPA_ENABLED`.
  **`magnitude * ps_particle_update_coeff`** — cvar из xrEngine масштабирует
  силу.
- **`#else`** (редактор, `#ifdef _EDITOR`): скалярная версия, без
  `ps_particle_update_coeff` (только `magnitude`), длина через
  `vel.magnitude()`.
- Обе версии: `vel += D*magnitude` и затем **нормализация до исходной
  длины** (турбулентность меняет только направление скорости).
- Ленивая инициализация шума: `static int noise_start = 1;
extern void noise3Init();` — вызывается один раз в начале `Execute`.
  Internals `noise3/fractalsum3` — порцион 4.
- **Мёртвый код**: `PATurbulenceExecuteStream(LPVOID)` (L1724–1791) —
  Win32-thread-callback (структура `TES_PARAMS`, `LPVOID`), копия
  SSE-тела, **не имеет ни одного вызывающего**. Legacy.

### IO (`particle_actions_collection_io.cpp`)

Единая схема для всех 29 действий:

1. `ParticleAction::Load/Save(F)` — `m_Flags` (u32) + `type` (u32);
2. затем — фиксированный порядок полей:
   - `pDomain` — бинарный `F.r(&x, sizeof(pDomain))` / `F.w(...)` (13
     таких в файле: PAAvoid, PABounce, PAJet ×2, PASink, PASinkVelocity,
     PARandom* ×3, PASource ×5 — нет, уточнение: PAAvoid, PABounce, PAJet
     (`acc`), PASink, PASinkVelocity, PARandomAccel/Displace/Velocity,
     PASource ×5 = 13);
   - векторы — `F.r_fvector3` / `F.w_fvector3`;
   - скаляры — `r_float`/`r_u32`/`r_s32`.

World-домены пишутся, `L`-копии восстанавливаются при `Load`
(`positionL = position`). `PAMove::Load` — только базовый вызов (полей нет).
`PATurbulence::Load` **не читает поле `age`** (оно в struct, но в IO его
нет — `age` живёт только в рантайме).

## 5. Взаимодействие

- **Кто вызывает**: только `CParticleManager` (порцион 1,
  [PAPI и ядро](core.md)): `CreateAction` (фабрика: `PActionEnum` →
  `xr_new<PA*>`, quirk DID-маппинга — там же), `LoadActions`/`SaveActions`
  (через виртуальные `Load`/`Save`), `Update` → `Execute` под `lock()`,
  `Transform` → `Transform(m)` под `lock()`, `PlayEffect`/`StopEffect` →
  прямое изменение полей `PASource::flSilent`, `PAExplosion::age`,
  `PATurbulence::age`.
- **Кого вызывают**:
  - `particle_core` — `pDomain::Generate/Within/transform/transform_dir`
    ([PAPI и ядро](core.md), §4); `NRand` (`PASource`);
  - `particle_effect` — `effect->Add` (`PASource`), `effect->Remove`
    (kill-действия), прямой доступ к `particles[]`;
  - xrCore — `IReader`/`IWriter`, `xr_delete`, `::Random` (через `drand48()`);
  - xrEngine — `ps_particle_update_coeff` (extern float, SSE-путь
    `PATurbulence`);
  - `noise.h/.cpp` — `noise3Init`/`fractalsum3` (`PATurbulence`, порцион 4);
  - `xrCPU_Pipe` — `ttapi.h` (профилирование, `_GPA_ENABLED`).
- **Редактор** (xrRender): `ParticleEffectActions.h/.cpp` — 30 классов
  `EPA*` (одинаковые имена без `PA`), `actions_token[]`,
  `PARTICLE_ACTION_VERSION 0x0001`, quirk DID-маппинга — покрыто в
  [Частицы и wallmarks](../renderer/particles-wallmarks.md) (ит.3 порц.16).
  Та страница — редакторская (compile-time) сторона; эта — рантайм.

## 6. Потоки данных

```mermaid
sequenceDiagram
    participant PE as CParticleEffect (xrRender)
    participant PM as CParticleManager
    participant PA as ParticleActions (lock)
    participant A as PA*::Execute
    participant EF as ParticleEffect
    Note over PM: LoadActions (1 раз)
    PM->>PA: CreateAction + Load(F) per action
    Note over PE,EF: каждый fixed-step (порц.1, ит.3 порц.16)
    PE->>PM: Update(eff, alist, dt)
    PM->>PA: lock
    loop per action (порядок = семантика)
        PA->>A: Execute(pe, dt, kill_old_time)
        A->>EF: pos/vel/age/size/цвет, Add/Remove
    end
    PM->>PA: unlock
    Note over PM: при смене xform эффекта
    PE->>PM: Transform(eff, alist, M, vel)
    PM->>PA: lock; Transform(m) per action
    PA->>PA: world-поля = f(L-поля, m); PASource.parent_vel
    PM->>PA: unlock
```

## 7. Конфигурация

- cvar'ов в модуле нет. Внешние:
  - `ps_particle_update_coeff` (xrEngine) — множитель `magnitude` в
    SSE-пути `PATurbulence` (в рантайме `CParticleEffect::OnFrame` —
    ит.3 порц.16 — тот же cvar масштабирует и `uDT_STEP`);
  - `STEP_DEFAULT 0.033F` — локальный `#define` в
    `particle_actions_collection.cpp` L1480, нормирует
    `PATargetColor` на шаг 33 мс (аналог `m_uStep` в `CPEDef`).

## 8. Известные ограничения и дебаг

1. **`ParticleActions::copy` — объявлен, но не определён** (никакого
   `ParticleActions::copy` в `src/`; `particle_actions.cpp` — 5 строк).
   Не используется — dead declaration; при вызове будет linker error.
2. **`ParticleAction::Load/Save` по умолчанию реализованы в
   `particle_actions_collection_io.cpp`** (L7–17), а не в
   `particle_actions.cpp` — файл-«ядро» действий фактически пуст.
3. **TBB**: `<tbb/parallel_for.h>` / `<tbb/blocked_range.h>` включены
   (particle_actions_collection.cpp L7–8), но **не используются** — все
   циклы `Execute` последовательные (тот же quirk, что
   `ParticleRenderStream` в ит.3 порц.16).
4. **Копии веток `max_radius < P_MAXFLOAT`** — два почти идентичных
   цикла в каждом центральном действии; в `PAOrbitPoint`/`PAOrbitLine`
   комментарии «Avoids/Removed because it causes pipeline stalls»
   на одинаковом коде — fossil.
5. **`PATurbulence`**: (а) мёртвый `PATurbulenceExecuteStream`
   (Win32-thread-callback, 0 вызовов); (б) расхождение путей — SSE
   масштабирует на `ps_particle_update_coeff`, скалярный (`_EDITOR`) — нет;
   (в) поле `age` не читается в `Load`; (г) `Transform` — no-op (
   `offset` не трансформируется).
6. **`PAKillOld` мутирует `tm_max`** (side effect на аргумент
   `CParticleManager::Update`).
7. **`PAMove`**: `m.velB = m.vel;` закомментирован (нет действия,
   использующего `velB` — поле `velB` в `Particle` не существует,
   `Particle` — 76 B без него); `PACopyVertexB` — блок копирования `vel`
   закомментирован (аналогично).
8. **`PAGravity::Transform` — no-op**: `direction` (pVector) хранится
   только в world-копии (нет `directionL`) — направление гравитации
   **не следует за вращением эффекта**, при трансформации остаётся
   в мировых координатах загрузки.
9. **O(n²)**: `PAGravitate`/`PAMatchVelocity` — попарные циклы
   (без пространственного разбиения) — тяжёлые для больших эффектов.
10. **`PAMove`** — единственный интегратор: порядок действий в списке
    = порядок применения (см. §4); нет встроенного «настройки интегратора».
11. **29 vs 30**: план ит.4 и редактор (`actions_token[]`) говорят про
    30 действий; в `particle_actions_collection.h` **29 struct**.
    `PActionEnum` (порцион 1) — 31 код, включая
    `PACallActionListID_obsolette`-заглушку; DID-коды
    (`PATargetRotateDID`, `PATargetVelocityDID`) мапятся фабрикой на
    недвойственные классы (quirk, порцион 1 + ит.3 порц.16).
