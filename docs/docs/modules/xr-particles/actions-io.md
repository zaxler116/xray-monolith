# Actions: IO

`particle_actions_collection_io.cpp` (507 строк) — `Load`/`Save` всех
29 подклассов `ParticleAction`. Файл `.pe` хранит действия в порядке
выполнения; порядок `Load`/`Save` внутри подкласса — **контракт
формата** (бинарный, без версий — сдвиг порядка = поломка файлов).

## Базовый заголовок

```cpp
void ParticleAction::Load(IReader& F) {
    m_Flags.assign(F.r_u32());
    type = (PActionEnum)F.r_u32();
}
void ParticleAction::Save(IWriter& F) {
    F.w_u32(m_Flags.get());
    F.w_u32(type);
}
```

2 × u32: флаги + тип. Подклассы **всегда** начинают с
`ParticleAction::Load/Save(F)` и дописывают свои поля.

## Примитивы IO

| Тип данных | Read | Write |
| --- | --- | --- |
| `pVector` (3 float) | `F.r_fvector3(v)` | `F.w_fvector3(v)` |
| `float` | `F.r_float()` | `F.w_float(f)` |
| `u32` | `F.r_u32()` | `F.w_u32(u)` |
| `s32` | `F.r_s32()` | `F.w_s32(s)` |
| `pDomain` (структура) | `F.r(&d, sizeof(pDomain))` | `F.w(&d, sizeof(pDomain))` |

`pDomain` сериализуется **сырьём** (`sizeof` = 4×float + union-поля),
без вложенного `Load` — потому что `pDomain` — plain data (см.
[core.md](core.md)).

## Dual-domain при Load

Все действия с world/L-копиями делают после чтения world-поля:

```cpp
F.r(&position, sizeof(pDomain));   // world
...
positionL = position;             // L = world (копия)
```

`Transform(m)` затем перевычисляет `L` из world при переносе эффекта.
При `Save` пишется **только world** — `L`-копии не сериализуются
(восстанавливаются `Load`-копией).

## Поля каждого подкласса (порядок в файле)

| Подкласс | Поля (в порядке записи) |
| --- | --- |
| `PAAvoid` | `position`(pDomain), `look_ahead`, `magnitude`, `epsilon` (f) |
| `PABounce` | `position`, `oneMinusFriction`, `resilience`, `cutoffSqr` (f) |
| `PACopyVertexB` | `copy_pos` (u32) |
| `PADamping` | `damping` (v3), `vlowSqr`, `vhighSqr` (f) |
| `PAExplosion` | `center` (v3), `velocity`, `magnitude`, `stdev`, `age`, `epsilon` (f) |
| `PAFollow` | `magnitude`, `epsilon`, `max_radius` (f) |
| `PAGravitate` | `magnitude`, `epsilon`, `max_radius` (f) |
| `PAGravity` | `direction` (v3) |
| `PAJet` | `center` (v3), `acc` (pDomain), `magnitude`, `epsilon`, `max_radius` (f) |
| `PAKillOld` | `age_limit` (f), `kill_less_than` (u32) |
| `PAMatchVelocity` | `magnitude`, `epsilon`, `max_radius` (f) |
| `PAMove` | — (только базовый заголовок) |
| `PAOrbitLine` | `p`, `axis` (v3), `magnitude`, `epsilon`, `max_radius` (f) |
| `PAOrbitPoint` | `center` (v3), `magnitude`, `epsilon`, `max_radius` (f) |
| `PARandomAccel` | `gen_acc` (pDomain) |
| `PARandomDisplace` | `gen_disp` (pDomain) |
| `PARandomVelocity` | `gen_vel` (pDomain) |
| `PARestore` | `time_left` (f) |
| `PAScatter` | `center` (v3), `magnitude`, `epsilon`, `max_radius` (f) |
| `PASink` | `kill_inside` (u32), `position` (pDomain) |
| `PASinkVelocity` | `kill_inside` (u32), `velocity` (pDomain) |
| `PASpeedLimit` | `min_speed`, `max_speed` (f) |
| `PASource` | `position`, `velocity`, `rot`, `size`, `color` (pDomain ×5), `alpha`, `particle_rate`, `age`, `age_sigma` (f), `parent_vel` (v3), `parent_motion` (f) |
| `PATargetColor` | `color` (v3), `alpha`, `scale`, `timeFrom`, `timeTo` (f) |
| `PATargetSize` | `size`, `scale` (v3 ×2) |
| `PATargetRotate` | `rot` (v3), `scale` (f) |
| `PATargetVelocity` | `velocity` (v3), `scale` (f) |
| `PAVortex` | `center`, `axis` (v3 ×2), `magnitude`, `epsilon`, `max_radius` (f) |
| `PATurbulence` | `frequency` (f), `octaves` (**s32**), `magnitude`, `epsilon` (f), `offset` (v3) |

## Dual-domain: кто копирует L

| Копирует `L = world` при Load | Не имеет L (world-only или скаляры) |
| --- | --- |
| `PAAvoid`, `PABounce`, `PAExplosion` (center), `PAGravity` (direction), `PAJet` (center + acc), `PAOrbitLine` (p + axis), `PAOrbitPoint` (center), `PARandomAccel/Displace/Velocity` (gen_*), `PAScatter` (center), `PASink` (position), `PASinkVelocity` (velocity), `PASource` (position + velocity), `PATargetVelocity` (velocity), `PAVortex` (center + axis) | `PACopyVertexB`, `PADamping`, `PAFollow`, `PAGravitate`, `PAKillOld`, `PAMatchVelocity`, `PAMove`, `PARestore`, `PASpeedLimit`, `PATargetColor`, `PATargetSize`, `PATargetRotate`, `PATurbulence` |

Особенность `PASource`: из 5 pDomain копируются **только 2**
(`position`, `velocity`); `rot`/`size`/`color` — world-only
(`Transform` трансформирует только эти же 2).

## Quirks

1. **`PATurbulence::Load` не читает `age`** — в .cpp `age += dt`
   (Execute), в .h поле есть, но IO его пропускает. Загруженная
   турбулентность всегда стартует с `age = 0` (значением, данным
   конструктором/мусором). `Save` тоже не пишет `age` — формат
   стабилен, но состояние теряется.
2. **`PACopyVertexB`** — `copy_vel` в .cpp закомментирован, в IO нет
   поля; в .h поле `copy_vel`, вероятно, не существует (только
   `copy_pos`) — формат = 1 × u32.
3. **`PASource`** — `parent_motion` пишется/читается, но в
   `Execute` не используется (мёртвое поле формата).
4. **`PAMove`** — единственный подкласс без полей: `Load`/`Save` =
   только базовые 2 × u32.
5. **`octaves` — единственный `s32`** во всём IO (остальное — f/u32).
6. **`PASink`/`PASinkVelocity`** — `kill_inside` (u32) пишется **до**
   pDomain (у `PAAvoid`/`PABounce` pDomain первый) — порядок полей
   уникален на подкласс, перестановка = несовместимость.

## Связь с форматом `.pe`

Действия читаются в `ParticleEffect::Load` (см.
[core.md](core.md)): заголовок эффекта → список действий
(каждое = базовые 2 × u32 + поля подкласса) → остальные чанки.
`type` (PActionEnum) — диспетчер: фабрика создаёт объект подкласса
**до** вызова его `Load` (поэтому в IO нет виртуального dispatch —
конкретный `Load` уже выбран по `type`).

См. также: [29 подклассов](actions-actions.md),
[каркас](actions.md), [ядро](core.md).
