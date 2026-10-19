# План документирования — Итерация 5: xrGame: Actor / GameObject / Entity — объект-модель

> Рабочий документ-план. Удаляется после завершения итерации 5 (лог — в §9 GENERATE-DOCS.md).

## Масштаб

Итерация 5 — «объектная модель» xrGame: всё, из чего состоит мир в игре и кто есть кто. Ядро:

- `CObject` (xrEngine) → `CGameObject` (C++-ядро + Lua-обёртка)
- `CGameObject` → `CEntity` → `CEntityAlive` → `CActor` (игрок) и `CCustomMonster` (мост в AI)
- `CGameObject` → `CInventoryItemObject` (вещи инвентаря; оружие → итерация 8)
- `CGameObject` → `CSpaceRestrictor` → `CCustomZone` (зоны)
- Lua-обёртка `CScriptGameObject` (скрипты видят все объекты мира)

Покрытие: ~45 классов/файлов в `xrGame/` (Actor, CGameObject, CEntity*, CInventoryItemObject, CCustomZone и др.) + `CScriptGameObject` (~1300 строк). Остальные ~650 файлов xrGame (AI, физика, оружие, UI, уровни) — итерации 6–14.

## Решения (фиксация)

1. **`CGO` (legacy) не документируется отдельно** — упомянуть в порционах 2 и 48 как исторический предок `CGameObject`.
2. **AI-внутренности (`ai/` папка) — итерация 6.** Порцион 9 (`CCustomMonster`) — только как мост: класс, `Think()`, интерфейс для AI.
3. **`CWeapon` и наследники — итерация 8.** Порцион 10 лишь отмечает место в дереве.
4. **`Level_*` / сетевой код — итерация 12.** В порционах 4/10/19/27/30/35/41/43 — только сигнатуры `save/load/net_*` как «здесь будет сеть».
5. **Полный Lua API `CScriptGameObject` — итерация 14.** Порционы 13–18 — только механика обёртки и группы API.
6. **`architecture/object-model.md` (ит. 0) ОБНОВЛЯЕТСЯ** после порциона 2 и порциона 48.
7. **Разбивка (2026-10-12)**: порционы 1–12 задокументированы и **не перенумеруются**. Порционы 13–30 — слишком крупные куски, раскрошены на 36 (13–47) по коду, а не по страницам: один порцион = один самодостаточный блок кода. Страницы переиспользуются (заглушки уже существуют), новые страницы — только там, где кусок реально отдельный (`actor-death.md`, `script-object-api-2.md`, `zones-subclasses-2.md`). Кросс-ссылки на «порц. N» в готовых страницах (1–12) и в логах GENERATE-DOCS §9 **не переписываются** — старые номера сохраняются как «порц. N (старая нумерация)»; таблица 48 — единственная рабочая.

## Порционы

| #   | Порционы                                                               | Страницы                                                               |
| --- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 1   | Фундамент: CObject (xrEngine)                                          | `index.md`, `xr-object.md`                                             |
| 2   | Фабрика и регистр классов                                              | `class-registry.md`                                                    |
| 3   | CGameObject: каркас                                                    | `game-object.md`                                                       |
| 4   | CGameObject: сеть и жизненный цикл                                     | `game-object-net.md`                                                   |
| 5   | CEntity: здоровье и урон                                               | `entity.md`                                                            |
| 6   | EntityCondition: статы                                                 | `entity-condition.md`                                                  |
| 7   | CEntityAlive: живое существо                                           | `entity-alive.md`                                                      |
| 8   | Травмы, раны, смерть                                                   | `wounds-death.md`                                                      |
| 9   | CCustomMonster: мост в AI                                              | `custom-monster.md`                                                    |
| 10  | CInventoryItemObject                                                   | `inventory-item.md`                                                    |
| 11  | Torch/Flashlight/CustomDevice/Detectors                                | `misc-items.md`                                                        |
| 12  | PDA/Dosimeter/Antirad/Medkit                                           | `medical-items.md`                                                     |
| 13  | Lua-обёртка CScriptGameObject: механика                                | `script-game-object.md`                                                |
| 14  | CScriptGameObject: базовый API                                         | `script-game-object.md`                                                |
| 15  | CScriptGameObject: инвентарь и предметы                                | `script-game-object.md`                                                |
| 16  | CScriptGameObject: оружие, holder, снайпинг                            | `script-game-object.md`                                                |
| 17  | CScriptGameObject: мир, эффекты, звук                                  | `script-game-object.md`                                                |
| 18  | CScriptGameObject: Lua-регистрация (luabind)                           | `script-game-object.md`                                                |
| 19  | CActor: каркас                                                         | `actor.md`                                                             |
| 20  | CActor: условия и статы (CActorCondition)                              | `actor.md`                                                             |
| 21  | CActor: Hit/PHHit и урон                                               | `actor.md`                                                             |
| 22  | CActor: смерть                                                         | `actor-death.md` (новая)                                               |
| 23  | CActor: движение и физика                                              | `actor-movement.md`                                                    |
| 24  | CActor: input                                                          | `actor-input.md`                                                       |
| 25  | CActor: feel                                                           | `actor-input.md`                                                       |
| 26  | CActor: holder (транспорт)                                             | `actor-weapon.md`                                                      |
| 27  | CActor: оружие-обвязка (снаряды, drop, misc)                           | `actor-weapon.md`                                                      |
| 28  | CActor: сеть и интерполяция                                            | `actor-net.md`                                                         |
| 29  | CActorMP: клиент, state, импорт/экспорт                                | `actor-mp.md`                                                          |
| 30  | CActorMP: сервер и RPC                                                 | `actor-mp.md`                                                          |
| 31  | CActor: анимация                                                       | `actor-animation.md`                                                   |
| 32  | CActor: HUD и рендер                                                   | `actor-hud.md`                                                         |
| 33  | CActor: инвентарь-события, артефакты на поясе                          | `actor-misc.md`                                                        |
| 34  | CActor: сон, ночное зрение, прочее                                     | `actor-misc.md`                                                        |
| 35  | Зоны: CSpaceRestrictor + space_restriction (shape/holder/bridge)       | `zones-restrictor.md`                                                  |
| 36  | Зоны: composition/manager/abstract + CSpaceRestrictor-обвязка          | `zones-restrictor.md`                                                  |
| 37  | Зоны: CCustomZone FSM и частицы                                        | `zones-custom.md`                                                      |
| 38  | Зоны: свет, ветер, grass, аномалии-эффекторы                           | `zones-custom.md`                                                      |
| 39  | Зоны: подклассы (MosquitoBald, CZoneCampfire, CVisualZone, CHairsZone) | `zones-subclasses-2.md` (новая)                                        |
| 40  | Зоны: аномалии и smart-террейн                                         | `zones-anomaly.md`                                                     |
| 41  | Feel: Touch (xrEngine + GlobalFeelTouch)                               | `feel.md`                                                              |
| 42  | Feel: Vision и Sound                                                   | `feel.md`                                                              |
| 43  | Script: CScriptEntity (control, actions, visibility)                   | `script-entity.md`                                                     |
| 44  | Script: CScriptHit + monster hit info + действия                       | `script-entity.md`                                                     |
| 45  | Script: attachments                                                    | `script-attachment.md`                                                 |
| 46  | Script: CScriptBinder                                                  | `script-attachment.md`                                                 |
| 47  | Script: CScriptObject и прочее (actions/sound/effector/zone)           | `script-attachment.md`                                                 |
| 48  | Связность: object-model + зависимости                                  | update `architecture/object-model.md` + `dependencies.md` + `index.md` |

### Состав раскрошенных порционов (что в каждом)

**13–18. `CScriptGameObject`** (`script_game_object.h` 1352 + `script_game_object*.cpp` ~7 200 стр.). Обещание ит. 14 остаётся: здесь — механика обёртки и описание групп API; полный Lua API — итерация 14.

- **13. Механика**: класс (1352 стр. .h), ctor/dtor, `is_valid()` (back-reference), `object()`, `Parent()`, `clsid()`, `Hit(CScriptHit*)`, связь с `CGameObject::lua_game_object()` (порц. 3), `CScriptBinder`-мост (детали — порц. 46), `net_Spawn`/`net_Destroy`/`save`/`load` обёртки, DEBUG-режимы.
- **14. Базовый API**: базовые поля/геттеры (name, clsid, ID, position, direction, visual, radius, `center`), `net_*`-примитивы, `set_*`/`get_*` состояние, `feel_*`-обвязка, `use_*` базовые.
- **15. Инвентарь и предметы**: `script_game_object_inventory_owner.cpp` (3031 стр.): `owner()`, `slot_*`, `give_item`, `take_item`, `is_in`, `items_count`, `get_item`, `place_*`, `belt_*`, `outfit_*`, `set_belt_position`, `sort`, `find`, `can_*`-проверки, `add_to_inventory`, `remove_from_inventory`, `update_on_*`.
- **16. Оружие, holder, снайпинг**: `script_game_object3.cpp` (2229 стр.): `holder_*` (attach/detach/switch), `fire_*`, `reload_*`, `zoom_*`, `sight_*`/снайпинг (`SightManager`), `weapon_*`, `ammo_*`, `attach_*` (оружейные навески), `shot_*`, `unlimited_ammo`, `missile_*`.
- **17. Мир, эффекты, звук**: `script_game_object2.cpp` (816) + `script_game_object4.cpp` (581) + `script_game_object_use*.cpp` (671): `set_*`/`get_*` мира (position, orientation, visibility, `set_position`, `set_direction`), эффекторы (`ef_*`, `switch_effect`), звук (`play_sound`, `stop_sound`, `sound_*`), `use*`-действия (`use`, `use_iron`, `use_binocular`, `use_grenade`, `use_medkit`, …), `script_game_object_smart_covers.cpp` (509), trader-обвязка (69 + 22).
- **18. Lua-регистрация**: `script_game_object_script*.cpp` (149 + 466 + 712 + 22): luabind-деklarации классов (`class_<CScriptGameObject>`), экспорт методов по группам 14–17, `register_script_classes()` (порц. 2), `script_clsid` (порц. 2), `CObjectFactory`-связь.

**19–22. `CActor` — каркас** (`Actor.h` 922, `Actor.cpp` 3128; поля 134 шт.).

- **19. Каркас**: ctor/dtor, `reinit`/`reload`/`Load` (ini: `box_*`, `ph_*`, `camera`, `sprint`, `jump`…), `set_box`/`set_box_y_offset`/`set_actor_box_y_offset`, `set_state_box`, `initFPCam`/`removeFPCam`, `reinit`-связь с `CActorCondition` (порц. 20), `create_entity_condition` (фабрика → `CActorCondition`), `shedule_Update` (каркас: порядок вызовов, дефер-сообщения, `UpdateDefferedMessages`), `set_safemode`, `SwitchOutBorder`, `start_tutorial`, `On_SetEntity`, `OnDifficultyChanged`, `GetMass`/`is_on_ground`/`is_ai_obstacle`, `NeedToDestroyObject`, `GetRestoreSpeed`, `OnItemTake/OnItemDrop/OnItemRuck/OnItemBelt` (каркас — детали порц. 33), `can_attach`, `use_center_to_aim`, `use_default_throw_force`, `missile_throw_force`, `unlimited_ammo`, `SetZoomRndSeed`/`SetShotRndSeed`, `GetMissileOffset`/`SetMissileOffset`, `g_SetAnimation`/`g_SetSprintAnimation`, `SetCallbacks`/`ResetCallbacks`, `GetDefaultActionForObject`, `HUDview`, `canRenderLegs`.
- **20. Условия и статы**: `ActorCondition.cpp` (1110) + `ActorCondition_script.cpp` (177): `CActorCondition` (наследник `CEntityConditionSimple`, порц. 6), `LoadCondition`/`LoadImmunities`, `UpdateCondition`/`UpdateConditionTime`, `ConditionHit` (переопределение), `radiation`/`poisoning`/`fatigue`/`thirst`/`hunger`/`bleeding`/`burn`/`wound`/`psychic` — специфика актёра, `SetCondition`/`GetCondition`, `hit/wound_bone_scale`-override, `GetWhoHitLastTime`, `reinit` (25.f), `save`/`load`, `remove_links`.
- **21. Hit/PHHit и урон**: `Hit` (554–834), `PHHit` (541–554), `HitMark` (766–834), `HitSignal` (834–864), `HitArtefactsOnBelt` (2744–2776), `GetProtection_ArtefactsOnBelt` (2776–2792), `OnCritical*HealthLoss`-обвязка (порц. 8), `HitSector`, связь с `CDamageManager` (порц. 5/8), `SetHitInfo` (порц. 3).
- **22. Смерть**: `Die` (867–1013, 146 строк — большой кусок), `ACTOR_BEFORE_DEATH_CALLBACK`-форк (порц. 5), `start_tutorial`-связь, `TimePassedAfterDeath`, `respawn_delay`/`respawn_auto`, `patch_frames`, `on_requested_spawn` (1722–1758), `net_ExportDeadBody`, `OnEvent`-смерть (порц. 4), `kill_type`/`SPECIAL_KILL_TYPE`-обвязка (порц. 8), `killer_victim_angle` (порц. 8), `CCharacterPhysicsSupport::in_Die` (ит. 7), `OnDeath`-callback (порц. 13).

**23–27. `CActor` — движение, input, feel, holder, оружие.**

- **23. Движение и физика**: `Actor_Movement.cpp` (733): `g_Physics` (1027–1100), `set_state_box` (1760–1773), `CanMove`/`CanJump`/`CanSprint`/`CanRun`/`CanAccelerate`, `StopAnyMove`/`StopSprint`, `AnyAction`/`AnyMove`, `is_jump`, `GetMovementSpeed`, `GetFireDispertion`, `MaxWalkWeight`/`get_additional_weight`/`GetWeaponAccuracyStm`, `ConvState`, `g_Orientate`/`g_cl_Orientate`/`g_sv_Orientate`, `g_LadderOrient`, `cam_Lookout`, `camUpdateLadder`/`cam_SetLadder`/`cam_UnsetLadder`, `camUpdateFreelook`/`cam_SetFreelook`/`cam_UnsetFreelook`, `CanUseFreelook`, `CameraHeight`, `SetPhPosition` (2542–2551), `ForceTransform` (2551–2568), `set_input_external_handler`/`input_external_handler_installed`, `g_cl_CheckControls`/`g_cl_ValidateMState`.
- **24. Input**: `ActorInput.cpp` (855) + `actor_input_handler.cpp` (37): `IInputReceiver`-обвязка (порц. 2 ит. 2), `IR_OnKeyboardPress/Release/Hold`, `IR_OnMousePress/Release/Hold/Move`, `key_*`/`mouse_*`-маппинг, `g_cl_CheckControls` (детали), `set_input_external_handler`, `PickupModeUpdate`/`PickupModeUpdate_COD`/`PickupInfoDraw`/`DrawPickupItems`, `GetDefaultActionForObject` (детали), `kINVENTORY`/`kWPN_*`/`kQUIT`-обработчики (порц. 11–12).
- **25. Feel**: `Actor_Feel.cpp` (397): `Feel::Touch`/`Vision`/`Sound`-обвязка актёра (порц. 41–42), `feel_touch_*` (PDA-контакты — порц. 12), `feel_vision_*`, `feel_sound_*`, `ActorCondition`-связь (порц. 20), `OnH_*`-обработчики (порц. 3), `feel_visible_isRelevant`.
- **26. Holder (транспорт)**: `attach_Vehicle` (1490–1555), `detach_Vehicle` (1555–1617), `use_HolderEx` (1480–1490 и 1617–1722, 2 реализации), `steer_Vehicle`, `use_Holder`/`ActorUse`, `CHolderCustom`-мост (ит. 8 — holder-объекты), `attach_Vehicle`-связь с `CCharacterPhysicsSupport` (ит. 7).
- **27. Оружие-обвязка (снаряды, drop, misc)**: `Actor_Weapon.cpp` (405): `g_PerformDrop` (2368–2383), `use_default_throw_force` (2383–2391), `missile_throw_force` (2391–2401), `OnItemDropUpdate` (2647–2660), `UpdateArtefactsOnBeltAndOutfit` (2672–2744), `HitArtefactsOnBelt`/`GetProtection_ArtefactsOnBelt` (детали — порц. 21), `SetZoomRndSeed`/`SetShotRndSeed` (детали — порц. 19), `GetMissileOffset`/`SetMissileOffset` (детали — порц. 19), `unlimited_ammo` (3044–3049), `SwitchNightVision`-обвязка (порц. 34).

**28. `CActor` — сеть и интерполяция** (`Actor_Network.cpp` 2028): `UpdateCL` (1128–1408, 280 строк — каркас + детали), `net_Import_Base`/`net_Import_Physic`/`net_Import_Base_proceed`/`net_Import_Physic_proceed`, `net_ExportDeadBody` (детали — порц. 22), `CalculateInterpolationParams`, `g_PerformDrop`-связь (порц. 27), `RPC_UpdateFaction`/`RPC_UpdateRank`/`RPC_UpdateReputation` (1421–1480), `SetActorVisibility`/`GetActorVisibility` (2094–2122), `GetVisibility`/`SetVisibilityValue`/`SetVisibilityFromObject` (2122–2184), `net_Spawn`/`net_Destroy` (порц. 3–4), `net_update`-формат (ит. 12 — сигнатуры).

**29–30. `CActorMP`** (`actor_mp_client*.cpp` 89+127+166, `actor_mp_server*.cpp` 44+48+48, `actor_mp_state.cpp` 288): `CActorMP : CActor, IAnticheatDumpable`, `actor_mp_state` (Fquaternion/Fvector/health/radiation/…), `m_state_holder`, `fill_state`/`process_packet`/`postprocess_packet`, `net_Export`/`net_Import` (client/server), `net_Relevant`, `actor_mp_state`-сериализация (`USE_DIFFERENCES`-закомм.), `IAnticheatDumpable`-обвязка, `CActorMP`-регистрация в `CObjectFactory` (порц. 2, `ADD_MP` L235 — единственный MP-класс), `CSE_ActorMP` (ит. 11/12), `RPC_*`-связь (порц. 28).

- **29. Клиент, state, импорт/экспорт**: `actor_mp_client.h` (89) + `actor_mp_client.cpp` (89) + `actor_mp_state.h` (288) + `actor_mp_state_inline.h` + `actor_mp_state.cpp` (288) + `actor_mp_client_import.cpp` (166) + `actor_mp_client_export.cpp` (127).
- **30. Сервер и RPC**: `actor_mp_server.h` (44) + `actor_mp_server.cpp` (44) + `actor_mp_server_import.cpp` (48) + `actor_mp_server_export.cpp` (48) + `RPC_UpdateFaction`/`RPC_UpdateRank`/`RPC_UpdateReputation` (порц. 28 — детали) + `actor_communication.cpp` (221) + `actor_statistic_mgr.cpp` (189) + `actor_memory.cpp` (50).

**31. `CActor` — анимация** (`ActorAnimation.cpp` 783): `AnimTorsoPlayCallBack` (2821–2832), `on_animation_start`/`on_animation_stop`/`on_animation_switch`, `animation_movement_controlled` (порц. 3), `create/destroy_anim_mov_ctrl` (порц. 3), `select_animation` (порц. 9 — `CCustomMonster`), `actor_anim_defs.h`, `CActorAnimation`-обвязка (порц. 19), `AnimationManager`-связь (ит. 6 — `CAI_Stalker`), `CScriptEntityAction`-обвязка (порц. 44).

**32. `CActor` — HUD и рендер** (`Actor.cpp` 1128–1408 `UpdateCL` HUD-часть, 2184–2440 рендер, 2401–2494 HUD): `renderable_Render` (2246–2368, 122 строк), `RenderCamAttached` (2184–2215), `AllowActorShadow` (2215–2235), `canRenderLegs` (2235–2246), `OnHUDDraw` (2401–2440), `RenderIndicator` (2440–2494), `RenderText` (2494–2542), `SetPhPosition`-связь (порц. 23), `UpdateCL` HUD-часть (порц. 28 — каркас), `CHudItem`/`CHUDManager`-мост (ит. 10), `renderable_ShadowGenerate` (детали), `g_fov`/`currentFOV` (1100–1128).

**33. `CActor` — инвентарь-события, артефакты на поясе** (`Actor.cpp` 2611–2744, 2792–2814, 2857–2875, 2896–2916): `OnItemTake` (2611–2617), `OnItemDrop` (2617–2647), `OnItemDropUpdate` (2647–2660), `OnItemRuck` (2660–2665), `OnItemBelt` (2665–2672), `UpdateArtefactsOnBeltAndOutfit` (2672–2744, 72 строк), `HitArtefactsOnBelt` (2744–2776), `GetProtection_ArtefactsOnBelt` (2776–2792), `can_attach` (2857–2875), `UpdateArtefactsOnBeltAndOutfit`-связь с `CInventory` (ит. 8), `CArtefact`-обвязка (ит. 10), `CInventoryOwner`-мост (порц. 10), `OnItemDropUpdate`-связь с `CInventoryItemObject` (порц. 10).

**34. `CActor` — сон, ночное зрение, прочее** (`Actor_Sleep.cpp` 1, `ActorNightVision.cpp` 89, `ActorHelmet.cpp` 344, `ActorBackpack.cpp` 102, `Actor_Events.cpp` 378): `Actor_Sleep.cpp` (1 строка — заглушка, документировать как мёртвый код), `ActorNightVision` (89): `SwitchNightVision` (3049–3113), `NV_*`-состояния, `CHelmet`-обвязка (344): `ActorHelmet` (наследник `CEquipment`? — проверить в коде), `CBackpack` (102): `CActorBackpack`, `Actor_Events.cpp` (378): `OnEvent`-обработчики актёра (порц. 4 — каркас), `GE_*`-обвязка, `SetActorVisibility`-связь (порц. 28), `actor_defs.h`/`actor_statistic_defs.h`/`actor_communication.cpp` (221 — детали порц. 30).

**35–40. Зоны** (`space_restriction*.h/.cpp` 2602, `CustomZone.cpp` 1764 + `CustomZone.h` 378, подклассы ~1 000).

- **35. CSpaceRestrictor + space_restriction (shape/holder/bridge)**: `space_restrictor.h/.cpp` (75+332): `CSpaceRestrictor` (базовый класс зон, порц. 24 — детали), `inside(Fsphere)` (порц. 9 — feel_touch), `on_restrictions_change` (порц. 9), `restrictions()` (порц. 9), `space_restriction.h/.cpp` (96+377): `SpaceRestriction` (базовый), `space_restriction_shape.h/.cpp` (47+237): `Shape` (OBB/sphere), `space_restriction_holder.h/.cpp` (62+224): `Holder` (связь с `CGameObject`), `space_restriction_bridge.h/.cpp` (47+155): `Bridge` (мост между зонами).
- **36. Composition/manager/abstract + CSpaceRestrictor-обвязка**: `space_restriction_composition.h/.cpp` (54+236): `Composition` (композиция зон), `space_restriction_manager.h/.cpp` (85+318): `Manager` (глобальный менеджер зон, `ai().restrictions()`, порц. 9), `space_restriction_abstract.h` (45): `Abstract` (сетевая обвязка, ит. 11/12), `space_restriction_inline.h` (88), `CSpaceRestrictor`-обвязка (порц. 24 — детали), `alife_anomalous_zone.cpp` (139 — детали порц. 40).
- **37. CCustomZone FSM и частицы**: `CustomZone.h` (378) + `CustomZone.cpp` (1764, 1–900): `CCustomZone : CSpaceRestrictor` (L37), ctor/dtor, `Load`/`reload`/`reinit`, `net_Spawn`/`net_Destroy`/`net_Export`/`net_Import`/`net_Relcase` (ит. 11/12 — сигнатуры), `shedule_Update` (FSM), `SwitchZoneState`/`OnStateSwitch`, `EZoneState` (eDisabled/eEnabled/eBlowout…), `AwakingState`/`AccumulateState`/`IdleState`/`BlowoutState`, `CheckForAwaking`, `enter_Zone`/`exit_Zone`, `AffectObjects`, `feel_touch_*` (порц. 41), `Hit`/`CreateHit`/`UpdateSecondaryHit`, `ef_anomaly_type`/`ef_weapon_type` (порц. 3), `Power`/`RelativePower`/`effective_radius`, `CalcDistanceTo`, `generate_orthonormal_basis`, `AlwaysTheCrow`, `MoveScript`, `OnMove`, `OnEvent` (порц. 4 — каркас), `save`/`load`.
- **38. Свет, ветер, grass, аномалии-эффекторы**: `CustomZone.cpp` (1764, 900–1764): `UpdateCL`, `UpdateOnOffState`, `UpdateIdleLight`/`StartIdleLight`/`StopIdleLight`, `UpdateBlowoutLight`/`StartBlowoutLight`/`StopBlowoutLight`, `UpdateWind`/`StartWind`/`StopWind`, `UpdateWorkload`, `GrassZoneUpdate` (grass-обвязка, ит. 9), `ChangeIdleParticles`, `PlayIdleParticles`/`StopIdleParticles`/`PlayObjectIdleParticles`/`StopObjectIdleParticles`, `PlayHitParticles`/`PlayBulletParticles`/`PlayEntranceParticles`/`PlayBoltEntranceParticles`/`PlayAccumParticles`/`PlayAwakingParticles`/`PlayBlowoutParticles` (частицы — ит. 4), `o_switch_` (2 реализации), `ZoneEnable`/`ZoneDisable`, `GoEnabledState`/`GoDisabledState`.
- **39. Подклассы (MosquitoBald, CZoneCampfire, CVisualZone, CHairsZone)**: `MosquitoBald.h/.cpp` (123): `CMosquitoBald : CCustomZone`, `Load`, `Affect`, `BlowoutState`, `UpdateSecondaryHit`; `ZoneCampfire.h/.cpp` (32+191): `CZoneCampfire : CMosquitoBald` (детали); `ZoneVisual.h/.cpp` (87): `CVisualZone : CCustomZone` (`m_idle_animation`/`m_attack_animation`, `SwitchZoneState`, `Load`, `UpdateBlowout`); `HairsZone.h/.cpp` (23+77): `CHairsZone : CVisualZone`; `team_base_zone.cpp` (184): `CTeamBaseZone`; `mosquito_bald_script.cpp` (30), `ZoneCampfire`-скрипт (ит. 14), `HairsZone_script.cpp`, `CustomOutfit_script.cpp` (70), `searchlight_script.cpp` (28).
- **40. Аномалии и smart-террейн**: `alife_anomalous_zone.cpp` (139), `alife_smart_zone.cpp` (35), `smart_zone.h` (19): `CSmartZone : CSpaceRestrictor`, `AmebaZone.cpp` (111), `FryupZone.cpp` (16), `GraviZone.cpp` (268), `NoGravityZone.cpp` (64), `RadioactiveZone.cpp` (161), `TorridZone.cpp` (78), `zone_effector.cpp` (86): `ZoneEffector` (эффектор-мост, ит. 9), `script_zone.cpp`/`script_zone.h`/`script_zone_script.cpp` (Lua-обвязка зон — ит. 14), `CustomZone_script.cpp` (14).

**41–42. Feel** (`xrEngine/Feel_Touch.cpp` 108 + `Feel_Touch.h` 44, `xrEngine/Feel_Vision.cpp` 319 + `Feel_Vision.h` 78, `xrEngine/Feel_Sound.h` 15, `xrGame/GlobalFeelTouch.cpp` 56 + `GlobalFeelTouch.hpp` 19, `xrGame/Actor_Feel.cpp` 397 — порц. 25).

- **41. Touch**: `xrEngine/Feel_Touch.h/.cpp`: `Feel::Touch` (базовый класс), `feel_touch_new`/`feel_touch_delete`/`feel_touch_on_contact`/`feel_touch_contact`/`feel_touch_update`, `CDetectList<K>` (порц. 11), `CAfList`/`CZoneList` (порц. 11), `ITEM_TYPE`/`ITEM_INFO`; `xrGame/GlobalFeelTouch.cpp/.hpp`: `GlobalFeelTouch` (глобальный touch-менеджер, `m_artefacts` — порц. 11); `xrGame/Actor_Feel.cpp` (397) — детали порц. 25.
- **42. Vision и Sound**: `xrEngine/Feel_Vision.h/.cpp`: `Feel::Vision` (319 строк): `feel_vision_query`/`feel_vision_update`/`feel_vision_mtl_transp` (порц. 9 — `CCustomMonster`), `feel_visible_isRelevant` (порц. 9), `eye_pp_*` (порц. 9 — 2-хступенчатое зрение), `update_range_fov` (порц. 9); `xrEngine/Feel_Sound.h` (15): `Feel::Sound` (базовый, `feel_sound_new`/`feel_sound_delete`/`feel_sound_update`, порц. 9 — `CCustomMonster`); `CScriptEntity::process_sound_callbacks` (порц. 43).

**43–47. Script-обвязка** (`script_entity*.cpp` 769+15+24, `script_hit*.cpp` 14+47, `script_attachment*.cpp`, `script_binder*.cpp`, `script_object*.cpp`, `script_*_action*.cpp`).

- **43. CScriptEntity (control, actions, visibility)**: `script_entity.h` (127) + `script_entity.cpp` (769) + `script_entity_inline.h` (15) + `script_entity_space.h` (24): `CScriptEntity` (базовый класс скриптовой сущности, порц. 9 — `CCustomMonster`), `init`/`reinit`, `net_Spawn`/`net_Destroy` (порц. 3–4 — каркас), `shedule_Update`/`UpdateCL`, `GetUpdatedMatrix`, `vfUpdateParticles`/`vfUpdateSounds` (ит. 4), `vfFinishAction`, `SetScriptControl`/`GetScriptControl`/`GetScriptControlName` (порц. 9 — `CCustomMonster`), `CheckObjectVisibility`/`CheckTypeVisibility`, `CheckIfCompleted`, `GetCurrentAction`, `AddAction`, `ProcessScripts` (порц. 9 — `CCustomMonster`), `ResetScriptData`, `ClearActionQueue`, `bfAssignMovement`/`bfAssignWatch`/`bfAssignAnimation`/`bfAssignSound`/`bfAssignParticles`/`bfAssignObject`/`bfAssignMonsterAction`, `sound_callback`, `GetPatrolPathName`, `bfScriptAnimation` (порц. 9 — `CCustomMonster`), `GetActionCount`/`GetActionByIndex`, `GetCurrentEnemy`/`GetCurrentCorpse`, `get_enemy_strength` (ит. 6), `process_sound_callbacks` (порц. 9 — `CCustomMonster`), `set_script_capture`/`can_script_capture`, `object()`.
- **44. CScriptHit + monster hit info + действия**: `script_hit.h` (39) + `script_hit_inline.h` (49) + `script_hit.cpp` (14) + `script_hit_script.cpp` (47): `CScriptHit` (Lua-обёртка `SHit`, порц. 8), `ApplyScriptHit` (порц. 8 — `SHit`), `CScriptMonsterHitInfo` (30+37): `script_monster_hit_info.h/.cpp` (монстр-hit-info, ит. 6), `script_animation_action.cpp` (14), `script_effector.cpp` (37), `script_sound_action.cpp` (35), `script_movement_action.cpp`, `script_watch_action.cpp`, `script_object_action.cpp`, `script_particle_action.cpp`, `script_abstract_action.cpp`, `script_action_condition.cpp` (базовые action-классы, ит. 14 — полный API).
- **45. Attachments**: `script_attachment_manager.cpp/.h`, `script_attachment_script.cpp`, `script_attachment.h`: `CAttachmentOwner` (порц. 10 — `CInventoryItemObject`), `script_attachment` (Lua-обёртка), `CGameObject::m_script_attachments` (порц. 3), `add_attachment`/`get_attachment`/`remove_child`/`remove_attachment`/`iterate_attachments` (порц. 3), `update_visbox`/`update_visbox_hud` (порц. 3), `eSA_World`/`eSA_HUD` (порц. 3).
- **46. CScriptBinder**: `script_binder.h` (37) + `script_binder.cpp` (276) + `script_binder_inline.h` (22) + `script_binder_object.cpp` (L22–276) + `script_binder_object.h` + `script_binder_object_script.cpp` + `script_binder_object_wrapper.cpp` (порц. 4 — полный разбор): `CScriptBinder` (мост C++↔Lua), `m_object` (`CScriptBinderObject*`), `reload(section)`, `net_Spawn(DC)`, `net_Destroy()`, `set_object(object)` (`IsGameTypeSingle()` — MP: не привязываются), `shedule_Update`, `save`/`load`, `net_SaveRelevant`, `net_Relcase`, `try/catch`-обвязка (Lua-исключения), `CScriptBinderObject` (базовый обёртка).
- **47. CScriptObject и прочее (actions/sound/effector/zone)**: `script_object.h` (149) + `script_object.cpp`: `CScriptObject` (базовый скриптовый объект), `script_lanim.cpp`/`script_lanim.h` (Lua-обёртка lanim, ит. 3), `script_light.h`/`script_light_inline.h`/`script_light_script.cpp` (Lua-обёртка света, ит. 3), `script_particles.cpp`/`script_particles.h`/`script_particles_inline.h`/`script_particles_script.cpp` (Lua-обёртка частиц, ит. 4), `script_sound.cpp`/`script_sound.h`/`script_sound_inline.h`/`script_sound_script.cpp` (Lua-обёртка звука, ит. 4), `script_world_property.h`/`script_world_property_inline.h`/`script_world_property_script.cpp` (Lua-обёртка world-property, ит. 14), `script_imgui.h`/`script_imgui_inline.h`/`script_imgui_script.cpp` (ImGui-обвязка, ит. 10), `script_wallmarks_manager.cpp`/`script_wallmarks_manager.h`/`script_wallmarks_script.cpp` (wallmarks, порц. 7 — `CEntityAlive`), `script_abstract_action`/`script_action_condition`/`script_action_wrapper`/`script_action_planner_action_wrapper`/`script_action_planner_wrapper` (action-обвязка, ит. 14 — полный API).

**48. Связность**: `architecture/object-model.md` (ит. 0 — обновление с реальной иерархией), `dependencies.md` (зависимости), `index.md` (таблица 48 порционов, статусы), `mkdocs.yml` (nav — 3 новые страницы: `actor-death.md`, `script-object-api-2.md`, `zones-subclasses-2.md`), `GENERATE-DOCS.md` §9 (лог итерации 5), `ITERATION-5-PLAN.md` (удаление этого файла).

## Статус

| #   | Статус |
| --- | ------ |
| 1   | ✅     |
| 2   | ✅     |
| 3   | ✅     |
| 4   | ✅     |
| 5   | ✅     |
| 6   | ⬜     |
| 7   | ✅     |
| 8   | ✅     |
| 9   | ✅     |
| 10  | ✅     |
| 11  | ✅     |
| 12  | ✅     |
| 13  | ✅     |
| 14  | ✅     |
| 15  | ✅     |
| 16  | ✅     |
| 17  | ✅     |
| 18  | ✅     |
| 19  | ✅     |
| 20  | ✅     |
| 21  | ✅     |
| 22  | ✅     |
| 23  | ✅     |
| 24  | ✅     |
| 25  | ✅     |
| 26  | ✅     |
| 27  | ✅     |
| 28  | ✅     |
| 29  | ✅     |
| 30  | ✅     |
| 31  | ✅     |
| 32  | ✅     |
| 33  | ✅     |
| 34  | ✅     |
| 35  | ⬜     |
| 36  | ⬜     |
| 37  | ⬜     |
| 38  | ⬜     |
| 39  | ⬜     |
| 40  | ⬜     |
| 41  | ⬜     |
| 42  | ⬜     |
| 43  | ⬜     |
| 44  | ⬜     |
| 45  | ⬜     |
| 46  | ⬜     |
| 47  | ⬜     |
| 48  | ⬜     |

## Обещанное из итераций 0–4

- **`architecture/object-model.md`** (ит. 0) — обновляется в порционах 2, 48
- **`CObject`/`CGO`/`CSE_Abstract`** — порционы 1–2
- **`IKinematics`/`CBoneInstance`/`CVisual`** (ит. 3 порц. 8) — только использование из `CGameObject`/`CEntityAlive`
- **`CSE_ALife*`/`CSE_Object`** (ит. 11) — только `CSE_Abstract*` как аргумент `net_Spawn`
- **`CParticlesObject`** (ит. 4) — только как «соседний объект» в зонах/ранах
- **`CWeapon`-иерархия** (ит. 8) — только место в дереве (порцион 10)
- **AI-внутренности** (ит. 6) — только интерфейс `CCustomMonster` (порцион 9)
- **`CScriptGameObject` API** (ит. 14) — только механика обёртки (порционы 13–18)
- **`Level_*`/сетевой код** (ит. 12) — только сигнатуры (порционы 4/10/19/27/30/35/41/43)

## После итерации 5

- `docs/docs/modules/game/` — ~23 страницы (index, xr-object, game-object, entity*, actor*×11, zones*, items*, script*, feel) + 3 новые (`actor-death.md`, `script-object-api-2.md`, `zones-subclasses-2.md`)
- `docs/docs/architecture/object-model.md` — обновлена с реальной иерархией
- Лог итерации 5 → `GENERATE-DOCS.md` §9
- Этот файл удаляется
