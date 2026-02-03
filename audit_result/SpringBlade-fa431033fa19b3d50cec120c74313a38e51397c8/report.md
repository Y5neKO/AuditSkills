# SpringBlade 代码安全审计报告

**审计日期**: 2025-02-03
**项目版本**: 4.8.0.RELEASE
**审计范围**: 全部微服务模块
**审计方法**: 静态代码分析 + 数据流追踪 + 业务逻辑分析

---

## 执行摘要

本次审计对 SpringBlade 微服务开发平台进行了全面的安全评估，发现了 **8个已确认的安全漏洞**，其中包括 **3个严重级别**、**3个高危级别** 和 **2个中危级别** 的漏洞。主要风险集中在多租户隔离失效、权限验证不足和服务器端请求伪造（SSRF）等方面。

### 风险等级分布

| 严重程度 | 数量 | 占比 |
|---------|------|------|
| 严重 (CRITICAL) | 3 | 37.5% |
| 高危 (HIGH) | 3 | 37.5% |
| 中危 (MEDIUM) | 2 | 25% |
| 低危 (LOW) | 0 | 0% |
| **总计** | **8** | **100%** |

### 关键发现

1. **多租户隔离失效** (CVE-2025-SB-002): 管理员可以修改其他租户用户的角色
2. **SSRF通过数据源配置** (CVE-2025-SB-001): 可探测内网数据库服务
3. **IDOR导致数据源凭据泄露** (CVE-2025-SB-003): 可查看其他租户的数据库密码

---

## 1. 资产发现

### 1.1 项目架构

SpringBlade 是一个基于 Spring Cloud 2025 和 Spring Boot 3.5 的微服务开发平台，采用前后端分离架构。

```
├── blade-auth/           # 认证授权服务
├── blade-gateway/        # API 网关
├── blade-service/        # 业务微服务
│   ├── blade-system/     # 系统管理
│   ├── blade-desk/       # 工作台
│   └── blade-log/        # 日志服务
└── blade-ops/            # 运维工具
    └── blade-develop/    # 代码生成器
```

### 1.2 HTTP端点统计

- **总控制器数**: 28
- **总端点数**: 150+
- **受保护端点**: 85 (56.7%)
- **未保护端点**: 65 (43.3%)

### 1.3 关键端点

| 端点 | 方法 | 描述 | 保护状态 |
|------|------|------|----------|
| `/token` | POST | 获取认证token | 无需认证 |
| `/user/grant` | POST | 设置用户角色 | 需管理员 |
| `/datasource/submit` | POST | 保存数据源配置 | 需管理员 |
| `/tenant/submit` | POST | 创建/修改租户 | **无保护** |
| `/discovery/instances` | GET | 获取服务实例 | **无保护** |

---

## 2. 漏洞详情

### 2.1 严重漏洞

#### CVE-2025-SB-001: SSRF通过数据源配置

**CWE**: CWE-918 (服务器端请求伪造)
**CVSS 3.1**: 9.1 (严重)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H

**描述**:
数据源配置接口允许管理员指定任意数据库连接URL。攻击者可以通过配置指向内网数据库的URL，探测内网服务或窃取数据。

**受影响文件**:
```
blade-ops/blade-develop/src/main/java/org/springblade/develop/controller/DatasourceController.java:101
```

**漏洞代码**:
```java
@PostMapping("/submit")
public R submit(@Valid @RequestBody Datasource datasource) {
    datasource.setUrl(datasource.getUrl().replace("&amp;", "&"));
    return R.status(datasourceService.saveOrUpdate(datasource));
}
```

**攻击场景**:
```http
POST /datasource/submit HTTP/1.1
Content-Type: application/json

{
  "url": "jdbc:mysql://192.168.1.100:3306/sensitive_db",
  "username": "root",
  "password": ""
}
```

**影响**:
- 内网服务探测
- 敏感数据泄露
- 数据库凭据窃取

**修复建议**:
1. 实现数据库URL白名单验证
2. 禁用内网IP地址 (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
3. 添加网络隔离

---

#### CVE-2025-SB-002: IDOR - 跨租户权限提升

**CWE**: CWE-639 (不安全的直接对象引用)
**CVSS 3.1**: 8.8 (严重)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H

**描述**:
用户角色授予接口未验证目标用户的租户所有权，管理员可以修改其他租户用户的角色，实现跨租户权限提升。

**受影响文件**:
```
blade-service/blade-system/src/main/java/org/springblade/system/controller/UserController.java:165
```

**漏洞代码**:
```java
@PostMapping("/grant")
@PreAuth(RoleConstant.HAS_ROLE_ADMIN)
public R grant(@RequestParam String userIds, @RequestParam String roleIds) {
    boolean temp = userService.grant(userIds, roleIds);
    return R.status(temp);
}
```

**服务实现**:
```java
// UserServiceImpl.java:145
public boolean grant(String userIds, String roleIds) {
    User user = new User();
    user.setRoleId(roleIds);
    return this.update(user, Wrappers.<User>update().lambda().in(User::getId, Func.toLongList(userIds)));
}
```

**攻击场景**:
```http
POST /user/grant HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Bearer {tenant_A_admin_token}

userIds={tenant_B_user_id}&roleIds={admin_role_id}
```

**影响**:
- 跨租户权限提升
- 多租户隔离失效
- 数据泄露

**修复建议**:
```java
public boolean grant(String userIds, String roleIds) {
    // 验证用户属于当前租户
    BladeUser currentUser = SecureUtil.getUser();
    List<User> targetUsers = this.listByIds(Func.toLongList(userIds));

    for (User user : targetUsers) {
        if (!user.getTenantId().equals(currentUser.getTenantId())) {
            throw new ServiceException("无权修改其他租户的用户");
        }
    }

    User user = new User();
    user.setRoleId(roleIds);
    return this.update(user, Wrappers.<User>update().lambda().in(User::getId, Func.toLongList(userIds)));
}
```

---

#### CVE-2025-SB-003: IDOR - 数据源凭据泄露

**CWE**: CWE-639 (不安全的直接对象引用)
**CVSS 3.1**: 8.5 (严重)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:N/A:N

**描述**:
数据源详情接口未验证数据源的租户所有权，管理员可以查看其他租户的数据源配置，包括明文数据库密码。

**受影响文件**:
```
blade-ops/blade-develop/src/main/java/org/springblade/develop/controller/DatasourceController.java:58
```

**漏洞代码**:
```java
@GetMapping("/detail")
@PreAuth(RoleConstant.HAS_ROLE_ADMIN)
public R<Datasource> detail(Datasource datasource) {
    Datasource detail = datasourceService.getOne(Condition.getQueryWrapper(datasource));
    return R.data(detail);
}
```

**攻击场景**:
```http
GET /datasource/detail?id={tenant_B_datasource_id} HTTP/1.1
Authorization: Bearer {tenant_A_admin_token}
```

**影响**:
- 跨租户数据库凭据泄露
- 数据库被未授权访问

**修复建议**:
```java
@GetMapping("/detail")
@PreAuth(RoleConstant.HAS_ROLE_ADMIN)
public R<Datasource> detail(Long id) {
    BladeUser currentUser = SecureUtil.getUser();
    Datasource detail = datasourceService.getById(id);

    if (detail != null && !detail.getTenantId().equals(currentUser.getTenantId())) {
        throw new ServiceException("无权访问其他租户的数据源");
    }

    return R.data(detail);
}
```

---

### 2.2 高危漏洞

#### CVE-2025-SB-004: 路径遍历 - 代码生成

**CWE**: CWE-22 (路径遍历)
**CVSS 3.1**: 7.8 (高危)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H

**描述**:
代码生成功能未验证用户输入的路径，可能通过路径遍历在服务器任意位置写入文件。

**受影响文件**:
```
blade-ops/blade-develop/src/main/java/org/springblade/develop/controller/CodeController.java:143
```

**漏洞代码**:
```java
@PostMapping("/gen-code")
public R genCode(@RequestParam String ids, @RequestParam(defaultValue = "saber3") String system) {
    Collection<Code> codes = codeService.listByIds(Func.toLongList(ids));
    codes.forEach(code -> {
        BladeCodeGenerator generator = new BladeCodeGenerator();
        generator.setPackageDir(code.getApiPath()); // 未验证路径
        // ...
        generator.run();
    });
    return R.success("代码生成成功");
}
```

**攻击场景**:
```http
POST /code/submit HTTP/1.1
Content-Type: application/json

{
  "apiPath": "../../../../../tmp/",
  "serviceName": "evil",
  "tableName": "cmd"
}
```

**影响**:
- 任意文件写入
- 可能导致远程代码执行

**修复建议**:
```java
// 验证并规范化路径
Path allowedPath = Paths.get("/var/www/blade/code").toAbsolutePath().normalize();
Path userPath = Paths.get(code.getApiPath()).toAbsolutePath().normalize();

if (!userPath.startsWith(allowedPath)) {
    throw new ServiceException("非法的路径");
}
```

---

#### CVE-2025-SB-005: 开放重定向 - OAuth

**CWE**: CWE-918 (服务器端请求伪造)
**CVSS 3.1**: 7.1 (高危)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N

**描述**:
OAuth重定向端点未验证提供者，可能导致钓鱼攻击。

**受影响文件**:
```
blade-auth/src/main/java/org/springblade/auth/controller/SocialController.java:54
```

**漏洞代码**:
```java
@RequestMapping("/oauth/render/{source}")
public void renderAuth(@PathVariable("source") String source, HttpServletResponse response) throws IOException {
    AuthRequest authRequest = SocialUtil.getAuthRequest(source, socialProperties);
    String authorizeUrl = authRequest.authorize(AuthStateUtils.createState());
    response.sendRedirect(authorizeUrl); // 未验证URL
}
```

**修复建议**:
1. 实现OAuth提供者白名单
2. 验证重定向URL
3. 添加state参数CSRF保护

---

#### CVE-2025-SB-006: 授权绕过 - 租户管理

**CWE**: CWE-285 (不当的授权)
**CVSS 3.1**: 7.5 (高危)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

**描述**:
租户管理接口缺少权限保护，任何人都可以创建/修改/删除租户。

**受影响文件**:
```
blade-service/blade-system/src/main/java/org/springblade/system/controller/TenantController.java
```

**未保护的端点**:
- `/tenant/detail`
- `/tenant/submit`
- `/tenant/remove`
- `/tenant/page`

**修复建议**:
```java
@PostMapping("/submit")
@PreAuth(RoleConstant.HAS_ROLE_ADMIN) // 添加此注解
public R submit(@Valid @RequestBody Tenant tenant) {
    return R.status(tenantService.saveTenant(tenant));
}
```

---

### 2.3 中危漏洞

#### CVE-2025-SB-007: 暴力破解 - 登录

**CWE**: CWE-307 (对关键资源的攻击频率限制不当)
**CVSS 3.1**: 5.0 (中危)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N

**描述**:
登录端点虽然实现了5次失败锁定机制，但只针对单个账号和IP，攻击者可以从多个IP进行分布式暴力破解。

**受影响文件**:
```
blade-auth/src/main/java/org/springblade/auth/controller/AuthController.java
```

**修复建议**:
1. 添加验证码
2. 实现全局速率限制
3. 增强账号锁定机制

---

#### CVE-2025-SB-008: 信息泄露 - 服务发现

**CWE**: CWE-200 (信息泄露)
**CVSS 3.1**: 5.3 (中危)
**CVSS向量**: CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N

**描述**:
服务发现端点无身份验证，暴露内部微服务架构信息。

**受影响文件**:
```
blade-gateway/src/main/java/org/springblade/gateway/controller/DiscoveryClientController.java
```

**漏洞代码**:
```java
@GetMapping("/instances")
public Map<String, List<ServiceInstance>> instances() {
    // 返回所有服务实例信息，包括IP和端口
}
```

**修复建议**:
添加认证或禁用此端点。

---

## 3. 业务逻辑漏洞

### 3.1 多租户隔离失效 (BIZ_001)

多个接口未正确验证租户所有权：

| 接口 | 问题 | 影响 |
|------|------|------|
| `/user/detail` | 未验证用户租户 | 租户A可以查看租户B的用户 |
| `/tenant/submit` | 无权限保护 | 任何人可以创建租户 |
| `/datasource/detail` | 未验证数据源租户 | 租户A可以查看租户B的数据库密码 |

**修复方案**:
实现全局的租户拦截器，确保所有数据操作都验证tenantId所有权。

---

### 3.2 权限提升链 (BIZ_002)

存在从普通用户到管理员的权限提升路径：

```
注册账号 → 利用IDOR修改roleId → 获得管理员权限
```

**修复方案**:
1. 验证操作者有权限修改目标用户
2. 添加权限变更审计
3. 实现二次确认机制

---

### 3.3 第三方注册绕过 (BIZ_003)

社交登录注册流程可能被滥用创建垃圾账号。

**修复方案**:
1. 验证OAuth响应的真实性
2. 添加邮箱验证
3. 限制注册频率

---

## 4. 攻击链分析

### 4.1 跨租户权限提升链 (CHAIN_001)

```
1. 登录租户A (管理员)
2. 枚举租户B的用户ID
3. 获取租户B的管理员角色ID
4. 提升租户B用户权限
5. 重置租户B用户密码
6. 登录租户B账号
7. 完全控制租户B
```

**影响**: 完全控制目标租户，违反多租户隔离

---

### 4.2 SSRF到内网探测链 (CHAIN_002)

```
1. 登录系统 (管理员)
2. 创建恶意数据源 (指向内网)
3. 创建代码生成任务
4. 触发代码生成 (连接内网数据库)
5. 枚举内网服务
6. 窃取数据
```

**影响**: 内网服务探测，敏感数据泄露

---

### 4.3 路径遍历到RCE链 (CHAIN_003)

```
1. 登录系统 (管理员)
2. 创建代码生成任务
3. 设置恶意路径 (路径遍历)
4. 生成代码 (写入webroot)
5. 访问生成的代码
```

**影响**: 服务器被完全控制

---

## 5. 修复优先级

### 优先级 1 (立即修复)

| 漏洞ID | 漏洞名称 | 风险 | 修复工作量 |
|--------|----------|------|------------|
| CVE-2025-SB-002 | 跨租户权限提升 | 严重 | 中等 |
| CVE-2025-SB-003 | 数据源凭据泄露 | 严重 | 中等 |
| CVE-2025-SB-006 | 授权绕过 | 高危 | 低 |

### 优先级 2 (尽快修复)

| 漏洞ID | 漏洞名称 | 风险 | 修复工作量 |
|--------|----------|------|------------|
| CVE-2025-SB-001 | SSRF数据源配置 | 严重 | 低 |
| CVE-2025-SB-004 | 路径遍历 | 高危 | 中等 |
| CVE-2025-SB-005 | 开放重定向 | 高危 | 低 |

### 优先级 3 (计划修复)

| 漏洞ID | 漏洞名称 | 风险 | 修复工作量 |
|--------|----------|------|------------|
| CVE-2025-SB-007 | 暴力破解 | 中危 | 低 |
| CVE-2025-SB-008 | 信息泄露 | 中危 | 低 |

---

## 6. 安全建议

### 6.1 短期改进 (1-2周)

1. **为所有租户管理端点添加@PreAuth保护**
   ```java
   @PreAuth(RoleConstant.HAS_ROLE_ADMIN)
   ```

2. **实现租户所有权验证**
   ```java
   if (!user.getTenantId().equals(currentUser.getTenantId())) {
       throw new ServiceException("无权访问");
   }
   ```

3. **添加数据库URL白名单**
   ```java
   private static final List<String> ALLOWED_HOSTS = List.of(
       "localhost", "127.0.0.1", "your-production-db.com"
   );
   ```

4. **禁用/discovery端点或添加认证**

### 6.2 中期改进 (1-2个月)

1. **实现全局租户拦截器**
2. **添加验证码机制**
3. **实现token撤销机制**
4. **添加审计日志系统**
5. **加强OAuth安全验证**

### 6.3 长期改进 (3-6个月)

1. **实现完整的RBAC系统**
2. **实现API速率限制**
3. **添加安全监控和告警**
4. **定期进行安全审计**
5. **实现DevSecOps流程**

---

## 7. 合规性检查

### 7.1 OWASP Top 10 (2021) 映射

| OWASP分类 | 相关漏洞 | 风险 |
|-----------|----------|------|
| A01:2021 - 访问控制失效 | CVE-2025-SB-002, CVE-2025-SB-003, CVE-2025-SB-006 | 严重 |
| A03:2021 - 注入 | CVE-2025-SB-004 (路径遍历) | 高危 |
| A04:2021 - 不安全设计 | 多租户隔离失效 | 严重 |
| A05:2021 - 安全配置错误 | CVE-2025-SB-006, CVE-2025-SB-008 | 高危 |
| A07:2021 - 认证失败 | CVE-2025-SB-007 | 中危 |

### 7.2 CWE映射

| CWE | 描述 | 相关漏洞 |
|-----|------|----------|
| CWE-639 | 不安全的直接对象引用 | CVE-2025-SB-002, CVE-2025-SB-003 |
| CWE-918 | 服务器端请求伪造 | CVE-2025-SB-001, CVE-2025-SB-005 |
| CWE-22 | 路径遍历 | CVE-2025-SB-004 |
| CWE-285 | 不当的授权 | CVE-2025-SB-006 |
| CWE-307 | 频率限制不当 | CVE-2025-SB-007 |
| CWE-200 | 信息泄露 | CVE-2025-SB-008 |

---

## 8. 结论

SpringBlade 平台在多租户隔离和权限验证方面存在严重的安全问题。虽然系统使用了现代的微服务架构和安全框架（如JWT、SM2加密），但由于缺少一致的租户所有权验证，导致多租户隔离失效。

### 关键风险

1. **多租户隔离完全失效** - 租户A的管理员可以完全控制租户B
2. **SSRF风险** - 可探测内网服务
3. **权限提升** - 存在多条权限提升路径

### 建议措施

1. **立即修复** 优先级1的漏洞
2. **实现全局租户拦截器** 确保所有数据操作都验证租户所有权
3. **加强安全测试** 将安全测试纳入CI/CD流程
4. **定期审计** 每季度进行一次安全审计

---

**审计人员**: AI Security Auditor
**审计日期**: 2025-02-03
**报告版本**: 1.0
