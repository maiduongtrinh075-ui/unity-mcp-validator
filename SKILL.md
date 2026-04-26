---
name: unity-mcp-validator
version: "1.1.0"
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

Prefer this skill whenever the task involves runtime behavior, scene wiring, prefab correctness, input issues, animation/state timing, or a "works for a while then breaks" bug.

## Core Contract
<!-- 核心契约：每次执行必须遵守 -->

Follow this contract on every run:

1. **Classify first** — 先分类变更类型，再选择工具
2. **Cheapest first** — 从最便宜的验证层开始
3. **Escalate when needed** — 只在变更类型需要时才升级到 PlayMode
4. **Probe runtime bugs** — PlayMode bug 是探针任务，不只是单元测试
5. **Be honest about blockers** — Unity-MCP 不可用时明确说明，不要假装完成
6. **No premature acceptance** — 必需的验证层未执行时，不要标记"可接受"

## Quick Start
<!-- 快速开始 -->

1. Read [references/route-matrix.md](references/route-matrix.md) and classify the change.
2. Run the required validation layers for that class.
3. If the issue is PlayMode-only or flaky, read [references/runtime-probes.md](references/runtime-probes.md) and gather runtime evidence.
4. Report using [references/output-contract.md](references/output-contract.md).
5. For full tool reference, see [references/tool-reference.md](references/tool-reference.md) — **77 个工具完整列表**。

If the task spans multiple categories, validate against the **strictest** route, not the cheapest one.

For issues encountered during validation, see [references/troubleshooting.md](references/troubleshooting.md).

## Validation Layers

## Installation

This skill requires the Unity-MCP plugin. Setup steps:

1. **Install Unity-MCP plugin** in your Unity project:
   ```bash
   npm install -g unity-mcp-cli
   unity-mcp-cli install-plugin ./YourUnityProject
   ```

2. **Configure MCP server** in Claude Code settings

3. **Add custom input tools** (optional) — copy files from `references/custom-tools-input.md` to your project

4. **Create config file** (optional) — copy `validation-config.example.yaml` to project root

### Prerequisites Check

Before using, verify:

| Check | How to Verify | Action on Failure |
|-------|---------------|-------------------|
| Unity-MCP plugin installed | Check `Assets/Scripts/MCP/` exists | Install via unity-mcp-cli |
| Unity Editor open | `editor-application-get-state` | Launch Unity manually |
| MCP server connected | Tool calls succeed | Restart MCP server |
| Input System enabled | Project Settings → Player | Enable Input System Package |

If any prerequisite fails, report the blocker and continue with available layers.

## Preflight Checks

Before executing validation, verify:

| Check | Unity-MCP Tool | Action on Failure |
|-------|-----------------|-------------------|
| Unity Editor is open | `editor-application-get-state` | Report blocker, halt |
| Project compiles | `script-execute` simple code | Fix compilation first |
| Target scene known | `scene-list-opened` | Ask user or infer |
| MCP connection works | Any tool call | Report MCP disconnected |

If any check fails, report it explicitly and continue with available layers.

## Quick Start
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

This layer answers "is the scene/prefab/editor state actually assembled the way the code expects?"
<!-- 这层回答：场景/预制体/编辑器状态是否真的按代码期望的方式组装？ -->

### Layer 4: PlayMode Automation
<!-- 第四层：运行模式自动化，输入交互/动画/运行时bug验证 -->

Use this layer for input, interaction, animation/state transitions, UI button flows, and any runtime-only bug.

**Unity-MCP 工具：**
| 工具 | 用途 |
|------|------|
| `editor-application-get-state` | 获取编辑器状态（PlayMode、暂停、编译） |
| `editor-application-set-state` | 控制编辑器状态（启动/停止/暂停 PlayMode） |
| `screenshot-game-view` | 捕获游戏视图截图 |
| `screenshot-scene-view` | 捕获场景视图截图 |
| `screenshot-camera` | 从指定相机捕获截图 |
| `console-get-logs` | 获取运行时日志 |
| `reflection-method-call` | 调用任何 C# 方法（运行时探针） |
| `reflection-method-find` | 查找项目中的方法 |
| `script-execute` | 动态编译执行 C# 代码探针 |

**输入模拟需自定义 Tool（安装到你的 Unity 项目）：**
Unity-MCP 没有内置输入模拟，需要在你的 Unity 项目中添加自定义 Tool：

```
你的Unity项目/
└── Assets/
    └── Scripts/
        └── MCP/
            ├── Tool_MouseInput.cs       # 世界空间鼠标输入
            ├── Tool_MouseUI.cs          # UI 空间鼠标输入
            ├── Tool_KeyboardInput.cs    # 键盘输入
            └── Tool_InputRecording.cs   # 录制与回放
```

详细代码见 [references/custom-tools-input.md](references/custom-tools-input.md)。
将这些 C# 文件复制到你的 Unity 项目后，Unity-MCP 会自动识别并注册。

This layer is required for "I played a few times and then it stopped responding" bugs.
<!-- 这层是必需的，针对"玩了几次然后没反应了"这类bug -->

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
4. PlayMode automation and runtime probes — 运行模式自动化和运行时探针

If a layer fails for environmental reasons, say so explicitly and continue with the remaining non-blocked layers. Do not silently downgrade the verdict.

### Step 4: Probe failures, not just symptoms
<!-- 步骤4：探针失败原因，不只是症状 -->

When PlayMode behavior is wrong, do not stop at "click did nothing." Use Unity-MCP reflection tools to gather:

- logs — 日志（`console-get-logs`）
- screenshots — 截图（`screenshot-game-view`）
- current controller/state-machine phase — 用 `reflection-method-call` 或 `script-execute` 检查
- input lock state — 输入锁状态
- selection state — 选择状态
- whether colliders, camera, and raycast path line up — 碰撞体/相机/射线检测是否对齐
- whether an animation callback never returned — 动画回调是否未返回

Use [references/runtime-probes.md](references/runtime-probes.md) for the exact probe ladder.

### Step 5: Report with a hard verdict
<!-- 步骤5：输出硬性结论 -->

Always state:

- what route was chosen — 选择了什么路由
- what layers actually ran — 实际运行了哪些层
- what evidence was collected — 收集了什么证据
- what remains blocked — 还有什么被阻塞
- whether the change is acceptable now, acceptable with known gaps, or not acceptable — 是否可接受

Use [references/output-contract.md](references/output-contract.md).

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

### Core Invariants to Guard

Define invariants your project must protect. Examples:

| Invariant | Check Method |
|-----------|--------------|
| Input locked during critical animations | `reflection-method-call` to check InputLocked |
| View never mutates model directly | Static review: look for Model writes in View |
| Controller always returns to idle | `reflection-method-call` to check CurrentPhase |
| Scene wiring matches script assumptions | `assets-get-data`, `gameobject-component-get` |

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
```

If no config file exists, the validator will use generic heuristics and prompt for clarification when needed.

## Runtime Bug Policy
<!-- 运行时bug策略 -->

When a bug appears only after repeated manual play:

1. Enter PlayMode with `editor-application-set-state` — 用 editor-application-set-state 进入 PlayMode
2. Capture baseline screenshot — 捕获基线截图
3. Interact or replay scenario — 交互或回放场景
4. Capture logs and screenshot when bug appears — bug出现时捕获日志和截图
5. Use `reflection-method-call` or `script-execute` to inspect runtime state — 用反射工具检查运行时状态
6. Report the stuck state, not just the visible symptom — 报告卡住的状态，不只是可见症状

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

### Check selection state
<!-- 检查选择状态 -->

```csharp
var selected = Selection.activeGameObject;
return selected != null ? $"Selected: {selected.name}" : "Nothing selected";
```

## Do Not Do This
<!-- 禁止行为 -->

- Do not rely on EditMode tests alone for input or animation bugs. — 输入/动画bug不能只靠EditMode测试
- Do not claim prefab or scene setup is correct without inspecting with Unity-MCP tools. — 没检查编辑器状态不能声称预制体/场景正确
- Do not mark a feature "verified" because it compiles. — 编译通过不等于验证通过
- Do not replace required PlayMode validation with manual guesswork when Unity-MCP can probe. — Unity-MCP能探针时不要用手工猜测替代
- Do not hide blocked Unity-MCP steps inside a generic "manual verification needed" sentence. Name the exact missing step. — 不要用"需要手工验证"掩盖阻塞步骤，要明确说缺什么

## Output Expectation

Use the structure in [references/output-contract.md](references/output-contract.md). Keep the report short but decisive. The user should be able to see:

- why this route was chosen — 为什么选这个路由
- what passed — 通过了什么
- what failed — 失败了什么
- what still needs Unity-MCP work — 还需要什么 Unity-MCP 工作
- the next most useful command or action — 下一步最有用的命令或操作

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

**Unity-MCP 更强大：**
- 反射系统可查找/调用任何方法
- 自定义 Tool 只需一行代码
- 运行时可在编译后的游戏内使用
- 工具更丰富，覆盖更全面

**uloop-* 更完整（当前）：**
- 内置输入模拟和录制回放
- 更轻量，依赖更少