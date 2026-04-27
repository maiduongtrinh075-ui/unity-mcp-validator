# State Reset Strategies
<!-- 状态沙盒重置策略：确保每次验证在纯净上下文中运行 -->

解决 Layer 4 PlayMode 验证中最大的可靠性威胁——**测试间状态污染**。
前一个测试修改的分数、对象、场景状态，不能影响后一个测试的基线。

---

## 问题场景

```
测试 A：拖拽消除 → 分数 = 30, Gem 数 = 60
测试 B：验证初始分数显示 → 期望分数 = 0，实际 = 30  ❌ 污染！
```

---

## 重置策略分级

| 策略 | 代价 | 可靠性 | 适用场景 |
|------|------|--------|----------|
| **方法调用** | 低 (10-50ms) | 中 | 项目有 Reset API |
| **场景重载** | 中 (1-3s) | 高 | 通用场景 |
| **PlayMode 重启** | 高 (3-8s) | 最高 | 需要完全干净状态 |

---

## 策略 1：方法调用重置（推荐首选）

通过 `reflection-method-call` 或 `script-execute` 调用项目的重置方法。

### 前提条件

项目需要暴露一个重置方法（见 [test-backdoors.md](test-backdoors.md)）：

```csharp
#if UNITY_EDITOR
public class GameStateReset : MonoBehaviour
{
    /// <summary>
    /// 重置游戏到初始状态（不重载场景）
    /// </summary>
    [ContextMenu("Reset Game State")]
    public static void ResetAll()
    {
        // 重置核心管理器
        var controller = FindObjectOfType<GameController>();
        if (controller != null)
            controller.ResetState();

        // 重置 UI
        var uiManager = FindObjectOfType<UIManager>();
        if (uiManager != null)
            uiManager.ResetUI();

        // 重置输入
        var inputManager = FindObjectOfType<InputManager>();
        if (inputManager != null)
        {
            inputManager.UnlockInput();
            inputManager.ResetSelection();
        }

        // 清空日志
        Debug.ClearDeveloperConsole();

        Debug.Log("[StateReset] Game state reset completed.");
    }
}
#endif
```

### MCP 工具调用

```bash
# 方法调用重置
reflection-method-call typeName="GameStateReset" methodName="ResetAll"

# 或通过 script-execute
script-execute code="
    GameStateReset.ResetAll();
    return \"[StateReset] Done\";
"
```

### 配置

```yaml
# validation-config.yaml
state_reset:
  method: "method_call"
  reset_method: "GameStateReset.ResetAll"
  reset_assembly: "Assembly-CSharp"
```

---

## 策略 2：场景重载（通用可靠）

重载当前活动场景，回到场景保存时的初始状态。

### MCP 工具调用

```bash
# 方式 A：通过 script-execute
script-execute code="
    var scene = UnityEngine.SceneManagement.SceneManager.GetActiveScene();
    UnityEngine.SceneManagement.SceneManager.LoadScene(scene.name);
    return $\"[StateReset] Reloading scene: {scene.name}\";
"

# 方式 B：如果项目有封装好的重载方法
reflection-method-call typeName="GameStateReset" methodName="ReloadCurrentScene"
```

### 注意事项

1. **PlayMode 内重载**：在 PlayMode 运行中重载场景会触发所有 `Awake()`/`Start()` 重新执行
2. **DontDestroyOnLoad 对象**：标记了 `DontDestroyOnLoad` 的对象不会被场景重载清除，需要额外处理
3. **异步加载**：`SceneManager.LoadScene` 是异步的，重载后需要等待场景加载完成

```bash
# 安全的场景重载流程
script-execute code="
    var scene = UnityEngine.SceneManagement.SceneManager.GetActiveScene();
    UnityEngine.SceneManagement.SceneManager.LoadScene(scene.name);
    return \"[StateReset] Scene reload initiated\";
"

# 等待场景加载完成
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=10
wait-for-frame-count frameCount=3
```

### 配置

```yaml
# validation-config.yaml
state_reset:
  method: "scene_reload"
  reload_scene_on_each_test: true
  wait_after_reload_seconds: 2
  dont_destroy_objects_to_cleanup:
    - "AudioManager"
    - "GameManager"
```

---

## 策略 3：PlayMode 重启（完全干净）

退出 PlayMode 再重新进入。Unity 会自动还原所有运行时修改。

### MCP 工具调用

```bash
# 退出 PlayMode
editor-application-set-state playMode=false

# 等待编辑器稳定
wait-for-frame-count frameCount=5

# 重新进入 PlayMode
editor-application-set-state playMode=true

# 等待游戏初始化
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=10
```

### 注意事项

1. **最可靠但最慢**：3-8 秒开销
2. **域名重载**：如果 Unity 配置了 Domain Reload，退出 PlayMode 会重置所有静态变量
3. **Enter PlayMode Settings**：如果项目启用了 "Reload Domain" = Off，静态变量不会重置，需要策略 1 配合

---

## Layer 4 重置流程规范

### 标准流程（含重置）

```
┌─────────────────────────────────────────────────┐
│ Layer 4 标准验证流程（含状态重置）                │
├─────────────────────────────────────────────────┤
│                                                 │
│  [Pre-Reset] 状态重置                            │
│    ├── 方法调用 / 场景重载 / PlayMode 重启        │
│    └── wait-until-condition 确认初始化完成        │
│                                                 │
│  [4A] 进入 PlayMode                             │
│    └── editor-application-set-state playMode=true│
│                                                 │
│  [4C] 基线截图 + 状态快照                        │
│    ├── screenshot-game-view                     │
│    ├── ui-hierarchy-snapshot                    │
│    └── reflection-method-call → 初始状态         │
│                                                 │
│  [4B] 输入模拟                                  │
│    └── simulate-drag-world / simulate-click-*   │
│                                                 │
│  [Wait] 等待结算                                │
│    └── wait-until-condition / wait-for-stable   │
│                                                 │
│  [4C] 证据收集                                  │
│    ├── screenshot-game-view                     │
│    ├── ui-hierarchy-snapshot                    │
│    ├── reflection-method-call → 结果状态         │
│    └── console-get-logs                         │
│                                                 │
│  [4A] 退出 PlayMode                             │
│    └── editor-application-set-state playMode=false│
│                                                 │
└─────────────────────────────────────────────────┘
```

### 多测试用例流程

```
[Pre-Reset] → Test 1 → [Inter-Reset] → Test 2 → [Inter-Reset] → Test 3 → Report
```

每次测试之间执行**间次重置 (Inter-Reset)**，确保独立。

---

## MCP 自定义工具：State Reset Helper

可选的专用重置工具，封装重置逻辑：

```csharp
// Assets/Scripts/MCP/Tool_StateReset.cs

using UnityEngine;
using UnityEngine.SceneManagement;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_StateReset
{
    /// <summary>
    /// 执行状态重置（根据项目配置选择策略）
    /// </summary>
    [McpPluginTool("state-reset", Title = "Reset game state for testing")]
    [Description("Reset the game to a clean state before running a test. " +
                 "Uses the configured reset strategy (method_call, scene_reload, or playmode_restart). " +
                 "重置游戏到干净状态用于测试。使用配置的重置策略。")]
    public static string StateReset
    (
        [Description("Reset strategy: 'auto', 'method_call', 'scene_reload', 'playmode_restart'. " +
                     "重置策略：'auto' 自动选择，'method_call' 方法调用，'scene_reload' 场景重载，'playmode_restart' PlayMode重启")]
        string strategy = "auto"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            // 如果在 PlayMode 中，先退出
            bool wasPlaying = UnityEditor.EditorApplication.isPlaying;
            if (wasPlaying && strategy == "playmode_restart")
            {
                UnityEditor.EditorApplication.isPlaying = false;
                System.Threading.Thread.Sleep(1000); // 等待退出
            }

            switch (strategy)
            {
                case "method_call":
                    return ResetViaMethod();
                case "scene_reload":
                    return ResetViaSceneReload();
                case "playmode_restart":
                    return ResetViaPlaymodeRestart(wasPlaying);
                case "auto":
                default:
                    // 优先级：方法调用 > 场景重载 > PlayMode重启
                    var methodResult = ResetViaMethod();
                    if (!methodResult.StartsWith("[Error]"))
                        return methodResult;

                    var sceneResult = ResetViaSceneReload();
                    if (!sceneResult.StartsWith("[Error]"))
                        return sceneResult;

                    return ResetViaPlaymodeRestart(wasPlaying);
            }
        });
    }

    private static string ResetViaMethod()
    {
        // 尝试查找并调用 GameStateReset.ResetAll
        var resetType = System.Type.GetType("GameStateReset, Assembly-CSharp");
        if (resetType == null)
            return "[Error] GameStateReset type not found. Create a reset method first.";

        var resetMethod = resetType.GetMethod("ResetAll",
            System.Reflection.BindingFlags.Public | System.Reflection.BindingFlags.Static);
        if (resetMethod == null)
            return "[Error] ResetAll method not found on GameStateReset.";

        resetMethod.Invoke(null, null);
        return "[Success] State reset via method call completed.";
    }

    private static string ResetViaSceneReload()
    {
        var scene = SceneManager.GetActiveScene();
        if (string.IsNullOrEmpty(scene.name))
            return "[Error] No active scene to reload.";

        SceneManager.LoadScene(scene.name);
        return $"[Success] State reset via scene reload: {scene.name}";
    }

    private static string ResetViaPlaymodeRestart(bool wasPlaying)
    {
        if (wasPlaying)
        {
            UnityEditor.EditorApplication.isPlaying = false;
            System.Threading.Thread.Sleep(1000);
        }

        UnityEditor.EditorApplication.isPlaying = true;
        return "[Success] State reset via PlayMode restart.";
    }
}
```

---

## 配置参考

```yaml
# validation-config.yaml — 状态重置配置

state_reset:
  # 重置策略：auto | method_call | scene_reload | playmode_restart
  method: "auto"

  # 方法调用策略配置
  reset_method: "GameStateReset.ResetAll"
  reset_assembly: "Assembly-CSharp"

  # 场景重载策略配置
  reload_scene_on_each_test: true
  wait_after_reload_seconds: 2

  # PlayMode 重启策略配置
  restart_playmode_on_each_test: false  # 仅在最严格模式下开启

  # DontDestroyOnLoad 对象清理列表
  dont_destroy_cleanup:
    - "AudioManager"
    - "GameManager"
    - "AdManager"

  # 重置后等待条件
  post_reset_wait_condition: "GameController.Instance != null"
  post_reset_wait_timeout: 10
```

---

## 最佳实践

### 1. 每个项目都应该有 ResetAll 方法

即使不使用 MCP 验证，`ResetAll` 在手动调试时也有用。这是最低成本的投资。

### 2. 重置策略选择逻辑

```
if (项目有 GameStateReset.ResetAll)
    → 用方法调用（最快最可靠）
else if (场景可干净重载)
    → 用场景重载
else
    → 用 PlayMode 重启（最慢但最可靠）
```

### 3. 验证前必须确认重置成功

```bash
# ❌ 不检查
state-reset strategy="method_call"
# 直接开始测试...

# ✅ 检查确认
state-reset strategy="method_call"
wait-until-condition condition="GameController.Instance.CurrentPhase == Phase.Idle" timeoutSeconds=5
reflection-method-call → GameController.Instance.Score == 0
# 确认初始状态正确后再开始测试
```

### 4. DontDestroyOnLoad 对象的特殊处理

这些对象不会被场景重载清除。需要手动处理：

```csharp
// GameStateReset.ResetAll 中
var persistentObjects = GameObject.FindObjectsOfType<GameObject>()
    .Where(go => go.scene.name == "DontDestroyOnLoad");

foreach (var obj in persistentObjects)
{
    if (obj.name.Contains("AudioManager")) continue; // 保留
    if (obj.name.Contains("GameManager"))
    {
        obj.GetComponent<GameManager>().Reset();
        continue;
    }
    // 其他需要清理的
}
```
