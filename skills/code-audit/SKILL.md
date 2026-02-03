---
name: code-audit
description: AI代码安全审计工具。从HTTP入口出发进行污点分析，自动发现Web漏洞，生成完整调用链和PoC。支持正向分析（从source到sink）和反向分析（从sink到source），可组合多个漏洞构建攻击链。增强版：考虑全局过滤器、库版本影响和框架配置的安全上下文感知分析。使用 /code-audit 或当用户请求代码审计、漏洞扫描、安全分析时激活。
argument-hint: [project-path] [options]
context: fork
agent: general-purpose
allowed-tools: Read, Grep, Glob, Write, Bash, Task
---

# Code Audit - 代码安全审计工具

通过**增强型污点分析 (Enhanced Taint Analysis)** 技术自动审计Web应用代码，发现安全漏洞并生成可验证的PoC。

## 核心特性

1. **安全上下文感知分析** - 考虑全局过滤器、框架配置和库版本的影响
2. **双向污点分析** - 正向（source→sink）和反向（sink→source）
3. **完整调用链追踪** - 记录完整的数据传播路径
4. **版本感知分析** - 根据库版本评估风险
5. **绕过检测** - 识别过滤器的绕过方法

## 快速开始

### 基本用法（增强模式）

```
/code-audit /path/to/project
```

### 标准模式（不考虑全局过滤）

```
/code-audit /path/to/project --mode standard
```

### 指定审计模式

```
/code-audit /path/to/project --mode forward   # 正向：从入口追踪
/code-audit /path/to/project --mode backward  # 反向：从sink回溯
```

### 指定框架

```
/code-audit /path/to/project --framework express
```

## 增强型审计流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        增强型代码安全审计流程                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  阶段 1: 安全上下文分析 (并行)                                                  │
│  ├─ [1.1] 框架配置分析 ────────────────┐                                     │
│  │   - 配置文件定位                    │                                     │
│  │   - 安全设置分析                    │                                     │
│  │   - 配置安全评分                    │                                     │
│  ├────────────────────────────────────┤                                     │
│  ├─ [1.2] 库版本分析 ─────────────────┤                                     │
│  │   - 依赖文件解析                    │                                     │
│  │   - 已知漏洞检测                    │                                     │
│  │   - 版本特性映射                    │                                     │
│  ├────────────────────────────────────┤                                     │
│  └─ [1.3] 全局过滤分析 ───────────────┘                                     │
│      - 中间件/拦截器检测                                                │
│      - 过滤有效性评估                                                    │
│      - 绕过方法检测                                                      │
│           ↓                                                                  │
│  阶段 2: 数据整合                                                               │
│  └─ 构建完整安全上下文模型                                                      │
│           ↓                                                                  │
│  阶段 3: 增强型污点分析                                                          │
│  ├─ Source点识别 (考虑全局过滤)                                               │
│  ├─ 污点追踪 (记录所有过滤点)                                                  │
│  ├─ Sink点检测 (考虑版本影响)                                                 │
│  └─ 漏洞判定 (综合评估)                                                        │
│           ↓                                                                  │
│  阶段 4: 报告生成                                                               │
│  └─ 生成带绕过方法的PoC                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 为什么需要增强型分析

### 传统污点分析的盲点

```javascript
// 全局过滤器存在
app.use((req, res, next) => {
  if (req.params.id) {
    req.params.id = parseInt(req.params.id);  // 全局类型转换
  }
  next();
});

// 路由处理
app.get('/user/:id', (req, res) => {
  const userId = req.params.id;  // SOURCE - 已经被转换
  const query = `SELECT * FROM users WHERE id = ${userId}`;  // SINK
  db.query(query);
});
```

**传统分析**: 发现 SQL 注入漏洞
**增强分析**: 识别出全局 `parseInt()` 过滤，但标注绕过方法（大数值注入）

### 库版本影响

```python
# Jinja2 < 3.0.0 - 沙箱弱，SSTI 高危
# Jinja2 >= 3.0.0 - 沙箱增强，但仍需验证
@app.route('/template')
def render():
    template = request.args.get('tpl')
    return render_template_string(template)  # 风险级别因版本而异
```

## 支持的漏洞类型

| 漏洞类型 | 描述 | CWE |
|---------|------|-----|
| SQL注入 | 用户输入进入SQL查询 | CWE-89 |
| XSS | 用户输入输出到HTML | CWE-79 |
| SSRF | 用户输入用于URL请求 | CWE-918 |
| 命令注入 | 用户输入进入系统命令 | CWE-78 |
| 路径遍历 | 用户输入用于文件路径 | CWE-22 |
| 反序列化 | 用户数据反序列化 | CWE-502 |
| SSTI | 用户输入进入模板 | CWE-94 |

## 支持的框架

**Node.js/JavaScript**: Express, Koa, NestJS, Fastify, Hapi
**Python**: Flask, Django, FastAPI, Tornado
**Java**: Spring Boot, Spring MVC, Struts
**PHP**: Laravel, Symfony, CodeIgniter

## 核心概念

### Source点（数据来源）

用户可控的输入点，如：
- HTTP参数：`req.params`, `req.query`, `req.body`
- HTTP头：`req.headers`
- Cookie：`req.cookies`
- 文件上传：`req.files`

### Sink点（危险函数）

可能导致安全问题的函数，如：
- SQL执行：`query()`, `execute()`
- 命令执行：`exec()`, `spawn()`
- 文件操作：`fs.readFile()`
- 模板渲染：`render_template()`

### 过滤器类型

1. **局部过滤器**: 代码中的过滤函数（如 `escape()`）
2. **全局过滤器**: 中间件、拦截器级别的过滤
3. **框架防护**: 模板引擎自动转义、ORM参数化等

## 子代理

本技能使用专门的子代理完成不同任务：

### 核心分析代理

| 子代理 | 职责 | 配置文件 |
|-------|------|---------|
| enhanced-audit-orchestrator | 增强型审计协调器 | [agents/enhanced-audit-orchestrator/agent.md](agents/enhanced-audit-orchestrator/agent.md) |
| forward-auditor | 正向污点分析 | [agents/forward-auditor/agent.md](agents/forward-auditor/agent.md) |
| backward-auditor | 反向污点分析 | [agents/backward-auditor/agent.md](agents/backward-auditor/agent.md) |

### 上下文分析代理

| 子代理 | 职责 | 配置文件 |
|-------|------|---------|
| security-context-analyzer | 安全上下文分析（全局过滤器、中间件） | [agents/security-context-analyzer/agent.md](agents/security-context-analyzer/agent.md) |
| library-version-analyzer | 库版本分析（已知漏洞、版本特性） | [agents/library-version-analyzer/agent.md](agents/library-version-analyzer/agent.md) |
| framework-config-analyzer | 框架配置分析（安全设置） | [agents/framework-config-analyzer/agent.md](agents/framework-config-analyzer/agent.md) |

### 报告生成代理

| 子代理 | 职责 | 配置文件 |
|-------|------|---------|
| poc-generator | PoC生成（含绕过方法） | [agents/poc-generator/agent.md](agents/poc-generator/agent.md) |
| report-generator | 报告生成 | [agents/report-generator/agent.md](agents/report-generator/agent.md) |
| attack-chain-orchestrator | 攻击链编排 | [agents/attack-chain-orchestrator/agent.md](agents/attack-chain-orchestrator/agent.md) |

## 规则库

### Source点规则

各语言HTTP入口点检测规则，见 [rules/sources/](rules/sources/)

### Sink点规则

各语言危险函数检测规则，见 [rules/sinks/](rules/sinks/)

## 输出报告

审计完成后生成：

```
.workspace/code-audit/
├── report.json          # 完整JSON报告
├── report.md            # Markdown报告
├── security-context.json # 安全上下文报告
├── library-version.json  # 库版本报告
├── framework-config.json # 框架配置报告
├── pocs/                # PoC脚本（含绕过）
└── attack-chains/       # 攻击链报告
```

### 增强型报告结构

```json
{
  "audit_metadata": {
    "timestamp": "2025-01-15T10:30:00Z",
    "project_path": "/path/to/project",
    "framework": "Express.js",
    "language": "JavaScript",
    "audit_mode": "enhanced_forward"
  },
  "security_context": {
    "global_filters": [...],
    "framework_features": {...},
    "library_versions": {...},
    "framework_config": {...}
  },
  "vulnerabilities": [
    {
      "vuln_id": "VULN-001",
      "vuln_type": "sqli",
      "base_severity": "CRITICAL",
      "adjusted_severity": "MEDIUM",
      "confidence": "MEDIUM",

      "call_chain": [
        {
          "location": "routes/users.js:10",
          "type": "SOURCE",
          "global_filters_applied": ["parseInt()"]
        },
        {
          "location": "routes/users.js:12",
          "type": "SINK"
        }
      ],

      "security_context": {
        "global_filters_effective": "PARTIAL",
        "bypass_methods": ["large_number_injection"],
        "framework_protection": "NONE",
        "version_impact": "NONE"
      },

      "poc": {
        "original": "GET /user/1' OR '1'='1",
        "bypass": "GET /user/99999999999999999",
        "bypass_explanation": "大数值可能导致整数溢出"
      }
    }
  ]
}
```

## 执行审计

当用户请求审计时：

1. **安全上下文分析**（并行）:
   - 框架配置分析
   - 库版本分析
   - 全局过滤器分析

2. **数据整合**:
   - 合并所有上下文信息
   - 构建安全上下文模型

3. **增强型污点分析**:
   - Source点识别（考虑全局过滤）
   - 污点追踪（记录所有过滤点）
   - Sink点检测（考虑版本影响）
   - 漏洞判定（综合评估）

4. **报告生成**:
   - 生成带绕过方法的PoC
   - 生成结构化报告

## 参考资源

- 增强型审计协调器：[agents/enhanced-audit-orchestrator/agent.md](agents/enhanced-audit-orchestrator/agent.md)
- 安全上下文分析：[agents/security-context-analyzer/agent.md](agents/security-context-analyzer/agent.md)
- 库版本分析：[agents/library-version-analyzer/agent.md](agents/library-version-analyzer/agent.md)
- 框架配置分析：[agents/framework-config-analyzer/agent.md](agents/framework-config-analyzer/agent.md)
- 输出示例：[examples/sample-report.json](examples/sample-report.json)
