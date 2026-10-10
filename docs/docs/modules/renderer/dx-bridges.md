# Renderer: UI-обвязка (dx-bridges)

UI-ядро `xrRender`: мосты от `xrEngine`-интерфейсов рендера к `RCache`/`HW` для UI, шрифтов, ImGui, консоли. Каждый класс — реализация `I*Render`-интерфейса (объявлены в `xrRender`-заголовках, итерация 2), создаётся фабрикой `dxRenderFactory` ([Точка входа](factory.md)).

> Статистика, окружение, флэры, молнии, debug-рендер, экран загрузки, GIF и 3DFluid — на отдельной странице [Прочее](misc.md).

## Ответственность

| Класс             | Интерфейс        | Задача                                                                                         |
| ----------------- | ---------------- | ---------------------------------------------------------------------------------------------- |
| `dxUIRender`      | `IUIRender`      | Примитивная отрисовка UI: geom-буферы (LIT/TL), push/flush точек, shaders, scissor, xform/cull |
| `dxUIShader`      | `IUIShader`      | Кэш UI-шейдеров `shader.create(sh, tex)` по ключу `tex_sh`                                     |
| `dxFontRender`    | `IFontRender`    | Батчинг строк `CGameFont` в LIT-геометрию (4 вершины/символ)                                   |
| `dxImGuiRender`   | `IImGuiRender`   | Тонкая обёртка над `imgui_impl_dx11/dx10/dx9`                                                  |
| `dxConsoleRender` | `IConsoleRender` | Отрисовка фона консоли (квад)                                                                  |

## Место в архитектуре

- Зависимости: `RCache` (vertex/index cache, [Устройство рендера](render-device.md)), `HW`/`StateManager` ([render-device.md](render-device.md)), `shader` (shader bus, [Shader Bus](shader-bus.md)), `CTexture`/`CGameFont` (xrEngine, итерация 2).
- Вызовчики: UI-классы `xrEngine`/`xrUI` через глобалы `::UIRender` и т.п. (устанавливаются `DllMainXrRenderR4`, [factory.md](factory.md)).
- Соседи-реализации тех же интерфейсов: `dxStatsRender`/`dxStatGraphRender` ([misc.md](misc.md)), `dxDebugRender` ([misc.md](misc.md)).

## Публичный API

### `dxUIRender` (глобал `UIRenderImpl`)

```cpp
void CreateUIGeom(); void DestroyUIGeom();   // hGeom_TL / hGeom_LIT через RCache.Vertex.Buffer()
void SetShader(IUIShader& shader);          // VERIFY(hShader); (dxUIShader*)&shader
void SetAlphaRef(BOOL b);                   // RCache.set_AlphaRef
void SetScissor(Frect* rect, BOOL bEnable); // RCache.set_Scissor + StateManager.OverrideScissoring
void PushPoint(float x, float y, float z, u32 color, float u, float v);
void StartPrimitive(int iMaxVerts, PrimitiveType, pointType);  // lock hGeom_LIT / hGeom_TL
void FlushPrimitive();                      // unlock, set_Geometry, RCache.Render
void CacheSetXformWorld(const Fmatrix& m);  // CULL_NONE + m
void CacheSetCullMode(CULLMODE m);          // CULL_NONE + m
Frect GetActiveTextureResolution();         // из RCache.get_ActiveTexture(0)
void UpdateShaderName(const shared_string& sh_name, shared_string& r_sh_name); // caps ≥ 2.0 + .ogm → "hud\\movie"
```

- `PushPoint` — `switch (m_PointType)`: `pttLIT` → `LIT_pv`, `pttTL` → `TL_pv`; в R4 активен только перегруз `(x, y, z, u32 C, u, v)`.
- `FlushPrimitive` — primCount зависит от `PrimitiveType`: `ptTriStrip` = p_cnt−2, `ptTriList` = p_cnt/3, `ptLineStrip` = p_cnt−1, `ptLineList` = p_cnt/2.
- Крупный **закомментированный блок** (старые `StartTriList`/`FlushTriList`/`PushPoint`-перегрузки) — D3D9-legacy, мёртв.

### `dxUIShader`

```cpp
ref_shader& GetCachedUIShader(shared_string& sh, CTexture* tex); // ключ: tex + "_" + sh
void create(shared_string& sh, CTexture* tex, BOOL no_cache = FALSE);
void Copy(IUIShader& _in);   // *this = *((dxUIShader*)&_in) — небезопасный каст (quirk)
```

- Глобал `g_UIShadersCache` (`xr_unordered_map<std::string, ref_shader>`); промах кэша → `shader.create(sh, tex)`.
- `no_cache=TRUE` → прямой `hShader.create` без кэша. `destroy()` закомментирован.
- Friends: `dxUIRender`, `dxDebugRender`, `dxWallMarkArray`, `CRender` — прямой доступ к `hShader`.

### `dxFontRender`

```cpp
void Initialize(ref_shader cShader, CTexture* cTexture);  // pShader.create + pGeom.create(FVF::F_TL, RCache.QuadIB)
void OnRender(CGameFont& owner);
```

`OnRender`: `VERIFY(g_bRendering)`; ленивая инициализация `fsValid` (разрешение `RCache.get_ActiveTexture(0)` → `vTS`, `fTCHeight`); строки батчатся (first-fit до `MAX_MB_CHARS` на один lock); пер-символ: `mbhMulti2Wide` для multi-byte, выравнивание center/right через `SizeOf_`, `fsGradient` → верхний цвет = половина RGB. DX10/11: X/Y/Y2 −= 0.5 («vertex shader will cancel a DX9 correction»); half-pixel UV-offset — **DX9-only** (`#if !defined(USE_DX10) && !defined(USE_DX11)`). 4 вершины/символ (2 треугольника), advance `vInterval.x`; multi-byte: `X -= 2` и `IsNeedSpaceCharacter` → `+ fXStep`. Отрисовка — `TRIANGLELIST`.

### `dxImGuiRender`

Тонкая обёртка над `imgui_impl_dx11`/`dx10`/`dx9`:

- `OnDeviceCreate(context)` — ставит аллокаторы `xr_malloc/xr_free`, `SetCurrentContext`, `ImGui_ImplDX11_Init(HW.pDevice, HW.pContext)`;
- `Frame()` = `ImGui_ImplDX11_NewFrame`; `Render(data)` = `ImGui_ImplDX11_RenderDrawData`;
- `SetState` — в DX10/11 ставит **только viewport** (остальной блок — DX9-era, закомментирован);
- жизненный цикл устройства: `InvalidateDeviceObjects`/`CreateDeviceObjects`.

### `dxConsoleRender`

```cpp
dxConsoleRender();        // DX10/11: m_Shader.create("hud\\crosshair") + m_Geom (F_TL, QuadIB)
void OnRender(BOOL bGame);
void Copy(IConsoleRender& _in);  // небезопасный каст (quirk)
```

`OnRender`: rect = всё окно, **`if (bGame) R.y2 /= 2`** (в игре консоль — нижняя половина). DX10/11: рисует 4-вершинный квад `D3DCOLOR_XRGB(32,32,32)` через `m_Shader->E[0]` — TODO-комментарий «Implement console background clearing for DX10»: вместо `Clear` рисуется тёмный квад. DX9: `HW.pDevice->Clear(...)`.

## Внутреннее устройство

- Геометрия UI — два `ref_geom` (`hGeom_TL`/`hGeom_LIT`), буферы через `RCache.Vertex.Buffer`; `DestroyUIGeom` дополнительно очищает `g_UIShadersCache`.
- `dxUIShader` — единственный владелец реального `ref_shader hShader` (создаётся через [shader bus](shader-bus.md)); все потребители (UI, debug, wallmarks, `CRender`) работают через friends.
- `dxFontRender` — единственный класс с реальным алгоритмом (батчинг символов); остальные — мосты.

## Взаимодействие

- `xrEngine` UI → `IUIRender`/`IFontRender` → `RCache` (vertex buffer lock/unlock, `Render`).
- `dxUIRender::SetShader` ← `dxUIShader` (кэш) ← [shader bus](shader-bus.md).
- `dxImGuiRender` → `HW.pDevice`/`HW.pContext` напрямую (без `RCache`).
- `dxConsoleRender` → `HW` (DX9 `Clear`) или `RCache` (DX10/11 квад).
- Общее состояние кадра (xform/cull/alpha-ref) — через `RCache`/`StateManager`, см. [render-device.md](render-device.md).

## Потоки данных

```
CGameFont::OutString → dxFontRender::OnRender → lock hGeom(F_TL) → 4 vert/char → RCache.Render(TRIANGLELIST)
UI-примитивы        → dxUIRender StartPrimitive/PushPoint/FlushPrimitive → RCache.Render
ImGui               → ImGui_ImplDX11_* (DrawData → GPU)
Консоль             → dxConsoleRender::OnRender → квад 32,32,32 (или HW Clear в DX9)
```

## Конфигурация

Прямого `xr_settings`-конфига нет. Шейдерные имена хардкодированы: `"hud\\crosshair"` (консоль), `"hud\\movie"` (видео в UI при caps ≥ 2.0 + `.ogm`, `UpdateShaderName`).

## Известные ограничения-дебаг

- `SetShader`/`Copy` (`dxUIShader`, `dxConsoleRender`, и т.п.) — **небезопасные касты** `*this = *((dxX*)&_in)`; работает, потому что `Copy` вызывается между объектами одного типа (установленный паттерн, см. [particles-wallmarks.md](particles-wallmarks.md)).
- `dxUIRender` — крупный мёртвый D3D9-блок (`StartTriList`/`FlushTriList`/старые `PushPoint`).
- `dxImGuiRender::SetState` — в DX11 только viewport; полный state-setup не перенесён.
- `dxConsoleRender` — фон в DX11 рисуется квадом, а не `Clear` (TODO).
