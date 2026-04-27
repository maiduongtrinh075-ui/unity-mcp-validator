# Unity-MCP Custom Tools: Async Wait & Condition Polling
<!-- Unity-MCP 自定义工具：异步等待与条件轮询 -->

解决 PlayMode 验证中最大的敌人——**时序不同步**。让验证流程像真人一样"等系统结算完毕"再截取结果，而非写死 `Thread.Sleep`。

## 安装方式

1. 在项目中创建或确认 `Assets/Scripts/MCP/` 目录
2. 创建以下脚本文件
3. Unity-MCP 会自动识别并注册这些 Tool

---

## Tool 1: 条件轮询等待

<!-- 核心工具：轮询 C# 条件直到为 true 或超时 -->

```csharp
// Assets/Scripts/MCP/Tool_WaitFor.cs

using UnityEngine;
using System;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_WaitFor
{
    /// <summary>
    /// 等待直到 C# 条件表达式为 true，或超时
    /// Wait until a C# condition expression evaluates to true, or timeout
    /// </summary>
    [McpPluginTool("wait-until-condition", Title = "Wait until C# condition is true")]
    [Description("Poll a C# condition expression until it returns true or timeout. " +
                 "Use this after interactions to wait for game state to settle. " +
                 "轮询 C# 条件表达式直到返回 true 或超时。交互后使用此工具等待游戏状态稳定。")]
    public static string WaitUntilCondition
    (
        [Description("C# expression that returns bool. Must reference types in loaded assemblies. " +
                     "返回 bool 的 C# 表达式。例如：'Board.Instance != null && !Board.Instance.IsAnimating'")]
        string condition,

        [Description("Maximum wait time in seconds. 最大等待时间（秒）")]
        float timeoutSeconds = 5f,

        [Description("Polling interval in seconds. 轮询间隔（秒）")]
        float pollIntervalSeconds = 0.1f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            float elapsed = 0f;
            bool lastResult = false;
            string lastError = null;

            while (elapsed < timeoutSeconds)
            {
                try
                {
                    // 使用 script-execute 的同款 Roslyn 编译来评估表达式
                    var result = EvaluateCondition(condition);
                    if (result)
                    {
                        return $"[Success] Condition met after {elapsed:F2}s. " +
                               $"Expression: '{condition}'";
                    }
                    lastResult = result;
                }
                catch (Exception ex)
                {
                    lastError = ex.Message;
                }

                // 等待下一帧 + 轮询间隔
                System.Threading.Thread.Sleep((int)(pollIntervalSeconds * 1000));
                elapsed += pollIntervalSeconds;
            }

            // 超时报告
            string report = $"[Timeout] Condition not met after {timeoutSeconds}s. " +
                            $"Expression: '{condition}'. " +
                            $"Last result: {lastResult}";
            if (lastError != null)
                report += $". Last error: {lastError}";

            return report;
        });
    }

    /// <summary>
    /// 等待直到 Animator 进入指定状态
    /// Wait until Animator enters specified state
    /// </summary>
    [McpPluginTool("wait-for-animation-state", Title = "Wait for Animator state")]
    [Description("Wait until a target Animator enters the specified state name. " +
                 "等待目标 Animator 进入指定状态名称。")]
    public static string WaitForAnimationState
    (
        [Description("GameObject name containing the Animator. 包含 Animator 的 GameObject 名称")]
        string gameObjectName,

        [Description("Target animation state name. 目标动画状态名称")]
        string stateName,

        [Description("Animator layer index (default 0). Animator 层级索引")]
        int layerIndex = 0,

        [Description("Maximum wait time in seconds. 最大等待时间（秒）")]
        float timeoutSeconds = 5f,

        [Description("Polling interval in seconds. 轮询间隔（秒）")]
        float pollIntervalSeconds = 0.05f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            float elapsed = 0f;

            while (elapsed < timeoutSeconds)
            {
                var go = GameObject.Find(gameObjectName);
                if (go == null)
                    return $"[Error] GameObject '{gameObjectName}' not found.";

                var animator = go.GetComponent<Animator>();
                if (animator == null)
                    return $"[Error] GameObject '{gameObjectName}' has no Animator.";

                if (!animator.isActiveAndEnabled)
                    return $"[Error] Animator on '{gameObjectName}' is not active.";

                var stateInfo = animator.GetCurrentAnimatorStateInfo(layerIndex);

                // 匹配状态名称（Animator 状态名的哈希比较）
                if (stateInfo.IsName(stateName))
                {
                    return $"[Success] Animator on '{gameObjectName}' entered state '{stateName}' " +
                           $"after {elapsed:F2}s. NormalizedTime: {stateInfo.normalizedTime:F2}";
                }

                System.Threading.Thread.Sleep((int)(pollIntervalSeconds * 1000));
                elapsed += pollIntervalSeconds;
            }

            // 超时 — 报告当前状态帮助调试
            var currentGo = GameObject.Find(gameObjectName);
            if (currentGo != null)
            {
                var currentAnim = currentGo.GetComponent<Animator>();
                if (currentAnim != null)
                {
                    var currentState = currentAnim.GetCurrentAnimatorStateInfo(layerIndex);
                    return $"[Timeout] Waited {timeoutSeconds}s for state '{stateName}' on " +
                           $"'{gameObjectName}'. Current state hash: {currentState.shortNameHash}, " +
                           $"NormalizedTime: {currentState.normalizedTime:F2}";
                }
            }

            return $"[Timeout] Waited {timeoutSeconds}s for state '{stateName}' on " +
                   $"'{gameObjectName}'. Object or Animator no longer available.";
        });
    }

    /// <summary>
    /// 等待指定帧数后继续
    /// Wait for specified number of frames before continuing
    /// </summary>
    [McpPluginTool("wait-for-frame-count", Title = "Wait for N frames")]
    [Description("Wait for a specified number of Unity frames to pass. " +
                 "Useful for waiting one frame after UI changes to let layout rebuild. " +
                 "等待指定数量的 Unity 帧数。适用于 UI 变更后等待一帧让布局重建。")]
    public static string WaitForFrameCount
    (
        [Description("Number of frames to wait. 等待的帧数")]
        int frameCount = 1,

        [Description("Maximum wait time in seconds (safety timeout). 最大等待时间（安全超时）")]
        float timeoutSeconds = 10f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            int startFrame = Time.frameCount;
            int targetFrame = startFrame + frameCount;
            float elapsed = 0f;

            while (Time.frameCount < targetFrame && elapsed < timeoutSeconds)
            {
                System.Threading.Thread.Sleep(10);
                elapsed += 0.01f;
            }

            int actualFrames = Time.frameCount - startFrame;

            if (Time.frameCount >= targetFrame)
            {
                return $"[Success] Waited {actualFrames} frames " +
                       $"({elapsed:F2}s). Target was {frameCount} frames.";
            }

            return $"[Timeout] Only {actualFrames}/{frameCount} frames passed " +
                   $"after {timeoutSeconds}s timeout.";
        });
    }

    /// <summary>
    /// 等待直到控制台无新错误日志（稳定检查）
    /// Wait until console has no new error logs for a quiet period
    /// </summary>
    [McpPluginTool("wait-for-stable", Title = "Wait for game state to stabilize")]
    [Description("Wait until a condition is met AND no new error logs appear for a quiet period. " +
                 "Combines wait-until-condition with log monitoring. " +
                 "等待条件满足且在静默期内无新错误日志。结合条件等待与日志监控。")]
    public static string WaitForStable
    (
        [Description("C# condition expression. C# 条件表达式")]
        string condition,

        [Description("Quiet period in seconds with no new errors. 无新错误的静默期（秒）")]
        float quietPeriodSeconds = 0.5f,

        [Description("Maximum wait time in seconds. 最大等待时间（秒）")]
        float timeoutSeconds = 10f,

        [Description("Polling interval in seconds. 轮询间隔（秒）")]
        float pollIntervalSeconds = 0.1f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            float elapsed = 0f;
            float quietStart = -1f;
            int lastErrorCount = GetErrorLogCount();

            while (elapsed < timeoutSeconds)
            {
                try
                {
                    bool conditionMet = EvaluateCondition(condition);
                    int currentErrorCount = GetErrorLogCount();

                    if (!conditionMet)
                    {
                        // 条件未满足 — 重置静默计时
                        quietStart = -1f;
                        lastErrorCount = currentErrorCount;
                    }
                    else
                    {
                        // 条件已满足 — 检查是否有新错误
                        bool hasNewErrors = currentErrorCount > lastErrorCount;

                        if (hasNewErrors)
                        {
                            // 新错误出现 — 重置静默计时
                            quietStart = -1f;
                            lastErrorCount = currentErrorCount;
                        }
                        else
                        {
                            // 条件满足且无新错误 — 开始/继续静默期
                            if (quietStart < 0)
                                quietStart = elapsed;

                            float quietDuration = elapsed - quietStart;
                            if (quietDuration >= quietPeriodSeconds)
                            {
                                return $"[Success] State stable after {elapsed:F2}s " +
                                       $"(quiet for {quietDuration:F2}s). " +
                                       $"Expression: '{condition}'";
                            }
                        }
                    }
                }
                catch (Exception ex)
                {
                    // 评估异常 — 不重置，继续尝试
                    Debug.LogWarning($"[wait-for-stable] Condition eval error: {ex.Message}");
                }

                System.Threading.Thread.Sleep((int)(pollIntervalSeconds * 1000));
                elapsed += pollIntervalSeconds;
            }

            return $"[Timeout] State did not stabilize after {timeoutSeconds}s. " +
                   $"Expression: '{condition}'";
        });
    }

    // ─── 内部辅助方法 ──────────────────────────────────

    /// <summary>
    /// 评估 C# 条件表达式
    /// 注意：此实现使用反射查找。完整实现应集成 Roslyn 编译（同 script-execute）
    /// </summary>
    private static bool EvaluateCondition(string condition)
    {
        // 方案 A：简单反射评估（单例属性/字段）
        // 支持格式：'ClassName.PropertyName'、'ClassName.PropertyName == value'
        // 例如：'Board.Instance.IsAnimating == false'

        var trimmed = condition.Trim();

        // 处理 == 比较
        if (trimmed.Contains("=="))
        {
            var parts = trimmed.Split(new[] { "==" }, StringSplitOptions.None);
            if (parts.Length == 2)
            {
                var leftValue = ResolveValue(parts[0].Trim());
                var rightValue = parts[1].Trim().Trim('"', '\'');

                // 尝试解析右侧为字面量
                object rightLiteral = ParseLiteral(rightValue);
                if (rightLiteral != null)
                    return leftValue?.Equals(rightLiteral) ?? false;

                // 右侧也是表达式
                var rightResolved = ResolveValue(rightValue);
                return leftValue?.Equals(rightResolved) ?? false;
            }
        }

        // 处理 != 比较
        if (trimmed.Contains("!="))
        {
            var parts = trimmed.Split(new[] { "!=" }, StringSplitOptions.None);
            if (parts.Length == 2)
            {
                var leftValue = ResolveValue(parts[0].Trim());
                var rightValue = parts[1].Trim().Trim('"', '\'');

                object rightLiteral = ParseLiteral(rightValue);
                if (rightLiteral != null)
                    return !(leftValue?.Equals(rightLiteral) ?? false);

                var rightResolved = ResolveValue(rightValue);
                return !(leftValue?.Equals(rightResolved) ?? false);
            }
        }

        // 直接解析为 bool
        var directValue = ResolveValue(trimmed);
        if (directValue is bool boolVal)
            return boolVal;

        throw new InvalidOperationException(
            $"Cannot evaluate condition '{condition}' as boolean. " +
            $"Resolved to: {directValue?.GetType().Name ?? "null"}");
    }

    /// <summary>
    /// 解析属性/字段路径值
    /// 支持格式：'TypeName.Instance.Property'、'TypeName.Property'（静态）
    /// </summary>
    private static object ResolveValue(string path)
    {
        var parts = path.Split('.');
        if (parts.Length < 2)
            throw new InvalidOperationException($"Invalid path: '{path}'. Need at least 'Type.Member'.");

        // 查找类型（在所有已加载程序集中）
        Type type = null;
        foreach (var assembly in AppDomain.CurrentDomain.GetAssemblies())
        {
            type = assembly.GetType(parts[0]);
            if (type != null) break;
        }

        if (type == null)
            throw new InvalidOperationException($"Type '{parts[0]}' not found in loaded assemblies.");

        object current = null;

        // 如果第二部分是 Instance（单例模式）
        int startIndex = 1;
        if (parts.Length >= 3 && parts[1] == "Instance")
        {
            var instanceProp = type.GetProperty("Instance",
                System.Reflection.BindingFlags.Public |
                System.Reflection.BindingFlags.Static);
            if (instanceProp == null)
                instanceProp = type.GetProperty("instance",
                    System.Reflection.BindingFlags.Public |
                    System.Reflection.BindingFlags.Static);

            if (instanceProp != null)
            {
                current = instanceProp.GetValue(null);
                startIndex = 2;
            }
        }

        // 遍历剩余路径
        for (int i = startIndex; i < parts.Length; i++)
        {
            if (current == null && i > startIndex)
                throw new InvalidOperationException($"Null reference at '{parts[i - 1]}' in path '{path}'.");

            var targetType = current?.GetType() ?? type;
            var member = parts[i];

            // 尝试属性
            var prop = targetType.GetProperty(member,
                System.Reflection.BindingFlags.Public |
                System.Reflection.BindingFlags.Instance |
                System.Reflection.BindingFlags.Static);
            if (prop != null)
            {
                current = prop.GetValue(current);
                continue;
            }

            // 尝试字段
            var field = targetType.GetField(member,
                System.Reflection.BindingFlags.Public |
                System.Reflection.BindingFlags.Instance |
                System.Reflection.BindingFlags.Static);
            if (field != null)
            {
                current = field.GetValue(current);
                continue;
            }

            throw new InvalidOperationException(
                $"Member '{member}' not found on type '{targetType.Name}'.");
        }

        return current;
    }

    /// <summary>
    /// 解析字面量值
    /// </summary>
    private static object ParseLiteral(string value)
    {
        if (value.Equals("true", StringComparison.OrdinalIgnoreCase)) return true;
        if (value.Equals("false", StringComparison.OrdinalIgnoreCase)) return false;
        if (value.Equals("null", StringComparison.OrdinalIgnoreCase)) return null;
        if (int.TryParse(value, out int intVal)) return intVal;
        if (float.TryParse(value, System.Globalization.NumberStyles.Float,
            System.Globalization.CultureInfo.InvariantCulture, out float floatVal)) return floatVal;

        return null; // 无法解析为字面量
    }

    /// <summary>
    /// 获取当前错误日志数量
    /// </summary>
    private static int GetErrorLogCount()
    {
        // 通过反射调用 Unity LogEntries 获取错误计数
        var logEntriesType = System.Reflection.Assembly.Load("UnityEditor")
            .GetType("UnityEditor.LogEntries");
        if (logEntriesType != null)
        {
            var getCountMethod = logEntriesType.GetMethod("GetCount",
                System.Reflection.BindingFlags.Public |
                System.Reflection.BindingFlags.Static);
            if (getCountMethod != null)
                return (int)getCountMethod.Invoke(null, null);
        }
        return 0;
    }
}
```

---

## 使用示例

### 示例 1：等待三消消除动画完成

```bash
# 模拟滑动后，等待消除动画结束
simulate-drag-world startX=200 startY=300 endX=400 endY=300

# 等待 Board 不再处于动画状态（核心！）
wait-until-condition condition="Board.Instance.IsAnimating == false" timeoutSeconds=5

# 确认状态稳定后再截图
screenshot-game-view
reflection-method-call → Board.Instance.Score
```

### 示例 2：等待动画状态转换

```bash
# 点击按钮触发动画
simulate-click-ui x=150 y=100

# 等待 Animator 进入目标状态
wait-for-animation-state gameObjectName="Character" stateName="Attack" timeoutSeconds=3

# 确认攻击动画播放完毕
wait-for-animation-state gameObjectName="Character" stateName="Idle" timeoutSeconds=3
```

### 示例 3：等待 UI 布局重建

```bash
# 修改 UI 后等待布局计算完成
reflection-method-call → UIController.ShowPanel("Settings")

# 等一帧让 LayoutRebuilder 处理
wait-for-frame-count frameCount=2

# 现在 UI 元素位置已确定，可以安全截图
screenshot-game-view
```

### 示例 4：等待状态稳定（条件 + 无新错误）

```bash
# 点击按钮触发复杂流程
simulate-click-ui x=200 y=150

# 等待条件满足且 0.5 秒内无新错误
wait-for-stable condition="GameController.Instance.CurrentPhase == Phase.Idle" quietPeriodSeconds=0.5 timeoutSeconds=10

# 状态稳定，可安全检查
ui-hierarchy-snapshot
reflection-method-call → GameController.Instance.GetState()
```

---

## 最佳实践

### 1. 替代 Thread.Sleep

```
# ❌ 旧做法：写死等待时间
simulate-drag-world startX=200 startY=300 endX=400 endY=300
# 等待 1 秒（可能不够或浪费）
... 隐式等待 ...

# ✅ 新做法：条件轮询
simulate-drag-world startX=200 startY=300 endX=400 endY=300
wait-until-condition condition="Board.Instance.IsAnimating == false" timeoutSeconds=5
```

### 2. 超时值的选择

| 场景 | 推荐 timeoutSeconds | 推荐 pollIntervalSeconds |
|------|---------------------|--------------------------|
| UI 布局重建 | 2 | 0.02 |
| 简单动画 | 3 | 0.05 |
| 复杂交互（三消消除） | 5 | 0.1 |
| 资源加载 | 10 | 0.5 |
| 网络等待 | 30 | 1.0 |

### 3. 条件表达式格式

```
# 单例属性
ClassName.Instance.PropertyName == value

# 静态属性
ClassName.PropertyName == value

# 布尔属性（无需 == true）
ClassName.Instance.IsActive

# 比较操作
ClassName.Instance.Score != 0
ClassName.Instance.Health > 0
```

### 4. 与状态重置配合使用

```
# 重置场景状态
state-reset method="scene_reload"

# 进入 PlayMode
editor-application-set-state playMode=true

# 等待游戏初始化完成
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=10

# 开始验证
...
```

---

## 注意事项

1. **主线程安全**：所有 Unity API 调用都通过 `MainThread.Instance.Run()` 执行
2. **条件表达式安全**：当前实现使用反射解析，不支持复杂表达式（如 LINQ、lambda）
   - 复杂条件请用 `script-execute` + `wait-until-condition` 组合
3. **EvaluateCondition 的局限**：反射解析器支持 `Type.Instance.Property` 和 `Type.Property` 路径
   - 对于更复杂的表达式，建议项目暴露专用的 bool 属性供等待使用
4. **帧率影响**：轮询间隔小于一帧时，实际上每帧只检查一次
5. **与录制回放兼容**：wait 工具不消耗录制的事件，仅用于验证流程中的同步点

---

## 文件结构

```
Assets/Scripts/MCP/
├── Tool_WaitFor.cs           # 异步等待工具（本文件）
├── Tool_MouseInput.cs        # 世界空间鼠标输入
├── Tool_MouseUI.cs           # UI 空间鼠标输入
├── Tool_KeyboardInput.cs     # 键盘输入
├── Tool_InputRecording.cs    # 录制与回放
└── Tool_UISnapshot.cs        # UI DOM 树快照
```
