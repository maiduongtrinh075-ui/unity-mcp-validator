# Unity-MCP Custom Tools: Input Simulation & Recording (v2.2)
<!-- Unity-MCP 自定义工具：输入模拟与录制回放 -->

将此文件中的代码添加到你的 Unity 项目中，即可扩展 Unity-MCP 的输入模拟和录制回放功能。

**v2.2 一体化体验（与 unity-cli-loop 一致）：**
- ✅ **按 Label 操作**：`simulate-click-by-label label="A"` — 自动查找坐标，无需手动输入
- ✅ **按名称操作**：`simulate-click-by-name name="StartButton"` — 直接点击
- ✅ **智能拖拽**：`simulate-drag-by-label startLabel="A" endLabel="B"` — 自动坐标
- ✅ **分步 Drag**：DragStart/DragMove/DragEnd 解决不稳定问题
- ✅ **完善错误处理**：检查元素类型、可拖拽性、EventSystem 状态

## 安装方式

1. 在项目中创建 `Assets/Scripts/MCP/` 目录
2. 创建以下脚本文件
3. Unity-MCP 会自动识别并注册这些 Tool

**注意**：UI 元素定位工具见 [ui-snapshot-tool.md](ui-snapshot-tool.md)，包含：
- `ui-annotate-elements` — 一键获取所有可交互元素坐标（类似 CLI 的 screenshot --annotate-elements）

---

## 一体化工作流程（推荐）

```
# Step 1: 获取所有可交互元素（类似 CLI）
ui-annotate-elements format="summary"

# 输出：
# A: LevelItem_01 (SimX=200, SimY=150) [draggable]
# B: StartButton (SimX=400, SimY=300) [clickable]
# C: SettingsButton (SimX=500, SimY=300) [clickable]

# Step 2: 按 Label 操作（无需手动输入坐标！）
simulate-click-by-label label="B"          # 点击 StartButton
simulate-drag-by-label startLabel="A" endLabel="D"  # 拖拽 LevelItem
```

---

## Tool 1: 智能操作工具（按 Label/名称）

**核心工具**：自动查找坐标，实现真正的一体化体验。

```csharp
// Assets/Scripts/MCP/Tool_SmartUI.cs

using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;
using System.Collections.Generic;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_SmartUI
{
    // 缓存的元素列表（来自 ui-annotate-elements）
    private static List<CachedElement> cachedElements = null;
    private static float cacheTime = 0f;
    private static float cacheExpirySeconds = 5f;

    private class CachedElement
    {
        public string Label;
        public string Name;
        public int SimX;
        public int SimY;
        public bool IsDraggable;
        public bool IsClickable;
        public GameObject ObjectRef;
    }

    /// <summary>
    /// 按 Label 点击 UI 元素（自动查找坐标）
    /// Click UI element by Label (auto-find coordinates)
    /// </summary>
    [McpPluginTool("simulate-click-by-label", Title = "Click UI by Label")]
    [Description("Click UI element by its Label (A, B, C...). " +
                 "Use ui-annotate-elements first to get Labels, then click without manual coordinates. " +
                 "按 Label (A/B/C...) 点击 UI 元素，无需手动输入坐标。")]
    public static string ClickByLabel
    (
        [Description("Label from ui-annotate-elements (A, B, C...). 来自 ui-annotate-elements 的标签")]
        string label,
        
        [Description("Refresh element cache if stale. 刷新元素缓存")]
        bool refreshCache = false
    )
    {
        return MainThread.Instance.Run(() =>
        {
            // 获取或刷新缓存
            var elements = GetCachedElements(refreshCache);
            if (elements == null || elements.Count == 0)
                return "[Error] No cached elements. Run 'ui-annotate-elements' first.";

            // 查找 Label
            var elem = elements.Find(e => e.Label == label.ToUpper());
            if (elem == null)
                return $"[Error] Label '{label}' not found. Available: {string.Join(", ", elements.ConvertAll(e => e.Label))}";

            if (!elem.IsClickable)
                return $"[Warning] '{elem.Name}' (Label={label}) is not clickable. Try simulate-drag-by-label for draggable elements.";

            // 执行点击
            return ExecuteClick(elem.ObjectRef, elem.SimX, elem.SimY);
        });
    }

    /// <summary>
    /// 按 Label 拖拽 UI 元素（自动查找坐标）
    /// Drag UI element by Labels (auto-find coordinates)
    /// </summary>
    [McpPluginTool("simulate-drag-by-label", Title = "Drag UI by Label")]
    [Description("Drag UI element from startLabel to endLabel. " +
                 "Auto-finds coordinates from cached ui-annotate-elements results. " +
                 "按 Label 拖拽 UI 元素，自动查找坐标。")]
    public static string DragByLabel
    (
        [Description("Start element Label. 起始元素 Label")]
        string startLabel,
        
        [Description("End element Label (or use custom coordinates). 结束元素 Label")]
        string endLabel = null,
        
        [Description("End X (overrides endLabel). 结束 X（覆盖 endLabel）")]
        int endX = -1,
        
        [Description("End Y (overrides endLabel). 结束 Y（覆盖 endLabel）")]
        int endY = -1,
        
        [Description("Duration in seconds. 持续时间")]
        float duration = 0.3f,
        
        [Description("Refresh cache. 刷新缓存")]
        bool refreshCache = false
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var elements = GetCachedElements(refreshCache);
            if (elements == null || elements.Count == 0)
                return "[Error] No cached elements. Run 'ui-annotate-elements' first.";

            var startElem = elements.Find(e => e.Label == startLabel.ToUpper());
            if (startElem == null)
                return $"[Error] Start Label '{startLabel}' not found.";

            if (!startElem.IsDraggable)
                return $"[Error] '{startElem.Name}' is not draggable. Use ui-annotate-elements to find draggable elements.";

            int startX = startElem.SimX;
            int startY = startElem.SimY;
            int targetEndX = endX;
            int targetEndY = endY;

            // 如果指定了 endLabel，使用其坐标
            if (!string.IsNullOrEmpty(endLabel))
            {
                var endElem = elements.Find(e => e.Label == endLabel.ToUpper());
                if (endElem == null)
                    return $"[Error] End Label '{endLabel}' not found.";
                targetEndX = endElem.SimX;
                targetEndY = endElem.SimY;
            }

            // 如果没有有效的终点坐标，报错
            if (targetEndX < 0 || targetEndY < 0)
                return "[Error] Must specify either endLabel or endX/endY coordinates.";

            // 执行拖拽
            return ExecuteDrag(startElem.ObjectRef, startX, startY, targetEndX, targetEndY, duration);
        });
    }

    /// <summary>
    /// 按 GameObject 名称点击 UI 元素
    /// Click UI element by GameObject name
    /// </summary>
    [McpPluginTool("simulate-click-by-name", Title = "Click UI by name")]
    [Description("Click UI element by its GameObject name. " +
                 "Finds element and clicks it directly. " +
                 "按 GameObject 名称点击 UI 元素。")]
    public static string ClickByName
    (
        [Description("GameObject name. GameObject 名称")]
        string name
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var go = FindUIElementByName(name);
            if (go == null)
                return $"[Error] UI element '{name}' not found. Run 'ui-annotate-elements' to list all elements.";

            var canvas = go.GetComponentInParent<Canvas>();
            var camera = canvas?.worldCamera ?? null;
            var rectTransform = go.GetComponent<RectTransform>();
            
            if (rectTransform == null)
                return $"[Error] '{name}' has no RectTransform.";

            Vector2 center = RectTransformUtility.WorldToScreenPoint(camera, rectTransform.position);

            return ExecuteClick(go, (int)center.x, (int)center.y);
        });
    }

    /// <summary>
    /// 按 GameObject 名称拖拽 UI 元素
    /// Drag UI element by GameObject names
    /// </summary>
    [McpPluginTool("simulate-drag-by-name", Title = "Drag UI by name")]
    [Description("Drag UI element from startName to endName. " +
                 "按名称拖拽 UI 元素。")]
    public static string DragByName
    (
        [Description("Start element name. 起始元素名称")]
        string startName,
        
        [Description("End element name. 结束元素名称")]
        string endName = null,
        
        [Description("End X override. 结束 X")]
        int endX = -1,
        
        [Description("End Y override. 结束 Y")]
        int endY = -1,
        
        [Description("Duration. 持续时间")]
        float duration = 0.3f
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var startGo = FindUIElementByName(startName);
            if (startGo == null)
                return $"[Error] Start element '{startName}' not found.";

            if (!ExecuteEvents.CanHandleEvent<IDragHandler>(startGo))
                return $"[Error] '{startName}' is not draggable.";

            var startRect = startGo.GetComponent<RectTransform>();
            var startCanvas = startGo.GetComponentInParent<Canvas>();
            Vector2 startCenter = RectTransformUtility.WorldToScreenPoint(startCanvas?.worldCamera, startRect.position);

            int targetEndX = endX;
            int targetEndY = endY;

            if (!string.IsNullOrEmpty(endName))
            {
                var endGo = FindUIElementByName(endName);
                if (endGo == null)
                    return $"[Error] End element '{endName}' not found.";
                
                var endRect = endGo.GetComponent<RectTransform>();
                var endCanvas = endGo.GetComponentInParent<Canvas>();
                Vector2 endCenter = RectTransformUtility.WorldToScreenPoint(endCanvas?.worldCamera, endRect.position);
                
                targetEndX = (int)endCenter.x;
                targetEndY = (int)endCenter.y;
            }

            if (targetEndX < 0 || targetEndY < 0)
                return "[Error] Must specify endName or endX/endY.";

            return ExecuteDrag(startGo, (int)startCenter.x, (int)startCenter.y, targetEndX, targetEndY, duration);
        });
    }

    // ========================================
    // 辅助方法
    // ========================================

    private static List<CachedElement> GetCachedElements(bool refresh)
    {
        // 如果缓存过期或需要刷新，重新获取
        if (refresh || cachedElements == null || (Time.time - cacheTime) > cacheExpirySeconds)
        {
            RefreshCache();
        }
        return cachedElements;
    }

    private static void RefreshCache()
    {
        cachedElements = new List<CachedElement>();
        cacheTime = Time.time;

        var canvases = Object.FindObjectsByType<Canvas>(FindObjectsSortMode.None);
        
        foreach (var canvas in canvases)
        {
            if (!canvas.gameObject.activeInHierarchy)
                continue;

            FindInteractiveElementsForCache(canvas.transform, cachedElements, canvas);
        }

        // 排序
        cachedElements.Sort((a, b) => {
            var canvasA = a.ObjectRef.GetComponentInParent<Canvas>();
            var canvasB = b.ObjectRef.GetComponentInParent<Canvas>();
            return (canvasB?.sortingOrder ?? 0) - (canvasA?.sortingOrder ?? 0);
        });

        // 添加 Label
        for (int i = 0; i < cachedElements.Count; i++)
        {
            cachedElements[i].Label = GetLabel(i);
        }
    }

    private static void FindInteractiveElementsForCache(Transform parent, List<CachedElement> elements, Canvas rootCanvas)
    {
        foreach (Transform child in parent)
        {
            if (!child.gameObject.activeInHierarchy)
                continue;

            var rectTransform = child.GetComponent<RectTransform>();
            if (rectTransform == null)
                continue;

            var selectable = child.GetComponent<Selectable>();
            var dragHandler = child.GetComponent<IDragHandler>();
            var clickHandler = child.GetComponent<IPointerClickHandler>();
            
            bool isInteractive = selectable != null || dragHandler != null || clickHandler != null;

            if (isInteractive)
            {
                var canvas = child.GetComponentInParent<Canvas>();
                var camera = canvas?.worldCamera ?? null;
                Vector2 center = RectTransformUtility.WorldToScreenPoint(camera, rectTransform.position);

                elements.Add(new CachedElement
                {
                    Label = "",
                    Name = child.name,
                    SimX = (int)center.x,
                    SimY = (int)center.y,
                    IsDraggable = dragHandler != null,
                    IsClickable = clickHandler != null || selectable != null,
                    ObjectRef = child.gameObject
                });
            }

            FindInteractiveElementsForCache(child, elements, rootCanvas);
        }
    }

    private static GameObject FindUIElementByName(string name)
    {
        var go = GameObject.Find(name);
        if (go != null)
            return go;

        // 递归查找
        var canvases = Object.FindObjectsByType<Canvas>(FindObjectsSortMode.None);
        foreach (var canvas in canvases)
        {
            go = FindChildRecursive(canvas.transform, name);
            if (go != null)
                return go;
        }
        return null;
    }

    private static GameObject FindChildRecursive(Transform parent, string name)
    {
        if (parent.name == name)
            return parent.gameObject;
        foreach (Transform child in parent)
        {
            var found = FindChildRecursive(child, name);
            if (found != null)
                return found;
        }
        return null;
    }

    private static string GetLabel(int index)
    {
        if (index < 26) return ((char)('A' + index)).ToString();
        int first = index / 26 - 1;
        int second = index % 26;
        return ((char)('A' + first)).ToString() + ((char)('A' + second)).ToString();
    }

    private static string ExecuteClick(GameObject target, int x, int y)
    {
        var eventSystem = EventSystem.current;
        if (eventSystem == null)
            return "[Error] No EventSystem found.";

        var pointerData = new PointerEventData(eventSystem);
        pointerData.position = new Vector2(x, y);
        pointerData.button = PointerEventData.InputButton.Left;

        // 尝试 Button.onClick
        var btn = target.GetComponent<Button>();
        if (btn != null && btn.interactable)
        {
            btn.onClick.Invoke();
            return $"[Success] Clicked '{target.name}' (via Button.onClick)";
        }

        // 尝试 ExecuteEvents
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerDownHandler);
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerUpHandler);
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerClickHandler);

        return $"[Success] Clicked '{target.name}' at ({x}, {y})";
    }

    private static string ExecuteDrag(GameObject target, int startX, int startY, int endX, int endY, float duration)
    {
        var eventSystem = EventSystem.current;
        if (eventSystem == null)
            return "[Error] No EventSystem found.";

        var pointerData = new PointerEventData(eventSystem);
        pointerData.position = new Vector2(startX, startY);
        pointerData.button = PointerEventData.InputButton.Left;

        // BeginDrag
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerDownHandler);
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.beginDragHandler);

        // 中间移动（如果有 duration）
        if (duration > 0)
        {
            int steps = 10;
            float stepTime = duration / steps;
            
            for (int i = 1; i <= steps; i++)
            {
                float t = i / (float)steps;
                pointerData.position = new Vector2(
                    startX + (endX - startX) * t,
                    startY + (endY - startY) * t
                );
                ExecuteEvents.Execute(target, pointerData, ExecuteEvents.dragHandler);
                System.Threading.Thread.Sleep((int)(stepTime * 1000));
            }
        }

        // EndDrag
        pointerData.position = new Vector2(endX, endY);
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.dragHandler);
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.endDragHandler);
        ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerUpHandler);

        return $"[Success] Dragged '{target.name}' from ({startX},{startY}) to ({endX},{endY})";
    }
}
```

---

## Tool 2: 基础鼠标操作（UI 空间）

保留原有的手动坐标版本，用于精细控制。

```csharp
// Assets/Scripts/MCP/Tool_MouseUI.cs

using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;
using System.Collections.Generic;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_MouseUI
{
    private static GameObject currentDragTarget = null;
    private static PointerEventData currentDragPointerData = null;
    private static bool isDragging = false;

    /// <summary>
    /// 模拟 UI 点击（手动坐标）
    /// Simulate UI click (manual coordinates)
    /// </summary>
    [McpPluginTool("simulate-click-ui", Title = "Click UI at coordinates")]
    [Description("Click UI element at specified screen coordinates. " +
                 "Use ui-annotate-elements to get SimX/SimY first. " +
                 "点击指定屏幕坐标的 UI 元素。")]
    public static string SimulateClickUI(int x, int y, string button = "Left")
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found.";

            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);
            pointerData.button = button == "Right" ? PointerEventData.InputButton.Right
                              : button == "Middle" ? PointerEventData.InputButton.Middle
                              : PointerEventData.InputButton.Left;

            var results = new List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return $"[Success] Click at ({x}, {y}) - No UI element hit.";

            var target = results[0].gameObject;
            var btn = target.GetComponent<Button>();
            if (btn != null && btn.interactable)
            {
                btn.onClick.Invoke();
                return $"[Success] Clicked Button '{target.name}'";
            }

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
    [McpPluginTool("simulate-long-press-ui", Title = "Long press UI")]
    [Description("Press and hold UI element for specified duration. " +
                 "长按 UI 元素指定时长。")]
    public static string SimulateLongPressUI(int x, int y, float duration = 1.0f)
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found.";

            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);

            var results = new List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return $"[Success] Long press at ({x}, {y}) - No UI element.";

            var target = results[0].gameObject;
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerDownHandler);
            System.Threading.Thread.Sleep((int)(duration * 1000));
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerUpHandler);

            return $"[Success] Long pressed '{target.name}' for {duration}s";
        });
    }

    /// <summary>
    /// 分步拖拽：开始
    /// Split drag: Start
    /// </summary>
    [McpPluginTool("simulate-drag-ui-start", Title = "Start UI drag")]
    [Description("Begin drag at position. MUST call drag-ui-end to release. " +
                 "开始拖拽，必须调用 drag-ui-end 释放。")]
    public static string SimulateDragUIStart(int x, int y)
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

            var results = new List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return $"[Error] No UI element at ({x}, {y}).";

            var target = results[0].gameObject;
            if (!ExecuteEvents.CanHandleEvent<IDragHandler>(target))
                return $"[Error] '{target.name}' is not draggable.";

            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.pointerDownHandler);
            ExecuteEvents.Execute(target, pointerData, ExecuteEvents.beginDragHandler);

            currentDragTarget = target;
            currentDragPointerData = pointerData;
            isDragging = true;

            return $"[Success] Drag started on '{target.name}'";
        });
    }

    /// <summary>
    /// 分步拖拽：移动
    /// Split drag: Move
    /// </summary>
    [McpPluginTool("simulate-drag-ui-move", Title = "Move UI drag")]
    [Description("Move drag position. Must call after drag-ui-start. " +
                 "移动拖拽位置。")]
    public static string SimulateDragUIMove(int x, int y)
    {
        return MainThread.Instance.Run(() =>
        {
            if (!isDragging || currentDragTarget == null)
                return "[Error] Not dragging. Call simulate-drag-ui-start first.";

            currentDragPointerData.position = new Vector2(x, y);
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.dragHandler);

            return $"[Success] Drag moved to ({x}, {y})";
        });
    }

    /// <summary>
    /// 分步拖拽：结束
    /// Split drag: End
    /// </summary>
    [McpPluginTool("simulate-drag-ui-end", Title = "End UI drag")]
    [Description("End drag and release. Must call after drag-ui-start. " +
                 "结束拖拽并释放。")]
    public static string SimulateDragUIEnd(int x, int y)
    {
        return MainThread.Instance.Run(() =>
        {
            if (!isDragging || currentDragTarget == null)
                return "[Error] Not dragging. Call simulate-drag-ui-start first.";

            currentDragPointerData.position = new Vector2(x, y);
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.dragHandler);
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.endDragHandler);
            ExecuteEvents.Execute(currentDragTarget, currentDragPointerData, ExecuteEvents.pointerUpHandler);

            var name = currentDragTarget.name;
            currentDragTarget = null;
            currentDragPointerData = null;
            isDragging = false;

            return $"[Success] Drag ended on '{name}' at ({x}, {y})";
        });
    }

    /// <summary>
    /// 一体化拖拽
    /// One-shot drag
    /// </summary>
    [McpPluginTool("simulate-drag-ui", Title = "Drag UI")]
    [Description("Drag from start to end in one call. " +
                 "一次性拖拽操作。")]
    public static string SimulateDragUI(int startX, int startY, int endX, int endY, float duration = 0.3f)
    {
        return MainThread.Instance.Run(() =>
        {
            var startResult = SimulateDragUIStart(startX, startY);
            if (startResult.StartsWith("[Error]"))
                return startResult;

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
}
```

---

## Tool 3: 世界空间鼠标操作

```csharp
// Assets/Scripts/MCP/Tool_MouseInput.cs

using UnityEngine;
using UnityEngine.InputSystem;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_MouseInput
{
    [McpPluginTool("simulate-click-world", Title = "Click world")]
    [Description("Click at screen coordinates for world-space gameplay. " +
                 "点击世界空间对象。")]
    public static string SimulateClickWorld(int x, int y, int button = 0)
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";

            Mouse.current.position.value = new Vector2(x, y);
            var btnControl = button == 0 ? Mouse.current.leftButton
                          : button == 1 ? Mouse.current.rightButton
                          : Mouse.current.middleButton;
            btnControl.press();
            btnControl.release();

            var ray = Camera.main.ScreenPointToRay(new Vector2(x, y));
            if (Physics.Raycast(ray, out var hit, 100f))
                return $"[Success] Clicked at ({x}, {y}). Hit: {hit.collider.gameObject.name}";
            return $"[Success] Clicked at ({x}, {y}). No hit.";
        });
    }

    [McpPluginTool("simulate-drag-world", Title = "Drag world")]
    [Description("Drag from start to end for world-space (match3, etc). " +
                 "世界空间拖拽。")]
    public static string SimulateDragWorld(int startX, int startY, int endX, int endY, float duration = 0.3f)
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
                for (int i = 1; i <= steps; i++)
                {
                    float t = i / (float)steps;
                    Mouse.current.position.value = new Vector2(
                        startX + (endX - startX) * t,
                        startY + (endY - startY) * t
                    );
                    System.Threading.Thread.Sleep((int)(duration * 1000 / steps));
                }
            }

            Mouse.current.position.value = new Vector2(endX, endY);
            Mouse.current.leftButton.release();

            return $"[Success] Dragged from ({startX},{startY}) to ({endX},{endY})";
        });
    }

    [McpPluginTool("simulate-mouse-scroll", Title = "Scroll")]
    [Description("Inject scroll wheel input. 注入滚轮输入。")]
    public static string SimulateMouseScroll(float scrollY = 120f)
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";
            Mouse.current.scroll.value = new Vector2(0, scrollY);
            return $"[Success] Scroll: {scrollY}";
        });
    }

    [McpPluginTool("simulate-mouse-delta", Title = "Mouse delta")]
    [Description("Inject mouse movement delta (FPS camera). " +
                 "注入鼠标增量。")]
    public static string SimulateMouseDelta(float deltaX = 0, float deltaY = 0)
    {
        return MainThread.Instance.Run(() =>
        {
            if (Mouse.current == null)
                return "[Error] Mouse device not available.";
            Mouse.current.delta.value = new Vector2(deltaX, deltaY);
            return $"[Success] Delta: ({deltaX}, {deltaY})";
        });
    }
}
```

---

## Tool 4: 键盘操作

```csharp
// Assets/Scripts/MCP/Tool_KeyboardInput.cs

using UnityEngine;
using UnityEngine.InputSystem;
using System.Collections.Generic;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_KeyboardInput
{
    private static HashSet<Key> heldKeys = new HashSet<Key>();

    [McpPluginTool("simulate-key-press", Title = "Press key")]
    [Description("Press and release a key. 按下并释放按键。")]
    public static string SimulateKeyPress(string key, float duration = 0f)
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard not available.";

            var keyEnum = ParseKey(key);
            if (keyEnum == Key.None)
                return $"[Error] Key '{key}' not recognized.";

            Keyboard.current[keyEnum].press();
            if (duration > 0) System.Threading.Thread.Sleep((int)(duration * 1000));
            Keyboard.current[keyEnum].release();

            return $"[Success] Key '{key}' pressed";
        });
    }

    [McpPluginTool("simulate-key-down", Title = "Key down")]
    [Description("Press key and hold. 按下按键并保持。")]
    public static string SimulateKeyDown(string key)
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard not available.";

            var keyEnum = ParseKey(key);
            if (keyEnum == Key.None)
                return $"[Error] Key '{key}' not recognized.";

            if (heldKeys.Contains(keyEnum))
                return $"[Warning] Key '{key}' already held.";

            Keyboard.current[keyEnum].press();
            heldKeys.Add(keyEnum);
            return $"[Success] Key '{key}' held";
        });
    }

    [McpPluginTool("simulate-key-up", Title = "Key up")]
    [Description("Release held key. 释放按键。")]
    public static string SimulateKeyUp(string key)
    {
        return MainThread.Instance.Run(() =>
        {
            if (Keyboard.current == null)
                return "[Error] Keyboard not available.";

            var keyEnum = ParseKey(key);
            if (keyEnum == Key.None)
                return $"[Error] Key '{key}' not recognized.";

            if (!heldKeys.Contains(keyEnum))
                return $"[Warning] Key '{key}' not held.";

            Keyboard.current[keyEnum].release();
            heldKeys.Remove(keyEnum);
            return $"[Success] Key '{key}' released";
        });
    }

    private static Key ParseKey(string name)
    {
        name = name.ToLower().Replace(" ", "").Replace("-", "");
        if (name.Length == 1 && name[0] >= 'a' && name[0] <= 'z')
            return (Key)Enum.Parse(typeof(Key), name.ToUpper());
        if (name.Length == 1 && name[0] >= '0' && name[0] <= '9')
            return (Key)Enum.Parse(typeof(Key), "Digit" + name);
        switch (name) {
            case "space": return Key.Space;
            case "enter": case "return": return Key.Enter;
            case "escape": case "esc": return Key.Escape;
            case "tab": return Key.Tab;
            case "backspace": return Key.Backspace;
            case "delete": return Key.Delete;
            case "leftshift": case "shift": return Key.LeftShift;
            case "leftctrl": case "ctrl": return Key.LeftCtrl;
            case "arrowup": case "up": return Key.UpArrow;
            case "arrowdown": case "down": return Key.DownArrow;
            case "arrowleft": case "left": return Key.LeftArrow;
            case "arrowright": case "right": return Key.RightArrow;
            default: return Key.None;
        }
    }
}
```

---

## 使用对比

| 场景 | CLI 命令 | MCP 工具 |
|------|---------|---------|
| 获取元素坐标 | `screenshot --annotate-elements --elements-only` | `ui-annotate-elements` |
| 按 Label 点击 | ❌ 需手动输入坐标 | ✅ `simulate-click-by-label label="B"` |
| 按 Label 拖拽 | ❌ 需手动输入坐标 | ✅ `simulate-drag-by-label startLabel="A" endLabel="D"` |
| 按名称点击 | ❌ | ✅ `simulate-click-by-name name="StartButton"` |
| 手动坐标点击 | `simulate-mouse-ui --action Click --x 400 --y 300` | `simulate-click-ui x=400 y=300` |
| 分步拖拽 | `simulate-mouse-ui --action DragStart/DragMove/DragEnd` | `simulate-drag-ui-start/move/end` |

---

## 文件结构

```
Assets/Scripts/MCP/
├── Tool_SmartUI.cs          # 智能操作（按 Label/名称）★ 推荐
├── Tool_MouseUI.cs          # 基础 UI 鼠标操作（手动坐标）
├── Tool_MouseInput.cs       # 世界空间鼠标操作
├── Tool_KeyboardInput.cs    # 键盘操作
└── Tool_UISnapshot.cs       # UI 元素定位（见 ui-snapshot-tool.md）
```

---

## 推荐 workflow

```
# 一体化流程（最简单）
ui-annotate-elements → 查看列表
simulate-click-by-label label="B" → 点击
simulate-drag-by-label startLabel="A" endLabel="D" → 拖拽

# 或按名称直接操作（无需先获取列表）
simulate-click-by-name name="StartButton"
simulate-drag-by-name startName="LevelItem" endName="TargetSlot"
```