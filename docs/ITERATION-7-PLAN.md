# Итерация 7: Физика (PH) — план порционов

> Рабочий документ-план. После завершения итерации 7 содержание
> переносится в `GENERATE-DOCS.md` §9 (лог итераций) и файл удаляется.

## Масштаб

**Калибрка (итерация 6, ALIFE):** 33 порциона, 33 страницы, `xrGame/ai/` + `alife_*`.

**Физика** — два модуля:

| Корзина | Файлов (.h+.cpp) | Строк (примерно) | Что |
| --- | --- | --- | --- |
| `xrPhysics/` | ~150 | ~27 000 | ODE-обвязка: `CPHWorld`, `CPHShell`, `CPHElement`, `CPHJoint`, `CPHCharacter` (+Actor/AI/Simple), `PhysicsShell` (C++ API), `PhysicsShellAnimator`, `PHFracture`, `PHIsland`, `PHInterpolation`, `PHDynamicData`, `PHCollideValidator`, `PHContactBodyEffector`, `PHActivationShape`, `PHStaticGeomShell`, `PHSplitedShell`, `PHShellSplitter`, `PHMoveStorage`, `PHShellBuildJoint`, `PHShellActivate`, `PHShellNetState`, `PHElementNetState`, `PHJointDestroyInfo`, `PHDisabling`, `PHCapture`, `PHAICharacter`, `PHActorCharacter`, `PHSimpleCharacter`, `MovementBoxDynamicActivate`, `phvalide`, `ph_valid_ode`, `ode_redefine`, `ode_include`, `matrix_utils`, `MathUtils`, `SpaceUtils`, `ShellHit`, `PhysicsExternalCommon`, `PhysicsCommon`, `Physics.h/cpp`, `params`, `ph_defs`, `phupdateobject`, `PhysicsShellAnimatorBoneData`, `debug_output`, `physics_scripted`, `iphysics_scripted`, `IPHWorld`, `IPHStaticGeomShell`, `IPHCapture`, `IPhysicsShellHolder`, `xrPhysics.h/cpp` (DllMain), `phvalide`, `SpaceUtils` |
| `xrGame/PH*` + `physics_*` | ~35 | ~11 000 | Игровая обвязка: `CPhysicObject`, `CPhysicsShellHolder`, `CPhysicsSkeletonObject`, `CCharacterPhysicsSupport`, `CPHMovementControl`, `CPHCommander`, `CPHDestroyable`, `CPHCollisionDamageReceiver`, `CPHShellCreator`, `CPHSoundPlayer`, `CPHSimpleCalls`, `CPHScriptCall`, `CPHReqComparer`, `CPHDebug`, `CPHMovementDynamicActivate`, `CDestroyablePhysicsObject`, `PhysicsGamePars`, `PHSkeleton`, `PHDestroyableNotificate`, `PHShellCreator`, `moving_object*` (7), `physics_game`, `physics_element_scripted`, `physics_shell_scripted`, `physics_shell_animated`, `physics_joint_scripted`, `physics_world_scripted` |
| **Итого** | **~185** | **~38 000** | ≈ 1.2× ALIFE по строкам |

**Вывод: 10 порционов.**

## Решения (фиксация)

1. **`xrPhysics` — один модуль, один подкаталог в mkdocs.**
   Страницы: `docs/docs/modules/physics/` (новый уровень, наравне с `game/`, `xr-core/`, `xr-engine/`).
   Старая заглушка `docs/docs/modules/game/physics.md` заменяется перенаправлением
   (`> Перенесено в [Физика](../physics/index.md)`).

2. **ODE / PhysX.** Проект использует **ODE** (`ode_include.h`, `ode_redefine.h`,
   `3rd party/ode/include/ode/*`). Заголовок `xrPhysics.vcxproj` ссылается на ODE,
   не на PhysX. В документации пишем «ODE (Open Dynamics Engine)», а не «PhysX».

3. **`moving_object*` (7 файлов, 1329 строк)** — не физика, а **движение/коллизии
   на уровне `CEntityAlive`** (static/dynamic query, action_type).
   Документируем в порционе 7 как часть игровой обвязки, не в `xrPhysics`.

4. **`physics_*_scripted.*`** (5 файлов) — Lua-биндинг физики.
   Кратко в порционе 2 (интерфейсы `cphysics_scripted`), детали Lua API —
   итерация 14 (Lua API).

5. **`PHDebug`** — только `#ifdef DEBUG`, 111 строк.
   Абзац в порционе 6 (debug-обвязка), отдельной страницы нет.

6. **`ik_collide_data.h` / `ik_foot_collider.*`** — IK-коллизии ног,
   не физический движок. Не трогаем (принадлежат к анимации/IK).

## Порционы

| # | Порцион | Страницы | Содержание |
| --- | --- | --- | --- |
| 1 | Точка входа, мир, базовые типы | `physics/index.md`, `ph-world.md`, `ph-object.md` | `xrPhysics.h/cpp` (DllMain, глобалы), `CPHWorld` (цикл обновления, freeze, step count, gravity), `PHObject` (базовый ODE-обёрток: body/geom/active/freeze), `PHDefs.h` (enum, typedef, DEFINE_VECTOR), `PHUpdateObject`, `phvalide`/`ph_valid_ode`, `ode_redefine`/`ode_include`, `SpaceUtils`, `matrix_utils`, `MathUtils`, `params` |
| 2 | C++ API и Lua-скрипты | `physics/shell-api.md` | `PhysicsShell.h/cpp` (C++ facade: `CreateShell`, `Activate`, `ApplyImpulse`, `GetElementTransform`, …), `PhysicsCommon.h`, `PhysicsExternalCommon`, `PhysicsShellAnimator.h/cpp` + `PhysicsShellAnimatorBoneData`, `physics_scripted` / `iphysics_scripted`, `physics_world_scripted`, `physics_element_scripted`, `physics_shell_scripted`, `physics_joint_scripted`, `IPhysicsShellHolder`, `IPHWorld`, `IPHStaticGeomShell`, `IPHCapture`, `ShellHit` |
| 3 | CPHShell и элементы | `physics/shell.md` | `CPHShell` (активация, elements/joints storage, splitter holder, traced geoms, bones callback), `CPHElement` (geom, body, bone, mass, activation, net state), `PHElementNetState`, `PHDynamicData`, `PHMoveStorage`, `PHActivationShape`, `PHShellActivate`, `PHShellBuildJoint`, `PHStaticGeomShell`, `PHSplitedShell`, `PHShellSplitter` |
| 4 | Суставы, трещины, острова | `physics/joints-fracture.md` | `CPHJoint` (типы, position, break), `PHJointDestroyInfo`, `CPHFracture` (break force/torque, mass split), `CShellSplitInfo`, `PHIsland` (island detection, activation), `PHInterpolation` (spline для сети), `PHShellNetState` |
| 5 | Персонажи | `physics/characters.md` | `CPHCharacter` (base: body, interpolation, mass, collision damage, environment), `CPHActorCharacter` (player: capture, restriction, elevator), `CPHAICharacter` (AI: path following, step), `PHSimpleCharacter` (inline: cylinder), `PHAICharacter`/`PHActorCharacterInline`, `MovementBoxDynamicActivate` |
| 6 | Коллизии, эффекты, дисейблинг | `physics/collide-effects.md` | `PHCollideValidator`, `PHContactBodyEffector`, `PHDisabling` (translational/rotational), `Physics.h` (NearCallback, CollideStatic, FixBody, ApplyGravityAccel, BodyCutForce, SaveContacts), `PHCapture` (bone capture), `PHActorCharacterInline` (capture logic), `PHDebug` (debug-глобалы, DRAW_CONTACTS) |
| 7 | Игровая обвязка: shell holder | `game/physics-shells.md` | `CPhysicsShellHolder` (CGameObject + IPhysicsShellHolder + IObjectPhysicsCollision), `CPhysicObject` (CPhysicsShellHolder + net + CSE_ALifeObjectPhysic), `CPhysicsSkeletonObject` (CSE_PHSkeleton, net state), `CCharacterPhysicsSupport` (CPHSkeleton + CPHDestroyable, actor/stalker/biting), `CPHSkeleton` (split, spawn copy, auto-remove), `PHDestroyable`, `PHDestroyableNotificate`, `CPHShellCreator`, `CDestroyablePhysicsObject` (health, destroy sound/particles), `PhysicsGamePars` (collide volume, effect pars) |
| 8 | Игровая обвязка: движение и урон | `game/physics-movement-damage.md` | `CPHMovementControl` (PHCapture, VirtualMoveTo, restriction, elevator, box update, in_shedule_Update), `CPHCommander` (CPHCondition/CPHAction/CPHCall/CPHCommander — request-система), `CPHCollisionDamageReceiver` (controled bones, CollisionHit), `CPHSoundPlayer` (hit sound by velocity), `CPHSimpleCalls` + `CPHSimpleCallsScript` + `CPHScriptCall`, `CPHReqComparer`, `CPHMovementDynamicActivate`, `physics_game.h/cpp` (ContactShotMark, CharacterContactShotMark) |
| 9 | Движение сущностей (moving_object) | `game/moving-objects.md` | `moving_object` (action_type, obstacles_query, predict_position, update_position), `moving_objects.h/cpp` (реестр), `moving_objects_inline.h`, `moving_objects_impl.h`, `moving_objects_dynamic.cpp`, `moving_objects_dynamic_collision.cpp`, `moving_objects_static.cpp`, `obstacles_query` |
| 10 | Связность и обновление арх. страниц | `physics/index-consistency.md` | Перекрёстные ссылки: `physics/index.md` ↔ `game/physics-shells.md` ↔ `game/physics-movement-damage.md` ↔ `game/moving-objects.md` ↔ `game/alife/*` (PHMovementControl в сталкерах) ↔ `xr-engine/collide-physics.md` (facade). Обновить: `architecture/object-model.md` (ветка CPhysicsShellHolder), `architecture/dependencies.md` (xrPhysics тянет ODE, xrGame тянет xrPhysics), `modules/game/index.md` (ссылка на physics/), `mkdocs.yml` (новый подкаталог `physics/`, перенаправление старой заглушки) |

## Статус

| # | Порцион | Статус |
| --- | --- | --- |
| 1 | Точка входа, мир, базовые типы | ✅ |
| 2 | C++ API и Lua-скрипты | ⬜ |
| 3 | CPHShell и элементы | ⬜ |
| 4 | Суставы, трещины, острова | ⬜ |
| 5 | Персонажи | ⬜ |
| 6 | Коллизии, эффекты, дисейблинг | ⬜ |
| 7 | Игровая обвязка: shell holder | ⬜ |
| 8 | Игровая обвязка: движение и урон | ⬜ |
| 9 | Движение сущностей (moving_object) | ⬜ |
| 10 | Связность и обновление арх. страниц | ⬜ |

## Обещанное из предыдущих итераций (кросс-ссылки, которые итерация 7 обязана закрыть)

- `xrEngine/collide-physics.md` (ит. 2) — ссылки на `CPHShell`/`CPHElement`/`CPHWorld`
  → порционы 1, 3, 6
- `game/objects.md` (ит. 5) — `CPhysicsShellHolder` как базовый класс
  → порцион 7
- `game/alife/steering-steps.md` (ит. 6, порц. 25) — `CPHMovementControl`
  → порцион 8
- `game/alife/stalker-movement.md` (ит. 6, порц. 14) — движение сталкера
  → порцион 8
- `game/zones.md` (ит. 5) — заглушка, физика зон не связана, не трогаем

## После итерации 7

- обновить `architecture/object-model.md` (ветка `CPhysicsShellHolder` →
  `CPhysicObject` / `CDestroyablePhysicsObject` / `CCharacterPhysicsSupport`)
- обновить `architecture/dependencies.md` (xrPhysics → ODE, xrGame → xrPhysics)
- обновить `modules/game/index.md` (ссылка на `physics/`)
- заменить заглушку `game/physics.md` на перенаправление
- `mkdocs.yml`: новый подкаталог `physics/` (10 пунктов), `game/physics-shells.md`,
  `game/physics-movement-damage.md`, `game/moving-objects.md`
- перенести лог в `GENERATE-DOCS.md` §9
- удалить этот файл
