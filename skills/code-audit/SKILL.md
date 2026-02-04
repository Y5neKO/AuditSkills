---
name: code-audit
description: Comprehensive AI-powered code security audit tool for web applications. Performs complete security assessment including asset discovery, technical vulnerability detection (SQL injection, XSS, SSRF, command injection, path traversal, deserialization, XXE), business logic flaws, and attack chain construction. Supports Node.js, Python, Java, and PHP. Use when the user requests code review, security audit, vulnerability assessment, or penetration testing of source code.
---

# Code Audit - AI代码安全审计工具

## 核心理念

**任何可达的 Source → Sink 数据流都是潜在漏洞，无论权限等级**

权限等级仅影响风险评级、利用难度和修复优先级，不影响是否报告。

---

## Stage 0: 准备工作 [MANDATORY]

### Step 0.1: 检测项目

```bash
find PROJECT_PATH -name "package.json" -o -name "pom.xml" -o -name "requirements.txt" -o -name "composer.json" | head -1
```

### Step 0.2: 加载检测规则 [MANDATORY - MUST READ FILES]

**YOU MUST READ THESE FILES:**

For Java:
```
Read rules/sinks/java.json
Read rules/sources/java.json
```

For JavaScript:
```
Read rules/sinks/javascript.json
Read rules/sources/javascript.json
```

For Python:
```
Read rules/sinks/python.json
Read rules/sources/python.json
```

For PHP:
```
Read rules/sinks/php.json
Read rules/sources/php.json
```

**REQUIRED OUTPUT:**
```
✅ Language: Java
✅ Loaded 52 sink patterns
✅ Loaded 31 source patterns
```

### Step 0.3: 创建工作空间

```bash
mkdir -p .workspace/code-audit/{phase1_discovery,phase2_analysis,phase3_validation,pocs}
```

---

## Stage 1: 资产发现 [4 PARALLEL TASKS]

### Task 1.1: HTTP入口发现

**MANDATORY OUTPUT: .workspace/code-audit/phase1_discovery/web_entries.json**

```
Task: web_entry_scanner
Prompt: |
  PROJECT_PATH: {path}
  SOURCE_RULES: {rules/sources/LANG.json content}
  
  Scan all HTTP endpoints using SOURCE_RULES patterns.
  
  YOU MUST CREATE FILE: .workspace/code-audit/phase1_discovery/web_entries.json
  
  Format:
  {
    "entry_points": [
      {
        "entry_id": "ENTRY-001",
        "method": "POST",
        "path": "/api/endpoint",
        "file": "Controller.java",
        "line": 50,
        "handler": "methodName",
        "parameters": [{"name": "param", "type": "query"}],
        "auth_required": "AUTHENTICATED"
      }
    ],
    "statistics": {"total": 45, "by_auth": {"PUBLIC": 5, "AUTHENTICATED": 35, "ADMIN": 5}}
  }
  
  Output: "✅ Found X endpoints (PUBLIC: Y, AUTH: Z, ADMIN: W)"
```

### Task 1.2: Sink点扫描

**MANDATORY OUTPUT: .workspace/code-audit/phase1_discovery/sink_points.json**

```
Task: sink_scanner
Prompt: |
  PROJECT_PATH: {path}
  SINK_RULES: {rules/sinks/LANG.json content}
  
  CRITICAL: Search ALL patterns in SINK_RULES.
  
  YOU MUST CREATE FILE: .workspace/code-audit/phase1_discovery/sink_points.json
  
  Format:
  {
    "sinks": [
      {
        "sink_id": "SINK-001",
        "type": "sql_injection",
        "pattern_matched": "\\.createQuery\\(",
        "function": "createQuery",
        "file": "Action.java",
        "line": 248,
        "code_snippet": "query(sql)",
        "tainted_variable": "sql",
        "severity": "HIGH",
        "cwe": "CWE-89"
      }
    ],
    "statistics": {"total": 47, "by_type": {...}, "patterns_searched": 52}
  }
  
  Output: "✅ Found X sinks, searched Y patterns"
```

### Task 1.3: 安全资产扫描

**MANDATORY OUTPUT: .workspace/code-audit/phase1_discovery/security_assets.json**

```
Task: security_asset_scanner
Prompt: |
  Search security configs and credentials.
  
  YOU MUST CREATE FILE: .workspace/code-audit/phase1_discovery/security_assets.json
  
  Format:
  {
    "assets": [...],
    "statistics": {"total": 12}
  }
  
  Output: "✅ Found X assets"
```

### Task 1.4: 数据模型分析

**MANDATORY OUTPUT: .workspace/code-audit/phase1_discovery/data_models.json**

```
Task: data_model_analyzer
Prompt: |
  Analyze data models.
  
  YOU MUST CREATE FILE: .workspace/code-audit/phase1_discovery/data_models.json
  
  Format:
  {
    "models": [...],
    "statistics": {"total": 8}
  }
  
  Output: "✅ Analyzed X models"
```

### Stage 1 验证 [MANDATORY]

```bash
# Main flow MUST:
Read .workspace/code-audit/phase1_discovery/web_entries.json
Read .workspace/code-audit/phase1_discovery/sink_points.json
Read .workspace/code-audit/phase1_discovery/security_assets.json
Read .workspace/code-audit/phase1_discovery/data_models.json
```

**REQUIRED OUTPUT:**
```
[1/4] ✅ Asset Discovery
  ✅ web_entries.json (45 endpoints)
  ✅ sink_points.json (47 sinks, 52 patterns)
  ✅ security_assets.json (12 assets)
  ✅ data_models.json (8 models)
```

---

## Stage 2: 深度分析 [3 SEQUENTIAL TASKS - 依赖链执行]

> ⚠️ **CRITICAL: 以下任务必须按顺序执行，每个任务依赖前一个任务的输出**

### Step 2.1: LSP代码分析

**依赖输入:**
- `.workspace/code-audit/phase1_discovery/sink_points.json`

**MANDATORY OUTPUT: .workspace/code-audit/phase2_analysis/lsp_analysis.json**

```
Task: lsp_analyzer
Prompt: |
  Read .workspace/code-audit/phase1_discovery/sink_points.json

  Use LSP if available, else grep.

  YOU MUST CREATE FILE: .workspace/code-audit/phase2_analysis/lsp_analysis.json

  Format:
  {
    "lsp_available": true,
    "variable_flows": [...],
    "statistics": {"total_flows": 47}
  }

  Output: "✅ LSP: {status}, Flows: X"
```

**完成验证:**
```bash
# MUST verify output exists:
Read .workspace/code-audit/phase2_analysis/lsp_analysis.json
```

**REQUIRED OUTPUT:**
```
[2.1/3] ✅ LSP Analysis
  ✅ lsp_analysis.json (LSP: yes, 47 flows)
```

---

### Step 2.2: 反向污点追踪

**依赖输入:**
- `.workspace/code-audit/phase1_discovery/sink_points.json`
- `.workspace/code-audit/phase1_discovery/web_entries.json`
- `.workspace/code-audit/phase2_analysis/lsp_analysis.json` ⬅️ 来自 Step 2.1

**MANDATORY OUTPUT: .workspace/code-audit/phase2_analysis/backward_traces.json**

```
Task: backward_tracer
Prompt: |
  Read .workspace/code-audit/phase1_discovery/sink_points.json
  Read .workspace/code-audit/phase1_discovery/web_entries.json
  Read .workspace/code-audit/phase2_analysis/lsp_analysis.json

  CRITICAL: Trace ALL sinks.

  YOU MUST CREATE FILE: .workspace/code-audit/phase2_analysis/backward_traces.json

  Format:
  {
    "traces": [
      {
        "trace_id": "TRACE-001",
        "sink_id": "SINK-001",
        "entry_id": "ENTRY-015",
        "auth_required": "AUTHENTICATED",
        "call_chain": [...],
        "is_vulnerable": true
      }
    ],
    "statistics": {"total_traces": 47, "vulnerable": 23}
  }

  Output: "✅ Traced X/Y, Z vulnerable"
```

**完成验证:**
```bash
# MUST verify output exists:
Read .workspace/code-audit/phase2_analysis/backward_traces.json
```

**REQUIRED OUTPUT:**
```
[2.2/3] ✅ Backward Tracing
  ✅ backward_traces.json (47 traces, 23 vulnerable)
```

---

### Step 2.3: 正向污点追踪

**依赖输入:**
- `.workspace/code-audit/phase1_discovery/web_entries.json`
- `.workspace/code-audit/phase1_discovery/sink_points.json`
- `.workspace/code-audit/phase2_analysis/backward_traces.json` ⬅️ 来自 Step 2.2

**MANDATORY OUTPUT: .workspace/code-audit/phase2_analysis/forward_traces.json**

```
Task: forward_tracer
Prompt: |
  Read .workspace/code-audit/phase1_discovery/web_entries.json
  Read .workspace/code-audit/phase1_discovery/sink_points.json
  Read .workspace/code-audit/phase2_analysis/backward_traces.json

  YOU MUST CREATE FILE: .workspace/code-audit/phase2_analysis/forward_traces.json

  Format:
  {
    "traces": [...],
    "statistics": {"validated_backward": 22}
  }

  Output: "✅ Validated X traces"
```

**完成验证:**
```bash
# MUST verify output exists:
Read .workspace/code-audit/phase2_analysis/forward_traces.json
```

**REQUIRED OUTPUT:**
```
[2.3/3] ✅ Forward Tracing
  ✅ forward_traces.json (validated 22)
```

### Stage 2 汇总输出 [MANDATORY]

**所有步骤完成后输出:**
```
[2/4] ✅ Deep Analysis Complete
  Step 2.1: ✅ lsp_analysis.json (LSP: yes, 47 flows)
  Step 2.2: ✅ backward_traces.json (47 traces, 23 vulnerable)
  Step 2.3: ✅ forward_traces.json (validated 22)
```

---

## Stage 3: 漏洞验证 [SEQUENTIAL]

### Step 3.1: 读取所有产物 [MANDATORY]

```bash
# MUST READ ALL:
Read .workspace/code-audit/phase1_discovery/web_entries.json
Read .workspace/code-audit/phase1_discovery/sink_points.json
Read .workspace/code-audit/phase1_discovery/security_assets.json
Read .workspace/code-audit/phase1_discovery/data_models.json
Read .workspace/code-audit/phase2_analysis/lsp_analysis.json
Read .workspace/code-audit/phase2_analysis/backward_traces.json
Read .workspace/code-audit/phase2_analysis/forward_traces.json
```

### Step 3.2: 漏洞验证和评级

**MANDATORY OUTPUT: .workspace/code-audit/phase3_validation/validated_vulnerabilities.json**

```
For each trace in backward_traces:
  1. Confirm source-to-sink path
  2. Assess filters
  3. Adjust severity by auth level:
     - PUBLIC: increase severity
     - AUTHENTICATED: keep severity
     - ADMIN: decrease severity
  4. Calculate CVSS
```

**Format:**
```json
{
  "vulnerabilities": [
    {
      "vuln_id": "VULN-001",
      "type": "sql_injection",
      "severity": "HIGH",
      "auth_required": "AUTHENTICATED",
      "cvss": 8.1,
      "file": "Action.java",
      "line": 248,
      "priority": "P1"
    }
  ],
  "statistics": {"total": 23, "by_severity": {...}, "by_auth": {...}}
}
```

**CREATE FILE:**
```bash
Write .workspace/code-audit/phase3_validation/validated_vulnerabilities.json
```

### Step 3.3: PoC生成 [MANDATORY]

**MANDATORY: Create one PoC file per vulnerability**

```bash
# For each vulnerability:
Write .workspace/code-audit/pocs/VULN-{id}_poc.txt
```

**Format:**
```
# VULN-001: SQL Injection

## Auth: AUTHENTICATED

## Path
Source: req.getParameter("sql") line 206
Sink: jdbc.query(sql) line 248

## PoC
POST /endpoint
Cookie: SESSION=...
sql=...

## Risk
Severity: HIGH (auth required)
CVSS: 8.1
Priority: P1
```

### Step 3.4: 攻击链 [MANDATORY]

**MANDATORY OUTPUT: .workspace/code-audit/phase3_validation/attack_chains.json**

```json
{
  "attack_chains": [
    {
      "chain_id": "CHAIN-001",
      "name": "SQLi to RCE",
      "severity": "CRITICAL",
      "steps": [...]
    }
  ],
  "statistics": {"total_chains": 5}
}
```

**CREATE FILE:**
```bash
Write .workspace/code-audit/phase3_validation/attack_chains.json
```

### Stage 3 验证 [MANDATORY]

```bash
# MUST verify:
Read .workspace/code-audit/phase3_validation/validated_vulnerabilities.json
Read .workspace/code-audit/phase3_validation/attack_chains.json
ls .workspace/code-audit/pocs/*.txt
```

**REQUIRED OUTPUT:**
```
[3/4] ✅ Validation
  ✅ validated_vulnerabilities.json (23 vulns)
  ✅ attack_chains.json (5 chains)
  ✅ 23 PoC files created
```

---

## Stage 4: 报告生成 [MANDATORY]

### Step 4.1: 读取所有产物 [MANDATORY]

```bash
# MUST READ ALL 10 FILES:
Read .workspace/code-audit/phase1_discovery/web_entries.json
Read .workspace/code-audit/phase1_discovery/sink_points.json
Read .workspace/code-audit/phase1_discovery/security_assets.json
Read .workspace/code-audit/phase1_discovery/data_models.json
Read .workspace/code-audit/phase2_analysis/lsp_analysis.json
Read .workspace/code-audit/phase2_analysis/backward_traces.json
Read .workspace/code-audit/phase2_analysis/forward_traces.json
Read .workspace/code-audit/phase3_validation/validated_vulnerabilities.json
Read .workspace/code-audit/phase3_validation/attack_chains.json
```

### Step 4.2: 生成报告 [MANDATORY]

**MANDATORY OUTPUT: .workspace/code-audit/report.md**

```markdown
# 代码安全审计报告

## 摘要
- 漏洞: 23 (CRITICAL:2, HIGH:15, MEDIUM:5, LOW:1)
- 覆盖: 100% (45/45 endpoints, 47/47 sinks, 52/52 patterns)

## 漏洞详情
### VULN-001: SQL Injection [HIGH]
- Auth: AUTHENTICATED
- CVSS: 8.1
- File: Action.java:248
- PoC: (详细)
- Fix: (具体代码)

## 攻击链
(5 chains)

## 优先级
P0: 2, P1: 15, P2: 5, P3: 1
```

**MANDATORY OUTPUT: .workspace/code-audit/report.json**

```json
{
  "summary": {"total": 23, "by_severity": {...}},
  "coverage": {"endpoints": "45/45", "sinks": "47/47", "patterns": "52/52"},
  "vulnerabilities": [...],
  "attack_chains": [...],
  "lsp_enabled": true
}
```

**CREATE FILES:**
```bash
Write .workspace/code-audit/report.md
Write .workspace/code-audit/report.json
```

### Step 4.3: 打包证据 [MANDATORY]

```bash
tar -czf .workspace/code-audit/audit-evidence.tar.gz \
  .workspace/code-audit/phase1_discovery/ \
  .workspace/code-audit/phase2_analysis/ \
  .workspace/code-audit/phase3_validation/
```

### Stage 4 验证 [MANDATORY]

```bash
ls .workspace/code-audit/report.md
ls .workspace/code-audit/report.json
ls .workspace/code-audit/audit-evidence.tar.gz
```

**REQUIRED FINAL OUTPUT:**
```
[4/4] ✅ Audit Complete!

📊 Summary:
  Vulnerabilities: 23
    - CRITICAL: 2 (no auth)
    - HIGH: 15 (auth)
    - MEDIUM: 5 (privileged)
  
  Coverage: 100%
    - Endpoints: 45/45
    - Sinks: 47/47
    - Patterns: 52/52
  
  Attack Chains: 5
  LSP: ✅ Enabled
  Time: 45s

📁 Files:
  ✅ report.md
  ✅ report.json
  ✅ audit-evidence.tar.gz
  ✅ 23 PoC files

🔴 Critical (P0 - Fix Now):
  1. VULN-001: SQL Injection (CVSS 9.8)
  2. VULN-002: RCE (CVSS 9.1)

🟠 High (P1 - Fix in 1 Week):
  3-17. (15 issues)

Total Files Created: 35
  Stage 1: 4
  Stage 2: 3
  Stage 3: 25
  Stage 4: 3
```

---

## CRITICAL REQUIREMENTS

**The audit FAILS if:**
- ❌ Rules not read in Stage 0
- ❌ Any Task output file missing
- ❌ Main flow doesn't read Task outputs
- ❌ Any sink not traced
- ❌ Reports not created

**Success requires:**
- ✅ Read 2 rules files in Stage 0
- ✅ Create 4 files in Stage 1
- ✅ Read 4 files after Stage 1
- ✅ Create 3 files in Stage 2
- ✅ Read 7 files after Stage 2
- ✅ Create 25 files in Stage 3
- ✅ Read 10 files in Stage 4
- ✅ Create 3 final files
- ✅ All 52 patterns searched
- ✅ All 47 sinks traced
