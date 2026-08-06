# MateClaw 与 hblocal-cloud 集成方案（兼容 CAS）

> 本文档描述将 MateClaw（Java 21 + Spring Boot 3.5）作为独立 AI 中台服务集成到 hblocal-cloud（Java 8 + Spring Boot 2.7.13 + Spring Cloud 2021.0.8 + Nacos + Spring Cloud Gateway）微服务项目的完整方案。
>
> 方案覆盖三大目标：
> 1. **跨世代集成**：hblocal-cloud 与 MateClaw 处于不同 Java/Spring Boot 世代，必须进程级隔离、注册中心级打通；
> 2. **个人 workspace**：每个登录用户自动拥有独立的个人 workspace（slug=`user-{id}`），用户默认位于个人空间，可加入/创建团队空间；
> 3. **SSO 兼容**：通过 `SsoProvider` 抽象接入 CAS / OAuth2 / OIDC 等任意 IdP，业务层完全无感。

---

## 1. 背景与现状

### 1.1 版本差异（硬约束）

| 维度 | hblocal-cloud | MateClaw | 直接合并 |
|---|---|---|---|
| Java | 1.8 | 21 | ❌ |
| Spring Boot | 2.7.13 | 3.5.16 | ❌ |
| Spring Cloud | 2021.0.8 | 无 | ❌ |
| 注册中心 | Nacos | — | ✅ 服务端跨版本 |
| 鉴权 | Spring Cloud Gateway 统一 | JWT 自签 + Spring Security | ⚠️ 需 token 翻译 |

**结论**：MateClaw 不能被纳入 hblocal-cloud 的 Maven reactor，必须**独立部署 + Gateway 路由 + 注册中心打通**。

### 1.2 hblocal-cloud 用户表结构

`hblocal.sys_user`（RuoYi 风格）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `user_id` | VARCHAR(64) PK | 用户主键（字符串） |
| `user_name` | VARCHAR(64) | 用户名 |
| `nick_name` | VARCHAR(64) | 昵称 |
| `user_type` | VARCHAR(2) | 00=普通员工 |
| `email` | VARCHAR(64) | 邮箱 |
| `phonenumber` | VARCHAR(32) | 手机 |
| `password` | VARCHAR(100) | **BCrypt 哈希** |
| `status` | CHAR(1) | 0=停用,1=启用 |
| `del_flag` | CHAR(1) | 0=正常,1=删除 |

**关键事实**：password 已经是 BCrypt 格式，可以直接复用算法。

### 1.3 MateClaw 当前用户体系

```
mate_user (id BIGINT PK, username UNIQUE, password BCrypt, role, enabled, ...)
mate_workspace (id BIGINT PK, slug UNIQUE, owner_id, ...)
mate_workspace_member (workspace_id, user_id, role[owner/admin/member/viewer])
```

`WorkspaceAccessInterceptor` 强制要求 `X-Workspace-Id` header，缺省时默认 `workspace=1`。

---

## 2. 架构总览

```
┌──────────────────────────────────────────────────────────────┐
│ 外部用户（hblocal-cloud 员工）                            │
└────────────────────────┬─────────────────────────────────────┘
                         │ 浏览器/HTTP
┌────────────────────────▼─────────────────────────────────────┐
│ hblocal-cloud Spring Cloud Gateway (SB 2.7 + SCG 4.x)        │
│   - 鉴权：JWT/OAuth2 from hblocal 鉴权中心                  │
│   - 路由：/mateclaw/**  → lb://mateclaw-server              │
│   - 转发：X-Gateway-Authed, X-User-Id, X-Workspace-Id,        │
│           X-Gateway-Jwt (新签的含 user_id/nick_name/email)   │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTP + Headers
┌────────────────────────▼─────────────────────────────────────┐
│ MateClaw server (SB 3.5 + Java 21, 独立进程, 端口 18088)     │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Filter 链                                              │ │
│  │  1. GatewayJwtAuthFilter (新)                          │ │
│  │     - 解 X-Gateway-Jwt → 拿 user_id/nick_name/email   │ │
│  │     - 调 AuthService.findOrCreateFromGateway          │ │
│  │  2. JwtAuthFilter (原有, 兜底)                         │ │
│  │     - 自家 JWT 登录路径                                │ │
│  └─────────────────────────────────────────────────────────┘ │
│                         │                                   │
│  ┌──────────────────────▼────────────────────────────────┐ │
│  │ AuthService.findOrCreateFromGateway (新)              │ │
│  │   1. 查 mate_user_external 映射表                     │ │
│  │   2. 没找到 → INSERT mate_user (username=hb_xxx)      │ │
│  │   3. INSERT mate_user_external 映射                    │ │
│  │   4. WorkspaceService.getOrCreatePersonalWorkspace    │ │
│  │        - 查 slug=user-{id}                            │ │
│  │        - 没找到 → INSERT mate_workspace                │ │
│  │        - INSERT mate_workspace_member (role=owner)    │ │
│  └───────────────────────────────────────────────────────┘ │
│                         │                                   │
│  ┌──────────────────────▼────────────────────────────────┐ │
│  │ WorkspaceAccessInterceptor (改 resolveWorkspaceId)    │ │
│  │   - 优先 X-Workspace-Id header                         │ │
│  │   - 没传 → SecurityContext → userId → 个人 workspace  │ │
│  └───────────────────────────────────────────────────────┘ │
└─────┬──────────────────────────────────┬────────────────────┘
      │                                  │
┌─────▼────────────┐              ┌─────▼────────────────┐
│ PostgreSQL 16    │              │ Nacos (同实例)        │
│ (MateClaw 自有)  │              │ mateclaw-server 注册 │
│ - mate_user      │              │                     │
│ - mate_user_ext  │              │                     │
│ - mate_workspace │              │                     │
└──────────────────┘              └─────────────────────┘
```

### 2.1 关键设计原则

1. **用户身份与数据归属解耦**：`mate_user.username` 是身份标识，映射表存 `(external_system, external_user_id) → mate_user.id` 才是真正主键。
2. **personal workspace 是一等公民**：每个用户天然有一个，无需显式加入。
3. **IdP 无关**：业务层只接收 `(external_system, external_user_id, nick_name, email, role)` 标准化身份，不关心协议。
4. **零协议绑定**：用 `nacos-client` 裸 SDK 注册到 Nacos，避开 Spring Cloud 版本冲突。

---

## 3. 环境与版本要求

| 组件 | 版本 | 备注 |
|---|---|---|
| MateClaw | 当前 2.0.0 | Java 21, Spring Boot 3.5.16 |
| hblocal-cloud | — | Java 1.8, Spring Boot 2.7.13, Spring Cloud 2021.0.8 |
| Nacos | 1.x / 2.x | 已有，跨版本兼容 |
| PostgreSQL | 16 | MateClaw 独立数据库 |
| JDK | 21（LTS） | 仅 MateClaw 端 |
| Maven | 3.9+ | flatten-maven-plugin 1.7.3 |
| Node（前端） | 18+ | Vue 3 + Vite |

**不需要**：
- ❌ spring-cloud-starter-alibaba-nacos-discovery（版本冲突）
- ❌ spring-cloud-starter-bootstrap（3.x 移除）
- ❌ 任何 Spring Cloud BOM 引入到 MateClaw

**新增依赖**：
- `com.alibaba.nacos:nacos-client:2.4.2`（裸 SDK）

---

## 4. 数据层改动（Flyway 迁移）

### 4.1 迁移脚本示例（PostgreSQL，放在 `db/migration/postgres/V3__hblocal_integration.sql`）

`mate_workspace.slug` 已有 UNIQUE 约束（`uk_workspace_slug`），无需新增。

```sql
-- =====================================================
-- V3: hblocal-cloud 集成 + 个人 workspace 支持
-- =====================================================

-- 1. mate_user 加外部关联字段
ALTER TABLE mate_user ADD COLUMN external_system  VARCHAR(64)  COMMENT '外部系统标识, 如 hblocal-cloud';
ALTER TABLE mate_user ADD COLUMN external_user_id VARCHAR(128) COMMENT '外部系统用户ID';

-- 兼容已有 data-en.sql / data-kingbase-en.sql 等脚本
CREATE INDEX IF NOT EXISTS idx_mate_user_external
    ON mate_user(external_system, external_user_id);

-- 2. 外部用户映射表
CREATE TABLE IF NOT EXISTS mate_user_external (
    id                BIGSERIAL    PRIMARY KEY,
    external_system   VARCHAR(64)  NOT NULL,
    external_user_id  VARCHAR(128) NOT NULL,
    mate_user_id      BIGINT       NOT NULL,
    create_time       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted           INT          NOT NULL DEFAULT 0,
    CONSTRAINT uq_user_external UNIQUE (external_system, external_user_id)
);

CREATE INDEX IF NOT EXISTS idx_user_external_mate
    ON mate_user_external(mate_user_id);

-- 3. mate_workspace_member 加 (workspace_id, user_id) 唯一索引
CREATE UNIQUE INDEX IF NOT EXISTS uq_workspace_member_active
    ON mate_workspace_member(workspace_id, user_id)
    WHERE deleted = 0;
```

### 4.2 等价 SQL（其他方言）

需要为 `db/migration/h2/`、`db/migration/mysql/`、`db/migration/kingbase/` 各放一份等价脚本，关键差异：

| 方言 | BIGSERIAL | 表注释 | 局部索引 |
|---|---|---|---|
| PostgreSQL | `BIGSERIAL` | `COMMENT '...'` | `WHERE deleted = 0` |
| MySQL | `BIGINT AUTO_INCREMENT` | `COMMENT '...'` | 不支持 partial index，改为普通 unique |
| H2 (dev) | `BIGINT AUTO_INCREMENT` 或 `IDENTITY` | 不支持 | `WHERE deleted = 0` |
| KingbaseES | `BIGSERIAL` | `COMMENT '...'` | `WHERE deleted = 0` |

> **重要**：每个方言独立一个迁移文件，文件名 `V3__hblocal_integration.sql`。

### 4.3 验证 SQL

```sql
-- 1. 检查 mate_user 字段已加
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'mate_user' AND column_name IN ('external_system', 'external_user_id');

-- 2. 检查 mate_user_external 表存在
SELECT * FROM information_schema.tables WHERE table_name = 'mate_user_external';

-- 3. 检查 unique 索引
SELECT indexname FROM pg_indexes WHERE tablename = 'mate_workspace_member';
```

---

## 5. 后端实现

### 5.1 依赖：`mateclaw-server/pom.xml`

追加 1 个依赖（裸 nacos-client）：

```xml
<dependencies>
    <!-- 现有依赖 ... -->

    <!-- Nacos 客户端（裸 SDK, 与 Spring Cloud 无关） -->
    <dependency>
        <groupId>com.alibaba.nacos</groupId>
        <artifactId>nacos-client</artifactId>
        <version>2.4.2</version>
    </dependency>
</dependencies>
```

### 5.2 实体改动

#### 5.2.1 `mateclaw-server/src/main/java/vip/mate/auth/model/UserEntity.java`

在已有字段后追加：

```java
/** 外部系统标识 (如 hblocal-cloud, cas, feishu)，nullable */
private String externalSystem;

/** 外部系统用户ID，nullable */
private String externalUserId;
```

#### 5.2.2 新建 `mateclaw-server/src/main/java/vip/mate/auth/model/MateUserExternalEntity.java`

```java
package vip.mate.auth.model;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;

import java.time.LocalDateTime;

/**
 * 外部用户 → MateClaw 用户映射表
 * <p>
 * 一个外部系统用户只对应一个 MateClaw 用户。
 */
@Data
@TableName("mate_user_external")
public class MateUserExternalEntity {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    /** 外部系统标识, 如 hblocal-cloud / cas / feishu */
    private String externalSystem;

    /** 外部系统用户ID */
    private String externalUserId;

    /** MateClaw 用户 ID */
    private Long mateUserId;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    private Integer deleted;
}
```

#### 5.2.3 新建 `MateUserExternalMapper.java`

```java
package vip.mate.auth.repository;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;
import vip.mate.auth.model.MateUserExternalEntity;

@Mapper
public interface MateUserExternalMapper extends BaseMapper<MateUserExternalEntity> {
}
```

### 5.3 `WorkspaceService` 新增方法

**文件：`mateclaw-server/src/main/java/vip/mate/workspace/core/service/WorkspaceService.java`**

在已有方法旁追加：

```java
/**
 * 获取或创建用户的个人 workspace
 * <p>
 * 约定: 个人 workspace 的 slug 固定为 "user-{userId}", 全局唯一
 * <p>
 * 幂等: 已存在直接返回; 不存在则在事务内建 workspace + 加 owner 成员关系
 */
@Transactional(rollbackFor = Exception.class)
public WorkspaceEntity getOrCreatePersonalWorkspace(Long userId) {
    String slug = "user-" + userId;

    // 1. 查已有
    WorkspaceEntity existing = workspaceMapper.selectOne(
            new LambdaQueryWrapper<WorkspaceEntity>()
                    .eq(WorkspaceEntity::getSlug, slug)
    );
    if (existing != null) {
        return existing;
    }

    // 2. 校验 user
    UserEntity user = userMapper.selectById(userId);
    if (user == null) {
        throw new MateClawException("err.workspace.user_not_found",
                400, "User not found: " + userId);
    }

    // 3. 建 workspace
    WorkspaceEntity personal = new WorkspaceEntity();
    personal.setName(user.getNickname() != null       // ★ UserEntity 字段名是 nickname
            ? user.getNickname() + " 的个人空间"
            : "用户 " + userId + " 的个人空间");
    personal.setSlug(slug);
    personal.setDescription("个人工作区");
    personal.setOwnerId(userId);
    workspaceMapper.insert(personal);

    // 4. 加 owner 成员关系
    WorkspaceMemberEntity owner = new WorkspaceMemberEntity();
    owner.setWorkspaceId(personal.getId());
    owner.setUserId(userId);
    owner.setRole("owner");
    owner.setDeleted(0);
    memberMapper.insert(owner);

    log.info("Created personal workspace id={} slug={} for user={}",
            personal.getId(), slug, userId);
    return personal;
}

/**
 * 删除用户时 CASCADE: 软删个人 workspace + 退出所有其他 workspace
 */
@Transactional(rollbackFor = Exception.class)
public void deletePersonalWorkspace(Long userId) {
    String slug = "user-" + userId;
    WorkspaceEntity personal = workspaceMapper.selectOne(
            new LambdaQueryWrapper<WorkspaceEntity>()
                    .eq(WorkspaceEntity::getSlug, slug)
    );

    if (personal != null) {
        // 软删 workspace 本身
        personal.setDeleted(1);
        workspaceMapper.updateById(personal);

        // 软删该 workspace 的所有成员关系
        memberMapper.selectList(
                new LambdaQueryWrapper<WorkspaceMemberEntity>()
                        .eq(WorkspaceMemberEntity::getWorkspaceId, personal.getId())
        ).forEach(m -> {
            m.setDeleted(1);
            memberMapper.updateById(m);
        });
    }

    // 退出所有其他 workspace（不删 workspace 本身）
    memberMapper.selectList(
            new LambdaQueryWrapper<WorkspaceMemberEntity>()
                    .eq(WorkspaceMemberEntity::getUserId, userId)
                    .ne(WorkspaceMemberEntity::getWorkspaceId,
                            personal != null ? personal.getId() : -1L)
    ).forEach(m -> {
        m.setDeleted(1);
        memberMapper.updateById(m);
    });
}
```

需在类顶部新增注入：`UserMapper userMapper`（现有 `WorkspaceService` 没有这个字段，需要在 `@RequiredArgsConstructor` 之外手动加或确认 Lombok 已生成构造器）。

#### 5.3.1 补充：如果 `@RequiredArgsConstructor` 已存在，会自动生成构造器，新加 `userMapper` 字段即可，无需手动写构造方法。

> 注：现有 `WorkspaceService` 已使用 `@RequiredArgsConstructor`（见 L37），新增 `private final UserMapper userMapper;` 后无需其他改动。

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class WorkspaceService {
    private final WorkspaceMapper workspaceMapper;
    private final WorkspaceMemberMapper memberMapper;
    private final UserMapper userMapper;        // ★ 新增
    private final ConversationMapper conversationMapper;
    private final WikiKnowledgeBaseService wikiKnowledgeBaseService;
    private final I18nService i18n;
    // ... existing cache and methods ...
}
```

### 5.4 `AuthService` 集成

**文件：`mateclaw-server/src/main/java/vip/mate/auth/service/AuthService.java`**

#### 5.4.1 注入依赖

```java
private final MateUserExternalMapper externalMapper;
private final WorkspaceService workspaceService;
```

#### 5.4.2 新增方法

```java
/**
 * 通过外部协议（hblocal-cloud / CAS / OAuth2 / SAML）首次访问的用户
 * <p>
 * 流程：
 * 1. 查 mate_user_external 映射
 * 2. 没找到 → 在 mate_user 建一条 (username=hb_{external_user_id})
 * 3. 写映射
 * 4. 调 WorkspaceService.getOrCreatePersonalWorkspace
 * <p>
 * 老用户：返回已有 user + 幂等保证 personal workspace 存在
 */
@Transactional(rollbackFor = Exception.class)
public UserEntity findOrCreateFromGateway(String externalUserId,
                                          String externalSystem,
                                          String nickName,
                                          String email,
                                          String role) {
    // 1. 查映射
    var mapped = externalMapper.selectOne(
            new LambdaQueryWrapper<MateUserExternalEntity>()
                    .eq(MateUserExternalEntity::getExternalSystem, externalSystem)
                    .eq(MateUserExternalEntity::getExternalUserId, externalUserId)
                    .eq(MateUserExternalEntity::getDeleted, 0)
    );
    if (mapped != null) {
        UserEntity user = userMapper.selectById(mapped.getMateUserId());
        if (user != null && Boolean.TRUE.equals(user.getEnabled())) {
            // 老用户: 幂等保证 personal workspace 存在
            workspaceService.getOrCreatePersonalWorkspace(user.getId());
            return user;
        }
    }

    // 2. 建 user
    UserEntity user = new UserEntity();
    user.setUsername("hb_" + externalUserId);
    user.setNickname(nickName);                        // ★ 字段名是 nickname, 不是 displayName
    user.setEmail(email);
    user.setRole(role != null ? role : "user");
    user.setEnabled(true);
    user.setExternalSystem(externalSystem);
    user.setExternalUserId(externalUserId);
    userMapper.insert(user);

    // 3. 写映射
    MateUserExternalEntity ext = new MateUserExternalEntity();
    ext.setExternalSystem(externalSystem);
    ext.setExternalUserId(externalUserId);
    ext.setMateUserId(user.getId());
    externalMapper.insert(ext);

    // 4. 建 personal workspace
    workspaceService.getOrCreatePersonalWorkspace(user.getId());

    log.info("Created user external: system={} userId={} → mateUserId={}",
            externalSystem, externalUserId, user.getId());
    return user;
}
```

#### 5.4.3 修改 `deleteUser`

在现有 `deleteUser` 方法中追加：

```java
@Transactional(rollbackFor = Exception.class)
public void deleteUser(Long userId) {
    // ... 已有软删 user 逻辑 ...

    // ★ 新增: 清理 personal workspace + 退出其他 workspace
    workspaceService.deletePersonalWorkspace(userId);
}
```

### 5.5 `WorkspaceAccessInterceptor` 调整

**文件：`mateclaw-server/src/main/java/vip/mate/config/WorkspaceAccessInterceptor.java`**

把 `resolveWorkspaceId` 方法替换为：

```java
private final WorkspaceService workspaceService;  // ★ 注入

private static final long FALLBACK_WORKSPACE_ID = 1L;

/**
 * 工作空间 ID 解析顺序：
 * 1. 显式 X-Workspace-Id header
 * 2. 当前用户的 personal workspace (slug=user-{userId})
 * 3. fallback workspace=1（未登录场景）
 */
private long resolveWorkspaceId(HttpServletRequest request) {
    // 1. 优先 header
    String header = request.getHeader("X-Workspace-Id");
    if (header != null && !header.isBlank()) {
        try {
            return Long.parseLong(header.trim());
        } catch (NumberFormatException ignored) {}
    }

    // 2. 从 SecurityContext 拿 userId → 查 personal workspace
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth != null && auth.isAuthenticated()
            && !"anonymousUser".equals(auth.getPrincipal())) {
        UserEntity user = authService.findByUsername(auth.getName());
        if (user != null) {
            WorkspaceEntity personal = workspaceService.getOrCreatePersonalWorkspace(user.getId());
            return personal.getId();
        }
    }

    // 3. fallback (未登录场景, 由 Spring Security 拦截)
    return FALLBACK_WORKSPACE_ID;
}
```

**时序要求**：`WorkspaceAccessInterceptor` 是 `HandlerInterceptor`，跑在 `JwtAuthFilter` / `HeaderAuthFilter` 之后，到 `preHandle` 时 `SecurityContextHolder` 已经填好 Authentication。

### 5.6 `GatewayJwtAuthFilter`（新文件）

**文件：`mateclaw-server/src/main/java/vip/mate/auth/header/GatewayJwtAuthFilter.java`**

```java
package vip.mate.auth.header;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;
import vip.mate.auth.model.UserEntity;
import vip.mate.auth.service.AuthService;

import java.io.IOException;
import java.util.List;

/**
 * 解析 hblocal-cloud Gateway 签发的 X-Gateway-Jwt -> 写 Authentication
 * <p>
 * JWT 共享密钥通过 spring.mateclaw.gateway-jwt.secret 配置 (从 .env 注入)
 * <p>
 * 必须先于 JwtAuthFilter 跑 (Ordered.HIGHEST_PRECEDENCE)
 */
@Slf4j
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
@RequiredArgsConstructor
@ConditionalOnProperty(name = "mateclaw.auth.gateway-jwt.enabled", havingValue = "true")
public class GatewayJwtAuthFilter extends OncePerRequestFilter {

    private final AuthService authService;
    private final ObjectMapper objectMapper;

    @Value("${mateclaw.auth.gateway-jwt.secret}")
    private String sharedSecret;

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse resp,
                                    FilterChain chain) throws ServletException, IOException {
        String jwt = req.getHeader("X-Gateway-Jwt");
        if (!StringUtils.hasText(jwt)) {
            chain.doFilter(req, resp);   // 无 JWT 头部, 给 JwtAuthFilter 兜底
            return;
        }

        try {
            Claims claims = Jwts.parser()
                    .setSigningKey(sharedSecret)
                    .parseClaimsJws(jwt)
                    .getBody();

            String externalUserId = claims.getSubject();   // sys_user.user_id
            String nickName = (String) claims.get("nick_name");
            String email = (String) claims.get("email");
            String role = claims.get("role") != null ? (String) claims.get("role") : "user";

            UserEntity user = authService.findOrCreateFromGateway(
                    externalUserId, "hblocal-cloud", nickName, email, role);

            UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(
                            user.getUsername(), null,
                            List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole().toUpperCase()))
                    );
            auth.setDetails(user.getId());
            SecurityContextHolder.getContext().setAuthentication(auth);

            chain.doFilter(req, resp);
        } catch (Exception e) {
            log.warn("GatewayJwtAuthFilter failed: {}", e.getMessage());
            // JWT 无效 → 不设 Authentication, 回落到 JwtAuthFilter
            chain.doFilter(req, resp);
        }
    }
}
```

**注意**：依赖注入 `Jwts` 需要：

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <!-- 版本已在父 pom 管理 -->
</dependency>
```

`mateclaw-server` 已有，无需新增。

### 5.7 Nacos 注册（新文件）

**文件：`mateclaw-server/src/main/java/vip/mate/core/nacos/NacosRegistrationAutoConfiguration.java`**

```java
package vip.mate.core.nacos;

import com.alibaba.nacos.api.NacosFactory;
import com.alibaba.nacos.api.PropertyKeyConst;
import com.alibaba.nacos.api.exception.NacosException;
import com.alibaba.nacos.api.naming.NamingService;
import com.alibaba.nacos.api.naming.po.Instance;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Configuration;

import java.net.InetAddress;
import java.util.Properties;

/**
 * 启动时向 Nacos 手动注册, 心跳由 SDK 维持
 * <p>
 * 用裸 nacos-client 而非 spring-cloud-starter-alibaba-nacos-discovery,
 * 避免把老版本 spring-cloud-commons 拉到 mateclaw (Spring Boot 3.5) 里
 */
@Slf4j
@Configuration
@ConditionalOnProperty(name = "mateclaw.nacos.enabled", havingValue = "true")
public class NacosRegistrationAutoConfiguration {

    @Value("${spring.application.name:mateclaw-server}")
    private String serviceName;

    @Value("${server.port:18088}")
    private int serverPort;

    @Value("${mateclaw.nacos.server-addr:127.0.0.1:8848}")
    private String serverAddr;

    @Value("${mateclaw.nacos.namespace:public}")
    private String namespace;

    @Value("${mateclaw.nacos.group:DEFAULT_GROUP}")
    private String group;

    @Value("${mateclaw.nacos.username:}")
    private String username;

    @Value("${mateclaw.nacos.password:}")
    private String password;

    private NamingService namingService;

    @PostConstruct
    public void register() throws NacosException {
        Properties props = new Properties();
        props.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
        props.put(PropertyKeyConst.NAMESPACE, namespace);
        props.put(PropertyKeyConst.GROUP_NAME, group);
        if (!username.isEmpty()) {
            props.put(PropertyKeyConst.USERNAME, username);
            props.put(PropertyKeyConst.PASSWORD, password);
        }

        namingService = NacosFactory.createNamingService(props);

        Instance instance = new Instance();
        try {
            instance.setIp(InetAddress.getLocalHost().getHostAddress());
        } catch (Exception e) {
            instance.setIp("127.0.0.1");
        }
        instance.setPort(serverPort);
        instance.setServiceName(serviceName);
        instance.setHealthy(true);
        instance.setWeight(1.0);
        instance.setEphemeral(true);
        instance.addMetadata("version", "2.0.0");
        instance.addMetadata("group", "ai");

        namingService.registerInstance(serviceName, group, instance);
        log.info("MateClaw registered to Nacos: service={}, group={}, instance={}:{}",
                serviceName, group, instance.getIp(), serverPort);
    }

    @PreDestroy
    public void shutdown() throws NacosException {
        if (namingService != null) {
            namingService.shutDown();
        }
    }
}
```

### 5.8 配置文件 `application.yml`

增加以下 sections（不要覆盖现有配置）：

```yaml
spring:
  application:
    name: mateclaw-server

server:
  port: 18088

# ====== hblocal-cloud 集成 ======
mateclaw:
  auth:
    gateway-jwt:
      enabled: true
      secret: ${GATEWAY_JWT_SECRET:please-change-me-in-production-32bytes-min}
  nacos:
    enabled: true
    server-addr: ${NACOS_SERVER:127.0.0.1:8848}
    namespace: ${NACOS_NAMESPACE:public}
    group: ${NACOS_GROUP:DEFAULT_GROUP}
    # username/password 按 Nacos 鉴权配置填写
```

`.env` 注入：

```bash
GATEWAY_JWT_SECRET=<32字节以上随机字符串, 与 hblocal Gateway 共享>
NACOS_SERVER=127.0.0.1:8848
```

---

## 6. hblocal-cloud 侧改动

### 6.1 Gateway 路由（5 行 YAML）

在 hblocal-cloud 的 Spring Cloud Gateway 服务的 `application.yml` 追加：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: mateclaw
          uri: lb://mateclaw-server
          predicates:
            - Path=/mateclaw/**,/webjars/**,/swagger-ui/**,/v3/api-docs/**
          filters:
            - StripPrefix=1
            - name: RequestHeader
              args:
                name: X-Gateway-Authed
                value: "true"
            - name: RequestHeader
              args:
                name: X-Gateway-Jwt
                value: "#{T(vip.mate.gateway.util.JwtUtil).createGatewayJwt(request.headers['X-User-Id'])}"
```

### 6.2 签 X-Gateway-Jwt 工具类

在 hblocal-cloud 项目新增：`JwtUtil.java`

```java
package vip.mate.gateway.util;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * 签 X-Gateway-Jwt 给下游 MateClaw 用
 * <p>
 * 共享密钥从 .env 注入, 必须与 mateclaw-auth-jwt-secret 一致
 */
@Component
public class JwtUtil {

    @Value("${gate.mateclaw.secret}")
    private String secret;

    public String createGatewayJwt(String userId) {
        // 实际场景从 sys_user 查 nick_name/email/role
        // 这里简化为 userId 即 username
        return Jwts.builder()
                .setSubject(userId)
                .claim("nick_name", userId)
                .claim("email", userId + "@placeholder.local")
                .claim("role", "user")
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + 3600_000))  // 1h
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
    }
}
```

**生产实现**：把 `nick_name` / `email` 替换为 `sysUserService.getById(userId)` 的真实字段。

---

## 7. CAS SSO Provider（可选）

如果客户希望走浏览器跳转 CAS 登录（而非 Gateway 透传），需要实现 `CasSsoProvider`。

### 7.1 `SsoProperties` 追加 CAS 段

```java
@Data
public static class Cas {
    private boolean enabled = false;
    private String serverUrl;
    private String redirectUri;
}
```

在 `SsoProperties` 顶部加：

```java
private Cas cas = new Cas();
```

### 7.2 `CasSsoProvider.java`

```java
package vip.mate.auth.sso.provider;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import vip.mate.auth.sso.SsoProperties;
import vip.mate.exception.MateClawException;

import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.Map;

@Slf4j
public class CasSsoProvider implements SsoProvider {

    private static final String ID = "cas";
    private static final String NAME = "CAS";

    private final SsoProperties.Cas cfg;
    private final HttpClient http;
    private final ObjectMapper mapper = new ObjectMapper();

    public CasSsoProvider(SsoProperties.Cas cfg) {
        this.cfg = cfg;
        this.http = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();
    }

    @Override public String id() { return ID; }
    @Override public String displayName() { return NAME; }

    @Override
    public String authorizeUrl(String state) {
        String service = cfg.getRedirectUri()
                + (cfg.getRedirectUri().contains("?") ? "&" : "?")
                + "state=" + encode(state);
        return cfg.getServerUrl() + "/login?service=" + encode(service);
    }

    @Override
    public SsoUserInfo resolve(String code, String state) {
        String ticket = code;
        String service = cfg.getRedirectUri()
                + (cfg.getRedirectUri().contains("?") ? "&" : "?")
                + "state=" + encode(state);

        String url = cfg.getServerUrl()
                + "/p3/serviceValidate?service=" + encode(service)
                + "&ticket=" + encode(ticket)
                + "&format=JSON";

        try {
            HttpRequest req = HttpRequest.newBuilder()
                    .uri(URI.create(url))
                    .timeout(Duration.ofSeconds(5))
                    .GET().build();
            HttpResponse<String> resp = http.send(req, HttpResponse.BodyHandlers.ofString());
            if (resp.statusCode() != 200) {
                throw new MateClawException("err.sso.cas_validate_failed",
                        502, "CAS serviceValidate 返回 " + resp.statusCode());
            }
            Map<String, Object> body = mapper.readValue(resp.body(), Map.class);
            Map<String, Object> svcResp = (Map<String, Object>) body.get("serviceResponse");
            Map<String, Object> success = svcResp == null ? null
                    : (Map<String, Object>) svcResp.get("authenticationSuccess");
            if (success == null) {
                throw new MateClawException("err.sso.cas_ticket_invalid",
                        401, "CAS ticket 校验失败");
            }
            String userId = (String) success.get("user");
            Map<String, Object> attrs = (Map<String, Object>) success.get("attributes");
            String email = null, nick = null;
            if (attrs != null) {
                email = firstAttr(attrs, "mail", "email");
                nick = firstAttr(attrs, "cn", "displayName", "nick_name");
            }
            return new SsoUserInfo(userId, null, nick, null, email, null);
        } catch (MateClawException e) {
            throw e;
        } catch (Exception e) {
            throw new MateClawException("err.sso.cas_resolve_failed",
                    502, "CAS 解析失败: " + e.getMessage());
        }
    }

    private static String firstAttr(Map<String, Object> attrs, String... keys) {
        for (String k : keys) {
            Object v = attrs.get(k);
            if (v != null) return v.toString();
        }
        return null;
    }

    private static String encode(String s) {
        return URLEncoder.encode(s, StandardCharsets.UTF_8);
    }
}
```

### 7.3 `SsoAutoConfiguration` 注册 Bean

```java
@Bean
@ConditionalOnProperty(name = "mateclaw.sso.cas.enabled", havingValue = "true")
public CasSsoProvider casSsoProvider(SsoProperties ssoProperties) {
    return new CasSsoProvider(ssoProperties.getCas());
}
```

### 7.4 `SsoService.handleCallback` 末尾追加

```java
// 已有 sign jwt 之后, 末尾追加:
workspaceService.getOrCreatePersonalWorkspace(user.getId());
```

### 7.5 `application.yml`

```yaml
mateclaw:
  sso:
    enabled: true
    cas:
      enabled: true
      server-url: https://cas.example.com
      redirect-uri: https://your-domain/login?sso=callback
```

---

## 8. 前端调整

### 8.1 workspace 切换器默认选中 personal

`mateclaw-ui/src/components/WorkspaceSwitcher.vue`：

```vue
<script setup>
onMounted(async () => {
    const res = await api.get('/api/v1/workspaces')
    workspaces.value = res.data
    // 默认选中 slug=user-{userId} 的 personal workspace
    const personal = workspaces.value.find(w => w.slug?.startsWith('user-'))
    if (personal) {
        currentWorkspace.value = personal
        api.defaults.headers['X-Workspace-Id'] = personal.id
    }
})
</script>
```

### 8.2 SSO 登录按钮

如果接 CAS，前端登录页加：

```vue
<script setup>
const providers = ref([])
async function loadProviders() {
    const res = await api.get('/api/v1/auth/sso/providers')
    providers.value = res.data
}
async function ssoLogin(providerId) {
    const res = await api.get(`/api/v1/auth/sso/authorize?provider=${providerId}`)
    window.location.href = res.data.authorizeUrl
}
onMounted(loadProviders)
</script>

<template>
    <div class="sso-buttons">
        <button v-for="p in providers" :key="p.id" @click="ssoLogin(p.id)">
            {{ p.displayName }} 登录
        </button>
    </div>
</template>
```

---

## 9. 数据库方言迁移对照

**PostgreSQL**（`db/migration/postgres/V3__hblocal_integration.sql`）：

```sql
ALTER TABLE mate_user ADD COLUMN external_system  VARCHAR(64);
ALTER TABLE mate_user ADD COLUMN external_user_id VARCHAR(128);
CREATE INDEX IF NOT EXISTS idx_mate_user_external
    ON mate_user(external_system, external_user_id);

CREATE TABLE IF NOT EXISTS mate_user_external (
    id BIGSERIAL PRIMARY KEY,
    external_system VARCHAR(64) NOT NULL,
    external_user_id VARCHAR(128) NOT NULL,
    mate_user_id BIGINT NOT NULL,
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted INT NOT NULL DEFAULT 0,
    CONSTRAINT uq_user_external UNIQUE (external_system, external_user_id)
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_workspace_member_active
    ON mate_workspace_member(workspace_id, user_id) WHERE deleted = 0;
```

**MySQL**（`db/migration/mysql/V3__hblocal_integration.sql`）：

```sql
ALTER TABLE mate_user ADD COLUMN external_system  VARCHAR(64) COMMENT '外部系统标识';
ALTER TABLE mate_user ADD COLUMN external_user_id VARCHAR(128) COMMENT '外部系统用户ID';
CREATE INDEX idx_mate_user_external ON mate_user(external_system, external_user_id);

CREATE TABLE IF NOT EXISTS mate_user_external (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    external_system VARCHAR(64) NOT NULL,
    external_user_id VARCHAR(128) NOT NULL,
    mate_user_id BIGINT NOT NULL,
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted INT NOT NULL DEFAULT 0,
    UNIQUE KEY uq_user_external (external_system, external_user_id)
);

-- MySQL 不支持 partial index, 改为普通 unique
CREATE UNIQUE INDEX uq_workspace_member_active
    ON mate_workspace_member(workspace_id, user_id, deleted);
```

**H2**（`db/migration/h2/V3__hblocal_integration.sql`，dev/test）：

```sql
ALTER TABLE mate_user ADD COLUMN external_system  VARCHAR(64);
ALTER TABLE mate_user ADD COLUMN external_user_id VARCHAR(128);
CREATE INDEX IF NOT EXISTS idx_mate_user_external
    ON mate_user(external_system, external_user_id);

CREATE TABLE IF NOT EXISTS mate_user_external (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    external_system VARCHAR(64) NOT NULL,
    external_user_id VARCHAR(128) NOT NULL,
    mate_user_id BIGINT NOT NULL,
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted INT NOT NULL DEFAULT 0,
    CONSTRAINT uq_user_external UNIQUE (external_system, external_user_id)
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_workspace_member_active
    ON mate_workspace_member(workspace_id, user_id, deleted);
```

**KingbaseES**（`db/migration/kingbase/V3__hblocal_integration.sql`）：

```sql
ALTER TABLE mate_user ADD COLUMN external_system  VARCHAR(64);
ALTER TABLE mate_user ADD COLUMN external_user_id VARCHAR(128);
CREATE INDEX IF NOT EXISTS idx_mate_user_external
    ON mate_user(external_system, external_user_id);

CREATE TABLE IF NOT EXISTS mate_user_external (
    id BIGSERIAL PRIMARY KEY,
    external_system VARCHAR(64) NOT NULL,
    external_user_id VARCHAR(128) NOT NULL,
    mate_user_id BIGINT NOT NULL,
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted INT NOT NULL DEFAULT 0,
    CONSTRAINT uq_user_external UNIQUE (external_system, external_user_id)
);

CREATE UNIQUE INDEX IF NOT EXISTS uq_workspace_member_active
    ON mate_workspace_member(workspace_id, user_id) WHERE deleted = 0;
```

---

## 10. 完整改动清单

### 10.1 MateClaw 端（10 个文件）

| # | 文件 | 操作 | 行数 |
|---|---|---|---|
| 1 | `mateclaw-server/pom.xml` | 追加 nacos-client 依赖 | +5 |
| 2 | `db/migration/{postgres,mysql,h2,kingbase}/V3__hblocal_integration.sql` | 新建 | ×4 |
| 3 | `auth/model/UserEntity.java` | 加 2 个字段 | +2 |
| 4 | `auth/model/MateUserExternalEntity.java` | 新建 | +30 |
| 5 | `auth/repository/MateUserExternalMapper.java` | 新建 | +5 |
| 6 | `workspace/core/service/WorkspaceService.java` | 加 2 个方法 | +60 |
| 7 | `auth/service/AuthService.java` | 加 1 个方法 + deleteUser 改 1 行 | +50 |
| 8 | `config/WorkspaceAccessInterceptor.java` | 改 resolveWorkspaceId | +20 |
| 9 | `auth/header/GatewayJwtAuthFilter.java` | 新建 | +80 |
| 10 | `core/nacos/NacosRegistrationAutoConfiguration.java` | 新建 | +70 |
| 11 | `application.yml` | 加 mateclaw 段 | +10 |
| 12 | `resources/application.yml` | 加配置 | +20 |

**总计**：~12 个文件，~400 行新代码。

### 10.2 hblocal-cloud 端（2 个文件）

| # | 文件 | 操作 | 行数 |
|---|---|---|---|
| 1 | `gateway/src/main/resources/application.yml` | 加 5 行路由 | +5 |
| 2 | `gateway/src/main/java/.../JwtUtil.java` | 新建 | +30 |

**总计**：~35 行。

### 10.3 MateClaw CAS SSO（可选，4 个文件）

| # | 文件 | 操作 |
|---|---|---|
| 1 | `auth/sso/provider/CasSsoProvider.java` | 新建 |
| 2 | `auth/sso/SsoProperties.java` | 加 Cas 内部类 |
| 3 | `auth/sso/SsoAutoConfiguration.java` | 加 Bean |
| 4 | `auth/sso/SsoService.java` | handleCallback 末尾 +1 行 |

---

## 11. 部署清单

### 11.1 启动顺序

1. **PostgreSQL** 启动
2. **Nacos** 启动
3. **MateClaw** 启动（Flyway 自动迁移 V3）
4. **hblocal-cloud Gateway** 启动

### 11.2 上线检查

```bash
# 1. MateClaw Flyway 迁移日志
grep "V3__hblocal_integration" logs/mateclaw.log

# 2. Nacos 注册成功
grep "MateClaw registered to Nacos" logs/mateclaw.log

# 3. health endpoint
curl http://mateclaw-host:18088/actuator/health

# 4. SSO providers 列表 (如果启用 CAS)
curl http://mateclaw-host:18088/api/v1/auth/sso/providers
# 预期: [{"id":"cas","displayName":"CAS"}]

# 5. 走一次 Gateway 鉴权
curl -H "Authorization: Bearer <hblocal-jwt>" \
     -H "X-Gateway-Jwt: <test-jwt>" \
     http://hblocal-gateway/mateclaw/api/v1/workspaces
# 预期: 200 OK, 列表里至少 1 个 personal workspace
```

### 11.3 回滚

```bash
# 1. 停止 MateClaw
docker-compose stop mateclaw-server

# 2. 回滚 Flyway (手动)
psql -h <host> -U mateclaw -d mateclaw \
     -c "DELETE FROM flyway_schema_history WHERE version = '3';"

# 3. 手动 DROP 新增的字段和表
psql -c "ALTER TABLE mate_user DROP COLUMN external_system;"
psql -c "ALTER TABLE mate_user DROP COLUMN external_user_id;"
psql -c "DROP TABLE IF EXISTS mate_user_external;"
psql -c "DROP INDEX IF EXISTS uq_workspace_member_active;"
```

### 11.4 监控

- MateClaw 注册 Nacos 状态：`GET /nacos/v1/ns/instance?serviceName=mateclaw-server`
- MateClaw 端 `X-Gateway-Jwt` 解码失败次数（业务日志 `WARN` 级别）
- `mate_user_external` 插入频率（反映新用户 onboarding 速度）

---

## 12. 测试用例

### 12.1 单元测试

```java
@SpringBootTest
class AuthServiceIntegrationTest {

    @Autowired AuthService authService;
    @Autowired WorkspaceService workspaceService;

    @Test
    @Transactional
    void firstTimeUserCreatesPersonalWorkspace() {
        UserEntity user = authService.findOrCreateFromGateway(
            "ceshi3", "hblocal-cloud", "测试", "test@qq.com", "user");

        // 1. 用户建好
        assertNotNull(user.getId());
        assertEquals("hb_ceshi3", user.getUsername());

        // 2. personal workspace 建好
        WorkspaceEntity personal = workspaceService.getOrCreatePersonalWorkspace(user.getId());
        assertEquals("user-" + user.getId(), personal.getSlug());
        assertEquals(user.getId(), personal.getOwnerId());
    }

    @Test
    @Transactional
    void secondCallIsIdempotent() {
        UserEntity u1 = authService.findOrCreateFromGateway("alice", "hblocal-cloud", null, null, "user");
        UserEntity u2 = authService.findOrCreateFromGateway("alice", "hblocal-cloud", null, null, "user");
        assertEquals(u1.getId(), u2.getId());  // 不会重复建
    }
}
```

### 12.2 集成测试

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class GatewayJwtAuthFilterTest {

    @LocalServerPort int port;
    @Autowired ObjectMapper mapper;

    @Test
    void validGatewayJwtAuthenticates() throws Exception {
        String jwt = Jwts.builder()
            .setSubject("ceshi3")
            .claim("nick_name", "测试")
            .claim("email", "test@qq.com")
            .claim("role", "user")
            .setExpiration(new Date(System.currentTimeMillis() + 60_000))
            .signWith(SignatureAlgorithm.HS512, TEST_SECRET)
            .compact();

        HttpClient http = HttpClient.newHttpClient();
        HttpRequest req = HttpRequest.newBuilder()
            .uri(URI.create("http://localhost:" + port + "/api/v1/workspaces"))
            .header("X-Gateway-Jwt", jwt)
            .GET().build();

        HttpResponse<String> resp = http.send(req, HttpResponse.BodyHandlers.ofString());
        assertEquals(200, resp.statusCode());
        // 默认在 personal workspace
        assertTrue(resp.body().contains("user-"));
    }
}
```

### 12.3 E2E 测试

```bash
# 1. hblocal 用户登录 hblocal-cloud
# 2. 通过 Gateway 访问 /mateclaw/api/v1/chat/send
# 3. 验证 mate_user_external 新增一条
psql -c "SELECT * FROM mate_user_external WHERE external_system='hblocal-cloud' AND external_user_id='ceshi3';"
# 4. 验证 mate_workspace 新增 personal workspace
psql -c "SELECT * FROM mate_workspace WHERE slug='user-<id>';"
# 5. 验证 mate_workspace_member 新增 owner 关系
psql -c "SELECT * FROM mate_workspace_member WHERE role='owner' AND user_id=<id>;"
```

---

## 13. 附录

### 13.1 数据流转示例

```
Step 1: 用户在 hblocal 前端点击 AI 助手
        → 浏览器 GET /mateclaw/api/v1/chat/init

Step 2: hblocal Gateway 鉴权 + 路由
        → 签 X-Gateway-Jwt (sub=ceshi3, nick_name=测试, email=...)
        → 转发到 mateclaw-server:18088

Step 3: GatewayJwtAuthFilter 解 JWT
        → external_user_id=ceshi3, external_system=hblocal-cloud
        → 调 AuthService.findOrCreateFromGateway

Step 4: AuthService.findOrCreateFromGateway
        → 查 mate_user_external (空)
        → INSERT mate_user (id=10001, username=hb_ceshi3, role=user, ...)
        → INSERT mate_user_external (system=hblocal-cloud, user_id=ceshi3, mate_user_id=10001)
        → WorkspaceService.getOrCreatePersonalWorkspace(10001)
              → 查 slug=user-10001 (空)
              → INSERT mate_workspace (id=20001, slug=user-10001, owner_id=10001)
              → INSERT mate_workspace_member (workspace_id=20001, user_id=10001, role=owner)

Step 5: SecurityContextHolder 装 Authentication
        principal=hb_ceshi3, details=10001, authorities=[ROLE_USER]

Step 6: WorkspaceAccessInterceptor.preHandle
        → 读 X-Workspace-Id (空)
        → resolveWorkspaceId → Authentication → userId=10001 → getOrCreatePersonalWorkspace (幂等, 直接返回 id=20001)
        → hasPermission(20001, 10001, "viewer") → true
        → 放行

Step 7: Controller 业务执行
        → 用户进去看到自己的 "测试 的个人空间"
```

### 13.2 关键 SQL 验证

```sql
-- 1. 确认迁移成功
SELECT version FROM flyway_schema_history WHERE version='3';

-- 2. 确认字段已添加
SELECT column_name FROM information_schema.columns
WHERE table_name='mate_user' AND column_name LIKE 'external%';

-- 3. 确认 unique 索引
SELECT indexname FROM pg_indexes
WHERE tablename='mate_workspace_member' AND indexname='uq_workspace_member_active';

-- 4. 查一个用户的完整链路
SELECT u.id, u.username, u.external_system, u.external_user_id,
       e.external_user_id AS mapped_user,
       w.id AS ws_id, w.slug
FROM mate_user u
LEFT JOIN mate_user_external e ON e.mate_user_id = u.id
LEFT JOIN mate_workspace w ON w.slug = 'user-' || u.id
WHERE u.username = 'hb_ceshi3';
```

### 13.3 常见问题

**Q1: 为什么 username 加 `hb_` 前缀？**  
A: 避免外部 user_id 与本地 username 冲突。`(external_system, external_user_id)` 联合唯一约束保证映射不重复，但 `username` 字段同时承担"本地登录查询"职责，前缀做命名空间隔离。

**Q2: 不加前缀可以吗？**  
A: 可以，但需要：
- `username` 允许 NULL；
- `findByUsername` 只能用于本地登录路径；
- 新增 `findByExternalId(system, userId)` 方法走专用路径。
详见会话记录。

**Q3: 为什么用 `slug=user-{id}` 而不是新加字段？**  
A: 现有 `mate_workspace.slug` 已经有 UNIQUE 约束，零迁移。约定优于配置。

**Q4: 用 `nacos-client` 裸 SDK 而不是 Spring Cloud Alibaba starter？**  
A: Spring Cloud Alibaba 2021.0.5.0 对应 Spring Cloud 2021，与 mateclaw Spring Boot 3.5 体系（Spring Cloud 2023）版本冲突。裸 SDK 跨版本。

**Q5: 二次访问会被 WorkspaceAccessInterceptor 拦吗？**  
A: 不会。`getOrCreatePersonalWorkspace` 幂等。第一次已建 personal workspace，第二次查 slug 直接返回 ~1ms。

**Q6: 用户被 hblocal 那边删除后 mateclaw 怎么办？**  
A: 当前方案**不会自动实时同步**。Hblocal 管理员可调用 MateClaw `DELETE /api/v1/admin/users/{id}` 触发 `deleteUser` + `deletePersonalWorkspace`。后续可加 hblocal 离职事件回调。

**Q7: 团队 workspace 怎么创建？**  
A: 走现有 `WorkspaceController` 的接口（无需新加）。任何用户可以创建 TEAM workspace，自动成为 owner。

**Q8: workspace=1 (默认/系统) 还要吗？**  
A: 保留。BundledSkillSyncer 把内置 skill 放 workspace=1。新用户不再**自动加入** workspace=1，但需要用内置 skill 时管理员可手动加入。

**Q9: 个人 workspace 怎么删除？**  
A: 不可删除（由 `is_deletable` 隐含约定）。用户注销时 `deleteUser` 自动软删。

**Q10: 离职员工能恢复数据吗？**  
A: 软删（`deleted=1`），可通过 SQL 恢复：`UPDATE mate_user SET deleted=0, enabled=1 WHERE id=?` + 重建 personal workspace。

---

## 14. 实施路线图

```
Week 1 (基础接入):
  Day 1: Flyway 迁移 V3 (4 个方言)
  Day 2: WorkspaceService 新增方法
  Day 3: AuthService.findOrCreateFromGateway
  Day 4: WorkspaceAccessInterceptor 调整
  Day 5: 单元测试 + 集成测试

Week 2 (打通 Gateway):
  Day 1: GatewayJwtAuthFilter 实现
  Day 2: NacosRegistrationAutoConfiguration
  Day 3: hblocal 侧 JwtUtil + 路由
  Day 4: 配置 .env + .gitignore
  Day 5: E2E 联调

Week 3 (可选 - CAS SSO):
  Day 1: CasSsoProvider 实现
  Day 2: SsoService 集成 personal workspace
  Day 3: 前端 SSO 按钮
  Day 4: 前端回调处理
  Day 5: 浏览器测试

每周可独立上线、独立回滚。
```

---

## 15. 总结

本方案在以下约束下取得最优：

| 约束 | 满足 |
|---|---|
| 改动最小 | 12 个文件, ~400 行新代码 |
| 不影响老用户 | 老用户走现有逻辑, 新用户多一个 personal workspace |
| 不破坏现有 RBAC | role 字段保留, workspace_member 复用 |
| IdP 无关 | 业务层只接收标准化用户身份 |
| 兼容 CAS | 新加 CasSsoProvider ~60 行 |
| 可演进 | 加 SAML/OIDC 只需新增 Provider |
| mateclaw 独立 | 不被 hblocal Java 1.8 拖死 |
| Docker 自洽 | mateclaw 仍可独立 docker-compose 一把梭 |

---

## 16. 个人工作空间对运行与维护的影响分析

本节系统梳理"每个用户一个个人工作空间（slug=`user-{userId}`）"这一设计在**运行 / 维护 / 性能 / 风险 / 代码改动**五个维度上的具体影响，所有结论都对应该仓库里实际存在的代码与配置。

### 16.1 运行影响（Runtime）

#### 16.1.1 启动阶段几乎无影响
- `WorkspaceSchemaMigration#ensureDefaultWorkspace` 仍然只创建 `id=1 / slug=default` 的系统工作区。
- `BundledSkillSyncer` 仍然只向 workspace 1 写入 `web_search / browser-cdp / ckjia-shopping` 等内置技能（`BUILTIN_WORKSPACE_ID = 1L`）。
- 个人工作空间是**按需懒创建**（在用户首次成功登录时），不增加启动耗时。

#### 16.1.2 必须改造 `WorkspaceAccessInterceptor` 的默认回退
仓库里 `DEFAULT_WORKSPACE_ID = 1L` 出现 **25+ 次**，典型例子：

| 文件 | 场景 |
|---|---|
| `WorkspaceAccessInterceptor#resolveWorkspaceId` | 缺 `X-Workspace-Id` 时回退 1L |
| `SkillService` / `SkillController` | skill 默认工作区 |
| `ApprovalGrantController` / `WorkspaceLookupCache` | 审批授权 |
| `CronConversationResolver` | 定时任务会话回退 |
| `BundledSkillSyncer` | 内置技能落地位置 |

如果用户带的是他刚创建出来的 `user-{id}` 个人空间，而请求头没带，**不能让它进到 workspace 1**，否则：
- 个人会话里的上传/对话被写到 default 工作区。
- cron 回写进 `tasks_1`，泄漏到所有人的"定时任务"列表。

**结论**：只在 `WorkspaceAccessInterceptor` 层做一次回退——header 缺失时，若已登录用户存在 personal workspace，优先用它；否则仍是 default。其他模块保持不变。

#### 16.1.3 内置技能天然兼容
`SkillRuntimeService#matchesWorkspace` 对 builtin 技能直接放行（不校验 workspaceId），因此 `web_search` 这类**全局共享**的技能不需要复制进个人空间，不会出现"新人没有 web_search"的问题。

#### 16.1.4 全局管理员的工作区列表会膨胀
`WorkspaceService#listWithRoleByUserId` 当 `isGlobalAdmin=true` 时直接 `listAll()` 返回**全部**工作区。万级用户上线后：
- 前端顶部下拉一次拉到 N 个工作区。
- `GET /api/v1/workspaces` 当前没有分页参数。

**建议**：给 `list` 加 `?keyword=&page=&size=` 并携带 `effectiveRole`，前端懒加载。

#### 16.1.5 上传路径必须显式规划
`ChatUploadLocationResolver#resolveWorkspaceScopedRoot` 走 `workspace.basePath`，没设时回退 `ChatUploadProperties.defaultRoot`。一旦让用户自由创建个人空间，**所有用户的 chat-uploads 默认会写同一棵目录**：

- 必须在 `createPersonalWorkspace` 里把 `basePath` 模板化为 `${uploadRoot}/users/{userId}`。
- 否则 1 万用户的文件全堆在同一目录，文件系统抗不住，也不好清退。

#### 16.1.6 Cron / Scheduled Tasks 天然适配
`CronConversationResolver#resolve` 按 `workspaceId` 解析 `tasks_{workspaceId}`。每个个人空间都会自动有一份自己的"定时任务"列表——**天然是好的**，无需任何额外配置。

### 16.2 维护影响（Maintenance）

#### 16.2.1 数据行增长（可控但需要规划）

| 实体 | 每用户新增 | 1万用户总增 |
|---|---|---|
| `mate_workspace` | 1 | 1 万 |
| `mate_workspace_member` | 1 | 1 万 |
| `mate_conversation` (`tasks_{id}`) | 1 | 1 万 |

行数级别不大，但 **`tasks_{id}` 即便是空对话也占行**。MySQL/H2 单表百万级以下都没问题。

更值得关注：
- **`mate_workspace_member` 没有 `(workspace_id, user_id)` 的 UNIQUE 约束**（V3 迁移已补 `uq_workspace_member_active`）。如果一个用户并发触发两次"创建个人空间"，不加应用层捕获就会重复行。`SsoService#createSsoUser` 现成的 `DuplicateKeyException` + 回滚模式可直接复用。
- `mate_workspace` 有 `uk_workspace_slug`，而我们要用的 slug 是 `user-{userId}`，天然不冲突，**不需要回滚** workspace，只需在 `addMember` 上兜底。

#### 16.2.2 创建策略：按需懒创建（单一策略）

由于本项目尚未上线，**不存在“老用户兼容”问题**——所有用户都将通过 hblocal-cloud SSO / CAS / Gateway JWT 首次进入 MateClaw，每个用户都会触发一次 personal workspace 的创建。因此**只需一种策略：按需懒创建**。

| 维度 | 说明 |
|---|---|
| 触发时机 | 用户首次经过 `findOrCreateFromGateway` 时，事务内一并创建 |
| 幂等保证 | `getOrCreatePersonalWorkspace(userId)` 每次先按 `slug=user-{userId}` 查询，存在即返回 |
| 上线初期表现 | 第一个用户登录 → 立刻看到自己的空间；第 N 个用户登录 → 同样立刻看到自己的空间 |
| 数据增长曲线 | 与“活跃首次访问用户数”线性相关，**不会出现历史集中补建的尖峰** |

> **可选的运维工具**：提供 `cli/PersonalWorkspaceWarmupCommand`（基于 Spring Boot `ApplicationRunner`），在运维初始化阶段主动从 `hblocal.sys_user` 拉一遍用户并预建 personal workspace。**仅作为“想让所有用户登录前就拥有可见空间”的运维开关**，不是迁移必需。

#### 16.2.3 工作区删除的双重护栏
`WorkspaceService#delete` 有两道硬约束：
- 不能删 `slug=default` 的（已默认放过）。
- 工作区下还有 KB 时直接 `409` 报错。

个人空间原则上不允许用户自删。如需强制，建议：
```java
if (existing.getSlug() != null && existing.getSlug().startsWith("user-")) {
    throw new MateClawException("err.workspace.personal_not_deletable",
            400, "个人工作区不可删除");
}
```

#### 16.2.4 缓存失效传播
- **服务端**：`membershipCache` 是 60s Caffeine（1000 entries），加 / 删成员时已正确 `evictMembershipCache`，无需改。
- **前端**：`useWorkspaceStore.ts` 用 `localStorage['mc-workspace-id']` 缓存当前工作区（19 位 Snowflake 以**字符串**存）。如果用户被某个工作区移除：
  - 路由守卫检测到 `getAccess` 返回 `effectiveRole=null` 时，要把 localStorage 清掉、再 fallback 到 personal。
  - 否则用户会卡在"我看不到这个工作区，但下拉还显示它"的诡异状态。

#### 16.2.5 监控与可观测
- 当前没有按 workspace 维度的指标（消息数、文件数、cron 数）。
- 个人工作空间膨胀后，**出问题想定位到具体用户会很难**。
- 建议引入轻量的 `WorkspaceMetrics`（Micrometer Counter 按 workspaceId 打点），管理后台用 tag 切片。

#### 16.2.6 RBAC 与 owner 转移
- 现有逻辑：**owner 角色不能通过成员管理接口授予**，防止越权。
- 个人工作空间 user → owner 是系统自动授予，OK。
- 但"账号注销 / 工作区合并"的场景需要额外路径：`WorkspaceService#transferOwnership(personalWsId, newOwnerId)`。**目前没有**，需要新增。

#### 16.2.7 并发首登幂等（全员触发场景）

由于**所有用户都会经过 `findOrCreateFromGateway`**，首登不是小概率事件——任何 hblocal 单点入口上线首日都会出现 N 个并发请求同时触发 personal workspace 创建：

- SSO 回跳失败重试 → 同一个用户可能在毫秒级被请求 2~3 次。
- 浏览器多标签页 + 同一用户 → 并发度更高。
- 早高峰集中登录 → `WorkspaceService.getOrCreatePersonalWorkspace` 会被高并发调用。

**必须**抄 `SsoService.createSsoUser` 的 `try { insert } catch (DuplicateKeyException) { 重新查询 }` 模式，且：
- `@Transactional(rollbackFor = Exception.class)` 必须保留。
- 拿到 `DuplicateKeyException` 后**重新查询**而不是直接报错。
- Caffeine `membershipCache` 在并发场景下要承受瞬时穿透，建议临时把 `expireAfterWrite` 从 60s 调到 5~10s 做压测后再定。

### 16.3 性能 / 资源（量化）

| 项 | 数据 | 评估 |
|---|---|---|
| 1 万用户新增行 | ~3 万行 / 3 张表 | MySQL 单机毫无压力 |
| 成员缓存容量 | Caffeine 1000 entries / 60s | 1 万用户下命中率约 90%，未命中走主键查询，仍是 ms 级 |
| 成员缓存失效传播 | 同进程同步 evict | 多实例部署时**没问题**（每个实例自己缓存 60s） |
| Cron / tasks_{id} 对话 | 每个工作区 1 个空对话 | 1 万 = 1 万空行；建议定期 `IN (...)` 批量归档 |
| 文件系统 | `basePath` 隔离 | **必做**，否则 1 万用户共享同一目录 |
| 工作区列表查询 | `selectBatchIds(workspaceIds)` | member 数 ≤ 20 时毫无压力；超过就要分页 |

### 16.4 风险清单（按优先级）

| # | 风险 | 等级 | 说明 | 缓解措施 |
|---|---|---|---|---|
| R1 | `basePath` 没规划 = 文件全部混在一起 | 🔴 | 万级用户共享同一目录 | `createPersonalWorkspace` 模板化 `basePath = ${uploadRoot}/users/{userId}` |
| R2 | `WorkspaceAccessInterceptor` 缺头回退未改 | 🔴 | 默认仍指向 workspace 1，个人空间形同虚设 | 改造 `resolveWorkspaceId` 优先用 personal |
| R3 | `mate_workspace_member(workspace_id, user_id)` 缺 UNIQUE | 🔴 | 并发首登会重复行 | V3 迁移已加；`DuplicateKeyException` 应用层兜底 |
| R4 | admin 列表不分页 | 🟠 | 上线后接口响应慢 / 前端渲染卡 | `list` 接口加分页 + keyword |
| R5 | 前端 localStorage 失效传播 | 🟠 | 被踢出 / 空间删除后下拉还显示 | 路由守卫清 localStorage + fallback personal |
| R7 | owner 转移路径缺失 | 🟡 | 注销 / 合并场景未实现，全员场景下会被放大 | 新增 `transferOwnership` |
| R8 | 没按 workspace 维度打点 | 🟢 | 故障定位困难 | 引入 `WorkspaceMetrics` Micrometer Counter |
| R9 | `listByUserId` 在 memberships 为空时回退 default | 🟡 | 全员场景下顺序错即全员错 | 强约束：`personal workspace` 必须在 `listWorkspaces` 返回前已建 |

### 16.5 关键代码改动点（补充到第 5 节）

| 改动 | 文件 | 关键点 |
|---|---|---|
| 个人空间创建服务 | `WorkspaceService`（新增 `createPersonalWorkspace`）/ `SsoService` 或 `AuthService` 首登分支 | `@Transactional` 包 user / workspace / member 三表；slug=`user-{userId}`；重复幂等 |
| 默认回退 | `WorkspaceAccessInterceptor#resolveWorkspaceId` | header 缺失时查 user → personal workspace → 没再回退 default |
| 工作区列表 | `WorkspaceController#list` + `WorkspaceService#listWithRoleByUserId` | 加分页 + 关键字 |
| 上传根 | `ChatUploadLocationResolver` 或 `createPersonalWorkspace` 模板 | `basePath` 按 userId 隔离 |
| 前端 Store | `mateclaw-ui/src/stores/useWorkspaceStore.ts` | 登录默认选 personal；删除 / 失联时清理 localStorage |
| 运维 warm-up 工具（可选） | `cli/PersonalWorkspaceWarmupCommand`（`ApplicationRunner`） | 上线初期预建 personal workspace（从 `hblocal.sys_user` 拉人）；非迁移必需 |
| owner 转移（可选） | `WorkspaceService#transferOwnership` | 仅账号注销 / 合并场景需要 |
| 监控打点 | `WorkspaceMetrics` Micrometer Counter | 按 workspaceId 打点（消息 / 上传 / cron） |

### 16.6 结论

把"每个用户一个个人工作空间"加上去，**对本项目的主要冲击集中在三处**：

1. **`WorkspaceAccessInterceptor` 的默认回退**——必须让"没有 `X-Workspace-Id` 时"指向 personal 而不是 default。
2. **`ChatUploadLocationResolver` 的 basePath 模板**——必须按 userId 隔离目录。
3. **工作区列表的分页与删除约束**——UI 和 admin 路径需要补。

启动、cron、skill、对话、成员缓存这些**几乎不受影响**——现有架构本来就是按 workspace 维度设计的，只是默认工作区个数从 1 变成 N。把上面三点改掉，整体是一个**为全新部署设计、低成本、零迁移**的演进。

由于项目尚未上线（不存在老用户），R1 / R2 / R3 / **R7** / **R9** 作为**上线硬门槛**——R7 / R9 在全员触发场景下被放大，必须同步解决。R4 / R5 作为**上线前必须**修复的体验项。R8 监控打点进入下一迭代即可。
