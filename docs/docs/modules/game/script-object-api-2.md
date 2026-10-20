# Script: CScriptObject и прочие Lua-обёртки — actions / light / particles / sound / imgui / wallmarks (порцион 47)

> **Модуль:** `xrGame` · **Файлы:** `script_object.h` (29) + `script_object.cpp` (62), `script_lanim.h` (8) + `script_lanim.cpp` (53), `script_light.h` (3) + `script_light_inline.h` (265) + `script_light_script.cpp` (57), `script_particles.h` (67) + `script_particles.cpp` (202) + `script_particles_inline.h` (9) + `script_particles_script.cpp` (41), `script_sound.h` (55) + `script_sound.cpp` (118) + `script_sound_inline.h` (125) + `script_sound_script.cpp` (58), `script_world_property.h` (22) + `script_world_property_inline.h` (14) + `script_world_property_script.cpp` (27), `script_imgui.h` (7) + `script_imgui_inline.h` (442) + `script_imgui_script.cpp` (936), `script_wallmarks_manager.h` (36) + `script_wallmarks_manager.cpp` (110) + `script_wallmarks_script.cpp` (28), `script_action_wrapper.h` (29) + `.cpp` (68), `script_action_planner_wrapper.h` (22) + `.cpp` (40), `script_action_planner_action_wrapper.h` (33) + `.cpp` (66), `action_base_script.cpp` (48), `action_planner_script.cpp` (65), `action_planner_action_script.cpp` (34), `xrServerEntities/script_export_space.h` (32) · **Порцион 47, итерация 5**

> **Страница:** сборная солянка из оставшихся Lua-обёрток `xrGame`, не вошедших в порционы 43–46. **`CScriptObject`** — минимальный скриптовый объект (`CGameObject` + `CScriptEntity`), база для `CProjector`. **`lanim_wrapper`** (`color_animator`) — расчёт цвета по `CLAItem`. **`ScriptLight`/`ScriptGlow`/`AttachmentScriptLight`** (`script_light`) — полный API света/glow с lanim/volume-режимами. **`CScriptParticles`** (`particles_object`) — обёртка `CParticlesObject` с path-аниматором. **`CScriptSound`** (`sound_object`) + `CSound_params` — звук с BeforePlay-хуком. **`world_property`** — `CWorldProperty` для планнеров. **`ScriptImGui`** (`ImGui.*`) — огромный модуль биндинга Dear ImGui. **`ScriptWallmarksManager`** (`wallmarks_manager`) — спавн статических/скелетных декалей. **Action-семейство** (`action_base`/`action_planner`/`planner_action` + luabind `wrap_base`-обёртки) — мост C++↔Lua для планнеров GOAP (используется в итерации 6).

**Чего это не делает**: не документирует сами планнеры GOAP (`CActionPlanner` internals — итерация 6, [alife/](alife/index.md)); класс `CCustomZone`/`CScriptZone` — [zones-custom.md](zones-custom.md) / [zones-anomaly.md](zones-anomaly.md); `CProjector` (носитель `CScriptObject`) — [zones-subclasses-2.md](zones-subclasses-2.md); полный Lua API `CScriptGameObject` — [script-game-object.md](script-game-object.md); механика Lua-регистрации (`script_register`, `DECLARE_SCRIPT_REGISTER_FUNCTION`) — [scripting-boundary.md](../../architecture/scripting-boundary.md); внутренности `CParticlesObject`/`ref_sound`/`IRender_Light` — итерации 3/4/9.

## 1. Назначение и место

Все классы страницы — «листовые» Lua-обёртки, регистрируемые собственным `script_register` (макрос `DECLARE_SCRIPT_REGISTER_FUNCTION` → `static void script_register(lua_State*)`, `script_export_space.h:16–17`):

```
CScriptEntity (порц. 43)          CGameObject (порц. 3)
      │                                   │
      └────────────┬──────────────────────┘
                   ▼
            CScriptObject            ← база для CProjector (порц. 39)
                   ▲
              CProjector (searchlight)

Отдельные листовые обёртки (не CObject):
  color_animator · script_light · script_glow · attachment_script_light
  particles_object · sound_object · sound_params · world_property
  wallmarks_manager · ImGui (модуль) · action_base/action_planner/planner_action
```

| Обёртка | Lua-класс/модуль | Файл регистрации |
| --- | --- | --- |
| `CScriptObject` | (не экспортируется напрямую) | `_script.cpp` нет |
| `lanim_wrapper` | `color_animator` | `script_lanim.cpp:43–53` |
| `ScriptLight` | `script_light` | `script_light_script.cpp:11` |
| `ScriptGlow` | `script_glow` | `script_light_script.cpp:44` |
| `AttachmentScriptLight` | `attachment_script_light` | `script_light_script.cpp:35` |
| `CScriptParticles` | `particles_object` | `script_particles_script.cpp:19` |
| `CScriptSound` | `sound_object` | `script_sound_script.cpp:27` |
| `CSound_params` | `sound_params` | `script_sound_script.cpp:20` |
| `CScriptWorldProperty` | `world_property` | `script_world_property_script.cpp:20` |
| `ScriptWallmarksManager` | `ScriptWallmarksManager` + `wallmarks_manager()` | `script_wallmarks_script.cpp:16` |
| `ScriptImGui` | `ImGui` (модуль) | `script_imgui_script.cpp:9` |
| `CScriptActionBase` | `action_base` | `action_base_script.cpp:20` |
| `CScriptActionPlanner` | `action_planner` | `action_planner_script.cpp:37` |
| `CScriptActionPlannerAction` | `planner_action` | `action_planner_action_script.cpp:20` |

## 2. `CScriptObject` (`script_object.h`/`.cpp`)

`class CScriptObject : public CGameObject, public CScriptEntity` (h:14–17). Тонкий «множественный» объект: почти каждую виртуальную функцию прокидывает в **обе** базы по порядку — `CScriptEntity` (скриптовое управление) и `CGameObject` (движок).

| Метод | Где | Поведение |
| --- | --- | --- |
| ctor / dtor | cpp:12–18 | пустые |
| `_construct()` | cpp:20–25 | `CGameObject::_construct()` + `CScriptEntity::_construct()`, `return this` |
| `reinit()` | cpp:27–31 | `CScriptEntity::reinit()` **→** `CGameObject::reinit()` (порядок: entity первым) |
| `net_Spawn(DC)` | cpp:33–39 | `CGameObject::net_Spawn(DC) && CScriptEntity::net_Spawn(DC)` |
| `net_Destroy()` | cpp:41–45 | `CGameObject::net_Destroy()` + `CScriptEntity::net_Destroy()` |
| `UsedAI_Locations()` | cpp:47–50 | **`FALSE`** (объект не участвует в AI-сетке) |
| `shedule_Update(DT)` | cpp:52–56 | `CGameObject::shedule_Update(DT)` + `CScriptEntity::shedule_Update(DT)` |
| `UpdateCL()` | cpp:58–62 | `CGameObject::UpdateCL()` + `CScriptEntity::UpdateCL()` |
| `cast_script_entity()` | h:28 | `return this` |

Единственный наследник в репозитории — **`CProjector`** (`searchlight.h:22`, под `PROJECTOR_NEW`), подробнее — [zones-subclasses-2.md](zones-subclasses-2.md).

## 3. `lanim_wrapper` — Lua-класс `color_animator` (`script_lanim.h`/`.cpp`)

`lanim_registrator` (h:5–8, только `DECLARE_SCRIPT_REGISTER_FUNCTION`) — точка регистрации. Сама обёртка `lanim_wrapper` объявлена **внутри `.cpp`** (cpp:16–40) и не имеет заголовка:

| Член | Где | Поведение |
| --- | --- | --- |
| `CLAItem* item` | cpp:18 | указатель на `CLAItem` (`LightAnimLibrary`) |
| `lanim_wrapper(LPCSTR)` | cpp:20 | `load(name)` в ctor |
| `load(name)` | cpp:22–26 | `LALib.FindItem(name)`, `R_ASSERT3(item, "Can't find color anim:", name)` |
| `length()` | cpp:28–32 | `item->Length_ms()` |
| `calculate(float T)` | cpp:34–39 | `Fcolor().set(item->CalculateRGB(T, frame))` |

Регистрация `class_<lanim_wrapper>("color_animator")` ctor + `load`/`calculate`/`length` (cpp:47–51).

## 4. `script_light` — `ScriptLight`, `ScriptGlow`, `AttachmentScriptLight`

Весь API — **inline** в `script_light_inline.h`; `script_light.h` — только 3 строки-`#include` (h:1–3). Файл `_script.cpp` регистрирует три класса.

### 4.1 `ScriptGlow` (inline:7–81) — Lua `script_glow`

Глоу-эффект (`ref_glow m_glow`, inline:10). Поля: `m_light_anim` (CLAItem), `fRange`, `fBrightness`, `texture`, `color`. Ctor (18–28): `Render->glow_create()`, range 10, цвет (1,1,1,1), enable. `Enable/SetRange/SetPosition×2/SetDirection×2/SetTexture/SetColor/Update`. `SetColor`/`SetLanim`; `Update` (67–80) — при заданном lanim пересчитывает цвет из `CalculateBGR(Device.fTimeGlobal, frame) × fBrightness`.

### 4.2 `ScriptLight` (inline:83–227) — Lua `script_light`

Полноценный источник света (`ref_light m_light`). Поля (85–100): `bShadow`, `bHudMode`, `fBrightness`, `fRange`, `color`, `texture`, `iType` + volume: `bVolEnable`, `fVolDistance`, `fVolQuality`, `fVolIntensity`. Ctor (103–122): `Render->light_create()`, `POINT`, shadow off, range 10, цвет (1,1,1,1), volume false (distance/intensity/quality .5). Полный набор геттеров/сеттеров: `SetPosition/SetDirection(×3)/SetCone/Enable/SetHudMode/SetShadow/SetRange/SetType/SetTexture/SetVolumetric*/SetColor/SetLanim/SetBrightness`. Нюанс: **`SetVolumetric(true)` принудительно включает `SetShadow(true)`** (inline:178–184). `SetColor`/`SetLanim` управляют lanim; `Update` (211–224) — как у glow.

### 4.3 `AttachmentScriptLight` (inline:231–264) — Lua `attachment_script_light`

`: public ScriptLight` с собственным `Fmatrix m_offset` (ctor — `Fidentity`). Переопределяет `SetPosition/SetDirection`: пишет **в offset**, а не в свет (позиция/direction откладываются); `SetXFORM(Fmatrix&)` (259–264) — `mat.mulB_43(m_offset)` → `m_light->set_position(mat.c)` + `set_rotation(mat.k, mat.i)`. Используется `script_attachment` ([script-attachment.md §10](script-attachment.md), порц. 45).

### 4.4 Регистрация (`script_light_script.cpp`)

`module(L)` с тремя `class_<>`: `script_light` (ctor + 3 `set_direction`/2 `set_position`/`set_cone`/`update` + свойства `color/texture/enabled/type/range/shadow/lanim/lanim_brightness/volumetric*/hud_mode`), `attachment_script_light` (наследник `script_light`, только 2 `set_position`+3 `set_direction`), `script_glow` (ctor + 2 `set_position`/2 `set_direction` + `enabled/texture/range/color/lanim/lanim_brightness`).

## 5. `CScriptParticles` — Lua `particles_object` (`script_particles.*`)

Обёртка `CParticlesObject` с опциональным path-движением через `CObjectAnimator`.

### 5.1 `CScriptParticlesCustom : public CParticlesObject` (h:18–34)

Ctor `(owner, caParticlesName)` → `CParticlesObject(caParticlesName, FALSE, true)` (cpp:13–18). Держит обратный указатель `m_owner` и `m_animator`. Ключевое: **`PSI_internal_delete()`/`PSI_destroy()` (cpp:25–37) обнуляют `m_owner->m_particles`** при внешнем удалении движком — иначе висячий указатель. `shedule_Update` (39–51): обновляет `m_animator`, считает скорость `vel` из дельты позиции, `UpdateParent(xform, vel)`. Path-API: `LoadPath` (53–61, ленивое `CObjectAnimator`, перезагрузка только при смене имени), `StartPath`/`StopPath`/`PausePath`. `remove_owner()` (81–85) — отвязка перед удалением владельца.

### 5.2 `CScriptParticles` (h:36–65, cpp:87–202)

Поля: `Fmatrix m_transform`, публичный `CScriptParticlesCustom* m_particles`. Методы:

| Метод | Где | Поведение |
| --- | --- | --- |
| ctor / dtor | cpp:87–102 | `xr_new<CScriptParticlesCustom>(this, name)`; dtor — `remove_owner()` + `PSI_destroy()` |
| `Play(bHudMode=false)` | cpp:104–108 | делегат |
| `PlayAtPos(pos, bHud=false)` | cpp:110–118 | `m_transform.translate_over(pos)` → `UpdateParent` → `Play` → `UpdateParent` (двойной для корректной позиции) |
| `Stop()` / `StopDeffered()` | cpp:120–130 | `Stop(FALSE)` / `Stop(TRUE)` |
| `MoveTo(pos, vel)` | cpp:132–137 | `translate_over` + `UpdateParent(transform, vel)` |
| `XFORMMoveTo(pos)` | cpp:139–144 | `translate_over` + `SetXFORM` |
| `SetDirection(dir)` | cpp:146–155 | строит ортонормированный базис (`generate_orthonormal_basis_normalized`), сохраняет `m_transform.c` |
| `SetOrientation(yaw,pitch,roll)` | cpp:157–164 | `setHPB` + сохранение позиции + `SetXFORM` |
| `SetHudMode/IsPlaying/IsLooped` | cpp:166–181 | делегаты |
| `LoadPath/StartPath/StopPath/PausePath` | cpp:183–202 | делегаты в `m_particles` |
| `LastPosition()` | h:58 | `m_transform.c` |

`script_particles_inline.h` — **пустой** (только `#pragma once`, 9 строк). Регистрация `particles_object` (script.cpp:19–40): ctor + `play/play_at_pos/stop/stop_deffered/playing/looped/move_to/set_position/set_direction/set_orientation/set_hud_mode/last_position/load_path/start_path/stop_path/pause_path`.

## 6. `CScriptSound` — Lua `sound_object` + `sound_params` (`script_sound.*`)

Обёртка `ref_sound`. Поля: `mutable ref_sound m_sound`, `shared_str m_caSoundToPlay`; `friend class CScriptSoundAction` (h:21). Ctor (cpp:48–60): если `FS.exist("$game_sounds$", name, ".ogg")` — создать, иначе лог `"File not found \"%s\"!"` + `$no_sound.ogg`. Dtor (cpp:62–69): в DEBUG `THROW3(!m_sound._feedback(), "playing sound is not completed…")` до `destroy()`.

**BeforePlay-хук (новое в моде, cpp:16–46):** статические функции `ambient_hook_volume_mult(output)` и `script_sound_hook_volume_mult(file, pos, object)` вызывают Lua-функтор **`_G.COnBeforePlayScriptSound`**, возвращающий таблицу `{ volume_mult }` (`0` — вето, `<1` — затухание, `nil` — vanilla). `Play`/`PlayAtPos`/`PlayNoFeedback` (cpp:85–118): после `THROW3` считают `volume_mult`, при `<= EPS_S` — **выход без проигрывания**; иначе play/play_at_pos/play_no_feedback и при `<1` — `set_volume(volume_mult)` (в `PlayNoFeedback` — домножает `vol`).

Inline (`script_sound_inline.h`) — все геттеры/сеттеры и короткие `Play`/`PlayAtPos`-перегрузки: `Length` (11–15, `iFloor(get_length_sec()*1000)`), `Play(obj)`/`Play(obj,delay)` (17–25), `PlayAtPos` (27–35), `SetMinDistance`/`SetMaxDistance` (37–47, через `set_range`), `GetFrequency/GetMinDistance/GetMaxDistance/GetVolume` (49–71), `IsPlaying` (73–78, `!!_feedback()`, `VERIFY` закомментирован для `-nosound`), `AttachTail/Stop/StopDeffered/SetPosition/SetFrequency/SetVolume/GetParams/SetParams`. `GetPosition` — в `.cpp` (71–83, лог при отсутствии params).

Регистрация (script.cpp): `class_<CSound_params>("sound_params")` с rw-полями `position/volume/frequency/min_distance/max_distance`; `class_<CScriptSound>("sound_object")` со встроенным enum **`sound_play_type`** (`looped`=`sm_Looped`, `s2d`=`sm_2D`, `s3d`=0), свойства `frequency/min_distance/max_distance/volume/position`, 2 ctor (`LPCSTR` и `LPCSTR, ESoundTypes`), методы `get_position/set_position/play`×3/`play_at_pos`×3/`play_no_feedback/stop/stop_deffered/playing/length/attach_tail`.

## 7. `world_property` — `CScriptWorldProperty` (`script_world_property.*`)

**`typedef GraphEngineSpace::CWorldProperty CScriptWorldProperty`** (h:14) — обёртка без тела. `CScriptWorldPropertyWrapper` (h:16–20) — только регистратор. `_inline.h` — целиком закомментирован (14 строк). Регистрация `world_property` (script.cpp:20–26): ctor `(condition_type, value_type)` + `condition()`/`value()` + операторы `const_self < other<...>` и `==`. Используется планнерами GOAP (итерация 6).

## 8. `ScriptWallmarksManager` — Lua `wallmarks_manager` (`script_wallmarks*`)

| Сущность | Где | Поведение |
| --- | --- | --- |
| `ScriptWallmarksArray` | h:5–11, cpp:8–22 | `FactoryPtr<IWallMarkArray>*` + `m_section`; ctor ищет ltx-секцию, читает ключ `wallmarks` (список имён через пробел) и `AppendMark` каждого |
| `m_script_wallmarks` | h:16 | `xr_vector<ScriptWallmarksArray*>` (кэш секций) |
| `~ScriptWallmarksManager` (cpp:26–33) | — | `xr_delete` массивов и их `m_wallmarks` |
| `FindSection(section)` (cpp:35–46) | — | линейный поиск по `m_section`, иначе создаёт и кэширует |
| `PlaceWallmark(…, ttl)` (cpp:48–53) | — | делегат в `(…, bool=true)` |
| `PlaceWallmark(…, bool random_rotation)` (cpp:56–61) | — | → `(…, float rotation)`; `random_rotation` → `Random.randF(-20,20)` |
| `PlaceWallmark(…, float rotation)` (cpp:63–95) | — | `ObjectSpace.RayPick` (`rqtBoth`, игнор `ignore_obj`); при попадании в **статическую** геометрию (`!result.O`) и материале **без `flSuppressWallmarks`** — `Render->add_StaticWallmark(FindSection, end_point, size, pTri, pVerts, ttl, true, rotation)` |
| `PlaceSkeletonWallmark(obj, …)` (cpp:97–109) | — | проверка `obj`, `Render->add_SkeletonWallmark(&xform, Kinematics, FindSection, start, dir, size, ttl, true)` |

Регистрация (script.cpp): `ScriptWallmarksManager` (ctor + 3 перегрузки `place` с bool/float rotation + `place_skeleton`) и **глобальная `wallmarks_manager()`** (cpp:6–9) → `&g_pGamePersistent->GetWallmarksManager()`. Класс обёрнут `typedef class_exporter<ScriptWallmarksManager> CScriptWallmarksManager` (h:36).

## 9. `ScriptImGui` — Lua-модуль `ImGui` (`script_imgui.*`)

Гигантский биндинг Dear ImGui. `script_imgui.h` — 7 строк, только `class ScriptImGui { DECLARE_SCRIPT_REGISTER_FUNCTION }`. `script_imgui_inline.h` (442) — ~150 `IC`-хелперов-адаптеров между X-Ray типами (`Fvector`/`Fvector2`/`Fcolor`) и ImGui (`ImVec2`/`ImVec4`), плюс два файловых статика: `static bool ugly_hack` и `static ::std::string imgui_text_buffer` (5–6) — буфер для `InputText*`. Регистрация `script_imgui_script.cpp` через **`module(L, "ImGui")`** (отдельное пространство имён Lua):

- **Функции** (def ~13–307): окна (`Begin`/`End`, `SetNextWindowPos/Size/…`, `GetWindow*`), скролл, стили (`PushStyleColor/Var`, `PopStyle*`, `GetStyleColor`), курсор, текст (`Text`= `TextUnformatted`, `TextColored/Disabled/Wrapped`, `LabelText`, `BulletText`), кнопки (`Button`/`SmallButton`/`InvisibleButton`/`ArrowButton`/`Checkbox`/`RadioButton`/`ProgressBar`), комбо, drag/slider/input (`DragFloat*`/`SliderFloat*`/`InputText*`/`InputFloat*`/`Color*`), деревья/`CollapsingHeader`, list-box, меню/`MenuItem`, тултипы, попапы, **таблицы** (`BeginTable`/`TableSetupColumn`/…), tab-bar, disabled/clip, item-запросы (`IsItem*`), клавиатура/мышь (`IsKey*`/`IsMouse*`/`Shortcut`), буфер обмена, шрифты (`PushFont`/`PopFont`).
- **Luabind-специфика:** `out_value<N>()` для выходных ссылок (`Begin` 15, `Checkbox` 134, `DragFloat*`, `SliderFloat*`, `InputText*` 153–155, `Color*`, `MenuItem` 188, попапы, `BeginTabItem` 232 и т.д.).
- **Enum-модули** (312–935): ~40 `class_<enum_exporter<…>>` — `ImGuiCond`, `ImGuiDir`, `ImGuiWindowFlags`, `ImGuiItemFlags`, `ImGuiSelectableFlags`, `ImGuiInputTextFlags`, `ImGuiTreeNodeFlags`, `ImGuiPopupFlags`, `ImGuiTable*Flags/BgTarget/RowFlags`, `ImGuiTabBar/ItemFlags`, `ImGuiHoveredFlags`, `ImGuiCol`, `ImGuiStyleVar`, `ImGuiComboFlags`, `ImGuiSortDirection`, `ImGuiMouseButton`, `ImGuiButtonFlags`, `ImGuiKey` (полный список клавиш), `ImGuiInputFlags`, `ImGuiColorEditFlags`, `ImGuiSliderFlags`, `ImGuiFocusedFlags`.

## 10. Action-семейство: `action_base` / `action_planner` / `planner_action`

Мост C++↔Lua для GOAP-планнеров: три C++-базы + три `luabind::wrap_base`-обёртки. Внутренности планнеров — итерация 6.

### 10.1 Обёртки (`luabind::wrap_base`)

Все три реализуют один паттерн: виртуальный метод → `::luabind::call_member<T>(this, "имя", …)` (диспетчер в Lua-переопределение), статический `*_static` → прямой вызов базовой C++-реализации (для «Lua не переопределила»).

| Класс | h | cpp | Виртуалы |
| --- | --- | --- | --- |
| `CScriptActionWrapper : CScriptActionBase, wrap_base` | 29 | 68 | `setup`/`initialize`/`execute`/`finalize` (+ `_static`); `weight`/`weight_static` **закомментированы** (h:25–26, cpp:55–68) |
| `CScriptActionPlannerWrapper : CScriptActionPlanner, wrap_base` | 22 | 40 | `setup`/`update` (+ `_static`); под `#ifdef LOG_ACTION` синхронизирует `set_use_log` с `psAI_Flags.test(aiGOAPScript)` (cpp:16–18, 29–32) |
| `CScriptActionPlannerActionWrapper : CScriptActionPlannerAction, wrap_base` | 33 | 66 | `setup`/`initialize`/`execute`/`finalize`/`weight` (+ `_static`); `typedef … inherited` h:17 |

### 10.2 Регистрация

- **`action_base_script.cpp`** (48): `class_<CScriptActionBase, CScriptActionWrapper>("action_base")` — `def_readonly("object"/"storage")`, 3 ctor, `add_precondition`/`add_effect` (→ `add_condition`/`add_effect`), `remove_precondition`/`remove_effect`, `setup`/`initialize`/`execute`/`finalize` (с `_static`), `set_weight`; `weight` и `show` (под `LOG_ACTION`) закомментированы/условны.
- **`action_planner_script.cpp`** (65): свободные `set_goal_world_state(planner, state)` → `set_target_state`, `get_actual(planner)` → `actual()`, `cast_planner(action)` → `smart_cast<CScriptActionPlanner*>`; `class_<CScriptActionPlanner, CScriptActionPlannerWrapper>("action_planner")` — `object`/`storage`, `actual`, `setup`/`update` (+`_static`), `add_action`(`adopt<3>`)/`remove_action`, `action`, `add_evaluator`(`adopt<3>`)/`remove_evaluator`, `evaluator`, `current_action_id`/`current_action`, `initialized`, `set_goal_world_state`, `clear`, `show` (LOG_ACTION); **`def("cast_planner", …)`** рядом.
- **`action_planner_action_script.cpp`** (34): `class_<CScriptActionPlannerAction, bases<CScriptActionPlanner, CScriptActionBase>, CScriptActionPlannerActionWrapper>("planner_action")` — 3 ctor, `setup`/`initialize`/`execute`/`finalize` (+`_static`), `show` (условно), `weight` (+`_static`).

## 11. Связи

```mermaid
graph TD
    subgraph Lua
        CA["color_animator"]
        SL["script_light / script_glow / attachment_script_light"]
        PO["particles_object"]
        SO["sound_object / sound_params"]
        WP["world_property"]
        WM["wallmarks_manager()"]
        IM["ImGui.* (модуль)"]
        AB["action_base"]
        AP["action_planner"]
        PA["planner_action"]
    end
    subgraph C++
        LAL["CLAItem (LALib)"]
        RL["IRender_Light / ref_glow"]
        CPO["CParticlesObject + CObjectAnimator"]
        RS["ref_sound + CSound_params"]
        CWP["GraphEngineSpace::CWorldProperty"]
        SWM["ScriptWallmarksManager"]
        IMG["ImGui (Dear ImGui)"]
        ASB["CScriptActionBase/Wrapper"]
        ASP["CScriptActionPlanner/Wrapper"]
        APA["CScriptActionPlannerAction/Wrapper"]
    end
    CA -->|FindItem/CalculateRGB| LAL
    SL -->|light_create/glow_create| RL
    PO -->|wrapper + path| CPO
    SO -->|_G.COnBeforePlayScriptSound| RS
    WP --> CWP
    WM -->|RamPick/add_*Wallmark| SWM
    IM --> IMG
    AB --> ASB
    AP --> ASP
    PA --> APA
    ASB -. luabind wrap_base .-> CA["Lua-класс"]
    SO -. порц. 43 .-> SEA["CScriptSoundAction"]
```

**Связи с другими страницами:** `CScriptObject`/`CProjector` — [zones-subclasses-2.md](zones-subclasses-2.md); `AttachmentScriptLight` в `script_attachment` — [script-attachment.md §10](script-attachment.md) (порц. 45); `CScriptSoundAction` (`sound_object` внутри action-пакета) — [script-entity.md §21](script-entity.md) (порц. 44); Lua-механика регистрации — [scripting-boundary.md](../../architecture/scripting-boundary.md); планнеры GOAP — [alife/](alife/index.md) (итерация 6); `CParticlesObject`/`ref_sound`/`IRender_Light` — итерации 3/4/9; `world_property` в `CWorldProperty`-условиях — итерация 6.

## 12. Замечания к плану

1. **`script_object.h` в плане — 149 строк; фактически 29.** Остаток плана опирается на неверное число; страница использует фактические размеры (§2).
2. **`script_light` — «только `.h`/`_inline.h`/`_script.cpp`»**: план назвал `.h` (подразумевая тело класса), но фактически весь класс `ScriptLight`/`ScriptGlow`/`AttachmentScriptLight` живёт в **`script_light_inline.h` (265)**, а `script_light.h` — 3 строки `#include`.
3. **В плане не названы `ScriptGlow` и `AttachmentScriptLight`** — оба полноценно экспортируются (`script_glow`, `attachment_script_light`).
4. **В плане `script_particles_inline.h`** упомянут как рабочий; фактически **пуст** (9 строк, `#pragma once`).
5. **BeforePlay-хук звука (`_G.COnBeforePlayScriptSound`)** — мод-расширение: план его не называет; `Play`/`PlayAtPos`/`PlayNoFeedback` получили volume-gate и вето (cpp:16–46, 89–117).
6. **`script_wallmarks_script.cpp`** (план писал `script_wallmarks_script.cpp` — совпадает; `_manager.cpp` начинается с `#pragma once` — бесполезно, как и у порц. 45).
7. **`script_world_property_inline.h` целиком закомментирован** (ctor `CScriptWorldProperty` → `inherited`), `_inline.h` не несёт кода.
8. **`CScriptWorldProperty` — `typedef`, а не класс** (`GraphEngineSpace::CWorldProperty`, h:14); план описывает его как обёртку.
9. **ImGui-модуль огромен и не покрыт по-элементно планом**: 936 строк регистрации, ~40 enum-классов; в `ImGuiTabBarFlags`/части `ImGuiTableColumnFlags` **значения-обёртки скопированы неверно** (например, `ImGuiTabBarFlags_Reorderable` экспортируется под именем `"Disabled"`, cpp:531–539) — вероятная копипаста; документируем как есть.
10. **`script_action_wrapper.h` включает `script_action_wrapper_inline.h`** (h:29), но такого файла **в репозитории нет** — `#include` «висячий» (сборка зависит от порядка), сам класс объявлен целиком в `.h`.
11. **`CScriptActionWrapper::weight`** закомментирован (h:25–26, cpp:55–68), а у `CScriptActionPlannerActionWrapper` — **активен** (cpp:54–66): асимметрия весов между семействами.
12. **`_script.cpp` в action-семействе нет** — регистрация по имени файла `action_*_script.cpp` (три штуки), план их не перечислял отдельно.
13. Размеры всех файлов сверены с фактическими (см. шапку); расхождения — в пп. 1–2.

## 13. Краткое резюме

Порцион 47 собирает «хвостовые» Lua-обёртки `xrGame`. **`CScriptObject`** — простейший скриптовый объект (`CGameObject`+`CScriptEntity`, `usedAI=false`), база `CProjector`. **`color_animator`** считает цвет из `CLAItem`. **`script_light`/`script_glow`/`attachment_script_light`** — полный API света (включая volume-режим, принудительно включающий shadow) и glow, с lanim-анимацией цвета; attachment-вариант откладывает transform в `m_offset` и применяет `SetXFORM`. **`particles_object`** — обёртка `CParticlesObject` с `CObjectAnimator`-путём и корректной отвязкой владельца при внешнем удалении. **`sound_object`** (+`sound_params`) — `ref_sound` с vanilla-проверкой существования `.ogg` и мод-хуком `_G.COnBeforePlayScriptSound` (volume-gate/вето). **`world_property`** — `typedef CWorldProperty` для планнеров. **`wallmarks_manager`** — кэш ltx-секций и `add_Static/SkeletonWallmark` с проверкой `flSuppressWallmarks`. **`ImGui`** — отдельный Lua-модуль на ~150 адаптерах + ~40 enum'ов. **Action-семейство** (`action_base`/`action_planner`/`planner_action`) — luabind-`wrap_base`-мост для GOAP-планнеров (виртуал → `call_member` в Lua, `*_static` → база); используется итерацией 6.

*Порцион 47 (итерация 5) · далее: порц. 48 — связность (object-model.md, dependencies.md, index.md, mkdocs.yml nav, GENERATE-DOCS.md §9), затем удаление ITERATION-5-PLAN.md.*
