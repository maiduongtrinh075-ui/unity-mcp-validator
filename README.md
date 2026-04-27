# Unity MCP Validator

A Claude Code skill that routes Unity game development changes to the correct validation flow using Unity-MCP tools.

## What It Does

Instead of ad-hoc "run some tests" behavior, this skill provides systematic validation via Unity-MCP:

1. **Classify** the change type (model, controller, input, animation, scene, UI, etc.)
2. **Route** to appropriate validation layers (static review → EditMode → Editor → PlayMode)
3. **Execute** required layers using 100+ Unity-MCP tools
4. **Probe** runtime bugs with reflection and dynamic code execution
5. **Report** with explicit verdicts and blockers (Markdown + JSON dual output)

## v2.2 Highlights

| Feature | Problem Solved | Solution |
|---------|---------------|----------|
| **Click/Drag by Label** | Manual coordinate input is tedious | `simulate-click-by-label label="B"` — auto-find coords |
| **Click/Drag by Name** | Need to know exact coordinates | `simulate-click-by-name name="StartButton"` — direct operation |
| **UI element finder** | Cannot find draggable | `ui-annotate-elements` — get SimX/SimY for all elements |
| **Split drag API** | One-shot drag is unstable | `simulate-drag-ui-start/move/end` — step-by-step |
| **Async wait tools** | Thread.Sleep is flaky | `wait-until-condition`, etc. |

## Unity-MCP Advantages

| Feature | Description |
|---------|-------------|
| **Reflection system** | `reflection-method-find/call` — find and call any C# method (including private) |
| **Dynamic execution** | `script-execute` — compile and run C# code snippets (Roslyn) |
| **Runtime embedding** | Works in compiled games, not just Editor |
| **100+ tools** | Assets, scenes, GameObjects, components, materials, prefabs, packages, tests, screenshots, logs |
| **Custom tools** | Add new tools with single line of code |

## Quick Start

### 1. Install as Claude Code Skill

```bash
# Clone the repo
git clone https://github.com/maiduongtrinh075-ui/unity-mcp-validator.git

# Copy (or symlink) into your Claude Code skills directory
cp -r unity-mcp-validator ~/.claude/skills/unity-mcp-validator
# Or for project-level: cp -r unity-mcp-validator .claude/skills/unity-mcp-validator
```

After cloning, Claude Code will automatically load `SKILL.md` when this skill is triggered.

### 2. Sync Updates

```bash
# Pull latest changes from upstream
cd ~/.claude/skills/unity-mcp-validator && git pull origin master
```

### 3. Full Setup

**See [SETUP.md](SETUP.md) for complete setup instructions (Unity-MCP plugin, custom tools, config).**

### Quick Setup Checklist

| Step | Command |
|------|---------|
| 1. Install Unity-MCP plugin | `npm install -g unity-mcp-cli && unity-mcp-cli install-plugin ./YourProject` |
| 2. Add custom input tools | Copy code from `references/custom-tools-input.md` |
| 3. **Add wait tools (v2.0)** | Copy code from `references/custom-tools-wait.md` |
| 4. **Add UI snapshot tools (v2.0)** | Copy code from `references/ui-snapshot-tool.md` |
| 5. **Add state reset tool (v2.0)** | Copy code from `references/state-reset.md` |
| 6. Configure MCP server | Add to Claude Code MCP config (see SETUP.md) |
| 7. Test connection | Call `editor-application-get-state` |
| 8. Create config (optional) | Copy `validation-config.example.yaml` |

## Validation Layers

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
| **`ui-hierarchy-snapshot`** | **UI DOM tree structure snapshot** |

### Layer 4: PlayMode Automation (v2.0 Enhanced)

Layer 4 now includes 5 sub-layers:

| Sub-layer | Purpose | Key Tools |
|-----------|---------|-----------|
| **4A-Pre** | State reset | `state-reset`, `reflection-method-call` |
| **4A** | PlayMode enter/exit | `editor-application-set-state` |
| **4B** | Input simulation | `simulate-click-ui`, `simulate-drag-world`, `simulate-key-press` |
| **4B-Post** | Wait for stability | `wait-until-condition`, `wait-for-animation-state` |
| **4C** | Evidence collection | `screenshot-game-view`, `ui-hierarchy-snapshot`, `reflection-method-call` |

**Complete validation flow (Match3 sliding example):**
```
[4A-Pre] state-reset → [4A] Enter PlayMode → [4A] wait-until-condition →
[4C] baseline screenshot + UI snapshot →
[4B] simulate-drag-world → [4B-Post] wait-until-condition →
[4C] result screenshot + UI snapshot + reflection probe →
[4A] Exit PlayMode
```

**See [EXAMPLES.md](EXAMPLES.md) for detailed examples.**

## v2.0 New Tools

### Async Wait Tools
| Tool | Purpose |
|------|---------|
| `wait-until-condition` | Poll C# condition until true or timeout |
| `wait-for-animation-state` | Wait for Animator to reach state |
| `wait-for-frame-count` | Wait N frames (for UI layout rebuild) |
| `wait-for-stable` | Wait for condition + no new errors |

### UI DOM Snapshot Tools
| Tool | Purpose |
|------|---------|
| `ui-hierarchy-snapshot` | Capture Canvas UI tree structure |
| `ui-element-find` | Find specific UI element by name |
| `ui-element-at-position` | Find UI element at screen position |

### State Reset Tool
| Tool | Purpose |
|------|---------|
| `state-reset` | Reset game to clean state between test cases |

### Test Backdoor API
Not a standalone MCP tool — project C# methods called via `reflection-method-call`:
```bash
reflection-method-call typeName="TestHelper" methodName="SkipTutorial"
reflection-method-call typeName="BoardTestHelper" methodName="SetupChainExplosion"
```

## Configuration

Copy `validation-config.example.yaml` to project root. v2.0 adds:

```yaml
custom_tools:
  wait_tools_installed: true
  ui_snapshot_installed: true
  state_reset_installed: true

state_reset:
  default_strategy: "auto"
  custom_method: "TestHelper.ResetAll"

test_backdoors:
  - class_name: "BoardTestHelper"
    methods:
      - name: "SetupChainExplosion"

wait_defaults:
  default_timeout_seconds: 5
  animation_timeout_seconds: 10

output:
  dual_format: true
  json_schema_version: "2.0"
```

## Output Format (v2.0 Dual Track)

Every validation produces two reports:

| Format | Consumer | Purpose |
|--------|----------|---------|
| Markdown | Human developer | Quick review, debugging |
| JSON | CI/CD, Codex | Automated pass/fail, tracking |

See [references/output-contract.md](references/output-contract.md) for Markdown format.
See [references/json-output-schema.md](references/json-output-schema.md) for JSON schema.

## Documentation Index

| File | Description |
|------|-------------|
| [SKILL.md](SKILL.md) | Skill definition and core contract |
| [SETUP.md](SETUP.md) | Complete setup guide |
| [EXAMPLES.md](EXAMPLES.md) | Validation examples (v2.0 enhanced) |
| [CHANGELOG.md](CHANGELOG.md) | Version history |
| [validation-config.example.yaml](validation-config.example.yaml) | Config template (v2.2) |
| [references/route-matrix.md](references/route-matrix.md) | Change classification → validation route |
| [references/output-contract.md](references/output-contract.md) | Markdown report format |
| [references/json-output-schema.md](references/json-output-schema.md) | JSON report schema (v2.0) |
| [references/runtime-probes.md](references/runtime-probes.md) | Runtime bug probe ladder |
| [references/tool-reference.md](references/tool-reference.md) | Unity-MCP tool reference |
| [references/troubleshooting.md](references/troubleshooting.md) | Troubleshooting guide |
| [references/custom-tools-input.md](references/custom-tools-input.md) | Input simulation tool code (v2.2) |
| [references/custom-tools-wait.md](references/custom-tools-wait.md) | Wait tool code (v2.0) |
| [references/ui-snapshot-tool.md](references/ui-snapshot-tool.md) | UI snapshot & annotate tool code (v2.2) |
| [references/state-reset.md](references/state-reset.md) | State reset tool code (v2.0) |
| [references/custom-tools-screenshot-vl.md](references/custom-tools-screenshot-vl.md) | **Screenshot + VL analysis tool (v2.2 NEW)** |
| [references/test-backdoors.md](references/test-backdoors.md) | Test backdoor API guide (v2.0) |

## Troubleshooting

See [SETUP.md](SETUP.md) for setup issues.
See [references/troubleshooting.md](references/troubleshooting.md) for runtime issues.

## Version

Current version: **2.2.0**

See [CHANGELOG.md](CHANGELOG.md) for version history.

## License

MIT License - see LICENSE file.

## Credits

Based on [Unity-MCP](https://github.com/IvanMurzak/Unity-MCP) by IvanMurzak.
