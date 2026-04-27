# Unity-MCP Custom Tools: Input Simulation & Recording (v2.1)
<!-- Unity-MCP 自定义工具：输入模拟与录制回放 -->

将此文件中的代码添加到你的 Unity 项目中，即可扩展 Unity-MCP 的输入模拟和录制回放功能。

**v2.1 关键改进（参考 unity-cli-loop）：**
- ✅ **UI 分步 Drag**：DragStart/DragMove/DragEnd 解决一次性拖拽不稳定问题
- ✅ **UI 元素定位**：`ui-find-interactive-elements` 返回可交互元素坐标，解决"找不到 draggable"
- ✅ **LongPress 支持**：长按 UI 元素
- ✅ **完善错误处理**：检查元素类型、可拖拽性、EventSystem 状态

## 安装方式

1. 在项目中创建 `Assets/Scripts/MCP/` 目录
2. 创建以下脚本文件
3. Unity-MCP 会自动识别并注册这些 Tool

---

## Tool 1: UI 元素定位（关键工具）
<!-- 解决坐标不准、找不到元素的问题 -->

```csharp
// Assets/Scripts/MCP/Tool_UIElementFinder.cs

using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;
using System.Collections.Generic;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_UIElementFinder
{
    /// <summary>
    /// 查找屏幕上所有可交互 UI 元素，返回坐标信息
    /// Find all interactive UI elements on screen with coordinates
    /// </summary>
    [McpPluginTool("ui-find-interactive-elements", Title = "Find interactive UI elements")]
    [Description("Find all clickable/draggable UI elements and return their screen coordinates. " +
                 "Use this BEFORE simulate-click-ui or simulate-drag-ui to get accurate coordinates. " +
                 "查找所有可点击/可拖拽的 UI 元素并返回屏幕坐标，解决坐标不准问题。")]
    public static string FindInteractiveElements
    (
        [Description("Include inactive elements. 是否包含隐藏元素")]
        bool includeInactive = false,
        
        [Description("Output format: 'json' or 'summary'. 输出格式")]
        string format = "summary"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found. Add EventSystem to scene.";

            // 查找所有 Canvas
            var canvases = Object.FindObjectsByType<Canvas>(FindObjectsSortMode.None);
            if (canvases.Length == 0)
                return "[Error] No Canvas found in scene.";

            var elements = new List<UIElementInfo>();
            
            foreach (var canvas in canvases)
            {
                if (!canvas.gameObject.activeInHierarchy && !includeInactive)
                    continue;

                // 遍历 Canvas 下的所有 UI 元素
                FindElementsRecursive(canvas.transform, elements, includeInactive);
            }

            if (elements.Count == 0)
                return "[Info] No interactive UI elements found.";

            // 按层级排序（最前面的优先）
            elements.Sort((a, b) => b.SortingOrder.CompareTo(a.SortingOrder));

            // 添加标签 (A, B, C...)
            for (int i = 0; i < elements.Count; i++)
            {
                elements[i].Label = GetLabel(i);
            }

            if (format == "json")
            {
                var json = Newtonsoft.Json.JsonConvert.SerializeObject(elements);
                return $"[Success] Found {elements.Count} elements.\n{json}";
            }
            else
            {
                var sb = new System.Text.StringBuilder();
                sb.AppendLine($"Found {elements.Count} interactive UI elements:");
                sb.AppendLine("(A = frontmost, use SimX/SimY for simulate-mouse-ui)");
                sb.AppendLine("");
                
                foreach (var elem in elements)
                {
                    sb.AppendLine($"{elem.Label}: {elem.Name}");
                    sb.AppendLine($"  Type: {elem.Type}");
                    sb.AppendLine($"  Position: SimX={elem.SimX}, SimY={elem.SimY}");
                    sb.AppendLine($"  Bounds: ({elem.BoundsMinX},{elem.BoundsMinY}) to ({elem.BoundsMaxX},{elem.BoundsMaxY})");
                    sb.AppendLine($"  SortingOrder: {elem.SortingOrder}");
                    sb.AppendLine("");
                }
                
                return sb.ToString();
            }
        });
    }

    /// <summary>
    /// 查找指定名称的 UI 元素
    /// Find UI element by name
    /// </summary>
    [McpPluginTool("ui-find-element-by-name", Title = "Find UI element by name")]
    [Description("Find a specific UI element by GameObject name and return its coordinates. " +
                 "通过 GameObject 名称查找特定 UI 元素并返回坐标。")]
    public static string FindElementByName
    (
        [Description("GameObject name. 对象名称")]
        string name,
        
        [Description("Include inactive. 是否包含隐藏元素")]
        bool includeInactive = false
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var go = GameObject.Find(name);
            if (go == null)
            {
                // 尝试在所有 Canvas 下查找
                var canvases = Object.FindObjectsByType<Canvas>(FindObjectsSortMode.None);
                foreach (var canvas in canvases)
                {
                    var found = FindChildByName(canvas.transform, name);
                    if (found != null)
                    {
                        go = found.gameObject;
                        break;
                    }
                }
            }

            if (go == null)
                return $"[Error] UI element '{name}' not found.";

            // 检查是否是 UI 元素
            var rectTransform = go.GetComponent<RectTransform>();
            if (rectTransform == null)
                return $"[Error] '{name}' has no RectTransform (not a UI element).";

            // 计算屏幕坐标
            Vector3[] corners = new Vector3[4];
            rectTransform.GetWorldCorners(corners);
            
            // 转换为屏幕坐标
            var canvas = go.GetComponentInParent<Canvas>();
            Vector2 center = RectTransformUtility.WorldToScreenPoint(canvas.worldCamera, rectTransform.position);
            
            // 获取元素类型
            string type = GetElementType(go);
            
            return $"[Success] Found '{name}'\n" +
                   $"Type: {type}\n" +
                   $"SimX: {(int)center.x}, SimY: {(int)center.y}\n" +
                   $"Bounds: ({(int)corners[0].x},{(int)corners[0].y}) to ({(int)corners[2].x},{(int)corners[2].y})\n" +
                   $"Active: {go.activeInHierarchy}";
        });
    }

    // 递归查找元素
    private static void FindElementsRecursive(Transform parent, List<UIElementInfo> elements, bool includeInactive)
    {
        foreach (Transform child in parent)
        {
            if (!child.gameObject.activeInHierarchy && !includeInactive)
                continue;

            // 检查是否是可交互元素
            var selectable = child.GetComponent<Selectable>();
            var dragHandler = child.GetComponent<IDragHandler>();
            var clickHandler = child.GetComponent<IPointerClickHandler>();
            
            if (selectable != null || dragHandler != null || clickHandler != null)
            {
                var rectTransform = child.GetComponent<RectTransform>();
                if (rectTransform != null)
                {
                    var canvas = child.GetComponentInParent<Canvas>();
                    Vector2 center = RectTransformUtility.WorldToScreenPoint(
                        canvas?.worldCamera ?? null, 
                        rectTransform.position
                    );
                    
                    Vector3[] corners = new Vector3[4];
                    rectTransform.GetWorldCorners(corners);

                    elements.Add(new UIElementInfo
                    {
                        Name = child.name,
                        Type = GetElementType(child.gameObject),
                        SimX = (int)center.x,
                        SimY = (int)center.y,
                        BoundsMinX = (int)corners[0].x,
                        BoundsMinY = (int)corners[0].y,
                        BoundsMaxX = (int)corners[2].x,
                        BoundsMaxY = (int)corners[2].y,
                        SortingOrder = canvas?.sortingOrder ?? 0,
                        Interactable = selectable?.interactable ?? true,
                        IsDraggable = dragHandler != null
                    });
                }
            }

            // 递归查找子元素
            FindElementsRecursive(child, elements, includeInactive);
        }
    }

    private static Transform FindChildByName(Transform parent, string name)
    {
        if (parent.name == name)
            return parent;

        foreach (Transform child in parent)
        {
            var found = FindChildByName(child, name);
            if (found != null)
                return found;
        }

        return null;
    }

    private static string GetElementType(GameObject go)
    {
        if (go.GetComponent<Button>() != null) return "Button";
        if (go.GetComponent<Toggle>() != null) return "Toggle";
        if (go.GetComponent<Slider>() != null) return "Slider";
        if (go.GetComponent<Dropdown>() != null) return "Dropdown";
        if (go.GetComponent<InputField>() != null) return "InputField";
        if (go.GetComponent<Scrollbar>() != null) return "Scrollbar";
        if (go.GetComponent<ScrollRect>() != null) return "ScrollRect";
        if (go.GetComponent<IDragHandler>() != null) return "Draggable";
        if (go.GetComponent<Selectable>() != null) return "Selectable";
        return "Unknown";
    }

    private static string GetLabel(int index)
    {
        if (index < 26) return ((char)('A' + index)).ToString();
        int first = index / 26 - 1;
        int second = index % 26;
        return ((char)('A' + first)).ToString() + ((char)('A' + second)).ToString();
    }

    private class UIElementInfo
    {
        public string Label;
        public string Name;
        public string Type;
        public int SimX;
        public int SimY;
        public int BoundsMinX;
        public int BoundsMinY;
        public int BoundsMaxX;
        public int BoundsMaxY;
        public int SortingOrder;
        public bool Interactable;
        public bool IsDraggable;
    }
}
```

---

## Tool 2: 鼠标点击模拟（UI 空间）- 增强版
<!-- 支持分步 Drag、LongPress，解决不稳定问题 -->

```csharp
// Assets/Scripts/MCP/Tool_MouseUI.cs

using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;
using System.Collections;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_MouseUI
{
    // 保持拖拽状态
    private static GameObject currentDragTarget = null;
    private static PointerEventData currentDragPointerData = null;
    private static bool isDragging = false;

    /// <summary>
    /// 模拟 UI 点击（通过 EventSystem）
    /// Simulate UI click via EventSystem
    /// </summary>
    [McpPluginTool("simulate-click-ui", Title = "Simulate UI click")]
    [Description("Simulate a click on Unity UI elements. Use ui-find-interactive-elements first to get accurate coordinates. " +
                 "模拟点击 Unity UI 元素。先用 ui-find-interactive-elements 获取准确坐标。")]
    public static string SimulateClickUI
    (
        [Description("Screen X coordinate (SimX from ui-find). 屏幕 X 坐标")]
        int x,
        
        [Description("Screen Y coordinate (SimY from ui-find). 屏幕 Y 坐标")]
        int y,
        
        [Description("Mouse button: Left, Right, Middle. 鼠标按钮")]
        string button = "Left"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found. Add EventSystem to scene.";

            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);
            
            // 设置按钮
            pointerData.button = button == "Right" ? PointerEventData.InputButton.Right
                              : button == "Middle" ? PointerEventData.InputButton.Middle
                              : PointerEventData.InputButton.Left;

            var results = new System.Collections.Generic.List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return $"[Success] Click at ({x}, {y}) - No UI element hit (empty space).";

            var target = results[0].gameObject;

            // 检查是否可点击
            if (!ExecuteEvents.CanHandleEvent<IPointerClickHandler>(target))
            {
                // 检查是否是 Button
                var btn = target.GetComponent<Button>();
                if (btn != null && btn.interactable)
                {
                    btn.onClick.Invoke();
                    return $"[Success] Clicked Button '{target.name}' via onClick.Invoke()";
                }
                return $"[Warning] '{target.name}' does not handle click events. Found: {target.name}";
            }

            // 执行点击事件序列
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerDownHandler);
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerUpHandler);
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerClickHandler);

            return $"[Success] Clicked '{target.name}' at ({x}, {y})";
        });
    }

    /// <summary>
    /// 长按 UI 元素
    /// Long press UI element
    /// </summary>
    [McpPluginTool("simulate-long-press-ui", Title = "Simulate UI long press")]
    [Description("Press and hold UI element for specified duration. No click event is fired. " +
                 "长按 UI 元素指定时长，不触发 click 事件。")]
    public static string SimulateLongPressUI
    (
        [Description("Screen X coordinate. 屏幕 X 坐标")]
        int x,
        
        [Description("Screen Y coordinate. 屏幕 Y 坐标")]
        int y,
        
        [Description("Hold duration in seconds. 按住时长（秒）")]
        float duration = 1.0f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found.";

            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);
            pointerData.button = PointerEventData.InputButton.Left;

            var results = new System.Collections.Generic.List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return $"[Success] Long press at ({x}, {y}) - No UI element.";

            var target = results[0].gameObject;

            // PointerDown
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerDownHandler);

            // 等待
            System.Threading.Thread.Sleep((int)(duration * 1000));

            // PointerUp (但不触发 click)
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerUpHandler);

            return $"[Success] Long pressed '{target.name}' for {duration}s at ({x}, {y})";
        });
    }

    /// <summary>
    /// 开始 UI 拖拽（按下并开始拖拽）
    /// Start UI drag - press and begin drag
    /// </summary>
    [McpPluginTool("simulate-drag-ui-start", Title = "Start UI drag")]
    [Description("Begin drag at position. MUST call simulate-drag-ui-end to release. " +
                 "在指定位置开始拖拽。必须调用 simulate-drag-ui-end 来释放。")]
    public static string SimulateDragUIStart
    (
        [Description("Screen X coordinate. 屏幕 X 坐标")]
        int x,
        
        [Description("Screen Y coordinate. 屏幕 Y 坐标")]
        int y
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (isDragging)
                return "[Error] Already dragging. Call simulate-drag-ui-end first.";

            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found.";

            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);
            pointerData.button = PointerEventData.InputButton.Left;

            var results = new System.Collections.Generic.List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return "[Error] No UI element at ({x}, {y}). Cannot start drag on empty space.";

            var target = results[0].gameObject;

            // 检查是否可拖拽
            if (!ExecuteEvents.CanHandleEvent<IDragHandler>(target))
                return $"[Error] '{target.name}' is not draggable. Use ui-find-interactive-elements to find draggable elements.";

            // PointerDown
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerDownHandler);

            // BeginDrag
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.beginDragHandler);

            // 保持状态
            currentDragTarget = target;
            currentDragPointerData = pointerData;
            isDragging = true;

            return $"[Success] Drag started on '{target.name}' at ({x}, {y})";
        });
    }

    /// <summary>
    /// 移动 UI 拖拽位置（拖拽过程中移动）
    /// Move during UI drag
    /// </summary>
    [McpPluginTool("simulate-drag-ui-move", Title = "Move UI drag")]
    [Description("Move drag position. Must have started drag with simulate-drag-ui-start. " +
                 "移动拖拽位置。必须先用 simulate-drag-ui-start 开始拖拽。")]
    public static string SimulateDragUIMove
    (
        [Description("Target screen X. 目标屏幕 X")]
        int x,
        
        [Description("Target screen Y. 目标屏幕 Y")]
        int y
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (!isDragging || currentDragTarget == null)
                return "[Error] Not currently dragging. Start with simulate-drag-ui-start first.";

            currentDragPointerData.position = new Vector2(x, y);
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.dragHandler);

            return $"[Success] Drag moved to ({x}, {y}) on '{currentDragTarget.name}'";
        });
    }

    /// <summary>
    /// 结束 UI 拖拽（释放）
    /// End UI drag - release
    /// </summary>
    [McpPluginTool("simulate-drag-ui-end", Title = "End UI drag")]
    [Description("End drag at position. MUST call after simulate-drag-ui-start. " +
                 "在指定位置结束拖拽。必须在 simulate-drag-ui-start 之后调用。")]
    public static string SimulateDragUIEnd
    (
        [Description("End screen X. 结束屏幕 X")]
        int x,
        
        [Description("End screen Y. 结束屏幕 Y")]
        int y
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (!isDragging || currentDragTarget == null)
                return "[Error] Not currently dragging. Start with simulate-drag-ui-start first.";

            currentDragPointerData.position = new Vector2(x, y);
            
            // Drag (final move)
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.dragHandler);
            
            // EndDrag
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.endDragHandler);
            
            // PointerUp
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.pointerUpHandler);

            var targetName = currentDragTarget.name;
            
            // 清理状态
            currentDragTarget = null;
            currentDragPointerData = null;
            isDragging = false;

            return $"[Success] Drag ended at ({x}, {y}) on '{targetName}'";
        });
    }

    /// <summary>
    /// 一体化 UI 拖拽（从起点拖到终点）
    /// One-shot UI drag from start to end
    /// </summary>
    [McpPluginTool("simulate-drag-ui", Title = "Drag UI element")]
    [Description("Drag UI element from start to end position. For complex drag, use drag-ui-start/move/end separately. " +
                 "从起点拖拽 UI 元素到终点。复杂拖拽请使用 drag-ui-start/move/end 分步操作。")]
    public static string SimulateDragUI
    (
        [Description("Start screen X. 起点屏幕 X")]
        int startX,
        
        [Description("Start screen Y. 起点屏幕 Y")]
        int startY,
        
        [Description("End screen X. 终点屏幕 X")]
        int endX,
        
        [Description("End screen Y. 终点屏幕 Y")]
        int endY,
        
        [Description("Duration in seconds (0 for instant). 持续时间（秒），0 为瞬间")]
        float duration = 0.3f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            // 使用分步 API
            var startResult = SimulateDragUIStart(startX, startY);
            if (startResult.StartsWith("[Error]"))
                return startResult;

            // 如果有持续时间，分步移动
            if (duration > 0)
            {
                int steps = 10;
                float stepTime = duration / steps;
                
                for (int i = 1; i <= steps; i++)
                {
                    float t = i / (float)steps;
                    int currentX = (int)(startX + (endX - startX) * t);
                    int currentY = (int)(startY + (endY - startY) * t);
                    
                    currentDragPointerData.position = new Vector2(currentX, currentY);
                    ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.dragHandler);
                    
                    System.Threading.Thread.Sleep((int)(stepTime * 1000));
                }
            }

            return SimulateDragUIEnd(endX, endY);
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
                return $"[Error] Button '{buttonName}' not found. Use ui-find-interactive-elements to list all buttons.";

            var buttonComponent = button.GetComponent<Button>();
            if (buttonComponent == null)
                return $"[Error] GameObject '{buttonName}' has no Button component.";

            if (!buttonComponent.interactable)
                return $"[Warning] Button '{buttonName}' is not interactable (disabled).";

            buttonComponent.onClick.Invoke();
            return $"[Success] Clicked button: {buttonName}";
        });
    }

    /// <summary>
    /// 悬停在 UI 元素上
    /// Hover over UI element
    /// </summary>
    [McpPluginTool("simulate-hover-ui", Title = "Hover over UI")]
    [Description("Move pointer over UI element without clicking. " +
                 "移动指针悬停在 UI 元素上但不点击。")]
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

            if (results.Count > 0)
            {
                var target = results[0].gameObject;
                ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerEnterHandler);
                return $"[Success] Hovering over: {target.name}";
            }

            // Exit previous hover
            if (eventSystem.currentSelectedGameObject != null)
            {
                ExecuteEvents.Execute(eventSystem.currentSelectedGameObject, pointerData, ExecuteEvents.pointerExitHandler);
            }

            return $"[Success] Hover at ({x}, {y}) - No UI element.";
        });
    }
}
```

---

## Tool 3: 鼠标点击模拟（世界空间）

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
        int y,
        
        [Description("Button: 0=left, 1=right, 2=middle. 按钮")]
        int button = 0
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available. Ensure Input System is enabled.";

            Mouse.current.position.value = new Vector2(x, y);
            
            var buttonControl = button == 0 ? Mouse.current.leftButton 
                          : button == 1 ? Mouse.current.rightButton 
                          : Mouse.current.middleButton;
            
            buttonControl.press();
            buttonControl.release();

            var ray = Camera.main.ScreenPointToRay(new Vector2(x, y));
            var hit = Physics.Raycast(ray, out var hitInfo, 100f);

            if (hit)
                return $"[Success] Clicked at ({x}, {y}). Hit: {hitInfo.collider.gameObject.name}";
            else
                return $"[Success] Clicked at ({x}, {y}). No collider hit (raycast miss).";
        });
    }

    /// <summary>
    /// 长按（用于挖掘等操作）
    /// Long press for mining etc.
    /// </summary>
    [McpPluginTool("simulate-long-press-world", Title = "Long press world")]
    [Description("Press and hold mouse for duration. Use for mining, charging etc. " +
                 "按住鼠标一段时间，用于挖掘、蓄力等操作。")]
    public static string SimulateLongPressWorld
    (
        [Description("Screen X. 屏幕 X")]
        int x,
        
        [Description("Screen Y. 屏幕 Y")]
        int y,
        
        [Description("Duration in seconds. 持续时间（秒）")]
        float duration = 2.0f,
        
        [Description("Button: 0=left, 1=right, 2=middle. 按钮")]
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
            System.Threading.Thread.Sleep((int)(duration * 1000));
            buttonControl.release();

            return $"[Success] Long pressed at ({x}, {y}) for {duration}s";
        });
    }

    /// <summary>
    /// 鼠标按下（不释放）
    /// </summary>
    [McpPluginTool("simulate-mouse-down", Title = "Mouse button down")]
    [Description("Press mouse button without releasing. " +
                 "按下鼠标按钮不释放。")]
    public static string SimulateMouseDown(int x, int y, int button = 0)
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
    /// 鼠标释放
    /// </summary>
    [McpPluginTool("simulate-mouse-up", Title = "Mouse button up")]
    [Description("Release previously pressed mouse button. " +
                 "释放之前按下的鼠标按钮。")]
    public static string SimulateMouseUp(int button = 0)
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
    /// 鼠标移动
    /// </summary>
    [McpPluginTool("simulate-mouse-move", Title = "Move mouse")]
    [Description("Move mouse to screen position without clicking. " +
                 "移动鼠标到屏幕位置但不点击。")]
    public static string SimulateMouseMove(int x, int y)
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
    /// 鼠标滚轮
    /// </summary>
    [McpPluginTool("simulate-mouse-scroll", Title = "Scroll mouse wheel")]
    [Description("Inject scroll wheel input. Positive = scroll up, Negative = scroll down. " +
                 "注入滚轮输入。正数向上滚动，负数向下滚动。")]
    public static string SimulateMouseScroll
    (
        [Description("Scroll amount. Typical value: 120 per notch. 滚动量")]
        float scrollY = 120f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            Mouse.current.scroll.value = new Vector2(0, scrollY);
            return $"[Success] Scroll injected: {scrollY}";
        });
    }

    /// <summary>
    /// 鼠标移动增量（FPS 相机控制）
    /// </summary>
    [McpPluginTool("simulate-mouse-delta", Title = "Inject mouse delta")]
    [Description("Inject mouse movement delta for FPS camera control etc. " +
                 "注入鼠标移动增量，用于 FPS 相机控制等。")]
    public static string SimulateMouseDelta
    (
        [Description("Delta X in pixels. Positive = right. 增量 X")]
        float deltaX = 0,
        
        [Description("Delta Y in pixels. Positive = up. 增量 Y")]
        float deltaY = 0
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            // Input System delta 是累积的
            Mouse.current.delta.value = new Vector2(deltaX, deltaY);
            return $"[Success] Mouse delta injected: ({deltaX}, {deltaY})";
        });
    }

    /// <summary>
    /// 世界空间拖拽
    /// </summary>
    [McpPluginTool("simulate-drag-world", Title = "Drag world")]
    [Description("Drag from start to end position for world-space gameplay (e.g. match3 sliding). " +
                 "从起点拖拽到终点，用于世界空间游戏交互（如三消滑动）。")]
    public static string SimulateDragWorld
    (
        [Description("Start X. 起点 X")]
        int startX,
        
        [Description("Start Y. 起点 Y")]
        int startY,
        
        [Description("End X. 终点 X")]
        int endX,
        
        [Description("End Y. 终点 Y")]
        int endY,
        
        [Description("Duration in seconds. 持续时间（秒）")]
        float duration = 0.3f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            Mouse.current.position.value = new Vector2(startX, startY);
            Mouse.current.leftButton.press();

            if (duration > 0)
            {
                int steps = 10;
                float stepTime = duration / steps;
                
                for (int i = 1; i <= steps; i++)
                {
                    float t = i / (float)steps;
                    int currentX = (int)(startX + (endX - startX) * t);
                    int currentY = (int)(startY + (endY - startY) * t);
                    
                    Mouse.current.position.value = new Vector2(currentX, currentY);
                    System.Threading.Thread.Sleep((int)(stepTime * 1000));
                }
            }

            Mouse.current.position.value = new Vector2(endX, endY);
            Mouse.current.leftButton.release();

            return $"[Success] Dragged from ({startX}, {startY}) to ({endX}, {endY})";
        });
    }
}
```

---

## Tool 4: 键盘输入模拟

```csharp
// Assets/Scripts/MCP/Tool_KeyboardInput.cs

using UnityEngine;
using UnityEngine.InputSystem;
using Io.Kmmurzak.Unity.Mcp.Plugin;
using System.Collections.Generic;

[McpPluginToolType]
public static class Tool_KeyboardInput
{
    // 保持按下的按键
    private static HashSet<Key> heldKeys = new HashSet<Key>();

    /// <summary>
    /// 按下并释放按键
    /// </summary>
    [McpPluginTool("simulate-key-press", Title = "Press key")]
    [Description("Press and release a keyboard key. " +
                 "按下并释放键盘按键。")]
    public static string SimulateKeyPress
    (
        [Description("Key name (W, A, S, D, Space, Enter, Escape, etc). 按键名称")]
        string key,
        
        [Description("Hold duration in seconds (0 for instant tap). 按住时长")]
        float duration = 0f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard device not available.";

            var keyControl = ParseKey(key);
            if (keyControl == Key.None)
                return $"[Error] Key '{key}' not recognized. Valid keys: A-Z, 0-9, Space, Enter, Escape, Tab, ArrowUp/Down/Left/Right, LeftShift, LeftCtrl.";

            Keyboard.current[keyControl].press();
            
            if (duration > 0)
                System.Threading.Thread.Sleep((int)(duration * 1000));
            
            Keyboard.current[keyControl].release();

            return $"[Success] Key '{key}' pressed";
        });
    }

    /// <summary>
    /// 按下按键（保持）
    /// </summary>
    [McpPluginTool("simulate-key-down", Title = "Key down")]
    [Description("Press key without releasing. Call key-up to release. " +
                 "按下按键不释放。调用 key-up 释放。")]
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

            var keyControl = ParseKey(key);
            if (keyControl == Key.None)
                return $"[Error] Key '{key}' not recognized.";

            if (heldKeys.Contains(keyControl))
                return $"[Warning] Key '{key}' is already held.";

            Keyboard.current[keyControl].press();
            heldKeys.Add(keyControl);

            return $"[Success] Key '{key}' pressed (held)";
        });
    }

    /// <summary>
    /// 释放按键
    /// </summary>
    [McpPluginTool("simulate-key-up", Title = "Key up")]
    [Description("Release previously held key. " +
                 "释放之前按住的按键。")]
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

            var keyControl = ParseKey(key);
            if (keyControl == Key.None)
                return $"[Error] Key '{key}' not recognized.";

            if (!heldKeys.Contains(keyControl))
                return $"[Warning] Key '{key}' is not currently held.";

            Keyboard.current[keyControl].release();
            heldKeys.Remove(keyControl);

            return $"[Success] Key '{key}' released";
        });
    }

    /// <summary>
    /// 输入文本
    /// </summary>
    [McpPluginTool("simulate-text-input", Title = "Type text")]
    [Description("Type a string of text character by character. " +
                 "逐字符输入文本。")]
    public static string SimulateTextInput
    (
        [Description("Text to type. 要输入的文本")]
        string text,
        
        [Description("Delay between keys in ms. 每个按键间隔")]
        int delayMs = 50
    )
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard device not available.";

            foreach (char c in text)
            {
                var key = CharToKey(c);
                if (key != Key.None)
                {
                    Keyboard.current[key].press();
                    System.Threading.Thread.Sleep(delayMs);
                    Keyboard.current[key].release();
                    System.Threading.Thread.Sleep(delayMs);
                }
            }

            return $"[Success] Typed: '{text}'";
        });
    }

    private static Key ParseKey(string keyName)
    {
        keyName = keyName.ToLower().Replace(" ", "").Replace("-", "");
        
        // 字母
        if (keyName.Length == 1 && keyName[0] >= 'a' && keyName[0] <= 'z')
            return (Key)Enum.Parse(typeof(Key), keyName.ToUpper());
        
        // 数字
        if (keyName.Length == 1 && keyName[0] >= '0' && keyName[0] <= '9')
            return (Key)Enum.Parse(typeof(Key), "Digit" + keyName);
        
        // 特殊键
        switch (keyName)
        {
            case "space": return Key.Space;
            case "enter": case "return": return Key.Enter;
            case "escape": case "esc": return Key.Escape;
            case "tab": return Key.Tab;
            case "backspace": return Key.Backspace;
            case "delete": return Key.Delete;
            case "leftshift": case "shift": return Key.LeftShift;
            case "rightshift": return Key.RightShift;
            case "leftctrl": case "ctrl": return Key.LeftCtrl;
            case "rightctrl": return Key.RightCtrl;
            case "leftalt": case "alt": return Key.LeftAlt;
            case "rightalt": return Key.RightAlt;
            case "arrowup": case "up": return Key.UpArrow;
            case "arrowdown": case "down": return Key.DownArrow;
            case "arrowleft": case "left": return Key.LeftArrow;
            case "arrowright": case "right": return Key.RightArrow;
            case "home": return Key.Home;
            case "end": return Key.End;
            case "pageup": return Key.PageUp;
            case "pagedown": return Key.PageDown;
            case "insert": return Key.Insert;
            case "f1": return Key.F1;
            case "f2": return Key.F2;
            case "f3": return Key.F3;
            case "f4": return Key.F4;
            case "f5": return Key.F5;
            case "f6": return Key.F6;
            case "f7": return Key.F7;
            case "f8": return Key.F8;
            case "f9": return Key.F9;
            case "f10": return Key.F10;
            case "f11": return Key.F11;
            case "f12": return Key.F12;
            default: return Key.None;
        }
    }

    private static Key CharToKey(char c)
    {
        if (c >= 'a' && c <= 'z')
            return (Key)Enum.Parse(typeof(Key), c.ToString().ToUpper());
        if (c >= 'A' && c <= 'Z')
            return (Key)Enum.Parse(typeof(Key), c.ToString());
        if (c >= '0' && c <= '9')
            return (Key)Enum.Parse(typeof(Key), "Digit" + c);
        if (c == ' ')
            return Key.Space;
        return Key.None;
    }
}
```

---

## Tool 5: 输入录制与回放

```csharp
// Assets/Scripts/MCP/Tool_InputRecording.cs

using UnityEngine;
using UnityEngine.InputSystem;
using System.Collections.Generic;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_InputRecording
{
    private static List<InputEvent> recordedEvents = new List<InputEvent>();
    private static bool isRecording = false;
    private static float recordStartTime = 0f;

    [System.Serializable]
    private struct InputEvent
    {
        public float time;
        public string type;     // "click", "key", "move", "scroll"
        public int x, y;
        public string key;
        public bool isDown;
        public float scrollY;
        public int button;
    }

    [System.Serializable]
    private class RecordingData
    {
        public InputEvent[] events;
        public int totalFrames;
        public float durationSeconds;
    }

    [McpPluginTool("record-start", Title = "Start recording")]
    [Description("Start recording input events. " +
                 "开始录制输入事件。")]
    public static string StartRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            recordedEvents.Clear();
            isRecording = true;
            recordStartTime = Time.time;
            return "[Success] Recording started. Use 'record-stop' to stop.";
        });
    }

    [McpPluginTool("record-stop", Title = "Stop recording")]
    [Description("Stop recording and return event count. " +
                 "停止录制并返回事件数量。")]
    public static string StopRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            isRecording = false;
            float duration = Time.time - recordStartTime;
            return $"[Success] Recording stopped. {recordedEvents.Count} events, {duration:F2}s.";
        });
    }

    [McpPluginTool("record-summary", Title = "Recording summary")]
    [Description("Get summary of recorded events. " +
                 "获取录制事件摘要。")]
    public static string GetRecordSummary()
    {
        return MainThread.Instance.Run(() =>
        {
            if (recordedEvents.Count == 0)
                return "[Info] No events recorded.";

            var sb = new System.Text.StringBuilder();
            sb.AppendLine($"Total events: {recordedEvents.Count}");
            sb.AppendLine($"Duration: {recordedEvents[recordedEvents.Count - 1].time:F2}s");
            
            int clicks = 0, keys = 0, moves = 0, scrolls = 0;
            foreach (var e in recordedEvents)
            {
                if (e.type == "click") clicks++;
                else if (e.type == "key") keys++;
                else if (e.type == "move") moves++;
                else if (e.type == "scroll") scrolls++;
            }
            
            sb.AppendLine($"Clicks: {clicks}, Keys: {keys}, Moves: {moves}, Scrolls: {scrolls}");
            return sb.ToString();
        });
    }

    [McpPluginTool("record-add-click", Title = "Add click event")]
    [Description("Add click event to recording. " +
                 "添加点击事件到录制中。")]
    public static string RecordAddClick(int x, int y, bool isDown = true, int button = 0)
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
                isDown = isDown,
                button = button
            });

            return $"[Success] Added click ({x}, {y}) at {Time.time - recordStartTime:F2}s";
        });
    }

    [McpPluginTool("record-add-key", Title = "Add key event")]
    [Description("Add key event to recording. " +
                 "添加按键事件到录制中。")]
    public static string RecordAddKey(string key, bool isDown = true)
    {
        return MainThread.Instance.Run(() =>
        {
            if (!isRecording)
                return "[Error] Not recording.";

            recordedEvents.Add(new InputEvent
            {
                time = Time.time - recordStartTime,
                type = "key",
                key = key,
                isDown = isDown
            });

            return $"[Success] Added key '{key}' at {Time.time - recordStartTime:F2}s";
        });
    }

    [McpPluginTool("replay-input", Title = "Replay recorded input")]
    [Description("Replay all recorded events. " +
                 "回放所有录制事件。")]
    public static string ReplayInput(float speed = 1f)
    {
        return MainThread.Instance.Run(() =>
        {
            if (recordedEvents.Count == 0)
                return "[Error] No events to replay.";

            if (isRecording)
                return "[Error] Still recording. Stop first.";

            float replayStart = Time.time;
            int count = 0;

            foreach (var e in recordedEvents)
            {
                float targetTime = replayStart + (e.time / speed);
                float wait = targetTime - Time.time;
                
                if (wait > 0)
                    System.Threading.Thread.Sleep((int)(wait * 1000));

                if (e.type == "click")
                {
                    if (Mouse.current != null)
                    {
                        Mouse.current.position.value = new Vector2(e.x, e.y);
                        var btn = e.button == 0 ? Mouse.current.leftButton
                               : e.button == 1 ? Mouse.current.rightButton
                               : Mouse.current.middleButton;
                        if (e.isDown) btn.press(); else btn.release();
                    }
                }
                else if (e.type == "key")
                {
                    if (Keyboard.current != null)
                    {
                        // Key replay logic
                    }
                }
                else if (e.type == "move")
                {
                    if (Mouse.current != null)
                        Mouse.current.position.value = new Vector2(e.x, e.y);
                }

                count++;
            }

            return $"[Success] Replayed {count} events at {speed}x speed.";
        });
    }

    [McpPluginTool("record-export", Title = "Export recording")]
    [Description("Export recording as JSON. " +
                 "导出录制为 JSON。")]
    public static string ExportRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            if (recordedEvents.Count == 0)
                return "[Info] No events to export.";

            var data = new RecordingData
            {
                events = recordedEvents.ToArray(),
                totalFrames = recordedEvents.Count,
                durationSeconds = recordedEvents[recordedEvents.Count - 1].time
            };

            var json = UnityEngine.JsonUtility.ToJson(data, true);
            return $"[Success] Exported {recordedEvents.Count} events.\n{json}";
        });
    }

    [McpPluginTool("record-import", Title = "Import recording")]
    [Description("Import recording from JSON. " +
                 "从 JSON 导入录制。")]
    public static string ImportRecording(string json)
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
                return $"[Error] Import failed: {ex.Message}";
            }
        });
    }

    [McpPluginTool("record-clear", Title = "Clear recording")]
    [Description("Clear all recorded events. " +
                 "清除所有录制事件。")]
    public static string ClearRecording()
    {
        return MainThread.Instance.Run(() =>
        {
            recordedEvents.Clear();
            return "[Success] Recording cleared.";
        });
    }
}
```

---

## 推荐工作流程

### 1. UI 交互（推荐流程）

```
# Step 1: 获取 UI 元素坐标（关键！）
ui-find-interactive-elements format="summary"
# 输出：A: StartButton (SimX=400, SimY=300), B: LevelItem_Draggable (SimX=200, SimY=150)...

# Step 2: 使用返回的坐标操作
simulate-click-ui x=400 y=300
# 或拖拽
simulate-drag-ui startX=200 startY=150 endX=400 endY=150
```

### 2. 复杂拖拽（分步操作）

```
# 使用分步 API 更稳定
simulate-drag-ui-start x=200 y=150
# 可以在这里截图检查状态
screenshot-game-view
simulate-drag-ui-move x=250 y=150
simulate-drag-ui-end x=400 y=150
```

### 3. 录制回放

```
record-start
record-add-click x=100 y=200 isDown=true
record-add-click x=100 y=200 isDown=false
record-add-key key="Space" isDown=true
record-add-key key="Space" isDown=false
record-stop
record-export  # 保存 JSON
replay-input speed=1.0
```

---

## 工具对比表

| 工具 | 用途 | 坐标来源 |
|------|------|----------|
| `ui-find-interactive-elements` | **首选**：获取所有可交互元素坐标 | 自动计算 |
| `ui-find-element-by-name` | 查找特定元素坐标 | 按名称 |
| `simulate-click-ui` | 点击 UI 按钮/开关 | 用 ui-find 的 SimX/SimY |
| `simulate-drag-ui-start/move/end` | **分步拖拽（稳定）** | 用 ui-find 的坐标 |
| `simulate-drag-ui` | 一体化拖拽 | 起点 + 终点 |
| `simulate-long-press-ui` | 长按 UI | 用 ui-find 的坐标 |
| `simulate-click-world` | 点击游戏世界对象 | 估算或 raycast |
| `simulate-drag-world` | 世界空间拖拽（三消） | 起点 + 终点 |
| `simulate-key-press/down/up` | 键盘操作 | 按键名称 |

---

## 文件结构

```
Assets/Scripts/MCP/
├── Tool_UIElementFinder.cs    # UI 元素定位（新增！关键）
├── Tool_MouseUI.cs            # UI 鼠标操作（增强版）
├── Tool_MouseInput.cs         # 世界空间鼠标
├── Tool_KeyboardInput.cs      # 键盘
└── Tool_InputRecording.cs     # 录制回放
```

---

## 常见问题解决

### 问题：Drag 经常报"找不到 draggable"

**原因**：坐标不准确，没有点在可拖拽元素上

**解决**：
```
# 先用 ui-find-interactive-elements 获取准确坐标
ui-find-interactive-elements
# 查找 IsDraggable=true 的元素，使用其 SimX/SimY
simulate-drag-ui-start x=<SimX> y=<SimY>
```

### 问题：拖拽不稳定、时好时坏

**解决**：使用分步 API
```
# 一体化 drag 可能太快
simulate-drag-ui startX=... startY=... endX=... endY=... duration=0.3

# 改用分步 API，可在中间检查状态
simulate-drag-ui-start x=... y=...
wait-until-condition condition="..."  # 等待状态
simulate-drag-ui-move x=... y=...
simulate-drag-ui-end x=... y=...
```

### 问题：找不到某个按钮

**解决**：
```
ui-find-interactive-elements format="summary"
# 检查输出列表，确认按钮名称和坐标

# 或直接按名称查找
ui-find-element-by-name name="SettingsButton"
```