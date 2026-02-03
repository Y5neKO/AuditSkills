---
name: code-audit
description: AI代码安全审计工具。从HTTP入口出发进行污点分析，自动发现Web漏洞，生成完整调用链和PoC。支持正向分析（从source到sink）和反向分析（从sink到source），可组合多个漏洞构建攻击链。使用 /code-audit 或当用户请求代码审计、漏洞扫描、安全分析时激活。
argument-hint: [project-path] [options]
context: fork
agent: general-purpose
allowed-tools: Read, Grep, Glob, Write, Bash
---

# Code Audit - 代码安全审计工具

通过**污点分析 (Taint Analysis)** 技术自动审计Web应用代码，发现安全漏洞并生成可验证的PoC。

## 快速开始

### 基本用法

```
/code-audit /path/to/project
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

## 审计流程

```
┌─────────────────────────────────────────────────────────────┐
│                     代码审计工作流                             │
├─────────────────────────────────────────────────────────────┤
│ 1. 项目分析                                                   │
│    ├─ 检测编程语言和Web框架                                   │
│    ├─ 定位HTTP入口点（路由、控制器）                           │
│    └─ 加载对应的source/sink规则库                             │
│                                                              │
│ 2. 污点分析                                                   │
│    ├─ 正向: HTTP入口 → 追踪数据流 → 危险函数                   │
│    └─ 反向: 危险函数 → 反向追溯 → HTTP入口                     │
│                                                              │
│ 3. 漏洞验证                                                   │
│    ├─ 检查调用链完整性                                        │
│    ├─ 检查过滤/净化点                                         │
│    └─ 评估可利用性                                            │
│                                                              │
│ 4. 报告生成                                                   │
│    ├─ 生成完整调用链                                          │
│    ├─ 生成HTTP请求PoC                                         │
│    └─ 生成结构化漏洞报告                                      │
└─────────────────────────────────────────────────────────────┘
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

### Sanitizer（净化点）

清除污点的函数，如：
- 参数化查询
- 转义函数：`escape()`, `htmlspecialchars()`
- 类型转换：`parseInt()`

## 调用链示例

### SQL注入漏洞

```javascript
app.get('/user/:id', async (req, res) => {
  const userId = req.params.id;              // SOURCE
  const query = `SELECT * FROM users WHERE id = ${userId}`;  // PROPAGATION
  const result = await db.query(query);      // SINK
  res.json(result);
});
```

**PoC**:
```http
GET /user/1' OR '1'='1 HTTP/1.1
Host: example.com
```

## 子代理

本技能使用专门的子代理完成不同任务：

| 子代理 | 职责 | 配置文件 |
|-------|------|---------|
| forward-auditor | 正向污点分析 | [agents/forward-auditor/agent.md](agents/forward-auditor/agent.md) |
| backward-auditor | 反向污点分析 | [agents/backward-auditor/agent.md](agents/backward-auditor/agent.md) |
| poc-generator | PoC生成 | [agents/poc-generator/agent.md](agents/poc-generator/agent.md) |
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
├── pocs/                # PoC脚本
└── attack-chains/       # 攻击链报告
```

### 报告结构

```json
{
  "audit_metadata": {
    "timestamp": "2025-01-15T10:30:00Z",
    "project_path": "/path/to/project",
    "framework": "Express.js",
    "language": "JavaScript"
  },
  "vulnerabilities": [
    {
      "vuln_id": "VULN-001",
      "vuln_type": "sqli",
      "severity": "CRITICAL",
      "entry_point": {
        "method": "GET",
        "path": "/api/users/:id"
      },
      "call_chain": [...],
      "poc": {...}
    }
  ]
}
```

## 执行审计

当用户请求审计时：

1. 分析项目结构和框架
2. 加载对应语言的source/sink规则
3. 执行污点分析（正向或反向）
4. 验证发现的漏洞
5. 生成报告和PoC

## 参考资源

- 完整正向审计流程：[agents/forward-auditor/agent.md](agents/forward-auditor/agent.md)
- 完整反向审计流程：[agents/backward-auditor/agent.md](agents/backward-auditor/agent.md)
- 输出示例：[examples/sample-report.json](examples/sample-report.json)
