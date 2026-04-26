# Output Contract
<!-- 输出规范：验证报告必须遵循的格式 -->

Use this structure when reporting validation work driven by this skill.

## Required Sections
<!-- 必需章节 -->

### 1. Route
<!-- 1. 路由 -->

State:

- the selected change class — 选择的变更类型
- why that route was chosen — 为什么选择这个路由
- which files or symptoms drove the classification — 哪些文件或症状驱动了分类

### 2. Layers Run
<!-- 2. 运行的层级 -->

List each layer that actually ran:

- Static review — 静态审查
- EditMode tests — 编辑模式测试
- Unity Editor inspection — Unity编辑器检查
- PlayMode automation — 播放模式自动化

For each layer, say whether it ran, was skipped by route, or was blocked by environment.
<!-- 对每个层级，说明是运行了、被路由跳过、还是被环境阻塞 -->

### 3. Unity-MCP Tools Used
<!-- 3. Unity-MCP 工具使用 -->

List the specific Unity-MCP tools called:

```
Example:
- script-execute: compiled successfully
- tests-run: 3 tests passed
- scene-get-data: hierarchy valid
- reflection-method-call: checked controller state
- screenshot-game-view: captured before/after
- console-get-logs: no errors
```

### 4. Evidence
<!-- 4. 证据 -->

Summarize the hard evidence:

- tests and their result — 测试及其结果
- hierarchy or prefab findings — 层级或预制体发现
- screenshot observations — 截图观察
- log findings — 日志发现
- runtime probe findings — 运行时探针发现

### 5. Findings
<!-- 5. 发现 -->

Call out the actual issue or confirmed pass conditions. Prefer precise statements such as:

- input lock stayed true after action completion — 操作完成后输入锁保持为 true
- controller never returned to idle state — 控制器从未返回空闲状态
- prefab was missing required component — 预制体缺失必需组件
- route passed with no logs and expected state transitions — 路由通过，无日志，状态转换符合预期

### 6. Unity-MCP Status
<!-- 6. Unity-MCP 状态 -->

Separate these clearly:

- Unity-MCP work completed — Unity-MCP 已完成的工作
- Unity-MCP work not completed — Unity-MCP 未完成的工作
- Unity-MCP connection status — Unity-MCP 连接状态（connected/disconnected/unavailable）

Do not merge them into one vague sentence.
<!-- 不要合并成一句模糊的话 -->

### 7. Verdict
<!-- 7. 结论 -->

Use one of these exact verdict styles:
<!-- 使用以下精确的结论风格之一 -->

- `Acceptable now` — 现在可接受
- `Conditionally acceptable with known gaps` — 有已知缺口，条件性可接受
- `Not acceptable yet` — 尚不可接受

Then give one sentence that explains why.
<!-- 然后用一句话解释原因 -->

### 8. Next Action
<!-- 8. 下一步行动 -->

Give the single most useful next Unity-MCP command or manual action.
<!-- 给出最有用的下一个 Unity-MCP 命令或手动操作 -->

### 9. Blockers (if any)
<!-- 9. 阻塞项（如有） -->

If any validation layers could not run, list:
- Which layer was blocked
- The exact blocker (e.g., "Unity Editor not connected", "Compilation failed")
- Whether custom tools were unavailable (e.g., "Input simulation tools not installed")

### 10. Troubleshooting Reference
<!-- 10. 故障排除参考 -->

If issues occurred, reference [troubleshooting.md](troubleshooting.md) for relevant solutions.

## Short Example
<!-- 简短示例 -->

```text
Route: Input / interaction + PlayMode runtime regression.
路由：输入/交互 + PlayMode运行时回归。
Why: The changed files are under input handling and the symptom appears only after several interactions.
原因：变更文件在输入处理下，症状只在多次交互后出现。

Layers run:
层级运行情况：
- Static review: ran — 静态审查：已运行
- EditMode: skipped by route — 编辑模式测试：被路由跳过
- Unity Editor inspection: ran — Unity编辑器检查：已运行
- PlayMode automation: ran — PlayMode自动化：已运行

Unity-MCP tools used:
Unity-MCP工具使用：
- script-execute: compiled successfully — 编译成功
- scene-get-data: hierarchy valid — 层级有效
- editor-application-set-state: entered PlayMode — 进入PlayMode
- screenshot-game-view: captured before/after — 捕获前后截图
- reflection-method-call: controller.CurrentPhase = 'Processing' — 控制器状态为Processing
- console-get-logs: no errors — 无错误日志

Evidence:
证据：
- Compile passed. — 编译通过。
- Scene wiring was valid. — 场景连线有效。
- After the third action, input lock remained true. — 第三次操作后，输入锁保持为 true。
- Logs were clean. — 日志干净。

Findings:
发现：
- Action completed visually, but the controller never released input.
- 操作视觉上完成，但控制器从未释放输入。

Unity-MCP status:
Unity-MCP状态：
- Completed: compile, scene inspection, PlayMode repro, runtime state probe
- 已完成：编译、场景检查、PlayMode复现、运行时状态探针
- Not completed: none
- 未完成：无
- Connection: connected
- 连接状态：已连接

Verdict: Not acceptable yet.
结论：尚不可接受。
Reason: The system can become permanently non-interactive after a valid repro path.
原因：系统在有效复现路径后可能永久不可交互。

Next action: Use reflection-method-call to inspect the action completion callback and unlock path.
下一步：使用 reflection-method-call 检查操作完成回调和解锁路径。

Blockers:
阻塞项：
- None
- 无

Troubleshooting reference: N/A
故障排除参考：不适用
```