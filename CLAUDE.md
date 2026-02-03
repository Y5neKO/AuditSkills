# AuditSkills - AI代码安全审计工具集

## 项目概述

本项目提供基于Claude AI的代码安全审计技能，使用**污点分析 (Taint Analysis)** 技术自动审计Web应用代码，发现安全漏洞并生成可验证的概念验证代码(PoC)。

## 项目结构

```
AuditSkills/
├── CLAUDE.md                 # 本文件 - 项目级AI上下文
├── README.md                 # 项目说明文档
├── skills/                   # Claude技能目录
│   └── code-audit/           # 代码审计主技能
│       ├── SKILL.md          # 主技能入口文件
│       ├── agents/           # 子代理配置
│       │   ├── forward-auditor/     # 正向审计代理
│       │   ├── backward-auditor/    # 反向审计代理
│       │   ├── poc-generator/       # PoC生成代理
│       │   ├── report-generator/    # 报告生成代理
│       │   └── attack-chain-orchestrator/  # 攻击链编排代理
│       ├── rules/            # 污点分析规则库
│       │   ├── sources/      # Source点规则（HTTP入口）
│       │   └── sinks/        # Sink点规则（危险函数）
│       ├── examples/         # 输出示例
│       └── templates/        # 报告模板
└── code-audit/               # 旧版本（待迁移）
    ├── .claude-plugin/
    ├── agents/
    ├── rules/
    └── README.md
```

## 使用方式

### 作为Claude插件使用

将此目录复制到Claude插件目录：

```bash
cp -r /path/to/AuditSkills ~/.claude/plugins/audit-skills
```

### 使用技能

```
/code-audit /path/to/project
```

## 技能说明

### code-audit技能

**描述**: AI代码安全审计工具，支持正向/反向污点分析

**用法**:
- `/code-audit analyze --project /path/to/project`
- `/code-audit /path/to/project --mode forward`

**核心特性**:
- 双向污点分析（正向：source→sink，反向：sink→source）
- 完整调用链追踪
- 多语言/多框架支持
- 自动生成PoC
- 攻击链编排

## 规则库

规则库定义了污点分析的source点和sink点：

### Source点规则 (rules/sources/)

用户可控输入点的检测模式：
- HTTP参数：`req.params`, `req.query`, `req.body`
- HTTP头：`req.headers`
- Cookie：`req.cookies`
- 文件上传：`req.files`

支持语言：JavaScript, Python, Java, PHP

### Sink点规则 (rules/sinks/)

危险函数的检测模式：
- SQL注入：`query()`, `execute()`
- 命令注入：`exec()`, `spawn()`
- 文件操作：`fs.readFile()`
- 模板注入：`render_template()`

支持语言：JavaScript, Python, Java, PHP

## 子代理架构

本项目使用多个专门子代理协作完成审计任务：

| 子代理 | 职责 |
|-------|------|
| forward-auditor | 从HTTP入口追踪数据流到危险函数 |
| backward-auditor | 从危险函数反向追溯数据来源 |
| poc-generator | 生成漏洞验证PoC代码 |
| report-generator | 生成结构化审计报告 |
| attack-chain-orchestrator | 组合多个漏洞构建攻击链 |

## 开发指南

### 添加新语言支持

1. 在 `skills/code-audit/rules/sources/` 创建 `{language}.json`
2. 在 `skills/code-audit/rules/sinks/` 创建 `{language}.json`
3. 更新子代理配置以支持新语言

### 添加新漏洞类型

1. 在对应语言的 `sinks/{language}.json` 中添加新的sink点
2. 在 `poc-generator/agent.md` 中添加PoC模板
3. 更新报告模板

### 创建新技能

1. 在 `skills/` 下创建新目录 `skill-name/`
2. 创建 `SKILL.md` 文件（必需）
3. 添加支持文件（agents/, templates/, examples/）

## 编码规范

- **SKILL.md**: YAML frontmatter + 简洁说明（建议<500行）
- **agents/**: 每个子代理一个目录，包含 `agent.md`
- **rules/**: JSON格式，包含 `language`, `framework` 字段
- **代码注释**: 与现有代码库注释语言保持一致

## 许可证

MIT License

## 作者

Y5 Security Research Team
