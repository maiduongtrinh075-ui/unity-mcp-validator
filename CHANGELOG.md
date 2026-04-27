# Changelog

All notable changes to this skill will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.1.0] - 2026-04-27

### Added — v2.1: Enhanced Input Tools (参考 unity-cli-loop)

**Problem: UI Drag 不稳定、找不到 draggable 元素**

- `references/custom-tools-input.md` — 大幅增强输入模拟工具
  - **NEW: `ui-find-interactive-elements`** — 查找所有可交互 UI 元素，返回 SimX/SimY 坐标（解决坐标不准问题）
  - **NEW: `ui-find-element-by-name`** — 按名称查找 UI 元素坐标
  - **NEW: `simulate-drag-ui-start/move/end`** — 分步 Drag API（解决一次性拖拽不稳定问题）
  - **NEW: `simulate-long-press-ui`** — 长按 UI 元素（不触发 click）
  - **NEW: `simulate-mouse-scroll`** — 鼠标滚轮模拟
  - **NEW: `simulate-mouse-delta`** — FPS 相机控制（鼠标增量）
  - **NEW: `simulate-long-press-world`** — 世界空间长按（挖掘/蓄力）

**Enhanced Error Handling:**
- 检查 EventSystem 是否存在
- 检查元素是否可拖拽（IDragHandler）
- 检查元素是否可点击（IPointerClickHandler）
- 检查 Button 是否 interactable

**Recommended Workflow:**
```
# 先获取坐标，再操作
ui-find-interactive-elements → 获取 SimX/SimY
simulate-drag-ui-start x=<SimX> y=<SimY>
simulate-drag-ui-move x=...
simulate-drag-ui-end x=...
```

### Fixed

- SKILL.md — 删除重复的 `## Quick Start` 标题，修正 Validation Layers 结构

## [2.0.0] - 2026-04-27

### Added — v2.0 Core: Solving Three Hard Problems

**Problem 1: Timing Sync (时序不同步)**
- `references/custom-tools-wait.md` — Async wait tools C# code
  - `wait-until-condition`: Poll C# condition until true or timeout
  - `wait-for-animation-state`: Wait for Animator state
  - `wait-for-frame-count`: Wait N frames (UI layout rebuild)
  - `wait-for-stable`: Wait for condition + no new errors
- Core Contract rule: "Wait for state to settle — 禁止写死 Thread.Sleep"
- Layer 4B-Post sub-layer for wait-after-interaction

**Problem 2: Test State Contamination (测试状态污染)**
- `references/state-reset.md` — State sandbox reset strategy & C# code
  - `state-reset` MCP tool with auto/reload/custom strategies
  - `TestHelper.cs` project-level reset helper template
  - Core Contract rule: "Reset between tests — 多个 PlayMode 用例间必须重置"
- Layer 4A-Pre sub-layer for state reset before PlayMode
- `validation-config.example.yaml`: state_reset section

**Problem 3: Screenshot Assertion Blind Spots (截图断言盲区)**
- `references/ui-snapshot-tool.md` — UI DOM tree snapshot tools C# code
  - `ui-hierarchy-snapshot`: Capture Canvas UI tree structure
  - `ui-element-find`: Find specific UI element by name
  - `ui-element-at-position`: Find UI element at screen position
- Core Contract rule: "Assert with data, not visuals — UI 验证优先 ui-hierarchy-snapshot"
- Comparison table: screenshot vs UI snapshot vs reflection probe

**Bonus: Test Backdoor API**
- `references/test-backdoors.md` — Architecture spec for boundary testing
  - Call project helper methods via `reflection-method-call`
  - Template helpers: BoardTestHelper, EconomyTestHelper, UITestHelper
  - `validation-config.example.yaml`: test_backdoors section

**Bonus: JSON Structured Output**
- `references/json-output-schema.md` — Machine-consumable JSON schema
  - Schema version 2.0 with state_snapshots, ui_snapshots, wait_results
  - Dual-track output: Markdown (human) + JSON (machine)
  - Core Contract rule: "Output both Markdown and JSON reports"

### Changed — All Documentation Updated

- **SKILL.md** — v2.0.0, 10 core contract rules, 5 sub-layers, new tool sections
- **references/route-matrix.md** — 4A-Pre/4B-Post in routing table, new tools in guide
- **references/output-contract.md** — JSON report section, wait/UI evidence fields, sub-layer reporting
- **EXAMPLES.md** — All 7 cases upgraded to v2.0 (state-reset + wait + ui-snapshot)
  - Case A: Match3 sliding (full v2.0 flow)
  - Case B: UI button click (wait + ui-snapshot)
  - Case C: Keyboard shortcut (wait for state)
  - Case D: Complex recording/replay (wait-for-stable)
  - Case E: Pre-acceptance sweep (full v2.0 flow)
  - Case F: State reset isolation (NEW)
  - Case G: UI data assertion vs screenshot (NEW)
- **SETUP.md** — v2.0 tool install steps (wait/ui-snapshot/state-reset/backdoor), config reference
- **validation-config.example.yaml** — v2.0 sections: state_reset, test_backdoors, wait_defaults, output
- **README.md / README_CN.md** — v2.0 feature overview, new tool tables, documentation index
- **agents/openai.yaml** — v2.0.0, new capabilities

## [1.1.0] - 2026-04-26

### Added
- Version number in SKILL.md frontmatter (`version: 1.1.0`)
- `README.md` - GitHub entry documentation
- `README_CN.md` - Chinese version
- `CHANGELOG.md` - Version history
- `LICENSE` - MIT license
- `SETUP.md` - Complete setup guide for users and agents
- `EXAMPLES.md` - Complete validation examples including input simulation
- `validation-config.example.yaml` - Configuration template
- `references/troubleshooting.md` - Troubleshooting guide
- Preflight checks section in SKILL.md
- Installation section with detailed steps
- Blockers section in output-contract.md
- MCP server configuration instructions
- **Layer 4 split into 4A/4B/4C sub-processes (input simulation as core)**
- **Input simulation integrated into routing table**
- **Core Contract updated: "Simulate human testing" added**

### Changed
- Improved `agents/openai.yaml` with version info and capabilities
- Added Mermaid decision flowchart to route-matrix.md
- Fixed namespace typo in custom-tools-input.md
- Quick Start now references SETUP.md
- **Routing table now explicitly requires Layer 4B for input/interaction validation**

## [1.0.0] - 2025-12-01

### Added
- Initial release
- Core validation routing logic (same as unity-validation-router)
- Unity-MCP tool reference (77 tools documented)
- Custom input simulation tool templates
- Runtime probes for PlayMode debugging
- Output contract for validation reports
