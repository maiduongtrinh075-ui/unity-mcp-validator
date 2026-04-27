# Test Backdoor API Architecture
<!-- 测试后门 API 架构规范：让验证流程极速构造边界场景 -->

不要让 Claude 从零一步步点 UI 来构造测试环境。
暴露后门方法，直接通过 `reflection-method-call` 调用，极速构造"即将触发连锁爆炸"或"资源极度匮乏"的边界场景。

---

## 核心原则

1. **只存在于 Editor**：后门代码用 `#if UNITY_EDITOR` 或 Editor-only asmdef 保护，**绝不进生产构建**
2. **最小暴露**：只暴露构造状态的方法，不暴露破坏性方法
3. **幂等安全**：同一方法调用两次结果相同，不会累积脏状态
4. **文档化**：每个后门方法都有 `[Description]` 和参数说明，LLM 可直接理解

---

## 项目结构

```
Assets/
├── Scripts/
│   └── Editor/                        # Editor-only 目录（不进构建）
│       └── TestHelpers/               # 测试后门目录
│           ├── TestHelper.cs           # 主入口 + 通用方法
│           ├── BoardTestHelper.cs      # 棋盘/消除相关
│           ├── EconomyTestHelper.cs    # 经济/资源相关
│           ├── UITestHelper.cs         # UI/流程相关
│           └── TestHelpers.asmdef      # Editor-only 程序集定义
```

### 程序集定义文件

```json
// Assets/Scripts/Editor/TestHelpers/TestHelpers.asmdef
{
    "name": "TestHelpers",
    "references": [
        "Assembly-CSharp",
        "UnityEditor"
    ],
    "includePlatforms": [
        "Editor"
    ],
    "excludePlatforms": [],
    "allowUnsafeCode": false
}
```

**关键**：`"includePlatforms": ["Editor"]` 确保这些代码**不会被打包到任何目标平台**。

---

## Template: TestHelper 主入口

```csharp
// Assets/Scripts/Editor/TestHelpers/TestHelper.cs

#if UNITY_EDITOR
using UnityEngine;
using System;

/// <summary>
/// 测试后门 API — 供 MCP reflection-method-call 调用
/// 所有方法必须是 public static，且返回 string 格式的结果
/// </summary>
public static class TestHelper
{
    // ═══════════════════════════════════════════
    // 通用方法
    // ═══════════════════════════════════════════

    /// <summary>
    /// 重置游戏到初始状态
    /// </summary>
    [ContextMenu("TestHelper/Reset All")]
    public static string ResetAll()
    {
        try
        {
            // 重置各子系统
            BoardTestHelper.ForceClearGrid();
            EconomyTestHelper.SetResourceAmount(0);
            UITestHelper.CloseAllPanels();

            // 重置时间和暂停
            Time.timeScale = 1f;

            return "[Success] Game state fully reset.";
        }
        catch (Exception ex)
        {
            return $"[Error] Reset failed: {ex.Message}";
        }
    }

    /// <summary>
    /// 获取当前游戏完整状态快照
    /// </summary>
    public static string GetFullStateSnapshot()
    {
        try
        {
            var sb = new System.Text.StringBuilder();
            sb.AppendLine("{");
            sb.AppendLine($"  \"time_scale\": {Time.timeScale},");
            sb.AppendLine($"  \"frame_count\": {Time.frameCount},");

            // 动态收集各子系统状态
            var controller = FindObjectOfType<GameController>();
            if (controller != null)
            {
                sb.AppendLine($"  \"controller_phase\": \"{controller.CurrentPhase}\",");
                sb.AppendLine($"  \"input_locked\": {controller.InputLocked},");
            }

            var board = FindObjectOfType<Board>();
            if (board != null)
            {
                sb.AppendLine($"  \"board_gem_count\": {board.GemCount},");
                sb.AppendLine($"  \"board_score\": {board.Score},");
                sb.AppendLine($"  \"board_is_animating\": {board.IsAnimating},");
            }

            sb.AppendLine("}");
            return sb.ToString();
        }
        catch (Exception ex)
        {
            return $"[Error] Snapshot failed: {ex.Message}";
        }
    }

    /// <summary>
    /// 跳过教程
    /// </summary>
    public static string SkipTutorial()
    {
        try
        {
            var tutorial = FindObjectOfType<TutorialManager>();
            if (tutorial != null)
            {
                tutorial.ForceComplete();
                return "[Success] Tutorial skipped.";
            }
            return "[Info] No TutorialManager found. Already past tutorial?";
        }
        catch (Exception ex)
        {
            return $"[Error] Skip tutorial failed: {ex.Message}";
        }
    }

    /// <summary>
    /// 设置时间缩放
    /// </summary>
    public static string SetTimeScale(float scale)
    {
        Time.timeScale = scale;
        return $"[Success] Time scale set to {scale}.";
    }

    /// <summary>
    /// 强制触发垃圾回收
    /// </summary>
    public static string ForceGC()
    {
        GC.Collect();
        GC.WaitForPendingFinalizers();
        return $"[Success] GC forced. Heap size: {GC.GetTotalMemory(false) / 1024}KB";
    }
}
#endif
```

---

## Template: 领域特定 Helper

```csharp
// Assets/Scripts/Editor/TestHelpers/BoardTestHelper.cs

#if UNITY_EDITOR
using UnityEngine;

/// <summary>
/// 棋盘/消除相关测试后门
/// </summary>
public static class BoardTestHelper
{
    /// <summary>
    /// 清空棋盘上所有 Gem
    /// </summary>
    public static string ForceClearGrid()
    {
        var board = Object.FindObjectOfType<Board>();
        if (board == null) return "[Error] No Board found in scene.";

        board.ClearAllGems();
        return $"[Success] Grid cleared. Gem count: {board.GemCount}";
    }

    /// <summary>
    /// 强制设置棋盘为指定配置
    /// 传入 Gem 类型的二维数组，直接覆盖棋盘
    /// </summary>
    public static string ForceBoardState(string gridConfig)
    {
        var board = Object.FindObjectOfType<Board>();
        if (board == null) return "[Error] No Board found in scene.";

        // gridConfig 格式："1,2,3;4,5,6;7,8,9"（分号分行，逗号分列）
        try
        {
            var rows = gridConfig.Split(';');
            int[][] grid = new int[rows.Length][];

            for (int r = 0; r < rows.Length; r++)
            {
                var cols = rows[r].Split(',');
                grid[r] = new int[cols.Length];
                for (int c = 0; c < cols.Length; c++)
                    grid[r][c] = int.Parse(cols[c].Trim());
            }

            board.SetGridState(grid);
            return $"[Success] Board state forced to {rows.Length}x{grid[0].Length} grid.";
        }
        catch (Exception ex)
        {
            return $"[Error] Invalid grid config: {ex.Message}. " +
                   "Expected format: '1,2,3;4,5,6;7,8,9'";
        }
    }

    /// <summary>
    /// 构造"即将触发连锁消除"的边界场景
    /// </summary>
    public static string SetupChainExplosion()
    {
        var board = Object.FindObjectOfType<Board>();
        if (board == null) return "[Error] No Board found in scene.";

        board.SetupChainExplosionScenario();
        return "[Success] Chain explosion scenario set up. " +
               "One move will trigger cascade.";
    }

    /// <summary>
    /// 设置分数到指定值
    /// </summary>
    public static string SetScore(int score)
    {
        var board = Object.FindObjectOfType<Board>();
        if (board == null) return "[Error] No Board found in scene.";

        board.SetScore(score);
        return $"[Success] Score set to {score}.";
    }

    /// <summary>
    /// 获取棋盘详细状态
    /// </summary>
    public static string GetBoardState()
    {
        var board = Object.FindObjectOfType<Board>();
        if (board == null) return "[Error] No Board found in scene.";

        return $"Board State:\n" +
               $"  Gems: {board.GemCount}\n" +
               $"  Score: {board.Score}\n" +
               $"  IsAnimating: {board.IsAnimating}\n" +
               $"  Phase: {board.CurrentPhase}";
    }
}
#endif
```

---

## Template: 经济/资源 Helper

```csharp
// Assets/Scripts/Editor/TestHelpers/EconomyTestHelper.cs

#if UNITY_EDITOR
using UnityEngine;

/// <summary>
/// 经济/资源相关测试后门
/// </summary>
public static class EconomyTestHelper
{
    /// <summary>
    /// 设置资源数量
    /// </summary>
    public static string SetResourceAmount(int amount)
    {
        var economy = Object.FindObjectOfType<EconomyManager>();
        if (economy == null) return "[Error] No EconomyManager found.";

        economy.SetResource(amount);
        return $"[Success] Resource set to {amount}.";
    }

    /// <summary>
    /// 构造"资源极度匮乏"场景
    /// </summary>
    public static string SetupResourceScarce()
    {
        var economy = Object.FindObjectOfType<EconomyManager>();
        if (economy == null) return "[Error] No EconomyManager found.";

        economy.SetResource(0);
        economy.DisablePassiveIncome();
        return "[Success] Resource scarce scenario: 0 resources, no passive income.";
    }

    /// <summary>
    /// 构造"资源充裕"场景
    /// </summary>
    public static string SetupResourceAbundant()
    {
        var economy = Object.FindObjectOfType<EconomyManager>();
        if (economy == null) return "[Error] No EconomyManager found.";

        economy.SetResource(999999);
        economy.EnablePassiveIncome();
        return "[Success] Resource abundant scenario: 999999 resources, passive income enabled.";
    }

    /// <summary>
    /// 解锁所有内容
    /// </summary>
    public static string UnlockAll()
    {
        var unlockManager = Object.FindObjectOfType<UnlockManager>();
        if (unlockManager == null) return "[Error] No UnlockManager found.";

        unlockManager.UnlockAllContent();
        return "[Success] All content unlocked.";
    }
}
#endif
```

---

## Template: UI/流程 Helper

```csharp
// Assets/Scripts/Editor/TestHelpers/UITestHelper.cs

#if UNITY_EDITOR
using UnityEngine;

/// <summary>
/// UI/流程相关测试后门
/// </summary>
public static class UITestHelper
{
    /// <summary>
    /// 关闭所有弹出面板
    /// </summary>
    public static string CloseAllPanels()
    {
        var uiManager = Object.FindObjectOfType<UIManager>();
        if (uiManager == null) return "[Error] No UIManager found.";

        uiManager.CloseAllPanels();
        return "[Success] All panels closed.";
    }

    /// <summary>
    /// 强制显示指定面板
    /// </summary>
    public static string ShowPanel(string panelName)
    {
        var uiManager = Object.FindObjectOfType<UIManager>();
        if (uiManager == null) return "[Error] No UIManager found.";

        uiManager.ShowPanel(panelName);
        return $"[Success] Panel '{panelName}' shown.";
    }

    /// <summary>
    /// 强制触发游戏结束
    /// </summary>
    public static string ForceGameOver()
    {
        var controller = Object.FindObjectOfType<GameController>();
        if (controller == null) return "[Error] No GameController found.";

        controller.TriggerGameOver();
        return "[Success] Game over triggered.";
    }

    /// <summary>
    /// 跳过当前关卡
    /// </summary>
    public static string SkipCurrentLevel()
    {
        var levelManager = Object.FindObjectOfType<LevelManager>();
        if (levelManager == null) return "[Error] No LevelManager found.";

        levelManager.CompleteCurrentLevel();
        return "[Success] Current level skipped/completed.";
    }

    /// <summary>
    /// 设置到指定关卡
    /// </summary>
    public static string SetLevel(int levelNumber)
    {
        var levelManager = Object.FindObjectOfType<LevelManager>();
        if (levelManager == null) return "[Error] No LevelManager found.";

        levelManager.GoToLevel(levelNumber);
        return $"[Success] Jumped to level {levelNumber}.";
    }
}
#endif
```

---

## MCP 调用方式

```bash
# 重置游戏状态
reflection-method-call typeName="TestHelper" methodName="ResetAll"

# 构造边界场景
reflection-method-call typeName="BoardTestHelper" methodName="SetupChainExplosion"
reflection-method-call typeName="EconomyTestHelper" methodName="SetupResourceScarce"

# 直接设置状态
reflection-method-call typeName="BoardTestHelper" methodName="SetScore" parameters=["100"]
reflection-method-call typeName="EconomyTestHelper" methodName="SetResourceAmount" parameters=["999"]

# 获取状态快照
reflection-method-call typeName="TestHelper" methodName="GetFullStateSnapshot"
reflection-method-call typeName="BoardTestHelper" methodName="GetBoardState"

# 跳过流程
reflection-method-call typeName="TestHelper" methodName="SkipTutorial"
reflection-method-call typeName="UITestHelper" methodName="SkipCurrentLevel"
```

---

## validation-config.yaml 配置

```yaml
# 测试后门 API 配置
test_backdoors:
  # 是否已安装 TestHelpers 程序集
  installed: true

  # 程序集名称
  assembly: "TestHelpers"

  # 可用的后门方法（LLM 参考用）
  methods:
    # 通用
    - name: "TestHelper.ResetAll"
      description: "重置游戏到初始状态"
      returns: "string"

    - name: "TestHelper.GetFullStateSnapshot"
      description: "获取当前游戏完整状态 JSON 快照"
      returns: "string (JSON)"

    - name: "TestHelper.SkipTutorial"
      description: "跳过教程流程"
      returns: "string"

    - name: "TestHelper.SetTimeScale"
      description: "设置时间缩放"
      parameters: ["float scale"]
      returns: "string"

    # 棋盘
    - name: "BoardTestHelper.ForceClearGrid"
      description: "清空棋盘"
      returns: "string"

    - name: "BoardTestHelper.ForceBoardState"
      description: "强制设置棋盘配置（格式：1,2,3;4,5,6;7,8,9）"
      parameters: ["string gridConfig"]
      returns: "string"

    - name: "BoardTestHelper.SetupChainExplosion"
      description: "构造即将触发连锁消除的场景"
      returns: "string"

    - name: "BoardTestHelper.SetScore"
      description: "直接设置分数"
      parameters: ["int score"]
      returns: "string"

    # 经济
    - name: "EconomyTestHelper.SetResourceAmount"
      description: "设置资源数量"
      parameters: ["int amount"]
      returns: "string"

    - name: "EconomyTestHelper.SetupResourceScarce"
      description: "构造资源极度匮乏场景"
      returns: "string"

    # UI
    - name: "UITestHelper.CloseAllPanels"
      description: "关闭所有弹出面板"
      returns: "string"

    - name: "UITestHelper.ForceGameOver"
      description: "强制触发游戏结束"
      returns: "string"

    - name: "UITestHelper.SkipCurrentLevel"
      description: "跳过当前关卡"
      returns: "string"
```

---

## 安全红线

| ❌ 禁止 | ✅ 正确做法 |
|---------|-----------|
| 后门方法留在生产代码中 | `#if UNITY_EDITOR` 包裹或 Editor-only asmdef |
| 暴露 `DeleteAllSaveData()` 等破坏性方法 | 只暴露 `ResetState()` 等可控方法 |
| 后门方法依赖外部服务 | 后门只操作内存状态，不调用网络/文件/支付 |
| 后门方法有副作用累积 | 所有方法必须幂等 |
| 不写文档 | `[Description]` + config.yaml 记录 |

---

## 快速接入清单

新项目接入测试后门的步骤：

1. **创建目录**：`Assets/Scripts/Editor/TestHelpers/`
2. **创建 asmdef**：复制上面的 JSON，设置为 Editor-only
3. **复制模板**：复制 `TestHelper.cs`、`BoardTestHelper.cs` 等
4. **适配项目**：替换 `FindObjectOfType<X>()` 为项目实际类型
5. **配置 YAML**：在 `validation-config.yaml` 中声明可用方法
6. **验证安装**：在 MCP 中调用 `reflection-method-call typeName="TestHelper" methodName="ResetAll"`
