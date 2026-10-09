# 冷战积分榜 UI 实现方案

## 一、仓库调研结论

mod 中已存在一套完整的「自定义按钮 + 弹窗 + 动态国家列表 + 变量排序」范式，**冷战积分榜可完全复刻 GDP 排行榜的成熟结构**，风险低：

- **按钮触发**：scripted_gui 通过 `parent_window_token = politics_tab` 在政治界面挂一个图标按钮，点击 `set_country_flag`，主窗口以该 flag 控制 `visible`，关闭按钮清 flag。
  - 参照：[parliament_gui.txt](file:///c:/Users/ZhenJiuZhe/Documents/Paradox%20Interactive/Hearts%20of%20Iron%20IV/mod/D-R-1946/common/scripted_guis/parliament_gui.txt#L51-L63)
- **国家积分排序**：scripted_effect 清空全局数组 → `every_country` 填入 → `get_sorted_scored_countries` + 自定义 scorer 排序 → `for_each_scope_loop` 给每个国家写入名次变量。
  - 参照：[00_gdp_ranking.txt](file:///c:/Users/ZhenJiuZhe/Documents/Paradox%20Interactive/Hearts%20of%20Iron%20IV/mod/D-R-1946/common/scripted_effects/00_gdp_ranking.txt)、[adorable_heart_GDP_rank_scorer.txt](file:///c:/Users/ZhenJiuZhe/Documents/Paradox%20Interactive/Hearts%20of%20Iron%20IV/mod/D-R-1946/common/scorers/country/adorable_heart_GDP_rank_scorer.txt)
- **列表渲染**：GUI 用 `gridboxtype` + entry 模板；scripted_gui 用 `dynamic_lists`（`change_scope = yes`）绑定全局数组；每行显示 `[?THIS.cw_ranking]`、国旗 `[This.GetFlag]`、`[?THIS.GetName]`、`[?cw_score]`。
  - 参照：[DR_econmic_window.gui](file:///c:/Users/ZhenJiuZhe/Documents/Paradox%20Interactive/Hearts%20of%20Iron%20IV/mod/D-R-1946/interface/DR_econmic_window.gui#L213-L263)、[GTMX_gui.txt](file:///c:/Users/ZhenJiuZhe/Documents/Paradox%20Interactive/Hearts%20of%20Iron%20IV/mod/D-R-1946/common/scripted_guis/GTMX_gui.txt#L4-L34)
- **数值原语（已验证可用）**：`num_of_military_factories`、`num_of_civilian_factories`、`num_of_naval_factories`、`num_of_factories`、`state_population_k`、`every_owned_state`、`every_subject`。
  - 参照：[00_money_system.txt](file:///c:/Users/ZhenJiuZhe/Documents/Paradox%20Interactive/Hearts%20of%20Iron%20IV/mod/D-R-1946/common/scripted_effects/00_money_system.txt)
- **刷新时机**：`on_startup` + `on_monthly` 脉冲，参照 [eco.txt](file:///c:/Users/ZhenJiuZhe/Documents/Paradox%20Interactive/Hearts%20of%20Iron%20IV/mod/D-R-1946/common/on_actions/eco.txt#L10-L62)。开窗口时也会立即重算一次，保证数据即时。

## 二、功能设计

### 界面布局
1. **入口按钮**：政治界面（politics_tab）左下角新增一个积分榜图标（与议会、图书馆按钮同区，避免重叠，初定 `x=20 y=435`，进游戏微调）。
2. **主窗口**：620×720 可拖动窗口，标题「冷战积分榜」，右上角关闭按钮（ESC 也可关）。
3. **阵营总览条**：标题下方四个小格，实时显示四大阵营总积分：民主阵营 / 共产阵营 / 法西斯阵营 / 中立阵营（按执政党 `has_government` 划分）。
4. **国家排行榜（滚动）**：表头「名次 / 国旗 / 国家 / 阵营 / 积分」，按积分降序动态列出全部国家。
5. **本国信息栏（底部固定）**：本国国旗 + 国名 +「第 X 名 · Y 分」，让玩家无需翻列表。

### 积分公式（五维综合国力，质量优先）
每月为每个国家计算 `cw_score`。针对初版「人口/工厂线性碾压」的问题，已重配平为五维度，削弱人口与民工、新增现役军力/科技/核武/阵营影响：

| 维度 | 项目 | 权重 |
|------|------|------|
| A 军事工业 | 军工厂 / 船坞 / 民用工厂 | ×6 / ×5 / ×2 |
| B 现役军力 | 陆军营总数 / 海军（航母×2、战列战巡×1.5、重巡×1、轻巡×0.7、驱逐×0.3、潜艇×0.15）/ 部署飞机 | ÷10 / 加权求和 / ÷50 |
| C 科技核武 | 已研究科技 / 科研槽 / 拥核 | ×2 / ×20 / +150 |
| D 人口潜力 | 总人口（千） | ÷1500（大幅衰减） |
| E 国际影响 | 核心州 / 傀儡国 / 阵营领袖 | ×1 / ×5 / +50 |

> 典型结果：美国≈苏联 > 日本≈中国，体量大但科技军力落后的国家不会无脑登顶。数值原语均为官方标准 value trigger（num_battalions、num_ships_with_type@…、num_deployed_planes、num_researched_technologies、num_research_slots、has_nuclear_bomb），权重集中写在 scripted_effect 顶部，后续随时可调。

### 阵营判定
用 `defined_text`（scripted_localisation）按 `has_government = democratic/communism/fascism/neutrality` 返回「民主阵营/共产阵营/法西斯阵营/中立阵营」，列表每行与总览条共用。

## 三、新增/修改文件

| 文件 | 操作 | 内容 |
|------|------|------|
| `common/scripted_effects/DR_coldwar_scoreboard_effects.txt` | 新建 | `coldwar_calc_scores`（算分）+ `coldwar_generate_rankings`（建数组、阵营汇总、scorer 排序、写名次） |
| `common/scorers/country/DR_coldwar_score_scorer.txt` | 新建 | `coldwar_score_scorer`，按 `cw_score` 排序 |
| `common/scripted_guis/DR_coldwar_scoreboard.txt` | 新建 | 入口按钮 + 主窗口 visible 控制、`dynamic_lists` 绑定、国旗属性、打开/关闭 effects |
| `interface/DR_coldwar_scoreboard.gui` | 新建 | 按钮容器、主窗口、阵营总览条、表头、gridbox、entry 模板、本国信息栏 |
| `interface/DR_coldwar_scoreboard.gfx` | 新建 | 注册按钮图标 `GFX_DR_coldwar_scoreboard_button`（图片素材后补，占位 png 路径） |
| `common/on_actions/DR_coldwar_on_actions.txt` | 新建 | `on_startup`、`on_monthly` 各重算一次 |
| `common/scripted_localisation/DR_Faction_Desc.txt` | 追加 | `GetColdWarCamp` 四阵营文本 |
| `localisation/simp_chinese/DR_new_l_simp_chinese.yml` | 追加 | 标题、表头、按钮提示、阵营名等全部中文键 |

## 四、实施步骤（按依赖顺序）

1. 建 scorer 文件 `coldwar_score_scorer.txt`。
2. 建 scripted_effect：算分 → 四个阵营全局总分变量 → 全局国家数组 → scorer 排序 → 写 `cw_ranking`。
3. 建 on_actions，挂载 startup/monthly 脉冲。
4. 在 `DR_Faction_Desc.txt` 追加 `GetColdWarCamp` 阵营判定文本。
5. 建 `coldwar_scoreboard.gui`（窗口 + 列表 + 本国栏）与 `.gfx`（图标注册）。
6. 建 `coldwar_scoreboard.txt` 注册 scripted_gui：按钮挂 politics_tab、窗口 flag 开关、`dynamic_lists` 绑定 `global.cw_countries`、打开时立即重算。
7. 追加中文本地化（遵循现有 `:0` 格式与字体 `dr_font_21`/`dr_font_16`）。
8. 进游戏验证并微调按钮坐标/窗口尺寸。

## 五、依赖与注意事项

- 列表 entry 内文本变量必须用 `[?THIS.xxx]`（`change_scope = yes` 时 scope 切到条目国家）；本国栏用 `[?cw_ranking]`、`[?cw_score]`。
- 全局数组/变量统一加 `cw_` / `global.cw_` 前缀，避免与 GDP 系统（`econ_`、`indu_`）冲突。
- 窗口 visible 用独立 flag `coldwar_scoreboard_open`，关闭时 clear。
- 图标 png 素材如暂时没有，先引用现有 `GFX_economy_button` 保证界面可用，之后替换 .gfx 路径即可，无需改逻辑。
- 字体、背景框（`GFX_tiled_window2_1b_border`）、关闭按钮（`GFX_closebutton`）、滚动条（`right_vertical_slider`）全部复用 mod 现有资源，保持风格统一。

## 六、验证方式

- 启动游戏无报错（重点看 scorer 与 scripted_effect 语法）。
- 政治界面出现积分榜按钮，点击开窗、再点关闭/ESC 关窗正常。
- 列表按积分降序排列，名次连续，国旗/国名/阵营/分数显示正确。
- 阵营总览条四格数值与列表成员之和一致。
- 本国信息栏名次、分数与列表中本国所在行一致；跨月后数值随工厂/人口变化刷新。

## 七、风险与应对

- **风险 1：每月对全部国家算分轻微卡顿**。计算仅遍历工厂数/州/傀儡，量级很小，与现有每月 GDP 统计同构，可忽略。
- **风险 2：全局阵营总分变量累加方式兼容性**。将使用 GDP 系统同款 `global.xxx` 前缀写法；若累加不生效，降级方案为阵营总分也由 scorer 排序后的数组在玩家开窗时临时汇总。
- **风险 3：按钮坐标与现有按钮重叠**。初定坐标留足间距，进游戏后按实际截图微调。
