---
name: code-audit
description: AI代码安全审计工具。完整的代码安全审计流程：资产发现→入口功能分析→技术漏洞审计（反向+正向交叉验证）→业务逻辑审计→攻击链分析→报告生成。使用 /code-audit [项目路径] 启动完整审计。
argument-hint: [project-path] [--scope technical|business|all] [--phase 1,2,3,4,5]
context: fork
agent: general-purpose
model: inherit
allowed-tools: Read, Grep, Glob, Write, Bash, Task
---

# Code Audit - 代码安全审计工具

完整的代码安全审计流程，涵盖资产发现、技术漏洞、业务逻辑和攻击链分析。

## 快速开始

```
/code-audit /path/to/project                    # 完整审计
/code-audit /path/to/project --scope technical   # 仅技术漏洞
/code-audit /path/to/project --scope business    # 仅业务逻辑
/code-audit /path/to/project --phase 1,2        # 仅执行指定阶段
```

---

## 执行流程

当用户请求代码审计时，**必须严格按照以下步骤执行**，每一步都要使用 `Task` 工具调用相应的子代理：

### 阶段 0: 准备工作

1. **验证项目路径**
   - 检查项目路径是否存在
   - 识别项目语言和框架
   - 创建输出目录 `.workspace/code-audit/`

2. **报告进度**
   ```
   [1/7] 正在执行资产发现...
   ```

---

### 阶段 1: 资产发现（并行调用 4 个子代理）

使用 `Task` 工具**并行**调用以下子代理：

```javascript
// 并行调用所有资产发现子代理
// 每个子代理会将结果写入 .workspace/code-audit/phase1_discovery/ 目录
Task(subagent_type="web-entry-discovery",
     prompt="扫描项目 {{PROJECT_PATH}}，发现所有HTTP入口点。
              输出目录: .workspace/code-audit/phase1_discovery/
              输出文件: web_entries.json
              完成后返回: ✅ 发现 XX 个HTTP入口")

Task(subagent_type="sink-point-scanner",
     prompt="扫描项目 {{PROJECT_PATH}}，发现所有危险Sink点。
              输出目录: .workspace/code-audit/phase1_discovery/
              输出文件: sink_points.json
              完成后返回: ✅ 发现 XX 个危险Sink点")

Task(subagent_type="security-asset-scanner",
     prompt="扫描项目 {{PROJECT_PATH}}，识别敏感配置和凭据。
              输出目录: .workspace/code-audit/phase1_discovery/
              输出文件: security_assets.json
              完成后返回: ✅ 发现 XX 个敏感资产")

Task(subagent_type="data-model-analyzer",
     prompt="分析项目 {{PROJECT_PATH}} 的数据模型和所有权关系。
              输出目录: .workspace/code-audit/phase1_discovery/
              输出文件: data_models.json
              完成后返回: ✅ 分析 XX 个数据模型")
```

**等待所有子代理完成**，然后：
- 读取 `.workspace/code-audit/phase1_discovery/` 中的产物文件
- 合并结果到内存中供后续阶段使用
- 报告进度：`[2/7] 资产发现完成 - 发现 XX 个入口，XX 个Sink点`

---

### 阶段 1.5: 入口功能分析

```javascript
Task(subagent_type="entry-function-analyzer",
     prompt="深度分析每个HTTP入口的业务逻辑、参数处理和返回值。
              输入: phase1_discovery/web_entries.json
              输出目录: .workspace/code-audit/phase1_5_entry_analysis/
              输出文件: entry_functions.json
              完成后返回: ✅ 分析 XX 个入口函数")
```

- 报告进度：`[3/7] 入口功能分析完成`

---

### 阶段 2: 技术漏洞审计（反向追踪）

**顺序执行**以下子代理，每步读取上一阶段的文件产物，写入新的产物：

```javascript
// 2.1 数据流反向追踪
Task(subagent_type="dataflow-tracer",
     prompt="从Sink点反向追踪到Source点。
              输入: phase1_discovery/ 中的 web_entries.json 和 sink_points.json
              输出目录: .workspace/code-audit/phase2_technical_audit/
              输出文件: traces.json
              完成后返回: ✅ 追踪 XX 条数据流路径")

// 2.2 漏洞验证（读取 traces.json）
Task(subagent_type="vulnerability-validator",
     prompt="验证已发现数据流路径的真实可利用性。
              输入: phase2_technical_audit/traces.json
              输出目录: .workspace/code-audit/phase2_technical_audit/
              输出文件: validated_vulns.json
              完成后返回: ✅ 确认 XX 个漏洞，排除 XX 个误报")

// 2.3 漏洞关联分析（读取 validated_vulns.json）
Task(subagent_type="vulnerability-correlator",
     prompt="搜索同类漏洞模式，发现系统性安全问题。
              输入: phase2_technical_audit/validated_vulns.json
              输出目录: .workspace/code-audit/phase2_technical_audit/
              输出文件: correlated_vulns.json
              完成后返回: ✅ 发现 XX 组关联漏洞")

// 2.4 PoC生成（读取 correlated_vulns.json）
Task(subagent_type="poc-generator",
     prompt="为已验证的漏洞生成可验证的PoC。
              输入: phase2_technical_audit/correlated_vulns.json
              输出目录: .workspace/code-audit/phase2_technical_audit/pocs/
              完成后返回: ✅ 生成 XX 个PoC脚本")
```

- 报告进度：`[4/7] 技术漏洞审计完成 - 确认 XX 个漏洞`

---

### 阶段 2.5: 正向数据流追踪（交叉验证）

```javascript
Task(subagent_type="forward-flow-tracer",
     prompt="从Source点正向追踪到Sink点，与阶段2的反向结果交叉验证。
              输入: phase1_discovery/ 中的文件
              输出目录: .workspace/code-audit/phase2_5_forward_trace/
              输出文件: forward_traces.json
              完成后返回: ✅ 正向追踪完成，XX 条路径与反向结果匹配")
```

- 对比正向和反向结果
- 报告进度：`[5/7] 正向追踪和交叉验证完成`

---

### 阶段 3: 业务逻辑审计

```javascript
// 3.1 访问控制审计
Task(subagent_type="access-control-auditor",
     prompt="分析项目的访问控制，发现IDOR、垂直越权、水平越权漏洞。
              输入: phase1_discovery/data_models.json 和 phase1_5_entry_analysis/
              输出目录: .workspace/code-audit/phase3_business_audit/
              输出文件: access_control.json
              完成后返回: ✅ 发现 XX 个越权漏洞")

// 3.2 业务逻辑审计
Task(subagent_type="business-logic-auditor",
     prompt="分析项目的业务逻辑，发现支付、工作流等业务流程缺陷。
              输入: phase1_5_entry_analysis/
              输出目录: .workspace/code-audit/phase3_business_audit/
              输出文件: business_logic.json
              完成后返回: ✅ 发现 XX 个业务逻辑漏洞")
```

- 报告进度：`[6/7] 业务逻辑审计完成`

---

### 阶段 4: 攻击链分析

```javascript
Task(subagent_type="attack-chain-orchestrator",
     prompt="组合已发现的所有漏洞，构建完整的攻击链场景。
              输入: phase2_technical_audit/correlated_vulns.json 和 phase3_business_audit/
              输出目录: .workspace/code-audit/phase4_attack_chains/
              输出文件: attack_chains.json
              完成后返回: ✅ 构建 XX 条攻击链")
```

- 报告进度：`[7/7] 攻击链分析完成`

---

### 阶段 5: 报告生成

```javascript
Task(subagent_type="report-generator",
     prompt="整合所有阶段的发现，生成完整的Markdown和JSON审计报告。
              输入: 所有阶段的产物文件
              输出目录: .workspace/code-audit/
              输出文件: report.md, report.json
              完成后返回: ✅ 报告已生成")
```

- 报告进度：`审计完成！`

---

## 可用子代理

| 子代理名称 | 调用方式 | 用途 |
|-----------|---------|------|
| web-entry-discovery | `Task(subagent_type="web-entry-discovery")` | 发现HTTP入口 |
| sink-point-scanner | `Task(subagent_type="sink-point-scanner")` | 扫描危险Sink点 |
| security-asset-scanner | `Task(subagent_type="security-asset-scanner")` | 识别敏感配置 |
| data-model-analyzer | `Task(subagent_type="data-model-analyzer")` | 分析数据模型 |
| entry-function-analyzer | `Task(subagent_type="entry-function-analyzer")` | 分析入口功能 |
| dataflow-tracer | `Task(subagent_type="dataflow-tracer")` | 反向数据流追踪 |
| vulnerability-validator | `Task(subagent_type="vulnerability-validator")` | 验证漏洞 |
| vulnerability-correlator | `Task(subagent_type="vulnerability-correlator")` | 关联同类漏洞 |
| poc-generator | `Task(subagent_type="poc-generator")` | 生成PoC |
| forward-flow-tracer | `Task(subagent_type="forward-flow-tracer")` | 正向数据流追踪 |
| access-control-auditor | `Task(subagent_type="access-control-auditor")` | 访问控制审计 |
| business-logic-auditor | `Task(subagent_type="business-logic-auditor")` | 业务逻辑审计 |
| attack-chain-orchestrator | `Task(subagent_type="attack-chain-orchestrator")` | 攻击链分析 |
| report-generator | `Task(subagent_type="report-generator")` | 生成报告 |

---

## 支持的漏洞类型

- **SQL注入** (CWE-89)
- **XSS** (CWE-79)
- **SSRF** (CWE-918)
- **命令注入** (CWE-78)
- **路径遍历** (CWE-22)
- **IDOR/越权** (CWE-639)
- **业务逻辑缺陷**

## 支持的框架

- **Node.js**: Express, Koa, NestJS
- **Python**: Flask, Django, FastAPI
- **Java**: Spring Boot
- **PHP**: Laravel

---

## 输出结构

```
.workspace/code-audit/
├── phase1_discovery/           # 资产发现结果
├── phase1_5_entry_analysis/    # 入口功能分析
├── phase2_technical_audit/     # 技术漏洞审计
├── phase2_5_forward_trace/     # 正向追踪结果
├── phase3_business_audit/      # 业务逻辑审计
├── phase4_attack_chains/       # 攻击链分析
├── report.md                   # 最终报告（Markdown）
└── report.json                 # 最终报告（JSON）
```

---

## 重要提示

1. **必须使用 Task 工具调用子代理**，不能自己执行分析
2. **并行调用**可以提高效率（阶段1的4个子代理）
3. **顺序执行**确保数据传递正确（阶段2的子代理）
4. **每步完成后报告进度**给用户
5. **保存所有中间结果**到对应目录
