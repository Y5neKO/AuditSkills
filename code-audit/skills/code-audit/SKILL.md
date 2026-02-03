---
name: code-audit
description: AI代码安全审计工具。从HTTP入口出发进行污点分析，自动发现Web漏洞，生成完整调用链和PoC。支持正向分析（从source到sink）和反向分析（从sink到source），可组合多个漏洞构建攻击链。
---

# 代码安全审计工具 (Code Audit)

## 概述

本工具通过**污点分析 (Taint Analysis)** 技术自动审计Web应用代码，发现安全漏洞并生成可验证的PoC。

### 核心特性

1. **双向分析模式**
   - 正向分析：从HTTP入口(source)追踪数据流到危险函数(sink)
   - 反向分析：从危险函数(sink)反向追溯数据来源(source)

2. **完整调用链追踪**
   - 利用LSP进行代码跳转和符号解析
   - 记录完整的数据传播路径
   - 识别中间过滤/净化点

3. **可验证漏洞报告**
   - 每个漏洞包含完整source→sink调用链
   - 自动生成HTTP请求PoC
   - 支持攻击链编排组合多个漏洞

4. **多框架支持**
   - 自动检测Web框架（Express, Flask, Django, Spring等）
   - 框架特定的source点和sink点规则

---

## 支持资源

本技能依赖以下支持文件：

- **正向审计Agent** - 从HTTP入口追踪到危险函数，见 [agents/forward-auditor.md](agents/forward-auditor.md)
- **反向审计Agent** - 从危险函数追溯数据来源，见 [agents/backward-auditor.md](agents/backward-auditor.md)
- **PoC生成器** - 生成漏洞验证代码，见 [agents/poc-generator.md](agents/poc-generator.md)
- **报告生成器** - 生成审计报告，见 [agents/report-generator.md](agents/report-generator.md)
- **攻击链编排器** - 组合漏洞构建攻击链，见 [agents/attack-chain-orchestrator.md](agents/attack-chain-orchestrator.md)
- **Source点规则** - HTTP入口点检测规则，见 [rules/sources/](rules/sources/)
- **Sink点规则** - 危险函数检测规则，见 [rules/sinks/](rules/sinks/)

## 快速开始

### 基本用法

```
/code-audit analyze --project /path/to/project
```

### 指定框架

```
/code-audit analyze --project /path/to/project --framework express
```

### 正向审计（推荐 - 从用户入口追踪）

```
/code-audit analyze --project /path/to/project --mode forward
```

### 反向审计（从危险函数回溯）

```
/code-audit analyze --project /path/to/project --mode backward
```

### 审计特定漏洞类型

```
/code-audit analyze --project /path/to/project --vuln-type sqli,ssrf,xss
```

---

## 工作流程

### 阶段1: 项目分析

```
┌─────────────────────────────────────────────────────────────┐
│                     项目初始化与框架检测                        │
├─────────────────────────────────────────────────────────────┤
│ 1. 扫描项目结构，识别编程语言和Web框架                        │
│ 2. 定位HTTP入口点（路由、控制器）                            │
│ 3. 构建代码索引（函数/类/方法的符号表）                       │
│ 4. 加载对应的source/sink规则库                               │
└─────────────────────────────────────────────────────────────┘
```

### 阶段2: 污点分析

```
┌─────────────────────────────────────────────────────────────┐
│                        污点分析执行                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  正向模式                     反向模式                        │
│    ↓                           ↓                            │
│ HTTP请求参数              危险函数(sink)                      │
│    ↓                           ↑                            │
│ 追踪数据流                 反向追溯来源                        │
│    ↓                           ↑                            │
│ 检查过滤点                 检查过滤点                         │
│    ↓                           ↑                            │
│ 到达sink?                 能到达HTTP?                        │
│    ↓                           ↑                            │
│ 记录调用链                 记录调用链                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 阶段3: 漏洞验证与报告生成

```
┌─────────────────────────────────────────────────────────────┐
│                    漏洞验证与PoC生成                          │
├─────────────────────────────────────────────────────────────┤
│ 1. 对发现的潜在漏洞进行调用链验证                            │
│ 2. 检查中间是否存在有效过滤                                  │
│ 3. 评估可利用性和严重程度                                    │
│ 4. 生成HTTP请求PoC                                          │
│ 5. 生成结构化漏洞报告                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 漏洞检测规则

### 支持的漏洞类型

| 漏洞类型 | 描述 | Source点 | Sink点示例 |
|---------|------|----------|-----------|
| SQL注入 | 用户输入进入SQL查询 | GET/POST参数 | `query()`, `execute()` |
| XSS | 用户输入输出到HTML | GET/POST参数 | `innerHTML`, `res.send()` |
| SSRF | 用户输入用于URL请求 | GET/POST参数 | `fetch()`, `http.get()` |
| 命令注入 | 用户输入进入系统命令 | GET/POST参数 | `exec()`, `spawn()` |
| 路径遍历 | 用户输入用于文件路径 | GET/POST参数 | `fs.readFile()` |
| 反序列化 | 用户数据反序列化 | Cookie/Body | `deserialize()`, `pickle.load()` |
| SSTI | 用户输入进入模板 | GET/POST参数 | `render_template()` |
| 代码注入 | 用户输入作为代码执行 | GET/POST参数 | `eval()`, `Function()` |

### Web框架检测与规则加载

系统会自动检测以下框架并加载对应规则：

**Node.js/JavaScript**
- Express.js
- Koa.js
- NestJS
- Fastify
- Hapi

**Python**
- Flask
- Django
- FastAPI
- Tornado

**Java**
- Spring Boot
- Spring MVC
- Struts

**PHP**
- Laravel
- Symfony
- CodeIgniter

---

## 污点分析详解

### Source点 (数据来源)

Source点是用户可控的输入点，常见包括：

```
HTTP请求相关:
  - req.query / req.params / req.body
  - request.GET / request.POST / request.FILES
  - $_GET / $_POST / $_REQUEST
  - @RequestParam / @RequestBody

HTTP Header相关:
  - req.headers
  - request.META
  - getallheaders()

Cookie相关:
  - req.cookies
  - request.COOKIES
  - $_COOKIE

文件上传:
  - req.files
  - request.FILES
  - $_FILES
```

### Sink点 (危险函数)

Sink点是可能导致安全问题的危险函数，常见包括：

```
SQL注入sink:
  - query() / execute() / raw()
  - cursor.execute()
  - $db->query()

命令执行sink:
  - exec() / spawn() / system()
  - subprocess.call()
  - Runtime.exec()

文件操作sink:
  - fs.readFile() / fs.writeFile()
  - open() / file()
  - file_get_contents()

模板渲染sink:
  - render_template()
  - engine.render()
  - @Template.render()
```

### 过滤/净化点 (Sanitizers)

过滤点是可能清除污点的函数，包括：

```
输入验证:
  - validator.escape()
  - htmlspecialchars()
  - bleach.clean()

参数化查询:
  - prepared statements
  - parameterized queries

类型转换:
  - parseInt()
  - parseFloat()
  - str()
```

---

## 调用链分析示例

### 示例1: SQL注入漏洞

```javascript
// app.js - Express应用
app.get('/user/:id', async (req, res) => {
  // Source: req.params.id
  const userId = req.params.id;

  // 无过滤，直接拼接SQL
  const query = `SELECT * FROM users WHERE id = ${userId}`;

  // Sink: db.query (SQL注入点)
  const result = await db.query(query);
  res.json(result);
});
```

**调用链分析:**
```
HTTP GET /user/:id
  ↓
req.params.id [SOURCE - 用户可控]
  ↓
变量 query = `SELECT * FROM users WHERE id = ${userId}`
  ↓
db.query(query) [SINK - SQL执行]
  ↓
[漏洞] SQL注入
```

**生成PoC:**
```http
GET /user/1' OR '1'='1 HTTP/1.1
Host: example.com
```

---

### 示例2: 带过滤的误报排除

```python
# app.py - Flask应用
@app.route('/search')
def search():
    # Source: request.args.get('q')
    query = request.args.get('q', '')

    # 过滤: 转义特殊字符
    from html import escape
    safe_query = escape(query)

    # Sink: cursor.execute (使用参数化查询)
    cursor.execute("SELECT * FROM products WHERE name LIKE %s",
                  (f'%{safe_query}%',))
    return jsonify(cursor.fetchall())
```

**调用链分析:**
```
HTTP GET /search?q=test
  ↓
request.args.get('q') [SOURCE]
  ↓
escape(query) [SANITIZER - 清除污点]
  ↓
cursor.execute(..., (safe_query,)) [SINK - 参数化查询]
  ↓
[安全] 无漏洞
```

---

## 报告格式

### 漏洞报告结构

```json
{
  "audit_metadata": {
    "timestamp": "2025-01-15T10:30:00Z",
    "project_path": "/path/to/project",
    "framework": "Express.js",
    "language": "JavaScript",
    "audit_mode": "forward"
  },
  "vulnerabilities": [
    {
      "vuln_id": "VULN-001",
      "vuln_type": "sqli",
      "severity": "CRITICAL",
      "confidence": "HIGH",
      "title": "SQL注入漏洞 - /api/users/:id",

      "entry_point": {
        "method": "GET",
        "path": "/api/users/:id",
        "file": "routes/users.js",
        "line": 45
      },

      "call_chain": [
        {
          "location": "routes/users.js:45",
          "code": "const userId = req.params.id",
          "type": "SOURCE",
          "description": "HTTP路径参数，用户可控"
        },
        {
          "location": "routes/users.js:47",
          "code": "const query = `SELECT * FROM users WHERE id = ${userId}`",
          "type": "PROPAGATION",
          "description": "污点传播，直接拼接SQL"
        },
        {
          "location": "routes/users.js:49",
          "code": "await db.query(query)",
          "type": "SINK",
          "description": "SQL执行，存在注入点"
        }
      ],

      "sanitizers_checked": [
        {
          "function": "escape",
          "present": false,
          "bypass_possible": false
        }
      ],

      "poc": {
        "description": "通过注入恶意SQL语句绕过认证或提取数据",
        "requests": [
          {
            "method": "GET",
            "url": "http://localhost:3000/api/users/1' OR '1'='1",
            "headers": {
              "User-Agent": "Mozilla/5.0"
            }
          }
        ],
        "expected_behavior": "返回所有用户数据",
        "verification_method": "检查响应中是否包含多个用户记录"
      },

      "remediation": {
        "recommendation": "使用参数化查询",
        "code_example": "await db.query('SELECT * FROM users WHERE id = ?', [userId])"
      },

      "references": [
        "CWE-89: SQL Injection",
        "OWASP Top 10 2021: A03:2021 – Injection"
      ]
    }
  ],

  "statistics": {
    "total_vulnerabilities": 5,
    "by_severity": {
      "CRITICAL": 1,
      "HIGH": 2,
      "MEDIUM": 1,
      "LOW": 1
    },
    "by_type": {
      "sqli": 1,
      "xss": 2,
      "ssrf": 1,
      "path_traversal": 1
    }
  }
}
```

---

## 攻击链编排

本工具支持组合多个漏洞构建完整攻击链：

### 示例攻击链

```
[1] 存储型XSS (低危用户)
    ↓
[2] IDOR获取管理员Token
    ↓
[3] SSRF访问内部元数据
    ↓
[4] RCE获取服务器控制权
```

### 攻击链模式

工具会自动识别可串联的漏洞：

1. **认证绕过链**: 未授权访问 → 越权操作 → 权限提升
2. **数据注入链**: XSS → CSRF → 敏感操作
3. **服务端链**: SQL注入 → 文件写入 → 代码执行

---

## Agent架构

本工具使用多个协作Agent完成审计任务：

| Agent | 职责 | 工具 |
|-------|------|------|
| `project-analyzer` | 项目分析、框架检测 | Glob, Grep, Read |
| `forward-auditor` | 正向污点分析 | Grep, Read, 符号解析 |
| `backward-auditor` | 反向污点分析 | Grep, Read, 符号解析 |
| `call-chain-tracer` | 调用链追踪与验证 | Read, Grep |
| `poc-generator` | PoC生成 | Write |
| `report-generator` | 报告生成 | Write |

---

## 使用规则库

规则库位于 `rules/` 目录：

```
rules/
├── sources/
│   ├── javascript.json
│   ├── python.json
│   ├── java.json
│   └── php.json
├── sinks/
│   ├── javascript.json
│   ├── python.json
│   ├── java.json
│   └── php.json
└── sanitizers/
    ├── javascript.json
    ├── python.json
    ├── java.json
    └── php.json
```

### 规则文件格式

```json
{
  "language": "JavaScript",
  "framework": "Express",
  "sources": [
    {
      "name": "express_request_body",
      "pattern": "req\\.body",
      "type": "http_body",
      "description": "Express HTTP请求体"
    },
    {
      "name": "express_request_query",
      "pattern": "req\\.query",
      "type": "http_query",
      "description": "Express URL查询参数"
    }
  ],
  "sinks": [
    {
      "name": "sql_query",
      "pattern": "(?:query|execute)\\(",
      "type": "sqli",
      "description": "SQL查询执行",
      "dangerous_param": 0
    }
  ],
  "sanitizers": [
    {
      "name": "mysql_escape",
      "pattern": "mysql\\.escape\\(",
      "type": "sql_sanitizer",
      "description": "MySQL转义函数"
    }
  ]
}
```

---

## 最佳实践

### 1. 从正向审计开始

正向审计更符合真实攻击场景，优先从HTTP入口追踪。

### 2. 双向验证

对可疑漏洞，使用反向审计验证数据来源。

### 3. 检查过滤点

工具会自动检测调用链中的过滤点，但仍需人工复核。

### 4. PoC验证

报告中的PoC应实际测试验证。

### 5. 框架特定规则

使用 `--framework` 参数指定框架可提高准确性。

---

## 限制与注意事项

1. **静态分析局限**: 无法检测运行时生成的代码
2. **误报可能**: 复杂的数据流分析可能产生误报
3. **需要人工复核**: 所有发现应由安全专家复核
4. **框架覆盖**: 仅支持常见Web框架
5. **语言支持**: 主要支持JavaScript/Python/Java/PHP

---

## 开发扩展

### 添加新框架支持

1. 在 `rules/` 创建对应语言和框架的规则文件
2. 定义source、sink和sanitizer规则
3. 更新框架检测逻辑

### 添加新漏洞类型

1. 在 `rules/sinks/` 中定义新的sink点
2. 在 `utils/poc-generator.md` 中添加PoC模板
3. 更新报告模板

---

## 输出文件

审计完成后生成以下文件：

```
.workspace/
├── code-audit/
│   ├── report.json          # 完整JSON报告
│   ├── report.md            # Markdown报告
│   ├── pocs/                # PoC脚本
│   │   ├── vuln-001.py
│   │   ├── vuln-002.py
│   │   └── ...
│   └── attack-chains/       # 攻击链报告
│       ├── chain-001.json
│       └── ...
```
