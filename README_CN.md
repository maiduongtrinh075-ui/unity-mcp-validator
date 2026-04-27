# Unity MCP Validator（中文说明）

基于 Unity-MCP 的 Claude Code 验证路由技能，用于 Unity 游戏开发的自动化变更验证。

## 功能概述

替代随意的"跑几个测试"行为，提供系统化验证流程：

1. **分类** 变更类型（模型、控制器、输入、动画、场景、UI 等）
2. **路由** 到合适的验证层级（静态审查 → EditMode → Editor → PlayMode）
3. **执行** 使用 100+ Unity-MCP 工具完成验证
4. **探针** 通过反射和动态代码执行诊断运行时问题
5. **报告** 明确的结论和阻塞项（Markdown + JSON 双轨输出）

## v2.0 核心增强

| 功能 | 解决的问题 | 方案 |
|------|-----------|------|
| **异步等待工具** | 时序不同步 — Thread.Sleep 不稳定 | `wait-until-condition`, `wait-for-animation-state` 等 |
| **UI DOM 快照** | 截图断言盲区 — 大模型判断截图不可靠 | `ui-hierarchy-snapshot` — 数据级 UI 断言 |
| **状态重置** | 测试间状态污染 | `state-reset` — 测试用例间重置干净状态 |
| **测试后门 API** | 难以构造边界场景 | `reflection-method-call` 调用 TestHelper 方法 |
| **JSON 输出** | 没有机器可消费的报告 | 结构化 JSON + Markdown 双轨报告 |

## Unity-MCP 独有优势

| 特性 | 说明 |
|------|------|
| **反射系统** | `reflection-method-find/call` — 查找和调用任意 C# 方法（含私有方法） |
| **动态执行** | `script-execute` — 编译并运行 C# 代码片段（Roslyn） |
| **运行时嵌入** | 可在编译后的游戏内使用，不仅限于 Editor |
| **100+ 工具** | 资源、场景、GameObject、组件、材质、预制体、包、测试、截图、日志 |
| **自定义工具** | 单行代码即可添加新工具 |

## 快速开始

### 1. 安装为 Claude Code Skill

```bash
# 克隆仓库
git clone https://github.com/maiduongtrinh075-ui/unity-mcp-validator.git

# 复制（或软链接）到 Claude Code skills 目录
cp -r unity-mcp-validator ~/.claude/skills/unity-mcp-validator
# 项目级安装：cp -r unity-mcp-validator .claude/skills/unity-mcp-validator
```

克隆后，Claude Code 会在触发此 Skill 时自动加载 `SKILL.md`。

### 2. 同步更新

```bash
# 拉取最新版本
cd ~/.claude/skills/unity-mcp-validator && git pull origin master
```

### 3. 完整安装

**完整安装说明见 [SETUP.md](SETUP.md)（Unity-MCP 插件、自定义工具、配置文件）**

### 快速安装清单

| 步骤 | 命令/操作 |
|------|-----------|
| 1. 安装 Unity-MCP 插件 | `npm install -g unity-mcp-cli && unity-mcp-cli install-plugin ./你的项目` |
| 2. 添加自定义输入工具 | 复制 `references/custom-tools-input.md` 中的代码 |
| 3. **添加等待工具 (v2.0)** | 复制 `references/custom-tools-wait.md` 中的代码 |
| 4. **添加 UI 快照工具 (v2.0)** | 复制 `references/ui-snapshot-tool.md` 中的代码 |
| 5. **添加状态重置工具 (v2.0)** | 复制 `references/state-reset.md` 中的代码 |
| 6. 配置 MCP 服务器 | 添加到 Claude Code MCP 配置（见 SETUP.md） |
| 7. 测试连接 | 调用 `editor-application-get-state` |
| 8. 创建配置文件（可选） | 复制 `validation-config.example.yaml` |

## 验证层级

### Layer 1: 静态审查
无需 MCP 工具 — 直接读取 diff 和代码。

### Layer 2: EditMode 测试
| 工具 | 用途 |
|------|------|
| `tests-run mode=EditMode` | 运行 EditMode 单元测试 |

### Layer 3: Editor 检查
| 工具 | 用途 |
|------|------|
| `script-execute` | 编译并运行代码片段 |
| `scene-get-data` | 获取场景层级 |
| `gameobject-find` | 查找 GameObject |
| `gameobject-component-get` | 获取组件详情 |
| `assets-get-data` | 获取资源/预制体数据 |
| `console-get-logs` | 获取 Unity Console 日志 |
| **`ui-hierarchy-snapshot`** | **UI DOM 树结构快照** |

### Layer 4: PlayMode 自动化（v2.0 增强）

Layer 4 现在包含 5 个子层级：

| 子层级 | 用途 | 关键工具 |
|--------|------|----------|
| **4A-Pre** | 状态重置 | `state-reset`, `reflection-method-call` |
| **4A** | 进入/退出 PlayMode | `editor-application-set-state` |
| **4B** | 输入模拟 | `simulate-click-ui`, `simulate-drag-world`, `simulate-key-press` |
| **4B-Post** | 等待状态稳定 | `wait-until-condition`, `wait-for-animation-state` |
| **4C** | 证据收集 | `screenshot-game-view`, `ui-hierarchy-snapshot`, `reflection-method-call` |

**完整验证流程（三消滑动示例）：**
```
[4A-Pre] state-reset → [4A] 进入 PlayMode → [4A] wait-until-condition →
[4C] 基线截图 + UI 快照 →
[4B] simulate-drag-world → [4B-Post] wait-until-condition →
[4C] 结果截图 + UI 快照 + 反射探针 →
[4A] 退出 PlayMode
```

**详细案例见 [EXAMPLES.md](EXAMPLES.md)**

## v2.0 新增工具

### 异步等待工具
| 工具 | 用途 |
|------|------|
| `wait-until-condition` | 轮询 C# 条件直到为 true 或超时 |
| `wait-for-animation-state` | 等待 Animator 进入指定状态 |
| `wait-for-frame-count` | 等待 N 帧（UI 布局重建） |
| `wait-for-stable` | 等待条件满足 + 无新错误 |

### UI DOM 快照工具
| 工具 | 用途 |
|------|------|
| `ui-hierarchy-snapshot` | 抓取 Canvas UI 树结构快照 |
| `ui-element-find` | 按名称查找 UI 元素状态 |
| `ui-element-at-position` | 查找屏幕坐标下的 UI 元素 |

### 状态重置工具
| 工具 | 用途 |
|------|------|
| `state-reset` | 重置游戏到干净状态 |

### 测试后门 API
不是独立 MCP 工具，而是项目中的 C# 方法，通过 `reflection-method-call` 调用：
```bash
reflection-method-call typeName="TestHelper" methodName="SkipTutorial"
reflection-method-call typeName="BoardTestHelper" methodName="SetupChainExplosion"
```

## 配置

复制 `validation-config.example.yaml` 到项目根目录。v2.0 新增配置：

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

output:
  dual_format: true
  json_schema_version: "2.0"
```

## 输出格式（v2.0 双轨输出）

每次验证产生两份报告：

| 格式 | 消费者 | 用途 |
|------|--------|------|
| Markdown | 人类开发者 | 快速审查、调试 |
| JSON | CI/CD、Codex | 自动化通过/失败判断、追踪 |

Markdown 格式见 [references/output-contract.md](references/output-contract.md)
JSON 格式见 [references/json-output-schema.md](references/json-output-schema.md)

## 文档索引

| 文件 | 说明 |
|------|------|
| [SKILL.md](SKILL.md) | Skill 定义和核心契约 |
| [SETUP.md](SETUP.md) | 完整安装指南 |
| [EXAMPLES.md](EXAMPLES.md) | 验证案例（v2.0 增强） |
| [CHANGELOG.md](CHANGELOG.md) | 版本历史 |
| [validation-config.example.yaml](validation-config.example.yaml) | 配置模板（v2.0） |
| [references/route-matrix.md](references/route-matrix.md) | 变更分类 → 验证路由 |
| [references/output-contract.md](references/output-contract.md) | Markdown 报告格式 |
| [references/json-output-schema.md](references/json-output-schema.md) | JSON 报告格式（v2.0） |
| [references/runtime-probes.md](references/runtime-probes.md) | 运行时探针阶梯 |
| [references/tool-reference.md](references/tool-reference.md) | Unity-MCP 工具参考 |
| [references/troubleshooting.md](references/troubleshooting.md) | 故障排除指南 |
| [references/custom-tools-input.md](references/custom-tools-input.md) | 输入模拟工具代码 |
| [references/custom-tools-wait.md](references/custom-tools-wait.md) | 等待工具代码（v2.0） |
| [references/ui-snapshot-tool.md](references/ui-snapshot-tool.md) | UI 快照工具代码（v2.0） |
| [references/state-reset.md](references/state-reset.md) | 状态重置工具代码（v2.0） |
| [references/test-backdoors.md](references/test-backdoors.md) | 测试后门 API 指南（v2.0） |

## 故障排除

安装问题见 [SETUP.md](SETUP.md)
运行问题见 [references/troubleshooting.md](references/troubleshooting.md)

## 版本

当前版本：**2.0.0**

版本历史见 [CHANGELOG.md](CHANGELOG.md)

## 许可证

MIT License - 见 LICENSE 文件。

## 致谢

基于 [Unity-MCP](https://github.com/IvanMurzak/Unity-MCP) by IvanMurzak。
