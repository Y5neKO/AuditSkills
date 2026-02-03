---
name: code-audit
description: Comprehensive AI-powered code security audit tool for web applications. Performs complete security assessment including asset discovery, technical vulnerability detection, business logic flaws, and attack chain construction. Supports Node.js, Python, Java, and PHP. Use when the user requests code review, security audit, vulnerability assessment, or penetration testing of source code.
---

# Code Audit - AI代码安全审计工具

## ⚠️ CRITICAL EXECUTION REQUIREMENTS

**YOU MUST FOLLOW EVERY STEP IN ORDER. DO NOT SKIP ANY STAGE.**

Each stage has **MANDATORY** checkpoints that MUST be completed before proceeding.

---

## Stage 0: Preparation [MANDATORY]

### Checkpoint 0.1: Detect Project Framework

```bash
# Execute this command FIRST
find PROJECT_PATH -name "package.json" -o -name "pom.xml" -o -name "requirements.txt" -o -name "composer.json" | head -5
```

**REQUIRED OUTPUT**: Identify framework type (Express/Spring/Django/Laravel)

### Checkpoint 0.2: Load Detection Rules [MANDATORY]

**YOU MUST READ THESE FILES BEFORE ANY ANALYSIS:**

Based on detected language, read the appropriate rule files:

**For Java projects:**
```
REQUIRED: Read rules/sinks/java.json
REQUIRED: Read rules/sources/java.json
```

**For JavaScript projects:**
```
REQUIRED: Read rules/sinks/javascript.json
REQUIRED: Read rules/sources/javascript.json
```

**For Python projects:**
```
REQUIRED: Read rules/sinks/python.json
REQUIRED: Read rules/sources/python.json
```

**For PHP projects:**
```
REQUIRED: Read rules/sinks/php.json
REQUIRED: Read rules/sources/php.json
```

**VERIFICATION**: Confirm you have loaded the rules by listing how many sink patterns and source patterns were loaded.

Example verification output:
```
✅ Loaded 45 sink patterns for Java
✅ Loaded 28 source patterns for Java
```

### Checkpoint 0.3: Create Workspace

```bash
mkdir -p .workspace/code-audit/{phase1_discovery,phase1_5_entry_analysis,phase2_technical_audit,phase2_5_forward_trace,phase3_business_audit,phase4_attack_chains}
```

**REQUIRED**: Confirm workspace created

---

## Stage 1: Asset Discovery [MANDATORY - ALL 4 TASKS]

**DO NOT PROCEED TO STAGE 2 UNTIL ALL 4 JSON FILES ARE CREATED**

### Task 1.1: Web Entry Discovery [MANDATORY]

**REQUIRED ACTIONS:**

1. **Search for HTTP endpoints** using framework-specific patterns from `rules/sources/LANGUAGE.json`

   For Java/Spring Boot:
   ```bash
   grep -rn "@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping\|@RequestMapping" PROJECT_PATH --include="*.java"
   ```

2. **Parse ALL results** (do not stop after finding a few)

3. **MUST CREATE FILE**: `.workspace/code-audit/phase1_discovery/web_entries.json`

   Required format:
   ```json
   {
     "project_info": {
       "framework": "...",
       "language": "...",
       "scan_time": "..."
     },
     "entry_points": [
       {
         "entry_id": "ENTRY-001",
         "method": "GET",
         "path": "/api/...",
         "file": "path/to/file.java",
         "line": 123,
         "handler": "methodName",
         "parameters": [...]
       }
     ],
     "statistics": {
       "total_entries": 0,
       "by_method": {}
     }
   }
   ```

**VERIFICATION CHECKPOINT**: 
- File created? YES/NO
- How many entry points found? (must be > 0 for most projects)

### Task 1.2: Sink Point Scanner [MANDATORY]

**REQUIRED ACTIONS:**

1. **Load sink patterns** from `rules/sinks/LANGUAGE.json` (if not already loaded in Stage 0)

2. **Search for EACH sink pattern** from the rules file:

   Example for Java (use actual patterns from java.json):
   ```bash
   # For SQL sinks
   grep -rn "\.createQuery\|\.createNativeQuery\|\.executeQuery" PROJECT_PATH --include="*.java"
   
   # For Command Injection sinks  
   grep -rn "Runtime\.getRuntime\(\)\.exec\|ProcessBuilder" PROJECT_PATH --include="*.java"
   
   # For XXE sinks
   grep -rn "DocumentBuilder\|SAXParser\|XMLReader" PROJECT_PATH --include="*.java"
   ```

3. **MUST CREATE FILE**: `.workspace/code-audit/phase1_discovery/sink_points.json`

   Required format:
   ```json
   {
     "sinks": [
       {
         "sink_id": "SINK-001",
         "type": "sql_injection",
         "function": "createNativeQuery",
         "file": "path/to/file.java",
         "line": 456,
         "code_snippet": "...",
         "risk_level": "HIGH"
       }
     ],
     "statistics": {
       "total_sinks": 0,
       "by_type": {}
     }
   }
   ```

**VERIFICATION CHECKPOINT**:
- File created? YES/NO
- How many sinks found? (list count by type)

### Task 1.3: Security Asset Scanner [MANDATORY]

**REQUIRED ACTIONS:**

1. Search for hardcoded credentials:
   ```bash
   grep -rn "password.*=.*\"\|apiKey.*=.*\"\|secret.*=.*\"" PROJECT_PATH
   ```

2. Find configuration files:
   ```bash
   find PROJECT_PATH -name "application.properties" -o -name "application.yml" -o -name ".env"
   ```

3. **MUST CREATE FILE**: `.workspace/code-audit/phase1_discovery/security_assets.json`

**VERIFICATION CHECKPOINT**: File created? YES/NO

### Task 1.4: Data Model Analyzer [MANDATORY]

**REQUIRED ACTIONS:**

1. Find data models:
   ```bash
   grep -rn "@Entity\|@Table\|@Data" PROJECT_PATH --include="*.java"
   ```

2. **MUST CREATE FILE**: `.workspace/code-audit/phase1_discovery/data_models.json`

**VERIFICATION CHECKPOINT**: File created? YES/NO

---

## Stage 1 COMPLETION CHECKPOINT [MANDATORY]

**BEFORE PROCEEDING TO STAGE 2, VERIFY:**

- [ ] web_entries.json created with at least 1 entry point
- [ ] sink_points.json created with all sink types searched
- [ ] security_assets.json created
- [ ] data_models.json created

**OUTPUT REQUIRED SUMMARY:**
```
[1/7] ✅ Asset Discovery Complete
  - Found X HTTP endpoints
  - Found Y dangerous sink points (breakdown by type)
  - Found Z security assets
  - Analyzed W data models
```

---

## Stage 2: Technical Vulnerability Audit [MANDATORY - SEQUENTIAL]

**DO NOT SKIP ANY SUB-STAGE**

### Stage 2.1: Backward Dataflow Tracing [MANDATORY]

**FOR EACH SINK in sink_points.json, YOU MUST:**

1. **Read the file** containing the sink
2. **Identify the variable** used in the dangerous function
3. **Trace backwards** to find where it comes from:
   - Use `grep` to find all assignments to that variable
   - Read each assignment location
   - Check if it originates from user input (HTTP parameters, etc.)
4. **Record the call chain**

**MANDATORY ALGORITHM:**

```
For each sink in sink_points.json:
  1. Read sink.file at sink.line
  2. Extract variable name used in dangerous function
  3. Search backwards for variable definition:
     - grep -n "variable_name.*=" sink.file
  4. For each definition found:
     - Read that line and surrounding context
     - Check if it's from user input (req.getParameter, @RequestParam, etc.)
     - Check if there's any filtering/sanitization
  5. Build call_chain array
  6. Save to traces.json
```

**MUST CREATE FILE**: `.workspace/code-audit/phase2_technical_audit/traces.json`

Required format:
```json
{
  "traces": [
    {
      "trace_id": "TRACE-001",
      "sink_id": "SINK-001",
      "source": "req.getParameter('id')",
      "call_chain": [
        {
          "step": 1,
          "location": "file.java:20",
          "code": "String id = req.getParameter(\"id\")",
          "type": "SOURCE",
          "tainted": true
        },
        {
          "step": 2,
          "location": "file.java:25",
          "code": "query(\"SELECT * WHERE id=\" + id)",
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

**CHECKPOINT**: How many traces found? Must trace EVERY sink from Stage 1.

### Stage 2.2: Vulnerability Validation [MANDATORY]

**FOR EACH TRACE in traces.json:**

1. Check if filters are effective
2. Check for bypasses
3. Classify as TRUE POSITIVE or FALSE POSITIVE

**MUST CREATE FILE**: `.workspace/code-audit/phase2_technical_audit/validated_vulns.json`

**CHECKPOINT**: How many vulnerabilities validated as TRUE POSITIVE?

### Stage 2.3: Vulnerability Correlation [MANDATORY]

Search for similar patterns across the entire codebase.

**MUST CREATE FILE**: `.workspace/code-audit/phase2_technical_audit/correlated_vulns.json`

### Stage 2.4: PoC Generation [MANDATORY]

For each validated vulnerability, generate PoC.

**MUST CREATE FILES**: `.workspace/code-audit/phase2_technical_audit/pocs/VULN-XXX_poc.txt`

---

## Stage 2 COMPLETION CHECKPOINT [MANDATORY]

**OUTPUT REQUIRED:**
```
[3/7] ✅ Technical Vulnerability Audit Complete
  - Traced X data flows
  - Validated Y vulnerabilities (Z false positives excluded)
  - Found W correlated vulnerabilities
  - Generated V PoCs
```

---

## Stage 3: Business Logic Audit [MANDATORY]

### Stage 3.1: Access Control Audit

**REQUIRED**: Check EVERY entry point for authorization

**MUST CREATE FILE**: `.workspace/code-audit/phase3_business_audit/access_control.json`

### Stage 3.2: Business Logic Flaws

**MUST CREATE FILE**: `.workspace/code-audit/phase3_business_audit/business_logic.json`

---

## Stage 4: Attack Chain Construction [MANDATORY]

Combine vulnerabilities into attack scenarios.

**MUST CREATE FILE**: `.workspace/code-audit/phase4_attack_chains/attack_chains.json`

---

## Stage 5: Report Generation [MANDATORY]

Generate final reports.

**MUST CREATE FILES**:
- `.workspace/code-audit/report.md`
- `.workspace/code-audit/report.json`

---

## FINAL COMPLETION VERIFICATION

**YOU MUST CONFIRM ALL FILES EXIST:**

```bash
ls -la .workspace/code-audit/phase1_discovery/
ls -la .workspace/code-audit/phase2_technical_audit/
ls -la .workspace/code-audit/phase3_business_audit/
ls -la .workspace/code-audit/phase4_attack_chains/
ls -la .workspace/code-audit/report.*
```

**OUTPUT THE FINAL FILE COUNT:**
```
✅ Audit Complete
Total files created: X
  - Phase 1: 4 files
  - Phase 2: 4 files
  - Phase 3: 2 files
  - Phase 4: 1 file
  - Reports: 2 files
```

---

## Rules File Format Reference

Your sink and source rules are in JSON format. Here's how to use them:

### rules/sinks/java.json Structure:
```json
{
  "sinks": [
    {
      "name": "JPA createNativeQuery",
      "type": "sql_injection",
      "patterns": [
        "\\.createNativeQuery\\(",
        "\\.createQuery\\("
      ],
      "severity": "HIGH",
      "cwe": "CWE-89"
    }
  ]
}
```

**How to use:**
1. Read the file
2. For each sink entry, extract the `patterns` array
3. For each pattern, run grep with that pattern
4. Collect all results

### rules/sources/java.json Structure:
```json
{
  "sources": [
    {
      "name": "HTTP Request Parameter",
      "type": "user_input",
      "patterns": [
        "@RequestParam",
        "@PathVariable",
        "request\\.getParameter\\("
      ]
    }
  ]
}
```

---

## Troubleshooting

**If you find yourself skipping stages:**
- STOP immediately
- Return to the last completed checkpoint
- Complete all mandatory tasks for that stage
- Verify all required files exist
- Then proceed

**If you're unsure about a step:**
- DO NOT guess
- DO NOT skip
- Follow the exact bash commands provided
- Create the exact JSON structure shown

**Remember:**
- Quality > Speed
- Complete > Fast
- Every sink must be traced
- Every entry point must be analyzed
