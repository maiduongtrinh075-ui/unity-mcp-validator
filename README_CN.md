# Unity MCP Validator (中文说明)

基于 Unity-MCP 的 Claude Code 验证路由技能，用于 Unity 游戏开发的变更验证。

## 功能概述

替代随意的"跑几个测试"行为，提供系统化验证流程：

1. **分类** 变更类型（模型、控制器、输入、动画、场景、UI 等）
2. **路由** 到合适的验证层级（静态审查 → EditMode → Editor → PlayMode）
3. **执行** 使用 100+ Unity-MCP 工具完成验证
4. **探针** 通过反射和动态代码执行诊断运行时问题
5. **报告** 明确的结论和阻塞项

## Unity-MCP 独有优势

| 特性 | 说明 |
|------|------|
| **反射系统** | `reflection-method-find/call` — 查找和调用任意 C# 方法（含私有方法） |
| **动态执行** | `script-execute` — 编译并运行 C# 代码片段（Roslyn） |
| **运行时嵌入** | 可在编译后的游戏内使用，不仅限于 Editor |
| **100+ 工具** | 资源、场景、GameObject、组件、材质、预制体、包、测试、截图、日志 |
| **自定义工具** | 单行代码即可添加新工具 |

## 安装

**完整安装说明见 [SETUP.md](SETUP.md)**

### 快速安装清单

| 步骤 | 命令/操作 |
|------|-----------|
| 1. 安装 Unity-MCP 插件 | `npm install -g unity-mcp-cli && unity-mcp-cli install-plugin ./你的项目` |
| 2. 添加自定义输入工具 | 复制 `references/custom-tools-input.md` 中的代码 |
| 3. 配置 MCP 服务器 | 添加到 Claude Code MCP 配置（见 SETUP.md） |
| 4. 测试连接 | 调用 `editor-application-get-state` |
| 5. 创建配置文件（可选） | 复制 `validation-config.example.yaml` |

## 工具参考

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

### Layer 4: PlayMode 自动化
| 工具 | 用途 |
|------|------|
| `editor-application-set-state` | 进入/退出 PlayMode |
| `screenshot-game-view` | 捕获游戏截图 |
| `reflection-method-call` | 运行时调用任意 C# 方法 |
| `reflection-method-find` | 查找方法 |

## 自定义工具（输入模拟）

Unity-MCP 没有内置输入模拟，需要添加自定义工具：

| 工具 | 用途 |
|------|------|
| `simulate-click-world` | 点击世界空间对象（碰撞体） |
| `simulate-click-ui` | 点击 UI 元素（EventSystem） |
| `simulate-key-press` | 按下键盘按键 |
| `record-start/stop` | 录制输入序列 |
| `replay-input` | 回放录制的输入 |

实现代码见 `references/custom-tools-input.md`。

## 配置

复制 `validation-config.example.yaml` 到项目根目录：

```yaml
invariants:
  - name: "input_lock_during_animation"
    description: "关键动画期间输入必须锁定"

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

## 使用方法

在 Claude Code 中调用：

```
使用 $unity-mcp-validator 验证控制器变更
```

预验收检查：

```
使用 $unity-mcp-validator 运行完整的预验收验证
```

## 输出格式

```text
Route: 输入/交互 + PlayMode 运行时回归
Layers run:
- 静态审查: 已运行
- EditMode: 被路由跳过
- Unity Editor 检查: 已运行
- PlayMode 自动化: 已运行

Unity-MCP tools used:
- script-execute: 编译成功
- scene-get-data: 层级有效
- reflection-method-call: controller.CurrentPhase = 'Processing'

Evidence: [截图、日志、状态快照]

Verdict: 现在可接受 / 有缺口条件性可接受 / 尚不可接受
Next action: [最有用的下一步命令]
```

## 故障排除

安装问题见 [SETUP.md](SETUP.md)
运行问题见 [references/troubleshooting.md](references/troubleshooting.md)

## 版本

当前版本：**1.1.0**

版本历史见 [CHANGELOG.md](CHANGELOG.md)

## 许可证

MIT License - 见 LICENSE 文件。

## 致谢

基于 [Unity-MCP](https://github.com/IvanMurzak/Unity-MCP) by IvanMurzak。