# JSON Output Schema
<!-- JSON 输出规范：机器可消费的结构化验证报告 -->

定义验证报告的 JSON Schema，供下游 Codex / 自动化审查系统解析。

**设计原则：**
- 与 Markdown 报告**双轨并行**，不替代人可读格式
- 每个 JSON 字段都有明确语义，不留模糊空间
- Codex 可直接从 JSON 中提取：失败的断言、卡住的状态机、异常堆栈

---

## Schema 定义

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "UnityMcpValidationReport",
  "version": "2.0.0",
  "type": "object",
  "required": ["schema_version", "test_id", "route", "layers", "verdict"],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "2.0.0",
      "description": "Schema 版本号，用于下游解析器兼容性检查"
    },
    "test_id": {
      "type": "string",
      "description": "唯一测试标识，格式：route_{class}_{timestamp}",
      "examples": ["route_input_interaction_20260427_161500"]
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "测试执行时间 (ISO 8601)"
    },
    "project": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "unity_version": { "type": "string" },
        "target_platform": { "type": "string" }
      }
    },
    "route": {
      "type": "object",
      "required": ["change_class", "reason"],
      "properties": {
        "change_class": {
          "type": "string",
          "enum": [
            "model_rules",
            "controller_state_machine",
            "input_interaction",
            "animation_input_lock",
            "scene_prefab_wiring",
            "ui_hud",
            "project_settings_package",
            "playmode_runtime_bug",
            "pre_acceptance_sweep"
          ],
          "description": "变更分类"
        },
        "reason": {
          "type": "string",
          "description": "选择此路由的原因"
        },
        "changed_files": {
          "type": "array",
          "items": { "type": "string" },
          "description": "变更文件列表"
        },
        "symptoms": {
          "type": "array",
          "items": { "type": "string" },
          "description": "观察到的症状描述"
        }
      }
    },
    "layers": {
      "type": "object",
      "required": ["static", "editmode", "editor", "playmode"],
      "properties": {
        "static": {
          "$ref": "#/$defs/layer_result"
        },
        "editmode": {
          "$ref": "#/$defs/layer_result"
        },
        "editor": {
          "$ref": "#/$defs/layer_result"
        },
        "playmode": {
          "type": "object",
          "required": ["status"],
          "properties": {
            "status": {
              "$ref": "#/$defs/layer_status"
            },
            "skip_reason": { "type": "string" },
            "blocker": { "type": "string" },
            "duration_ms": { "type": "integer" },
            "sub_layers": {
              "type": "object",
              "properties": {
                "4a_playmode_control": { "$ref": "#/$defs/layer_result" },
                "4b_input_simulation": { "$ref": "#/$defs/layer_result" },
                "4c_evidence_collection": { "$ref": "#/$defs/layer_result" }
              }
            }
          }
        }
      }
    },
    "tools_used": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["tool", "result"],
        "properties": {
          "tool": { "type": "string", "description": "MCP 工具 ID" },
          "params": { "type": "object", "description": "调用参数" },
          "result": {
            "type": "string",
            "enum": ["success", "error", "timeout", "not_available"],
            "description": "调用结果"
          },
          "duration_ms": { "type": "integer", "description": "执行耗时" },
          "output_summary": { "type": "string", "description": "输出摘要" }
        }
      }
    },
    "state_snapshots": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["phase", "timestamp_ms"],
        "properties": {
          "phase": {
            "type": "string",
            "description": "快照时机描述",
            "examples": ["before_interaction", "after_interaction", "bug_reproduced"]
          },
          "timestamp_ms": { "type": "integer", "description": "相对于测试开始的毫秒数" },
          "controller_phase": { "type": ["string", "null"] },
          "input_locked": { "type": ["boolean", "null"] },
          "custom_state": {
            "type": "object",
            "description": "项目特定的状态变量快照",
            "additionalProperties": true
          }
        }
      },
      "description": "运行时状态快照序列，用于状态机流转追踪"
    },
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["severity", "description"],
        "properties": {
          "severity": {
            "type": "string",
            "enum": ["pass", "warning", "fail", "blocker"],
            "description": "发现严重级别"
          },
          "description": { "type": "string" },
          "location": { "type": "string", "description": "代码位置或对象路径" },
          "exception_stack": {
            "type": "array",
            "items": { "type": "string" },
            "description": "用户代码帧 + 前3层引擎帧"
          },
          "expected": { "type": ["string", "number", "boolean", "null"] },
          "actual": { "type": ["string", "number", "boolean", "null"] }
        }
      }
    },
    "evidence": {
      "type": "object",
      "properties": {
        "screenshots": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "label": { "type": "string" },
              "path": { "type": "string" },
              "phase": { "type": "string" }
            }
          }
        },
        "logs": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "level": { "type": "string", "enum": ["error", "warning", "log"] },
              "count": { "type": "integer" },
              "sample": { "type": "string", "description": "前3条日志摘要" }
            }
          }
        },
        "ui_snapshots": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "label": { "type": "string" },
              "phase": { "type": "string" },
              "node_count": { "type": "integer" }
            }
          }
        }
      }
    },
    "verdict": {
      "type": "object",
      "required": ["status", "reason"],
      "properties": {
        "status": {
          "type": "string",
          "enum": ["Acceptable", "ConditionallyAcceptable", "NotAcceptable"],
          "description": "验证结论"
        },
        "reason": { "type": "string" },
        "known_gaps": {
          "type": "array",
          "items": { "type": "string" },
          "description": "已知但未验证的缺口（ConditionallyAcceptable 时必须列出）"
        }
      }
    },
    "next_action": {
      "type": "object",
      "properties": {
        "tool": { "type": "string", "description": "推荐的下一个 MCP 工具" },
        "params": { "type": "object" },
        "description": { "type": "string" }
      }
    },
    "blockers": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["layer", "blocker"],
        "properties": {
          "layer": { "type": "string" },
          "blocker": { "type": "string" },
          "action_needed": { "type": "string" }
        }
      }
    },
    "mcp_status": {
      "type": "object",
      "required": ["connection"],
      "properties": {
        "connection": {
          "type": "string",
          "enum": ["connected", "disconnected", "unavailable"]
        },
        "completed": {
          "type": "array",
          "items": { "type": "string" }
        },
        "not_completed": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    }
  },
  "$defs": {
    "layer_status": {
      "type": "string",
      "enum": ["ran", "skipped_by_route", "blocked", "not_available"]
    },
    "layer_result": {
      "type": "object",
      "required": ["status"],
      "properties": {
        "status": { "$ref": "#/$defs/layer_status" },
        "skip_reason": { "type": "string" },
        "blocker": { "type": "string" },
        "duration_ms": { "type": "integer" },
        "test_count": { "type": "integer" },
        "test_passed": { "type": "integer" },
        "test_failed": { "type": "integer" }
      }
    }
  }
}
```

---

## 完整示例

### 输入/交互验证报告

```json
{
  "schema_version": "2.0.0",
  "test_id": "route_input_interaction_20260427_161500",
  "timestamp": "2026-04-27T16:15:00+08:00",
  "project": {
    "name": "MergeEmpire",
    "unity_version": "2022.3.20f1",
    "target_platform": "Android"
  },
  "route": {
    "change_class": "input_interaction",
    "reason": "Changed files under input handling; symptom appears after multiple interactions",
    "changed_files": [
      "Assets/Scripts/Board.cs",
      "Assets/Scripts/Gem.cs"
    ],
    "symptoms": ["滑动消除后分数不增加"]
  },
  "layers": {
    "static": {
      "status": "ran",
      "duration_ms": 450,
      "findings_count": 1
    },
    "editmode": {
      "status": "ran",
      "duration_ms": 2300,
      "test_count": 8,
      "test_passed": 8,
      "test_failed": 0
    },
    "editor": {
      "status": "ran",
      "duration_ms": 1200
    },
    "playmode": {
      "status": "ran",
      "duration_ms": 8500,
      "sub_layers": {
        "4a_playmode_control": {
          "status": "ran",
          "duration_ms": 800
        },
        "4b_input_simulation": {
          "status": "ran",
          "duration_ms": 320
        },
        "4c_evidence_collection": {
          "status": "ran",
          "duration_ms": 7380
        }
      }
    }
  },
  "tools_used": [
    {
      "tool": "script-execute",
      "params": { "code": "return 1;" },
      "result": "success",
      "duration_ms": 450
    },
    {
      "tool": "tests-run",
      "params": { "mode": "EditMode", "filter": ".*Board.*" },
      "result": "success",
      "duration_ms": 2300,
      "output_summary": "8 tests passed"
    },
    {
      "tool": "scene-get-data",
      "result": "success",
      "duration_ms": 120
    },
    {
      "tool": "editor-application-set-state",
      "params": { "playMode": true },
      "result": "success",
      "duration_ms": 800
    },
    {
      "tool": "simulate-drag-world",
      "params": { "startX": 200, "startY": 300, "endX": 400, "endY": 300 },
      "result": "success",
      "duration_ms": 320
    },
    {
      "tool": "wait-until-condition",
      "params": { "condition": "Board.Instance.IsAnimating == false", "timeoutSeconds": 5 },
      "result": "success",
      "duration_ms": 1800,
      "output_summary": "Condition met after 1.80s"
    },
    {
      "tool": "screenshot-game-view",
      "result": "success",
      "duration_ms": 150
    },
    {
      "tool": "ui-hierarchy-snapshot",
      "params": { "format": "json" },
      "result": "success",
      "duration_ms": 80,
      "output_summary": "42 nodes captured"
    },
    {
      "tool": "reflection-method-call",
      "params": { "method": "Board.Instance.GetScore" },
      "result": "success",
      "duration_ms": 30,
      "output_summary": "30"
    },
    {
      "tool": "console-get-logs",
      "result": "success",
      "duration_ms": 50,
      "output_summary": "0 errors, 2 warnings"
    }
  ],
  "state_snapshots": [
    {
      "phase": "before_interaction",
      "timestamp_ms": 3200,
      "controller_phase": "Idle",
      "input_locked": false,
      "custom_state": {
        "score": 0,
        "gem_count": 64,
        "board_phase": "WaitingForInput"
      }
    },
    {
      "phase": "after_drag",
      "timestamp_ms": 3520,
      "controller_phase": "Processing",
      "input_locked": true,
      "custom_state": {
        "score": 0,
        "gem_count": 64,
        "board_phase": "Animating"
      }
    },
    {
      "phase": "after_settle",
      "timestamp_ms": 5320,
      "controller_phase": "Idle",
      "input_locked": false,
      "custom_state": {
        "score": 30,
        "gem_count": 60,
        "board_phase": "WaitingForInput"
      }
    }
  ],
  "findings": [
    {
      "severity": "pass",
      "description": "滑动操作触发成功，消除动画播放正确"
    },
    {
      "severity": "pass",
      "description": "分数从 0 增加到 30（正确）"
    },
    {
      "severity": "pass",
      "description": "Gem 数量从 64 减少到 60（正确）"
    },
    {
      "severity": "pass",
      "description": "输入锁在消除完成后正确释放"
    },
    {
      "severity": "pass",
      "description": "无错误日志"
    }
  ],
  "evidence": {
    "screenshots": [
      { "label": "baseline", "path": "screenshots/before_drag.png", "phase": "before_interaction" },
      { "label": "after_elimination", "path": "screenshots/after_elimination.png", "phase": "after_settle" }
    ],
    "logs": [
      { "level": "error", "count": 0 },
      { "level": "warning", "count": 2, "sample": "Texture size exceeds 1024..." }
    ],
    "ui_snapshots": [
      { "label": "after_settle", "phase": "after_settle", "node_count": 42 }
    ]
  },
  "verdict": {
    "status": "Acceptable",
    "reason": "滑动交互验证通过，消除流程完整，分数和状态正确",
    "known_gaps": []
  },
  "next_action": null,
  "blockers": [],
  "mcp_status": {
    "connection": "connected",
    "completed": ["static_review", "editmode_tests", "editor_inspection", "playmode_full"],
    "not_completed": []
  }
}
```

### 输入锁未释放报告（失败案例）

```json
{
  "schema_version": "2.0.0",
  "test_id": "route_input_interaction_20260427_170000",
  "timestamp": "2026-04-27T17:00:00+08:00",
  "route": {
    "change_class": "input_interaction",
    "reason": "Bug: game freezes after 5 interactions",
    "changed_files": ["Assets/Scripts/InputHandler.cs"],
    "symptoms": ["玩了几次后游戏卡住"]
  },
  "layers": {
    "static": { "status": "ran", "duration_ms": 300 },
    "editmode": { "status": "skipped_by_route" },
    "editor": { "status": "ran", "duration_ms": 900 },
    "playmode": {
      "status": "ran",
      "duration_ms": 15000,
      "sub_layers": {
        "4a_playmode_control": { "status": "ran" },
        "4b_input_simulation": { "status": "ran" },
        "4c_evidence_collection": { "status": "ran" }
      }
    }
  },
  "state_snapshots": [
    {
      "phase": "after_5th_interaction",
      "timestamp_ms": 12000,
      "controller_phase": "Processing",
      "input_locked": true,
      "custom_state": {
        "interaction_count": 5,
        "board_phase": "Idle",
        "last_action_completed": true
      }
    }
  ],
  "findings": [
    {
      "severity": "fail",
      "description": "第5次操作后 InputLock 未释放",
      "location": "InputHandler.cs:142",
      "expected": "input_locked == false",
      "actual": "input_locked == true"
    },
    {
      "severity": "warning",
      "description": "控制器停留在 Processing 阶段，但 Board 已回到 Idle",
      "location": "GameController.cs:89"
    }
  ],
  "verdict": {
    "status": "NotAcceptable",
    "reason": "系统在有效复现路径后可能永久不可交互：InputLock 在第5次操作后未释放",
    "known_gaps": []
  },
  "next_action": {
    "tool": "reflection-method-call",
    "params": {
      "method": "InputHandler.Instance.ReleaseLock"
    },
    "description": "检查 ReleaseLock 方法是否存在及其调用路径"
  },
  "blockers": [],
  "mcp_status": {
    "connection": "connected",
    "completed": ["static_review", "editor_inspection", "playmode_full_with_repro"],
    "not_completed": []
  }
}
```

---

## Codex 消费指南

### 快速解析关键信号

```python
# Codex 自动审查脚本示例
import json

def parse_validation_report(json_path):
    with open(json_path) as f:
        report = json.load(f)

    # 1. 检查结论
    verdict = report["verdict"]["status"]
    if verdict == "Acceptable":
        return "APPROVED"

    # 2. 提取失败发现
    failures = [f for f in report["findings"] if f["severity"] == "fail"]
    if failures:
        for fail in failures:
            print(f"FAIL: {fail['description']}")
            if fail.get("location"):
                print(f"  → {fail['location']}")
            if fail.get("expected"):
                print(f"  Expected: {fail['expected']}, Actual: {fail['actual']}")

    # 3. 状态机卡住分析
    snapshots = report.get("state_snapshots", [])
    for snap in snapshots:
        if snap.get("input_locked") and snap.get("controller_phase") != "Idle":
            print(f"STUCK: Phase={snap['controller_phase']}, InputLocked={snap['input_locked']}")

    # 4. 异常堆栈
    for finding in report["findings"]:
        if finding.get("exception_stack"):
            # 只保留用户代码帧
            user_frames = [f for f in finding["exception_stack"]
                          if "Assets/" in f or "Scripts/" in f]
            print(f"EXCEPTION at: {user_frames[:3]}")

    return "NEEDS_FIX"
```

### 与 PR 审查集成

```
验证报告 → validation-report.json 附加到 PR
         → Codex 读取 JSON
         → 自动生成审查评论：
           - 失败断言的代码位置
           - 状态机卡住的流转路径
           - 建议修复方案
```

---

## 版本兼容

| Schema 版本 | 变更 | 向下兼容 |
|-------------|------|----------|
| 2.0.0 | 新增 JSON Schema，双轨输出 | ✅ Markdown 格式不变 |
| 1.x | 仅 Markdown 输出 | — |

下游解析器应检查 `schema_version` 字段，遇到未知版本时降级为只读 Markdown 报告。
