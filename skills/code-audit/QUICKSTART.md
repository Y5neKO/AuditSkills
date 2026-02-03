# Code Audit Skill - 快速使用指南

## 安装

1. 解压 `code-audit-fixed.tar.gz`
2. 将 `fixed_code-audit` 目录重命名为 `code-audit`
3. 放入你的 Claude skills 目录:
   ```
   .claude/skills/code-audit/
   ```

## 快速开始

只需对 Claude 说:

```
"帮我审计这个项目: /path/to/my-webapp"
```

或:

```
"对 /home/user/projects/my-app 做安全代码审查"
```

## 触发词

以下任何一个都会触发这个 skill:
- "代码审计"
- "代码审查"
- "安全审查"
- "漏洞扫描"
- "security audit"
- "code review"
- "vulnerability assessment"
- "penetration test"

## 支持的项目类型

| 语言 | 框架 |
|------|------|
| JavaScript/TypeScript | Express, Koa, NestJS |
| Python | Flask, Django, FastAPI |
| Java | Spring Boot |
| PHP | Laravel |

## 审计流程

Claude 会自动执行以下 7 个阶段:

1. **准备工作** - 检测项目框架,创建工作目录
2. **资产发现** - 发现 HTTP 端点、危险函数、敏感配置
3. **入口功能分析** - 分析每个端点的业务逻辑
4. **技术漏洞审计** - 反向污点追踪,发现注入类漏洞
5. **正向追踪验证** - 交叉验证,减少误报
6. **业务逻辑审计** - 发现越权、逻辑缺陷
7. **攻击链分析** - 组合漏洞构建攻击场景
8. **报告生成** - 生成完整审计报告

## 输出文件

审计完成后会生成:

```
.workspace/code-audit/
├── report.md          # 人类可读的完整报告
├── report.json        # 机器可读的结构化报告
└── phase*_*/          # 各阶段的中间产物
```

## 示例会话

```
你: 帮我审计 /home/projects/my-blog 这个 Express.js 项目

Claude: 
🔍 Starting code security audit...
[0/7] Preparing workspace...
✅ Detected: Express.js (JavaScript)

[1/7] Asset Discovery...
  - Found 23 HTTP endpoints
  - Found 15 dangerous sink points
  - Found 5 security assets
  - Analyzed 3 data models

[2/7] Entry Function Analysis...
  - Analyzed 23 entry functions

[3/7] Technical Vulnerability Audit (Backward)...
  - Traced 12 data flows
  - Validated 8 vulnerabilities
  - Generated 8 PoCs

[4/7] Forward Taint Analysis...
  - Cross-validated 8/8 vulnerabilities

[5/7] Business Logic Audit...
  - Found 2 IDOR vulnerabilities
  - Found 1 privilege escalation

[6/7] Attack Chain Construction...
  - Built 3 attack chains

[7/7] Report Generation...
  ✅ Audit Complete!

📊 Summary:
- Total Vulnerabilities: 11
- Critical: 3, High: 4, Medium: 3, Low: 1
- Attack Chains: 3

📄 Full report: .workspace/code-audit/report.md
```

## 自定义选项

虽然 skill 会自动执行完整审计,但你可以指定重点:

```
"只审计 /path/to/project 的 SQL 注入漏洞"
"专注检查 /path/to/project 的访问控制问题"
"帮我看看 /path/to/project 的业务逻辑有没有问题"
```

## 常见问题

### Q: 审计需要多长时间?
A: 取决于项目大小:
- 小型项目 (<1000 行): 2-3 分钟
- 中型项目 (1000-5000 行): 5-10 分钟
- 大型项目 (>5000 行): 10-20 分钟

### Q: 会修改我的代码吗?
A: 不会。Skill 只读取和分析代码,所有输出都在 `.workspace/code-audit/` 目录中。

### Q: 误报率如何?
A: Skill 使用双向污点追踪 (backward + forward) 来减少误报,但建议人工复核高风险漏洞。

### Q: 支持自定义规则吗?
A: 支持。可以编辑 `rules/sinks/` 和 `rules/sources/` 中的 JSON 文件添加自定义模式。

### Q: 可以用于生产环境吗?
A: 这是一个辅助工具,建议:
1. 在开发/测试环境使用
2. 配合人工审查
3. 不要依赖它作为唯一的安全措施

## 高级用法

### 添加自定义 Sink 规则

编辑 `rules/sinks/javascript.json`:

```json
{
  "sinks": [
    {
      "name": "myCustomDangerousFunction",
      "type": "custom_vulnerability",
      "patterns": [
        "myLib\\.dangerousOp\\(",
        "customExec\\("
      ],
      "severity": "HIGH",
      "description": "Custom dangerous operation"
    }
  ]
}
```

### 框架特定参考

Skill 包含框架特定的安全模式参考,会自动加载:
- `references/express-security.md` - Express.js
- (可添加更多框架)

## 获取帮助

如果遇到问题:
1. 查看 `FIXES.md` 了解架构变更
2. 查看 `SKILL.md` 了解完整文档
3. 确保项目路径正确且可访问
4. 检查 `.workspace/code-audit/` 中的中间产物

## 更新日志

### v2.0 (当前版本)
- ✅ 重构为纯 skill 架构 (移除 agents)
- ✅ 移除对不支持的 Task 工具的依赖
- ✅ 改进 YAML frontmatter (仅保留必需字段)
- ✅ 添加详细的执行算法说明
- ✅ 优化 token 使用 (progressive disclosure)

### v1.0 (原版本)
- ❌ 使用 agents + Task 工具 (不兼容 Claude.ai)
