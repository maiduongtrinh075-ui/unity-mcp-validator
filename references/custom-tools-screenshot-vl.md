# Unity-MCP Custom Tools: Screenshot + VL Analysis
<!-- Unity-MCP 自定义工具：截图 + 视觉语言模型分析 -->

集成 Ollama VL 模型进行截图识别，实现自动化视觉验证。

**使用场景：**
- 判断 UI 状态（按钮是否可见、面板是否打开）
- 检测游戏画面（消除是否完成、角色位置）
- 验证视觉效果（动画是否播放、特效是否显示）

## 安装方式

1. 在项目中创建或确认 `Assets/Scripts/MCP/` 目录
2. 创建 `Tool_ScreenshotVL.cs` 脚本文件
3. Unity-MCP 会自动识别并注册

---

## Tool: 截图 + VL 分析

```csharp
// Assets/Scripts/MCP/Tool_ScreenshotVL.cs

using UnityEngine;
using System;
using System.IO;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_ScreenshotVL
{
    // Ollama 配置（可在 validation-config.yaml 中自定义）
    private static string ollamaUrl = "http://192.168.0.103:11434";
    private static string ollamaModel = "huihui_ai/qwen3-vl-abliterated:8b-instruct";
    private static int timeoutSeconds = 30;

    /// <summary>
    /// 配置 Ollama 连接（可选，用于更改默认配置）
    /// Configure Ollama connection
    /// </summary>
    [McpPluginTool("vl-config", Title = "Configure VL service")]
    [Description("Configure Ollama URL and model for screenshot analysis. " +
                 "配置 Ollama 地址和模型。")]
    public static string ConfigureVL
    (
        [Description("Ollama API URL. Ollama API 地址")]
        string url = null,
        
        [Description("VL model name. VL 模型名称")]
        string model = null,
        
        [Description("Timeout in seconds. 超时时间")]
        int timeout = 30
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (!string.IsNullOrEmpty(url))
                ollamaUrl = url;
            if (!string.IsNullOrEmpty(model))
                ollamaModel = model;
            timeoutSeconds = timeout;

            return $"[Success] VL config updated:\n" +
                   $"  URL: {ollamaUrl}\n" +
                   $"  Model: {ollamaModel}\n" +
                   $"  Timeout: {timeoutSeconds}s";
        });
    }

    /// <summary>
    /// 截图并发送到 VL 模型分析
    /// Capture screenshot and analyze with VL model
    /// </summary>
    [McpPluginTool("screenshot-analyze-vl", Title = "Screenshot + VL analysis")]
    [Description("Capture screenshot and send to Ollama VL model for analysis. " +
                 "截图并发送到 Ollama VL 模型进行分析。")]
    public static string ScreenshotAnalyzeVL
    (
        [Description("Analysis prompt. Example: 'Is the StartButton visible?', 'Describe the game state'. 分析提示")]
        string prompt = "Describe what you see in this game screenshot.",
        
        [Description("Save screenshot to file (optional). 保存截图路径")]
        string savePath = null,
        
        [Description("Image format: 'png' or 'jpg'. 图片格式")]
        string format = "png"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            // 1. 截图
            Texture2D screenshot = CaptureScreenshot();
            if (screenshot == null)
                return "[Error] Failed to capture screenshot.";

            // 2. 编码为 base64
            byte[] imageBytes = format == "jpg" 
                ? screenshot.EncodeToJPG(90) 
                : screenshot.EncodeToPNG();
            string base64Image = Convert.ToBase64String(imageBytes);

            // 3. 保存截图（可选）
            string savedPath = "";
            if (!string.IsNullOrEmpty(savePath))
            {
                try
                {
                    File.WriteAllBytes(savePath, imageBytes);
                    savedPath = $"Screenshot saved to: {savePath}";
                }
                catch (Exception ex)
                {
                    savedPath = $"Save failed: {ex.Message}";
                }
            }

            // 4. 调用 Ollama VL API
            string analysisResult = CallOllamaVL(base64Image, prompt);

            // 清理
            UnityEngine.Object.DestroyImmediate(screenshot);

            // 5. 返回结果
            return $"[Success] Screenshot analyzed.\n" +
                   $"Size: {imageBytes.Length / 1024}KB\n" +
                   $"{savedPath}\n\n" +
                   $"=== VL Analysis ===\n{analysisResult}";
        });
    }

    /// <summary>
    /// 仅截图（不分析）
    /// Capture screenshot only (no VL analysis)
    /// </summary>
    [McpPluginTool("screenshot-save", Title = "Save screenshot")]
    [Description("Capture and save screenshot to file. " +
                 "截图并保存到文件。")]
    public static string ScreenshotSave
    (
        [Description("Save path (absolute or relative to project). 保存路径")]
        string savePath = "screenshot.png",
        
        [Description("Image format. 图片格式")]
        string format = "png"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            Texture2D screenshot = CaptureScreenshot();
            if (screenshot == null)
                return "[Error] Failed to capture screenshot.";

            byte[] imageBytes = format == "jpg" 
                ? screenshot.EncodeToJPG(90) 
                : screenshot.EncodeToPNG();

            try
            {
                // 如果路径不是绝对路径，保存到项目根目录
                if (!Path.IsPathRooted(savePath))
                    savePath = Path.Combine(Application.dataPath, "..", savePath);

                File.WriteAllBytes(savePath, imageBytes);
                UnityEngine.Object.DestroyImmediate(screenshot);
                return $"[Success] Screenshot saved: {savePath}\n" +
                       $"Size: {imageBytes.Length / 1024}KB";
            }
            catch (Exception ex)
            {
                UnityEngine.Object.DestroyImmediate(screenshot);
                return $"[Error] Save failed: {ex.Message}";
            }
        });
    }

    /// <summary>
    /// 快速 UI 状态检查（使用预设 prompt）
    /// Quick UI state check with preset prompts
    /// </summary>
    [McpPluginTool("ui-check-vl", Title = "Check UI state with VL")]
    [Description("Quick check for specific UI element visibility or state. " +
                 "快速检查特定 UI 元素的可见性或状态。")]
    public static string UICheckVL
    (
        [Description("UI element to check. 要检查的 UI 元素")]
        string elementName,
        
        [Description("Check type: 'visible', 'active', 'text', 'color'. 检查类型")]
        string checkType = "visible"
    )
    {
        string prompt = "";
        switch (checkType.ToLower())
        {
            case "visible":
                prompt = $"Is the UI element '{elementName}' visible on screen? Answer YES or NO, and briefly explain.";
                break;
            case "active":
                prompt = $"Is the UI element '{elementName}' active/enabled? Answer YES or NO.";
                break;
            case "text":
                prompt = $"What text is displayed on the UI element '{elementName}'? Return the exact text.";
                break;
            case "color":
                prompt = $"What is the color of the UI element '{elementName}'? Describe briefly.";
                break;
            default:
                prompt = $"Describe the state of UI element '{elementName}'.";
                break;
        }

        return ScreenshotAnalyzeVL(prompt, null, "png");
    }

    /// <summary>
    /// 游戏状态检查
    /// Game state check with VL
    /// </summary>
    [McpPluginTool("game-state-vl", Title = "Check game state with VL")]
    [Description("Analyze current game state (score, phase, animation, etc). " +
                 "分析当前游戏状态。")]
    public static string GameStateVL
    (
        [Description("What to check: 'score', 'phase', 'animation', 'general'. 检查内容")]
        string checkType = "general"
    )
    {
        string prompt = "";
        switch (checkType.ToLower())
        {
            case "score":
                prompt = "What is the current score displayed? Return the exact number.";
                break;
            case "phase":
                prompt = "What game phase is currently shown? (e.g., menu, playing, paused, game-over)";
                break;
            case "animation":
                prompt = "Is any animation currently playing? Describe what you see.";
                break;
            default:
                prompt = "Describe the current game state: score, phase, visible UI elements, any animations.";
                break;
        }

        return ScreenshotAnalyzeVL(prompt, null, "png");
    }

    /// <summary>
    /// 对比两张截图（检测变化）
    /// Compare two screenshots for changes
    /// </summary>
    [McpPluginTool("screenshot-compare-vl", Title = "Compare screenshots")]
    [Description("Compare two screenshots and describe differences. " +
                 "对比两张截图并描述差异。")]
    public static string ScreenshotCompareVL
    (
        [Description("First screenshot path. 第一张截图路径")]
        string beforePath,
        
        [Description("Second screenshot path. 第二张截图路径")]
        string afterPath,
        
        [Description("What to focus on. 关注点")]
        string focusOn = "any changes"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            try
            {
                byte[] beforeBytes = File.ReadAllBytes(beforePath);
                byte[] afterBytes = File.ReadAllBytes(afterPath);
                string beforeBase64 = Convert.ToBase64String(beforeBytes);
                string afterBase64 = Convert.ToBase64String(afterBytes);

                // 调用 VL 模型对比两张图
                string prompt = $"Compare these two screenshots. Focus on: {focusOn}. " +
                               "Describe the differences between before and after.";

                string result = CallOllamaVLCompare(beforeBase64, afterBase64, prompt);
                
                return $"[Success] Comparison result:\n{result}";
            }
            catch (Exception ex)
            {
                return $"[Error] {ex.Message}";
            }
        });
    }

    // ========================================
    // 内部方法
    // ========================================

    private static Texture2D CaptureScreenshot()
    {
        // 等待帧结束以确保 UI 渲染完成
        // 注意：在 MCP 工具中可能需要不同方式

        int width = Screen.width;
        int height = Screen.height;

        Texture2D screenshot = new Texture2D(width, height, TextureFormat.RGB24, false);
        
        // 读取屏幕像素
        screenshot.ReadPixels(new Rect(0, 0, width, height), 0, 0);
        screenshot.Apply();

        return screenshot;
    }

    private static string CallOllamaVL(string base64Image, string prompt)
    {
        try
        {
            // 构建 Ollama API 请求
            // POST /api/generate
            var requestBody = new
            {
                model = ollamaModel,
                prompt = prompt,
                images = new[] { base64Image },
                stream = false
            };

            string json = Newtonsoft.Json.JsonConvert.SerializeObject(requestBody);

            using (var client = new HttpClient())
            {
                client.Timeout = TimeSpan.FromSeconds(timeoutSeconds);
                
                var content = new StringContent(json, Encoding.UTF8, "application/json");
                var response = client.PostAsync($"{ollamaUrl}/api/generate", content).Result;

                if (!response.IsSuccessStatusCode)
                {
                    return $"[Error] Ollama API failed: {response.StatusCode}";
                }

                string responseJson = response.Content.ReadAsStringAsync().Result;
                var result = Newtonsoft.Json.JsonConvert.DeserializeObject<OllamaResponse>(responseJson);

                return result?.response ?? "[Error] No response from VL model.";
            }
        }
        catch (TaskCanceledException)
        {
            return "[Error] VL request timed out.";
        }
        catch (Exception ex)
        {
            return $"[Error] VL call failed: {ex.Message}";
        }
    }

    private static string CallOllamaVLCompare(string base641, string base64_2, string prompt)
    {
        try
        {
            var requestBody = new
            {
                model = ollamaModel,
                prompt = prompt,
                images = new[] { base641, base64_2 },
                stream = false
            };

            string json = Newtonsoft.Json.JsonConvert.SerializeObject(requestBody);

            using (var client = new HttpClient())
            {
                client.Timeout = TimeSpan.FromSeconds(timeoutSeconds);
                
                var content = new StringContent(json, Encoding.UTF8, "application/json");
                var response = client.PostAsync($"{ollamaUrl}/api/generate", content).Result;

                if (!response.IsSuccessStatusCode)
                    return $"[Error] Ollama API failed: {response.StatusCode}";

                string responseJson = response.Content.ReadAsStringAsync().Result;
                var result = Newtonsoft.Json.JsonConvert.DeserializeObject<OllamaResponse>(responseJson);

                return result?.response ?? "[Error] No response.";
            }
        }
        catch (Exception ex)
        {
            return $"[Error] {ex.Message}";
        }
    }

    private class OllamaResponse
    {
        public string model;
        public string response;
        public bool done;
    }
}
```

---

## 配置文件支持

在 `validation-config.yaml` 中添加 VL 配置：

```yaml
# VL (Visual Language) 服务配置
vl_service:
  enabled: true
  url: "http://192.168.0.103:11434"
  model: "huihui_ai/qwen3-vl-abliterated:8b-instruct"
  timeout_seconds: 30
  
  # 预设 prompt 模板
  prompt_templates:
    ui_visible: "Is the UI element '{element}' visible? Answer YES or NO."
    score_check: "What is the current score? Return only the number."
    game_phase: "What game phase is shown? (menu/playing/paused/game-over)"
```

---

## 使用示例

### 1. 基本截图分析

```
screenshot-analyze-vl prompt="Describe the current game state"
```

输出：
```
[Success] Screenshot analyzed.
Size: 245KB

=== VL Analysis ===
The screenshot shows a match-3 puzzle game. The game board has 64 gems 
arranged in a 8x8 grid. The score displays "150" at the top. The game 
is in playing phase with no animations visible.
```

### 2. 快速 UI 检查

```
ui-check-vl elementName="StartButton" checkType="visible"
```

输出：
```
=== VL Analysis ===
YES - The StartButton is visible in the center of the screen, 
showing green text "START".
```

### 3. 游戏状态检查

```
game-state-vl checkType="score"
```

输出：
```
=== VL Analysis ===
150
```

### 4. 截图对比

```
# 先保存两张截图
screenshot-save savePath="before.png"
simulate-click-ui x=400 y=300
screenshot-save savePath="after.png"

# 对比
screenshot-compare-vl beforePath="before.png" afterPath="after.png" focusOn="score change"
```

---

## 工具对比

| 工具 | 功能 | 输出 |
|------|------|------|
| `screenshot-analyze-vl` | 截图 + 自定义 prompt 分析 | VL 描述文本 |
| `ui-check-vl` | 快速检查 UI 元素状态 | YES/NO 或具体值 |
| `game-state-vl` | 检查游戏状态 | 状态描述 |
| `screenshot-compare-vl` | 对比两张截图变化 | 差异描述 |
| `screenshot-save` | 仅截图保存 | 文件路径 |
| `vl-config` | 配置 Ollama 连接 | 配置确认 |

---

## 注意事项

1. **依赖 Newtonsoft.Json** — Unity 项目需要安装 Newtonsoft.Json 包
   ```
   Package Manager → Add package from git URL:
   com.unity.nuget.newtonsoft-json
   ```

2. **网络延迟** — VL 分析需要几秒钟，不适合高频调用

3. **PlayMode 必需** — 截图工具需要在 PlayMode 中使用

4. **模型名称** — 使用完整名称 `huihui_ai/qwen3-vl-abliterated:8b-instruct`

---

## 与现有流程集成

```
# Layer 4C 证据收集 + VL 分析
screenshot-game-view → 截图保存
screenshot-analyze-vl prompt="验证消除动画是否完成" → VL 分析
reflection-method-call → 数据验证

# 双轨验证：数据 + 视觉
```