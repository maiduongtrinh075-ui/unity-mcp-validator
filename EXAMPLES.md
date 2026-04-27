# Validation Examples (v2.0)
<!-- 验证案例：完整的验证流程演示 -->

本文档提供完整的验证案例，展示如何使用 Unity-MCP Validator v2.0 进行系统化验证。

**v2.0 关键增强：**
- 所有 PlayMode 测试前必须执行**状态重置**（4A-Pre）
- 交互操作后必须**等待状态稳定**（4B-Post），禁止 Thread.Sleep
- UI 验证必须配合 **ui-hierarchy-snapshot** 做数据断言
- 每次验证输出 **Markdown + JSON** 双轨报告

---

## 案例 A：验证三消游戏滑动消除（v2.0 完整版）

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

**路由决策：Static + Editor + PlayMode (4A-Pre+4A+4B+4B-Post+4C)**

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
ui-hierarchy-snapshot format="summary" → UI 层级预览
```

#### Layer 4: PlayMode Automation（v2.0 增强版）

**完整交互验证流程：**

```bash
# ── 4A-Pre: 状态重置 ──
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3

# ── 4A: 进入 PlayMode ──
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# ── 4C: 基线证据 ──
screenshot-game-view → 保存基线状态
ui-hierarchy-snapshot format="json" → 基线 UI 快照
reflection-method-call → Board.GetScore() = 0, Board.GemCount() = 64

# ── 4B: 模拟滑动（核心） ──
simulate-drag-world startX=200 startY=300 endX=400 endY=300 duration=0.3

# ── 4B-Post: 等待消除动画完成 ──
wait-until-condition condition="Board.Instance.IsAnimating == false" timeoutSeconds=5

# ── 4C: 结果证据 ──
screenshot-game-view → 消除后截图
ui-hierarchy-snapshot format="json" → 结果 UI 快照（断言 ScoreText 文本变化）
reflection-method-call → Board.GetScore(), Board.GemCount(), Board.CurrentPhase
console-get-logs → 检查是否有错误

# ── 4A: 退出 PlayMode ──
editor-application-set-state playMode=false
```

**验证结果：**
```
✓ 状态重置成功，干净起点
✓ 滑动操作触发成功
✓ 等待消除动画完成（0.8s，未超时）
✓ 消除动画播放正确
✓ 分数从 0 增加到 30（正确）
✓ Gem 数量从 64 减少到 60（正确）
✓ UI 快照：ScoreText 文本从 "0" 变为 "30"（正确）
✓ 无错误日志
```

### Step 4: 报告（双轨输出）

**Markdown 报告：**

```text
Route: Input / interaction + PlayMode 交互验证
Why: 变更涉及滑动消除算法，必须通过模拟交互验证流程正确性

Layers run:
- Static review: ran
- EditMode: ran (8 tests passed)
- Editor inspection: ran (含 UI 快照预览)
- PlayMode automation: ran (4A-Pre+4A+4B+4B-Post+4C)

PlayMode sub-layers:
- 4A-Pre (state reset): ran — 游戏状态重置成功
- 4A (enter/exit): ran
- 4B (input sim): ran — simulate-drag-world 成功
- 4B-Post (wait): ran — wait-until-condition 0.8s 消除完成
- 4C (evidence): ran — 截图 + UI 快照 + 反射探针

Unity-MCP tools used:
- script-execute: compiled successfully
- tests-run: 8 tests passed
- scene-get-data: hierarchy valid
- state-reset: game state reset successfully
- editor-application-set-state: entered/exited PlayMode
- wait-until-condition: initialization met after 0.2s
- simulate-drag-world: sliding simulation completed
- wait-until-condition: IsAnimating==false met after 0.8s
- screenshot-game-view: before/after captured
- ui-hierarchy-snapshot: ScoreText "0" → "30" (correct)
- reflection-method-call: Score=30, GemCount=60, Phase=Idle
- console-get-logs: no errors

Evidence:
- 基线截图：消除前状态
- 结果截图：消除后状态，3个 Gem 消除
- UI 快照：ScoreText 文本值精确匹配
- 分数变化：0 → 30
- Gem 数量：64 → 60
- 等待结果：消除动画 0.8s 完成，无超时

Findings:
- 滑动消除流程完整正确
- 算法变更后匹配逻辑工作正常
- 分数计算正确
- UI 数据断言通过

Unity-MCP status:
- Completed: full PlayMode validation with state reset, input sim, wait, UI snapshot
- Connection: connected
- Custom tools: wait=yes, ui-snapshot=yes, state-reset=yes

Verdict: Acceptable now.
Reason: 滑动交互验证通过，消除流程完整，分数和状态正确，UI 数据断言通过。

Next action: None required.
```

**JSON 报告：** 参见 [json-output-schema.md](references/json-output-schema.md) 生成 `validation-report.json`

---

## 案例 B：验证 UI 按钮点击流程（v2.0）

### 场景描述

变更内容：添加了新的 Settings 按钮，点击后应打开设置面板。

验证目标：确认按钮点击能打开面板，面板内容正确显示。

### Step 2: 分类

**分类结果：UI / HUD**

**路由决策：Static + Editor + PlayMode (4A-Pre+4A+4B+4B-Post+4C)**

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
ui-hierarchy-snapshot format="summary" → 预览 UI 层级结构
```

#### Layer 4: PlayMode（v2.0）

```bash
# 4A-Pre: 状态重置
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3

# 4A: 进入 PlayMode
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# 4C: 基线（面板应该隐藏）
screenshot-game-view → 面板未显示
ui-hierarchy-snapshot format="json" → SettingsPanel active=false
reflection-method-call → SettingsPanel.IsActive() = false

# 4B: 点击按钮
simulate-click-ui x=150 y=100

# 4B-Post: 等待 UI 布局更新
wait-for-frame-count frameCount=2

# 4C: 检查面板是否显示
screenshot-game-view → 面板已显示
ui-hierarchy-snapshot format="json" → SettingsPanel active=true, 子元素可见
ui-element-find name="SettingsPanel" → type=Panel, active=true
reflection-method-call → SettingsPanel.IsActive() = true

# 4A: 退出
editor-application-set-state playMode=false
```

**v2.0 vs v1.0 对比：**
| 操作 | v1.0 | v2.0 |
|------|------|------|
| 状态重置 | ❌ 无 | ✅ state-reset |
| 初始状态确认 | ❌ 无 | ✅ wait-until-condition |
| 点击后等待 | ❌ 无/Thread.Sleep | ✅ wait-for-frame-count |
| 面板状态断言 | ❌ 只靠截图 | ✅ ui-hierarchy-snapshot + reflection |
| 结果可靠性 | 中 | 高 |

---

## 案例 C：验证键盘快捷键（v2.0）

### 场景描述

变更内容：添加了 ESC 键关闭设置面板的功能。

### Step 4: PlayMode 验证（v2.0）

```bash
# 4A-Pre: 状态重置
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3

# 4A: 进入 PlayMode
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# 4B: 先打开面板（点击按钮）
simulate-click-ui x=150 y=100

# 4B-Post: 等待面板打开
wait-for-frame-count frameCount=2

# 4C: 确认面板打开
screenshot-game-view → 面板显示
ui-hierarchy-snapshot → SettingsPanel active=true

# 4B: 按 ESC 键
simulate-key-press key="Escape"

# 4B-Post: 等待面板关闭动画
wait-until-condition condition="SettingsPanel.IsActive() == false" timeoutSeconds=2

# 4C: 检查面板是否关闭
screenshot-game-view → 面板隐藏
ui-hierarchy-snapshot → SettingsPanel active=false
reflection-method-call → SettingsPanel.IsActive() = false

# 4A: 退出
editor-application-set-state playMode=false
```

---

## 案例 D：验证复杂交互序列 — 录制回放（v2.0）

### 场景描述

问题：用户报告"玩了 5 次后游戏卡住"

验证目标：复现问题，定位卡住原因。

### Step 4: PlayMode + 录制回放（v2.0）

```bash
# 4A-Pre: 状态重置
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3

# 4A: 进入 PlayMode
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# 4C: 基线证据
screenshot-game-view → 初始状态
ui-hierarchy-snapshot → 初始 UI 状态

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

# 4B-Post: 等待状态稳定
wait-for-stable condition="InputManager.IsLocked == false" timeoutSeconds=5 stableForFrames=10

# 4C: 检查状态（bug 复现时）
screenshot-game-view → 游戏界面卡住
ui-hierarchy-snapshot → UI 状态快照
console-get-logs → 检查错误日志
reflection-method-call → InputManager.IsLocked = true (卡住！)

# 深入探针：为什么输入锁没释放？
reflection-method-call → Controller.CurrentPhase = "Processing"
reflection-method-call → AnimationManager.IsPlaying = false
# 发现：动画已结束但回调未触发，状态未转换

# 4A: 退出
editor-application-set-state playMode=false
```

**发现：** 第 5 次操作后 InputLock 未释放，原因是动画回调丢失导致状态机卡在 Processing。

---

## 案例 E：预验收检查 — 全流程（v2.0）

### 场景描述

合并前完整验证。

### 完整流程

```bash
# ═══ Layer 1: Static Review ═══
- 代码审查完成

# ═══ Layer 2: EditMode ═══
tests-run mode=EditMode → 全部通过

# ═══ Layer 3: Editor ═══
scene-get-data → 场景配置正确
assets-get-data assetPath="Assets/Prefabs/*.prefab" → 预制体引用正确
ui-hierarchy-snapshot format="summary" → UI 层级完整
console-get-logs → 无编译错误

# ═══ Layer 4: PlayMode 冒烟测试 ═══

# 4A-Pre: 状态重置
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3

# 4A: 进入 PlayMode
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# 基线
screenshot-game-view → 主菜单截图
ui-hierarchy-snapshot → 主菜单 UI 状态

# 冒烟测试 1：主菜单按钮
simulate-click-ui x=100 y=100 → 主菜单按钮
wait-for-frame-count frameCount=2
screenshot-game-view → 按钮响应
ui-hierarchy-snapshot → UI 状态变化

# 冒烟测试 2：开始游戏
simulate-click-ui x=200 y=150 → 开始游戏
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Playing'" timeoutSeconds=3
screenshot-game-view → 游戏画面
reflection-method-call → 游戏状态正常

# 冒烟测试 3：滑动消除
simulate-drag-world startX=100 startY=200 endX=300 endY=200
wait-until-condition condition="Board.Instance.IsAnimating == false" timeoutSeconds=5
screenshot-game-view → 消除后画面
ui-hierarchy-snapshot → 分数/状态断言
reflection-method-call → 分数更新正确

# 冒烟测试 4：返回菜单
simulate-key-press key="Escape"
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Menu'" timeoutSeconds=2
screenshot-game-view → 菜单画面

# 状态检查
console-get-logs → 无错误

# 4A: 退出 PlayMode
editor-application-set-state playMode=false
```

**预验收输出：**
- `validation-report.md` — 人类阅读
- `validation-report.json` — 机器消费（CI/CD 集成）

---

## 案例 F：状态重置隔离验证（v2.0 新增）

### 场景描述

验证多个 PlayMode 测试用例之间的状态隔离。

### 问题

没有状态重置时，前一个测试的分数/状态会污染后一个测试。

### v2.0 方案

```bash
# ── 测试用例 1：正常消除 ──
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# 操作 + 验证
reflection-method-call → Score = 0 (确认重置成功)
simulate-drag-world startX=100 startY=200 endX=300 endY=200
wait-until-condition condition="Board.Instance.IsAnimating == false" timeoutSeconds=5
reflection-method-call → Score = 30 (验证通过)

editor-application-set-state playMode=false

# ── 测试用例 2：连锁消除 ──
# 如果不重置，Score 会从 30 开始！
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# 使用测试后门构造边界场景
reflection-method-call typeName="BoardTestHelper" methodName="SetupChainExplosion"
wait-until-condition condition="Board.Instance.IsReady == true" timeoutSeconds=2

reflection-method-call → Score = 0 (确认重置成功！)
simulate-drag-world startX=100 startY=200 endX=300 endY=200
wait-until-condition condition="Board.Instance.IsAnimating == false" timeoutSeconds=10
reflection-method-call → Score = 150 (连锁消除验证通过)

editor-application-set-state playMode=false
```

**关键点：** 每个测试用例前都执行 `state-reset`，确保 Score 从 0 开始，测试结果可重复。

---

## 案例 G：UI 数据断言 vs 截图断言（v2.0 新增）

### 场景描述

验证游戏结束后 GameOver 面板显示正确。

### ❌ v1.0 方案（仅截图）

```bash
# 问题：截图可能截到过渡帧、分辨率不同导致误判
screenshot-game-view → 交给大模型判断"是否显示了 Game Over"
# 大模型回答："看起来像是显示了 Game Over" → 不可靠！
```

### ✅ v2.0 方案（截图 + UI 快照 + 反射）

```bash
# 4A-Pre + 4A
state-reset strategy="auto"
wait-until-condition condition="GameController.Instance != null" timeoutSeconds=3
editor-application-set-state playMode=true
wait-until-condition condition="GameController.Instance.CurrentPhase == 'Idle'" timeoutSeconds=5

# 使用后门直接触发 Game Over
reflection-method-call typeName="UITestHelper" methodName="ForceGameOver"
wait-for-frame-count frameCount=2

# 截图（辅助证据）
screenshot-game-view → Game Over 画面

# UI 快照数据断言（核心证据）
ui-hierarchy-snapshot format="json"
ui-element-find name="GameOverPanel" → active=true ✅
ui-element-find name="FinalScoreText" → text="150" ✅
ui-element-find name="RestartButton" → active=true, interactable=true ✅

# 反射验证
reflection-method-call → GameController.Instance.CurrentPhase = "GameOver" ✅

editor-application-set-state playMode=false
```

**v2.0 优势：**
| 断言方式 | 分辨率依赖 | 帧时机 | 大模型判断 | 文本精确度 | 交互状态 |
|----------|-----------|--------|-----------|-----------|---------|
| 截图 | ❌ 是 | ❌ 过渡帧 | ❌ 不可靠 | ❌ OCR 出错 | ❌ 看不出 |
| UI 快照 | ✅ 否 | ✅ 即时 | ✅ 不需要 | ✅ 100% | ✅ 直接断言 |
| 反射探针 | ✅ 否 | ✅ 即时 | ✅ 不需要 | ✅ 100% | ✅ 直接断言 |

---

## 输入模拟工具速查表

### 基础输入工具（v1.0）

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

### v2.0 新增工具

| 类型 | 工具 | 参数示例 | 用途 |
|------|------|----------|------|
| 等待 | `wait-until-condition` | `condition="Board.Instance.IsAnimating == false" timeoutSeconds=5` | 等待状态满足条件 |
| 等待 | `wait-for-animation-state` | `objectName="Player" stateName="Idle" timeoutSeconds=3` | 等待动画完成 |
| 等待 | `wait-for-frame-count` | `frameCount=2` | 等待 N 帧 |
| 等待 | `wait-for-stable` | `condition="..." stableForFrames=10 timeoutSeconds=5` | 等待状态稳定 |
| UI 快照 | `ui-hierarchy-snapshot` | `format="json"` | 抓取 UI DOM 树 |
| UI 查找 | `ui-element-find` | `name="ScoreText"` | 查找 UI 元素状态 |
| UI 坐标 | `ui-element-at-position` | `x=150 y=100` | 查找坐标下的 UI 元素 |
| 状态重置 | `state-reset` | `strategy="auto"` | 重置游戏状态 |
| 后门 API | `reflection-method-call` | `typeName="TestHelper" methodName="SkipTutorial"` | 构造边界场景 |

---

## 常见验证流程模板（v2.0）

### 模板 1：按钮点击验证

```
1. state-reset strategy="auto"
2. wait-until-condition condition="GameController.Instance != null"
3. editor-application-set-state playMode=true
4. wait-until-condition condition="...CurrentPhase == 'Idle'"
5. screenshot-game-view（基线）
6. ui-hierarchy-snapshot（基线）
7. simulate-click-ui x=? y=?
8. wait-for-frame-count frameCount=2
9. screenshot-game-view（结果）
10. ui-hierarchy-snapshot（断言 UI 状态）
11. reflection-method-call（检查状态）
12. editor-application-set-state playMode=false
```

### 模板 2：滑动验证（三消）

```
1. state-reset strategy="auto"
2. wait-until-condition condition="GameController.Instance != null"
3. editor-application-set-state playMode=true
4. wait-until-condition condition="...CurrentPhase == 'Idle'"
5. screenshot-game-view（基线）
6. reflection-method-call → 基线状态（Score=0, GemCount=64）
7. simulate-drag-world startX=? startY=? endX=? endY=?
8. wait-until-condition condition="Board.Instance.IsAnimating == false"
9. screenshot-game-view（结果）
10. ui-hierarchy-snapshot（断言 ScoreText 文本）
11. reflection-method-call（检查分数/数量）
12. console-get-logs（检查错误）
13. editor-application-set-state playMode=false
```

### 模板 3：键盘输入验证

```
1. state-reset strategy="auto"
2. wait-until-condition condition="GameController.Instance != null"
3. editor-application-set-state playMode=true
4. wait-until-condition condition="...CurrentPhase == 'Idle'"
5. simulate-key-press key=?
6. wait-until-condition condition="[预期状态变化]"
7. screenshot-game-view
8. ui-hierarchy-snapshot（断言 UI 响应）
9. reflection-method-call（检查响应状态）
10. editor-application-set-state playMode=false
```

### 模板 4：复杂序列录制回放

```
1. state-reset strategy="auto"
2. wait-until-condition condition="GameController.Instance != null"
3. editor-application-set-state playMode=true
4. wait-until-condition condition="...CurrentPhase == 'Idle'"
5. record-start
6. [手动操作或 record-add-*]
7. record-stop
8. record-export（保存）
9. replay-input speed=1.0
10. wait-for-stable condition="[稳定条件]" stableForFrames=10
11. screenshot-game-view + ui-hierarchy-snapshot
12. reflection-method-call
13. console-get-logs
14. editor-application-set-state playMode=false
```

### 模板 5：边界场景验证（后门 API）

```
1. state-reset strategy="auto"
2. wait-until-condition condition="GameController.Instance != null"
3. editor-application-set-state playMode=true
4. wait-until-condition condition="...CurrentPhase == 'Idle'"
5. reflection-method-call typeName="XxxTestHelper" methodName="Setup[场景]"
6. wait-until-condition condition="[场景就绪条件]"
7. [执行待验证操作]
8. wait-until-condition condition="[预期结果条件]"
9. screenshot-game-view + ui-hierarchy-snapshot + reflection-method-call
10. console-get-logs
11. editor-application-set-state playMode=false
```
