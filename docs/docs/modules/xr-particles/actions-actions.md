# Actions: 29 подклассов

`particle_actions_collection.cpp` (55 KB, ~1900 строк) — реализации
`Execute`/`Transform` всех 29 подклассов `ParticleAction` (декларации — в
[particle_actions_collection.h](particle_actions_collection.h), каркас — в
[actions.md](actions.md)). Здесь — только конкретика каждого подкласса;
общие паттерны (dual-domain, 1/r², обратный обход, `PAMove`) — там.

## Классификация

| Группа | Подклассы |
| --- | --- |
| Столкновения | `PAAvoid`, `PABounce` |
| Интегратор / вспомогательные | `PAMove`, `PACopyVertexB`, `PAKillOld`, `PARestore` |
| Убийство (sinks) | `PASink`, `PASinkVelocity` |
| Гравитация / притяжение | `PAGravity`, `PAGravitate`, `PAOrbitPoint`, `PAOrbitLine`, `PAJet`, `PAFollow`, `PAMatchVelocity` |
| Отталкивание | `PAScatter`, `PAExplosion`, `PAVortex` |
| Случайность | `PARandomAccel`, `PARandomVelocity`, `PARandomDisplace` |
| Цели (target-лерпы) | `PATargetColor`, `PATargetSize`, `PATargetRotate`, `PATargetVelocity` |
| Скорость | `PADamping`, `PASpeedLimit` |
| Источник | `PASource` |
| Турбулентность | `PATurbulence` |

`Transform` тривиален в 14 из 29 (пустой `{ }` или одна `transform_tiny`/
`transform_dir`/`position.transform`) — не приводится ниже; приведены
только неочевидные.

## Столкновения

### PAAvoid — «отворачивание» от домена

Поворачивает **направление** скорости частиц от домена, сохраняя её длину
(не отражает, не добавляет энергию). `switch (position.type)` по всем 5
доменам; общий приём: `tmp = S * (magdt / (t*t + epsilon)) + Vn; vel = tmp *
(vm / tmp.length())` — «вектор безопасности» `S` смешивается с текущей
направкой, длина не меняется.

- **PDPlane**: `dist = pos·n + d` (`p2` = нормаль, `radius1` = d);
  `look_ahead < P_MAXFLOAT` — действует только при `dist < look_ahead`,
  иначе — все частицы; `S = n`, вес `1/(dist²+ε)`.
- **PDRectangle / PDTriangle**: аналитическое пересечение отрезка
  `pos → pos + vel·dt·look_ahead` с плоскостью (`distold*distnew >= 0` →
  пропускаем); барицентрические координаты по инверсной базе
  (`s1/s2` через `w = u×v`); вне области — `continue`. `S` = нормаль к
  ближайшей **рёберной прямой** (у прямоугольника — 4 ребра, у треугольника
  3; у прямоугольника `gofs/fofs` вычислены одинаково — copy-paste,
  фактически 3 ребра). Вес `1/(t²+ε)`.
- **PDDisc**: пересечение с плоскостью + `rad ∈ [r2², r1²]`;
  `S = offset` (радиальный).
- **PDSphere**: ray-sphere (`disc = r² − |L|² + v²`; `t = v − √disc`),
  отбор `t ∈ [0, vm·look_ahead]`; `S` через два cross-произведения
  (`C = Vn ^ L; S = Vn ^ C`) — касательное к сфере направление.

`Transform`: `position.transform(positionL, m)` (dual-domain).

### PABounce — отражение

Тот же аналитический пересечение, но меняет скорость:
`vn = n·(V·n)` (нормаль), `vt = V − vn` (тангенс),
`vel = vt·oneMinusFriction − vn·resilience`; если `|vt|² <= cutoffSqr`
— трение не применяется (`vel = vt − vn·resilience`).

- **PDTriangle / PDDisc / PDRectangle** — как в `PAAvoid` (те же формулы
  `s1/s2`, барицентр, `phit`), затем разложение `vn/vt`.
- **PDPlane** — без поиска точки пересечения (плоскость бесконечна).
- **PDSphere** — **нет** ray-sphere: проверяется `position.Within(pnext)`
  (следующая позиция внутри), нормаль `n = normalize(pos − p1)`; два
  режима: частица уже внутри (`pinside`) → просто разворот `vn`
  (отталкивание из ловушки); снаружи → полный bounce с трением/ресил.

## Интегратор и вспомогательные

### PAMove — единственный интегратор

```cpp
m.age += dt;
m.posB = m.pos;          // предыдущая позиция (для PARestore)
m.pos  += m.vel * dt;    // (m.velB = m.vel; — закомментировано)
```

Без полей, IO = только базовый. Порядок действий в `.pe` важен:
`PAMove` должен идти после сил и до sinks.

### PACopyVertexB — дублирование позиции

`if (copy_pos) m.posB = m.pos;` — блок `velB` закомментирован.
`copy_vel` в файлах не читается (quirk IO, см. [actions-io.md](actions-io.md)).

### PAKillOld — по возрасту

**Side effect: `tm_max = age_limit`** (мутирует аргумент).
Условие `!((m.age < age_limit) ^ kill_less_than)` — XOR-инверсия:
`kill_less_than` — убить **моложе** `age_limit`, иначе — старше.
Обратный обход (удаление swap-with-back, см. [core.md](core.md)).

### PARestore — возврат в исходные позиции

По `time_left` (секунд) подводит частицы к `posB` (записываемому
`PAMove`/`PASource`) по квадратичной траектории `v(t) = a·t + b + c`
(коэффициенты `a`/`b` вычислены inline по осям; helper `_pconstrain`
в файле закомментирован `#if 0` — его формулы встроены в код):

```
b = (−2·t·v + 3·posB − 3·pos) · 2dt/t²
a = ( t·v − posB − posB + pos + pos) · 3dt²/t³     // = t·v − 2·posB + 2·pos
vel += a + b
```

`time_left -= dt` **каждый тик** — действие «расходует» себя само;
по достижении 0 — принудительно `pos = posB, vel = 0` (и дальше
каждый кадр держит частицы на месте).

## Убийство (sinks)

### PASink / PASinkVelocity

Обратный обход; `if (!(position.Within(m.pos) ^ kill_inside)) Remove(i)` —
`kill_inside` = убить **внутри** домена, иначе — снаружи. `PASink` — по
`pos`, `PASinkVelocity` — по `vel` (домен над векторами скоростей).
`Transform`: `position.transform` / `velocity.transform_dir`.

## Гравитация и притяжение

Все используют «закон сил» `magdt / (√r² · (r² + ε))` (эффективно
~1/r² с soften-ε):

### PAGravity — постоянная гравитация

`vel += direction * dt` — без масштабирования на расстояние, без
`magnitude`. `Transform` пустой (направление локальное — world/L-копия
не нужна).

### PAGravitate — попарное притяжение, O(n²)

Двойной цикл `j = i+1..n`, `tohimlenSqr += EPS_S`;
`acc = tohim * (magdt / (√L²·(L²+ε)))`; `m.vel += acc; mj.vel -= acc` —
3-й закон Ньютона (импульс сохраняется). `max_radius < P_MAXFLOAT` —
включает отсев по радиусу; иначе — все пары.

### PAMatchVelocity — выравнивание скоростей, O(n²)

Та же пара, но `acc = mj.vel * (magdt / (L² + ε))`; `m.vel += acc;
mj.vel -= acc` — среднее скоростей сдвигается в обе стороны
(импульс «суммарного движения» сохраняется, относительный гаснет).

### PAFollow — «хвост»

`i < n−1`: притяжение к **следующей** частице в списке (`i+1`),
односторонне (без `-=` — это не ньютоновская пара). Используется
для «верёвок»/цепочек.

### PAOrbitPoint — притяжение к точке

`dir = center − pos`; `vel += dir * (magdt / (√r² + (r² + ε)))` —
здесь знаменатель **сумма** `√r² + r²` (не произведение, как в
`PAGravitate`) — сила убывает как ~1/r³. `max_radius` — отсев.

### PAOrbitLine — притяжение к прямой

`f = pos − p` (от начала линии), `w = axis·(f·axis)` (проекция),
`into = w − f` (вектор к ближайшей точке прямой); тот же знаменатель
`√r² + (r²+ε)`.

### PAJet — ускорение вдоль домена

`acc.Generate(accel)` (домен-источник ускорения, см. `pDomain::Generate`
в [core.md](core.md)); `vel += accel * (magdt / (r² + ε))` — ~1/r² без
`√`. `Transform`: `transform_tiny(center)` + `acc.transform_dir`.

## Отталкивание

### PAScatter — разлёт от центра

`accel = dir / √r²` (единичный вектор от `center`);
`vel += accel * (magdt / (r² + ε))`. `acc.Generate` закомментирован.

### PAExplosion — бегущая волна

Гaussian-оболочка радиусом `radius = velocity * age`, растущая со
временем (`age += dt` **в самом Execute** — состояние действия):

```
Gd = exp(−½·((radius − dist)/stdev)²) / (√2π·stdev)   // ONEOVERSQRT2PI = 1/√(2π)
vel += dir * (Gd * magdt / ((dist + EPS) * (distSqr + epsilon)))
```

Частицы толкаются от центра только когда оболочка проходит мимо
(`|dist − radius| < ~stdev`).

### PAVortex — вихрь (вращение, а не ускорение)

**Меняет `pos` напрямую** (не `vel`): вектор `offset = pos − center`
раскладывается в базис `u` (⊥ оси), `v = axis ^ u`, `w` (∥ оси);
угол `theta = magdt / (rSqr + epsilon)` (вращение тем быстрее, чем
ближе к оси); `offset = (u·cos θ + v·sin θ + w) * r; pos = offset + center`.
Позиция — мгновенно, без интегрирования.

## Случайность

Все три используют `pDomain::Generate` (домены — `gen_acc`/`gen_vel`/
`gen_disp`):

- **PARandomAccel**: `vel += Generate(...) * dt` — случайное ускорение.
- **PARandomVelocity**: `vel = Generate(...)` — **замена** скорости
  (комментарий: «не умножать на dt, скорость инвариантна»).
- **PARandomDisplace**: `pos += Generate(...) * dt` — мгновенный
  случайный сдвиг позиции.

`Transform`: `gen_*.transform_dir` (dual-domain).

## Цели (target-лерпы)

Все четыре — экспоненциальное сближение с целевым значением:
`x += (target − x) * (scale * dt)`.

- **PATargetVelocity**: `vel += (velocity − vel) * scale * dt`.
  `Transform`: `velocity.transform_dir`.
- **PATargetSize**: `size += (size_target − size) * scale * dt` —
  `scale` — **вектор** (разная скорость по осям).
- **PATargetRotate**: только `rot.x`: `dif = (|rot.x| − |m.rot.x|) *
  sign(m.rot.x) * scale * dt` — вращение притягивается к величине `r`,
  знак сохраняется; `rot.y/rot.z` не трогает.
- **PATargetColor** — особый случай:
  `timeFrom/timeTo` — окно по `m.age * tm_max` (доли жизни); вне окна —
  `continue`. `c_n = lerp(color_p, (color, alpha), scale * STEP_DEFAULT)`,
  `STEP_DEFAULT = 0.033`; затем `colorC -= (colorC − c_n) / COEFF`,
  `COEFF = STEP_DEFAULT / dt` — нормировка на размер кадра (при
  `dt = 0.033` делитель = 1).

## Скорость

### PADamping — демпфирование в полосе скоростей

`scale = 1 − (1 − damping) * dt` покомпонентно; применяется только если
`vlowSqr <= |vel|² <= vhighSqr` — можно демпфировать только средние
скорости.

### PASpeedLimit — жёсткие границы

`|vel| < min_speed` → масштаб до `min_speed`; `|vel| > max_speed` →
масштаб до `max_speed`. (Отдельный `if/else if` — при `min > max`
низкая ветка выигрывает.)

## Источник

### PASource — рождение частиц

```
if (m_Flags.is(flSilent)) return;
rate = floor(particle_rate * dt);
if (drand48() < frac) rate++;              // dither дробной части
if (p_count + rate > max_particles) rate = max − p_count;   // cap
```

Для каждой новой частицы: `position.Generate(pos)`, `size.Generate(siz)`
(`flSingleSize` → `siz.set(x,x,x)` — изотропный размер),
`rot.Generate(rt)`, `velocity.Generate(vel) + parent_vel`,
`color.Generate(col)`, `age = age + NRand(age_sigma)` (гаусс по
`age_sigma`); `color_argb_f(alpha, col.x/y/z)`.
`flVertexB_tracks` → `posB = pos` (частица «следит» за стартовой точкой),
иначе `posB` не задаётся (остаётся мусором/0).
`effect->Add(pos, posB, siz, rt, vel, argb, age)`.

**`parent_motion` поле в файле не используется** (quirk IO).

`Transform`: `position.transform` + `velocity.transform_dir`
(остальные домены — world-only).

## Турбулентность

### PATurbulence

См. подробный разбор SSE-пути и `_EDITOR`-варианта в
[actions.md](actions.md#sse-патурбулентность). Ключевое:
`pV = pos·offset + age` (адвекция по времени), 4 вызова
`fractalsum3(pV, frequency, octaves)` (+ по осям для градиента),
`D = (∇noise − d) * magnitude`, `vel += D` с **нормализацией
`vel *= |vel_old|/|vel_new|`** — длина скорости сохраняется, меняется
только направление. `age += dt` — состояние внутри действия.
`static noise_start` → `noise3Init()` один раз. `Transform` пустой.

## Quirks и особенности (сводка)

1. **29 подклассов**, а не 30 — план ит.4 говорит «30» (расхождение
   зафиксировано в порц.2).
2. `PAKillOld` **мутирует `tm_max`** (`tm_max = age_limit`) — side
   effect, влияющий на соседние target-действия, читающие `tm_max`
   (`PATargetColor`).
3. `PARestore` **расходует `time_left`** каждый тик — действие
   «кончится само» и потом держит частицы на месте.
4. `PAExplosion`/`PATurbulence` **носят `age`** внутри Execute —
   состояние действия, не только `Particle.age`.
5. `PAScatter` — `acc.Generate` закомментирован (только радиальный
   отталкивание); `PACopyVertexB` — `copy_vel` не читается;
   `PASource` — `parent_motion` не используется.
6. `PAAvoid` PDRectangle: `gofs/fofs` — одинаковое выражение
   (copy-paste, 4-е «ребро» дублирует 3-е).
7. `PATargetColor` — окно `timeFrom/timeTo` — **доли жизни** частицы
   (`m.age * tm_max`), не секунды; `STEP_DEFAULT = 0.033` —
   нормировка на 30 FPS.
8. Знаменатель `√r² + (r²+ε)` (`PAOrbitPoint/Line`) — **сумма**, а не
   произведение (как в `PAGravitate`) — разная физика: ~1/r³ vs ~1/r².
9. `PAGravity` не использует `magnitude` — только `direction`.
10. `PABounce` PDSphere — нет ray-sphere: только `Within(pnext)` +
    `normalize(pos − p1)` (нормаль неточна для быстрых частиц).
11. `PAMatchVelocity` — `mj.vel -= acc` (в отличие от `PAFollow` без `-=`)
    — импульс «суммарного» движения пары сохраняется.
12. `PAVortex` меняет **`pos` напрямую** — единственный action,
    не работающий через `vel` (кроме sinks/kill).

См. также: [IO подклассов](actions-io.md), [каркас и паттерны](actions.md),
[ядро симуляции](core.md).
