---
name: unity-mcp-validator
version: "2.2.0"
description: Route Unity changes to the correct validation flow using Unity-MCP tools. Use when deciding, executing, and reporting reliable validation for gameplay logic, controller/state-machine changes, input and interaction, animation timing, scene or prefab wiring, UI/HUD changes, ProjectSettings or package changes, PlayMode-only regressions, or final pre-acceptance checks. Especially use after edits under Assets/Scripts, Assets/Scenes, Assets/Prefabs, ProjectSettings, Packages, or when a bug appears only after playing for a while.
dependencies:
  - Unity-MCP plugin (https://github.com/IvanMurzak/Unity-MCP)
---

# Unity MCP Validator
<!-- Unity MCP 验证器：基于 IvanMurzak/Unity-MCP 的验证路由器 -->

## Overview
<!-- 概述 -->

Use this skill to turn a Unity change or bug report into a concrete validation route instead of ad hoc "run some tests" behavior. This skill decides which layers must run, in what order, which evidence to collect, and what cannot be claimed when Unity-MCP is unavailable.

**基于 [Unity-MCP](https://github.com/IvanMurzak/Unity-MCP) 的验证路由器，提供：**
- 100+ MCP 工具覆盖编辑器和运行时
- 反射系统可查找/调用任何 C# 方法
- Roslyn 动态编译执行 C# 代码
- 运行时支持（可在编译后的游戏内使用）
- 测试运行（EditMode/PlayMode）
- 截图、日志、状态检查
- **v2.0 新增**：异步等待（wait-until-condition）、状态沙盒重置、UI DOM 树快照、测试后门 API、JSON 结构化输出

Prefer this skill whenever the task involves runtime behavior, scene wiring, prefab correctness, input issues, animation/state timing, or a "works for a while then breaks" bug.

## Core Contract
<!-- 核心契约：每次执行必须遵守 -->

Follow this contract on every run:

1. **Classify first** — 先分类变更类型，再选择工具
2. **Cheapest first** — 从最便宜的验证层开始
3. **Escalate when needed** — 只在变更类型需要时才升级到 PlayMode
4. **Simulate human testing** — **交互验证必须模拟输入操作（点击、滑动、按键），这是验证流程正确性的核心**
5. **Probe runtime bugs** — PlayMode bug 是探针任务，不只是单元测试
6. **Be honest about blockers** — Unity-MCP 不可用时明确说明，不要假装完成
7. **No premature acceptance** — 必需的验证层未执行时，不要标记"可接受"
8. **Wait for state to settle** — **交互操作后必须等待游戏状态稳定（使用 wait-until-condition），禁止写死 Thread.Sleep**
9. **Reset between tests** — **多个 PlayMode 验证用例之间必须重置状态（见 [state-reset.md](references/state-reset.md)）**
10. **Assert with data, not visuals** — **UI 验证优先使用 ui-hierarchy-snapshot 做数据断言，截图作为辅助证据**

## Quick Start

**For setup instructions, see [SETUP.md](SETUP.md).**

1. Complete setup checklist in SETUP.md
2. Run preflight checks below
3. Read [references/route-matrix.md](references/route-matrix.md) and classify the change.
4. Run the required validation layers for that class.
5. If the issue is PlayMode-only or flaky, read [references/runtime-probes.md](references/runtime-probes.md) and gather runtime evidence.
6. Report using [references/output-contract.md](references/output-contract.md).
7. **Output both Markdown (human) and JSON (machine) reports** — see [references/json-output-schema.md](references/json-output-schema.md).

If the task spans multiple categories, validate against the **strictest** route, not the cheapest one.

For complete validation examples including input simulation, see [EXAMPLES.md](EXAMPLES.md).

For issues encountered during validation, see [references/troubleshooting.md](references/troubleshooting.md).

## Installation

This skill requires the Unity-MCP plugin. Setup steps:

1. **Install Unity-MCP plugin** in your Unity project:
   ```bash
   npm install -g unity-mcp-cli
   unity-mcp-cli install-plugin ./YourUnityProject
   ```

2. **Configure MCP server** in Claude Code settings

3. **Add custom input tools** (optional) — copy files from `references/custom-tools-input.md`

4. **Add wait/snapshot/reset tools** (recommended) — copy files from:
   - `references/custom-tools-wait.md` — 异步等待工具
   - `references/ui-snapshot-tool.md` — UI DOM 树快照
   - `references/state-reset.md` — 状态重置工具

5. **Add test backdoor API** (optional) — see `references/test-backdoors.md`

6. **Create config file** (optional) — copy `validation-config.example.yaml` to project root

### Prerequisites Check

Before using, verify:

| Check | How to Verify | Action on Failure |
|-------|---------------|-------------------|
| Unity-MCP plugin installed | Check `Assets/Scripts/MCP/` exists | Install via unity-mcp-cli |
| Unity Editor open | `editor-application-get-state` | Launch Unity manually |
| MCP server connected | Tool calls succeed | Restart MCP server |
| Input System enabled | Project Settings → Player | Enable Input System Package |
| Wait tools installed | Call `wait-until-condition` | Copy from `custom-tools-wait.md` |
| UI snapshot installed | Call `ui-hierarchy-snapshot` | Copy from `ui-snapshot-tool.md` |

If any prerequisite fails, report the blocker and continue with available layers.

## Preflight Checks

Before executing validation, verify:

| Check | Unity-MCP Tool | Action on Failure |
|-------|-----------------|-------------------|
| Unity Editor is open | `editor-application-get-state` | Report blocker, halt |
| Project compiles | `script-execute` simple code | Fix compilation first |
| Target scene known | `scene-list-opened` | Ask user or infer |
| MCP connection works | Any tool call | Report MCP disconnected |
| State is clean | `reflection-method-call` check initial state | Run state reset |

If any check fails, report it explicitly and continue with available layers.

## Validation Layers
<!-- 四层验证体系：从便宜到昂贵，按需升级 -->

Use these layers as building blocks. Do not skip a required layer for the routed category.

### Layer 1: Static Review
<!-- 第一层：静态代码审查，成本最低 -->

Read the diff and inspect the relevant code paths. Focus on:

- architecture boundaries (MVC, MVP, ECS, etc.) — 架构边界
- phase boundaries and task scope — 阶段边界和任务范围
- model/view ownership — 模型/视图所有权
- input lock acquire and release timing — 输入锁获取释放时机
- animation callback completion paths — 动画回调完成路径
- null-sensitive prefab or inspector references — 空敏感的预制体/检视面板引用
- changes to ProjectSettings, packages, or input backends — 项目设置/包/输入后端的变更

### Layer 2: EditMode / Test Runner
<!-- 第二层：编辑模式测试，确定性逻辑验证 -->

Use this layer for deterministic gameplay logic, rules, and controller contracts.

**Unity-MCP 工具：**
- `tests-run` — 执行 Unity 测试（EditMode/PlayMode），支持过滤和详细结果

If no suitable test exists and the change is logic-heavy, add the smallest targeted test that proves the regression.

### Layer 3: Unity Editor / Wiring Validation
<!-- 第三层：编辑器状态检查，Scene/Prefab/引用验证 -->

Use this layer when scenes, prefabs, inspector references, UI objects, or ProjectSettings may be involved.

**Unity-MCP 工具：**
| 工具 | 用途 |
|------|------|
| `script-execute` | 编译执行 C# 代码（Roslyn） |
| `scene-get-data` | 获取场景根对象列表 |
| `scene-list-opened` | 列出当前打开的场景 |
| `gameobject-find` | 查找特定 GameObject |
| `gameobject-component-get` | 获取组件详细信息 |
| `object-get-data` | 获取 Unity 对象数据 |
| `assets-get-data` | 获取资源数据 |
| `console-get-logs` | 获取 Unity 日志 |
| `editor-selection-get` | 获取当前选择 |
| `ui-hierarchy-snapshot` | **v2.0** 抓取 UI DOM 树结构快照 |
| `ui-element-find` | **v2.0** 查找指定 UI 元素详细状态 |
| `ui-element-at-position` | **v2.0** 查找指定坐标的 UI 元素 |

This layer answers "is the scene/prefab/editor state actually assembled the way the code expects?"
<!-- 这层回答：场景/预制体/编辑器状态是否真的按代码期望的方式组装？ -->

### Layer 4: PlayMode Automation
<!-- 第四层：运行模式自动化，输入交互/动画/运行时bug验证 -->

Use this layer for input, interaction, animation/state transitions, UI button flows, and any runtime-only bug.

**Layer 4 包含三个子流程 + 前置/后置步骤：**

#### 4A-Pre: 状态重置（v2.0 新增）

**在每次 PlayMode 验证前重置游戏状态，确保测试隔离。**

详细策略见 [references/state-reset.md](references/state-reset.md)。

| 工具 | 用途 |
|------|------|
| `state-reset` | 重置游戏到干净状态（自定义工具） |
| `reflection-method-call` | 调用 TestHelper.ResetAll() |
| `script-execute` | 执行重置代码 |

**流程：**
```
state-reset strategy="auto"   → 重置状态
wait-until-condition condition="GameController.Instance != null" → 确认初始化完成
```

#### 4A: PlayMode 控制

| 工具 | 用途 |
|------|------|
| `editor-application-get-state` | 获取编辑器状态（PlayMode、暂停、编译） |
| `editor-application-set-state` | 控制编辑器状态（启动/停止/暂停 PlayMode） |

**流程：**
```
editor-application-set-state playMode=true → 进入 PlayMode
wait-until-condition condition="GameController.Instance != null" → 等待初始化
[执行验证操作]
editor-application-set-state playMode=false → 退出 PlayMode
```

#### 4B: 输入模拟（核心）

**输入模拟是 Layer 4 的核心部分，用于：**
- 点击按钮验证 UI 流程
- 滑动验证三消游戏消除逻辑
- 拖拽验证拖放功能
- 按键验证快捷键/游戏输入
- 回放录制序列验证复杂交互

| 工具 | 用途 | 典型场景 |
|------|------|----------|
| `simulate-click-ui` | 点击 UI 元素 | 按钮、菜单、开关 |
| `simulate-click-world` | 点击世界空间对象 | 三消格子、游戏对象 |
| `simulate-drag-world` | 拖拽操作 | 三消滑动消除、拖放 |
| `simulate-key-press` | 按键 | 快捷键、游戏控制 |
| `record-start/stop` | 录制输入 | 手动复现 bug |
| `replay-input` | 回放录制 | 确定性复现问题 |

**测试后门 API（v2.0 新增）：**

通过 `reflection-method-call` 调用项目的后门方法，极速构造边界场景。详见 [references/test-backdoors.md](references/test-backdoors.md)。

```bash
# 构造"即将触发连锁爆炸"的边界场景
reflection-method-call typeName="BoardTestHelper" methodName="SetupChainExplosion"

# 构造"资源极度匮乏"的边界场景
reflection-method-call typeName="EconomyTestHelper" methodName="SetupResourceScarce"

# 跳过教程直接测试核心逻辑
reflection-method-call typeName="TestHelper" methodName="SkipTutorial"
```

#### 4B-Post: 等待状态稳定（v2.0 新增）

**交互操作后，必须等待游戏状态稳定再截取结果。禁止写死 Thread.Sleep。**

详细工具文档见 [references/custom-tools-wait.md](references/custom-tools-wait.md)。

| 工具 | 用途 | 典型场景 |
|------|------|----------|
| `wait-until-condition` | 轮询 C# 条件直到为 true 或超时 | 等待消除动画完成、等待状态转换 |
| `wait-for-animation-state` | 等待 Animator 进入指定状态 | 等待攻击动画播放完毕 |
| `wait-for-frame-count` | 等待 N 帧后继续 | UI 布局重建后等一帧 |
| `wait-for-stable` | 等待条件满足 + 无新错误日志 | 复杂交互后确认状态稳定 |

**关键流程：**
```
simulate-drag-world startX=200 startY=300 endX=400 endY=300  → 模拟滑动
wait-until-condition condition="Board.Instance.IsAnimating == false" → 等待消除完成
screenshot-game-view → 此时截图结果才可靠
```

#### 4C: 证据收集与状态探针

| 工具 | 用途 |
|------|------|
| `screenshot-game-view` | 捕获游戏视图截图 |
| `screenshot-camera` | 从指定相机捕获截图 |
| `console-get-logs` | 获取运行时日志 |
| `reflection-method-call` | 调用任何 C# 方法检查状态 |
| `reflection-method-find` | 查找项目中的方法 |
| `script-execute` | 动态编译执行 C# 代码探针 |
| `ui-hierarchy-snapshot` | **v2.0** 抓取 UI DOM 树快照做数据断言 |
| `ui-element-find` | **v2.0** 查找指定 UI 元素状态 |

**v2.0 断言升级：截图 + UI 快照组合验证**

| 维度 | 截图验证 | UI 快照断言 |
|------|----------|------------|
| 分辨率依赖 | ❌ 是 | ✅ 否 |
| 帧时机 | ❌ 可能截到过渡帧 | ✅ 数据是即时的 |
| 大模型判断 | ❌ 容易误判 | ✅ 精确匹配 |
| 文本内容 | ❌ OCR 可能出错 | ✅ 100% 精确 |
| 交互状态 | ❌ 看不出 interactable | ✅ 直接可断言 |

**验证三消消除后状态（v2.0 完整流程）：**
```csharp
// reflection-method-call 或 script-execute
var board = FindObjectOfType<Board>();
return $"Gems: {board.GemCount}, Score: {board.Score}, Phase: {board.CurrentPhase}";
```

---

**Layer 4 完整流程示例（三消滑动验证 — v2.0 增强版）：**

```
1. [4A-Pre] state-reset strategy="auto" → 重置游戏状态
2. [4A-Pre] wait-until-condition condition="GameController.Instance != null" → 确认重置成功
3. [4A] editor-application-set-state playMode=true → 进入 PlayMode
4. [4A] wait-until-condition condition="GameController.Instance.CurrentPhase == Phase.Idle" → 等待初始化
5. [4C] screenshot-game-view → 基线截图
6. [4C] ui-hierarchy-snapshot format="json" → 基线 UI 快照
7. [4C] reflection-method-call → 基线状态快照（Score=0, GemCount=64）
8. [4B] simulate-drag-world startX=100 startY=200 endX=300 endY=200 → 模拟滑动
9. [4B-Post] wait-until-condition condition="Board.Instance.IsAnimating == false" timeoutSeconds=5 → 等待消除完成
10. [4C] screenshot-game-view → 结果截图
11. [4C] ui-hierarchy-snapshot format="json" → 结果 UI 快照（断言 ScoreText 文本）
12. [4C] reflection-method-call → 结果状态（Score, GemCount, Phase）
13. [4C] console-get-logs → 检查是否有错误
14. [4A] editor-application-set-state playMode=false → 退出 PlayMode
```

This layer is required for "I played a few times and then it stopped responding" bugs and any interaction-based validation.
<!-- 这层是必需的，针对交互验证和"玩了几次然后没反应了"这类bug -->

## Routing Workflow

### Step 1: Gather the change surface
<!-- 步骤1：收集变更范围 -->

Before running tools, identify:

- changed files — 改了哪些文件
- touched subsystems — 涉及哪些子系统
- whether Unity scene or prefab state is part of the behavior — 是否依赖场景/预制体状态
- whether the bug is deterministic, intermittent, or only visible after multiple interactions — bug是确定的、间歇的、还是多次交互后才出现

If the change is unclear, infer from file paths first. Do not stop for a question unless the target scene or feature area is genuinely ambiguous.

### Step 2: Classify the change
<!-- 步骤2：分类变更类型 -->

Use [references/route-matrix.md](references/route-matrix.md). Typical classes:

- model/rules — 模型/规则
- controller/state machine — 控制器/状态机
- input/interaction — 输入/交互
- animation/input lock — 动画/输入锁
- scene/prefab wiring — 场景/预制体连线
- UI/HUD — 界面
- project settings or package backend — 项目设置/包后端
- PlayMode runtime regression — 运行时回归
- pre-acceptance regression sweep — 预验收检查

### Step 3: Execute required layers
<!-- 步骤3：按顺序执行验证层 -->

Run the required layers in order. A typical order is:

1. static review — 静态审查
2. script-execute or tests-run — 编译或测试
3. scene/gameobject inspection — 场景/对象检查
4. **state reset (if PlayMode required)** — 状态重置（如需 PlayMode）
5. PlayMode automation and runtime probes — 运行模式自动化和运行时探针
6. **wait for state to settle after each interaction** — 交互后等待状态稳定

If a layer fails for environmental reasons, say so explicitly and continue with the remaining non-blocked layers. Do not silently downgrade the verdict.

### Step 4: Probe failures, not just symptoms
<!-- 步骤4：探针失败原因，不只是症状 -->

When PlayMode behavior is wrong, do not stop at "click did nothing." Use Unity-MCP reflection tools to gather:

- logs — 日志（`console-get-logs`）
- screenshots — 截图（`screenshot-game-view`）
- **UI DOM tree — UI DOM 树**（`ui-hierarchy-snapshot`）
- current controller/state-machine phase — 用 `reflection-method-call` 或 `script-execute` 检查
- input lock state — 输入锁状态
- selection state — 选择状态
- whether colliders, camera, and raycast path line up — 碰撞体/相机/射线检测是否对齐
- whether an animation callback never returned — 动画回调是否未返回
- **state snapshots at each phase** — 每个阶段的状态快照（for JSON report）

Use [references/runtime-probes.md](references/runtime-probes.md) for the exact probe ladder.

### Step 5: Report with a hard verdict
<!-- 步骤5：输出硬性结论 -->

Always state:

- what route was chosen — 选择了什么路由
- what layers actually ran — 实际运行了哪些层
- what evidence was collected — 收集了什么证据
- what remains blocked — 还有什么被阻塞
- whether the change is acceptable now, acceptable with known gaps, or not acceptable — 是否可接受

**v2.0 双轨输出：**
- Use [references/output-contract.md](references/output-contract.md) for human-readable Markdown report.
- Use [references/json-output-schema.md](references/json-output-schema.md) for machine-consumable JSON report.
- **Both formats should be produced for every validation run.**

## Unity-MCP Tool Reference
<!-- Unity-MCP 工具参考 -->

### Project & Assets

| 工具 | 用途 |
|------|------|
| `assets-find` | 搜索资源数据库 |
| `assets-get-data` | 获取资源数据（含所有可序列化字段） |
| `assets-modify` | 修改资源文件 |
| `assets-prefab-instantiate` | 在场景中实例化预制体 |
| `assets-prefab-open` | 打开预制体编辑模式 |
| `package-add/remove/list` | 包管理 |

### Scene & Hierarchy

| 工具 | 用途 |
|------|------|
| `scene-get-data` | 获取场景根对象 |
| `scene-open/unload/save` | 场景管理 |
| `scene-list-opened` | 列出已打开场景 |
| `gameobject-find` | 查找 GameObject |
| `gameobject-create/destroy/modify` | GameObject 操作 |
| `gameobject-component-add/get/modify` | 组件操作 |
| `screenshot-*` | 截图（游戏视图/场景视图/相机） |

### Scripting & Editor

| 工具 | 用途 |
|------|------|
| `script-execute` | 动态编译执行 C#（Roslyn）— **核心探针工具** |
| `script-read/update-or-create/delete` | 脚本文件操作 |
| `reflection-method-find` | 查找方法（含私有方法） |
| `reflection-method-call` | 调用任何 C# 方法 — **核心探针工具** |
| `console-get-logs` | 获取日志 |
| `tests-run` | 运行测试（EditMode/PlayMode） |
| `editor-application-get/set-state` | 编辑器状态控制 |

### v2.0 新增：异步等待工具
<!-- v2.0 新增工具：解决时序不同步问题 -->

| 工具 | 用途 | 详见 |
|------|------|------|
| `wait-until-condition` | 轮询 C# 条件直到为 true 或超时 | [custom-tools-wait.md](references/custom-tools-wait.md) |
| `wait-for-animation-state` | 等待 Animator 进入指定状态 | [custom-tools-wait.md](references/custom-tools-wait.md) |
| `wait-for-frame-count` | 等待 N 帧后继续 | [custom-tools-wait.md](references/custom-tools-wait.md) |
| `wait-for-stable` | 等待条件满足 + 无新错误 | [custom-tools-wait.md](references/custom-tools-wait.md) |

### v2.0 新增：UI DOM 树快照工具
<!-- v2.0 新增工具：解决截图断言盲区 -->

| 工具 | 用途 | 详见 |
|------|------|------|
| `ui-hierarchy-snapshot` | 抓取 Canvas UI 树结构快照 | [ui-snapshot-tool.md](references/ui-snapshot-tool.md) |
| `ui-element-at-position` | 查找指定坐标的 UI 元素 | [ui-snapshot-tool.md](references/ui-snapshot-tool.md) |
| `ui-element-find` | 按名称查找 UI 元素详细状态 | [ui-snapshot-tool.md](references/ui-snapshot-tool.md) |

### v2.0 新增：状态重置工具
<!-- v2.0 新增工具：消除测试间状态污染 -->

| 工具 | 用途 | 详见 |
|------|------|------|
| `state-reset` | 重置游戏到干净状态 | [state-reset.md](references/state-reset.md) |

### v2.0 新增：测试后门 API
<!-- v2.0 新增：通过 reflection-method-call 调用项目后门方法 -->

不是独立 MCP 工具，而是项目中的 C# 方法，通过 `reflection-method-call` 调用。详见 [test-backdoors.md](references/test-backdoors.md)。

```bash
# 常用后门方法
reflection-method-call typeName="TestHelper" methodName="ResetAll"
reflection-method-call typeName="BoardTestHelper" methodName="SetupChainExplosion"
reflection-method-call typeName="EconomyTestHelper" methodName="SetupResourceScarce"
reflection-method-call typeName="TestHelper" methodName="SkipTutorial"
reflection-method-call typeName="UITestHelper" methodName="ForceGameOver"
```

## Project-Specific Acceptance Focus
<!-- 项目特定验收重点 -->

This skill supports project-specific configuration via `validation-config.yaml`.

### Setting Up Configuration

1. Copy `validation-config.example.yaml` to your Unity project root
2. Rename to `validation-config.yaml`
3. Customize for your project:
   - **invariants**: Key behaviors that must not regress
   - **key_classes**: Core classes for runtime probing via reflection
   - **file_patterns**: Project-specific path patterns for routing hints
   - **custom_tools**: Whether input simulation tools are installed
   - **state_reset**: v2.0 — State reset strategy for test isolation
   - **test_backdoors**: v2.0 — Available test backdoor API methods

### Core Invariants to Guard

Define invariants your project must protect. Examples:

| Invariant | Check Method |
|-----------|--------------|
| Input locked during critical animations | `reflection-method-call` to check InputLocked |
| View never mutates model directly | Static review: look for Model writes in View |
| Controller always returns to idle | `reflection-method-call` to check CurrentPhase |
| Scene wiring matches script assumptions | `assets-get-data`, `gameobject-component-get` |
| State resets between test cases | `state-reset` + `wait-until-condition` |

### Key Classes to Probe

Configure these for runtime state inspection via reflection:

```yaml
key_classes:
  controller:
    class_name: "YourController"
    assembly: "Assembly-CSharp"
  state_machine:
    enum_name: "YourPhaseEnum"
  input_lock:
    class_name: "YourInputManager"
    property_name: "IsLocked"
```

### Custom Input Tools

If you've installed the custom input simulation tools from `references/custom-tools-input.md`:

```yaml
custom_tools:
  input_simulation_installed: true
  wait_tools_installed: true
  ui_snapshot_installed: true
  state_reset_installed: true
```

If no config file exists, the validator will use generic heuristics and prompt for clarification when needed.

## Runtime Bug Policy
<!-- 运行时bug策略 -->

When a bug appears only after repeated manual play:

1. **Reset state** — 用 `state-reset` 或 `reflection-method-call` 调用 ResetAll
2. Enter PlayMode with `editor-application-set-state` — 用 editor-application-set-state 进入 PlayMode
3. Wait for initialization — 用 `wait-until-condition` 等待初始化完成
4. Capture baseline screenshot + UI snapshot — 捕获基线截图 + UI 快照
5. Interact or replay scenario — 交互或回放场景
6. **Wait for state to settle** — 用 `wait-until-condition` 等待状态稳定
7. Capture logs, screenshot, UI snapshot, and runtime state when bug appears — bug出现时捕获完整证据
8. Use `reflection-method-call` or `script-execute` to inspect runtime state — 用反射工具检查运行时状态
9. Report the stuck state, not just the visible symptom — 报告卡住的状态，不只是可见症状
10. **Output both Markdown and JSON reports** — 输出 Markdown + JSON 双轨报告

If Unity-MCP is disconnected or Unity Editor is not open, explicitly report that the runtime route is incomplete.

## Runtime Probe Examples
<!-- 运行时探针示例 -->

### Check controller state
<!-- 检查控制器状态 -->

```csharp
// 使用 reflection-method-call 调用静态方法
// 或使用 script-execute 动态执行

var controller = GameObject.FindObjectOfType<MyGameController>();
var phase = controller.CurrentPhase;
var inputLocked = controller.InputLocked;
return $"Phase: {phase}, InputLocked: {inputLocked}";
```

### Check if animation callback fired
<!-- 检查动画回调是否触发 -->

```csharp
var animator = GameObject.Find("MyObject").GetComponent<Animator>();
var state = animator.GetCurrentAnimatorStateInfo(0);
return $"Animation: {state.nameHash}, Time: {state.normalizedTime}";
```

### Check UI state with DOM tree snapshot (v2.0)
<!-- 用 UI DOM 树快照检查 UI 状态 -->

```bash
# 抓取完整 UI 树
ui-hierarchy-snapshot format="json"

# 查找特定元素
ui-element-find name="ScoreText"
# 期望输出：type=Text, text="Score: 30", active=true

# 查找指定坐标下的元素
ui-element-at-position x=150 y=100
# 用于验证点击目标是否正确
```

## Do Not Do This
<!-- 禁止行为 -->

- Do not rely on EditMode tests alone for input or animation bugs. — 输入/动画bug不能只靠EditMode测试
- Do not claim prefab or scene setup is correct without inspecting with Unity-MCP tools. — 没检查编辑器状态不能声称预制体/场景正确
- Do not mark a feature "verified" because it compiles. — 编译通过不等于验证通过
- Do not replace required PlayMode validation with manual guesswork when Unity-MCP can probe. — Unity-MCP能探针时不要用手工猜测替代
- Do not hide blocked Unity-MCP steps inside a generic "manual verification needed" sentence. Name the exact missing step. — 不要用"需要手工验证"掩盖阻塞步骤，要明确说缺什么
- **Do not use Thread.Sleep or fixed waits after interactions.** — 交互后禁止写死等待时间，必须用 wait-until-condition
- **Do not skip state reset between PlayMode test cases.** — PlayMode 测试用例之间不能跳过状态重置
- **Do not rely solely on screenshots for UI verification.** — UI 验证不能只靠截图，必须配合 ui-hierarchy-snapshot
- **Do not output only Markdown reports.** — 验证报告必须同时输出 Markdown（人）和 JSON（机器）格式

## Output Expectation

Use the structure in [references/output-contract.md](references/output-contract.md) for human-readable Markdown and [references/json-output-schema.md](references/json-output-schema.md) for machine-consumable JSON. Keep the report short but decisive. The user should be able to see:

- why this route was chosen — 为什么选这个路由
- what passed — 通过了什么
- what failed — 失败了什么
- what still needs Unity-MCP work — 还需要什么 Unity-MCP 工作
- the next most useful command or action — 下一步最有用的命令或操作

**v2.0 双轨输出要求：**
```
每次验证输出两份：
├── validation-report.md     ← 人类消费（开发者阅读）
└── validation-report.json   ← 机器消费（Codex 自动审查）
```

## Comparison: Unity-MCP vs uloop-*
<!-- 对比：Unity-MCP 与 uloop-* -->

| 功能 | uloop-* | Unity-MCP |
|------|---------|-----------|
| 编译执行 | uloop-compile | `script-execute` (Roslyn) |
| 运行测试 | uloop-run-tests | `tests-run` |
| 层级检查 | uloop-get-hierarchy | `scene-get-data`, `gameobject-find` |
| 动态代码 | uloop-execute-dynamic-code | `script-execute`, `reflection-method-call` |
| PlayMode控制 | uloop-control-play-mode | `editor-application-set-state` |
| 截图 | uloop-screenshot | `screenshot-game-view`, `screenshot-camera` |
| 日志 | uloop-get-logs | `console-get-logs` |
| 输入模拟 | uloop-simulate-mouse-* | 需自定义 Tool |
| 录制回放 | uloop-record/replay-input | 需自定义 Tool |
| 反射调用 | 无 | `reflection-method-call` ✅ |
| 方法查找 | 无 | `reflection-method-find` ✅ |
| 运行时嵌入 | 无 | 支持 ✅ |
| 工具数量 | ~10 | 100+ ✅ |
| 自定义工具 | 需改 skill | 单行代码添加 ✅ |
| **异步等待** | 无 | `wait-until-condition` ✅ v2.0 |
| **UI 快照** | 无 | `ui-hierarchy-snapshot` ✅ v2.0 |
| **状态重置** | 无 | `state-reset` ✅ v2.0 |
| **JSON 输出** | 无 | JSON Schema ✅ v2.0 |
| **测试后门** | 无 | `reflection-method-call` ✅ v2.0 |

**Unity-MCP 更强大：**
- 反射系统可查找/调用任何方法
- 自定义 Tool 只需一行代码
- 运行时可在编译后的游戏内使用
- 工具更丰富，覆盖更全面
- v2.0 解决了时序同步、状态污染、断言盲区三大硬伤

**uloop-* 更完整（当前）：**
- 内置输入模拟和录制回放
- 更轻量，依赖更少
