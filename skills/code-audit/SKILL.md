---
name: code-audit
description: Comprehensive AI-powered code security audit tool for web applications. Performs complete security assessment including: asset discovery (HTTP endpoints, dangerous sinks, sensitive data), technical vulnerability detection (SQL injection, XSS, SSRF, command injection, path traversal using both backward and forward taint analysis), business logic flaws (IDOR, privilege escalation, access control issues), and attack chain construction. Supports Node.js (Express/Koa/NestJS), Python (Flask/Django/FastAPI), Java (Spring Boot), and PHP (Laravel). Use when the user requests code review, security audit, vulnerability assessment, or penetration testing of source code.
---

# Code Audit - AI代码安全审计工具

完整的代码安全审计流程,涵盖资产发现、技术漏洞、业务逻辑和攻击链分析。

## Quick Start

```bash
# 在项目根目录运行
/code-audit /path/to/project
```

## Execution Workflow

执行代码审计时,**必须严格按照以下阶段顺序执行**:

### Stage 0: Preparation (准备工作)

1. **Verify project path and detect framework**
   ```bash
   # Check project exists
   ls -la PROJECT_PATH
   
   # Detect language and framework
   find PROJECT_PATH -name "package.json" -o -name "requirements.txt" -o -name "pom.xml" -o -name "composer.json"
   ```

2. **Create workspace structure**
   ```bash
   mkdir -p .workspace/code-audit/{phase1_discovery,phase1_5_entry_analysis,phase2_technical_audit,phase2_5_forward_trace,phase3_business_audit,phase4_attack_chains}
   ```

3. **Report progress to user**
   ```
   🔍 Starting code security audit...
   [0/7] Preparing workspace...
   ```

### Stage 1: Asset Discovery (资产发现)

**Purpose**: Discover all security-relevant assets in the codebase.

**Execute the following 4 discovery tasks in parallel using bash background jobs:**

#### 1.1 Web Entry Discovery

Scan for all HTTP endpoints (routes, controllers, API endpoints).

```bash
# For Express.js/Koa
grep -rn "app\.\(get\|post\|put\|delete\|patch\)" PROJECT_PATH --include="*.js" --include="*.ts" > /tmp/web_entries.txt
grep -rn "router\.\(get\|post\|put\|delete\|patch\)" PROJECT_PATH --include="*.js" --include="*.ts" >> /tmp/web_entries.txt

# For Flask/Django
grep -rn "@app\.route\|@bp\.route\|@api_view" PROJECT_PATH --include="*.py" >> /tmp/web_entries.txt
grep -rn "path(" PROJECT_PATH --include="*.py" >> /tmp/web_entries.txt

# For Spring Boot
grep -rn "@\(Get\|Post\|Put\|Delete\|Patch\)Mapping" PROJECT_PATH --include="*.java" >> /tmp/web_entries.txt

# For Laravel
grep -rn "Route::\(get\|post\|put\|delete\|patch\)" PROJECT_PATH --include="*.php" >> /tmp/web_entries.txt
```

**Parse results and save to JSON:**

Read the appropriate sink rules from `rules/sources/LANGUAGE.json` and match against discovered patterns.

Output format: `.workspace/code-audit/phase1_discovery/web_entries.json`

```json
{
  "project_info": {
    "framework": "Express.js",
    "language": "JavaScript",
    "scan_time": "2025-01-15T10:30:00Z"
  },
  "entry_points": [
    {
      "entry_id": "ENTRY-001",
      "method": "GET",
      "path": "/api/users/:id",
      "file": "routes/users.js",
      "line": 15,
      "handler": "getUserById",
      "parameters": [
        {"name": "id", "type": "path", "data_type": "string"}
      ]
    }
  ],
  "statistics": {
    "total_entries": 45,
    "by_method": {"GET": 20, "POST": 15, "PUT": 5, "DELETE": 3}
  }
}
```

#### 1.2 Sink Point Scanner

Scan for dangerous function calls (SQL queries, file operations, command execution, template rendering).

Load sink patterns from `rules/sinks/LANGUAGE.json` and search:

```bash
# Example for JavaScript
grep -rn "\.query\(.*\${\|\.execute\(.*\${\|eval(" PROJECT_PATH --include="*.js" > /tmp/sinks.txt
```

Output: `.workspace/code-audit/phase1_discovery/sink_points.json`

```json
{
  "sinks": [
    {
      "sink_id": "SINK-001",
      "type": "sql_injection",
      "function": "db.query",
      "file": "routes/users.js",
      "line": 25,
      "code_snippet": "db.query(`SELECT * FROM users WHERE id = ${userId}`)",
      "risk_level": "HIGH"
    }
  ],
  "statistics": {
    "total_sinks": 28,
    "by_type": {"sql_injection": 12, "xss": 8, "command_injection": 5, "path_traversal": 3}
  }
}
```

#### 1.3 Security Asset Scanner

Identify sensitive configurations, credentials, and security-relevant code.

```bash
# Search for hardcoded credentials
grep -rn "password\s*=\|api_key\s*=\|secret\s*=" PROJECT_PATH --include="*.js" --include="*.py" --include="*.java" --include="*.php"

# Find .env files and config files
find PROJECT_PATH -name ".env*" -o -name "config.json" -o -name "application.properties"

# Search for authentication/authorization code
grep -rn "authenticate\|authorize\|checkPermission\|isAdmin" PROJECT_PATH
```

Output: `.workspace/code-audit/phase1_discovery/security_assets.json`

#### 1.4 Data Model Analyzer

Analyze database schemas and data relationships.

```bash
# For Sequelize (Node.js)
grep -rn "sequelize\.define\|model\.associate" PROJECT_PATH --include="*.js"

# For Django (Python)
grep -rn "models\.Model\|class.*Model:" PROJECT_PATH --include="*.py"

# For JPA (Java)
grep -rn "@Entity\|@Table" PROJECT_PATH --include="*.java"
```

Output: `.workspace/code-audit/phase1_discovery/data_models.json`

**After all 4 tasks complete:**

```
[1/7] ✅ Asset Discovery Complete
  - Found 45 HTTP endpoints
  - Found 28 dangerous sink points
  - Found 12 security assets
  - Analyzed 8 data models
```

### Stage 1.5: Entry Function Analysis (入口功能分析)

**Purpose**: Deep analysis of each HTTP endpoint's business logic.

For each entry point discovered in Stage 1, analyze:

1. **Parameter handling**: How are parameters extracted, validated, and used?
2. **Business logic**: What does this endpoint do?
3. **Return values**: What data is returned to the user?
4. **Middleware chain**: What global middleware affects this endpoint?

Use `Read` tool to examine each handler function in detail.

Output: `.workspace/code-audit/phase1_5_entry_analysis/entry_functions.json`

```json
{
  "entry_functions": [
    {
      "entry_id": "ENTRY-001",
      "function_name": "getUserById",
      "file": "routes/users.js",
      "parameters": {
        "id": {
          "source": "req.params.id",
          "validation": "none",
          "sanitization": "none"
        }
      },
      "business_logic": "Fetch user by ID and return user details",
      "database_operations": ["User.findByPk(id)"],
      "return_data": ["user.email", "user.name", "user.role"],
      "middleware": ["authenticate", "rateLimit"]
    }
  ]
}
```

Progress: `[2/7] ✅ Entry Function Analysis Complete`

### Stage 2: Technical Vulnerability Audit - Backward Taint Analysis (反向污点追踪)

**Purpose**: Trace data flow from dangerous sinks back to user-controlled sources.

Execute the following sub-stages **sequentially**:

#### 2.1 Backward Dataflow Tracing

For each sink point, trace backwards to find:
- Source of tainted data (HTTP parameters, file uploads, etc.)
- Intermediate transformations
- Filtering/sanitization operations

**Algorithm:**

```python
def trace_backward(sink_location):
    """
    Trace from sink back to source
    """
    call_chain = []
    current = sink_location
    
    while current:
        # Read code at current location
        code = read_code(current.file, current.line)
        
        # Parse variable dependencies
        variables = extract_variables(code)
        
        # For each variable, find its definition
        for var in variables:
            definition = find_definition(var, current.file, current.line)
            
            # Check if this is a source (user input)
            if is_source(definition):
                call_chain.append({
                    "type": "SOURCE",
                    "location": definition.location,
                    "variable": var
                })
                break
            
            # Check if this is a filter
            if is_filter(definition):
                call_chain.append({
                    "type": "FILTER",
                    "location": definition.location,
                    "operation": extract_filter_operation(definition)
                })
            
            # Continue tracing backwards
            current = definition.location
    
    return call_chain
```

Use `Grep` tool to find variable definitions and `Read` tool to analyze code context.

Output: `.workspace/code-audit/phase2_technical_audit/traces.json`

```json
{
  "traces": [
    {
      "trace_id": "TRACE-001",
      "sink": "SINK-001",
      "source": "req.params.id",
      "call_chain": [
        {
          "step": 1,
          "location": "routes/users.js:20",
          "code": "const userId = req.params.id",
          "type": "SOURCE",
          "tainted": true
        },
        {
          "step": 2,
          "location": "routes/users.js:25",
          "code": "db.query(`SELECT * FROM users WHERE id = ${userId}`)",
          "type": "SINK",
          "tainted": true
        }
      ],
      "filters": [],
      "preliminary_vulnerable": true
    }
  ]
}
```

#### 2.2 Vulnerability Validation

For each trace, validate if it's truly exploitable:

**Validation checks:**

1. **Filter effectiveness**: Are filters adequate?
   - SQL injection: Check for parameterized queries, escaping
   - XSS: Check for HTML encoding, Content-Security-Policy
   - Command injection: Check for argument validation

2. **Type coercion**: Can attacker bypass filters via type confusion?

3. **Context analysis**: Is the vulnerability exploitable in the actual usage context?

Output: `.workspace/code-audit/phase2_technical_audit/validated_vulns.json`

```json
{
  "vulnerabilities": [
    {
      "vuln_id": "VULN-001",
      "trace_id": "TRACE-001",
      "vuln_type": "sql_injection",
      "severity": "CRITICAL",
      "confidence": "HIGH",
      "exploitable": true,
      "validation_notes": "No parameterization, no escaping. Direct string interpolation makes this exploitable.",
      "cwe": "CWE-89"
    }
  ]
}
```

#### 2.3 Vulnerability Correlation

Search for similar vulnerability patterns across the codebase.

```bash
# If VULN-001 uses pattern: db.query(`...${variable}`)
# Search for all similar patterns
grep -rn "db\.query.*\${" PROJECT_PATH --include="*.js"
```

Output: `.workspace/code-audit/phase2_technical_audit/correlated_vulns.json`

#### 2.4 PoC Generation

For each validated vulnerability, generate a proof-of-concept exploit.

Output: `.workspace/code-audit/phase2_technical_audit/pocs/VULN-001_poc.txt`

```
# SQL Injection PoC for VULN-001

## Vulnerable Endpoint
GET /api/users/:id

## Vulnerable Code
File: routes/users.js:25
Code: db.query(`SELECT * FROM users WHERE id = ${userId}`)

## Exploit
GET /api/users/1' OR '1'='1

## Expected Result
Returns all users in the database instead of just user with id=1

## Impact
- Complete database disclosure
- Potential data manipulation
- Possible RCE via xp_cmdshell (if MSSQL)
```

Progress: `[3/7] ✅ Technical Vulnerability Audit Complete - Found 8 vulnerabilities`

### Stage 2.5: Forward Taint Analysis (正向污点追踪)

**Purpose**: Cross-validate backward findings by tracing forward from sources to sinks.

**Algorithm:**

```python
def trace_forward(source_location):
    """
    Trace from source forward to potential sinks
    """
    call_chain = []
    current = source_location
    visited = set()
    
    queue = [current]
    
    while queue:
        current = queue.pop(0)
        
        if current in visited:
            continue
        visited.add(current)
        
        # Read code and find all uses of this variable
        uses = find_variable_uses(current.variable, current.file)
        
        for use in uses:
            # Check if this use is a sink
            if is_sink(use):
                call_chain.append({
                    "source": source_location,
                    "sink": use.location,
                    "path": reconstruct_path(source_location, use.location)
                })
            
            # Otherwise, add to queue for further tracing
            queue.append(use)
    
    return call_chain
```

Output: `.workspace/code-audit/phase2_5_forward_trace/forward_traces.json`

**Compare forward and backward results:**

- Paths found by both methods → High confidence
- Paths found only by backward → Verify manually
- Paths found only by forward → Verify manually

Progress: `[4/7] ✅ Forward Taint Analysis Complete - Validated 6/8 vulnerabilities`

### Stage 3: Business Logic Audit (业务逻辑审计)

#### 3.1 Access Control Audit

Analyze authorization logic to find:

1. **IDOR (Insecure Direct Object Reference)**
   - Check if object IDs are validated against user ownership
   - Pattern: `User.findById(id)` without ownership check

2. **Vertical Privilege Escalation**
   - Check if admin functions verify admin role
   - Pattern: Missing `checkAdmin()` middleware

3. **Horizontal Privilege Escalation**
   - Check if users can access other users' resources
   - Pattern: No comparison of `resourceOwnerId === currentUserId`

**Detection algorithm:**

```python
def check_access_control(entry_function, data_models):
    issues = []
    
    # Check for IDOR
    for db_op in entry_function.database_operations:
        if "findById" in db_op or "findByPk" in db_op:
            # Check if there's ownership validation
            if not has_ownership_check(entry_function, db_op):
                issues.append({
                    "type": "IDOR",
                    "location": entry_function.location,
                    "resource": extract_model(db_op)
                })
    
    # Check for privilege escalation
    if is_sensitive_operation(entry_function):
        if not has_role_check(entry_function):
            issues.append({
                "type": "privilege_escalation",
                "location": entry_function.location
            })
    
    return issues
```

Output: `.workspace/code-audit/phase3_business_audit/access_control.json`

#### 3.2 Business Logic Audit

Analyze business workflows to find logic flaws:

1. **Payment/Transaction Issues**
   - Race conditions in payment processing
   - Missing transaction atomicity
   - Price manipulation

2. **Workflow Bypass**
   - State machine violations
   - Missing state validation

3. **Rate Limiting Issues**
   - Missing rate limits on sensitive operations
   - Bypassable rate limits

Output: `.workspace/code-audit/phase3_business_audit/business_logic.json`

Progress: `[5/7] ✅ Business Logic Audit Complete - Found 4 business logic flaws`

### Stage 4: Attack Chain Construction (攻击链分析)

**Purpose**: Combine individual vulnerabilities into complete attack scenarios.

**Attack chain patterns:**

1. **Authentication Bypass → Privilege Escalation → Data Exfiltration**
2. **IDOR → Sensitive Data Access**
3. **SQL Injection → RCE via File Write**
4. **XSS → Session Hijacking → Account Takeover**

**Construction algorithm:**

```python
def build_attack_chains(technical_vulns, business_logic_vulns):
    chains = []
    
    # Pattern: Auth bypass + Privilege escalation
    auth_bypass = find_vulns_by_type(business_logic_vulns, "auth_bypass")
    priv_esc = find_vulns_by_type(business_logic_vulns, "privilege_escalation")
    
    for ab in auth_bypass:
        for pe in priv_esc:
            if can_chain(ab, pe):
                chains.append({
                    "name": "Authentication Bypass to Admin Access",
                    "steps": [ab, pe],
                    "impact": "Complete system compromise"
                })
    
    # Pattern: IDOR + Data access
    idor = find_vulns_by_type(business_logic_vulns, "IDOR")
    
    for i in idor:
        chains.append({
            "name": "Direct Object Reference to Sensitive Data",
            "steps": [i],
            "impact": "Unauthorized access to user data"
        })
    
    # Pattern: SQL Injection + File Write
    sqli = find_vulns_by_type(technical_vulns, "sql_injection")
    
    for s in sqli:
        if database_supports_file_write(s):
            chains.append({
                "name": "SQL Injection to Remote Code Execution",
                "steps": [
                    s,
                    {"type": "file_write", "method": "INTO OUTFILE"},
                    {"type": "rce", "method": "Webshell execution"}
                ],
                "impact": "Complete server compromise"
            })
    
    return chains
```

Output: `.workspace/code-audit/phase4_attack_chains/attack_chains.json`

```json
{
  "attack_chains": [
    {
      "chain_id": "CHAIN-001",
      "name": "SQL Injection to RCE",
      "severity": "CRITICAL",
      "steps": [
        {
          "step": 1,
          "vuln_id": "VULN-001",
          "description": "Exploit SQL injection in /api/users/:id",
          "payload": "1' UNION SELECT '<?php system($_GET[\"cmd\"]); ?>' INTO OUTFILE '/var/www/html/shell.php'--"
        },
        {
          "step": 2,
          "description": "Access webshell",
          "payload": "GET /shell.php?cmd=whoami"
        }
      ],
      "impact": "Complete server compromise, data exfiltration, lateral movement",
      "likelihood": "HIGH",
      "risk_score": 9.8
    }
  ]
}
```

Progress: `[6/7] ✅ Attack Chain Analysis Complete - Built 3 attack chains`

### Stage 5: Report Generation (报告生成)

**Purpose**: Generate comprehensive audit report in Markdown and JSON formats.

Read all previous stage outputs and generate:

#### 5.1 Executive Summary

```markdown
# Code Security Audit Report

## Executive Summary

**Audit Date**: 2025-01-15
**Project**: Example Web Application
**Framework**: Express.js
**Overall Risk**: CRITICAL

**Key Findings**:
- 8 Technical Vulnerabilities (3 Critical, 3 High, 2 Medium)
- 4 Business Logic Flaws (2 High, 2 Medium)
- 3 Exploitable Attack Chains

**Immediate Actions Required**:
1. Fix SQL injection vulnerabilities (VULN-001, VULN-003, VULN-005)
2. Implement access control checks for IDOR vulnerabilities
3. Add rate limiting to prevent brute force attacks
```

#### 5.2 Detailed Findings

For each vulnerability:
- Description
- Location (file:line)
- Severity and CWE classification
- Proof of Concept
- Remediation steps

#### 5.3 Remediation Guidance

Prioritized list of fixes with code examples.

Output files:
- `.workspace/code-audit/report.md` (Human-readable)
- `.workspace/code-audit/report.json` (Machine-readable)

Progress: `[7/7] ✅ Audit Complete! Report generated.`

## Output Structure

```
.workspace/code-audit/
├── phase1_discovery/
│   ├── web_entries.json
│   ├── sink_points.json
│   ├── security_assets.json
│   └── data_models.json
├── phase1_5_entry_analysis/
│   └── entry_functions.json
├── phase2_technical_audit/
│   ├── traces.json
│   ├── validated_vulns.json
│   ├── correlated_vulns.json
│   └── pocs/
│       ├── VULN-001_poc.txt
│       └── ...
├── phase2_5_forward_trace/
│   └── forward_traces.json
├── phase3_business_audit/
│   ├── access_control.json
│   └── business_logic.json
├── phase4_attack_chains/
│   └── attack_chains.json
├── report.md
└── report.json
```

## Supported Vulnerability Types

| CWE | Type | Description |
|-----|------|-------------|
| CWE-89 | SQL Injection | Unvalidated input in SQL queries |
| CWE-79 | XSS | Unescaped output in HTML/JS |
| CWE-918 | SSRF | Server-Side Request Forgery |
| CWE-78 | Command Injection | OS command execution |
| CWE-22 | Path Traversal | File system access |
| CWE-639 | IDOR | Insecure Direct Object Reference |
| CWE-284 | Access Control | Privilege escalation |

## Supported Frameworks

| Language | Frameworks |
|----------|------------|
| JavaScript/TypeScript | Express, Koa, NestJS |
| Python | Flask, Django, FastAPI |
| Java | Spring Boot, Play Framework |
| PHP | Laravel, Symfony |

## Important Notes

1. **Sequential Execution**: Stages must be executed in order as each stage depends on previous outputs
2. **Thoroughness**: Do not skip any stage unless explicitly instructed
3. **Progress Reporting**: Keep user informed with progress updates
4. **Intermediate Results**: Save all intermediate JSON files for audit trail
5. **Cross-Validation**: Use both backward and forward taint analysis for higher confidence

## Advanced Features

### Custom Sink/Source Rules

If the project uses custom or uncommon patterns, you can add entries to:
- `rules/sources/LANGUAGE.json` for user input sources
- `rules/sinks/LANGUAGE.json` for dangerous operations

### Framework-Specific Analysis

Reference files in `references/` for framework-specific security patterns:
- `references/express-security.md` - Express.js security best practices
- `references/django-security.md` - Django security patterns
- `references/spring-security.md` - Spring Boot security

## Troubleshooting

**Issue**: Cannot detect framework
- **Solution**: Manually specify framework in prompt: `/code-audit /path --framework express`

**Issue**: Too many false positives
- **Solution**: Adjust validation strictness in Stage 2.2

**Issue**: Missing vulnerabilities
- **Solution**: Check if sink/source patterns cover the project's code patterns
