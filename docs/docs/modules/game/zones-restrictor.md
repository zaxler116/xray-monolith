# Зоны: CSpaceRestrictor + space_restriction — shape/holder/bridge (порционы 35–36)

> **Модуль:** `xrGame` · **Файлы:** `space_restrictor.h/.cpp` (75+332), `space_restriction.h/.cpp` (96+377, +`space_restriction_inline.h` 88), `space_restriction_shape.h/.cpp` (47+237), `space_restriction_holder.h/.cpp` (62+224), `space_restriction_bridge.h/.cpp` (47+155), `space_restriction_abstract.h` (45, +inline 82), `space_restriction_composition.h/.cpp` (54+236, +inline 36), `space_restriction_manager.h/.cpp` (85+318, +inline 24) · **Порционы 35–36, итерация 5**
>
> **Страница:** устройство пространственных ристрикторов (зон-объектов) и слоя ограничений поверх level graph: геометрия `CSpaceRestrictor`, границы на графе `CSpaceRestrictionShape`, реестр `CSpaceRestrictionHolder`, мост `CSpaceRestrictionBridge`, out/in-пара `CSpaceRestriction` (35); абстрактная база, композиции, глобальный `CSpaceRestrictionManager` и его клиенты (36). Связи: `CSE_ALifeSpaceRestrictor`/`CSE_ALifeAnomalousZone` (ит. 5, порц. 40), `CCF_Shape` (xrEngine), `CCustomZone` (порц. 40), `CRestrictedObject`/`CMovementManager::on_restrictions_change` (порц. 14–16), `ai().restrictions()` (порц. 9), Feel/`cast_restrictor` (порц. 9).

## 1. Архитектура

```
CGameObject
 └── CSpaceRestrictor            — геометрия зоны (сфера/боксы), регистрация в менеджере

CSpaceRestrictionBase            — абстракция «множество вершин level graph» (border, inside, процесс бордера)
 ├── CSpaceRestrictionShape      — реализация поверх CSpaceRestrictor (одна зона)
 └── CSpaceRestrictionComposition— конкатенация нескольких ристрикторов (порц. 36)

CSpaceRestrictionBridge (CTimeIntrusiveBase)
 └── владеет CSpaceRestrictionBase* (m_object), делегирует все вызовы; смена реализации — change_implementation

CSpaceRestriction (CTimeIntrusiveBase, CSpaceRestrictionAbstract)
 — пара out_restrictions/in_restrictions для конкретного объекта (кто чем ограничен)

CSpaceRestrictionHolder          — реестр мостов по строке имён (xr_map)
 └── CSpaceRestrictionManager (порц. 36) — глобальный, Level() наследует
```

## 2. `CSpaceRestrictor` (`space_restrictor.h`, 75 строк)

- Заголовок (21–43): `PLANE_COUNT = 6`, `CPlanes` (массив из 6 `Fplane`), `SPHERES = xr_vector<Fsphere>`, `BOXES = xr_vector<CPlanes>`; **mutable**-кэш: `m_spheres`, `m_boxes`, `m_selfbounds`, `m_actuality`.
- Поля: `u8 m_space_restrictor_type` (public, 46), `cast_restrictor()` → `this` (63), **`register_schedule() = false`** (64 — объект вне расписания), `DECLARE_SCRIPT_REGISTER_FUNCTION`.
- **`net_Spawn`** (`space_restrictor.cpp:42–97`):
  1. `actual(false)`; приведение к `CSE_ALifeSpaceRestrictor` (R_ASSERT), `m_space_restrictor_type` из серверного объекта;
  2. `collidable.model = xr_new<CCF_Shape>(this)`; по `se_shape->shapes`: type 0 → `add_sphere`, type 1 → `add_box`; `ComputeBounds()`;
  3. **Alundaio** (80–84): если объект — не `CCustomZone`, либо `CRadioactiveZone`, либо `CZoneCampfire` → `spatial.type &= ~STYPE_VISIBLEFORAI` (глобал `g_ai_die_in_anomaly` объявлен на 25, но в условии не используется — закомментирован);
  4. `setEnabled(FALSE); setVisible(FALSE);` — ристриктор невидим/выключен как объект;
  5. без level graph или тип `eRestrictorTypeNone` → выход без регистрации; иначе **`Level().space_restriction_manager().register_restrictor(this, type)`** (93–94).
- **`net_Destroy`** (99–110): `inherited::net_Destroy()` → при наличии graph и типа ≠ None — `unregister_restrictor` (без проверки уровня/level graph).

## 3. Геометрия `CSpaceRestrictor` (`space_restrictor.cpp`)

| Метод | Строки | Логика |
|---|---|---|
| `Center(Fvector&)` / `Radius()` | 32–40 | `XFORM().transform_tiny(C, CFORM()->getSphere().P)` / `CFORM()->getRadius()` |
| `inside(const Fsphere&)` | 112–121 | `!actual()` → `prepare()`; быстрый отказ `!m_selfbounds.intersect(sphere)` → false; иначе `prepared_inside` |
| `prepare() const` | 134–205 | `m_selfbounds` = Center+Radius; для каждой формы `CCF_Shape`: **сфера** (type 0) → мировая `XFORM().transform_tiny`; **бокс** (type 1) → `sphere.mul_43(XFORM(), box)`, 8 углов (полубокс ±0.5) → 6 плоскостей `m_planes[i].build(...)`; `actual(true)` |
| `prepared_inside(sphere)` | 207–233 | сферы: `sphere.intersect(*I)` → true; боксы: `m_planes[i].classify(sphere.P) > sphere.R` для любой плоскости → следующий бокс (`goto continue_loop`), все 6 → true |
| `UsedAI_Locations()` | 123–126 | всегда `FALSE` (не использует точки AI-позиций) |
| `spatial_move()` | 128–132 | `inherited::spatial_move(); actual(false)` — инвалидация кэша при сдвиге |

**DEBUG `OnRender`** (242–331): при `bDebug` и флаге `dbg_net_Draw_Flags.dbg_draw_customzone` — `DRender->OnFrameEnd()`, эллипсы сфер/ОBB боксов (cyan — включённая `CCustomZone` (`IsEnabled()`), red — иначе); при близости камеры `<100` — 2D-надпись `Name()` + состояние `ZoneState()` (IDLE/AWAKING/BLOWOUT/ACCUMULATE/DISABLED).

## 4. `CSpaceRestriction` (`space_restriction.h/.cpp`, 96+377 + inline 88)

Наследует `RestrictionSpace::CTimeIntrusiveBase` + `CSpaceRestrictionAbstract`. Члены: `m_applied`, `m_out_restrictions`/`m_in_restrictions` (строки имён через запятую), `m_out_space_restriction`/`m_in_space_restriction` (`CBaseRestrictionPtr` = `intrusive_ptr<CSpaceRestrictionBridge>`), менеджер, `m_temp`. Ветка `USE_FREE_IN_RESTRICTIONS` (`CFreeInRestriction`, `merge_free_in_retrictions` 214–247) — **закомментирована/выключена** (макрос не определён, `space_restriction.h:16`).

- **`initialize()`** (249–294): `manager->restriction(out/in)`; рекурсивно инициализирует мосты; при out-мосте → `merge_in_out_restrictions()`; DEBUG: проверка `object().correct()` (`~ BAD out restrictions combination`), при отсутствии out — `m_border = in->border()`; `m_initialized = true`.
- **`merge_in_out_restrictions()`** (140–188): `m_border = out->border()` минус вершины, «внутри» in (`CMergeInOutPredicate<true>`); плюс in-бордер минус вершины внутри out (`<false>`); `sort + unique`.
- **`accessible(Fsphere)`** (56–87): lazy-init (нет инициализации → `true`); далее `valid_vertex_position` && (out: `inside && !on_border && !out_of_border`) && (in: `!inside && !on_border`).
- **`accessible(vertex, radius)`** (89–107): out `inside(id,false,radius)` && !in `inside(id,true,radius)`.
- **`inside(Fsphere)`** = `accessible` (inline 72–75); **`inside(vertex, partially)`** (inline 77–88): out `inside(partially)` && in `!inside(!partially)`.
- **`intersects(bridge0, bridge1)`** (109–130): по каждой вершине бордера bridge1 → `bridge0->inside(*I, true)` → true; затем первый элемент бордера bridge0 внутри bridge1; затем `set_intersection` бордеров. **`intersects(bridge)`** (132–138) — без out-моста → false, иначе `intersects(out, bridge)`.
- **`merge(bridge, temp)`** (190–212): склейка имён через запятую → `m_space_restriction_manager->restriction(temp)` (кэшируемая композиция).
- **`add_border` / `remove_border`** (inline 37–65 / cpp 296–323): `ai().level_graph().set_mask(border())` / `clear_mask(...)` — маскирование вершин графа; `m_applied`-флаг; при out-мосте маскируется out-бордер, иначе in-бордер.
- **`accessible_nearest`** (325–332): out → `out->accessible_nearest(this, position, result, true)`, иначе `VERIFY(in)` + `in->accessible_nearest(in, ..., false)`.
- **`affect(bridge, ...)`** (334–372): 4 перегрузки (сфера/vertex+radius/две позиции/два vertex) — фактически `bridge->inside(sphere) == false`; старое тело (distance ≤ radius + `dependent_distance = 100.f`, строка 17) — закомментировано.
- **`name()`** (374–377) → `m_out_restrictions`.

## 5. `CSpaceRestrictionShape` (`space_restriction_shape.h/.cpp`, 47+237)

Обёртка `CSpaceRestrictor`: поля `m_restrictor`, `m_default` (`default_restrictor`).

- **`fill_shape(shape)`** (57–106): AABB формы в мировых координатах (сфера: `P ± (R,0,R)` + `Position()`; бокс: 8 полуточек через `Q.mul_43(XFORM(), box)`), затем **`ai().level_graph().iterate_vertices(start, dest, CBorderMergePredicate)`** — сбор вершин, попавших в объём; DEBUG: второй проход `CShapeTestPredicate` → `m_test_storage` (все «полностью внутри» для `test_correctness`).
- **`CBorderMergePredicate`** (16–36): вершина кладётся в `m_border`, если `inside(id, true) && !inside(id, false)` (**частично внутри** — кандидат в бордер); второй `operator()` (bool) — «полностью вне» для `remove_if`.
- **`build_border()`** (108–136): очистка → `fill_shape` для всех форм → `remove_if(!inside(false))` → **`process_borders()`** (из `CSpaceRestrictionBase`) → `VERIFY3(!border().empty(), "space restrictor has no border", ...)` → DEBUG `test_correctness`.
- **DEBUG `test_correctness`** (139–215): `set_mask(border())` → `graph_engine().search(..., CFlooder())` (flood внутри ограничения) → `clear_mask` → `m_correct = (m_test_storage.size() == nodes.size())` (все собранные вершины достижимы; дамп-код закомментирован).
- **Делегаты:** `inside(sphere)` → `m_restrictor->inside` (218–223), `name()` → `cName()` (225–229), `sphere()` → Center/Radius (231–237).

## 6. `CSpaceRestrictionHolder` (`space_restriction_holder.h/.cpp`, 62+224)

Реестр мостов: `typedef intrusive_ptr<CSpaceRestrictionBridge> CBaseRestrictionPtr`; `RESTRICTIONS = xr_map<shared_str, CSpaceRestrictionBridge*>`; `MAX_RESTRICTION_PER_TYPE_COUNT = 128`; `time_to_delete = 300000` мс (23).

- **`normalize_string`** (37–94): разбор строки имён через запятую (буферы на `_alloca`), **сортировка** → детерминированный ключ кэша («a,b» == «b,a»).
- **`restriction(space_restrictors)`** (96–113): пусто → 0; normalize → поиск в кэше → `collect_garbage()` → `xr_new<CSpaceRestrictionComposition>(this, str)` + `xr_new<CSpaceRestrictionBridge>(composition)` → insert; возврат моста. (**Композиция — порц. 36.**)
- **`register_restrictor(restrictor, type)`** (115–154): тип ≠ `eDefaultRestrictorTypeNone` → дописать имя в `m_default_out_restrictions` (Out) или `m_default_in_restrictions` (In) через `normalize_string` → при изменении виртуальный **`on_default_restrictions_changed()`**; далее `xr_new<CSpaceRestrictionShape>(restrictor, type != None)`; если имя уже в кэше → `bridge->change_implementation(shape)`, иначе новый bridge.
- **`try_remove_string`** (156–181) / **`unregister_restrictor`** (183–205): убрать имя из дефолтных строк (+`on_default_restrictions_changed`); **заменить реализацию моста на `CSpaceRestrictionComposition(this, restrictor_id)`** (даже после удаления зоны имя остаётся в кэше как композиция), `collect_garbage`.
- **`collect_garbage`** (207–224): удалить bridge, если `!shape()` (композиция) && `intrusive_ref_count() == 0` && `Device.dwTimeGlobal ≥ m_last_time_dec + 300000` — отложенная GC.

## 7. `CSpaceRestrictionBridge` (`space_restriction_bridge.h/.cpp`, 47+155)

Мост между intrusive-пользователями (`CSpaceRestriction`) и реализацией (`CSpaceRestrictionBase`): `RestrictionSpace::CTimeIntrusiveBase`, `m_object`; деструктор — `xr_delete(m_object)` (16–19); **`change_implementation`** (21–25) — удалить старый объект, поставить новый (используется holder'ом при register/unregister).

Делегирование: `object()` (inline-заголовок), `border/initialized/initialize/name/accessible_nearest/shape/default_restrictor/sphere` (27–65); **`inside`×3** (67–89) — под `START_PROFILE("Restricted Object/Bridge/Inside Vertex|Position")`; **`on_border(position)`** (99–135): VERIFY valid position; `lower_bound` бордера по `position().xz()` (`CFindByXZ_predicate`); при промахе → false; иначе `vertex_id(position)` и поиск его в группе вершин с равным xz; **`out_of_border(position)`** (137–155): невалидная вершина → `true`, иначе сфера `EPS_L` с `P.y = vertex_plane_y(id, x, z)` → `!inside(temp)`.

## 8. Связи

- **Порц. 36** — `CSpaceRestrictionComposition`, `CSpaceRestrictionManager` (`ai().restrictions()`), `space_restriction_abstract.h/.cpp` + `space_restriction_inline.h` уже учтены здесь (§4).
- **Порц. 40 / ит. 5** — `CCustomZone` (DEBUG-рендер, `ZoneState`), `CSE_ALifeSpaceRestrictor` (`m_space_restrictor_type`, `shapes`).
- **Порц. 9 (Level/Feel)** — `Level().space_restriction_manager()`, `inside(Fsphere)` как публичный предикат; `cast_restrictor()` (`GameObject.h:92` → NULL, `space_restrictor.h:63` → this) — виртуальный доступ без `smart_cast` (вызовов в коде не найдено — фактически резервный API).
- **Порц. 14–16 (movement/AI)** — `CRestrictedObject` вызывает `m_object->on_restrictions_change()` (`restricted_object.cpp:443`) при смене строк ограничений; цепочки `CMovementManager::on_restrictions_change` (`movement_manager.cpp:357`), `CMemoryManager` (`memory_manager.cpp:336`), `CCustomMonster` (`CustomMonster.cpp:1075`), `CAI_Stalker` (`ai_stalker_cover.cpp:326`), `CItemManager`/`CLevelManagerTemplate`; геттеры `CRestrictedObject::in/out_restrictions` (`restricted_object.cpp:235–256`).
- **Ит. (level graph)** — `set_mask/clear_mask/iterate_vertices/valid_vertex_position/vertex_plane_y`, `graph_engine().search(..., CFlooder)`.
- **Lua** — `script_game_object.cpp:1107` (`smart_cast<CSpaceRestrictor*>` — обёртка для скриптов), `DECLARE_SCRIPT_REGISTER_FUNCTION`.

## 9. Замечания к плану

1. **Статическая заглушка страницы ссылается на «порцион 24»** — актуальная нумерация плана итерации 5: **35** (этот блок) и **36 (см. ниже)**.
2. **`on_restrictions_change` / `restrictions()` — не методы `CSpaceRestrictor`** (их в классе нет): это виртуалы `CMovementManager`/`CRestrictedObject` и вся AI-иерархии (список в §8); план отнёс их к этому блоку как связи.
3. **`g_ai_die_in_anomaly`** (`space_restrictor.cpp:25`) объявлен, но в условии `net_Spawn` не используется (закомментировано на 82) — действует только безусловное снятие `STYPE_VISIBLEFORAI` у не-зон/радио/костров (83).
4. Ветка `USE_FREE_IN_RESTRICTIONS` (`space_restriction.h:16`) **выключена** — `merge_free_in_retrictions`/`m_free_in_restrictions`/`FREE_IN_RESTRICTIONS`-ветки в `add_border/remove_border` не компилируются; фактически работает только out/in-модель.

---

# Порцион 36: composition/manager/abstract + обвязка

## 10. `CSpaceRestrictionAbstract` (`space_restriction_abstract.h`, 45 строк + inline 82)

Абстракция «множество вершин» для `CSpaceRestrictionShape`/`CSpaceRestrictionComposition`/`CSpaceRestriction`:

- Поля (14–17): `xr_vector<u32> m_border`, `bool m_initialized`, `m_accessible_neighbour_border` + `m_accessible_neighbour_border_actual`.
- **`border()`** (inline 17–25): lazy `initialize()` → `THROW(initialized())` → `THROW3(!m_border.empty(), "Space restrictor has no border!", *name())`.
- Чистые виртуалы: `initialize()` (33), `name()` (42).
- **`accessible_neighbour_border<T>(restriction, out_restriction)`** (inline 33–47): lazy `prepare_accessible_neighbour_border`, `VERIFY2` «has no accessible neighbours».
- **`accessible_neighbours<T>`** (inline 50–66): обход 4(8) смежных вершин через `level_graph().begin/value`, пропуск невалидных; сосед подходит, если `restriction->inside(current, !out_restriction) != out_restriction` — т.е. out-бордер должен иметь **внутреннего** соседа, in-бордер — **внешнего**.
- **`prepare_accessible_neighbour_border<T>`** (inline 69–82): фильтр `border()` — остаются только вершины с доступным соседом (кэш по флагу).

## 11. `CSpaceRestrictionComposition` (`space_restriction_composition.h/.cpp`, 54+236 + inline 36)

Композиция: строка `"restrictor1,restrictor2,..."` → вектор мостов. Поля: `m_restrictions` (bridge-ptr), `m_space_restrictors`, `m_space_restriction_holder`, `m_sphere`; глобал `int g_restriction_checker` (28, `++` в ctor inline 20 / `--` в dtor 30–33 — учёт живых композиций).

- **Ctor** (inline 11–21): holder+строка, DEBUG `check_restrictor_type()`.
- **`shape()` = `false`**, **`default_restrictor()` = `false`** (inline 28–36) — композиция не-зона; `name()` → строка (23–26).
- **`merge(restriction)`** (50–54): push в `m_restrictions` + конкатенация его `border()` в `m_border`.
- **`inside(Fsphere)`** (56–75): lazy-init (не инициализирован → `initialize()`, всё ещё нет → `true`); быстрый отказ по `m_sphere.intersect`; затем OR по всем элементам.
- **`initialize()`** (77–156):
  1. `n = _GetItemCount`; VERIFY(n); **`n == 1`** → только DEBUG `m_correct=true` + `check_restrictor_type()` и выход **без `m_initialized = true`** — следствие: `inside()` всегда возвращает `true` (одиночное имя не ограничивает; так живёт композиция после удалённой зоны, §6 `unregister_restrictor`), а `border()` у такой композиции дойдёт до `THROW(initialized())` (в `NON_FATAL_VERIFY`/по умолчанию → `VERIFY` (debug-only), при `XRAY_EXCEPTIONS` → `throw`; `StdAfx.h:28–40`);
  2. если хоть один элемент не `initialized()` → return (ждём);
  3. `merge` каждого элемента + сбор его `sphere()`;
  4. **оболочка**: `Fbox3` по всем сферам → `m_sphere.P` = центр бокса (`mad(min,max,.5f)`), `m_sphere.R` = максимум `dist(P, center_i)+R_i`, `+= EPS_L`;
  5. `m_initialized = true`; `m_border.erase(remove_if(CMergePredicate — полностью внутри → выкинуть))` — из бордеров остаются только **внешние/частичные** кромки композиции;
  6. `process_borders()`; DEBUG `test_correctness()`.
- **DEBUG `test_correctness`** (159–207): слияние `m_test_storage` всех элементов (sort+unique); пустое хранилище → `m_correct = false` + выход; далее для каждого элемента flood под маской общей границы от его `m_test_storage.back()`: `nodes.size() == 65535` → ok (спец-значение), иначе `m_correct = (m_test_storage.size() <= nodes.size())`; при `!m_correct` — break.
- **`sphere()`** (210–216): `NODEFAULT` — в релизе **нет возврата** (DEBUG возвращает `m_sphere`); в ветке `n==1` `m_sphere` не заполняется.
- **DEBUG `check_restrictor_type`** (219–235): guard `if (_GetItemCount == 1) return` (из ctor тоже) → работает **только при n>1**; guard `!ai().get_alife()`; ищет объект по имени **всей строки** (`Level().Objects.FindObjectByName(m_space_restrictors)` — для "a,b" почти не находит) → `VERIFY3` «you are trying to use object as a restrictor»; далее **два `VERIFY2` с противоположными условиями** (`restrictor_type() == eDefaultRestrictorTypeNone` и `!= eDefaultRestrictorTypeNone`) — при найденном объекте срабатывает ровно один (сообщения взаимно противоречивы: «...with type eRestrictorTypeNone» / «impossible situation: wrong net_Spawn branch used»).

## 12. `CSpaceRestrictionManager` (`space_restriction_manager.h/.cpp`, 85+318 + inline 24)

**Наследует `CSpaceRestrictionHolder`** (24) — это глобальный реестр; `CLevel` наследует его, доступ — `Level().space_restriction_manager()` / `ai().restrictions()`. Доп. карта клиентов:

- `CClientRestriction` (18–23): `CRestrictionPtr m_restriction` (`intrusive_ptr<CSpaceRestriction>`) + `m_base_out_restrictions`/`m_base_in_restrictions` (оригинальные строки);
- `SPACE_RESTRICTIONS m_space_restrictions` — кэш `out\x01in` → `CSpaceRestriction*`; `CLIENT_RESTRICTIONS* m_clients` — id → клиент; `time_to_delete = 300000` (16).

### API (по таблице)

| Метод | Строки | Логика |
|---|---|---|
| `CSpaceRestrictionManager()` / `~` | 25–34 | `m_clients = xr_new<CLIENT_RESTRICTIONS>()`; в деструкторе `xr_delete(m_clients)` + `delete_data(m_space_restrictions)` |
| `restrict(id, out, in)` | 123–144 | из default-строк вычесть конфликтующие (`difference_restrictions`: default_out − merged_in, default_in − merged_out), затем слить (`join_restrictions`); `VERIFY2` «Restriction cannot be changed since its border is still applied!»; `(*m_clients)[id] = {restriction(merged_out, merged_in), base_out, base_in}`; `collect_garbage` |
| `unrestrict(id)` | 146–152 | erase клиента + GC |
| `add_restrictions/remove_restrictions/change_restrictions` | 242–310 | `VERIFY(!applied)`; работа с **base**-строками клиента (join/difference) → повторный `restrict` (без клиента: `add`/`change` → `restrict(add)`, `remove` → no-op) |
| `restriction(out, in)` (protected) | 172–192 | обе пустые → 0; `normalize_string` обеих; ключ `out + "\x01" + in`; кэш-попадание или `xr_new<CSpaceRestriction>(this, out, in)` |
| `restriction(id)` | 98–103 | поиск клиента (VERIFY) |
| `accessible(id, sphere/vertex,radius)` | 154–170 | делегат `CSpaceRestriction::accessible`; vertex `-1` → `false`; без клиента → `true` |
| `accessible_nearest(id, pos, result)` | 194–199 | делегат (VERIFY клиента) |
| `add_border(id, p1, p2)` (template, inline 18–24) / `remove_border(id)` | 61–66 | делегат `CSpaceRestriction::add_border/remove_border` — маскирование вершин графа |
| `in_restrictions/out_restrictions(id)` | 68–82 | строки из `CSpaceRestriction`; без клиента `""` |
| `base_in/out_restrictions(id)` | 84–96 | строки клиента (VERIFY) |
| `restriction_presented(restrictions, restriction)` | 201–208 | есть ли имя в строке (проверяет каждый элемент через `_GetItem`/`xr_strcmp`; вызывается также `ai/monsters/anomaly_detector.cpp:90`) |
| `join_restrictions` / `difference_restrictions` | 210–240 | инлайн-методы (`IC void`), строковые операции через `_GetItemCount/_GetItem` + `restriction_presented` — добавить/вычесть элементы строк |
| `collect_garbage()` | 105–121 | удалить `CSpaceRestriction` без ссылок и старше `time_to_delete` |
| **`on_default_restrictions_changed()`** | 312–318 | **пересоздать все клиентские ограничения**: цикл `restrict(base_out, base_in)` по `m_clients` (вызывается holder'ом при register/unregister default-зон) |
| `clear()` | 53–59 | клиенты + `delete_data(m_space_restrictions)` + `CSpaceRestrictionHolder::clear()` |
| DEBUG `restrictions()` | inline 11–16 | доступ к `m_space_restrictions` (использует `level_graph_debug2.cpp:174`) |

Логгеры `show_restriction` (36–51) — Msg-дамп out/in строк.

## 13. Клиенты менеджера

- **`CRestrictedObject`** (`restricted_object.cpp`) — основной потребитель: `restrict` (104), `unrestrict` (113), `accessible`/`accessible_nearest` (129/148/166), `add_border` (182/199/216 — верш+радиус/две позиции/пара вершин), `remove_border` (229), `add_restrictions` (vector 347, shared_str 390), `remove_restrictions` (vector 374, shared_str 405), сброс `restrict(id, "", "")` (434); геттеры `in/out/base_in/base_out_restrictions` (235–259) → цепочка `on_restrictions_change` (443, порц. 14–16).
- **`CAnomalyDetector`** — `restriction_presented` (`ai/monsters/anomaly_detector.cpp:90`).
- **DEBUG** — `level_graph_debug2.cpp:174` (итерация `restrictions()`).
- **Регистрация зон** — из `CSpaceRestrictor::net_Spawn/net_Destroy` (§2); default-типы `eDefaultRestrictorTypeOut/In` → `m_default_*_restrictions` holder'а → `on_default_restrictions_changed` менеджера.

## 14. `CSE_ALifeAnomalousZone` (`alife_anomalous_zone.cpp`, 139 строк — кросс-ссылка, детали порц. 40)

Серверная ALife-часть зоны: `tpfGetBestWeapon` (22–29) — «оружия нет», `fHitPower=1`, `m_tHitType`; `tfGetActionType` (31–35) — всегда `eMeetActionTypeAttack`; `bfActive` (37–40) — `false || !interactive()`; `tpfGetBestDetector` (42–49) — `VERIFY2(false, ...)` (не должен вызываться); `spawn_artefacts` (51–128) — **закомментирован** (веса, `m_fAnomalyValue = m_maxPower*(1 − d/r)`); `on_spawn` (130–134) — только inherited; `keep_saved_data_anyway` (136–139) → `true`.

## 15. Связи (порц. 36)

- **Порц. 35** (§1–9 этой страницы) — shape/holder/bridge/`CSpaceRestriction`; `space_restriction_inline.h` (88) — `CSpaceRestriction`-инлайны.
- **Порц. 9 (Level/AI)** — `Level().space_restriction_manager()` как синглтон-точка входа; `ai().restrictions()` — альтернативный доступ; `CRestrictedObject` (порц. 14–16) — обвязка с `CMovementManager`.
- **Ит. 5/порц. 40** — `CSE_ALifeSpaceRestrictor` (`m_space_restrictor_type`, `shapes`), `CSE_ALifeAnomalousZone` (§14), `CCustomZone`.
- **Ит. (level graph)** — `set_mask/clear_mask`, `iterate_vertices`, `vertex_position/vertex_id/vertex_plane_y`, `CFlooder`.
- **Lua** — `script_game_object.cpp:1107`.

## 16. Замечания к плану (порц. 36)

1. `space_restriction_inline.h` (88) отнесён планом к блоку 36, фактически содержит инлайны **`CSpaceRestriction`** (порц. 35) — задокументирован в §4.
2. `CSpaceRestrictor`-обвязка («порц. 24» в плане) уже раскрыта в §2–3; номер порциона в плане — легаси-ссылка (см. «Замечания» §9.1).
3. `alife_anomalous_zone.cpp` подтверждён: 139 строк, активного спавна артефактов нет (закомментирован) — отсылка к порц. 40 (CEvent/CustomZone) сохранена.

## 17. Краткое резюме (порционы 35–36)

`CSpaceRestrictor` — невидимый объект-оболочка (`CCF_Shape`: сферы/ОBB), кэширует мировые сферы/6-плоскостные боксы и проверяет `inside(Fsphere)`; при спавне регистрируется в `Level().space_restriction_manager()`. Слой ограничений: `CSpaceRestrictionShape` строит бордер из вершин level graph (частично-внутренние), `CSpaceRestrictionBridge` — intrusive-мост с делегированием и `on_border`/`out_of_border`, `CSpaceRestriction` — пара out/in для объекта с ленивой инициализацией, слиянием бордеров и маскированием графа (`set_mask/clear_mask`); `CSpaceRestrictionHolder` кэширует мосты по нормализованным строкам имён и хранит default in/out-строки, GC удаляет неиспользуемые композиции через 300 с. Порц. 36: `CSpaceRestrictionAbstract` даёт border+`accessible_neighbour_border`, `CSpaceRestrictionComposition` склеивает N зон (общая оболочка-сфера, бордер = внешняя кромка, flood-проверка в DEBUG), а `CSpaceRestrictionManager` (наследник holder, синглтон через `Level()`) ведёт карту клиентов `id → {out,in}` с join/difference-алгеброй строк, default-пересозданием через `on_default_restrictions_changed` и кэшем `out\x01in → CSpaceRestriction`; главный клиент — `CRestrictedObject` (порц. 14–16).

---

*Порционы 35–36 (итерация 5) · далее: порц. 37–38 — `zones-custom.md` (CCustomZone/CRadioactiveZone)*

