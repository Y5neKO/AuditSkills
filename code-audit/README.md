# Code Audit Skill

> AI代码安全审计工具 - 从HTTP入口出发进行污点分析，自动发现Web漏洞，生成完整调用链和PoC

## 概述

Code Audit Skill是一个基于Claude AI的代码安全审计工具，使用**污点分析 (Taint Analysis)** 技术自动审计Web应用代码，发现安全漏洞并生成可验证的概念验证代码(PoC)。

### 核心特性

- ✅ **双向分析模式** - 正向分析(从source到sink)和反向分析(从sink到source)
- ✅ **完整调用链追踪** - 记录完整的数据传播路径
- ✅ **多语言支持** - JavaScript、Python、Java、PHP
- ✅ **多框架支持** - Express、Flask、Django、Spring、Laravel等
- ✅ **可验证PoC** - 自动生成HTTP请求PoC
- ✅ **攻击链编排** - 组合多个漏洞构建完整攻击场景

## 快速开始

### 安装

```bash
# 将此目录复制到Claude插件目录
cp -r code-audit ~/.claude/plugins/

# 或使用符号链接
ln -s /path/to/code-audit ~/.claude/plugins/code-audit
```

### 基本使用

```
/code-audit analyze --project /path/to/project
```

### 指定框架

```
/code-audit analyze --project /path/to/project --framework express
```

### 正向审计（推荐）

```
/code-audit analyze --project /path/to/project --mode forward
```

### 反向审计

```
/code-audit analyze --project /path/to/project --mode backward
```

## 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Code Audit Skill                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐     ┌──────────────┐                     │
│  │ 正向审计Agent │     │ 反向审计Agent │                     │
│  │  (Source→Sink)│     │  (Sink→Source)│                    │
│  └──────┬───────┘     └──────┬───────┘                     │
│         │                     │                              │
│         └──────────┬──────────┘                              │
│                    ↓                                         │
│         ┌──────────────────┐                                │
│         │  漏洞分析结果     │                                │
│         └────────┬─────────┘                                │
│                  ↓                                          │
│    ┌─────────────┼─────────────┐                            │
│    ↓             ↓             ↓                            │
│ ┌──────┐    ┌──────┐    ┌──────────┐                        │
│ │PoC生成│    │报告生成│    │攻击链编排│                        │
│ └──────┘    └──────┘    └──────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 目录结构

```
code-audit/
├── .claude-plugin/
│   └── plugin.json          # 插件配置
├── skills/
│   └── code-audit/
│       └── SKILL.md          # 主Skill文件
├── README.md                 # 本文件
├── rules/                    # 规则库
│   ├── sources/              # Source点规则
│   │   ├── javascript.json
│   │   ├── python.json
│   │   ├── java.json
│   │   └── php.json
│   └── sinks/                # Sink点规则
│       ├── javascript.json
│       ├── python.json
│       ├── java.json
│       └── php.json
└── agents/                   # Agent配置
    ├── forward-auditor.md    # 正向审计Agent
    ├── backward-auditor.md   # 反向审计Agent
    ├── poc-generator.md      # PoC生成器
    ├── report-generator.md   # 报告生成器
    └── attack-chain-orchestrator.md  # 攻击链编排器
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
| 代码注入 | 用户输入作为代码执行 | CWE-94 |

## 工作流程

```
项目分析
  ↓
框架检测 & 规则加载
  ↓
污点分析
  ├─ 正向: HTTP入口 → 追踪数据流 → 危险函数
  └─ 反向: 危险函数 → 追溯数据来源 → HTTP入口
  ↓
漏洞验证
  ├─ 检查调用链完整性
  ├─ 检查过滤点
  └─ 评估可利用性
  ↓
报告生成
  ├─ PoC生成
  ├─ 攻击链分析
  └─ 审计报告
```

## 输出示例

### JSON报告

```json
{
  "vulnerabilities": [
    {
      "vuln_id": "VULN-001",
      "type": "sqli",
      "severity": "CRITICAL",
      "entry_point": {
        "method": "GET",
        "path": "/api/users/:id"
      },
      "call_chain": [
        {
          "location": "routes/users.js:46",
          "type": "SOURCE",
          "code": "const userId = req.params.id"
        },
        {
          "location": "routes/users.js:49",
          "type": "SINK",
          "code": "await db.query(query)"
        }
      ],
      "poc": {
        "request": "GET /api/users/1' OR '1'='1"
      }
    }
  ]
}
```

## 配置

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
  ]
}
```

## 开发

### 添加新框架支持

1. 在 `rules/sources/` 创建对应语言的source规则
2. 在 `rules/sinks/` 创建对应语言的sink规则
3. 更新框架检测逻辑

### 添加新漏洞类型

1. 在 `rules/sinks/` 中定义新的sink点
2. 在 `agents/poc-generator.md` 中添加PoC模板
3. 更新报告模板

## 贡献

欢迎提交Issue和Pull Request！

## 许可证

MIT License

## 作者

Y5 Security Research Team

## 致谢

本工具受以下项目启发：
- CodeQL
- Semgrep
- SonarQube
- OWASP Dependency-Check
