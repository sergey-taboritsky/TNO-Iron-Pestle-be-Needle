# TNO 杂项法律（Assorted Laws）机制分析 + 叙利亚落地提案

> **澄清"TNO_Assorted_Laws"指什么**：用户说的 `TNO_Assorted_Laws` 实为 **`common/ideas/TNO_Assorted_Laws.txt`**——所有杂项法律的"空 idea 容器"定义文件。其中 **IBR = 伊比利亚（Iberia）**，**GNG = 广东（Guangdong）**。
>
> 本文为技术调研 + 设定落地提案，**不写任何代码**，仅梳理机制与给出 Syria 可做成杂项法律的清单。

---

## 一、机制四层结构

### 1. 定义层 —— `common/ideas/TNO_Assorted_Laws.txt`
- 每个法律是一个**空的 `hidden_ideas` token**，例如：
  - `IBR_iberian_labour_charter = {}`
  - `GNG_basic_law = {}`
- 文件按地区分组：`GENERAL / REICH / OFN / LATIN AMERICA / MEDITERRANEAN / MIDDLE EAST / AFRICA / RUSSIA / ASIA`。
- **关键发现：MIDDLE EAST 区块当前是空的**（`hidden_ideas = { }` 第 117–119 行）。即原版 TNO 的中东 tag（SYR/EGY/YEM）目前**没有任何杂项法律**——需要新增。
- token 本身是**纯显示容器**（empty idea，无内嵌修正）。真正的玩法修正由"授予该法的事件/焦点"另行施加（见第六节提醒）。

### 2. 持有层 —— 国家变量数组 `TNO_assorted_laws`
- 每个国家有一个数组变量 `TNO_assorted_laws`，存放它当前生效的法律 token。
- 开局填充由 scripted effect **`misc_law_array_startup`**（`common/scripted_effects/TNO_Laws_scripted_effects.txt`）按 tag 赋值：
  ```
  misc_law_array_startup = {
      if = { limit = { tag = GNG }
          add_to_array = { TNO_assorted_laws = token:GNG_basic_law }
          add_to_array = { TNO_assorted_laws = token:GNG_census_ordinance_law }
          add_to_array = { TNO_assorted_laws = token:GNG_japan_sofa_law }
          add_to_array = { TNO_assorted_laws = token:GNG_labor_standards_law }
      }
      else_if = { limit = { tag = IBR }
          add_to_array = { TNO_assorted_laws = token:IBR_iberian_labour_charter }
          add_to_array = { TNO_assorted_laws = token:IBR_non_united_colonies }
          add_to_array = { TNO_assorted_laws = token:IBR_prestige_foreign_policy }
          add_to_array = { TNO_assorted_laws = token:IBR_organic_bases_of_the_creation_of_the_union }
      }
      ...
      else = { # 全球通用 8 条（污染/安全/童工/难民/少数族裔/种族融合/女性/LGBT） }
  }
  ```
- 运行中增减：
  - `TNO_add_assorted_law` = `add_to_array = { TNO_assorted_laws = temp_law }` + tooltip
  - `TNO_remove_assorted_law` = `remove_from_array = { TNO_assorted_laws = temp_law }` + tooltip

### 3. 布局层（可选）—— `TNO_assorted_laws_display`
- 一个 5 元素数组，手动指定网格排布：`[ Law1, Law3, Law2, Law50, Law51 ]` 表示
  ```
  [ Law1 ] [ Law3 ]
  [ Law2 ] [ Law50 ]
  [ Law51 ]
  ```
- 元素含义：数字 1–10 = 预留的通用旧法 ID；具体数字 = 特定法 ID；`special` = 特殊区块；`0` = 空位。
- **注意**：`TNO_assorted_laws_display` 是"可选高级布局"。实际渲染时 scripted_gui 的 `assorted_laws_calculator_visible` 直接遍历 `TNO_assorted_laws` 数组（不依赖这个 5 元组），所以大多数 tag 不设这个变量也能正常显示。

### 4. 渲染层（GUI）
链路：
1. **`common/scripted_guis/TNO_Politics.txt`** → 块 `TNO_SG_SD_category_tno_assorted_laws`（"杂项法律"分类，由 `tno_opened_laws_category = 100` 识别）：
   - `triggers`：`assorted_laws_calculator_visible` 遍历 `TNO_assorted_laws` 数组，把每个值 `v` 写入临时数组 `TNO_assorted_laws_display_temp`，末尾补两个 `0`（占位）。
   - `dynamic_lists`：`TNO_Politics_Assorted_Law_Grid` 绑定 `array = TNO_assorted_laws_display_temp`，`entry_container = "[get_TNO_assorted_law_entry]"`，`value = temp_law`。
2. **`common/scripted_localisation/TNO_SG_Politics_Laws_Misc.txt`** → `get_TNO_assorted_law_entry` 选择条目模板：
   - `temp_law = 0` → `TNO_empty_assorted_law_entry`（空位）
   - 命中 8 个通用法之一 → `TNO_assorted_law_entry`
   - 其余（国家专属法）→ `TNO_country_specific_assorted_law_entry`
   - `get_TNO_assorted_law_name` / `_desc` 用 `[?temp_law.GetTokenKey]` 解析出本地化键与 `_desc` 键。
3. **`interface/TNO_SG_Politics_Misc.gui`** → 定义窗口与条目外观：
   - 窗口 `TNO_sd_category_tno_assorted_laws_window`，标题 `tno_assorted_laws`。
   - 网格 `gridBoxType = TNO_Politics_Assorted_Law_Grid`：**2 列**，`slotsize = 330×94`。
   - 条目模板 `containerWindowType = TNO_assorted_law_entry`：
     - 矩形底图图标 `GFX_assorted_laws_rectangle`（**通用矩形，非每条独立图标**）
     - 名称 `instantTextboxType name`：`text = "[?temp_law.GetTokenLocalizedKey]"`（token → 本地化名）
     - 状态 `instantTextboxType status`：`text = [get_law_category_namespace]`（通用法显示子类状态，如污染等级；国家专属法此处为空）
   - 空位模板 `TNO_empty_assorted_law_entry`。
4. **图标/贴图**：矩形为 `GFX_assorted_laws_rectangle`（在 `.gfx` 中定义）。**每条法律没有独立专属图标**，显示靠名字文本区分。

### 5. 本地化
- 通用 8 法在 `localisation/english/TNO_law_ideas_l_english.yml`。
- **国家专属法的 `TOKEN:` 与 `TOKEN_desc:` 写在各国自己的 loc 文件里**（已验证：GNG 在 `TNO_Guangdong_l_english.yml`，IBR 在 `TNO_Iberia_l_english.yml`）。
- → Syria 的法律名/描述应加进现有的 `localisation/simp_chinese/MiddleEast_SYR_l_simp_chinese.yml`。

---

## 二、IBR / GNG 实例对照

| Tag | 开局杂项法律（token） | 含义（OTL/设定参照） |
|---|---|---|
| **GNG 广东** | `GNG_basic_law` | 基本法 |
| | `GNG_census_ordinance_law` | 人口普查条例 |
| | `GNG_japan_sofa_law` | 对日（驻军）地位协定 |
| | `GNG_labor_standards_law` | 劳动标准法 |
| **IBR 伊比利亚** | `IBR_iberian_labour_charter` | 伊比利亚劳工宪章 |
| | `IBR_non_united_colonies` | 非统一殖民地 |
| | `IBR_prestige_foreign_policy` | 威望外交 |
| | `IBR_organic_bases_of_the_creation_of_the_union` | 联合创制的基本法基 |

> 注意 GNG/IBR 各自还有**大量**后续法（如 GNG 有 30+ 条 `GNG_morita_plan_law`、`GNG_icac_law` 等；IBR 有 `IBR_organic_law_of_the_state`、`IBR_iberian_constitution` 等），由事件/焦点在游戏中途 `TNO_add_assorted_law` 动态加入。

---

## 三、叙利亚可以做成杂项的法律（设定落地提案）

> 以设计稿（土耳其宗主 / 人民党商帮掌控 / 1950 宪法连续有效 / 1963 贬值危机 / 土地改革为分支开关）为权威。下列为**提案**，待用户拍板。token 命名沿用 `<TAG>_<slug>_law` 约定。

### A. 开局存量法（由 `misc_law_array_startup` 的 `tag = SYR` 分支写入）
| 提案 token | 名称 | 设定依据 |
|---|---|---|
| `SYR_1950_constitution_law` | 1950 年宪法 | 设定里由 1949–50 制宪会议（土方背书、人民党主导）产出，连续有效至开局 |
| `SYR_electoral_law_law` | 选举法（人民党优势） | "旋转门"轮替执政的制度地基 |
| `SYR_turkish_customs_union_law` | 对土关税同盟 | 取代 OTL 伊拉克轴，亲土政策条约基础（经济圈） |
| `SYR_pipeline_profit_sharing_law` | 油管过境利润分成法 | 正合阿勒颇商帮利益（OTL 1955–56 IPC/Tapline 50% 分成参照） |

### B. 开局后由危机/分支动态加入（事件/焦点调用 `TNO_add_assorted_law`）
| 提案 token | 名称 | 触发 |
|---|---|---|
| `SYR_land_reform_law` | 土地改革法 | **核心冲突开关**——商帮议会封杀；改革派/军方/复兴党胜则出（参照 Decree-Law-88 类） |
| `SYR_currency_stabilization_law` | 货币稳定应急法 | 1963 叙镑贬值危机 |
| `SYR_press_association_law` | 新闻与结社管制法 | 土方收紧 / 镇压时 |
| `SYR_state_of_emergency_law` | 紧急状态法 | 骚乱 / 游击时 |
| `SYR_nationalization_law` | 国有化法令 | 德经危机 / 石油危机触发辩论 |

### C. 实现所需（跨 DSR 同名覆盖，不回写原版）
1. `common/ideas/TNO_Assorted_Laws.txt` → 在 **MIDDLE EAST** 空 `hidden_ideas` 区块加入上述 `SYR_xxx_law = {}`。
2. `common/scripted_effects/TNO_Laws_scripted_effects.txt` → `misc_law_array_startup` 加 `else_if = { limit = { tag = SYR } ... }` 分支。
3. `localisation/simp_chinese/MiddleEast_SYR_l_simp_chinese.yml` → 加 `SYR_xxx_law:` 与 `SYR_xxx_law_desc:` 键。
4. **GUI 部分原版已通用**（窗口/网格/条目模板对所有 tag 生效），DSR **无需改 interface**，除非要自定义图标（目前所有法共用 `GFX_assorted_laws_rectangle` 矩形）。

---

## 四、提醒
- 这些 token 在 `TNO_Assorted_Laws.txt` 中是**空 hidden_idea（纯显示容器）**；真正修正由授予该法的事件/焦点另行施加（如 `add_ideas = SYR_land_reform_law` 同时设动态 modifier 变量）。要在面板"显示一条法"，只需把 token 加进 `TNO_assorted_laws` 数组并配好本地化。
- 通用 8 法（污染/安全/…）对 Syria 未必适用——可在 `misc_law_array_startup` 的 SYR 分支只加专属法，不走 `else`（全球通用）分支。
- 与议会框架文档（`TNO_SYR_Parliament_Framework.md`）联动：土地改革法（B 表）正是该文"开局后核心冲突开关"在 GUI 上的落点。
