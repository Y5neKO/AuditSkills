# Task 工具使用指南

## Task 工具是什么？

在 Claude Code 环境中，`Task` 工具允许你创建"子任务"，这些子任务会在独立的上下文中执行，然后返回结果。

## 何时使用 Task？

### ✅ 适合用 Task 的场景

1. **可并行的独立任务**
   ```
   例如：同时扫描 HTTP 入口、Sink 点、安全资产、数据模型
   这 4 个任务互不依赖，可以并行执行
   ```

2. **重复性的模式匹配工作**
   ```
   例如：对 52 个 sink pattern 分别执行搜索
   每个 pattern 的搜索是独立的
   ```

3. **需要隔离上下文的任务**
   ```
   例如：分析一个大型文件时，可以将文件分段处理
   避免主上下文过载
   ```

### ❌ 不适合用 Task 的场景

1. **需要综合多个前序结果的任务**
   ```
   例如：漏洞验证需要综合 backward/forward traces、LSP 分析等
   这需要在主上下文中访问所有数据
   ```

2. **需要频繁交互的任务**
   ```
   例如：需要根据中间结果动态调整策略的分析
   Task 是"发出后等待结果"，不支持中途交互
   ```

3. **输出需要直接展示给用户的任务**
   ```
   例如：最终报告生成
   应该在主上下文中生成，以便直接展示
   ```

## v4 中的 Task 使用策略

### Stage 1: Asset Discovery - 4 个并行 Task

**为什么用 Task？**
- 4 个扫描任务完全独立
- 可以同时执行，节省时间
- 每个任务的输出是独立的 JSON 文件

```
Task 1: web_entry_scanner (扫描 HTTP 入口)
Task 2: sink_scanner (扫描危险函数)
Task 3: security_asset_scanner (扫描安全配置)
Task 4: data_model_analyzer (分析数据模型)

并行执行 ⏱️ → 快 4 倍
```

### Stage 2: Deep Analysis - 3 个并行 Task

**为什么用 Task？**
- LSP 分析、反向追踪、正向追踪可以并行
- 每个任务处理不同方面的分析

```
Task 1: lsp_code_analyzer (LSP 精确分析)
Task 2: backward_tracer (反向污点追踪)
Task 3: forward_tracer (正向污点追踪)

并行执行 ⏱️ → 快 3 倍
```

### Stage 3: Validation - 不使用 Task

**为什么不用 Task？**
- 需要综合 Stage 1 和 Stage 2 的所有结果
- 需要做复杂的交叉验证
- 需要根据验证结果动态生成 PoC

```
在主上下文中顺序执行：
1. 读取所有前序 JSON 文件
2. 综合分析
3. 逐个验证漏洞
4. 生成 PoC
5. 构建攻击链
```

## Task 的输入输出格式

### 输入：Prompt

Task 的 prompt 应该：
- 明确任务目标
- 提供所有必需的数据
- 指定输出格式和位置
- 包含验证要求

示例：

```
Task: sink_scanner
Prompt: |
  PROJECT_PATH: /path/to/project
  LANGUAGE: Java
  SINK_RULES: {
    "sinks": [
      {
        "name": "JPA Native Query",
        "patterns": ["\\.createNativeQuery\\(", "\\.createQuery\\("],
        "type": "sql_injection",
        "severity": "HIGH"
      }
    ]
  }
  
  任务：扫描所有危险函数调用点
  
  步骤：
  1. 遍历 SINK_RULES 中的每一个 sink
  2. 对每个 pattern 执行搜索
  3. 提取匹配的代码上下文
  
  输出到: .workspace/code-audit/phase1_discovery/sink_points.json
  
  必须输出: "✅ Found X sinks (SQL: A, XSS: B, ...)"
```

### 输出：结果

Task 完成后会返回：
- 文本输出（包括验证信息）
- 创建的文件
- 执行状态

主任务可以：
- 读取验证信息确认完成
- 读取生成的 JSON 文件获取详细结果
- 继续下一步

## Task 的错误处理

### 场景 1: Task 未完成所有工作

**检测方式：**
```bash
# 检查输出文件是否存在
if [ ! -f .workspace/code-audit/phase1_discovery/sink_points.json ]; then
  echo "❌ Task failed: sink_points.json not created"
fi

# 检查验证输出
if ! grep -q "✅ Found" task_output.txt; then
  echo "❌ Task did not complete verification"
fi
```

**处理方式：**
- 重新运行 Task
- 或者在主上下文中手动执行该任务

### 场景 2: Task 输出不完整

**检测方式：**
```bash
# 检查 JSON 文件内容
SINK_COUNT=$(jq '.statistics.total' .workspace/code-audit/phase1_discovery/sink_points.json)

if [ "$SINK_COUNT" -eq 0 ]; then
  echo "⚠️  Warning: No sinks found - may be incomplete"
fi
```

**处理方式：**
- 读取 JSON 文件检查统计信息
- 与预期的规则数量对比

## 完整示例：并行扫描

```
# Stage 1: 启动 4 个并行 Task

启动 Task 1...
启动 Task 2...
启动 Task 3...
启动 Task 4...

等待所有 Task 完成...

Task 1 返回: "✅ Found 45 HTTP endpoints"
Task 2 返回: "✅ Found 47 sinks (SQL: 15, XSS: 8, ...)"
Task 3 返回: "✅ Found 12 security assets"
Task 4 返回: "✅ Analyzed 8 data models"

验证所有输出文件：
✅ web_entries.json exists
✅ sink_points.json exists
✅ security_assets.json exists
✅ data_models.json exists

继续 Stage 2...
```

## Task vs 直接执行对比

### 场景：扫描 47 个 sink point

**直接执行（顺序）：**
```
扫描 SINK-001... (2s)
扫描 SINK-002... (2s)
扫描 SINK-003... (2s)
...
扫描 SINK-047... (2s)

总计：94 秒
```

**使用 Task（如果能并行处理）：**
```
启动 Task: 批量扫描所有 sinks...
等待...
完成！

总计：~10 秒（取决于并行能力）
```

## v4 的 Task 策略总结

| Stage | 使用 Task? | 原因 |
|-------|-----------|------|
| Stage 0: 准备 | ❌ No | 需要加载规则到主上下文 |
| Stage 1: 资产发现 | ✅ Yes (4个) | 独立任务，可并行 |
| Stage 2: 深度分析 | ✅ Yes (3个) | 独立分析，可并行 |
| Stage 3: 漏洞验证 | ❌ No | 需要综合所有数据 |
| Stage 4: 报告生成 | ❌ No | 需要展示给用户 |

## 最佳实践

1. **明确的输入输出**
   - Task prompt 包含所有需要的数据
   - 明确输出文件路径和格式
   - 包含验证字符串

2. **可验证的完成标志**
   - 要求 Task 输出确认信息
   - 创建可检查的文件
   - 包含统计信息

3. **错误恢复机制**
   - 检查 Task 输出
   - 验证生成的文件
   - 必要时在主上下文重做

4. **合理的粒度**
   - 不要为单个 grep 创建 Task（开销太大）
   - 但也不要一个 Task 做太多事（难以调试）
   - 找到平衡：4-7 个并行 Task 比较理想
