# IRenderable

`IRenderable` — интерфейс, который объект должен реализовать, чтобы быть видимым для рендера.

## Ответственность

- Хранить рендер-состояние объекта: `xform` (матрица), `visual` (указатель на `IRenderVisual`), `pROS` (object-specific), `pROS_Allowed`.
- Предоставить `renderable_Render()` — чистый виртуальный, вызывается рендерером.
- Опционально: `renderable_ShadowGenerate()` / `renderable_ShadowReceive()` (по умолчанию `FALSE`).
- Опционально: `GetHotness()` / `GetTransparency()` / `GetGlowing()` (HeatVision / SilencerOverheat).

## Место в архитектуре

```mermaid
graph TD
    subgraph xrEngine
        IR[IRenderable]
        Obj[CGameObject]
        PSI[CPS_Instance]
        Render[IRender_interface]
    end
    subgraph xrRender
        RDev[RenderDeviceRender]
        RVis[IRenderVisual]
    end

    Obj --implements--> IR
    PSI --implements--> IR
    IR -->|renderable.visual| RVis
    IR -->|renderable_ROS| Render
    RDev -->|set_Object| IR
    RDev -->|add_Geometry| RVis
```

`IRenderable` — **мixin**, не singleton. Любой объект (игровой, частица, декорация) может реализовывать его. `CGameObject` реализует через `Visual()` → `IRenderVisual`. `CPS_Instance` реализует напрямую.

## Публичный API

```cpp
// IRenderable.h
class ENGINE_API IRenderable
{
public:
    struct
    {
        Fmatrix xform;
        IRenderVisual* visual;
        IRender_ObjectSpecific* pROS;
        BOOL pROS_Allowed;
    } renderable;

public:
    IRenderable();
    virtual ~IRenderable();
    IRender_ObjectSpecific* renderable_ROS();
    virtual void renderable_Render() = 0;
    virtual BOOL renderable_ShadowGenerate() { return FALSE; }
    virtual BOOL renderable_ShadowReceive() { return FALSE; }
    virtual float GetHotness() { return 0.0; }
    virtual float GetTransparency() { return 0.0; }
    virtual float GetGlowing() { return 0.0; }
};
```

## Внутреннее устройство

### Конструктор

```cpp
IRenderable::IRenderable()
{
    renderable.xform.identity();
    renderable.visual = NULL;
    renderable.pROS = NULL;
    renderable.pROS_Allowed = TRUE;

    ISpatial* self = fast_dynamic_cast<ISpatial*>(this);
    if (self) self->spatial.type |= STYPE_RENDERABLE;
}
```

- Инициализирует `xform` как identity.
- `visual = NULL` (ещё не назначен).
- `pROS_Allowed = TRUE` (ROS разрешён по умолчанию).
- Если объект является `ISpatial` — ставит флаг `STYPE_RENDERABLE` (для пространственного поиска).

### Деструктор

```cpp
IRenderable::~IRenderable()
{
    VERIFY(!g_bRendering);
    Render->model_Delete(renderable.visual);
    if (renderable.pROS) Render->ros_destroy(renderable.pROS);
    renderable.visual = NULL;
    renderable.pROS = NULL;
}
```

- **`VERIFY(!g_bRendering)`** — нельзя уничтожать рендеруемый объект во время рендера.
- Удаляет `visual` через `Render->model_Delete`.
- Удаляет `pROS` через `Render->ros_destroy`.

### `renderable_ROS()`

```cpp
IRender_ObjectSpecific* IRenderable::renderable_ROS()
{
    if (0 == renderable.pROS && renderable.pROS_Allowed)
        renderable.pROS = Render->ros_create(this);
    return renderable.pROS;
}
```

Ленивое создание `pROS` (object-specific data) через `Render->ros_create(this)`. Если `pROS_Allowed == FALSE` (например, `CPS_Instance`) — возвращает `NULL`.

## Взаимодействие

```mermaid
graph TD
    subgraph xrEngine
        IR[IRenderable]
        Obj[CGameObject]
        PSI[CPS_Instance]
    end
    subgraph xrRender
        RDev[RenderDeviceRender]
        RVis[IRenderVisual]
        ROS[IRender_ObjectSpecific]
    end

    Obj --implements--> IR
    PSI --implements--> IR
    IR -->|renderable.visual| RVis
    IR -->|renderable_ROS| ROS
    RDev -->|set_Object| IR
    RDev -->|add_Geometry| RVis
    RDev -->|ros_create| ROS
    RDev -->|model_Delete| RVis
    RDev -->|ros_destroy| ROS
```

- **Кто вызывает `renderable_Render()`**: `RenderDeviceRender` (xrRender) при рендере сцены.
- **Кто устанавливает `renderable.visual`**: `CGameObject::Visual()` (xrEngine) или `CPS_Instance` (частично).
- **Кто создаёт/удаляет `pROS`**: `renderable_ROS()` (ленивое), деструктор `~IRenderable()` (удаление).
- **`g_bRendering`** — глобальный флаг (`device.cpp`), `TRUE` во время `Device.Begin()` → `Device.End()`.

## Потоки данных

### Назначение visual

```mermaid
sequenceDiagram
    participant Obj as CGameObject
    participant IR as IRenderable
    participant Render as IRender_interface
    participant RDev as RenderDeviceRender

    Obj->>IR: renderable.visual = Visual()
    Obj->>IR: renderable.xform = XFORM()
    Render->>RDev: set_Object(IR)
    Render->>RDev: add_Geometry(renderable.visual)
```

### ROS lifecycle

```mermaid
sequenceDiagram
    participant Obj as IRenderable
    participant Render as IRender_interface
    participant RDev as RenderDeviceRender

    Note over Obj: renderable.pROS == NULL, pROS_Allowed == TRUE
    Obj->>Render: renderable_ROS()
    Render->>RDev: ros_create(this)
    RDev-->>Render: IRender_ObjectSpecific*
    Render-->>Obj: renderable.pROS

    Note over Obj: ... (рендер, lighting) ...

    Obj->>Render: ~IRenderable()
    Render->>RDev: ros_destroy(renderable.pROS)
    RDev-->>Render: (удалено)
```

## Конфигурация

- `renderable.pROS_Allowed` — флаг, разрешающий создание `pROS`. По умолчанию `TRUE`. `CPS_Instance` ставит `FALSE` (частицы не используют ROS).
- `STYPE_RENDERABLE` — битовый флаг `ISpatial.type`, ставится в конструкторе `IRenderable` (если объект `ISpatial`).

## Ограничения / дебаг

- **`VERIFY(!g_bRendering)`** в деструкторе: уничтожение `IRenderable` во время рендера → crash в DEBUG.
- **`renderable.visual`** — **сырой указатель** на `IRenderVisual`, управляемый `Render->model_Delete`. Нельзя вручную `delete`.
- **`renderable.xform`** — копируется из `CGameObject::XFORM()` каждый кадр (см. [xr_object](xr-object.md)).
- **`renderable_ShadowGenerate/Receive`** — по умолчанию `FALSE`. Переопределяется в `CGameObject` (см. [xr_object](xr-object.md)).
- **`GetHotness` / `GetTransparency` / `GetGlowing`** — HeatVision / SilencerOverheat, по умолчанию `0.0`. Переопределяется в конкретных объектах.
