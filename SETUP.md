# Setup Guide (v2.2)

Complete setup instructions for Unity-MCP Validator. Follow this guide to make the skill fully functional.

## Quick Setup Checklist

Complete these steps before using the validator:

| Step | Status | Command/Action |
|------|--------|----------------|
| 1. Install Unity-MCP plugin | ⬜ | `npm install -g unity-mcp-cli && unity-mcp-cli install-plugin ./YourProject` |
| 2. Copy custom input tools | ⬜ | Copy code from `references/custom-tools-input.md` |
| 3. **Add wait tools (v2.0)** | ⬜ | Copy code from `references/custom-tools-wait.md` |
| 4. **Add UI snapshot tools (v2.2)** | ⬜ | Copy code from `references/ui-snapshot-tool.md` |
| 5. **Add state reset tool (v2.0)** | ⬜ | Copy code from `references/state-reset.md` |
| 6. **Add VL screenshot tool (v2.2)** | ⬜ (optional) | Copy code from `references/custom-tools-screenshot-vl.md` |
| 7. **Add test backdoor API (v2.0)** | ⬜ (optional) | See `references/test-backdoors.md` |
| 8. Configure MCP server | ⬜ | Add to Claude Code MCP config |
| 9. Test connection | ⬜ | Call `editor-application-get-state` |
| 10. Create config file | ⬜ (optional) | Copy `validation-config.example.yaml` |

---

## Step 1: Install Unity-MCP Plugin

### Method A: Via CLI (Recommended)

```bash
# Install CLI tool globally
npm install -g unity-mcp-cli

# Install plugin into your Unity project
unity-mcp-cli install-plugin ./YourUnityProject
```

### Method B: Manual Installation

1. Download from https://github.com/IvanMurzak/Unity-MCP
2. Copy `UnityMcpPlugin` folder to `YourUnityProject/Assets/Scripts/MCP/`
3. Wait for Unity to compile

### Verify Installation

After installation, your project should have:

```
YourUnityProject/
└── Assets/
    └── Scripts/
        └── MCP/
            ├── UnityMcpPlugin/
            │   ├── Editor/
            │   ├── Runtime/
            │   └── ...
            └── ...
```

Unity Console should show:
```
[MCP] Plugin loaded successfully
[MCP] Server ready
```

---

## Step 2: Add Custom Input Tools

Unity-MCP doesn't have built-in input simulation. Add these tools for PlayMode validation.

### Create Scripts Folder

```
YourUnityProject/Assets/Scripts/MCP/
├── Tool_MouseInput.cs
├── Tool_MouseUI.cs
├── Tool_KeyboardInput.cs
└── Tool_InputRecording.cs
```

### Copy Code

Open `references/custom-tools-input.md` and copy each Tool's code into corresponding files:

| File | Tool Functions |
|------|----------------|
| `Tool_MouseInput.cs` | `simulate-click-world`, `simulate-mouse-down/up`, `simulate-mouse-move`, `simulate-drag-world` |
| `Tool_MouseUI.cs` | `simulate-click-ui`, `simulate-hover-ui`, `click-button-by-name` |
| `Tool_KeyboardInput.cs` | `simulate-key-press/down/up`, `simulate-text-input` |
| `Tool_InputRecording.cs` | `record-start/stop/summary`, `replay-input`, `record-export/import` |

### Verify Custom Tools

After Unity compiles, test in Unity Console:

```csharp
// Run in Unity Console or via script-execute
Debug.Log("Custom tools installed: simulate-click-world available");
```

Or via MCP: call `simulate-click-world x=100 y=100` — should return `[Success]` or `[Error]` with explanation.

---

## Step 3: Add Wait Tools (v2.0 新增)

**解决时序不同步问题：** 交互操作后必须等待状态稳定，禁止写死 Thread.Sleep。

### Copy Code

Open `references/custom-tools-wait.md` and copy the Tool code:

| File | Tool Functions |
|------|----------------|
| `Tool_WaitUntil.cs` | `wait-until-condition`, `wait-for-animation-state`, `wait-for-frame-count`, `wait-for-stable` |

### Place in Project

```
YourUnityProject/Assets/Scripts/MCP/
└── Tool_WaitUntil.cs
```

### Verify

```bash
# MCP 调用测试
wait-until-condition condition="true" timeoutSeconds=1
# 期望返回：Condition met immediately
```

---

## Step 4: Add UI Snapshot Tools (v2.0 新增)

**解决截图断言盲区：** UI 验证使用数据断言而非靠大模型判断截图。

### Copy Code

Open `references/ui-snapshot-tool.md` and copy the Tool code:

| File | Tool Functions |
|------|----------------|
| `Tool_UISnapshot.cs` | `ui-hierarchy-snapshot`, `ui-element-find`, `ui-element-at-position` |

### Place in Project

```
YourUnityProject/Assets/Scripts/MCP/
└── Tool_UISnapshot.cs
```

### Verify

```bash
# MCP 调用测试
ui-hierarchy-snapshot format="summary"
# 期望返回：Canvas 层级结构摘要

ui-element-find name="Canvas"
# 期望返回：Canvas 元素信息
```

---

## Step 5: Add State Reset Tool (v2.0 新增)

**消除测试间状态污染：** 多个 PlayMode 测试用例之间必须重置状态。

### Copy Code

Open `references/state-reset.md` and copy the Tool code and helper class:

| File | Purpose |
|------|---------|
| `Tool_StateReset.cs` | MCP 工具：`state-reset` |
| `TestHelper.cs` | 项目级重置辅助类（需按项目自定义） |

### Place in Project

```
YourUnityProject/Assets/Scripts/MCP/
└── Tool_StateReset.cs

YourUnityProject/Assets/Scripts/
└── TestHelper.cs      ← 按项目自定义 ResetAll() 方法
```

### Customize TestHelper

根据你的项目自定义 `TestHelper.ResetAll()`：

```csharp
// TestHelper.cs — 按你的项目修改
#if UNITY_EDITOR
public static class TestHelper
{
    public static void ResetAll()
    {
        // 重置单例
        if (GameController.Instance != null)
            GameController.Instance.ResetState();

        // 重置分数
        PlayerPrefs.DeleteAll();

        // 重置其他全局状态...
    }
}
#endif
```

### Verify

```bash
# MCP 调用测试
state-reset strategy="auto"
# 期望返回：State reset successfully
```

---

## Step 6: Add VL Screenshot Tool (v2.2 新增，可选)

**集成 Ollama VL 模型进行截图识别：** 自动化视觉验证，判断 UI 状态、检测游戏画面。

### Prerequisites

1. **Ollama 服务**：确保 Ollama 运行并部署了 VL 模型
2. **Newtonsoft.Json 包**：Unity 项目需要安装

```
Package Manager → Add package from git URL:
com.unity.nuget.newtonsoft-json
```

### Copy Code

Open `references/custom-tools-screenshot-vl.md` and copy the Tool code:

| File | Purpose |
|------|---------|
| `Tool_ScreenshotVL.cs` | 截图 + VL 分析工具 |

### Place in Project

```
YourUnityProject/Assets/Scripts/MCP/
└── Tool_ScreenshotVL.cs
```

### Configure

修改默认配置（或通过 MCP 工具配置）：

```csharp
// 默认配置（在代码中修改）
private static string ollamaUrl = "http://192.168.0.103:11434";
private static string ollamaModel = "huihui_ai/qwen3-vl-abliterated:8b-instruct";
```

或通过 MCP 工具动态配置：

```bash
vl-config url="http://192.168.0.103:11434" model="huihui_ai/qwen3-vl-abliterated:8b-instruct"
```

### Verify

```bash
# MCP 调用测试
screenshot-analyze-vl prompt="Describe what you see"
# 期望返回：VL 模型的描述

# 快速 UI 检查
ui-check-vl elementName="StartButton" checkType="visible"
# 期望返回：YES 或 NO
```

---

## Step 7: Add Test Backdoor API (v2.0 新增，可选)

**极速构造边界测试场景：** 通过 reflection-method-call 调用后门方法。

详见 `references/test-backdoors.md`。

### 创建后门类

```csharp
// BoardTestHelper.cs — 按你的项目修改
#if UNITY_EDITOR
public static class BoardTestHelper
{
    public static void SetupChainExplosion()
    {
        // 构造"即将触发连锁爆炸"的棋盘布局
    }

    public static void SetupNoValidMoves()
    {
        // 构造"无有效移动"的边界场景
    }
}
#endif
```

### Verify

```bash
# MCP 调用测试
reflection-method-call typeName="TestHelper" methodName="ResetAll"
# 期望返回：方法调用成功
```

---

## Step 7: Configure MCP Server

### Claude Code Configuration

Add Unity-MCP to your Claude Code MCP settings file:

**Location**: `~/.claude/mcp.json` (or Windows: `%USERPROFILE%\.claude\mcp.json`)

```json
{
  "mcpServers": {
    "unity-mcp": {
      "command": "unity-mcp-server",
      "args": ["--port", "8080"],
      "env": {}
    }
  }
}
```

### Alternative: Via Claude Desktop Config

If using Claude Desktop app:

**Location**: 
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "unity-mcp": {
      "command": "unity-mcp-server",
      "args": ["--port", "8080"]
    }
  }
}
```

### Start MCP Server

1. Open Unity Editor with your project
2. MCP server starts automatically when plugin loads
3. Verify server is running: check Unity Console for `[MCP] Server listening on port 8080`

---

## Step 8: Test Connection

### Quick Test via MCP Tool

Call any Unity-MCP tool to verify connection:

```
editor-application-get-state
```

Expected response:
```json
{
  "isPlaying": false,
  "isPaused": false,
  "isCompiling": false
}
```

### If Connection Fails

| Error | Solution |
|-------|----------|
| "MCP server not found" | Restart Claude Code, check MCP config |
| "Unity not connected" | Open Unity Editor, wait for plugin to load |
| "Port blocked" | Check firewall, change port in config |
| "Plugin not loaded" | Re-install plugin, check Unity Console for errors |

---

## Step 9: Create Config File (Optional)

### Copy Template

```bash
cp validation-config.example.yaml YourUnityProject/validation-config.yaml
```

### Customize

Edit `validation-config.yaml` for your project — see next section for v2.0 config options.

---

## Configuration Reference (v2.0)

### Core Config

```yaml
config_version: "2.0"

# Key behaviors that must not regress
invariants:
  - name: "input_lock_during_animation"
    description: "Input must stay locked during critical animations"
    check_hint: "Use reflection-method-call to check InputLocked during PlayMode"

# Core classes for runtime probing
key_classes:
  controller:
    class_name: "MyGameController"
    assembly: "Assembly-CSharp"
  state_machine:
    enum_name: "MyGamePhase"
  input_lock:
    class_name: "MyInputManager"
    property_name: "IsLocked"

# File path patterns for routing hints
file_patterns:
  model_rules:
    - "Scripts/Model/**"
  controller_state_machine:
    - "Scripts/Controller/**"
```

### v2.0 New Config Sections

```yaml
# Custom tools availability (v2.0)
custom_tools:
  input_simulation_installed: true
  wait_tools_installed: true        # NEW: wait-until-condition 等
  ui_snapshot_installed: true       # NEW: ui-hierarchy-snapshot 等
  state_reset_installed: true       # NEW: state-reset
  custom_tools_path: "Assets/Scripts/MCP/"

# State reset strategy (v2.0)
state_reset:
  default_strategy: "auto"          # auto | reload | custom
  custom_method: "TestHelper.ResetAll"
  confirm_condition: "GameController.Instance != null"
  timeout_seconds: 5

# Test backdoor API (v2.0)
test_backdoors:
  - class_name: "TestHelper"
    methods:
      - name: "ResetAll"
        description: "Reset all game state"
      - name: "SkipTutorial"
        description: "Skip tutorial to test core logic"
  - class_name: "BoardTestHelper"
    methods:
      - name: "SetupChainExplosion"
        description: "Setup board for chain explosion"
      - name: "SetupNoValidMoves"
        description: "Setup board with no valid moves"
  - class_name: "EconomyTestHelper"
    methods:
      - name: "SetupResourceScarce"
        description: "Setup extremely scarce resources"
  - class_name: "UITestHelper"
    methods:
      - name: "ForceGameOver"
        description: "Force game over screen"

# Wait defaults (v2.0)
wait_defaults:
  default_timeout_seconds: 5
  animation_timeout_seconds: 10
  stable_check_frames: 10
  frame_count_for_ui: 2

# Output format (v2.0)
output:
  dual_format: true                 # Both Markdown + JSON
  json_schema_version: "2.0"
  include_state_snapshots: true
  include_ui_snapshots: true
```

---

## Agent-Ready Setup Commands

For AI agents, execute these commands in sequence:

```bash
# 1. Install Unity-MCP CLI (if not installed)
npm install -g unity-mcp-cli

# 2. Install plugin into project
unity-mcp-cli install-plugin ./YourUnityProject

# 3. Create MCP scripts folder
mkdir -p YourUnityProject/Assets/Scripts/MCP

# 4. Copy custom input tools (read and write these files)
# Read: references/custom-tools-input.md
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_MouseInput.cs
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_MouseUI.cs
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_KeyboardInput.cs
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_InputRecording.cs

# 5. Copy wait tools (v2.0)
# Read: references/custom-tools-wait.md
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_WaitUntil.cs

# 6. Copy UI snapshot tools (v2.0)
# Read: references/ui-snapshot-tool.md
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_UISnapshot.cs

# 7. Copy state reset tool (v2.0)
# Read: references/state-reset.md
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_StateReset.cs

# 8. Create TestHelper.cs (customize for project)
# Write: YourUnityProject/Assets/Scripts/TestHelper.cs

# 9. Wait for Unity to compile (check Unity Console)

# 10. Test connection
# MCP call: editor-application-get-state
# Expected: {"isPlaying": false, ...}

# 11. Test v2.0 tools
# MCP call: wait-until-condition condition="true" timeoutSeconds=1
# MCP call: ui-hierarchy-snapshot format="summary"
# MCP call: state-reset strategy="auto"
```

---

## Verification Checklist (v2.0)

After setup, verify each component works:

| Check | MCP Tool Call | Expected Result |
|-------|---------------|-----------------|
| Unity connected | `editor-application-get-state` | `{"isPlaying": false}` |
| Can compile | `script-execute "return 1;"` | `[Success] 1` |
| Scene access | `scene-list-opened` | List of open scenes |
| GameObject find | `gameobject-find name="Camera"` | Camera object info |
| Reflection works | `reflection-method-find typeName="UnityEngine.Debug"` | Debug class methods |
| Custom input tools | `simulate-click-world x=100 y=100` | `[Success]` or raycast info |
| Screenshots | `screenshot-game-view` | Image path returned |
| **Wait tools** | **`wait-until-condition condition="true" timeoutSeconds=1`** | **Condition met** |
| **UI snapshot** | **`ui-hierarchy-snapshot format="summary"`** | **Canvas hierarchy** |
| **State reset** | **`state-reset strategy="auto"`** | **State reset success** |
| **Test backdoor** | **`reflection-method-call typeName="TestHelper" methodName="ResetAll"`** | **Method called** |
| **VL screenshot (v2.2)** | **`vl-config` + `screenshot-analyze-vl`** | **VL response** |

---

## Troubleshooting Quick Reference

| Issue | First Check | Fix |
|-------|-------------|-----|
| MCP tools not available | MCP config file | Add unity-mcp server |
| Unity not responding | Unity Editor open? | Open project in Unity |
| Custom input tools missing | Scripts/MCP folder? | Copy from custom-tools-input.md |
| **Wait tools missing** | **Tool_WaitUntil.cs?** | **Copy from custom-tools-wait.md** |
| **UI snapshot missing** | **Tool_UISnapshot.cs?** | **Copy from ui-snapshot-tool.md** |
| **State reset failing** | **TestHelper.cs configured?** | **Customize ResetAll() for project** |
| **VL tool missing** | **Tool_ScreenshotVL.cs?** | **Copy from custom-tools-screenshot-vl.md** |
| **VL connection failed** | **Ollama running?** | **Start Ollama, check URL** |
| **VL model error** | **Model name correct?** | **Use full name: huihui_ai/qwen3-vl-abliterated:8b-instruct** |
| Input System errors | Package installed? | Install Input System package |
| Compilation errors | Unity Console | Fix script errors first |
| Port conflict | MCP config port | Change port number |

See [references/troubleshooting.md](references/troubleshooting.md) for detailed solutions.

---

## One-Time Setup vs Per-Project

| Setup | Scope | Repeat? |
|-------|-------|---------|
| Install unity-mcp-cli | Global | Once per machine |
| MCP server config | Claude Code | Once per Claude instance |
| Unity-MCP plugin | Per project | Each new project |
| Custom input tools | Per project | Each new project |
| **Wait tools** | Per project | Each new project |
| **UI snapshot tools** | Per project | Each new project |
| **State reset tool** | Per project | Each new project |
| **Test backdoor API** | Per project | Each new project (customize) |
| validation-config.yaml | Per project | Each new project |

---

## Minimal Setup for Quick Testing

If you want to test quickly without full setup:

1. Install plugin: `unity-mcp-cli install-plugin ./YourProject`
2. Open Unity Editor
3. Call `script-execute` or `editor-application-get-state`

This works for Layer 1-3 validation. For PlayMode input testing, add custom input tools.

**For v2.0 full experience, also add:**
- Wait tools (solve timing issues)
- UI snapshot tools (solve screenshot assertion blind spots)
- State reset tool (solve test isolation)

---

## Tool Installation Priority

If you can't install everything at once, install in this order:

| Priority | Tool | Why |
|----------|------|-----|
| 1️⃣ | Unity-MCP plugin + input tools | Without these, no PlayMode validation |
| 2️⃣ | Wait tools | Without these, timing-dependent tests are flaky |
| 3️⃣ | UI snapshot tools | Without these, UI validation is unreliable |
| 4️⃣ | State reset tool | Without these, sequential tests contaminate each other |
| 5️⃣ | Test backdoor API | Optional — for edge case / boundary testing |
