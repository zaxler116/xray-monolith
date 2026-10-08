# xrCore: Типы и математика

Базовые типы, векторы, матрицы, кватернионы, геометрические примитивы, FPU/CPU.

## Базовые типы — `_types.h`

| Тип | C-эквивалент | Примечание |
|---|---|---|
| `s8`, `u8` | `signed/unsigned char` | 8 бит |
| `s16`, `u16` | `signed/unsigned short` | 16 бит |
| `s32`, `u32` | `signed/unsigned int` | 32 бит |
| `s64`, `u64` | `signed/unsigned __int64` | 64 бит |
| `f32` | `float` | |
| `f64` | `double` | |
| `pstr` | `char*` | |
| `pcstr` | `const char*` | |
| `xr_empty` | `{}` | пустой маркер |

**Windows-совместимость**: если `_WINDOWS_` не определено, `BOOL`/`LPSTR`/`LPCSTR`/`TRUE`/`FALSE` определяются вручную.

**Пределы типов**: макросы `type_max(T)`, `type_min(T)`, `type_zero(T)`, `type_epsilon(T)` → `std::numeric_limits`. Алиасы: `int_max`, `flt_max`, `dbl_max`, и т.д.

**Строковые буферы**: `string16`..`string4096` — `char[N]`, `string_path` — `char[2*_MAX_PATH]`.

## Векторы — `_vector3d.h`, `_vector2.h`, `_vector4.h`

### `Fvector` (alias `_vector3<float>`)

```cpp
struct Fvector {
    float x, y, z;
    // operator[], set, add, sub, mul, div, ...
};
```

- **Row-column order**: `x`, `y`, `z` — компоненты.
- **Доступ**: `v.x`, `v.y`, `v.z`, или `v[0]`, `v[1]`, `v[2]`.
- **Операции**: `set`, `add`, `sub`, `mul`, `div`, `dot`, `cross`, `length`, `square_length`, `normalize`, `lerp`, `slerp`, `angle`, `reflect`, `project`, `intersect`, ...
- **Возврат `Self&`**: все мутации возвращают `*this` (chainable).

### `Fvector2`, `Fvector4`

Аналогично, 2 и 4 компоненты.

### `Ivector3`, `Ivector4`, `Ivector2`

Целочисленные версии (`s32`).

## Матрицы — `_matrix.h`, `_matrix33.h`

### `Fmatrix` (alias `_matrix<float>`)

**DirectX-compliant, row-column order**:
```
m11 m12 m13 m14  ← первая строка
m21 m22 m23 m24
m31 m32 m33 m34
m41 m42 m43 m44  ← трансляция (m41, m42, m43)
```

**Хранение**: `m[4][4]` или union:
- `_11.._44` — поэлементно
- `i, j, k, c` — строки как `Fvector` (R, N, D, T)
- `m[4][4]` — массив

**Множение**: `[x'y'z'1] = [xyz1][M]`, т.е. **вектор-строка × матрица**.

**Примечания**:
- Положительный угол = **по часовой** (DirectX convention).
- `mul(A, B)` = трансформация B, затем A.
- Последовательность поворота: **ZXY**.
- `I, J, K, C` = `R, N, D, T` (Right, Normal, Direction, Translation).

**Ключевые методы**: `set`, `add`, `sub`, `mul`, `inverse`, `transpose`, `det`, `rotation`, `translation`, `lerp`, `slerp`, `axis_angle`, `euler_angles`, `look_at`, ...

### `Fmatrix33` (alias `_matrix33<float>`)

3×3, только ротация (без трансляции).

## Кватернионы — `_quaternion.h`

### `Fquaternion` (alias `_quaternion<float>`)

**Формат**: `(s, v)` где `s = w` (скаляр), `v = [x, y, z]` (вектор).

```
q = w + xi + yj + zk
q = (s, v) = [s, (x, y, z)]
```

**Свойства**:
- `||q||` = `sqrt(w² + x² + y² + z²)`; unit quaternion: `||q|| == 1`.
- `q'` (конъюгат) = `(w, -v)`.
- `qinverse` = `q' / (q·q')`; для unit: `qinverse == q'`.
- **Не коммутативны**: `q1*q2 != q2*q1`.
- **Ассоциативны**: `(q1*q2)*q3 == q1*(q2*q3)`.

**Ротация**: угол `t` вокруг единичного вектора `u`:
```
s = cos(t/2)
v = u * sin(t/2)
```

**Применение к точке** `p = (px, py, pz)`, `P = (0, p)`:
```
p_rotated = q * P * qinverse   (= q * P * q' для unit)
```

**Сложение раций** `q1`, затем `q2`: `qc = q2 * q1`.

**Интерполяция**:
- Линейная (lerp) — не гарантирует unit.
- **Slerp** (сферическая): `q(t) = q1 * sin((1-t)w)/sin(w) + q2 * sin(t*w)/sin(w)`, где `w = arccos(q1·q2)`.

**Множение**:
```
q1 = (s1, v1), q2 = (s2, v2)
q1*q2 = (s1*s2 - dot(v1,v2),  s1*v2 + s2*v1 + cross(v1,v2))
```

## Геометрические примитивы

| Тип | Файл | Описание |
|---|---|---|
| `Fsphere` | `_sphere.h` | Сфера: центр + радиус |
| `Fbox` | `_fbox.h` | Ориентированный box: center + 3 оси + half-extents |
| `Fbox2` | `_fbox2.h` | Аксиальный box |
| `Fplane` | `_plane.h` | Плоскость: нормаль + смещение |
| `Fplane2` | `_plane2.h` | Плоскость (альтернативное представл.) |
| `Fcylinder` | `_cylinder.h` | Цилиндр |
| `Fobb` | `_obb.h` | Ориентированный bounding box |
| `Frect` | `_rect.h` | Прямоугольник (2D) |

## FPU — `_math.h`

Управление точностью FPU:
```cpp
namespace FPU {
    void m24();   // 24-битная точность (float)
    void m24r();  // restore
    void m53();   // 53-битная (double)
    void m53r();
    void m64();   // 64-битная (extended)
    void m64r();
}
```

## CPU — `_math.h`, `cpuid.h`

### `_processor_info`

Заполняется `CPUID`-инструкцией при старте:
- ВENDOR, Family, Model, Stepping
- Флаги: SSE, SSSE3, SSE2, SSE3, SSE4.1, SSE4.2, AVX, AVX2, ...
- `clk_per_second`, `clk_per_milisec`, `clk_per_microsec`
- `qpc_freq`, `qpc_overhead` (QueryPerformanceCounter)

### `CPU::QPC()`

Высоко-точный таймер (QueryPerformanceCounter). Возвращает `u64` (ticks).

### `CPU::GetCLK()`

RDTSC (Raw Time Stamp Counter). Возвращает `u64` (такты CPU).

**Инициализация**: `_initialize_cpu()` — вызывается в `xrCore::_initialize`. Заполняет `CPU::ID`, вычисляет `clk_per_second` и т.д.

## Треды — `_math.h`

```cpp
typedef void thread_t(void*);
void thread_name(const char* name);
void thread_spawn(thread_t* entry, const char* name, unsigned stack, void* arglist);
```

Простой обёртка над `_beginthreadex` (MSVC) / `pthread` (Borland).

## SIMD-диспетчеризация

Многие математические функции имеют **несколько версий** (SSE, SSSE3, AVX), выбираемых по `CPU::ID` в runtime. Механизм — `xrCPU_Pipe` (см. [Архитектура: Карта модулей](../../architecture/module-map.md)).

> В `xrCore` — только **интерфейсы** (`FPU`, `CPU::QPC`, `_processor_info`). Конкретные SIMD-версии — в `xrCPU_Pipe`.

## Ключевые константы

| Константа | Значение |
|---|---|
| `PI` | 3.14159265358979f |
| `PI_MUL_2` | 2 * PI |
| `PI_MUL_4` | 4 * PI |
| `EPS_S` | 1e-6f (epsilon) |
| `FLT_MAX`, `FLT_MIN` | пределы float |

## Связанные страницы

- [Память](memory.md) — как выделются векторы/матрицы.
- [Строки](strings.md) — `shared_str` для имён.
- [Логирование](logging.md) — `Msg` с Fvector-аргументами.
