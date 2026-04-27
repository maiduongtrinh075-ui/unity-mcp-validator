# Unity-MCP Custom Tools: UI Hierarchy Snapshot
<!-- Unity-MCP 自定义工具：UI DOM 树快照 -->

解决截图验证的**视觉盲区**：分辨率敏感、帧时机不确定、大模型视觉判断不可靠。
输出结构化的 UI 节点树，让验证从"看图猜状态"升级为"断言数据"。

## 安装方式

1. 在项目中创建或确认 `Assets/Scripts/MCP/` 目录
2. 创建 `Tool_UISnapshot.cs` 脚本文件
3. Unity-MCP 会自动识别并注册

---

## Tool: UI Hierarchy Snapshot

```csharp
// Assets/Scripts/MCP/Tool_UISnapshot.cs

using UnityEngine;
using System.Collections.Generic;
using System.Text;
using Io.Kmmurzak.Unity.Mcp.Plugin;

[McpPluginToolType]
public static class Tool_UISnapshot
{
    /// <summary>
    /// 抓取当前 Unity Canvas 的 UI DOM 树结构快照
    /// Capture a structured snapshot of the current Unity Canvas UI hierarchy
    /// </summary>
    [McpPluginTool("ui-hierarchy-snapshot", Title = "Capture UI hierarchy snapshot")]
    [Description("Capture a structured JSON snapshot of all active UI elements in the scene. " +
                 "Returns node names, types, text content, positions, and interaction states. " +
                 "抓取场景中所有激活 UI 元素的结构化 JSON 快照。返回节点名称、类型、文本、坐标和交互状态。")]
    public static string UIHierarchySnapshot
    (
        [Description("Include inactive UI objects. 包含未激活的 UI 对象")]
        bool includeInactive = false,

        [Description("Maximum traversal depth (0=unlimited). 最大遍历深度（0=无限制）")]
        int maxDepth = 0,

        [Description("Output format: 'json' or 'tree'. 输出格式：'json' 或 'tree'")]
        string format = "json"
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var canvases = GameObject.FindObjectsOfType<Canvas>(includeInactive);
            if (canvases.Length == 0)
                return "[Warning] No Canvas found in scene.";

            var snapshots = new List<UINodeSnapshot>();

            foreach (var canvas in canvases)
            {
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

            // JSON 输出
            var json = JsonUtility.ToJson(new { canvases = snapshots }, true);
            return $"[Success] UI snapshot captured. {snapshots.Count} canvas(es), " +
                   $"{CountNodes(snapshots)} total nodes.\n\n{json}";
        });
    }

    /// <summary>
    /// 在指定坐标查找 UI 元素
    /// Find UI element at specified screen position
    /// </summary>
    [McpPluginTool("ui-element-at-position", Title = "Find UI element at screen position")]
    [Description("Find the topmost UI element at the given screen coordinates. " +
                 "Returns element name, type, and interaction state. " +
                 "查找指定屏幕坐标处最顶层的 UI 元素。返回元素名称、类型和交互状态。")]
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
            var eventSystem = UnityEngine.EventSystems.EventSystem.current;
            if (eventSystem == null)
                return "[Error] No EventSystem found. Add EventSystem to scene.";

            var pointerData = new UnityEngine.EventSystems.PointerEventData(eventSystem);
            pointerData.position = new Vector2(x, y);

            var results = new List<UnityEngine.EventSystems.RaycastResult>();
            eventSystem.RaycastAll(pointerData, results);

            if (results.Count == 0)
                return $"[Info] No UI element at ({x}, {y}).";

            var sb = new StringBuilder();
            sb.AppendLine($"[Success] {results.Count} UI element(s) at ({x}, {y}):");

            for (int i = 0; i < results.Count; i++)
            {
                var go = results[i].gameObject;
                var indent = new string(' ', i * 2);
                sb.AppendLine($"{indent}{i + 1}. {GetNodeDescription(go)}");
            }

            return sb.ToString();
        });
    }

    /// <summary>
    /// 查找指定名称的 UI 元素并返回其详细状态
    /// Find a UI element by name and return its detailed state
    /// </summary>
    [McpPluginTool("ui-element-find", Title = "Find UI element by name")]
    [Description("Find a UI element by GameObject name and return its detailed state. " +
                 "通过 GameObject 名称查找 UI 元素并返回其详细状态。")]
    public static string UIFindElement
    (
        [Description("UI element GameObject name. UI 元素 GameObject 名称")]
        string name,

        [Description("Include inactive objects. 包含未激活的对象")]
        bool includeInactive = false
    )
    {
        return MainThread.Instance.Run(() =>
        {
            var go = GameObject.Find(name);
            if (go == null && includeInactive)
            {
                // 尝试在根对象中查找（包括未激活的）
                var allTransforms = Resources.FindObjectsOfTypeAll<Transform>();
                foreach (var t in allTransforms)
                {
                    if (t.name == name && t.GetComponent<RectTransform>() != null)
                    {
                        go = t.gameObject;
                        break;
                    }
                }
            }

            if (go == null)
                return $"[Error] UI element '{name}' not found.";

            var rectTransform = go.GetComponent<RectTransform>();
            if (rectTransform == null)
                return $"[Error] '{name}' is not a UI element (no RectTransform).";

            var details = new StringBuilder();
            details.AppendLine($"[Success] UI element found: {name}");
            details.AppendLine($"  Active: {go.activeInHierarchy}");
            details.AppendLine($"  Layer: {LayerMask.LayerToName(go.layer)}");

            // RectTransform 信息
            details.AppendLine($"  Position: {rectTransform.position}");
            details.AppendLine($"  AnchoredPosition: {rectTransform.anchoredPosition}");
            details.AppendLine($"  SizeDelta: {rectTransform.sizeDelta}");
            details.AppendLine($"  Rect: {rectTransform.rect}");

            // 常见 UI 组件状态
            var selectable = go.GetComponent<UnityEngine.UI.Selectable>();
            if (selectable != null)
                details.AppendLine($"  Interactable: {selectable.interactable}");

            var button = go.GetComponent<UnityEngine.UI.Button>();
            if (button != null)
            {
                details.AppendLine($"  Type: Button");
                details.AppendLine($"  Interactable: {button.interactable}");
                details.AppendLine($"  onClick listeners: {button.onClick.GetPersistentEventCount()}");
            }

            var toggle = go.GetComponent<UnityEngine.UI.Toggle>();
            if (toggle != null)
                details.AppendLine($"  Type: Toggle, IsOn: {toggle.isOn}");

            var text = go.GetComponent<UnityEngine.UI.Text>();
            if (text != null)
                details.AppendLine($"  Type: Text, Content: '{text.text}', Color: {text.color}");

            var tmpText = go.GetComponent<TMPro.TMP_Text>();
            if (tmpText != null)
                details.AppendLine($"  Type: TMP_Text, Content: '{tmpText.text}', Color: {tmpText.color}");

            var image = go.GetComponent<UnityEngine.UI.Image>();
            if (image != null)
                details.AppendLine($"  Type: Image, Color: {image.color}, Sprite: {image.sprite?.name ?? "null"}");

            var inputField = go.GetComponent<UnityEngine.UI.InputField>();
            if (inputField != null)
                details.AppendLine($"  Type: InputField, Text: '{inputField.text}', Interactable: {inputField.interactable}");

            var slider = go.GetComponent<UnityEngine.UI.Slider>();
            if (slider != null)
                details.AppendLine($"  Type: Slider, Value: {slider.value}, Interactable: {slider.interactable}");

            // 子节点数量
            details.AppendLine($"  Child count: {go.transform.childCount}");

            return details.ToString();
        });
    }

    // ─── 内部类型 ──────────────────────────────────

    [System.Serializable]
    private class UINodeSnapshot
    {
        public string name;
        public string type;
        public bool active;
        public bool interactable;
        public string text;
        public float[] rect;     // x, y, width, height
        public string color;     // hex color
        public UINodeSnapshot[] children;
    }

    // ─── 内部方法 ──────────────────────────────────

    private static UINodeSnapshot BuildSnapshot(
        GameObject go, bool includeInactive, int maxDepth, int currentDepth)
    {
        var snapshot = new UINodeSnapshot
        {
            name = go.name,
            type = GetUIComponentType(go),
            active = go.activeInHierarchy,
            interactable = GetInteractableState(go),
            text = GetTextContent(go),
            rect = GetRectValues(go),
            color = GetColorHex(go)
        };

        // 递归子节点
        if (currentDepth < maxDepth)
        {
            var childList = new List<UINodeSnapshot>();
            foreach (Transform child in go.transform)
            {
                if (!includeInactive && !child.gameObject.activeInHierarchy)
                    continue;

                // 只处理有 RectTransform 的对象（UI 元素）
                if (child.GetComponent<RectTransform>() != null)
                {
                    childList.Add(BuildSnapshot(
                        child.gameObject, includeInactive, maxDepth, currentDepth + 1));
                }
            }
            snapshot.children = childList.ToArray();
        }
        else
        {
            snapshot.children = new UINodeSnapshot[0];
        }

        return snapshot;
    }

    private static string GetUIComponentType(GameObject go)
    {
        if (go.GetComponent<UnityEngine.UI.Button>() != null) return "Button";
        if (go.GetComponent<UnityEngine.UI.Toggle>() != null) return "Toggle";
        if (go.GetComponent<UnityEngine.UI.InputField>() != null) return "InputField";
        if (go.GetComponent<TMPro.TMP_InputField>() != null) return "TMP_InputField";
        if (go.GetComponent<UnityEngine.UI.Slider>() != null) return "Slider";
        if (go.GetComponent<UnityEngine.UI.Scrollbar>() != null) return "Scrollbar";
        if (go.GetComponent<UnityEngine.UI.ScrollRect>() != null) return "ScrollRect";
        if (go.GetComponent<UnityEngine.UI.Dropdown>() != null) return "Dropdown";
        if (go.GetComponent<TMPro.TMP_Dropdown>() != null) return "TMP_Dropdown";
        if (go.GetComponent<TMPro.TMP_Text>() != null) return "TMP_Text";
        if (go.GetComponent<UnityEngine.UI.Text>() != null) return "Text";
        if (go.GetComponent<UnityEngine.UI.Image>() != null) return "Image";
        if (go.GetComponent<UnityEngine.UI.RawImage>() != null) return "RawImage";
        if (go.GetComponent<Canvas>() != null) return "Canvas";
        if (go.GetComponent<UnityEngine.UI.LayoutGroup>() != null) return "LayoutGroup";
        if (go.GetComponent<RectTransform>() != null) return "RectTransform";
        return "GameObject";
    }

    private static bool GetInteractableState(GameObject go)
    {
        var selectable = go.GetComponent<UnityEngine.UI.Selectable>();
        if (selectable != null)
            return selectable.interactable;
        return true; // 默认可交互
    }

    private static string GetTextContent(GameObject go)
    {
        var text = go.GetComponent<UnityEngine.UI.Text>();
        if (text != null) return text.text;

        var tmpText = go.GetComponent<TMPro.TMP_Text>();
        if (tmpText != null) return tmpText.text;

        var inputField = go.GetComponent<UnityEngine.UI.InputField>();
        if (inputField != null) return inputField.text;

        var tmpInput = go.GetComponent<TMPro.TMP_InputField>();
        if (tmpInput != null) return tmpInput.text;

        return null;
    }

    private static float[] GetRectValues(GameObject go)
    {
        var rt = go.GetComponent<RectTransform>();
        if (rt == null) return null;

        // 获取世界空间边界
        var corners = new Vector3[4];
        rt.GetWorldCorners(corners);

        return new float[]
        {
            corners[0].x,  // minX (x)
            corners[0].y,  // minY (y)
            corners[2].x - corners[0].x,  // width
            corners[2].y - corners[0].y   // height
        };
    }

    private static string GetColorHex(GameObject go)
    {
        var image = go.GetComponent<UnityEngine.UI.Image>();
        if (image != null) return ColorUtility.ToHtmlStringRGBA(image.color);

        var text = go.GetComponent<UnityEngine.UI.Text>();
        if (text != null) return ColorUtility.ToHtmlStringRGBA(text.color);

        var tmpText = go.GetComponent<TMPro.TMP_Text>();
        if (tmpText != null) return ColorUtility.ToHtmlStringRGBA(tmpText.color);

        return null;
    }

    private static string GetNodeDescription(GameObject go)
    {
        var type = GetUIComponentType(go);
        var text = GetTextContent(go);
        var interactable = GetInteractableState(go);

        var desc = $"{go.name} [{type}]";
        if (!string.IsNullOrEmpty(text))
            desc += $" text='{text}'";
        if (!interactable)
            desc += " (not interactable)";
        if (!go.activeInHierarchy)
            desc += " (inactive)";

        return desc;
    }

    private static void RenderTree(UINodeSnapshot node, StringBuilder sb, int depth)
    {
        var indent = new string(' ', depth * 2);
        var prefix = depth == 0 ? "■ " : "├─ ";

        sb.Append($"{indent}{prefix}{node.name} [{node.type}]");

        if (!node.active) sb.Append(" ✗inactive");
        if (!node.interactable) sb.Append(" ✗not-interactable");
        if (!string.IsNullOrEmpty(node.text)) sb.Append($" \"{node.text}\"");

        sb.AppendLine();

        if (node.children != null)
        {
            foreach (var child in node.children)
            {
                RenderTree(child, sb, depth + 1);
            }
        }
    }

    private static int CountNodes(List<UINodeSnapshot> snapshots)
    {
        int count = 0;
        foreach (var s in snapshots)
            count += CountNodesRecursive(s);
        return count;
    }

    private static int CountNodesRecursive(UINodeSnapshot node)
    {
        int count = 1;
        if (node.children != null)
        {
            foreach (var child in node.children)
                count += CountNodesRecursive(child);
        }
        return count;
    }
}
```

---

## 输出格式

### JSON 模式（默认）

```json
{
  "canvases": [
    {
      "name": "Canvas",
      "type": "Canvas",
      "active": true,
      "interactable": true,
      "text": null,
      "rect": [0, 0, 1920, 1080],
      "color": null,
      "children": [
        {
          "name": "HUDPanel",
          "type": "Image",
          "active": true,
          "interactable": true,
          "text": null,
          "rect": [0, 1000, 1920, 80],
          "color": "00000080",
          "children": [
            {
              "name": "ScoreText",
              "type": "Text",
              "active": true,
              "interactable": true,
              "text": "Score: 30",
              "rect": [10, 1010, 200, 40],
              "color": "FFFFFFFF",
              "children": []
            },
            {
              "name": "SettingsButton",
              "type": "Button",
              "active": true,
              "interactable": true,
              "text": null,
              "rect": [1750, 1010, 80, 40],
              "color": "4CAF50FF",
              "children": [
                {
                  "name": "Text",
                  "type": "Text",
                  "active": true,
                  "interactable": true,
                  "text": "Settings",
                  "rect": [1750, 1010, 80, 40],
                  "color": "FFFFFFFF",
                  "children": []
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

### Tree 模式（人类可读）

```
■ Canvas [Canvas]
  ├─ HUDPanel [Image] ✗not-interactable
  │  ├─ ScoreText [Text] "Score: 30"
  │  ├─ SettingsButton [Button]
  │  │  └─ Text [Text] "Settings"
  │  └─ LivesText [Text] "Lives: 5"
  ├─ SettingsPanel [Image] ✗inactive
  │  └─ CloseButton [Button]
  └─ GameOverPanel [Image] ✗inactive
     └─ RestartButton [Button]
```

---

## 验证断言模式

Codex / 自动化审查可以用 JSON 快照做精确断言：

```json
// 断言示例：验证分数显示正确
{
  "assert": "ui_tree.find('ScoreText').text == 'Score: 30'",
  "assert": "ui_tree.find('SettingsButton').interactable == true",
  "assert": "ui_tree.find('SettingsPanel').active == false"
}
```

对比截图验证 vs UI 快照断言：

| 维度 | 截图验证 | UI 快照断言 |
|------|----------|------------|
| 分辨率依赖 | ❌ 是 | ✅ 否 |
| 帧时机 | ❌ 可能截到过渡帧 | ✅ 数据是即时的 |
| 大模型判断 | ❌ 容易误判 | ✅ 精确匹配 |
| 视觉效果 | ✅ 可验证 | ❌ 无法验证 |
| 文本内容 | ❌ OCR 可能出错 | ✅ 100% 精确 |
| 交互状态 | ❌ 看不出 interactable | ✅ 直接可断言 |

**最佳实践：截图 + UI 快照组合使用**——截图验证视觉效果，快照断言数据正确性。

---

## 与 Wait-For 工具配合

```bash
# 1. 点击按钮
simulate-click-ui x=150 y=100

# 2. 等待 UI 布局重建
wait-for-frame-count frameCount=2

# 3. 抓取 UI 快照（此时布局已稳定）
ui-hierarchy-snapshot format="json"

# 4. 精确断言：SettingsPanel 应该激活
ui-element-find name="SettingsPanel"
# 期望：active: true
```

---

## 注意事项

1. **uGUI 优先**：当前仅支持 uGUI（UnityEngine.UI），暂不支持 UI Toolkit
2. **TextMeshPro**：已支持 TMP_Text 和 TMP_InputField 的文本内容抓取
3. **性能**：大型 UI 树（1000+ 节点）的 JSON 序列化可能需要 50-100ms
4. **层级深度**：建议设置 `maxDepth=10` 避免过深的遍历
5. **inactive 节点**：默认不包含，如需检查面板是否正确隐藏，设置 `includeInactive=true`
