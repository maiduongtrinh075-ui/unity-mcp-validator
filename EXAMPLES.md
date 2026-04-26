# Validation Examples
<!-- 验证案例：完整的验证流程演示 -->

本文档提供完整的验证案例，展示如何使用 Unity-MCP Validator 进行系统化验证。

---

## 案例 A：验证三消游戏滑动消除

### 场景描述

变更内容：修改了 Board.cs 中的滑动消除逻辑，优化了匹配算法。

验证目标：确认滑动操作能正确触发消除，分数正确增加，状态正确转换。

### Step 1: 收集变更范围

```
变更文件：
- Scripts/Board.cs（滑动检测算法）
- Scripts/Gem.cs（消除动画）

涉及子系统：
- 消除逻辑
- 分数计算
- 动画流程
```

### Step 2: 分类变更类型

**分类结果：Input / interaction（输入/交互）**

理由：
- 涉及滑动输入处理
- 需要验证交互流程完整性
- 算法变更影响交互结果

**路由决策：Static + Editor + PlayMode (4A+4B+4C)**

### Step 3: 执行验证层

#### Layer 1: Static Review

检查代码变更：

```
✓ 滑动检测逻辑修改正确
✓ 匹配算法边界条件处理
⚠️ 注意：分数增加时机在动画完成后
```

#### Layer 2: EditMode

```bash
tests-run mode=EditMode filter-type=regex filter-value=".*Board.*"
```

结果：
```
✓ BoardMatchTests: 5 tests passed
✓ GemEliminateTests: 3 tests passed
```

#### Layer 3: Editor Inspection

```bash
# 检查场景配置
scene-get-data → 层级结构正确
gameobject-find name="Board" → Board 对象存在
gameobject-component-get gameObjectId="Board" component="Board" → 引用配置正确
```

#### Layer 4: PlayMode Automation

**完整交互验证流程：**

```bash
# 4A: 进入 PlayMode
editor-application-set-state playMode=true

# 4C: 基线截图
screenshot-game-view → 保存基线状态

# 4B: 模拟滑动（核心）
simulate-drag-world startX=200 startY=300 endX=400 endY=300 duration=0.3
# 说明：从格子A拖到格子B，模拟三消滑动

# 4C: 检查消除效果
screenshot-game-view → 对比基线，检查是否有消除

# 4C: 检查分数和状态
reflection-method-call → 调用 Board.GetScore() 和 Board.GetGemCount()
# 期望：分数增加，Gem 数量减少

# 4C: 检查日志
console-get-logs → 检查是否有错误

# 4A: 退出 PlayMode
editor-application-set-state playMode=false
```

**验证结果：**
```
✓ 滑动操作触发成功
✓ 消除动画播放正确
✓ 分数从 0 增加到 30（正确）
✓ Gem 数量从 64 减少到 60（正确）
✓ 无错误日志
```

### Step 4: 报告

```text
Route: Input / interaction + PlayMode 交互验证
Why: 变更涉及滑动消除算法，必须通过模拟交互验证流程正确性

Layers run:
- Static review: ran
- EditMode: ran (8 tests passed)
- Editor inspection: ran
- PlayMode automation: ran (4A+4B+4C)

Unity-MCP tools used:
- script-execute: compiled successfully
- tests-run: 8 tests passed
- scene-get-data: hierarchy valid
- editor-application-set-state: entered/exited PlayMode
- simulate-drag-world: sliding simulation completed
- screenshot-game-view: before/after captured
- reflection-method-call: Board.GetScore() = 30, Board.GetGemCount() = 60
- console-get-logs: no errors

Evidence:
- 基线截图：消除前状态
- 结果截图：消除后状态，3个 Gem 消除
- 分数变化：0 → 30
- Gem 数量：64 → 60

Findings:
- 滑动消除流程完整正确
- 算法变更后匹配逻辑工作正常
- 分数计算正确

Unity-MCP status:
- Completed: full PlayMode validation with input simulation
- Connection: connected

Verdict: Acceptable now.
Reason: 滑动交互验证通过，消除流程完整，分数和状态正确。

Next action: None required.
```

---

## 案例 B：验证 UI 按钮点击流程

### 场景描述

变更内容：添加了新的 Settings 按钮，点击后应打开设置面板。

验证目标：确认按钮点击能打开面板，面板内容正确显示。

### Step 2: 分类

**分类结果：UI / HUD**

**路由决策：Static + Editor + PlayMode (4A+4B+4C)**

### Step 3: 执行验证

#### Layer 1-2: Static + EditMode

```
✓ 按钮脚本逻辑正确
✓ UI 组件引用配置正确
```

#### Layer 3: Editor Inspection

```bash
gameobject-find name="SettingsButton" → 按钮存在
gameobject-find name="SettingsPanel" → 面板存在
gameobject-component-get gameObjectId="SettingsButton" component="Button" → onClick 引用正确
```

#### Layer 4: PlayMode

```bash
# 4A: 进入 PlayMode
editor-application-set-state playMode=true

# 4C: 基线截图（面板应该隐藏）
screenshot-game-view → 面板未显示

# 4B: 点击按钮
simulate-click-ui x=150 y=100
# 或使用按钮名称
click-button-by-name buttonName="SettingsButton"

# 4C: 检查面板是否显示
screenshot-game-view → 面板已显示

# 4C: 检查面板状态
reflection-method-call → SettingsPanel.IsActive() = true

# 4A: 退出
editor-application-set-state playMode=false
```

---

## 案例 C：验证键盘快捷键

### 场景描述

变更内容：添加了 ESC 键关闭设置面板的功能。

### Step 4: PlayMode 验证

```bash
# 4A: 进入 PlayMode
editor-application-set-state playMode=true

# 4B: 先打开面板（点击按钮）
simulate-click-ui x=150 y=100

# 4C: 确认面板打开
screenshot-game-view → 面板显示

# 4B: 按 ESC 键
simulate-key-press key="Escape"

# 4C: 检查面板是否关闭
screenshot-game-view → 面板隐藏
reflection-method-call → SettingsPanel.IsActive() = false

# 4A: 退出
editor-application-set-state playMode=false
```

---

## 案例 D：验证复杂交互序列（录制回放）

### 场景描述

问题：用户报告"玩了 5 次后游戏卡住"

验证目标：复现问题，定位卡住原因。

### Step 4: PlayMode + 录制回放

```bash
# 4A: 进入 PlayMode
editor-application-set-state playMode=true

# 4B: 开始录制
record-start

# 手动操作或添加事件
record-add-click x=100 y=200 isDown=true
record-add-click x=100 y=200 isDown=false
# ... 继续添加 5 次操作 ...

# 4B: 停止录制
record-stop

# 4B: 回放
replay-input speed=1.0

# 4C: 检查状态
screenshot-game-view
console-get-logs
reflection-method-call → InputManager.IsLocked = true (卡住！)

# 定位问题：输入锁未释放
```

**发现：** 第 5 次操作后 InputLock 未释放，导致后续无法交互。

---

## 案例 E：预验收检查（全流程）

### 场景描述

合并前完整验证。

### 完整流程

```bash
# Layer 1: Static Review
- 代码审查完成

# Layer 2: EditMode
tests-run mode=EditMode → 全部通过

# Layer 3: Editor
scene-get-data → 场景配置正确
assets-get-data assetPath="Assets/Prefabs/*.prefab" → 预制体引用正确
console-get-logs → 无编译错误

# Layer 4: PlayMode 冒烟测试
editor-application-set-state playMode=true

# 基本交互验证
simulate-click-ui x=100 y=100 → 主菜单按钮
simulate-click-ui x=200 y=150 → 开始游戏
simulate-drag-world startX=100 startY=200 endX=300 endY=200 → 滑动消除
simulate-key-press key="Escape" → 返回菜单

# 状态检查
reflection-method-call → 游戏状态正常

editor-application-set-state playMode=false
```

---

## 输入模拟工具速查表

| 交互类型 | 工具 | 参数示例 |
|----------|------|----------|
| 点击 UI 按钮 | `simulate-click-ui` | `x=150 y=100` |
| 点击 UI 按钮（按名称） | `click-button-by-name` | `buttonName="SettingsButton"` |
| 点击游戏对象 | `simulate-click-world` | `x=400 y=300` |
| 滑动（三消） | `simulate-drag-world` | `startX=200 startY=300 endX=400 endY=300` |
| 按键 | `simulate-key-press` | `key="Escape"` |
| 按键按下（持续） | `simulate-key-down` | `key="Space"` |
| 按键释放 | `simulate-key-up` | `key="Space"` |
| 录制输入 | `record-start` / `record-stop` | - |
| 回放输入 | `replay-input` | `speed=1.0` |

---

## 常见验证流程模板

### 模板 1：按钮点击验证

```
1. editor-application-set-state playMode=true
2. screenshot-game-view（基线）
3. simulate-click-ui x=? y=?
4. screenshot-game-view（结果）
5. reflection-method-call（检查状态）
6. editor-application-set-state playMode=false
```

### 模板 2：滑动验证（三消）

```
1. editor-application-set-state playMode=true
2. screenshot-game-view（基线）
3. simulate-drag-world startX=? startY=? endX=? endY=?
4. screenshot-game-view（结果）
5. reflection-method-call（检查分数/数量）
6. console-get-logs（检查错误）
7. editor-application-set-state playMode=false
```

### 模板 3：键盘输入验证

```
1. editor-application-set-state playMode=true
2. simulate-key-press key=?
3. screenshot-game-view
4. reflection-method-call（检查响应状态）
5. editor-application-set-state playMode=false
```

### 模板 4：复杂序列录制回放

```
1. editor-application-set-state playMode=true
2. record-start
3. [手动操作或 record-add-*]
4. record-stop
5. record-export（保存）
6. replay-input speed=1.0
7. screenshot-game-view + reflection-method-call
8. editor-application-set-state playMode=false
```