# Stellaris 4.4.x 失落帝国平衡Mod — 修复说明

## 修复概述

本分支修复了原mod在Stellaris 4.4版本中与蜂群失落帝国(Hive Fallen Empire)、合成女王危机、级联觉醒逻辑、物种权利等方面的兼容性问题和bug。

共修复 **8处bug**，涉及 **5个文件**。

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

### 3. 合成女王(Cetana)风暴异常居中显示
**文件**: `events/00_balanced_events_for_ciris_woman_king.txt` (事件 `crisis.8042`)

**问题**: Mod覆盖了原版的 `crisis.8042`（合成女王对失落帝国的削弱事件），引入了三处与原版不一致的修改：
- 添加了 `fog_machine_auto_tracking = yes` —— 原版游戏中**不存在**此命令，可能导致纳米风暴视觉异常
- 步骤2删除了舰队摧毁代码（原版随机摧毁80%舰船），导致失落帝国舰队不受影响
- 步骤3使用 `kill_all_pop` 替代原版的 `kill_single_pop`

**修复**: 删除无效的 `fog_machine_auto_tracking`，恢复步骤2的舰队摧毁逻辑，步骤3对齐原版行为。

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

## 涉及文件

| 文件 | 修改次数 |
|---|---|
| `events/00_new_events_for_balanced_fallen_empire.txt` | 3 |
| `events/00_balanced_events_for_ciris_woman_king.txt` | 1 |
| `events/00_new_events_for_rebuilt_computer.txt` | 2 |
| `common/species_rights/citizenship_types/01_balanced_citizenship_types_for_rebuild_computer.txt` | 1 |
| `common/species_rights/living_standards/01_balanced_living_standards_for_rebuild_computer.txt` | 1 |
| `common/archaeological_site_types/new_site_type_for_rebuilt_computer.txt` | 1 |

---

## 已知但不影响功能的问题

- `PlayAsGray_Opened` 全局标记在原版中不存在——来自其他mod的兼容标记，没有该mod时对应代码永远不会执行（死代码，不报错）
- `fallen_empire_crsis_level` 变量名有拼写错误（少了一个'i'）——内部一致，不影响功能
- `is_fallen_empire_machine` 覆盖了原版定义，添加了排除 LGate Grander 的条件——有意为之
- `on_game_start` / `on_yearly_pulse_country` 有空的事件块——无害死代码
