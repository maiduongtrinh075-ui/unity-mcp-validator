# Troubleshooting Guide
<!-- 故障排除指南 -->

Use this guide when validation encounters issues.

## Unity-MCP Connection Issues

### MCP server not responding

**Symptom**: Tool calls timeout or return connection errors.

**Causes**:
1. Unity Editor not running
2. Unity-MCP plugin not installed
3. MCP server configuration incorrect
4. Firewall blocking connection

**Solutions**:
1. Open Unity Editor with target project
2. Verify plugin exists: check `Assets/Scripts/MCP/` folder
3. Restart MCP server in Claude Code settings
4. Check firewall allows Unity Editor connections

### Unity-MCP plugin not found

**Symptom**: MCP tools not available or return "plugin not found".

**Solutions**:
```bash
# Install via CLI
npm install -g unity-mcp-cli
unity-mcp-cli install-plugin ./YourProject

# Or manually from GitHub
# https://github.com/IvanMurzak/Unity-MCP
```

### Tool call returns "Editor not available"

**Symptom**: `editor-application-*` or other Editor tools fail.

**Causes**:
1. Unity Editor is closed
2. Editor is busy (compiling, loading)
3. Wrong project open

**Solutions**:
1. Open correct Unity project
2. Wait for Editor to finish current operation
3. Use `script-execute` to test connection

## Compilation Issues

### script-execute fails

**Symptom**: `script-execute` returns compilation errors.

**Causes**:
1. Syntax errors in code snippet
2. Missing using directives
3. Project has existing compilation errors

**Solutions**:
1. Fix syntax in code snippet
2. Add required `using` directives at top
3. Fix project compilation errors first via Unity Console

### Project won't compile

**Symptom**: Unity Console shows compilation errors.

**Solutions**:
1. Check Unity Console for specific errors
2. Fix errors in C# scripts
3. Do NOT proceed to PlayMode until compilation passes

## PlayMode Issues

### Can't enter PlayMode

**Symptom**: `editor-application-set-state playMode=true` fails.

**Causes**:
1. Compilation errors exist
2. Infinite loop in Awake/Start
3. Missing scene loaded
4. Editor in broken state

**Solutions**:
1. Check Unity Console for errors
2. Try entering PlayMode manually in Editor
3. Open target scene: `scene-open`
4. Restart Unity Editor

### reflection-method-call fails

**Symptom**: Reflection calls return errors.

**Causes**:
1. Method doesn't exist
2. Wrong class name or assembly
3. Method is static but called with instance
4. Parameters mismatch

**Solutions**:
1. Use `reflection-method-find` to locate correct method
2. Check class name and assembly name
3. For static methods, omit instance parameter
4. Match parameter types exactly

### Input simulation not working

**Symptom**: Custom input tools return errors or do nothing.

**Causes**:
1. Custom tools not installed
2. Input System not enabled
3. Wrong tool for target (UI vs world-space)
4. EventSystem missing (for UI tools)

**Solutions**:
1. Copy custom tool scripts from `custom-tools-input.md`
2. Enable Input System in Project Settings → Player
3. Use `simulate-click-ui` for UI, `simulate-click-world` for gameplay
4. Add EventSystem to scene for UI interaction

## Routing Issues

### Wrong route selected

**Symptom**: Validator selects inappropriate validation layers.

**Solutions**:
1. Create `validation-config.yaml` with your file patterns
2. Specify change class explicitly:
   ```
   Validate this as input/interaction change
   ```

### Validator skips required layer

**Symptom**: PlayMode skipped when input/animation involved.

**Solutions**:
1. Explicitly request full validation:
   ```
   Run full pre-acceptance sweep including PlayMode
   ```
2. Check output for "blocked" vs "skipped by route"

## Tool-Specific Issues

### screenshot-game-view returns empty

**Symptom**: Screenshot captures black or empty image.

**Causes**:
1. PlayMode not entered
2. Game window minimized
3. Camera not rendering

**Solutions**:
1. Confirm PlayMode with `editor-application-get-state`
2. Keep Game window visible
3. Check camera is active

### console-get-logs returns no logs

**Symptom**: No logs captured even after actions.

**Causes**:
1. Console was cleared
2. Log filtering too strict
3. No errors/warnings occurred

**Solutions**:
1. Don't clear Console before capturing
2. Use `console-get-logs` without filter for all logs
3. Generate test log: `script-execute 'Debug.Log("test");'`

### gameobject-find returns nothing

**Symptom**: GameObject search fails.

**Causes**:
1. Object doesn't exist
2. Object is inactive
3. Wrong scene context
4. Prefab mode active

**Solutions**:
1. Check scene is open: `scene-list-opened`
2. Use `gameobject-find include-inactive=true`
3. Open correct scene first
4. Close prefab mode: `assets-prefab-close`

## Performance Issues

### Tool calls slow

**Symptom**: MCP tool calls take long time.

**Causes**:
1. Unity Editor busy
2. Large scene/hierarchy
3. Network latency (remote MCP)

**Solutions**:
1. Wait for Editor to finish current operation
2. Limit hierarchy depth: `scene-get-data max-depth=2`
3. Use local MCP server if possible

### PlayMode validation hangs

**Symptom**: Validation doesn't complete.

**Causes**:
1. Game stuck in infinite loop
2. Input lock never released
3. Long animation sequence

**Solutions**:
1. Stop PlayMode manually if needed
2. Use timeout in validation workflow
3. Report as "blocked by timeout" and continue

## Recovery Procedures

### When Unity Editor freezes

1. Force-quit Unity Editor
2. Kill Unity processes
3. Restart Editor and open project
4. Re-run validation from Layer 1

### When MCP connection lost

1. Restart MCP server in Claude Code settings
2. Restart Unity Editor
3. Test connection: `editor-application-get-state`
4. Re-run validation

### When custom tools broken

1. Check `Assets/Scripts/MCP/` folder exists
2. Re-copy scripts from `custom-tools-input.md`
3. Wait for Unity to recompile
4. Test: `simulate-click-world x=100 y=100`

## Reporting Blockers

Always report blockers explicitly:

```text
Unity-MCP status:
- Completed: static review, scene inspection
- Not completed: PlayMode validation
- Connection: disconnected (MCP server timeout)
- Blocker: Unity Editor not responding after 30 seconds
```

Do NOT hide blockers in vague statements like "manual verification needed".