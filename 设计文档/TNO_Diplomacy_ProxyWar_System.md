# TNO 外交博弈 / 代理人战争系统 · 机制知识汇总

> **文档用途**：本文件系统整理 TNO 原版「外交博弈 / 代理人战争（proxy war）」机制的完整链路，作为 DSR 子模组复刻 / 扩展中东代理人战争的参考底稿。
>
> **路径约定**
> - 原版 TNO（**只读参考，绝不可写**）：`D:\SteamLibrary\steamapps\workshop\content\394360\2438003901`
> - DSR 子模组（**唯一写入目标**）：`C:\Users\苗灏\Documents\GitHub\DSR`（模组名 `TNO:DSR[dev]`，v1.6.0）
> - 所有引用行号为探索时的快照，可能随原版更新漂移，以「文件名 + 关键字段」定位为准。

---

## 0. 机制分层总览

TNO 的「外交博弈」本质是对 HOI4 原生 `power_balance`（权力平衡）机制的改造封装。整条链路由三层 + 若干脚本原语构成：

```
【触发层】on_state_control_changed / forpol mission        ← 战场实时结果 / 玩家主动选择
      │
【面板层】TNO_add_fopo_conflict_tab（Global Conflicts 面板） ← 用通用原语开/关冲突 tab
      ├─ desc / icon 走 defined_text 脚本本地化动态分发
      └─ 入场动作：add_equipment_to_stockpile + TNO_increase_volunteer_cap_1
      │
【结算层】冲突结算事件（如 sho.4）
      ├─ Cold_War_GUI_ManipulateVictory  → 冷战胜利榜（可选展示层）
      ├─ USA_proxy_conclusion_effect     → 超级大国本国内政结算
      ├─ econ_join_sphere_* / *_won flag → 战区结局（标准干预胜利）
      └─ TNO_fopo_remove_<war>_tab        → 关面板（BoP cleanup）
```

**底层通用底座是 BoP（权力平衡）系统**，所有战区都复用同一套 `AddTab → set_power_balance → decision_category` 流程，而非硬编码。

---

## 1. 文件地图

| 层级 | 文件（相对原版根） | 职责 |
|---|---|---|
| 触发 | `common/on_actions/TNO_on_state_control_changed_actions.txt` | `on_state_control_changed` 下各战区的实时省份归属判定（含 MALAYAN WAR 块） |
| 触发/内容 | `common/decisions/TNO_forpol_USA.txt` | 美国外交博弈决议 / mission（`USA_BoP_category` 分类） |
| 事件 | `events/TNO_Malaya.txt` | 马来亚战争事件链（`sho.3` 美国下场、`sho.4` 胜利结算、`sho.16/17` 分支等） |
| BoP 注册 | `common/bop/TNO_BoP_Defines.txt` | 每个大国 `BoP_Decisions_<TAG> = { decision_category = … }` |
| BoP 驱动 | `common/scripted_effects/TNO_BoP_GUI_scripted_effects.txt` | `TNO_BoP_AddTab / RemoveTab` → `meta_effect { set_power_balance / remove_power_balance }` |
| 面板原语 | `common/scripted_effects/TNO_fopo_effects.txt` | `TNO_add_fopo_conflict_tab` / `TNO_remove_fopo_conflict_tab` 及各国 wrapper |
| 结算原语 | `common/scripted_effects/TNO_proxy_war_conclusion_effects.txt` | `USA_proxy_conclusion_effect` 等各国内政结算 |
| 脚本本地化 | `common/scripted_localisation/TNO_fopo_scripted_loc.txt` | `Get_fopo_desc` / `Get_fopo_icon`（defined_text） |
| 冷战积分 | `common/scripted_effects/TNO_Cold_War_scripted_effects.txt` | `AddVictory` / `ManipulateVictory` / `CalculateDisplayArrays` / `Score_Calculation` |
| 冷战码表 | `common/scripted_localisation/TNO_Cold_War_scripted_localisation.txt` | `Cold_War_GUI_GetVictoryOutcome`（Outcome 0–5 文案） |
| 动态修正 | `common/dynamic_modifiers/TNO_generic_proxy_war_dynamic_modifiers.txt` | `PROXYNATIONAL_CP_GAIN`（代理人战争指令力增益） |

---

## 2. 机制入口：`on_state_control_changed`

**文件**：`common/on_actions/TNO_on_state_control_changed_actions.txt`
**区块**：`### MALAYAN WAR ###`（约第 154–218 行），挂在 `on_state_control_changed` 下。

**Scope 约定（TNO 专用）**：
- `FROM` = 失去控制的国家（旧主人）
- `ROOT` = 取得新控制的国家（新主人）
- `FROM.FROM` = 控制权发生变更的**那个 state**

**三条子规则**（以马来亚为例）：

| 规则 | 触发条件 | 效果 |
|---|---|---|
| A | state **806** 从 SHO 易主到 MLY | `USA.activate_mission = USA_hold_the_line` |
| B | 反向：806 从 MLY 回到 SHO | `USA.remove_mission = USA_hold_the_line` |
| C | SHO 任何 state 易主到 MLY，且 MLY `controls_province` 命中 **10297 / 4424 / 12299** 之一 | `USA.country_event = sho.3`（延迟 1 小时） |

> **关键纠正（常见误解）**：on_action 里**没有**「MLY 占任意港口 → 美国介入」的宽松判定，而是硬编码三个省份 ID（疑似槟城/马六甲/新加坡类核心港）。真正的「美国下场」事件是 **`sho.3`**，由 Rule C 在满足条件后 1 小时推给 USA。
> on_action 本身**只做判定与推事件**，不含 `add_to_faction` / 志愿军 / 送装备 / 宣战等代理战争本体效果——那些在下游事件与决策里。

---

## 3. 介入层：forpol mission 与 BoP 决策组

**`USA_hold_the_line` 是 mission，不是 decision**：
- 定义于 `TNO_forpol_USA.txt`，挂在 `USA_BoP_category` 分类下。
- 内部是 `days_mission_timeout = 60` + `timeout_effect`（超时 → `USA.country_event = sho.16`），**没有** `complete_effect` / `available` 那套标准决议字段。
- `visible` 条件：`TNO_fopo_malaya_selected = yes` 且 `MLY.controls_state = 806`。
- `allowed = { original_tag = USA }`。
- on_action 的 Rule A/B 只是在**开关这个 mission**，不是在执行业已硬编码的效果。

**BoP 三层结构（通用底座）**：

```
TNO_BoP_Defines.txt          ← 注册层
   BoP_Decisions_USA = { decision_category = USA_BoP_category }
        │
TNO_BoP_GUI_scripted_effects.txt   ← 驱动层
   TNO_BoP_AddTab → meta_effect { set_power_balance = { id = BoP_Decisions_[TAG] } }
   TNO_BoP_RemoveTab → meta_effect { remove_power_balance = { id = BoP_Decisions_[TAG] } }
        │
TNO_forpol_USA.txt (USA_BoP_category)   ← 内容层（具体杠杆决议 / mission）
```

- **`meta_effect` 是 BoP 接入引擎的「唯一通道」**：`decision_category` 只是普通决议分类，真正让它变成「权力平衡条上可操作的势力」的是 `meta_effect` 调用引擎的 `set_power_balance { id = BoP_Decisions_[TAG] }`。
- 共约 15 个大国（GER/JAP/BRA/USA/TUR/…）走同一套流程。
- **命名红线**：`TNO_BoP_Defines.txt` 首行注明 `These MUST follow the naming standard... otherwise it WILL NOT WORK!!`——`BoP_Decisions_<TAG>` define 名、`[TAG]` token、`decision_category` 三者命名必须严格对齐，否则 `set_power_balance` **静默失效**（不报错，只是杠杆不出现）。

---

## 4. 开战 / 收尾事件：`sho.3` 与 `sho.4`

**`sho.3`（美国下场，第 101 行附近）**——这才是「美国加入博弈」的真动作：
- `add_equipment_to_stockpile { … producer = USA }`（convoy / 步兵 / 反坦 / 炮 / 支援）
- `TNO_increase_volunteer_cap_1`（志愿军上限 +1）
- `set_temp_variable { target = MLY }`
- `is_triggered_only` + `fire_only_once`

**`sho.4`（美国赢了的结算，第 157 行附近）**——由 `sho.16` / `sho.17` 等战事推进分支汇流而来（`country_event = { id = sho.4 days = 2 }`）：
- `set_country_flag = conclusion_victory`
- `USA_proxy_conclusion_effect`（美国内政结算，见 §6）
- `add_political_power = 75`
- `Cold_War_GUI_ManipulateVictory`（冷战胜利榜，见 §9）
- `set_global_flag = malay_usa_won`
- `MLY.econ_join_sphere_USA`（马来亚加入美势力经济圈）
- `USA` 解锁 `TNO_CIA_MLY_Operations_Can_Begin`（后续 CIA 行动）

---

## 5. `sho.4.a` 选项逐项职责划分

| 片段 | 性质 | 落点 |
|---|---|---|
| `conclusion_weeks=4` / `conclusion_bonus=0.01` + `USA_proxy_conclusion_effect` | 美国内政结算 | 选民倾向 + 执政党支持率（§6） |
| `Victory_ID=25` / `Victory_Outcome=4` / `Victory_Points=75` + `Cold_War_GUI_ManipulateVictory` | 冷战胜利 GUI 点数 | §9 |
| `set_global_flag = malay_usa_won` | 战区结局标记 | 标准干预胜利 |
| `every_country { … news_event = WORLD.18112 }` | 国际新闻 | 只对 `HasSouthAsiaInterest` 的非 AI 玩家推送 |
| `MLY = { econ_join_sphere_USA = yes }` | 经济圈收编 | 标准干预胜利 |
| `NZL = { NZL_World_Stage_var / NZ_Election_var / NZ_Skeleton_Effect }` | 新西兰剧情/选举 | 与主线机制无关（本 submod 可忽略） |
| `USA = { set_country_flag = TNO_CIA_MLY_Operations_Can_Begin; TNO_fopo_remove_malaya_civil_war_tab = yes }` | 解锁 + 面板清理 | §7 |
| `custom_effect_tooltip = TNO_CIA_MLY_unlocked_tt` | 提示文案 | — |

---

## 6. 美国内政结算：`USA_proxy_conclusion_effect`（通用、可复用）

**定义**：`common/scripted_effects/TNO_proxy_war_conclusion_effects.txt`（约第 558 行起）。

**函数头注释原文要点**：
> *"Since the election code has changed, this effect has been simplified to simply being a compilation of existing USA election effects... Set a number of weeks as `conclusion_weeks`. Set a bonus value for popularity change as `conclusion_bonus`... Set flag `conclusion_victory`, otherwise it will apply negative effects. You should fire this effect within the USA scope."*

**实际行为**：
- 读 `conclusion_weeks`（如 4）与 `conclusion_bonus`（如 0.01）；
- 按 `has_country_flag = conclusion_victory`（sho.4 immediate 里已置位）走**胜利分支**：
  - 给美国各选民集团（black / hispanic / minority / nativist / urban / rural / union）的外交政策倾向加 `conclusion_weeks` 周计数器（`Policy_Shift_*_foreign_policy_lib_Weeks`）；
  - 按当前执政党（`USA.presidents_party` 1–6）给对应党派的 `USA_BONUS_*_val` 加 `conclusion_bonus` 的 popularity 加成；
- 失败分支（`else`，无 `conclusion_victory` flag）走 `_con` 后缀变量（负面）且给**所有**党派加 penalty。

> **重要纠偏**：它**不是**「结束代理冲突的机制动作」，而是「美国大选 / 选民倾向系统的一次性结算」。冲突在机制层面的结束由 `TNO_fopo_remove_malaya_civil_war_tab` + `*_won` flag + `econ_join_sphere_*` 负责。该 effect 通用可复用（SAW 等其它战区也调它）。

---

## 7. 冲突面板原语：`TNO_add_fopo_conflict_tab` / `TNO_remove_fopo_conflict_tab`

**定义**：`common/scripted_effects/TNO_fopo_effects.txt`（add 约第 1–75 行，remove 约第 50–75 行）。

**`TNO_add_fopo_conflict_tab`（实定义要点）**：
```puppet
TNO_add_fopo_conflict_tab = {
    custom_effect_tooltip = TNO_add_fopo_conflict_tab_tt
    if = {
        limit = { NOT = { is_in_array = { TNO_active_conflicts = conflict_id } } }
        add_to_array = { TNO_active_conflicts = conflict_id }
        set_variable = { selected_conflict_id = conflict_id }
        clear_array = world_conflict_guis
        add_to_array = { world_conflict_guis = selected_conflict_id }
        set_variable = { TNO_fopo_selected_front = 1 }
    }
    set_temp_variable = { alert_id = token:Alert_Proxy_War_Active }
    add_TNO_alert_unless_exists = yes
    hidden_effect = {
        if = { limit = { check_variable = { NATIONAL_cp_gain_proxy = 0 } }
            add_dynamic_modifier = { modifier = PROXYNATIONAL_CP_GAIN }   # 仅首次挂一次
        }
        add_to_variable = { NATIONAL_cp_gain_proxy = 0.07 }   # ← 每次调用都 +0.07（在 if 外）
    }
}
```

**`TNO_remove_fopo_conflict_tab`（要点）**：
- 仅在 `TNO_active_conflicts^num = 0`（最后一个冲突也清掉）时才 `remove_TNO_alert`——alert 是全局共享的，不是每冲突一个；
- `add_to_variable = { NATIONAL_cp_gain_proxy = -0.07 }`（同样在 if 外，每次调用 -0.07，减到 0 才摘 `PROXYNATIONAL_CP_GAIN`）。

**wrapper 模板（复刻时照抄）**：
```puppet
TNO_fopo_add_<yourwar>_tab = {
    set_temp_variable = { conflict_id = token:GC_<YourWar> }
    TNO_add_fopo_conflict_tab = yes
    # …其它逻辑（如置位介入 flag）
}
TNO_fopo_remove_<yourwar>_tab = {
    set_temp_variable = { conflict_id = token:GC_<YourWar> }
    TNO_remove_fopo_conflict_tab = yes
}
```

**⚠️ 复刻时必踩的三个坑**：
1. **CP 增益按调用次数累加，不是按唯一冲突数**：`add_to_variable = { … = 0.07 }` 在 if **外面**，每次调用都 +0.07；remove 的 -0.07 也在 if 外。add / remove **必须严格 1:1 配对**，且 add 应用 flag / 状态守卫确保**只调一次**，否则永久泄漏 +0.07 指令力增益且 modifier 永不摘除。
2. **`world_conflict_guis` 是「当前正在浏览的那一页 id」（单元素数组）**，`TNO_active_conflicts` 才是全量冲突列表；`selected_conflict_id` 是「当前选中分页」。复用时不变量、不碰这些内部状态，只传 `conflict_id`。
3. **`GC_<YourWar>` 这个 token 必须在本地化里给页面标题键**（如 `GC_MalayaCW: "Malayan Emergency"`），否则标签页显示 raw key。

**提示文案（无需改）**：`TNO_add/remove_fopo_conflict_tab_tt` 只是「新增 / 移除一个 Global Conflicts 页面」的提示，颜色码为原版标准。

---

## 8. 脚本本地化：`Get_fopo_desc` / `Get_fopo_icon`

**文件**：`common/scripted_localisation/TNO_fopo_scripted_loc.txt`
**机制**：两者都是 HOI4 `defined_text` 块。GUI 调用 `[Get_fopo_desc]` / `[Get_fopo_icon]` 时，按顺序匹配 `text = { trigger localization_key }`，返回**第一个命中 trigger** 的 `localization_key`。

- **`Get_fopo_desc`（约第 356 行）**——按 `(冲突, 国家, 可选分页/flag)` 分发：
  ```puppet
  text = { trigger = { check_variable = { selected_conflict_id = token:GC_AngolaCW } tag = USA }
           localization_key = USA_angola_civil_war_desc }
  ```
  你的 `USA_angola_civil_war_desc` 就是在这里被选中的 key；真正的文本在 `TNO_fopo_l_*.yml`。触发器组合 = `selected_conflict_id = token:GC_<冲突>` + `tag = <国家>`（+ 可选 `has_global_flag` / `TNO_fopo_selected_front = N`）。

- **`Get_fopo_icon`（约第 1035 行）**——同样的机制，但返回的 `localization_key` 实际是 **GFX 精灵键**：
  ```puppet
  text = { trigger = { check_variable = { conflict_id = token:GC_MalayanCW } tag = USA }
           localization_key = "GFX_fopo_malaya_civil_war_icon_us" }
  ```
  `GFX_fopo_*_icon_*` 在 `.gfx` 文件里定义（非纯文本）。

> **⚠️ 隐含差异（复刻必看）**：`Get_fopo_desc` 匹配 **`selected_conflict_id`**，而 `Get_fopo_icon` 匹配 **`conflict_id`**——这是两个不同的变量（见 §7）。加新冲突时，desc 块抄 `selected_conflict_id` 模式、icon 块抄 `conflict_id` 模式，**不要混**。
>
> 具体介绍文案没有固定格式，本地化给出 `GC_<YourWar>` 标题键 + 各 `TAG_<war>_desc` 键即可。

---

## 9. 冷战积分系统

**文件**：`common/scripted_effects/TNO_Cold_War_scripted_effects.txt`
**本质**：每个超级大国持有三条并行全局数组 `TNO_Cold_War_GUI_Victory_Entry / _Outcome / _Points`，**下标 == 胜利 ID**（阿留申危机=1，德国内战=14，马来亚=25…）。

| 原语 | 行号（约） | 作用 | 参数 |
|---|---|---|---|
| `Cold_War_GUI_AddVictory` | 654 | **注册**新胜利：三条数组 `resize` 到 `Victory_ID+1`，在 `^Victory_ID` 下标写 1 / Outcome / Points | `Victory_ID`（必填）、Outcome / Points（可选） |
| `Cold_War_GUI_ManipulateVictory` | 694 | **增或改**：若 `Entry^ID = 0`（未注册）→ 自动转调 `AddVictory`；否则更新该下标 Outcome + Points | `Victory_ID` + `Victory_Outcome` + `Victory_Points`（**三者都必填，否则 ERROR**） |
| `Cold_War_GUI_CalculateDisplayArrays` | 731 | 重建 `*_Display` 展示数组（只纳入 `Entry^i = 1` 的活跃项），喂 GUI 网格 | 无 |
| `Cold_War_GUI_Score_Calculation` | 紧随 | 重算总分 | 无 |

**`ManipulateVictory` 三重守卫**：
1. `is_superpower = no` → ERROR；
2. `Victory_ID` / `Victory_Outcome` / `Victory_Points` 任一未 set → ERROR；
3. 依赖的这三个变量**必须在调用作用域内先 set 成变量**。

**标准调用姿势（即 sho.4 写法）**：
```puppet
USA = {
    set_temp_variable = { Victory_ID = 25 }
    set_temp_variable = { Victory_Outcome = 4 }
    set_temp_variable = { Victory_Points = 75 }
    Cold_War_GUI_ManipulateVictory = yes
}
```

**对 submod 的结论**：一个能跑的代理战争（tab + desc + icon + 送装备 + 志愿军）**完全不需要**碰冷战积分系统——它只是「把冲突胜利折算进超级大国总分」的可选展示层。若想让新中东冲突出现在冷战榜上，才需：① 选一个未占用 Victory_ID（原版已用到 ≥25，取 26+ 或确认空闲）；② 给该 ID 补显示名（见 §10 遗留项）；③ 在胜利结算事件里于获胜大国作用域下 set 三变量 + 调 `ManipulateVictory`（Outcome / Points 数值照抄原版某胜利事件，勿臆造码值）。

---

## 10. `Victory_Outcome` 码表（7 态）

**文件**：`common/scripted_localisation/TNO_Cold_War_scripted_localisation.txt` → `Cold_War_GUI_GetVictoryOutcome`（约第 193 行，`defined_text`）。
**GUI 用法**：`^TNO_Cold_War_GUI_Victories_i` 循环读 `_Outcome_Display` 数组取文案。

| 码值 | localization_key | 中文 | 语义 |
|---|---|---|---|
| 0 | `TNO_Cold_War_Victory_Outcome_Ongoing` | 进行中 | 冲突进行中 |
| 1 | `…_Total_Defeat` | 完败 | 彻底失败 |
| 2 | `…_Defeat` | 失败 | 失败 |
| 3 | `…_Stalemate` | 平局 | 僵持 / 平手 |
| 4 | `…_Victory` | 胜利 | 获胜 |
| 5 | `…_Total_Victory` | 完胜 | 压倒性胜利 |
| 其他 / 未初始化 | `…_Error` | 错误 | 兜底（**永远不该落到这**） |

- 最后那个无 `trigger` 的 `text = { localization_key = "…_Error" }` 是「其他情况」兜底。写 6 / 7 或留空就会被 GUI 显示成「错误」。
- 实操：开战事件里 `ManipulateVictory` 写 `Outcome = 0` 让它显示「进行中」，结算时改成 `4/5`（胜）或 `1/2`（败）或 `3`（平），Points 改终值。全程只用 0–5。
- 这 7 个 key 原版已定义，中文环境直接继承英文或在 simp_chinese 覆盖均可，无需新建。

---

## 11. Submod 复刻落地清单（中东代理人战争）

1. **定义冲突 token**：`token:GC_<YourWar>`，并在本地化 yml 给 `GC_<YourWar>` 写页面标题键。
2. **开面板 wrapper**：写 `TNO_fopo_add_<yourwar>_tab`（塞 `conflict_id` + `TNO_add_fopo_conflict_tab = yes`），**用 flag / 状态守卫确保只调用一次**。
3. **关面板 wrapper**：镜像 `TNO_fopo_remove_<yourwar>_tab`（塞 `conflict_id` + `TNO_remove_fopo_conflict_tab = yes`），与 add **严格 1:1**。
4. **desc / icon**：在 `Get_fopo_desc` 加 `text = { trigger { selected_conflict_id = token:GC_<YourWar> tag = <TAG> } localization_key = <TAG>_<war>_desc }`；在 `Get_fopo_icon` 加 `text = { trigger { conflict_id = token:GC_<YourWar> tag = <TAG> } localization_key = GFX_<yourwar>_icon_<tag> }`；并分别在本地化 yml 定义 desc 键、在 `.gfx` 定义精灵键。
5. **入场动作**（参照 `sho.3`）：`add_equipment_to_stockpile { producer = 大国 }` + `TNO_increase_volunteer_cap_1` + `set_temp_variable { target = 代理人 }`。
6. **结算事件**：照 `sho.4` 拆分——`conclusion_victory` flag + `USA_proxy_conclusion_effect`（如需大国内政结算）+ `econ_join_sphere_*` + `set_global_flag *_won` + 关面板 wrapper。
7. **（可选）冷战榜**：选空闲 Victory_ID，补显示名，结算事件于获胜大国作用域下 set 三变量 + `Cold_War_GUI_ManipulateVictory`。
8. **原语 / 动态修正 / alert** 全部沿用原版，一行不改；只加自己的 wrapper、`defined_text` 分发块、本地化键、GFX 键。

**核心红线回顾**：
- 绝不写原版 TNO 路径；只在 DSR 同名覆盖 / 新增。
- BoP define 命名三要素严格对齐，否则静默失效。
- add / remove 面板 1:1 配对 + flag 守卫防 CP 增益泄漏。
- `desc` 用 `selected_conflict_id`、`icon` 用 `conflict_id`，不可混。
- `Victory_Outcome` 只用 0–5。

---

## 12. 待查 / 遗留项（动手前最后一步）

1. **各 Victory_ID 的显示名登记表 + `TNO_Cold_War_GUI_Victory_Entry` 数组的声明位置**：确认新 ID 不与某原版冲突，并给它补标题键（建议 grep `TNO_Cold_War_GUI_Victory_Entry` 声明处 + `CW_Victory` 类本地化键）。
2. **`TNO_fopo_malaya_selected` 的置位点**：确认「玩家点哪个 forpol 决议 → 触发 `TNO_BoP_AddTab` → 马来亚 tab 出现 → `hold_the_line` 可见」的完整闭环（复刻「玩家主动选择介入」入口时必需）。
3. **Global Conflicts 这个 decision category 如何用 `TNO_active_conflicts` 数组生成标签页**：即 `scripted_guis` / `TNO_fopo_decisions` 分类——标签页真正被渲染出来的 GUI 层。

---

*整理自多轮对原版 TNO 的只读逆向探索，聚焦 DSR 子模组复刻中东代理人战争所需的最小可复用知识。所有原版路径仅供定位参考，写入一律落在 DSR 仓库。*
