# Unity-MCP Custom Tools: UI Hierarchy Snapshot & Annotated Elements
<!-- Unity-MCP 自定义工具：UI DOM 树快照 + 标注元素坐标 -->

解决截图验证的**视觉盲区**：分辨率敏感、帧时机不确定、大模型视觉判断不可靠。
输出结构化的 UI 节点树，让验证从"看图猜状态"升级为"断言数据"。

**v2.1 新增**：`ui-annotate-elements` — 类似 unity-cli-loop 的 `screenshot --annotate-elements`，一键获取所有可交互元素坐标。

## 安装方式

1. 在项目中创建或确认 `Assets/Scripts/MCP/` 目录
2. 创建 `Tool_UISnapshot.cs` 脚本文件
3. Unity-MCP 会自动识别并注册

---

## Tool 1: UI Annotated Elements（一键获取交互坐标）

**核心工具**：类似 `uloop screenshot --annotate-elements --elements-only`，直接返回所有可交互元素坐标。

```csharp
// Assets/Scripts/MCP/Tool_UISnapshot.cs

using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;
using System.Collections.Generic;
using System.Text;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_UISnapshot
{
    /// <summary>
    /// 一键获取所有可交互 UI 元素的坐标（类似 CLI 的 annotate-elements）
    /// Get all interactive UI element coordinates in one call (like CLI's annotate-elements)
    /// </summary>
    [McpPluginTool("ui-annotate-elements", Title = "Get annotated interactive UI elements")]
    [Description("Returns all clickable/draggable UI elements with SimX/SimY coordinates. " +
                 "Use this BEFORE simulate-click-ui or simulate-drag-ui for accurate coordinates. " +
                 "Output format matches unity-cli-loop's screenshot --annotate-elements. " +
                 "一键返回所有可点击/可拖拽 UI 元素的坐标，输出格式类似 CLI 版本。")]
    public static string UIAnnotateElements
    (
        [Description("Output format: 'summary' (human-readable) or 'json' (machine-readable). 输出格式")]
        string format = "summary"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var eventSystem = EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found. Add EventSystem to scene.";

            var canvases = Object.FindObjectsByType<Canvas>(FindObjectsSortMode.None);
            if (canvases.Length == 0)
                return "[Error] No Canvas found in scene.";

            var elements = new List<AnnotatedElement>();
            
            foreach (var canvas in canvases)
            {
                if (!canvas.gameObject.activeInHierarchy)
                    continue;

                FindInteractiveElements(canvas.transform, elements, canvas);
            }

            if (elements.Count == 0)
                return "[Info] No interactive UI elements found.";

            // 按层级排序（最前面的优先，类似 CLI）
            elements.Sort((a, b) => {
                // 先按 sortingOrder 排序
                if (a.SortingOrder != b.SortingOrder)
                    return b.SortingOrder - a.SortingOrder;
                // 再按 z 位置
                return b.PositionZ.CompareTo(a.PositionZ);
            });

            // 添加标签 (A, B, C...) - 与 CLI 一致
            for (int i = 0; i < elements.Count; i++)
            {
                elements[i].Label = GetLabel(i);
            }

            if (format == "json")
            {
                return BuildJsonOutput(elements);
            }
            else
            {
                return BuildSummaryOutput(elements);
            }
        });
    }

    private static void FindInteractiveElements(Transform parent, List<AnnotatedElement> elements, Canvas rootCanvas)
    {
        foreach (Transform child in parent)
        {
            if (!child.gameObject.activeInHierarchy)
                continue;

            var rectTransform = child.GetComponent<RectTransform>();
            if (rectTransform == null)
                continue;

            // 检查是否是可交互元素
            var selectable = child.GetComponent<Selectable>();
            var dragHandler = child.GetComponent<IDragHandler>();
            var clickHandler = child.GetComponent<IPointerClickHandler>();
            
            bool isInteractive = selectable != null || dragHandler != null || clickHandler != null;

            if (isInteractive)
            {
                var canvas = child.GetComponentInParent<Canvas>();
                var camera = canvas.worldCamera ?? null;
                
                // 计算屏幕中心坐标（与 CLI 的 SimX/SimY 一致）
                Vector2 center = RectTransformUtility.WorldToScreenPoint(camera, rectTransform.position);
                
                // 计算边界
                Vector3[] corners = new Vector3[4];
                rectTransform.GetWorldCorners(corners);
                
                Vector2 min = RectTransformUtility.WorldToScreenPoint(camera, corners[0]);
                Vector2 max = RectTransformUtility.WorldToScreenPoint(camera, corners[2]);

                elements.Add(new AnnotatedElement
                {
                    Label = "", // 后面填充
                    Name = child.name,
                    Type = GetElementType(child.gameObject),
                    SimX = (int)center.x,
                    SimY = (int)center.y,
                    BoundsMinX = (int)min.x,
                    BoundsMinY = (int)min.y,
                    BoundsMaxX = (int)max.x,
                    BoundsMaxY = (int)max.y,
                    SortingOrder = canvas?.sortingOrder ?? 0,
                    PositionZ = rectTransform.position.z,
                    Interactable = selectable?.interactable ?? true,
                    IsDraggable = dragHandler != null,
                    IsClickable = clickHandler != null || selectable != null
                });
            }

            // 递归查找子元素
            FindInteractiveElements(child, elements, rootCanvas);
        }
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

    private static string BuildSummaryOutput(List<AnnotatedElement> elements)
    {
        var sb = new StringBuilder();
        sb.AppendLine($"AnnotatedElements: {elements.Count} elements found");
        sb.AppendLine("(A = frontmost, use SimX/SimY directly with simulate-mouse-ui)");
        sb.AppendLine("");
        
        foreach (var elem in elements)
        {
            sb.AppendLine($"{elem.Label}: {elem.Name}");
            sb.AppendLine($"  Type: {elem.Type}");
            sb.AppendLine($"  SimX: {elem.SimX}, SimY: {elem.SimY}");
            sb.AppendLine($"  Bounds: ({elem.BoundsMinX},{elem.BoundsMinY}) to ({elem.BoundsMaxX},{elem.BoundsMaxY})");
            
            var flags = new List<string>();
            if (elem.IsClickable) flags.Add("clickable");
            if (elem.IsDraggable) flags.Add("draggable");
            if (!elem.Interactable) flags.Add("disabled");
            
            sb.AppendLine($"  Flags: {string.Join(", ", flags)}");
            sb.AppendLine($"  SortingOrder: {elem.SortingOrder}");
            sb.AppendLine("");
        }
        
        return sb.ToString();
    }

    private static string BuildJsonOutput(List<AnnotatedElement> elements)
    {
        // 输出格式与 CLI 的 AnnotatedElements 数组一致
        var sb = new StringBuilder();
        sb.AppendLine("{");
        sb.AppendLine("  \"AnnotatedElements\": [");
        
        for (int i = 0; i < elements.Count; i++)
        {
            var elem = elements[i];
            sb.AppendLine("    {");
            sb.AppendLine($"      \"Label\": \"{elem.Label}\",");
            sb.AppendLine($"      \"Name\": \"{elem.Name}\",");
            sb.AppendLine($"      \"Type\": \"{elem.Type}\",");
            sb.AppendLine($"      \"SimX\": {elem.SimX},");
            sb.AppendLine($"      \"SimY\": {elem.SimY},");
            sb.AppendLine($"      \"BoundsMinX\": {elem.BoundsMinX},");
            sb.AppendLine($"      \"BoundsMinY\": {elem.BoundsMinY},");
            sb.AppendLine($"      \"BoundsMaxX\": {elem.BoundsMaxX},");
            sb.AppendLine($"      \"BoundsMaxY\": {elem.BoundsMaxY},");
            sb.AppendLine($"      \"SortingOrder\": {elem.SortingOrder},");
            sb.AppendLine($"      \"SiblingIndex\": 0,");
            sb.AppendLine($"      \"Interactable\": {elem.Interactable.ToString().ToLower()},");
            sb.AppendLine($"      \"IsDraggable\": {elem.IsDraggable.ToString().ToLower()}");
            sb.Append("    }");
            if (i < elements.Count - 1) sb.AppendLine(",");
            else sb.AppendLine();
        }
        
        sb.AppendLine("  ],");
        sb.AppendLine($"  \"TotalElements\": {elements.Count}");
        sb.AppendLine("}");
        
        return sb.ToString();
    }

    private class AnnotatedElement
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
        public float PositionZ;
        public bool Interactable;
        public bool IsDraggable;
        public bool IsClickable;
    }

    // ========================================
    // Tool 2: UI Hierarchy Snapshot（完整 DOM 树）
    // ========================================

    /// <summary>
    /// 抓取当前 Unity Canvas 的 UI DOM 树结构快照
    /// Capture a structured snapshot of the current Unity Canvas UI hierarchy
    /// </summary>
    [McpPluginTool("ui-hierarchy-snapshot", Title = "Capture UI hierarchy snapshot")]
    [Description("Capture a structured JSON snapshot of all active UI elements in the scene. " +
                 "Returns node names, types, text content, positions, and interaction states. " +
                 "抓取场景中所有激活 UI 元素的结构化 JSON 快照。")]
    public static string UIHierarchySnapshot
    (
        [Description("Include inactive UI objects. 包含未激活的 UI 对象")]
        bool includeInactive = false,

        [Description("Maximum traversal depth (0=unlimited). 最大遍历深度")]
        int maxDepth = 0,

        [Description("Output format: 'json' or 'tree'. 输出格式")]
        string format = "json"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var canvases = Object.FindObjectsByType<Canvas>(FindObjectsSortMode.None);
            if (canvases.Length == 0)
                return "[Warning] No Canvas found in scene.";

            var snapshots = new List<UINodeSnapshot>();

            foreach (var canvas in canvases)
            {
                if (!includeInactive && !canvas.gameObject.activeInHierarchy)
                    continue;

                var rootSnapshot = BuildSnapshot(
                    canvas.gameObject,
                    includeInactive,
                    maxDepth == 0 ? int.MaxValue : maxDepth,
                    0
                );
                snapshots.Add(rootSnapshot);
            }

            if (format == "tree")
            {
                var sb = new StringBuilder();
                foreach (var snapshot in snapshots)
                {
                    RenderTree(snapshot, sb, 0);
                }
                return sb.ToString();
            }

            var json = JsonUtility.ToJson(new { canvases = snapshots }, true);
            return $"[Success] UI snapshot captured. {CountNodes(snapshots)} total nodes.\n\n{json}";
        });
    }

    private static UINodeSnapshot BuildSnapshot(GameObject obj, bool includeInactive, int maxDepth, int currentDepth)
    {
        var snapshot = new UINodeSnapshot();
        snapshot.name = obj.name;
        snapshot.active = obj.activeInHierarchy;
        snapshot.depth = currentDepth;

        var rectTransform = obj.GetComponent<RectTransform>();
        if (rectTransform != null)
        {
            snapshot.position = rectTransform.anchoredPosition.ToString();
            snapshot.size = rectTransform.sizeDelta.ToString();
        }

        // 检查 UI 组件
        var text = obj.GetComponent<Text>();
        if (text != null)
        {
            snapshot.type = "Text";
            snapshot.text = text.text;
        }
        else if (obj.GetComponent<Button>() != null)
        {
            snapshot.type = "Button";
            snapshot.interactable = obj.GetComponent<Button>().interactable;
        }
        else if (obj.GetComponent<Toggle>() != null)
        {
            snapshot.type = "Toggle";
            snapshot.interactable = obj.GetComponent<Toggle>().interactable;
            snapshot.value = obj.GetComponent<Toggle>().isOn;
        }
        else if (obj.GetComponent<Slider>() != null)
        {
            snapshot.type = "Slider";
            var slider = obj.GetComponent<Slider>();
            snapshot.interactable = slider.interactable;
            snapshot.value = slider.value.ToString();
        }
        else if (obj.GetComponent<InputField>() != null)
        {
            snapshot.type = "InputField";
            snapshot.text = obj.GetComponent<InputField>().text;
        }
        else if (obj.GetComponent<Image>() != null)
        {
            snapshot.type = "Image";
        }
        else if (obj.GetComponent<Canvas>() != null)
        {
            snapshot.type = "Canvas";
        }
        else if (rectTransform != null)
        {
            snapshot.type = "RectTransform";
        }

        // 递归子节点
        if (currentDepth < maxDepth)
        {
            snapshot.children = new List<UINodeSnapshot>();
            foreach (Transform child in obj.transform)
            {
                if (!includeInactive && !child.gameObject.activeInHierarchy)
                    continue;
                snapshot.children.Add(BuildSnapshot(child.gameObject, includeInactive, maxDepth, currentDepth + 1));
            }
        }

        return snapshot;
    }

    private static void RenderTree(UINodeSnapshot node, StringBuilder sb, int indent)
    {
        sb.Append(new string(' ', indent * 2));
        sb.AppendLine($"[{node.type}] {node.name} (active:{node.active})");
        
        if (!string.IsNullOrEmpty(node.text))
        {
            sb.Append(new string(' ', indent * 2 + 2));
            sb.AppendLine($"text: \"{node.text}\"");
        }
        
        if (node.children != null)
        {
            foreach (var child in node.children)
            {
                RenderTree(child, sb, indent + 1);
            }
        }
    }

    private static int CountNodes(List<UINodeSnapshot> snapshots)
    {
        int count = 0;
        foreach (var snapshot in snapshots)
        {
            count += CountNodesRecursive(snapshot);
        }
        return count;
    }

    private static int CountNodesRecursive(UINodeSnapshot node)
    {
        int count = 1;
        if (node.children != null)
        {
            foreach (var child in node.children)
            {
                count += CountNodesRecursive(child);
            }
        }
        return count;
    }

    [Serializable]
    private class UINodeSnapshot
    {
        public string name;
        public string type;
        public bool active;
        public int depth;
        public string position;
        public string size;
        public string text;
        public bool interactable;
        public string value;
        public List<UINodeSnapshot> children;
    }

    // ========================================
    // Tool 3: 查找特定 UI 元素
    // ========================================

    /// <summary>
    /// 查找指定名称的 UI 元素并返回坐标
    /// Find UI element by name and return coordinates
    /// </summary>
    [McpPluginTool("ui-element-find", Title = "Find UI element by name")]
    [Description("Find a specific UI element by GameObject name and return SimX/SimY coordinates for simulate-mouse-ui. " +
                 "通过名称查找 UI 元素并返回 SimX/SimY 坐标。")]
    public static string UIElementFind
    (
        [Description("GameObject name to find. 要查找的对象名称")]
        string name
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var go = GameObject.Find(name);
            if (go == null)
            {
                // 递归查找
                var canvases = Object.FindObjectsByType<Canvas>(FindObjectsSortMode.None);
                foreach (var canvas in canvases)
                {
                    go = FindChildRecursive(canvas.transform, name);
                    if (go != null) break;
                }
            }

            if (go == null)
                return $"[Error] UI element '{name}' not found.";

            var rectTransform = go.GetComponent<RectTransform>();
            if (rectTransform == null)
                return $"[Error] '{name}' has no RectTransform.";

            var canvas = go.GetComponentInParent<Canvas>();
            var camera = canvas?.worldCamera ?? null;
            Vector2 center = RectTransformUtility.WorldToScreenPoint(camera, rectTransform.position);

            string type = GetElementType(go);

            return $"[Success] Found '{name}'\n" +
                   $"Type: {type}\n" +
                   $"SimX: {(int)center.x}, SimY: {(int)center.y}\n" +
                   $"Use: simulate-click-ui x={(int)center.x} y={(int)center.y}";
        });
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

    // ========================================
    // Tool 4: 查找屏幕坐标下的 UI 元素
    // ========================================

    /// <summary>
    /// 查找指定屏幕坐标下的 UI 元素
    /// Find UI element at screen position
    /// </summary>
    [McpPluginTool("ui-element-at-position", Title = "Find UI element at position")]
    [Description("Find what UI element is at a given screen position. Returns element name and SimX/SimY. " +
                 "查找指定屏幕坐标下的 UI 元素。")]
    public static string UIElementAtPosition
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
                return "[Error] No EventSystem found.";

            var pointerData = new PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);

            var results = new List<RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return $"[Info] No UI element at ({x}, {y}).";

            var sb = new StringBuilder();
            sb.AppendLine($"Elements at ({x}, {y}):");
            
            for (int i = 0; i < results.Count; i++)
            {
                var result = results[i];
                var elem = result.gameObject;
                string type = GetElementType(elem);
                sb.AppendLine($"{i + 1}. {elem.name} ({type})");
            }

            sb.AppendLine($"\nTopmost: {results[0].gameObject.name}");
            sb.AppendLine($"SimX: {x}, SimY: {y}");

            return sb.ToString();
        });
    }
}
```

---

## 一体化使用流程（与 CLI 版本一致）

### Step 1: 获取所有可交互元素坐标

```
ui-annotate-elements format="summary"
```

输出（与 CLI 格式一致）：
```
AnnotatedElements: 5 elements found
(A = frontmost, use SimX/SimY directly with simulate-mouse-ui)

A: LevelItem_01
  Type: Draggable
  SimX: 200, SimY: 150
  Bounds: (150,100) to (250,200)
  Flags: clickable, draggable
  SortingOrder: 10

B: StartButton
  Type: Button
  SimX: 400, SimY: 300
  Bounds: (350,250) to (450,350)
  Flags: clickable
  SortingOrder: 5
...
```

### Step 2: 直接使用坐标操作

```
# 点击 B 元素
simulate-click-ui x=400 y=300

# 拖拽 A 元素
simulate-drag-ui startX=200 startY=150 endX=400 endY=150
```

---

## 工具对比

| CLI 命令 | MCP 工具 | 功能 |
|----------|---------|------|
| `uloop screenshot --annotate-elements --elements-only` | `ui-annotate-elements` | 获取可交互元素坐标 |
| `uloop screenshot --annotate-elements` | `ui-annotate-elements` + `screenshot-game-view` | 截图 + 坐标 |
| `uloop simulate-mouse-ui --action Click` | `simulate-click-ui` | 点击 UI |
| `uloop simulate-mouse-ui --action Drag` | `simulate-drag-ui` 或分步 API | 拖拽 UI |

---

## 文件结构

```
Assets/Scripts/MCP/
└── Tool_UISnapshot.cs    # 包含所有 UI 相关工具
    ├── ui-annotate-elements      # 一键获取坐标（新增！）
    ├── ui-hierarchy-snapshot     # UI DOM 树快照
    ├── ui-element-find           # 按名称查找
    └── ui-element-at-position    # 按坐标查找
```

---

## 注意事项

1. **坐标系一致** — SimX/SimY 与 CLI 版本一致，直接用于 simulate-mouse-ui
2. **Label 标签** — A=最前，B=次前，与 CLI 一致
3. **SortingOrder** — 用于判断层级顺序
4. **IsDraggable/IsClickable** — 标识元素可执行的操作类型