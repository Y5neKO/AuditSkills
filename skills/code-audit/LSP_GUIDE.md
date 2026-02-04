# LSP (Language Server Protocol) 集成指南

## 什么是 LSP？

LSP 是微软开发的协议，让编辑器/IDE 能够与语言服务器通信，获得：
- 精确的代码导航（Go to Definition）
- 类型信息（Hover）
- 引用查找（Find References）
- 符号定义（Document Symbols）

## 为什么需要 LSP？

### 传统方法（Grep）的局限性

```java
// 示例代码
String userId = req.getParameter("id");
processUser(userId);

void processUser(String id) {
    String query = "SELECT * FROM users WHERE id = " + id;
    db.execute(query);
}
```

**使用 Grep 追踪：**
```bash
# 1. 在 sink 点找到变量 "query"
grep -n "query" file.java
  Line 10: String query = "SELECT * FROM users WHERE id = " + id;
  Line 11: db.execute(query);

# 2. 追踪变量 "id"
grep -n "id" file.java
  Line 5: String userId = req.getParameter("id");
  Line 6: processUser(userId);
  Line 9: void processUser(String id) {
  Line 10: String query = "SELECT * FROM users WHERE id = " + id;
  ... 可能还有几十个其他 "id" 匹配
```

**问题：**
- ❌ 无法区分同名的不同变量
- ❌ 无法跨函数精确追踪
- ❌ 容易漏掉或误判

### 使用 LSP 的优势

**LSP 追踪：**
```
1. textDocument/definition on "query" at line 11
   → 精确定位到 line 10 的定义

2. textDocument/definition on "id" at line 10
   → 精确定位到函数参数 line 9

3. textDocument/references on "processUser"
   → 找到 line 6 的调用

4. textDocument/definition on "userId" at line 6
   → 精确定位到 line 5 的定义
```

**优势：**
- ✅ 100% 准确的变量追踪
- ✅ 自动处理作用域
- ✅ 支持跨文件追踪
- ✅ 提供类型信息

## 支持的语言服务器

### Java: Eclipse JDT Language Server (jdtls)

**启动命令：**
```bash
# 下载 jdtls
wget https://download.eclipse.org/jdtls/snapshots/jdt-language-server-latest.tar.gz
tar -xzf jdt-language-server-latest.tar.gz -C ~/.local/share/jdtls

# 启动 LSP 服务器
java -Declipse.application=org.eclipse.jdt.ls.core.id1 \
     -Dosgi.bundles.defaultStartLevel=4 \
     -Declipse.product=org.eclipse.jdt.ls.core.product \
     -Dlog.level=ALL \
     -jar ~/.local/share/jdtls/plugins/org.eclipse.equinox.launcher_*.jar \
     -configuration ~/.local/share/jdtls/config_linux \
     -data /tmp/jdtls-workspace &

LSP_PID=$!
```

**LSP 通信示例：**
```json
// 请求：找到变量定义
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/definition",
  "params": {
    "textDocument": {
      "uri": "file:///path/to/DatasourceServletAction.java"
    },
    "position": {
      "line": 247,
      "character": 30
    }
  }
}

// 响应：返回定义位置
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "uri": "file:///path/to/DatasourceServletAction.java",
    "range": {
      "start": {"line": 206, "character": 10},
      "end": {"line": 206, "character": 13}
    }
  }
}
```

### JavaScript/TypeScript: typescript-language-server

**启动命令：**
```bash
npm install -g typescript-language-server typescript

typescript-language-server --stdio &
LSP_PID=$!
```

### Python: Pyright

**启动命令：**
```bash
npm install -g pyright

pyright-langserver --stdio &
LSP_PID=$!
```

### PHP: Intelephense

**启动命令：**
```bash
npm install -g intelephense

intelephense --stdio &
LSP_PID=$!
```

## v4 中的 LSP 集成策略

### Task 2.1: LSP 辅助的代码分析

**流程：**

```
1. 检测项目语言
   ↓
2. 尝试启动对应的 LSP 服务器
   ↓
3. 如果启动成功：
   - 使用 LSP 进行精确分析
   - 对每个 sink 变量调用 textDocument/definition
   - 递归追踪到 source
   ↓
4. 如果启动失败：
   - 降级到 grep-based 分析
   - 记录 LSP 不可用
```

**实现示例（伪代码）：**

```python
def analyze_with_lsp(sink_points):
    # 1. 尝试启动 LSP
    lsp_server = None
    try:
        if language == "Java":
            lsp_server = start_jdtls()
        elif language == "JavaScript":
            lsp_server = start_typescript_ls()
        # ...
    except Exception as e:
        print(f"⚠️  LSP not available: {e}")
        print("Falling back to grep-based analysis")
        return analyze_with_grep(sink_points)
    
    # 2. 使用 LSP 分析
    variable_flows = []
    for sink in sink_points:
        # 读取 sink 所在的代码
        code = read_file(sink.file, sink.line)
        
        # 提取变量名（例如 "sql"）
        variable = extract_variable(code)
        
        # 使用 LSP 查找定义
        definition = lsp_server.find_definition(
            file=sink.file,
            line=sink.line,
            character=position_of(variable)
        )
        
        # 递归追踪
        chain = []
        current = definition
        while current:
            chain.append(current)
            
            # 检查是否是用户输入
            if is_user_input(current):
                break
            
            # 继续追踪上一级
            current = lsp_server.find_definition(
                file=current.file,
                line=current.line,
                character=position_of(current.variable)
            )
        
        variable_flows.append({
            "variable": variable,
            "sink": sink,
            "chain": chain
        })
    
    # 3. 关闭 LSP
    lsp_server.shutdown()
    
    return {
        "lsp_available": True,
        "lsp_type": "jdtls",
        "variable_flows": variable_flows
    }
```

### 降级策略

**如果 LSP 不可用，降级到增强的 Grep 分析：**

```python
def analyze_with_grep(sink_points):
    variable_flows = []
    
    for sink in sink_points:
        # 提取变量
        variable = extract_variable_from_sink(sink)
        
        # 使用 grep 查找所有定义
        definitions = grep_variable_definitions(
            variable=variable,
            file=sink.file
        )
        
        # 启发式选择最可能的定义
        # 1. 优先选择同一函数内的定义
        # 2. 检查作用域（简单的大括号匹配）
        # 3. 选择距离 sink 最近的定义
        likely_definition = select_likely_definition(
            definitions, 
            sink_line=sink.line
        )
        
        # 继续追踪
        chain = trace_variable_grep(likely_definition)
        
        variable_flows.append({
            "variable": variable,
            "sink": sink,
            "chain": chain,
            "confidence": "MEDIUM"  # Grep 的置信度较低
        })
    
    return {
        "lsp_available": False,
        "lsp_type": "grep-based",
        "variable_flows": variable_flows
    }
```

## LSP 的局限性

### 1. 需要项目构建配置

LSP 通常需要：
- Java: pom.xml / build.gradle
- JavaScript: package.json / tsconfig.json
- Python: setup.py / pyproject.toml

**解决方案：**
- 自动检测这些文件
- 如果不存在，使用 grep fallback

### 2. 初始化时间

LSP 服务器可能需要：
- Java: 10-30 秒（索引项目）
- JavaScript: 5-10 秒
- Python: 3-5 秒

**解决方案：**
- 在 Stage 0 启动 LSP，后台索引
- Stage 2 使用时已经就绪
- 设置超时（60秒），超时则降级

### 3. 资源消耗

LSP 服务器可能消耗：
- 内存: 500MB - 2GB
- CPU: 中等

**解决方案：**
- 审计完成后立即关闭 LSP
- 监控资源使用
- 大型项目考虑分批处理

## 输出格式

### LSP 可用时

```json
{
  "lsp_available": true,
  "lsp_type": "jdtls",
  "lsp_startup_time": "15.3s",
  "variable_flows": [
    {
      "sink_id": "SINK-001",
      "variable": "sql",
      "type": "java.lang.String",
      "definition_chain": [
        {
          "step": 1,
          "location": "DatasourceServletAction.java:206:10",
          "code": "String sql = req.getParameter(\"sql\")",
          "type": "VARIABLE_DECLARATION",
          "source_type": "HttpServletRequest.getParameter"
        },
        {
          "step": 2,
          "location": "DatasourceServletAction.java:248:30",
          "code": "jdbc.queryForList(sql, map)",
          "type": "USAGE_IN_SINK"
        }
      ],
      "confidence": "HIGH"
    }
  ],
  "statistics": {
    "total_traces": 47,
    "successful_traces": 47,
    "cross_file_traces": 12
  }
}
```

### LSP 不可用时

```json
{
  "lsp_available": false,
  "lsp_type": "grep-based",
  "fallback_reason": "jdtls not installed",
  "variable_flows": [
    {
      "sink_id": "SINK-001",
      "variable": "sql",
      "type": "unknown",
      "definition_chain": [
        {
          "step": 1,
          "location": "DatasourceServletAction.java:206",
          "code": "String sql = req.getParameter(\"sql\")",
          "type": "VARIABLE_DECLARATION",
          "detection_method": "grep + heuristic"
        },
        {
          "step": 2,
          "location": "DatasourceServletAction.java:248",
          "code": "jdbc.queryForList(sql, map)",
          "type": "USAGE_IN_SINK"
        }
      ],
      "confidence": "MEDIUM",
      "note": "Grep-based analysis may have lower accuracy"
    }
  ],
  "statistics": {
    "total_traces": 47,
    "successful_traces": 43,
    "ambiguous_traces": 4
  }
}
```

## 最佳实践

### 1. 优雅降级

```
优先级顺序：
1. LSP 分析（最准确）
2. Grep + 作用域分析（中等准确）
3. Grep + 距离启发式（基本准确）
```

### 2. 混合使用

```
即使 LSP 可用，也可以用 Grep 做初步筛选：
1. Grep 快速找到所有可能的 sink
2. LSP 精确追踪每个 sink 的数据流
```

### 3. 验证 LSP 结果

```
对于关键漏洞，即使 LSP 给出了结果，也应该：
1. 手动读取相关代码片段
2. 确认追踪链是合理的
3. 在报告中注明是 LSP 辅助分析
```

### 4. 处理 LSP 超时

```python
import timeout

try:
    with timeout(60):  # 60 秒超时
        result = lsp_server.find_definition(...)
except TimeoutError:
    print("⚠️  LSP timeout, falling back to grep")
    result = grep_find_definition(...)
```

## 集成到 v4 Workflow

```
Stage 0:
  - 检测语言
  - 启动 LSP 服务器（后台）
  - 开始索引项目
  ↓
Stage 1:
  - 扫描 sinks/sources（不需要 LSP）
  ↓
Stage 2:
  Task 2.1: LSP Analysis
    - LSP 此时应该已经就绪
    - 使用 LSP 精确追踪
    - 如果仍在索引，等待或降级
  ↓
Stage 3:
  - 综合 LSP 结果和其他分析
  ↓
Stage 4:
  - 生成报告
  - 关闭 LSP 服务器
```

## 总结

| 特性 | LSP | Grep |
|------|-----|------|
| 准确性 | 极高 | 中等 |
| 跨文件追踪 | ✅ 完美 | ⚠️ 困难 |
| 类型信息 | ✅ 提供 | ❌ 无 |
| 设置复杂度 | 中等 | 简单 |
| 资源消耗 | 较高 | 很低 |
| 速度（首次） | 慢（需索引）| 快 |
| 速度（后续） | 快 | 快 |

**v4 策略：优先 LSP，降级 Grep，混合验证**
