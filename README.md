# Unity MCP Validator

A Claude Code skill that routes Unity game development changes to the correct validation flow using Unity-MCP tools.

## What It Does

Instead of ad-hoc "run some tests" behavior, this skill provides systematic validation via Unity-MCP:

1. **Classify** the change type (model, controller, input, animation, scene, UI, etc.)
2. **Route** to appropriate validation layers (static review → EditMode → Editor → PlayMode)
3. **Execute** required layers using 100+ Unity-MCP tools
4. **Probe** runtime bugs with reflection and dynamic code execution
5. **Report** with explicit verdicts and blockers

## Unity-MCP Advantages

| Feature | Description |
|---------|-------------|
| **Reflection system** | `reflection-method-find/call` — find and call any C# method (including private) |
| **Dynamic execution** | `script-execute` — compile and run C# code snippets (Roslyn) |
| **Runtime embedding** | Works in compiled games, not just Editor |
| **100+ tools** | Assets, scenes, GameObjects, components, materials, prefabs, packages, tests, screenshots, logs |
| **Custom tools** | Add new tools with single line of code |

## Installation

**See [SETUP.md](SETUP.md) for complete setup instructions.**

### Quick Setup Checklist

| Step | Command |
|------|---------|
| 1. Install Unity-MCP plugin | `npm install -g unity-mcp-cli && unity-mcp-cli install-plugin ./YourProject` |
| 2. Add custom input tools | Copy code from `references/custom-tools-input.md` |
| 3. Configure MCP server | Add to Claude Code MCP config (see SETUP.md) |
| 4. Test connection | Call `editor-application-get-state` |
| 5. Create config (optional) | Copy `validation-config.example.yaml` |

## Skills Reference

### Layer 1: Static Review
No MCP tools needed — read diff and code.

### Layer 2: EditMode Tests
| Tool | Purpose |
|------|---------|
| `tests-run mode=EditMode` | Run EditMode unit tests |

### Layer 3: Editor Inspection
| Tool | Purpose |
|------|---------|
| `script-execute` | Compile and run code snippets |
| `scene-get-data` | Get scene hierarchy |
| `gameobject-find` | Find GameObjects |
| `gameobject-component-get` | Get component details |
| `assets-get-data` | Get asset/prefab data |
| `console-get-logs` | Get Unity Console logs |

### Layer 4: PlayMode Automation
| Tool | Purpose |
|------|---------|
| `editor-application-set-state` | Enter/exit PlayMode |
| `screenshot-game-view` | Capture game screenshots |
| `reflection-method-call` | Call any C# method at runtime |
| `reflection-method-find` | Find methods to call |

## Custom Tools (Input Simulation)

Unity-MCP doesn't have built-in input simulation. Add these custom tools:

| Tool | Purpose |
|------|---------|
| `simulate-click-world` | Click world-space objects (colliders) |
| `simulate-click-ui` | Click UI elements (EventSystem) |
| `simulate-key-press` | Press keyboard key |
| `record-start/stop` | Record input sequence |
| `replay-input` | Replay recorded input |

See `references/custom-tools-input.md` for implementation code.

## Configuration

Copy `validation-config.example.yaml` to project root:

```yaml
invariants:
  - name: "input_lock_during_animation"
    description: "Input must stay locked during critical animations"

key_classes:
  controller:
    class_name: "GameController"
  input_lock:
    class_name: "InputManager"
    property_name: "IsLocked"

file_patterns:
  model_rules:
    - "Scripts/Model/**"
  controller_state_machine:
    - "Scripts/Controller/**"
```

## Usage

Invoke the skill in Claude Code:

```
Use $unity-mcp-validator to validate the controller changes
```

Or for pre-acceptance sweep:

```
Run full pre-acceptance validation using $unity-mcp-validator
```

## Output Format

```text
Route: Input / interaction + PlayMode runtime regression
Layers run:
- Static review: ran
- EditMode: skipped by route
- Unity Editor inspection: ran
- PlayMode automation: ran

Unity-MCP tools used:
- script-execute: compiled successfully
- scene-get-data: hierarchy valid
- reflection-method-call: controller.CurrentPhase = 'Processing'

Evidence: [screenshots, logs, state snapshots]

Verdict: Acceptable now / Conditionally acceptable / Not acceptable yet
Next action: [most useful Unity-MCP command]
```

## Troubleshooting

See [SETUP.md](SETUP.md) for setup issues.
See [references/troubleshooting.md](references/troubleshooting.md) for runtime issues.

## Version

Current version: **1.1.0**

See [CHANGELOG.md](CHANGELOG.md) for version history.

## License

MIT License - see LICENSE file.

## Credits

Based on [Unity-MCP](https://github.com/IvanMurzak/Unity-MCP) by IvanMurzak.