# Unity-MCP Complete Tool Reference
<!-- Unity-MCP 完整工具参考 -->

Unity-MCP 提供 50+ 内置工具，覆盖 Unity 编辑器和运行时的各个方面。本文档按功能分类列出所有工具及其验证用途。

---

## 📂 Asset Management 资源管理
<!-- 管理文件、文件夹和项目资源 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `assets-create-folder` | 创建文件夹 | 创建新目录（支持嵌套路径） | 设置测试目录结构 |
| `assets-delete` | 删除资源 | 删除特定资源或文件 | 清理测试资源 |
| `assets-find` | 查找资源 | 使用搜索过滤器查找资源（如 `t:Texture`） | 确认资源存在 |
| `assets-find-built-in` | 查找内置资源 | 搜索 Unity 编辑器内置资源 | 检查内置资源引用 |
| `assets-copy` | 复制资源 | 复制资源 | 创建测试副本 |
| `assets-move` | 移动/重命名资源 | 移动或重命名资源 | 组织项目结构 |
| `assets-get-data` | 获取资源数据 | 获取资源的元数据或内容，含所有可序列化字段 | **核心探针**：检查预制体引用、材质属性等 |
| `assets-modify` | 修改资源 | 修改项目中的资源文件 | 修改配置 |
| `assets-refresh` | 刷新资源库 | 强制刷新 AssetDatabase | 确保资源变更生效 |

### Materials & Shaders 材质与着色器

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `assets-material-create` | 创建材质 | 创建新的 Material 资源 | 创建测试材质 |
| `assets-shader-list-all` | 列出所有着色器 | 列出项目中所有可用的着色器 | 确认着色器引用有效 |

### Prefabs 预制体

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `assets-prefab-instantiate` | 实例化预制体 | 在活动场景中生成预制体 | **核心验证**：确认预制体实例化正确 |
| `assets-prefab-create` | 创建预制体 | 从场景对象创建预制体 | 创建测试预制体 |
| `assets-prefab-open` | 打开预制体 | 打开预制体编辑模式 | 检查预制体内部结构 |
| `assets-prefab-close` | 关闭预制体 | 关闭预制体编辑模式 | 退出预制体检查 |
| `assets-prefab-save` | 保存预制体 | 保存预制体编辑模式中的更改 | 保存预制体修改 |

---

## 🎮 GameObject 游戏对象
<!-- 管理场景对象和层级 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `gameobject-create` | 创建对象 | 创建新的 GameObject（空或基本体） | 创建测试对象 |
| `gameobject-destroy` | 销毁对象 | 删除 GameObject | 清理测试对象 |
| `gameobject-duplicate` | 复制对象 | 克隆 GameObject | 创建副本 |
| `gameobject-find` | 查找对象 | 按名称、标签或类型查找对象 | **核心探针**：定位目标对象 |
| `gameobject-modify` | 修改对象 | 更新 Transform、名称、标签、层级、激活状态 | 修改对象属性 |
| `gameobject-set-parent` | 设置父对象 | 更改层级父级 | 组织层级结构 |

### Components 组件

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `gameobject-component-add` | 添加组件 | 添加组件（如 `Rigidbody`） | 添加测试组件 |
| `gameobject-component-destroy` | 销毁组件 | 删除组件 | 清理组件 |
| `gameobject-component-get` | 获取组件详情 | 获取组件详细信息，含所有字段和属性 | **核心探针**：检查组件状态 |
| `gameobject-component-modify` | 修改组件 | 设置字段、属性或对象引用 | 修改组件配置 |
| `gameobject-component-list-all` | 列出所有组件类型 | 列出可用的组件类型 | 查找可添加的组件 |

---

## 🎬 Scene Management 场景管理
<!-- 管理场景 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `scene-create` | 创建场景 | 创建新的 Scene 资源 | 创建测试场景 |
| `scene-open` | 打开场景 | 在编辑器中打开场景 | 打开目标场景 |
| `scene-save` | 保存场景 | 保存当前场景 | 保存修改 |
| `scene-unload` | 卸载场景 | 卸载附加场景 | 清理场景 |
| `scene-set-active` | 设置活动场景 | 设置活动场景 | 切换活动场景 |
| `scene-get-data` | 获取场景数据 | 获取场景中的根对象列表 | **核心探针**：检查场景层级 |
| `scene-list-opened` | 列出已打开场景 | 列出当前打开的场景 | 确认目标场景已打开 |

---

## 📝 Scripting 脚本
<!-- C# 脚本管理 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `script-update-or-create` | 创建/更新脚本 | 创建或更新 C# 脚本文件 | 修改代码 |
| `script-read` | 读取脚本 | 读取 `.cs` 文件内容 | 检查脚本内容 |
| `script-delete` | 删除脚本 | 删除脚本文件 | 清理脚本 |
| `script-execute` | 执行脚本 | 动态编译并运行 C# 代码片段 | **核心探针**：运行时检查任意代码 |

**`script-execute` 是最强大的探针工具**，可用于：
- 检查任意运行时状态
- 调用任意方法
- 验证任意逻辑
- 创建临时测试代码

---

## 📦 Package Manager 包管理
<!-- Unity 包管理 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `package-list` | 列出已安装包 | 列出已安装的包 | 检查依赖 |
| `package-add` | 安装包 | 安装包（Registry、Git、本地） | 添加依赖 |
| `package-remove` | 卸载包 | 卸载包 | 移除依赖 |
| `package-search` | 搜索包 | 搜索 Unity Registry | 查找可用包 |

---

## 🧩 Object 对象
<!-- Unity 对象通用操作 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `object-get-data` | 获取对象数据 | 获取 Unity 对象的序列化数据，含属性和字段 | **核心探针**：检查任意对象状态 |
| `object-modify` | 修改对象 | 直接修改 Unity 对象的字段和属性 | 修改对象状态 |

---

## 📸 Screenshot 截图
<!-- 捕获视觉证据 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `screenshot-camera` | 相机截图 | 从指定相机捕获截图（默认主相机） | 检查渲染结果 |
| `screenshot-game-view` | 游戏视图截图 | 从 Unity 编辑器游戏视图捕获截图 | **核心证据**：捕获 PlayMode 状态 |
| `screenshot-scene-view` | 场景视图截图 | 从 Unity 编辑器场景视图捕获截图 | 检查编辑器状态 |

---

## 🧪 Testing 测试
<!-- 运行 Unity 测试 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `tests-run` | 运行测试 | 执行 Unity 测试（EditMode/PlayMode），支持过滤和详细结果 | **核心验证**：确定性逻辑验证 |

---

## 💡 Advanced & Editor 高级与编辑器
<!-- 编辑器状态控制与反射 -->

### Console 控制台

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `console-get-logs` | 获取日志 | 获取 Unity Console 日志，支持过滤 | **核心证据**：检查错误、警告、输出 |

### Editor Application 编辑器状态

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `editor-application-get-state` | 获取编辑器状态 | 检查 Play/Pause/Edit 模式状态 | 确认 PlayMode 状态 |
| `editor-application-set-state` | 设置编辑器状态 | 设置 Play/Pause 状态 | **核心控制**：进入/退出 PlayMode |

### Editor Selection 编辑器选择

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `editor-selection-get` | 获取当前选择 | 获取编辑器当前选择的对象 | 检查选中状态 |
| `editor-selection-set` | 设置当前选择 | 设置编辑器当前选择的对象 | 设置选择 |

### Reflection 反射
<!-- 最强大的运行时探针工具 -->

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `reflection-method-find` | 查找方法 | 查找任何 C# 方法（公开/私有） | 定位要调用的方法 |
| `reflection-method-call` | 调用方法 | 执行找到的方法，可传递参数和对象引用 | **核心探针**：调用任意方法获取状态 |

**`reflection-method-call` 是最强大的运行时探针**：
- 可调用任何公开或私有方法
- 可传递任何参数（包括运行时对象引用）
- 可获取任何状态信息
- 甚至可以调用 DLL 中的方法

---

## 🔧 Custom Tools 自定义工具
<!-- 需要自行添加的工具 -->

以下工具需要根据 `custom-tools-input.md` 创建：

### Mouse Input 鼠标输入（世界空间）

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `simulate-click-world` | 点击世界空间 | 模拟鼠标点击屏幕坐标 | 点击带碰撞体的游戏对象 |
| `simulate-mouse-down` | 鼠标按下 | 按下鼠标按钮不释放 | 拖拽开始 |
| `simulate-mouse-up` | 鼠标释放 | 释放鼠标按钮 | 拖拽结束 |
| `simulate-mouse-move` | 鼠标移动 | 移动鼠标位置 | 悬停检查 |
| `simulate-drag-world` | 拖拽操作 | 从起点拖到终点 | 拖拽交互 |

### Mouse UI 鼠标输入（UI 空间）

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `simulate-click-ui` | 点击 UI | 通过 EventSystem 点击 UI 元素 | 点击按钮、滑块等 |
| `simulate-hover-ui` | UI 悬停 | 悬停在 UI 元素上 | UI 悬停检查 |
| `click-button-by-name` | 按名称点击按钮 | 通过 GameObject 名称点击按钮 | 直接点击已知按钮 |

### Keyboard 键盘输入

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `simulate-key-press` | 按键 | 模拟按下并释放键盘按键 | 键盘输入 |
| `simulate-key-down` | 按键按下 | 按下键盘按键不释放 | 持续按键 |
| `simulate-key-up` | 按键释放 | 释放键盘按键 | 结束按键 |
| `simulate-text-input` | 文本输入 | 模拟输入一段文本 | 打字输入 |

### Recording 录制与回放

| Tool ID | 中文名称 | 描述 | 验证用途 |
|---------|----------|------|----------|
| `record-start` | 开始录制 | 开始录制所有输入 | 录制输入序列 |
| `record-stop` | 停止录制 | 停止录制并返回事件数量 | 结束录制 |
| `record-summary` | 录制摘要 | 获取录制事件的摘要 | 查看录制内容 |
| `record-add-click` | 添加点击事件 | 手动添加点击事件到录制 | 手动构建录制 |
| `record-add-key` | 添加按键事件 | 手动添加按键事件到录制 | 手动构建录制 |
| `replay-input` | 回放输入 | 按时间回放录制的事件 | **核心验证**：确定性复现 |
| `record-export` | 导出录制 | 导出录制为 JSON | 保存录制 |
| `record-import` | 导入录制 | 从 JSON 导入录制 | 加载录制 |
| `record-clear` | 清除录制 | 清除所有录制事件 | 清空录制 |

---

## 📊 工具统计

| 分类 | 内置工具数 | 自定义工具数 | 总计 |
|------|-----------|-------------|------|
| Asset Management | 15 | 0 | 15 |
| GameObject | 11 | 0 | 11 |
| Scene Management | 7 | 0 | 7 |
| Scripting | 4 | 0 | 4 |
| Package Manager | 4 | 0 | 4 |
| Object | 2 | 0 | 2 |
| Screenshot | 3 | 0 | 3 |
| Testing | 1 | 0 | 1 |
| Advanced & Editor | 8 | 0 | 8 |
| Mouse Input | 0 | 5 | 5 |
| Mouse UI | 0 | 3 | 3 |
| Keyboard | 0 | 4 | 4 |
| Recording | 0 | 9 | 9 |
| **总计** | **56** | **21** | **77** |

---

## 🎯 验证层级工具映射

### Layer 1: Static Review（静态审查）
- 无需 MCP 工具
- 直接读取文件和 diff

### Layer 2: EditMode / Test Runner（编辑模式测试）
| 场景 | 推荐工具 |
|------|----------|
| 运行 EditMode 测试 | `tests-run mode=EditMode` |
| 编译确认 | `script-execute` 执行简单代码 |

### Layer 3: Unity Editor / Wiring Validation（编辑器检查）
| 场景 | 推荐工具 |
|------|----------|
| 检查场景层级 | `scene-get-data`, `gameobject-find` |
| 检查预制体引用 | `assets-get-data`, `assets-prefab-open` |
| 检查组件状态 | `gameobject-component-get`, `object-get-data` |
| 检查脚本内容 | `script-read` |
| 检查日志 | `console-get-logs` |
| 检查包依赖 | `package-list` |

### Layer 4: PlayMode Automation（运行模式）
| 场景 | 推荐工具 |
|------|----------|
| 进入 PlayMode | `editor-application-set-state playMode=true` |
| 点击游戏对象 | `simulate-click-world`（自定义） |
| 点击 UI | `simulate-click-ui`（自定义） |
| 键盘输入 | `simulate-key-press`（自定义） |
| 捕获截图 | `screenshot-game-view` |
| 检查日志 | `console-get-logs` |
| 检查运行时状态 | `reflection-method-call`, `script-execute` |
| 录制输入序列 | `record-start`, `record-stop`（自定义） |
| 回放输入序列 | `replay-input`（自定义） |
| 退出 PlayMode | `editor-application-set-state playMode=false` |

---

## 💡 最佳实践

### 1. 最小化工具调用
- 先用 `gameobject-find` 定位目标
- 再用 `gameobject-component-get` 检查状态
- 不要重复调用相同工具

### 2. 批量获取数据
- `assets-get-data` 一次获取所有序列化字段
- `object-get-data` 一次获取所有属性
- 不要逐字段调用

### 3. 使用 reflection-method-call 进行复杂检查
```csharp
// 一次性获取多个状态
var controller = FindObjectOfType<MyController>();
return $"Phase:{controller.phase}, Lock:{controller.inputLocked}, Score:{controller.score}";
```

### 4. 使用 script-execute 进行验证代码
```csharp
// 执行验证逻辑
var board = FindObjectOfType<Board>();
var count = board.GetGemCount();
return $"Gems: {count}, Valid: {count == 64}";
```

### 5. 截图 + 日志 + 状态 = 完整证据
每次 PlayMode 问题至少收集：
- `screenshot-game-view` → 可见状态
- `console-get-logs` → 错误和警告
- `reflection-method-call` → 运行时状态

---

## 🔄 工具组合示例

### 检查预制体连线是否正确
```
1. assets-prefab-open prefabPath="Assets/Prefabs/Gem.prefab"
2. gameobject-find name="Gem" (in prefab context)
3. gameobject-component-get component="SpriteRenderer"
4. assets-get-data assetPath="Assets/Prefabs/Gem.prefab"
5. assets-prefab-close
```

### 复现并诊断运行时 bug
```
1. editor-application-set-state playMode=true
2. screenshot-game-view → 基线截图
3. simulate-click-world x=100 y=200 → 第一次点击
4. simulate-click-world x=300 y=400 → 第二次点击
5. screenshot-game-view → bug 截图
6. console-get-logs → 日志
7. reflection-method-call → 检查状态
8. editor-application-set-state playMode=false
```

### 录制并回放复杂交互
```
1. record-start
2. [手动操作游戏]
3. record-stop
4. record-export → 保存 JSON
5. editor-application-set-state playMode=true
6. record-import json=<saved_data>
7. replay-input speed=1.0
8. screenshot-game-view → 结果截图
```

---

## 📁 文件结构

完整的自定义工具文件结构：
```
Assets/Scripts/MCP/
├── Tool_MouseInput.cs       # 世界空间鼠标输入
├── Tool_MouseUI.cs          # UI 空间鼠标输入
├── Tool_KeyboardInput.cs    # 键盘输入
├── Tool_InputRecording.cs   # 录制与回放
└── InputRecorder.cs         # 自动录制监听器（可选 MonoBehaviour）
```