# 代码安全审计报告

## 审计概要

| 项目 | 信息 |
|------|------|
| 审计时间 | ${TIMESTAMP} |
| 项目路径 | ${PROJECT_PATH} |
| 编程语言 | ${LANGUAGE} |
| Web框架 | ${FRAMEWORK} |
| 审计模式 | ${AUDIT_MODE} |
| 审计时长 | ${AUDIT_DURATION} |

## 漏洞统计

| 严重程度 | 数量 |
|----------|------|
| CRITICAL | ${CRITICAL_COUNT} |
| HIGH | ${HIGH_COUNT} |
| MEDIUM | ${MEDIUM_COUNT} |
| LOW | ${LOW_COUNT} |
| **总计** | ${TOTAL_COUNT} |

---

${VULNERABILITIES_SECTION}

---

## 攻击链分析

${ATTACK_CHAINS_SECTION}

---

## 修复建议

### 通用建议

1. **参数化查询**: 所有数据库查询使用参数化查询或ORM
2. **输入验证**: 对所有用户输入进行验证和净化
3. **输出编码**: 根据上下文对输出进行适当编码
4. **认证授权**: 实施强认证和细粒度授权
5. **安全头部**: 添加安全相关HTTP头部

### 优先级

1. **立即修复**: CRITICAL级别漏洞
2. **尽快修复**: HIGH级别漏洞
3. **计划修复**: MEDIUM级别漏洞
4. **可选修复**: LOW级别漏洞

---

## 附录

### 漏洞类型说明

- **SQL注入**: 用户输入影响SQL查询结构
- **XSS**: 用户输入在HTML上下文中未转义
- **SSRF**: 用户输入控制URL请求
- **命令注入**: 用户输入进入系统命令
- **路径遍历**: 用户输入控制文件路径
- **反序列化**: 不可信数据反序列化

### 参考资源

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CWE Top 25](https://cwe.mitre.org/top25/)
- [代码安全最佳实践](https://cheatsheetseries.owasp.org/)

---

*本报告由 AI Code Audit Tool 自动生成*
*报告生成时间: ${GENERATION_TIME}*
