# Unity-MCP Custom Tools: Input Simulation & Recording
<!-- Unity-MCP 自定义工具：输入模拟与录制回放 -->

将此文件中的代码添加到你的 Unity 项目中，即可扩展 Unity-MCP 的输入模拟和录制回放功能。

## 安装方式

1. 在项目中创建 `Assets/Scripts/MCP/` 目录
2. 创建以下脚本文件
3. Unity-MCP 会自动识别并注册这些 Tool

---

## Tool 1: 鼠标点击模拟（世界空间）
<!-- 针对游戏世界中的点击，如点击棋盘上的棋子 -->

```csharp
// Assets/Scripts/MCP/Tool_MouseInput.cs

using UnityEngine;
using UnityEngine.InputSystem;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_MouseInput
{
    /// <summary>
    /// 模拟鼠标点击屏幕坐标（世界空间交互）
    /// Simulate mouse click at screen coordinates (world-space interaction)
    /// </summary>
    [McpPluginTool("simulate-click-world", Title = "Simulate world-space mouse click")]
    [Description("Simulate a mouse click at screen position for world-space gameplay objects. " +
                 "Use this for clicking game objects with colliders, not UI elements. " +
                 "用于点击带有碰撞体的游戏对象，不是 UI 元素。")]
    public static string SimulateClickWorld
    (
        [Description("Screen X coordinate in pixels. 屏幕 X 坐标（像素）")]
        int x,
        
        [Description("Screen Y coordinate in pixels. 屏幕 Y 坐标（像素）")]
        int y
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
            {
                return "[Error] Mouse device not available. Ensure Input System is enabled.";
            }

            // 设置鼠标位置
            Mouse.current.position.value = new Vector2(x, y);
            
            // 模拟按下和释放
            Mouse.current.leftButton.press();
            Mouse.current.leftButton.release();

            // 执行射线检测确认命中
            var ray = Camera.main.ScreenPointToRay(new Vector2(x, y));
            var hit = Physics.Raycast(ray, out var hitInfo, 100f);

            if (hit)
            {
                return $"[Success] Clicked at ({x}, {y}). Hit: {hitInfo.collider.gameObject.name}";
            }
            else
            {
                return $"[Success] Clicked at ({x}, {y}). No collider hit (raycast miss).";
            }
        });
    }

    /// <summary>
    /// 模拟鼠标按下（不释放）
    /// Simulate mouse button press without release
    /// </summary>
    [McpPluginTool("simulate-mouse-down", Title = "Simulate mouse button down")]
    [Description("Press mouse button without releasing. Useful for drag operations. " +
                 "按下鼠标按钮不释放，用于拖拽操作。")]
    public static string SimulateMouseDown
    (
        [Description("Screen X coordinate. 屏幕 X 坐标")]
        int x,
        
        [Description("Screen Y coordinate. 屏幕 Y 坐标")]
        int y,
        
        [Description("Button: 0=left, 1=right, 2=middle. 按钮：0=左键，1=右键，2=中键")]
        int button = 0
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            Mouse.current.position.value = new Vector2(x, y);
            
            var buttonControl = button == 0 ? Mouse.current.leftButton 
                          : button == 1 ? Mouse.current.rightButton 
                          : Mouse.current.middleButton;
            
            buttonControl.press();

            return $"[Success] Mouse button {button} pressed at ({x}, {y})";
        });
    }

    /// <summary>
    /// 模拟鼠标释放
    /// Simulate mouse button release
    /// </summary>
    [McpPluginTool("simulate-mouse-up", Title = "Simulate mouse button up")]
    [Description("Release previously pressed mouse button. " +
                 "释放之前按下的鼠标按钮。")]
    public static string SimulateMouseUp
    (
        [Description("Button: 0=left, 1=right, 2=middle. 按钮")]
        int button = 0
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            var buttonControl = button == 0 ? Mouse.current.leftButton 
                          : button == 1 ? Mouse.current.rightButton 
                          : Mouse.current.middleButton;
            
            buttonControl.release();

            return $"[Success] Mouse button {button} released";
        });
    }

    /// <summary>
    /// 模拟鼠标移动
    /// Simulate mouse movement
    /// </summary>
    [McpPluginTool("simulate-mouse-move", Title = "Simulate mouse movement")]
    [Description("Move mouse to screen position without clicking. " +
                 "移动鼠标到屏幕位置但不点击。")]
    public static string SimulateMouseMove
    (
        [Description("Target screen X. 目标屏幕 X")]
        int x,
        
        [Description("Target screen Y. 目标屏幕 Y")]
        int y
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            Mouse.current.position.value = new Vector2(x, y);
            return $"[Success] Mouse moved to ({x}, {y})";
        });
    }

    /// <summary>
    /// 模拟拖拽操作（从起点拖到终点）
    /// Simulate drag operation from start to end
    /// </summary>
    [McpPluginTool("simulate-drag-world", Title = "Simulate world-space drag")]
    [Description("Simulate drag from start position to end position in world-space. " +
                 "模拟从起点拖拽到终点的世界空间操作。")]
    public static string SimulateDragWorld
    (
        [Description("Start screen X. 起点屏幕 X")]
        int startX,
        
        [Description("Start screen Y. 起点屏幕 Y")]
        int startY,
        
        [Description("End screen X. 终点屏幕 X")]
        int endX,
        
        [Description("End screen Y. 终点屏幕 Y")]
        int endY,
        
        [Description("Duration in seconds. 持续时间（秒）")]
        float duration = 0.5f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            // 按下起点
            Mouse.current.position.value = new Vector2(startX, startY);
            Mouse.current.leftButton.press();

            // 等待
            System.Threading.Thread.Sleep((int)(duration * 1000));

            // 移动到终点并释放
            Mouse.current.position.value = new Vector2(endX, endY);
            Mouse.current.leftButton.release();

            return $"[Success] Dragged from ({startX}, {startY}) to ({endX}, {endY})";
        });
    }
}
```

---

## Tool 2: 鼠标点击模拟（UI 空间）
<!-- 针对 Unity UI 的点击，如按钮、滑块 -->

```csharp
// Assets/Scripts/MCP/Tool_MouseUI.cs

using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.InputSystem;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_MouseUI
{
    /// <summary>
    /// 模拟 UI 点击（通过 EventSystem）
    /// Simulate UI click via EventSystem
    /// </summary>
    [McpPluginTool("simulate-click-ui", Title = "Simulate UI button click")]
    [Description("Simulate a click on Unity UI elements (buttons, toggles, sliders). " +
                 "Uses EventSystem for proper UI interaction. " +
                 "模拟点击 Unity UI 元素（按钮、开关、滑块），使用 EventSystem。")]
    public static string SimulateClickUI
    (
        [Description("Screen X coordinate. 屏幕 X 坐标")]
        int x,
        
        [Description("Screen Y coordinate. 屏幕 Y 坐标")]
        int y
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
            {
                return "[Error] No EventSystem found. Add EventSystem to scene.";
            }

            // 创建指针事件数据
            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);

            // 执行射线检测
            var results = new System.Collections.Generic.List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
            {
                return $"[Success] Click at ({x}, {y}) - No UI element hit.";
            }

            var target = results[0].gameObject;
            
            // 模拟点击事件
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerClickHandler);

            return $"[Success] Clicked UI element: {target.name} at ({x}, {y})";
        });
    }

    /// <summary>
    /// 模拟 UI 悬停（移动鼠标到 UI 元素）
    /// Simulate UI hover
    /// </summary>
    [McpPluginTool("simulate-hover-ui", Title = "Simulate UI hover")]
    [Description("Move mouse over UI element without clicking. " +
                 "移动鼠标悬停在 UI 元素上但不点击。")]
    public static string SimulateHoverUI
    (
        [Description("Screen X. 屏幕 X")]
        int x,
        
        [Description("Screen Y. 屏幕 Y")]
        int y
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found.";

            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);

            var results = new System.Collections.Generic.List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            // 模拟进入悬停
            if (results.Count > 0)
            {
                var target = results[0].gameObject;
                ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerEnterHandler);
                return $"[Success] Hovering over: {target.name}";
            }

            // 模拟退出悬停（如果有之前悬停的对象）
            if (eventSystem.currentSelectedGameObject != null)
            {
                ExecuteEvents.Execute(eventSystem.currentSelectedGameObject, pointerData, ExecuteEvents.pointerExitHandler);
            }

            return $"[Success] Hover at ({x}, {y}) - No UI element.";
        });
    }

    /// <summary>
    /// 通过 GameObject 名称点击 UI 按钮
    /// Click UI button by GameObject name
    /// </summary>
    [McpPluginTool("click-button-by-name", Title = "Click UI button by name")]
    [Description("Find and click a UI button by its GameObject name. " +
                 "通过 GameObject 名称查找并点击 UI 按钮。")]
    public static string ClickButtonByName
    (
        [Description("Button GameObject name. 按钮对象名称")]
        string buttonName
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var button = GameObject.Find(buttonName);
            if (button == null)
            {
                return $"[Error] Button '{buttonName}' not found.";
            }

            var buttonComponent = button.GetComponent<UnityEngine.UI.Button>();
            if (buttonComponent == null)
            {
                return $"[Error] GameObject '{buttonName}' has no Button component.";
            }

            // 直接调用 onClick
            buttonComponent.onClick.Invoke();

            return $"[Success] Clicked button: {buttonName}";
        });
    }
}
```

---

## Tool 3: 键盘输入模拟
<!-- 模拟键盘按键 -->

```csharp
// Assets/Scripts/MCP/Tool_KeyboardInput.cs

using UnityEngine;
using UnityEngine.InputSystem;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_KeyboardInput
{
    /// <summary>
    /// 模拟按键按下并释放
    /// Simulate key press and release
    /// </summary>
    [McpPluginTool("simulate-key-press", Title = "Simulate keyboard key press")]
    [Description("Simulate pressing and releasing a keyboard key. " +
                 "模拟按下并释放键盘按键。")]
    public static string SimulateKeyPress
    (
        [Description("Key name (e.g. 'A', 'Space', 'Enter', 'Escape'). 按键名称")]
        string key
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard device not available.";

            var keyControl = GetKeyControl(key);
            if (keyControl == null)
                return $"[Error] Key '{key}' not recognized.";

            keyControl.press();
            keyControl.release();

            return $"[Success] Key '{key}' pressed and released";
        });
    }

    /// <summary>
    /// 模拟按键按下（不释放）
    /// Simulate key down
    /// </summary>
    [McpPluginTool("simulate-key-down", Title = "Simulate key down")]
    [Description("Press key without releasing. " +
                 "按下按键不释放。")]
    public static string SimulateKeyDown
    (
        [Description("Key name. 按键名称")]
        string key
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard device not available.";

            var keyControl = GetKeyControl(key);
            if (keyControl == null)
                return $"[Error] Key '{key}' not recognized.";

            keyControl.press();
            return $"[Success] Key '{key}' pressed (held)";
        });
    }

    /// <summary>
    /// 模拟按键释放
    /// Simulate key up
    /// </summary>
    [McpPluginTool("simulate-key-up", Title = "Simulate key up")]
    [Description("Release previously pressed key. " +
                 "释放之前按下的按键。")]
    public static string SimulateKeyUp
    (
        [Description("Key name. 按键名称")]
        string key
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard device not available.";

            var keyControl = GetKeyControl(key);
            if (keyControl == null)
                return $"[Error] Key '{key}' not recognized.";

            keyControl.release();
            return $"[Success] Key '{key}' released";
        });
    }

    /// <summary>
    /// 模拟文本输入
    /// Simulate text input
    /// </summary>
    [McpPluginTool("simulate-text-input", Title = "Simulate text input")]
    [Description("Simulate typing a string of text. " +
                 "模拟输入一段文本。")]
    public static string SimulateTextInput
    (
        [Description("Text to type. 要输入的文本")]
        string text,
        
        [Description("Delay between keys in ms. 每个按键间隔（毫秒）")]
        int delayMs = 50
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard device not available.";

            foreach (char c in text)
            {
                var keyControl = GetKeyControlFromChar(c);
                if (keyControl != null)
                {
                    keyControl.press();
                    System.Threading.Thread.Sleep(delayMs);
                    keyControl.release();
                    System.Threading.Thread.Sleep(delayMs);
                }
            }

            return $"[Success] Typed text: '{text}'";
        });
    }

    // 辅助方法：获取按键控制
    private static KeyControl GetKeyControl(string keyName)
    {
        var keyboard = Keyboard.current;
        
        // 尝试直接匹配
        if (keyboard[keyName.ToLower()] != null)
            return keyboard[keyName.ToLower()] as KeyControl;

        // 特殊键名映射
        switch (keyName.ToLower())
        {
            case "space": return keyboard.spaceKey;
            case "enter": return keyboard.enterKey;
            case "return": return keyboard.enterKey;
            case "escape": return keyboard.escapeKey;
            case "esc": return keyboard.escapeKey;
            case "tab": return keyboard.tabKey;
            case "backspace": return keyboard.backspaceKey;
            case "delete": return keyboard.deleteKey;
            case "shift": return keyboard.leftShiftKey;
            case "ctrl": return keyboard.leftCtrlKey;
            case "alt": return keyboard.leftAltKey;
            case "arrowup": return keyboard.upArrowKey;
            case "arrowdown": return keyboard.downArrowKey;
            case "arrowleft": return keyboard.leftArrowKey;
            case "arrowright": return keyboard.rightArrowKey;
            default: return null;
        }
    }

    // 辅助方法：从字符获取按键
    private static KeyControl GetKeyControlFromChar(char c)
    {
        var keyboard = Keyboard.current;
        
        if (c >= 'a' && c <= 'z')
            return keyboard[(Key)((int)Key.A + (c - 'a'))];
        if (c >= 'A' && c <= 'Z')
            return keyboard[(Key)((int)Key.A + (c - 'A'))];
        if (c >= '0' && c <= '9')
            return keyboard[(Key)((int)Key.Digit0 + (c - '0'))];
        if (c == ' ')
            return keyboard.spaceKey;
        
        return null;
    }
}
```

---

## Tool 4: 输入录制与回放
<!-- 录制和回放输入序列 -->

```csharp
// Assets/Scripts/MCP/Tool_InputRecording.cs

using UnityEngine;
using UnityEngine.InputSystem;
using System.Collections.Generic;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_InputRecording
{
    // 录制数据存储
    private static List<InputEvent> recordedEvents = new List<InputEvent>();
    private static bool isRecording = false;
    private static float recordStartTime = 0f;

    /// <summary>
    /// 输入事件结构
    /// </summary>
    private struct InputEvent
    {
        public float time;          // 相对于录制开始的时间
        public string type;         // "click", "key", "move"
        public int x, y;            // 鼠标位置
        public string key;          // 按键名称
        public bool isDown;         // 是否按下（vs释放）
    }

    /// <summary>
    /// 开始录制输入
    /// Start recording input
    /// </summary>
    [McpPluginTool("record-start", Title = "Start recording input")]
    [Description("Start recording all mouse and keyboard inputs. " +
                 "开始录制所有鼠标和键盘输入。")]
    public static string StartRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            recordedEvents.Clear();
            isRecording = true;
            recordStartTime = Time.time;

            // 注册输入回调
            if (Mouse.current != null)
            {
                // 鼠标位置变化
                // 注意：Input System 需要通过 polling 或事件监听
            }

            return "[Success] Recording started. Use 'record-stop' to stop and 'replay-input' to replay.";
        });
    }

    /// <summary>
    /// 停止录制输入
    /// Stop recording input
    /// </summary>
    [McpPluginTool("record-stop", Title = "Stop recording input")]
    [Description("Stop recording and return recorded event count. " +
                 "停止录制并返回录制的事件数量。")]
    public static string StopRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            isRecording = false;
            return $"[Success] Recording stopped. Recorded {recordedEvents.Count} events.";
        });
    }

    /// <summary>
    /// 获取录制摘要
    /// Get recording summary
    /// </summary>
    [McpPluginTool("record-summary", Title = "Get recording summary")]
    [Description("Get summary of recorded events. " +
                 "获取录制事件的摘要。")]
    public static string GetRecordSummary()
    {
        return MainThread.Instance.Run(() =>
        {
            if (recordedEvents.Count == 0)
                return "[Info] No events recorded.";

            var sb = new System.Text.StringBuilder();
            sb.AppendLine($"Total events: {recordedEvents.Count}");
            sb.AppendLine($"Duration: {recordedEvents[recordedEvents.Count - 1].time:F2}s");
            
            // 统计事件类型
            int clicks = 0, keys = 0, moves = 0;
            foreach (var e in recordedEvents)
            {
                if (e.type == "click") clicks++;
                else if (e.type == "key") keys++;
                else if (e.type == "move") moves++;
            }
            
            sb.AppendLine($"Clicks: {clicks}, Keys: {keys}, Moves: {moves}");
            
            // 列出关键事件
            sb.AppendLine("\nKey events:");
            foreach (var e in recordedEvents)
            {
                if (e.type == "click")
                    sb.AppendLine($"  [{e.time:F2}s] Click ({e.x}, {e.y})");
                else if (e.type == "key")
                    sb.AppendLine($"  [{e.time:F2}s] Key {e.key} {(e.isDown ? "down" : "up")}");
            }

            return sb.ToString();
        });
    }

    /// <summary>
    /// 回放录制的输入
    /// Replay recorded input
    /// </summary>
    [McpPluginTool("replay-input", Title = "Replay recorded input")]
    [Description("Replay all recorded input events with original timing. " +
                 "按原始时间回放所有录制的输入事件。")]
    public static string ReplayInput
    (
        [Description("Speed multiplier (1=normal, 2=fast, 0.5=slow). 速度倍率")]
        float speed = 1f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (recordedEvents.Count == 0)
                return "[Error] No events to replay. Record first with 'record-start'.";

            if (isRecording)
                return "[Error] Still recording. Stop with 'record-stop' first.";

            float replayStartTime = Time.time;
            int replayedCount = 0;

            foreach (var event_ in recordedEvents)
            {
                // 计算等待时间
                float targetTime = replayStartTime + (event_.time / speed);
                float waitTime = targetTime - Time.time;
                
                if (waitTime > 0)
                    System.Threading.Thread.Sleep((int)(waitTime * 1000));

                // 执行事件
                if (event_.type == "click")
                {
                    Mouse.current.position.value = new Vector2(event_.x, event_.y);
                    if (event_.isDown)
                        Mouse.current.leftButton.press();
                    else
                        Mouse.current.leftButton.release();
                }
                else if (event_.type == "key")
                {
                    // 需要按键查找逻辑
                    // SimulateKeyPress(event_.key);
                }
                else if (event_.type == "move")
                {
                    Mouse.current.position.value = new Vector2(event_.x, event_.y);
                }

                replayedCount++;
            }

            return $"[Success] Replayed {replayedCount} events at {speed}x speed.";
        });
    }

    /// <summary>
    /// 手动添加点击事件（用于手动录制）
    /// Manually add click event
    /// </summary>
    [McpPluginTool("record-add-click", Title = "Add click to recording")]
    [Description("Manually add a click event to current recording. " +
                 "手动添加点击事件到当前录制中。")]
    public static string RecordAddClick
    (
        [Description("Screen X. 屏幕 X")]
        int x,
        
        [Description("Screen Y. 屏幕 Y")]
        int y,
        
        [Description("Is button down (true) or up (false). 是否按下")]
        bool isDown = true
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (!isRecording)
                return "[Error] Not recording. Start with 'record-start'.";

            recordedEvents.Add(new InputEvent
            {
                time = Time.time - recordStartTime,
                type = "click",
                x = x,
                y = y,
                isDown = isDown
            });

            return $"[Success] Added click ({x}, {y}) at time {Time.time - recordStartTime:F2}s";
        });
    }

    /// <summary>
    /// 手动添加按键事件（用于手动录制）
    /// Manually add key event
    /// </summary>
    [McpPluginTool("record-add-key", Title = "Add key to recording")]
    [Description("Manually add a key event to current recording. " +
                 "手动添加按键事件到当前录制中。")]
    public static string RecordAddKey
    (
        [Description("Key name. 按键名称")]
        string key,
        
        [Description("Is key down (true) or up (false). 是否按下")]
        bool isDown = true
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (!isRecording)
                return "[Error] Not recording. Start with 'record-start'.";

            recordedEvents.Add(new InputEvent
            {
                time = Time.time - recordStartTime,
                type = "key",
                key = key,
                isDown = isDown
            });

            return $"[Success] Added key '{key}' at time {Time.time - recordStartTime:F2}s";
        });
    }

    /// <summary>
    /// 清除录制数据
    /// Clear recording data
    /// </summary>
    [McpPluginTool("record-clear", Title = "Clear recording")]
    [Description("Clear all recorded input events. " +
                 "清除所有录制的输入事件。")]
    public static string ClearRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            recordedEvents.Clear();
            return "[Success] Recording cleared.";
        });
    }

    /// <summary>
    /// 导出录制数据为 JSON
    /// Export recording as JSON
    /// </summary>
    [McpPluginTool("record-export", Title = "Export recording as JSON")]
    [Description("Export recorded events as JSON string for saving. " +
                 "导出录制事件为 JSON 字符串用于保存。")]
    public static string ExportRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            if (recordedEvents.Count == 0)
                return "[Info] No events to export.";

            var json = UnityEngine.JsonUtility.ToJson(new RecordingData { events = recordedEvents.ToArray() });
            return $"[Success] Exported {recordedEvents.Count} events.\nJSON: {json}";
        });
    }

    /// <summary>
    /// 从 JSON 导入录制数据
    /// Import recording from JSON
    /// </summary>
    [McpPluginTool("record-import", Title = "Import recording from JSON")]
    [Description("Import recorded events from JSON string. " +
                 "从 JSON 字符串导入录制事件。")]
    public static string ImportRecording
    (
        [Description("JSON data. JSON 数据")]
        string json
    )
    {
        return MainThread.Instance.Run(() =>
        {
            try
            {
                var data = UnityEngine.JsonUtility.FromJson<RecordingData>(json);
                recordedEvents.Clear();
                foreach (var e in data.events)
                    recordedEvents.Add(e);

                return $"[Success] Imported {recordedEvents.Count} events.";
            }
            catch (System.Exception ex)
            {
                return $"[Error] Failed to import: {ex.Message}";
            }
        });
    }

    [System.Serializable]
    private class RecordingData
    {
        public InputEvent[] events;
    }
}
```

---

## Tool 5: 自动录制监听（可选）
<!-- 自动监听并录制输入 -->

```csharp
// Assets/Scripts/MCP/InputRecorder.cs
// 这是一个 MonoBehaviour，用于自动监听输入并录制

using UnityEngine;
using UnityEngine.InputSystem;
using System.Collections.Generic;

public class InputRecorder : MonoBehaviour
{
    public static InputRecorder Instance { get; private set; }

    private List<InputEvent> events = new List<InputEvent>();
    private bool isRecording = false;
    private float startTime;

    private void Awake()
    {
        Instance = this;
    }

    public void StartRecording()
    {
        events.Clear();
        isRecording = true;
        startTime = Time.time;
    }

    public void StopRecording()
    {
        isRecording = false;
    }

    public List<InputEvent> GetEvents() => events;

    private void Update()
    {
        if (!isRecording) return;

        float currentTime = Time.time - startTime;

        // 监听鼠标
        if (Mouse.current != null)
        {
            var pos = Mouse.current.position.value;
            
            // 监听左键
            if (Mouse.current.leftButton.wasPressedThisFrame)
            {
                events.Add(new InputEvent
                {
                    time = currentTime,
                    type = "click",
                    x = (int)pos.x,
                    y = (int)pos.y,
                    isDown = true
                });
            }
            if (Mouse.current.leftButton.wasReleasedThisFrame)
            {
                events.Add(new InputEvent
                {
                    time = currentTime,
                    type = "click",
                    x = (int)pos.x,
                    y = (int)pos.y,
                    isDown = false
                });
            }
        }

        // 监听键盘（需要遍历所有按键）
        if (Keyboard.current != null)
        {
            // 可以扩展监听更多按键
        }
    }

    [System.Serializable]
    public struct InputEvent
    {
        public float time;
        public string type;
        public int x, y;
        public string key;
        public bool isDown;
    }
}
```

---

## 使用示例

### 示例 1：点击游戏世界中的对象

```
# Unity-MCP Tool 调用
simulate-click-world x=500 y=300

# 返回
[Success] Clicked at (500, 300). Hit: Gem_Red_01
```

### 示例 2：点击 UI 按钮

```
# 通过坐标
simulate-click-ui x=200 y=150

# 或通过名称
click-button-by-name buttonName="RestartButton"
```

### 示例 3：录制并回放输入序列

```
# 开始录制
record-start

# 手动添加事件（或在游戏中自动录制）
record-add-click x=100 y=200 isDown=true
record-add-click x=100 y=200 isDown=false
record-add-click x=300 y=400 isDown=true
record-add-click x=300 y=400 isDown=false

# 停止录制
record-stop

# 查看摘要
record-summary

# 回放
replay-input speed=1.0
```

---

## 注意事项

1. **Input System 配置**
   - 确保 Unity Input System 已启用
   - 项目设置中勾选 "Active Input Handling" = "Input System Package (New or Both)"

2. **EventSystem**
   - UI 点击需要场景中有 EventSystem
   - 如果没有，会返回错误提示

3. **MainThread**
   - 所有 Unity API 调用必须通过 `MainThread.Instance.Run()`
   - 这是 Unity-MCP 的要求

4. **时间精度**
   - 回放的时间精度取决于 Unity 的帧率
   - 高精度场景可能需要调整等待逻辑

---

## 文件结构建议

```
Assets/Scripts/MCP/
├── Tool_MouseInput.cs       # 世界空间鼠标输入
├── Tool_MouseUI.cs          # UI 空间鼠标输入
├── Tool_KeyboardInput.cs    # 键盘输入
├── Tool_InputRecording.cs   # 录制与回放
└── InputRecorder.cs         # 自动录制监听器（可选 MonoBehaviour）
```

将这些文件放入项目后，Unity-MCP 会自动识别并注册这些 Tool，LLM 即可通过 MCP 调用它们。