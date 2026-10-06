# 任务页签与受阻原因（BuildingChoresPanel）参考

> 来源：TaskBlockers 模组（任务受阻提示）的完整调研与实现。游戏版本 U59-744825-SCRPN；行号指反编译源码 Assembly-CSharp.decompiled.cs，仅作路标（版本可能漂移，类名/id 优先）。
> 适用：改「任务」页签显示、做"为什么没人来"类提示、修改此类 mod 的文案。

## 机制（先读这个）

- 建筑面板三个页签：状态 / 任务 / 性质。任务页签 = `BuildingChoresPanel`（:429215，TargetPanel 子类），每帧 `RefreshDetails()`。
- 数据：`GlobalChoreProvider.Instance.choreWorldMap[worldId]` 与 `fetchMap[worldId]`，过滤 `chore.gameObject == 选中目标`。
- 每行由 `BuildingChoresPanelDupeRow.Init`（:429495-429512）渲染：
  - `SUCCESS_ROW "{Duplicant} -- {Rank}"` → "亚伯 -- #2"（`RANK_FORMAT` = "#{0}"）。完全通过、或仅输给 `IsMoreSatisfyingLate`（"低优先级"）时 `IsPotentialSuccess()=true`。
  - `FAILURE_ROW "{Duplicant} -- {Reason}"` → "亚伯 -- 无法到达"。Reason = `chore.GetPreconditions()[failedPreconditionId].condition.description`。
- **行数据来自 `ChoreConsumer.GetLastPreconditionSnapshot()`：上次求值快照，平时不重算** → 任务实际到不了、行里却仍显示 #N（误导玩家的根源）；拉警报或改优先级触发重算后才更新。
- chore group ≠ chore type：拆解任务（Deconstruct）属于 Build（建造）组，所以任务行前缀是"建造X"。

## "无法到达"判定（提示类 mod 的核心）

- 四个移动类前置条件共用文案键 `CAN_MOVE_TO`（"无法到达"）：`CanMoveTo` / `CanMoveToCell` / `CanMoveToDynamicCell` / `CanMoveToDynamicCellUntilBegun`（:85162-85275）。实现都是 `ChoreConsumer.GetNavigationCost(...)`（:95641、:95664）。
- **WorkChore 构造函数恒挂 `CanMoveTo`**（data = workable，IApproachable）→ 拆解/建造/挖掘/维修等全部工作类任务天然覆盖。
- 边界：Fetch 类卡在 `CanPickup`（文案"无法拾取"，另一条键）；清扫无人机是 `IsPermitted`（"未被允许"）。做泛化提示时按实际前置条件分档。
- 可靠做法（TaskBlockers 模式）：不读快照，按前置条件 **id** 对每个工人做新鲜求值（节流 ~0.5s）。**按 id 匹配，不要按文案匹配**——简中里 IsChattable（CanChat）的文案恰好也是"无法到达"，纯文案匹配会误伤。
- 无法判定的项按"能到"处理（保守），只在有把握时才提示；没有工人时不提示（防空殖民地误报）。

## 关键 API（均 public，可跨程序集直调）

| 用途 | API |
| --- | --- |
| 任务清单 | `GlobalChoreProvider.Instance.choreWorldMap` / `.fetchMap`（`Dictionary<int, List<Chore/FetchChore>>`） |
| 前置条件 | `chore.GetPreconditions()` → `List<PreconditionInstance>`（public struct：`condition` + `data`）；用 `condition.id` / `.description` |
| 执行中 | `chore.driver`（非 null = 有人正在做）、`chore.isNull` |
| 寻路 | `ChoreConsumer.GetNavigationCost(IApproachable, out int)` / `(int cell, out int)`（已处理 Navigator 与搬运臂） |
| 工人清单 | `Components.LiveMinionIdentities` / `Components.LiveRobotsIdentities` → `.gameObject.GetComponent<ChoreConsumer>()` |
| 面板目标 | `TargetPanel.selectedTarget` 是 **protected**——补丁里用 `AccessTools.Field(typeof(TargetPanel), "selectedTarget")` 读 |

## 文案与本地化

- 受阻原因文案全部在官方简中文本 `OxygenNotIncluded_Data/StreamingAssets/strings/strings_preinstalled_zh_klei.po`，键 `STRINGS.DUPLICANTS.CHORES.PRECONDITIONS.*`（70 个：42 条挂在 ChorePreconditions 字段，其余为 FAILURE_ROW / SUCCESS_ROW / RANK_FORMAT / HEADER 等格式串）。
- 任务类型键：`STRINGS.DUPLICANTS.CHORES.<KEY>.NAME/.STATUS/.TOOLTIP`（ChoreTypes 共 143 个）。
- **全量对照表已生成，改文案/查原因先查表，不必再反编译**：`E:\MOD BY AI\缺氧\TaskBlockers\任务类型与受阻原因清单.md`（脚本 `tools/generate_task_stats.py` 可复现/重生成）。
- 泛化候选（缺技能 HAS_SKILL_PERK、无可运送物品 IS_FETCH_TARGET_AVAILABLE、未被允许 IS_PERMITTED、被优先级占用 IS_PREEMPTABLE / IS_MORE_SATISFYING）见表。

## 模板：TaskBlockers（任务页签摘要行 mod）

实例：`E:\MOD BY AI\缺氧\TaskBlockers\`。`dist/TaskBlockers/` 为可安装产物；`build.sh` 只产出 dist，**不写游戏目录**（安装由用户决定）。

结构：`TaskBlockers.csproj` + `mod.yaml` + `mod_info.yaml` + `build.sh` + `src/{TaskBlockersMod, TaskBlockersStrings, UnreachableProbe, BuildingChoresPanelPatches}.cs` + `translations/zh.po`。

要点：

- 入口 `UserMod2.OnLoad`：`Localization.RegisterForTranslation` → 自行加载 `translations/<语言代码>.po`（`OverloadStrings`；原版不自动扫描该目录）→ Harmony 自检日志（确认补丁已挂上，进 Player.log 便于排装）。
- UI 摘要行：**游戏标签 `LocText` 是 TextMeshProUGUI 的子类**。摘要行用 `TextMeshProUGUI`，font / fontSharedMaterial / 字号从同面板的 LocText 复制；**不要用 UnityEngine.UI.Text / Font**。
- 插入位置：找 `text == UI.DETAILTABS.BUILDING_CHORES.AVAILABLE_CHORES` 的 LocText，把摘要行放到同一父级、其 SiblingIndex 之前；加 LayoutElement 占行高（父级非布局组时靠顶部锚点兜底，位置需游戏内确认）。
- 判定节流 + 结果缓存；目标变化立即重算；补丁挂在 `OnPrefabInit`（建行）与 `RefreshDetails`（每帧刷新，内部节流）。

## 常见坑

- csproj 必须额外引用 `Unity.TextMeshPro` 和 `UnityEngine.TextRenderingModule`（`Font` 类型在后者）——StalledDiagnostics 模板原本没有这两项。
- `enableWordWrapping` 已过时 → `textWrappingMode = TextWrappingModes.NoWrap`。
- 修改此类 mod 文案：改 `src/*Strings.cs`（英文默认值）+ `translations/zh.po`（msgctxt 键路径 = 命名空间.类.字段）的 msgstr，`bash build.sh` 重打 dist。
- 原版 status item（ConstructionUnreachable / DigUnreachable / MopUnreachable 等）是头顶图标/状态页签用的**另一套机制**，与任务行原因不同源；拆除类没有 `DeconstructUnreachable`。
