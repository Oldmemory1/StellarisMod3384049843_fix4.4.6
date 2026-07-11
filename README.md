# Stellaris 4.4.6 失落帝国平衡Mod — 修复说明

## 修复概述

本分支修复了原mod在Stellaris 4.4版本中与蜂群失落帝国(Hive Fallen Empire)、合成女王危机、级联觉醒逻辑、物种权利等方面的兼容性问题和bug。

共修复 **16处bug**，涉及 **12个文件**。

---

## 修复清单

### 1. 蜂群失落帝国觉醒使用错误的觉醒事件
**文件**: `events/00_new_events_for_balanced_fallen_empire.txt` (事件 `balanced_fallen_empire_events.7`)

**问题**: `balanced_fallen_empire_events.7` 在强制觉醒时，无论帝国类型如何，始终调用普通帝国的觉醒事件 `fallen_empires_awakening.3`。这导致蜂群失落帝国觉醒时：
- 被错误套用了普通帝国的政体（`civic_revanchist_fervor` + `civic_ancient_caches_of_technology`）
- 无法吸收另外两个蜂群节点碎片
- 无法占据"最后的思绪"星系

**修复**: 添加了 `is_hive_empire` 分支判断，蜂群帝国走正确的 `awaken_strongest_fe_fragment` → `fallen_empires_awakening.10` 路径。

---

### 2. 蜂群碎片吸收触发全银河级联觉醒
**文件**: `events/00_new_events_for_balanced_fallen_empire.txt` (事件 `balanced_fallen_empire_events.8`)

**问题**: 蜂群觉醒时 `fallen_empires_awakening.10` 通过 `integrate_country_effect` 吸收其他节点碎片，碎片国家被摧毁后触发 `on_country_destroyed` → `balanced_fallen_empire_events.8` → 对所有剩余堕落帝国执行 `balanced_fallen_empire_events.7`，导致**所有堕落帝国同时觉醒**。

**修复**: 在 `.8` 的触发条件中添加 `NOT = { has_country_flag = got_assimialted }`。原版 `.10` 在吸收碎片前会在碎片上设置此标记，用于区分"内部吸收"和"被真正征服摧毁"。

---

### 3. 合成女王(Cetana)风暴异常 + 击败事件多处Bug
**文件**: `events/00_balanced_events_for_ciris_woman_king.txt`

#### 3a. 事件 `crisis.8042`（FE削弱事件）
Mod覆盖了原版的 `crisis.8042`，引入了三处与原版不一致的修改：
- 添加了 `fog_machine_auto_tracking = yes` —— 原版游戏中**不存在**此命令
- 步骤2删除了舰队摧毁代码（原版随机摧毁80%舰船）
- 步骤3使用 `kill_all_pop` 替代原版的 `kill_single_pop`

**修复**: 删除无效的 `fog_machine_auto_tracking`，恢复步骤2的舰队摧毁逻辑，步骤3对齐原版。

#### 3b. 事件 `crisis.23015`（合成女王击败事件）
Mod版本是旧版代码，缺少4.0+版本的更新内容：
- `has_planet_flag = synth_queen_bastille` — 原版使用 `has_carrier_flag`，标记类型不匹配导致**bastille星球清理永远不执行**
- 缺少 `remove_global_flag = galactic_crisis_recently_fired` — 导致**击败Cetana后无法触发第二个危机**
- 缺少 `remove_global_flag galactic_crisis_early_defeat_tracker_*` + `multiply_crisis_strength` — 导致**早期击败的危机强度缩放失效**

**修复**: 修正flag类型、补回危机冷却清除、补回早期击败强度缩放逻辑。

---

### 4. 级联觉醒重复执行天堂之战覆盖首个觉醒者
**文件**: `events/00_new_events_for_balanced_fallen_empire.txt` (事件 `balanced_fallen_empire_events.7`)

**问题**: 当任意失落帝国被摧毁时，`.8` 对**所有**剩余非机器失落帝国触发 `.7`。每个级联跟随者都会：
- 覆盖 `event_target:first_awakener`
- 重新执行天堂之战 `random_list`（40/20/40）
- 反复设置 `no_war_in_heaven` 全局标记

`no_war_in_heaven` 反复被设置导致 Cetana 天灾的"反无聊"权重加成（2倍）几乎必然触发，使合成女王极大概率被选为终末危机。

**修复**: 
- 用 `is_first_sleeper_awake` 国家标记追踪"真正的首个觉醒者"
- 只有首个觉醒者保存 `first_awakener` 并执行天堂之战
- 级联跟随者照常觉醒并通知玩家，但跳过天堂之战和 `first_awakener` 覆盖
- 添加 `is_country_type = fallen_empire` 守卫防止对已觉醒帝国重复执行觉醒

---

### 5. 失控机仆(Rogue Servitor)无法设置活体陈设
**文件**: 
- `common/species_rights/citizenship_types/01_balanced_citizenship_types_for_rebuild_computer.txt`
- `common/species_rights/living_standards/01_balanced_living_standards_for_rebuild_computer.txt`

**问题**: Mod覆盖了 `citizenship_organic_trophy`（活体陈设）和 `living_standard_organic_trophy` 的定义，在 `potential` 块中添加了 `has_synthetic_dawn = yes` 条件。此条件**在原版中不存在**，导致 `potential` 永远返回false，活体陈设公民权选项完全不出现。

**修复**: 删除无效的 `has_synthetic_dawn = yes`，补回原版中缺失的 `is_disconnected_drone_pop = no` 检查。

---

### 6. 机器失落帝国事件错误引用排外FE的Event Target
**文件**: `events/00_new_events_for_rebuilt_computer.txt` (事件 `rebuilt_computer_events.21`)

**问题**: 机器失落帝国领袖创建事件中，存在从排外失落帝国事件复制粘贴的残留代码：
- `event_target:xenophobe_fallen_empire_leader_country` — 此target仅在排外FE事件中设置，机器FE上下文中可能不存在
- `name_xenophobe_fallen` 和 `[xenophobe_fallen_empire.GetName]` — 引用了错误的FE命名

**修复**: 将target替换为正确的 `event_target:yunfan_leader_country`，名称key替换为 `name_machine_fallen`，变量引用替换为 `[machine_fallen_empire2.GetName]`。

---

### 7. 女性领袖分支复制粘贴Bug
**文件**: `events/00_new_events_for_rebuilt_computer.txt` (事件 `rebuilt_computer_events.21`)

**问题**: 玩家选择女性领袖 (`yunfan_is_woman`) 后，`create_leader` 块中的 `gender` 参数仍然写了 `male`。无论玩家选男还是选女，都生成男性领袖。

**修复**: 将女性分支的 `gender` 改为 `female`。

---

### 8. 考古遗址阶段数不匹配
**文件**: `common/archaeological_site_types/new_site_type_for_rebuilt_computer.txt`

**问题**: `tech_fallen_ship_site` 考古遗址声明 `stages = 3`，但实际定义了**4个**stage块。第4个stage（难度8，触发 `tech_fallen_events.6`）永远无法被玩家触发，造成最终奖励无法获取。

**修复**: 将 `stages` 改为 `4`。

---

### 9. 传奇提督领袖特质引用无效（4.x 领袖系统不兼容）
**文件**: `events/00_new_events_for_fallen_empire_ship_leader.txt` (事件 `fallen_leader_events.2`)

**问题**: Tubonerian 传奇提督使用了基于旧版 Stellaris（4.0前）三级特质系统的引用：
- `leader_trait_aggressive_2` — 4.x中只有 `leader_trait_aggressive`（无后缀升级版）
- `leader_trait_wrecker_3` — 4.x中最高 `leader_trait_wrecker_2`（Tier 2 老兵级）
- `leader_trait_commanding_presence_3` — 4.x中最高 `leader_trait_commanding_presence_2`（Tier 2 老兵级）
- `leader_trait_resilient_2` — 4.x中只有 `leader_trait_resilient`（无后缀升级版）

4.0 领袖系统重做后，普通特质不再有级别后缀，老兵特质最多到 `_2`。

**修复**: 
- `_aggressive_2` → `leader_trait_aggressive`
- `_wrecker_3` → `leader_trait_wrecker_2`（需 Paragon DLC + subclass_commander_admiral，领袖已有）
- `_commanding_presence_3` → `leader_trait_commanding_presence_2`
- `_resilient_2` → `leader_trait_resilient`

---

### 10. 机器FE活体陈设建筑使用不存在的触发条件
**文件**: `common/buildings/00_ap_nano_buildings.txt`

**问题**: `building_new_organic_paradise_for_fallen_machine` 的 `ai_resource_production` 的 `trigger` 块中使用了 `has_unemployed_pop_of_category = bio_trophy`，该条件在原版中不存在，CWTools 报错。

Mod 作者意图：当星球上有失业的活体陈设人口时，AI 应重视在此建造该建筑。

**修复**: 替换为最接近的合法触发条件 `num_unemployed > 0`（星球上有失业人口）。

---

### 11. 地块定义语法错误——无效的modifier包裹和defense_armies_add
**文件**: `common/deposits/21_cpu_plan_b.txt`

**问题**: 
- `d_cpu_b_flag` 的 `triggered_planet_modifier` 中岗位键 `job_researcher_add`/`job_brain_drone_add`/`job_calculator_add` 被错误包裹在 `modifier = { }` 内；其中 `job_brain_drone_add` 和 `job_calculator_add` 在原版 deposits 中根本不存在
- `d_using_nano_machine` 的 `planet_modifier` 中有 `defense_armies_add = 3`，该键在原版任何 deposit 中都不存在

**修复**: 
- 删除 `modifier = { }` 包裹，岗位键直接放在 `triggered_planet_modifier` 下
- 蜂群/机器统一用 `is_gestalt = yes` + `job_calculator_engineer_add`（参照原版 deposits 的具体工种键名体系）
- 删除无效的 `defense_armies_add`

---

### 12. 建筑岗位键名在triggered_planet_modifier中无效
**文件**: `common/buildings/14_balanced_fallen_buildings.txt`

**问题**: `building_master_archive` 和 `building_master_archive2` 的蜂群/机器 `triggered_planet_modifier` 块中使用了 `job_brain_drone_add` 和 `job_calculator_add`，这两个键在原版 buildings 中也不存在。且被错误包裹在 `modifier = { }` 内——原版蜂群/机器块中岗位键是直接放在 `triggered_planet_modifier` 下的。

**修复**: 参照原版 `jobs/researchers_add` inline script 的模式：
- 蜂群块：删除 `modifier` 包裹，改为 `job_calculator_physicist_add` / `job_calculator_biologist_add` / `job_calculator_engineer_add` 直接放置
- 机器块：同上
- 常规帝国块保持原样（`job_head_researcher_add` 是合法键，`modifier` 包裹对常规帝国块是正确语法）

---

### 13. 岗位防御陆军语法错误
**文件**: `common/pop_jobs/zzz_fallen_jobs.txt`

**问题**: `fe_guardian_bot` 岗位中 `planet_modifier` 块放入 `planet_crime_add = -25`，而防御陆军 `pop_defense_armies_add = 3` 却错误放在独立的 `pop_modifier` 块中。原版没有在岗位中使用 `pop_modifier` 或 `pop_defense_armies_add` 的模式，CWTools 报告这是非法的。

**修复**: 将 `pop_defense_armies_add` 替换为原版中合法的 `planet_defense_armies_add`，合并到已有的 `planet_modifier` 块中，生成干净的 `planet_modifier = { planet_crime_add = -25; planet_defense_armies_add = 3 }`。

---

### 14. 贸易修正使用已废弃的键名
**文件**: `common/static_modifiers/00_new_static_modifiers_for_balanced_fallen_empire.txt`

**问题**: `stay_home_caused_virus` 静态修正使用了 `trade_value_mult = -0.80`。在 Stellaris 4.x 贸易系统重做后，此键已被 `planet_jobs_trade_produces_mult` 取代，CWTools 报告 `trade_value_mult` 不存在。

**修复**: `trade_value_mult = -0.80` → `planet_jobs_trade_produces_mult = -0.80`，对齐4.x贸易系统。

---

### 15. 武器模板使用不存在的属性键
**文件**: `common/component_templates/00_new_weapons_xl_for_balanced_section_templates.txt`

**问题**: 4个泰坦武器模板使用了 `use_ship_kill_target = no`。此键在原版中不存在——正确的键名是 `use_ship_main_target`（参照原版 `PERDITION_BEAM_TITAN` 的定义），含义是泰坦武器不跟随舰船主目标而是自行选择最有价值目标。

**修复**: 所有4处 `use_ship_kill_target` → `use_ship_main_target`。

---

### 16. PlayAsGray兼容代码引用不存在的舰船设计
**文件**:
- `common/fallen_empires/HM_fallen_empire.txt`
- `events/HM_fallen_empires.txt`

**问题**: L星门大帝(L-Gate Grander)机器失落帝国初始化时，`PlayAsGray_Opened` 兼容块引用了三个不存在的全局舰船设计：
- `NAME_Nanite_Mothership_AG_1`
- `NAME_Nanite_Mothership_AG_carrier_1`
- `NAME_Nanite_Mothership_AG_carrier_2`

这三个名称在原版和mod自身中都从未定义。原版纳米飞船仅定义了 `NAME_Nanite_Mothership`、`NAME_Gray_Warship`、`NAME_Nanite_Interdictor`。

事件文件中还调用了 `create_AG_graygoo_tempest_fleet = yes`，该 scripted effect 在原版和mod中同样不存在。

**修复**: 删除所有无效的 PlayAsGray 兼容块引用（4处 `add_global_ship_design` + 1处 `create_AG_graygoo_tempest_fleet`）。已经通过 `NAME_Nanite_Mothership` + `NAME_Nanite_Interdictor` 获得了纳米船设计能力。

## 涉及文件

| 文件 | 修改次数 |
|---|---|
| `events/00_new_events_for_balanced_fallen_empire.txt` | 3 |
| `events/00_balanced_events_for_ciris_woman_king.txt` | 2 |
| `events/00_new_events_for_rebuilt_computer.txt` | 2 |
| `events/00_new_events_for_fallen_empire_ship_leader.txt` | 1 |
| `events/HM_fallen_empires.txt` | 1 |
| `common/buildings/00_ap_nano_buildings.txt` | 1 |
| `common/buildings/14_balanced_fallen_buildings.txt` | 1 |
| `common/deposits/21_cpu_plan_b.txt` | 1 |
| `common/pop_jobs/zzz_fallen_jobs.txt` | 1 |
| `common/static_modifiers/00_new_static_modifiers_for_balanced_fallen_empire.txt` | 1 |
| `common/component_templates/00_new_weapons_xl_for_balanced_section_templates.txt` | 1 |
| `common/fallen_empires/HM_fallen_empire.txt` | 1 |
| `common/species_rights/citizenship_types/01_balanced_citizenship_types_for_rebuild_computer.txt` | 1 |
| `common/species_rights/living_standards/01_balanced_living_standards_for_rebuild_computer.txt` | 1 |
| `common/archaeological_site_types/new_site_type_for_rebuilt_computer.txt` | 1 |

---

## 已知但不影响功能的问题

- `fallen_empire_crsis_level` 变量名有拼写错误（少了一个'i'）——内部一致，不影响功能
- `is_fallen_empire_machine` 覆盖了原版定义，添加了排除 LGate Grander 的条件——有意为之
- `on_game_start` / `on_yearly_pulse_country` 有空的事件块——无害死代码
