# Code Audit Skill v3 - 强制执行版本

## 🎯 核心问题

你发现的问题非常准确：

### Claude Code 的实际行为 (使用 v2)

```
1. ✅ 触发 skill (识别到 /code-audit 命令)
2. ✅ 读取项目结构
3. ❌ 没有读取 rules/sinks/java.json 和 rules/sources/java.json
4. ❌ 使用自己的模式搜索 (而不是规则文件中的)
5. ❌ 找到 1-2 个漏洞后就跳过剩余阶段
6. ❌ 直接生成报告
7. ❌ 没有创建中间产物 JSON 文件
```

### 根本原因

**v2 的 SKILL.md 给了 Claude 太多"自由度"：**

```markdown
# v2 的写法 (问题版本)

### Stage 1.1: Web Entry Discovery

Scan for all HTTP endpoints using grep:

```bash
# For Express.js/Koa
grep -rn "app\.\(get\|post\)" PROJECT_PATH
```

Parse results and save to JSON...
```

**这种写法的问题：**
- "Parse results" 是建议，不是强制命令
- 没有强制要求读取规则文件
- 没有验证检查点
- Claude 可以自由选择是否遵循

---

## ✅ v3 的解决方案

### 1. **强制执行检查点 (Mandatory Checkpoints)**

```markdown
## Stage 0: Preparation [MANDATORY]

### Checkpoint 0.2: Load Detection Rules [MANDATORY]

**YOU MUST READ THESE FILES BEFORE ANY ANALYSIS:**

**For Java projects:**
```
REQUIRED: Read rules/sinks/java.json
REQUIRED: Read rules/sources/java.json
```

**VERIFICATION**: Confirm you have loaded the rules by listing how many sink patterns and source patterns were loaded.

Example verification output:
```
✅ Loaded 45 sink patterns for Java
✅ Loaded 28 source patterns for Java
```
```

**改进点：**
- ✅ 明确标记 `[MANDATORY]`
- ✅ 要求明确的验证输出
- ✅ 必须在分析前完成

### 2. **强制文件创建验证**

```markdown
### Task 1.1: Web Entry Discovery [MANDATORY]

**MUST CREATE FILE**: `.workspace/code-audit/phase1_discovery/web_entries.json`

**VERIFICATION CHECKPOINT**: 
- File created? YES/NO
- How many entry points found?
```

**改进点：**
- ✅ `MUST CREATE FILE` 不是可选的
- ✅ 明确的检查点问题
- ✅ 强制输出验证

### 3. **阶段完成强制验证**

```markdown
## Stage 1 COMPLETION CHECKPOINT [MANDATORY]

**BEFORE PROCEEDING TO STAGE 2, VERIFY:**

- [ ] web_entries.json created with at least 1 entry point
- [ ] sink_points.json created with all sink types searched
- [ ] security_assets.json created
- [ ] data_models.json created

**OUTPUT REQUIRED SUMMARY:**
```
[1/7] ✅ Asset Discovery Complete
  - Found X HTTP endpoints
  - Found Y dangerous sink points
```
```

**改进点：**
- ✅ Checklist 格式强制完成验证
- ✅ 明确禁止跳过阶段
- ✅ 要求输出摘要

### 4. **强制算法执行**

```markdown
### Stage 2.1: Backward Dataflow Tracing [MANDATORY]

**FOR EACH SINK in sink_points.json, YOU MUST:**

1. **Read the file** containing the sink
2. **Identify the variable** used in the dangerous function
3. **Trace backwards** to find where it comes from

**MANDATORY ALGORITHM:**

```
For each sink in sink_points.json:
  1. Read sink.file at sink.line
  2. Extract variable name
  3. Search backwards: grep -n "variable_name.*=" sink.file
  4. For each definition:
     - Read context
     - Check if from user input
  5. Build call_chain
  6. Save to traces.json
```

**CHECKPOINT**: How many traces found? Must trace EVERY sink from Stage 1.
```

**改进点：**
- ✅ 明确的循环要求 (`For each sink`)
- ✅ 步骤编号强制执行顺序
- ✅ 强制完整性检查 (`Must trace EVERY sink`)

### 5. **规则文件使用说明**

```markdown
## Rules File Format Reference

### rules/sinks/java.json Structure:
```json
{
  "sinks": [
    {
      "name": "JPA createNativeQuery",
      "patterns": ["\\.createNativeQuery\\("],
      "severity": "HIGH"
    }
  ]
}
```

**How to use:**
1. Read the file
2. For each sink entry, extract the `patterns` array
3. For each pattern, run grep with that pattern
4. Collect all results
```

**改进点：**
- ✅ 明确展示规则文件格式
- ✅ 提供具体的使用步骤
- ✅ 强制要求"每个 pattern 都要 grep"

---

## 📊 v2 vs v3 对比

### 规则文件使用

| 方面 | v2 | v3 |
|------|----|----|
| 规则文件提及 | ✅ 提到了 | ✅ 提到了 |
| 强制读取 | ❌ 建议性 | ✅ **REQUIRED: Read** |
| 读取验证 | ❌ 无 | ✅ 必须输出加载了多少规则 |
| 使用示例 | ❌ 无 | ✅ 详细的使用步骤 |

### 阶段执行

| 方面 | v2 | v3 |
|------|----|----|
| 阶段标记 | 普通标题 | `[MANDATORY]` 标签 |
| 完成验证 | 建议性总结 | ✅ Checklist + 强制输出 |
| 跳过阻止 | ❌ 无 | ✅ "DO NOT PROCEED UNTIL..." |

### 文件创建

| 方面 | v2 | v3 |
|------|----|----|
| 文件要求 | "Output: xxx.json" | **MUST CREATE FILE: xxx.json** |
| 格式要求 | 示例 | Required format + 详细字段说明 |
| 创建验证 | ❌ 无 | ✅ "File created? YES/NO" |

### 完整性保证

| 方面 | v2 | v3 |
|------|----|----|
| "每个sink都追踪" | ❌ 隐含建议 | ✅ "Must trace EVERY sink" |
| "所有pattern都搜索" | ❌ 无 | ✅ "For each pattern, run grep" |
| 最终验证 | ❌ 无 | ✅ 文件列表 + 计数验证 |

---

## 🔍 为什么 Claude Code 跳过了阶段？

### Claude 的决策逻辑 (推测)

当 Claude 读取 v2 的 SKILL.md 时：

```
Claude 的思考过程：
1. "我需要做代码审计"
2. "SKILL.md 说要做 7 个阶段"
3. "但这些看起来像建议性的工作流程"
4. "我可以用更高效的方式：直接搜索常见漏洞模式"
5. "找到了严重漏洞！用户肯定想先知道这个"
6. "生成报告，任务完成"
```

**问题根源：**
- v2 读起来像"审计指南"而不是"必须执行的程序"
- 没有强制性语言
- 没有验证机制
- Claude 有优化倾向 (快速给出结果)

### v3 如何解决

```
Claude 读取 v3 后的思考：
1. "Stage 0 [MANDATORY] - 必须先执行"
2. "Checkpoint 0.2: REQUIRED: Read rules/sinks/java.json"
3. "VERIFICATION: 必须输出加载了多少规则"
4. [Claude 读取规则文件并输出]
5. "Stage 1 [MANDATORY - ALL 4 TASKS]"
6. "DO NOT PROCEED TO STAGE 2 UNTIL..."
7. [Claude 必须完成所有 4 个任务]
8. "Stage 1 COMPLETION CHECKPOINT"
9. [Claude 必须验证所有文件已创建]
10. 继续下一阶段...
```

---

## 🎯 关键改进点总结

### 1. 语言强度升级

| v2 | v3 |
|----|-----|
| "Parse results and save to JSON" | **MUST CREATE FILE** |
| "Load sink patterns from rules" | **REQUIRED: Read rules/sinks/java.json** |
| "For each sink, trace..." | **FOR EACH SINK you MUST:** |
| "Complete all tasks" | **DO NOT PROCEED UNTIL all files created** |

### 2. 验证机制

v2: ❌ 无验证
v3: ✅ 三层验证
- Checkpoint 验证 (每个任务后)
- Stage 验证 (每个阶段后)
- Final 验证 (所有阶段后)

### 3. 强制完整性

v2: 建议遍历所有项
v3: 
- "Must trace **EVERY** sink"
- "For **EACH** pattern, run grep"
- "**ALL 4 TASKS** must complete"

### 4. 明确的失败处理

v3 新增:
```markdown
## Troubleshooting

**If you find yourself skipping stages:**
- STOP immediately
- Return to the last completed checkpoint
- Complete all mandatory tasks
```

---

## 📦 使用 v3

### 安装
```bash
cp -r code-audit-v3 ~/.claude/skills/code-audit
```

### 测试验证

对 Claude 说：
```
"使用 code-audit skill 审计 /path/to/blade-tool"
```

**预期行为 (v3):**

```
🔍 Starting code audit...

[Stage 0: Preparation]
✅ Detected framework: Spring Boot (Java)
✅ Reading rules/sinks/java.json...
✅ Loaded 52 sink patterns for Java
✅ Reading rules/sources/java.json...
✅ Loaded 31 source patterns for Java
✅ Workspace created

[Stage 1: Asset Discovery - ALL 4 TASKS MANDATORY]

[Task 1.1: Web Entry Discovery]
Searching for HTTP endpoints...
- Pattern: @GetMapping
- Pattern: @PostMapping
- Pattern: @RequestMapping
...
✅ Created web_entries.json with 23 endpoints

[Task 1.2: Sink Point Scanner]
Searching for dangerous sinks...
- Pattern: \.createNativeQuery\(
  Found: DatasourceServletAction.java:248
- Pattern: \.executeQuery\(
  Found: ...
...
✅ Created sink_points.json with 47 sinks

[Task 1.3: Security Asset Scanner]
...
✅ Created security_assets.json

[Task 1.4: Data Model Analyzer]
...
✅ Created data_models.json

[STAGE 1 COMPLETION CHECKPOINT]
✅ web_entries.json created - 23 entries
✅ sink_points.json created - 47 sinks (SQL:15, CMD:3, XXE:2, ...)
✅ security_assets.json created - 12 assets
✅ data_models.json created - 8 models

[1/7] ✅ Asset Discovery Complete

[Stage 2: Technical Vulnerability Audit]

[Stage 2.1: Backward Dataflow Tracing]
Processing sink SINK-001...
- Reading DatasourceServletAction.java:248
- Variable: sql
- Tracing backward...
  Step 1: line 206 - String sql = req.getParameter("sql")
  Source found: HTTP parameter (user input)
✅ Trace TRACE-001 complete

Processing sink SINK-002...
...

✅ Created traces.json with 47 traces

[Continuing through all stages...]
```

---

## 总结

### v2 的问题
- ❌ 建议性语言让 Claude 可以"优化"跳过步骤
- ❌ 没有强制读取规则文件
- ❌ 没有验证机制
- ❌ Claude 倾向于"快速出结果"而不是"完整执行"

### v3 的解决方案
- ✅ **MANDATORY** 标签强制执行
- ✅ **REQUIRED** 强制读取规则文件
- ✅ **VERIFICATION** 检查点验证
- ✅ **DO NOT PROCEED** 阻止跳过
- ✅ **FOR EACH** 强制完整遍历
- ✅ Checklist 格式强制确认

### 预期效果
v3 应该能让 Claude Code：
1. ✅ 读取并使用 rules/ 中的检测规则
2. ✅ 完整执行所有 7 个阶段
3. ✅ 为每个 sink 点都做污点追踪
4. ✅ 创建所有中间产物 JSON 文件
5. ✅ 生成完整的审计报告
