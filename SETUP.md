# Setup Guide

Complete setup instructions for Unity-MCP Validator. Follow this guide to make the skill fully functional.

## Quick Setup Checklist

Complete these steps before using the validator:

| Step | Status | Command/Action |
|------|--------|----------------|
| 1. Install Unity-MCP plugin | ⬜ | `npm install -g unity-mcp-cli && unity-mcp-cli install-plugin ./YourProject` |
| 2. Copy custom input tools | ⬜ | Copy code from `references/custom-tools-input.md` |
| 3. Configure MCP server | ⬜ | Add to Claude Code MCP config |
| 4. Test connection | ⬜ | Call `editor-application-get-state` |
| 5. Create config file | ⬜ (optional) | Copy `validation-config.example.yaml` |

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

## Step 3: Configure MCP Server

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

## Step 4: Test Connection

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

## Step 5: Create Config File (Optional)

### Copy Template

```bash
cp validation-config.example.yaml YourUnityProject/validation-config.yaml
```

### Customize

Edit `validation-config.yaml` for your project:

```yaml
key_classes:
  controller:
    class_name: "MyGameController"  # Your main controller
    assembly: "Assembly-CSharp"
  state_machine:
    enum_name: "MyGamePhase"        # Your phase enum
  input_lock:
    class_name: "MyInputManager"
    property_name: "IsLocked"

file_patterns:
  model_rules:
    - "Scripts/Model/**"
    - "Scripts/GameLogic/**"
  controller_state_machine:
    - "Scripts/Controller/**"
    - "Scripts/Flow/**"
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

# 4. Copy custom tools (agent should read and write these files)
# Read: references/custom-tools-input.md
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_MouseInput.cs
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_MouseUI.cs
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_KeyboardInput.cs
# Write: YourUnityProject/Assets/Scripts/MCP/Tool_InputRecording.cs

# 5. Wait for Unity to compile (check Unity Console)

# 6. Test connection
# MCP call: editor-application-get-state
# Expected: {"isPlaying": false, ...}
```

---

## Verification Checklist

After setup, verify each component works:

| Check | MCP Tool Call | Expected Result |
|-------|---------------|-----------------|
| Unity connected | `editor-application-get-state` | `{"isPlaying": false}` |
| Can compile | `script-execute "return 1;"` | `[Success] 1` |
| Scene access | `scene-list-opened` | List of open scenes |
| GameObject find | `gameobject-find name="Camera"` | Camera object info |
| Reflection works | `reflection-method-find typeName="UnityEngine.Debug"` | Debug class methods |
| Custom tools | `simulate-click-world x=100 y=100` | `[Success]` or raycast info |
| Screenshots | `screenshot-game-view` | Image path returned |

---

## Troubleshooting Quick Reference

| Issue | First Check | Fix |
|-------|-------------|-----|
| MCP tools not available | MCP config file | Add unity-mcp server |
| Unity not responding | Unity Editor open? | Open project in Unity |
| Custom tools missing | Scripts/MCP folder? | Copy from custom-tools-input.md |
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
| validation-config.yaml | Per project | Each new project |

---

## Minimal Setup for Quick Testing

If you want to test quickly without full setup:

1. Install plugin: `unity-mcp-cli install-plugin ./YourProject`
2. Open Unity Editor
3. Call `script-execute` or `editor-application-get-state`

This works for Layer 1-3 validation. For PlayMode input testing, add custom tools.