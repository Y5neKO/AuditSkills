# Code Audit Skill - 修复版

这是针对 Claude.ai 环境修复和优化的代码安全审计 skill。

## 📦 包含文件

```
fixed_code-audit/
├── SKILL.md              # ⭐ 主 skill 文件 (完整的审计流程)
├── QUICKSTART.md         # 🚀 快速开始指南
├── FIXES.md              # 🔧 详细修复说明
├── COMPARISON.md         # 📊 修复前后对比
├── rules/                # 📋 漏洞检测规则库
│   ├── sinks/           #    危险函数定义 (SQL, XSS, 命令注入等)
│   └── sources/         #    用户输入源定义
├── references/           # 📚 框架特定参考文档
│   └── express-security.md
└── examples/             # 📄 示例输出
    └── sample-report.json
```

## 🎯 主要修复内容

1. **移除对不支持特性的依赖**
   - ❌ 移除 Task 工具调用
   - ❌ 移除独立的 agents/ 目录
   - ❌ 移除不支持的 YAML 字段

2. **重构为纯 skill 架构**
   - ✅ 所有逻辑整合到 SKILL.md
   - ✅ 使用标准工具 (bash, grep, Read, Write)
   - ✅ 完全自包含,无外部依赖

3. **改进 token 效率**
   - ✅ 核心逻辑在 SKILL.md
   - ✅ 可选文档在 references/ (按需加载)
   - ✅ 规则库独立存储

## 📖 阅读顺序

### 如果你想快速使用:
1. **QUICKSTART.md** - 5 分钟了解如何使用

### 如果你想了解修复内容:
1. **FIXES.md** - 详细的修复说明和原理
2. **COMPARISON.md** - 修复前后的详细对比

### 如果你想深入了解:
1. **SKILL.md** - 完整的技术文档和实现细节

## 🚀 快速开始

### 安装
```bash
# 1. 解压文件
tar -xzf code-audit-fixed.tar.gz

# 2. 放入 Claude skills 目录
mv fixed_code-audit ~/.claude/skills/code-audit
```

### 使用
对 Claude 说:
```
"帮我审计这个项目的安全性: /path/to/my-webapp"
```

就这么简单!

## ✨ 核心功能

- ✅ **资产发现**: HTTP 端点、危险函数、敏感配置、数据模型
- ✅ **技术漏洞**: SQL 注入、XSS、SSRF、命令注入、路径遍历
- ✅ **业务逻辑**: IDOR、权限提升、访问控制缺陷
- ✅ **污点追踪**: 双向追踪 (backward + forward)
- ✅ **攻击链**: 组合漏洞构建完整攻击场景
- ✅ **完整报告**: Markdown + JSON 格式

## 🔧 支持的技术栈

| 语言 | 框架 |
|------|------|
| JavaScript/TypeScript | Express, Koa, NestJS |
| Python | Flask, Django, FastAPI |
| Java | Spring Boot |
| PHP | Laravel |

## 📊 与原版本的主要区别

| 特性 | 原版本 | 修复版 |
|------|--------|--------|
| 架构 | Skill + 20+ Agents | 单一 Skill |
| 可用性 | ❌ 无法在 Claude.ai 使用 | ✅ 完全可用 |
| 文件数 | 25+ | 1 + resources |
| 维护性 | 复杂 | 简单 |
| Token 效率 | 低 | 高 |

详见 **COMPARISON.md** 了解完整对比。

## 🎓 审计流程

完整的 7 阶段审计流程:

```
Stage 0: Preparation          准备工作和框架检测
    ↓
Stage 1: Asset Discovery      资产发现 (并行)
    ↓
Stage 1.5: Entry Analysis     入口功能分析
    ↓
Stage 2: Technical Audit      技术漏洞审计 (反向)
    ↓
Stage 2.5: Forward Trace      正向追踪验证
    ↓
Stage 3: Business Logic       业务逻辑审计
    ↓
Stage 4: Attack Chains        攻击链构建
    ↓
Stage 5: Report Generation    报告生成
```

## 📝 输出示例

审计完成后生成:

```
.workspace/code-audit/
├── report.md          # 完整的审计报告
├── report.json        # 结构化数据
└── phase*_*/          # 各阶段中间产物
    ├── web_entries.json
    ├── sink_points.json
    ├── traces.json
    ├── validated_vulns.json
    └── ...
```

## 🛠️ 自定义

### 添加自定义检测规则

编辑 `rules/sinks/LANGUAGE.json`:

```json
{
  "sinks": [
    {
      "name": "myCustomDangerousFunction",
      "type": "custom_vulnerability",
      "patterns": ["myLib\\.danger\\("],
      "severity": "HIGH"
    }
  ]
}
```

### 添加框架支持

在 `references/` 中添加新的框架参考文档。

## ❓ 常见问题

**Q: 这个 skill 安全吗?**
A: 完全安全。只读取代码,不会修改任何文件。所有输出在 `.workspace/` 目录。

**Q: 误报率如何?**
A: 使用双向污点追踪降低误报。建议人工复核高风险漏洞。

**Q: 多长时间?**
A: 小型项目 2-3 分钟,中型 5-10 分钟,大型 10-20 分钟。

**Q: 可以审计生产代码吗?**
A: 可以,但建议:
- 先在测试环境验证
- 配合人工审查
- 不要作为唯一的安全措施

## 📚 更多文档

- **QUICKSTART.md** - 快速开始指南
- **FIXES.md** - 详细修复说明 (为什么要修复,如何修复)
- **COMPARISON.md** - 修复前后详细对比
- **SKILL.md** - 完整技术文档

## 🤝 贡献

发现问题或有改进建议?

1. 编辑相应的文件 (主要是 SKILL.md)
2. 测试修改
3. 提交反馈

## 📄 许可

(根据你的需求添加许可信息)

---

**最后更新**: 2025-02-03
**版本**: 2.0 (修复版)
**兼容**: Claude.ai, Claude Code

---

## 🎉 开始使用

```bash
# 解压
tar -xzf code-audit-fixed.tar.gz

# 查看快速指南
cat fixed_code-audit/QUICKSTART.md

# 安装
mv fixed_code-audit ~/.claude/skills/code-audit

# 使用
# 对 Claude 说: "帮我审计 /path/to/project"
```

祝审计愉快! 🔍🛡️
