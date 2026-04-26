# Runtime Probes
<!-- 运行时探针：PlayMode验证的详细步骤 -->

Use this file when the selected route requires PlayMode validation or when the bug only appears during interaction.

## Preconditions
<!-- 前置条件 -->

Before probing runtime behavior, confirm:

- Unity-MCP plugin installed — Unity-MCP 插件已安装
- Unity Editor is open on the target project — Unity Editor 已打开目标项目
- MCP server reachable — MCP 服务器可达
- the target scene is known — 目标场景已知

Check with Unity-MCP tools:
<!-- 用 Unity-MCP 工具检查 -->

```
editor-application-get-state  → 确认编辑器状态
scene-list-opened             → 确认目标场景已打开
```

If any of these fail, report the exact blocker and do not pretend runtime validation happened.
<!-- 如果任何一项失败，报告具体阻塞原因，不要假装完成了运行时验证 -->

## Probe Ladder
<!-- 探针阶梯：按顺序执行 -->

Run this ladder in order unless the task clearly needs fewer steps.

### 1. Stabilize the baseline
<!-- 1. 稳定基线 -->

- compile the project — 编译项目
- clear Console logs — 清空控制台日志
- make sure the intended scene is open — 确保目标场景已打开
- capture one baseline screenshot — 捕获一张基线截图

**Unity-MCP tools:**
```
script-execute         → 编译执行简单代码确认编译成功
console-get-logs       → 检查是否有编译错误
scene-list-opened      → 确认场景
screenshot-game-view   → 基线截图
```

### 2. Enter PlayMode
<!-- 2. 进入播放模式 -->

Enter PlayMode and confirm it actually started before sending input.
<!-- 进入 PlayMode 并确认真正启动后再发送输入 -->

**Unity-MCP tools:**
```
editor-application-set-state playMode=true paused=false
editor-application-get-state → 确认 PlayMode 已启动
```

### 3. Interact or trigger the scenario
<!-- 3. 交互或触发场景 -->

Use the tool that matches the interaction path:

| Interaction type | Approach |
|------------------|----------|
| UI buttons, toggles, sliders | 自定义 `simulate-mouse-ui` Tool 或 `reflection-method-call` 调用 UI 方法 |
| Gameplay input (clicks, drags) | 自定义 `simulate-mouse-input` Tool 或 `reflection-method-call` 调用输入处理方法 |
| Complex multi-step scenario | 录制回放需自定义 Tool，或用 `script-execute` 执行完整序列 |

**注意：** Unity-MCP 没有内置输入模拟，需要自定义 Tool：
```csharp
[McpPluginToolType]
public class Tool_Input
{
    [McpPluginTool("simulate-click", Title = "Simulate click")]
    public static string SimulateClick(int x, int y)
    {
        return MainThread.Instance.Run(() =>
        {
            // 实现点击逻辑，如调用 Input System 或 UI EventSystem
            return "[Success] Click simulated at ({x}, {y})";
        });
    }
}
```

### 4. Capture evidence after each action
<!-- 4. 每个操作后捕获证据 -->

After the important interaction steps, collect:

- screenshot — 截图
- logs — 日志
- relevant runtime state — 相关运行时状态

Do this after the first successful step and again when the bug appears. The difference often explains the failure.
<!-- 在第一个成功步骤后和 bug 出现时都做一次。差异往往解释了失败原因 -->

**Unity-MCP tools:**
```
screenshot-game-view   → 捕获当前游戏视图
console-get-logs       → 捕获日志
```

### 5. Probe runtime state
<!-- 5. 探测运行时状态 -->

Use `reflection-method-call` or `script-execute` to inspect live state instead of guessing.
<!-- 用反射工具检查运行时状态而不是猜测 -->

**Unity-MCP tools:**
```
reflection-method-call → 调用静态方法获取状态
reflection-method-find → 查找需要调用的方法
script-execute         → 执行任意 C# 代码片段
```

For your project, prefer checking:
<!-- 针对你的项目，优先检查 -->

```csharp
// 示例：检查控制器状态
// 用 reflection-method-call 或 script-execute

var controller = UnityEngine.Object.FindObjectOfType<MyGameController>();
var phase = controller.CurrentPhase;
var inputLocked = controller.InputLocked;
return $"Phase: {phase}, InputLocked: {inputLocked}";
```

Key runtime state to check:
- primary controller current phase/state — 主控制器当前阶段/状态
- whether input is locked — 输入是否锁定
- current selection or target state — 当前选择或目标状态
- whether game objects still exist and remain interactive — 游戏对象是否仍然存在且可交互
- whether an async operation or tween flag never cleared — 异步操作或补间标志是否未清除
- whether the camera, colliders, or layer masks still line up with the input path — 相机、碰撞体或层蒙版是否仍然与输入路径对齐

### 6. Exit or reset cleanly
<!-- 6. 干净退出或重置 -->

Stop PlayMode after collecting enough evidence unless the task explicitly needs a longer-running observation.
<!-- 收集足够证据后停止 PlayMode，除非任务明确需要更长时间观察 -->

**Unity-MCP tools:**
```
editor-application-set-state playMode=false
```

## Suggested Probe Recipes
<!-- 建议的探针配方 -->

### A. Input / interaction bug
<!-- A. 输入/交互bug -->

Use this recipe for:

- click does nothing — 点击无效
- interaction does not start — 交互不开始
- action does not complete or return — 操作不完成或不返回
- works for a few actions, then freezes — 几次操作后冻结

**Unity-MCP sequence:**
```
1. script-execute → 编译确认
2. screenshot-game-view → 基线截图
3. editor-application-set-state playMode=true → 进入 PlayMode
4. [自定义输入工具] → 执行交互
5. screenshot-game-view + console-get-logs → 捕获证据
6. reflection-method-call → 检查控制器状态和输入锁
7. 重复直到失败或足够信心
```

### B. UI bug
<!-- B. UI bug -->

Use this recipe for:

- button does nothing — 按钮无效
- UI text does not update — UI文本不更新
- panel exists but is not wired — 面板存在但未连线

**Unity-MCP sequence:**
```
1. gameobject-find → 检查 UI 对象是否存在
2. gameobject-component-get → 检查组件引用
3. editor-application-set-state playMode=true → 进入 PlayMode
4. [自定义 UI 点击工具] → 点击 UI
5. screenshot-game-view + console-get-logs → 捕获证据
6. reflection-method-call → 检查运行时字段
```

### C. Scene / prefab bug
<!-- C. 场景/预制体bug -->

Use this recipe for:

- object exists but looks wrong — 对象存在但看起来不对
- prefab instantiated without component/reference — 预制体实例化但缺组件/引用
- scene compiles but runtime is broken — 场景编译通过但运行时坏了

**Unity-MCP sequence:**
```
1. script-execute → 编译
2. scene-get-data + gameobject-find → 检查层级和对象
3. assets-get-data → 检查预制体引用
4. console-get-logs → 检查日志
5. editor-application-set-state playMode=true → 编辑器侧看起来正常后进入 PlayMode
6. screenshot-game-view → 截图
7. reflection-method-call → 检查运行时组件
```

## Failure Signatures
<!-- 失败特征：常见问题的诊断方向 -->

### Clicks never work from the first frame
<!-- 点击从一开始就不工作 -->

Check with Unity-MCP:
```
reflection-method-call → 调用方法检查:
- Input System backend 和 project settings
- collider 存在性和大小
- layer mask
- camera mode 和 screen-to-world 转换
- 代码是否监听正确的输入路径
```

### Works briefly, then stops responding
<!-- 工作一会儿后停止响应 -->

Check with Unity-MCP:
```
reflection-method-call → 检查:
- input lock 是否仍被持有
- controller phase 是否从未返回空闲
- async callback 是否从未触发
- 日志中是否有异常（console-get-logs）
- selection state 是否卡在已销毁或替换的对象上
```

### Action animates incorrectly or never completes
<!-- 动画不正确或从未完成 -->

Check with Unity-MCP:
```
reflection-method-call → 检查:
- callback 完成路径
- 视觉状态恢复
- 逻辑状态恢复
- 操作完成后的解锁时机
```

### Objects render but interactions are dead
<!-- 对象渲染但交互死亡 -->

Check with Unity-MCP:
```
gameobject-component-get → 检查预制体碰撞体
reflection-method-call → 检查:
- raycast 路径
- event system 干扰
- camera 和 layer mask 是否匹配可交互对象
```

## Evidence Minimum
<!-- 证据最低要求 -->

For any runtime bug report, collect at least:

- one screenshot before the problematic interaction — 问题交互前的一张截图（`screenshot-game-view`）
- one screenshot after the bug appears — bug出现后的一张截图（`screenshot-game-view`）
- one log capture from the same session — 同一会话的一次日志捕获（`console-get-logs`）
- one runtime state snapshot that explains whether the system is stuck, unlocked, missing references, or silently failing — 一个运行时状态快照（`reflection-method-call` 或 `script-execute`）