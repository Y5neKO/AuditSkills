# Code Audit Skill 修复说明

## 修复概述

将原有的 `agents/` + `skills/` 架构重构为纯 `skill` 架构,移除了对不支持的特性的依赖,使其能在 Claude.ai 环境中正常工作。

## 主要问题与修复

### 1. ❌ 问题: 使用了不支持的 YAML 字段

**原始代码 (SKILL.md frontmatter):**
```yaml
---
name: code-audit
description: ...
argument-hint: [project-path] [--scope technical|business|all]
context: fork
agent: general-purpose
model: inherit
allowed-tools: Read, Grep, Glob, Write, Bash, Task
---
```

**问题:**
- `argument-hint`: 不是标准字段
- `context`: 不支持
- `agent`: 不支持
- `allowed-tools`: 不支持

**✅ 修复:**
```yaml
---
name: code-audit
description: Comprehensive AI-powered code security audit tool... (详细描述何时使用)
---
```

只保留必需的 `name` 和 `description` 字段。

### 2. ❌ 问题: 依赖 Task 工具调用子代理

**原始代码:**
```javascript
Task(subagent_type="web-entry-discovery",
     prompt="PROJECT_PATH={{PROJECT_PATH}}
              OUTPUT_DIR=.workspace/code-audit
              扫描项目发现所有HTTP入口点")
```

**问题:**
- Claude.ai 不支持 `Task` 工具
- 无法调用外部"子代理"
- 这种架构在 Claude.ai 中无法工作

**✅ 修复:**

将所有"子代理"的逻辑直接嵌入到 SKILL.md 的 workflow 中,使用标准工具 (`bash`, `grep`, `Read`, `Write`) 实现相同功能:

```markdown
### Stage 1.1: Web Entry Discovery

```bash
# For Express.js/Koa
grep -rn "app\.\(get\|post\|put\|delete\)" PROJECT_PATH --include="*.js" > /tmp/web_entries.txt

# Parse results
# ... (process grep output and save to JSON)
```
```

### 3. ❌ 问题: 独立的 agents/ 目录

**原始结构:**
```
_claude/
├── agents/
│   ├── web-entry-discovery.md
│   ├── sink-point-scanner.md
│   ├── dataflow-tracer.md
│   └── ... (20+ agent files)
└── skills/
    └── code-audit/
        └── SKILL.md
```

**问题:**
- Claude.ai 不支持独立的 agent 文件
- 无法"调用"这些 agents
- 这些文件会被忽略

**✅ 修复:**

将所有 agent 的功能整合到单一 SKILL.md 中作为 workflow stages:

```
fixed_code-audit/
├── SKILL.md (包含所有原 agents 的逻辑)
├── rules/
│   ├── sinks/
│   └── sources/
├── references/
│   └── express-security.md (可选的参考文档)
└── examples/
    └── sample-report.json
```

### 4. ❌ 问题: Description 字段混入使用指令

**原始代码:**
```yaml
description: AI代码安全审计工具。完整的代码安全审计流程...使用 /code-audit [项目路径] 启动完整审计。
```

**问题:**
- Description 应该只描述"是什么"和"何时触发"
- 不应包含使用命令

**✅ 修复:**
```yaml
description: Comprehensive AI-powered code security audit tool for web applications. Performs complete security assessment including: asset discovery, technical vulnerability detection, business logic flaws, and attack chain construction. Supports Node.js, Python, Java, and PHP. Use when the user requests code review, security audit, vulnerability assessment, or penetration testing of source code.
```

- 清晰描述 skill 的功能
- 明确列出触发条件 (code review, security audit, vulnerability assessment, penetration testing)
- 列出支持的语言/框架

## 架构对比

### 原架构 (不可用):
```
用户请求
    ↓
code-audit skill (SKILL.md)
    ↓
调用 Task 工具
    ↓
Task 工具加载并执行各个 agent
    ↓ (web-entry-discovery.md)
    ↓ (sink-point-scanner.md)
    ↓ (dataflow-tracer.md)
    ↓ ...
    ↓
产生结果
```

### 新架构 (可用):
```
用户请求
    ↓
code-audit skill (SKILL.md)
    ↓
直接执行 workflow stages:
    ├─ Stage 0: Preparation (bash)
    ├─ Stage 1: Asset Discovery (grep + bash)
    │   ├─ Web Entry Discovery
    │   ├─ Sink Point Scanner
    │   ├─ Security Asset Scanner
    │   └─ Data Model Analyzer
    ├─ Stage 1.5: Entry Function Analysis (Read + analysis)
    ├─ Stage 2: Technical Audit (Read + grep + analysis)
    │   ├─ Backward Taint Analysis
    │   ├─ Vulnerability Validation
    │   ├─ Correlation
    │   └─ PoC Generation
    ├─ Stage 2.5: Forward Taint Analysis
    ├─ Stage 3: Business Logic Audit
    ├─ Stage 4: Attack Chain Construction
    └─ Stage 5: Report Generation (Write)
```

## 功能保留情况

### ✅ 保留的功能
- 完整的 7 阶段审计流程
- 资产发现 (HTTP endpoints, sinks, security assets, data models)
- 反向污点追踪 (backward taint analysis)
- 正向污点追踪 (forward taint analysis)
- 漏洞验证和关联
- PoC 生成
- 业务逻辑审计 (IDOR, privilege escalation)
- 攻击链构建
- 报告生成
- 支持多语言/框架 (Node.js, Python, Java, PHP)
- Source/Sink 规则库

### 🔄 改进的功能
- **更清晰的执行流程**: 每个阶段都有明确的 bash/grep/Read 命令
- **更详细的算法说明**: 污点追踪等复杂逻辑有伪代码说明
- **Progressive disclosure**: 框架特定的详细信息移到 references/ 目录
- **更好的 token 效率**: 只在需要时加载 references 文件

### ❌ 移除的不可用特性
- Task 工具调用
- 独立的 agent 文件
- "并行"调用子代理 (改为顺序执行,但可用 bash 后台任务模拟)
- 不支持的 YAML 字段

## 使用方式

### 触发 skill
当用户说以下任何一句时,skill 会自动触发:
- "帮我审计这个代码"
- "做代码安全审查"
- "检查这个项目有没有漏洞"
- "penetration test this application"
- "security assessment"

### 执行审计
```
用户: 帮我审计 /path/to/my-web-app

Claude 会:
1. 读取 SKILL.md
2. 检测项目框架
3. 执行 7 个阶段:
   - 准备工作
   - 资产发现
   - 入口功能分析
   - 技术漏洞审计 (反向)
   - 正向追踪验证
   - 业务逻辑审计
   - 攻击链分析
   - 报告生成
4. 输出完整报告
```

## 文件结构

```
fixed_code-audit/
├── SKILL.md                           # 主 skill 文件 (包含所有逻辑)
├── rules/                             # 漏洞检测规则库
│   ├── sinks/
│   │   ├── javascript.json           # JS 危险函数
│   │   ├── python.json               # Python 危险函数
│   │   ├── java.json                 # Java 危险函数
│   │   └── php.json                  # PHP 危险函数
│   └── sources/
│       ├── javascript.json           # JS 输入源
│       ├── python.json               # Python 输入源
│       ├── java.json                 # Java 输入源
│       └── php.json                  # PHP 输入源
├── references/                        # 可选的参考文档
│   └── express-security.md           # Express.js 安全模式参考
└── examples/
    └── sample-report.json            # 示例报告格式
```

## 与 .claude 标准结构的差异

标准的 `.claude/` 目录结构是:
```
.claude/
└── skills/
    └── your-skill/
        └── SKILL.md
```

但你的原始结构有额外的 `agents/` 目录:
```
_claude/
├── agents/      # ← 这不是标准结构
└── skills/
```

修复后的结构符合标准:
```
fixed_code-audit/    # 这就是一个完整的 skill 目录
├── SKILL.md
└── ... (bundled resources)
```

可以直接放入 `.claude/skills/code-audit/` 使用。

## 测试建议

1. **基本触发测试**:
   ```
   "帮我审计这个项目的代码安全性"
   ```
   预期: skill 应该触发并开始执行

2. **完整流程测试**:
   提供一个简单的 Express.js 项目,检查是否:
   - 正确检测到框架
   - 发现 HTTP endpoints
   - 识别危险函数 (sinks)
   - 生成报告

3. **漏洞检测测试**:
   提供包含已知漏洞的代码 (如 SQL 注入),检查是否:
   - 正确追踪数据流
   - 识别漏洞
   - 生成 PoC

## 后续优化建议

1. **添加更多框架支持**:
   - 在 references/ 中添加 Django, Spring Boot, Laravel 的安全模式文档

2. **扩展规则库**:
   - 添加更多语言特定的 source/sink 模式
   - 添加框架特定的危险函数

3. **改进算法**:
   - 优化污点追踪算法的准确性
   - 减少误报

4. **更好的报告**:
   - 添加可视化 (调用图、数据流图)
   - 生成 HTML 报告

## 关键改进总结

| 方面 | 原版本 | 修复版本 |
|------|--------|----------|
| 架构 | Skill + 20+ Agents | 单一 Skill |
| 工具依赖 | Task (不支持) | bash, grep, Read, Write |
| YAML 字段 | 5+ 字段 | 2 字段 (name, description) |
| 可用性 | ❌ 无法在 Claude.ai 中使用 | ✅ 完全可用 |
| Token 效率 | 所有 agents 都加载 | 按需加载 references |
| 维护性 | 分散在多个文件 | 集中在单一文件 |
