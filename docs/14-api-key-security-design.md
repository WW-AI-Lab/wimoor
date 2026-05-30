# 14. API Key 安全认证方案设计

> 版本：v1.0  日期：2026-05-29  
> 前置文档：[13-agent-product-listing-integration.md](13-agent-product-listing-integration.md)

---

## 1. 现状分析

### 1.1 当前认证体系

Wimoor 目前只有一套认证机制，完整链路如下：

```
前端/外部系统
    → POST /admin/api/v1/oauth/token（账号+密码）
    → LoginController 验证账号 → 生成 Shiro session
    → 写入 Redis：login_tokens:{jsessionid} = UserInfo JSON（TTL 3 小时）
    → 返回 jsessionid

后续请求
    → Header: jsessionid={token}
    → Gateway: SecurityGlobalFilter 查 Redis 验证 session
    → 注入 X-USERINFO header
    → 下游服务: UserInfoFilter 解析 UserInfo → UserInfoContext
```

**关键特性：**

- Token 是 Shiro Session ID，不是 HMAC 签名 JWT。
- 单 session 只有 3 小时 TTL，每次请求会重置过期时间。
- 整个系统只有这一套用户身份体系（admin / manager / support / actor 四种 usertype）。
- 鉴权规则存在 Redis hash `system:perm_roles_rule:url:` 中，key 形如 `GET:/erp/api/v1/...`。
- admin / manager usertype 直接绕过角色匹配。

### 1.2 第三方 API 配置表（现有）

系统已经有 `t_erp_thirdparty_api` 表，用于保存**对外调用第三方服务**时需要的密钥配置（快递单号 API、海外仓 API 等），字段包括：

```
id, api, name, appkey, appsecret, token, shopid, url, system, isdelete, operator, opttime
```

这张表的语义是：**Wimoor 调用外部系统时使用的凭证**，不是**外部系统调用 Wimoor 时的身份凭证**。方向相反。

### 1.3 当前存在的 4 个安全缺口

| 缺口 | 描述 | 风险等级 |
|---|---|---|
| 没有外部 API Key 体系 | 外部选品系统/Agent 只能复用人工登录 session，无法做服务级身份隔离 | 高 |
| session 被盗用风险 | 3 小时 TTL 内任何持有 jsessionid 的系统都可代入操作 | 高 |
| 缺少调用来源追踪 | 所有操作只记录 operator（用户 ID），无法区分"由哪个外部系统发起" | 中 |
| 没有调用频率控制 | Agent 类系统高频调用时无任何保护机制 | 中 |

---

## 2. 设计目标

1. **最小侵入**：不改现有 jsessionid / session 认证链路，新增一条平行的 API Key 认证路径。
2. **租户隔离**：每个企业（shopid）管理自己的 API Key，相互之间完全隔离。
3. **最小权限**：API Key 只能访问其被授权的 Agent facade 接口前缀，不能访问任意内部接口。
4. **可审计**：每次 API Key 调用均记录来源、路径、时间，支持后续审计。
5. **可吊销**：Admin 可随时禁用或删除某个 API Key，立即生效（不等 TTL 自然过期）。
6. **无状态验证**：API Key 认证应该是无状态的，Gateway 层从 Redis 校验后注入身份，下游服务不感知变化。

---

## 3. 整体认证架构

### 3.1 两条认证路径并行

```
┌──────────────────────────────────────────────────────────────────┐
│                         请求入口（Gateway）                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   路径 A（已有）                  路径 B（新增）                   │
│   Header: jsessionid             Header: X-API-KEY              │
│         │                               │                        │
│   SecurityGlobalFilter           ApiKeyGlobalFilter（新增）       │
│         │                               │                        │
│   Redis: login_tokens:*          Redis: api_keys:*               │
│         │                               │                        │
│   注入 X-USERINFO                  注入 X-USERINFO               │
│   （真实用户 JSON）               （虚拟 API Bot 用户 JSON）       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              │
                    下游服务（不变）
              UserInfoFilter 解析 UserInfoContext
```

### 3.2 API Bot 用户的身份构造

API Key 认证成功后，Gateway 不会生成真实用户，而是构造一个"API Bot 用户"注入下游：

```json
{
  "id": "apikey-{keyId}",
  "account": "apikey:{keyId}",
  "usertype": "apibot",
  "companyid": "{shopid}",
  "session": "{apiKeyValue}",
  "roles": ["{apiKeyId}"],
  "datalimits": [],
  "apiKeyId": "{keyId}",
  "apiKeyName": "{keyName}",
  "externalSource": "{keyCreatedBy}"
}
```

下游服务通过 `UserInfoContext.get()` 拿到上述对象后，业务层可以用 `userinfo.getUsertype().equals("apibot")` 区分机器请求与人工操作。

---

## 4. 数据库设计

### 4.1 API Key 主表（db_admin.t_sys_api_key）

推荐放在 admin 库，与用户、权限体系同库，便于统一管理和审计：

```sql
CREATE TABLE `t_sys_api_key` (
  `id`              bigint unsigned NOT NULL COMMENT '主键(Snowflake)',
  `shopid`          bigint unsigned NOT NULL COMMENT '企业ID(租户隔离)',
  `name`            varchar(100) NOT NULL COMMENT 'API Key 名称(用户可读)',
  `description`     varchar(500) COMMENT '用途说明',
  `key_prefix`      varchar(20)  NOT NULL COMMENT 'Key前缀，用于快速识别，格式如 wm_live_',
  `key_hash`        varchar(128) NOT NULL COMMENT 'API Key 的 SHA-256 哈希值（不存明文）',
  `key_hint`        varchar(20)  NOT NULL COMMENT 'Key末8位明文，用于用户识别自己的 key',
  `scopes`          varchar(2000) COMMENT '授权范围，逗号分隔的 URL 前缀，如 /erp/api/v1/agent/**,/amazon/api/v1/agent/**',
  `rate_limit`      int DEFAULT 60 COMMENT '每分钟最大请求数，0=不限制',
  `ip_whitelist`    varchar(1000) COMMENT 'IP 白名单，逗号分隔，NULL=不限制',
  `expire_time`     datetime COMMENT '过期时间，NULL=永不过期',
  `last_used_time`  datetime COMMENT '最后一次成功调用时间',
  `last_used_ip`    varchar(45) COMMENT '最后调用来源 IP',
  `call_count`      bigint DEFAULT 0 COMMENT '总调用次数（近似值）',
  `is_enabled`      tinyint NOT NULL DEFAULT 1 COMMENT '是否启用 1=是 0=否',
  `creator`         bigint unsigned COMMENT '创建人用户ID',
  `create_time`     datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time`     datetime ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_shop_name` (`shopid`, `name`),
  KEY `idx_key_hash` (`key_hash`),
  KEY `idx_shopid` (`shopid`)
) ENGINE=InnoDB COMMENT='外部 API Key 管理表';
```

**安全设计要点：**

- 数据库中**不保存 API Key 明文**，只存 SHA-256 哈希值（`key_hash`）。
- 生成时只向用户展示一次完整 Key，之后只能看到末8位（`key_hint`）用于识别。
- `scopes` 严格限制该 Key 可以访问的 URL 路径前缀。

### 4.2 API Key 调用日志表（db_admin.t_sys_api_key_log）

```sql
CREATE TABLE `t_sys_api_key_log` (
  `id`          bigint unsigned NOT NULL COMMENT '主键',
  `key_id`      bigint unsigned NOT NULL COMMENT 'FK → t_sys_api_key.id',
  `shopid`      bigint unsigned NOT NULL COMMENT '企业ID',
  `request_path` varchar(500) NOT NULL COMMENT '请求路径',
  `http_method` varchar(10) NOT NULL COMMENT 'HTTP 方法',
  `client_ip`   varchar(45) COMMENT '来源 IP',
  `response_code` int COMMENT 'HTTP 响应码',
  `latency_ms`  int COMMENT '响应耗时毫秒',
  `request_id`  varchar(64) COMMENT '请求唯一 ID（便于链路追踪）',
  `call_time`   datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '调用时间',
  PRIMARY KEY (`id`),
  KEY `idx_key_id` (`key_id`),
  KEY `idx_shopid_time` (`shopid`, `call_time`)
) ENGINE=InnoDB COMMENT='API Key 调用日志';
```

---

## 5. API Key 生成规范

### 5.1 Key 格式

```
wm_{env}_{random32}

示例：wm_live_k8j2mn4a9xvqr3z0p7twc5y6fd1ei8s
```

- `wm_` 固定前缀，便于识别是 Wimoor 系统 Key。
- `{env}` 区分环境：`live`（生产）/ `test`（测试）/ `dev`（开发）。
- `{random32}` 使用 `SecureRandom` 生成 32 字节后 Base58 编码（避免特殊字符）。

### 5.2 生成与存储流程

```
1. 用户在管理后台点击"创建 API Key"
2. 后端生成 rawKey = "wm_live_" + Base58(SecureRandom(32))
3. 计算 keyHash = SHA-256(rawKey)
4. 截取 keyHint = rawKey 的末8位
5. 仅此一次向前端返回完整 rawKey（前端提示用户保存）
6. 数据库只写入 keyHash 和 keyHint，rawKey 不落库
7. 将 keyHash → keyId 的映射写入 Redis（用于 Gateway 快速查找）
```

### 5.3 Redis 缓存结构

```
# Hash：key_hash → 序列化的 ApiKeyInfo（含 shopid、scopes、rateLimit、isEnabled 等）
api_keys:hash:{SHA-256(rawKey)}  →  ApiKeyInfo JSON（TTL 30 分钟，自动刷新）

# 限频计数（滑动窗口 / 固定窗口均可）
api_keys:rate:{keyId}:{当前分钟时间戳}  →  调用次数（TTL 2 分钟）
```

---

## 6. Gateway 层认证过滤器设计

### 6.1 新增 ApiKeyGlobalFilter

需要在 wimoor-gateway 模块新增一个 `ApiKeyGlobalFilter`，Order 设置为 `SecurityGlobalFilter.Order - 1`（优先执行），主要逻辑如下：

```
1. 读取请求 Header: X-API-KEY
2. 如果 Header 不存在，跳过（交给 SecurityGlobalFilter 走 jsessionid 流程）
3. 如果 Header 存在，进入 API Key 认证流程：

   a. 计算 keyHash = SHA-256(X-API-KEY)
   b. 从 Redis 查 api_keys:hash:{keyHash}
      - 不存在 → 返回 401 UNAUTHORIZED
      - 存在 → 反序列化 ApiKeyInfo
   
   c. 验证 ApiKeyInfo：
      - is_enabled = false → 返回 403 FORBIDDEN
      - expire_time 已过 → 返回 401 UNAUTHORIZED
      - ip_whitelist 不为空且请求 IP 不在白名单 → 返回 403 FORBIDDEN
   
   d. 验证请求路径 scope：
      - 当前请求路径是否匹配 ApiKeyInfo.scopes 中的任一前缀
      - 不匹配 → 返回 403 FORBIDDEN
      - 匹配 → 继续
   
   e. 检查限频（若 rate_limit > 0）：
      - 读取 Redis 当前分钟计数
      - 超限 → 返回 429 TOO_MANY_REQUESTS
      - 未超限 → 计数 +1
   
   f. 构造 ApiBot UserInfo JSON（见第 3.2 节）
   g. URL Encode 后注入 X-USERINFO header
   h. 异步更新 last_used_time 和 call_count（不阻塞主请求）
   i. 记录调用日志（可异步写入，不影响响应延迟）
   j. 放行请求
```

### 6.2 路径 scope 匹配规则

scope 使用 Ant 路径匹配（与现有权限系统一致）：

```
# 推荐的 Agent facade scope 前缀配置
/erp/api/v1/agent/**
/amazon/api/v1/agent/**

# 示例：允许读取商品主数据（需要显式授权只读接口）
/erp/api/v1/material/getMaterialInfo
/erp/api/v1/material/media/list
/erp/api/v1/material/media/pool
```

**重要规则：**

- scope 中不允许使用 `/**` 根通配符，必须至少包含 `/erp/` 或 `/amazon/` 等服务前缀。
- 建议每个 API Key 创建时只开放其真实需要的 facade 前缀，而不是整个服务路径。
- 管理界面提供"权限模板"供用户快速选择常用 scope 组合（选品同步 / Listing 发布 / 媒体处理 / 只读查询）。

---

## 7. Admin 管理功能设计

### 7.1 管理界面位置

建议放在现有管理后台的**系统设置 → API Key 管理**菜单下，对应：

- 后端模块：`wimoor-admin`，新增 controller 路径 `/admin/api/v1/apikey/*`
- 前端页面：`wimoorui/src/views/admin/apikey/index.vue`

### 7.2 功能清单

#### 列表页功能

| 功能 | 说明 |
|---|---|
| 查看 API Key 列表 | 显示名称、Key Hint（末8位）、授权范围、最后调用时间、状态 |
| 创建 API Key | 填写名称、描述、scope、限频、IP 白名单、过期时间 |
| 停用/启用 API Key | 立即生效（删除 Redis 缓存，写库更新 is_enabled） |
| 删除 API Key | 软删除，清除 Redis 缓存 |
| 查看调用日志 | 按时间段、path、响应码查询 t_sys_api_key_log |
| 查看调用统计 | 每日/每小时调用量曲线、成功率、高频路径 |

#### 创建 API Key 弹窗字段

| 字段 | 类型 | 说明 |
|---|---|---|
| 名称 | 文本 | 必填，同一企业唯一 |
| 用途说明 | 文本 | 可选，说明这个 Key 给谁用 |
| 权限范围 | 多选 | 从预定义模板选择，或自定义 scope 前缀 |
| 限频（每分钟请求数） | 数字 | 默认 60，0 表示不限制 |
| IP 白名单 | 文本 | 逗号分隔，留空不限制 |
| 过期时间 | 日期 | 留空永不过期 |

#### 创建成功弹窗（只出现一次）

```
┌─────────────────────────────────────────────────┐
│  API Key 创建成功                                 │
│  请立即复制并妥善保存，关闭后将无法再次查看完整 Key  │
│                                                  │
│  wm_live_k8j2mn4a9xvqr3z0p7twc5y6fd1ei8s        │
│                                          [复制]   │
│                                                  │
│  [我已保存，关闭]                                  │
└─────────────────────────────────────────────────┘
```

### 7.3 权限范围模板建议

| 模板名称 | 包含 scope | 适用场景 |
|---|---|---|
| 选品同步 | /erp/api/v1/agent/material/** | 外部选品系统写入商品主数据 |
| 只读查询 | /erp/api/v1/material/getMaterialInfo，/erp/api/v1/material/list | 外部系统读取商品基础信息 |
| Listing 发布 | /amazon/api/v1/agent/listing/** | AI Agent 发布/更新 Listing |
| 媒体处理 | /erp/api/v1/agent/media/** | 图片/视频 AI 处理与回传 |
| 全 Agent 访问 | /erp/api/v1/agent/**，/amazon/api/v1/agent/** | 受信任的全功能 Agent |

---

## 8. UserInfo 扩展方案

当前 `UserInfo` 类需要做一处扩展：增加 `usertype = "apibot"` 的支持，以及两个 Agent 专属字段：

```java
// UserInfo 新增字段（向后兼容，只在 apibot 场景填入）
private String apiKeyId;       // API Key ID
private String externalSource; // 来源标识（如"1688-agent"、"media-agent"）
```

这样下游服务的业务代码可以做精确的来源判断：

```java
UserInfo user = UserInfoContext.get();
if ("apibot".equals(user.getUsertype())) {
    // 机器请求，记录 externalSource 到操作日志
    log.info("API Bot request from: {}", user.getExternalSource());
}
```

---

## 9. 下游服务需要适配的点

### 9.1 不需要改动的部分

- `UserInfoFilter`：已经能正确解析 `X-USERINFO`，无需改动。
- `UserInfoContext.get()`：正常使用即可。
- `@SystemControllerLog`：现有日志注解会自动记录 operator，apibot 场景下记录 apiKeyId。

### 9.2 需要少量改动的部分

**Agent facade controller 层**（将来新增时注意）：

1. 如果接口需要限制"只允许 API Key 调用，不允许人工前端操作"，在 facade controller 里加一行：

```java
if (!"apibot".equals(UserInfoContext.get().getUsertype())) {
    throw new BizException("该接口仅允许通过 API Key 调用");
}
```

2. 如果接口允许两种方式调用，直接使用 `UserInfoContext.get().getCompanyid()` 即可，无需判断 usertype。

### 9.3 鉴权规则补充

Agent facade 接口路径（如 `/erp/api/v1/agent/**`）需要加入 Redis `system:perm_roles_rule:url:` 中，配置允许访问的角色。

建议给所有 agent facade 路径配置一个特殊角色 `ROLE_APIBOT`，然后：

- 在创建 API Key 时，Gateway 层自动为该 Key 的 ApiBot UserInfo 注入 `roles = ["ROLE_APIBOT"]`。
- 在 Redis perm_roles_rule 中配置 `/erp/api/v1/agent/**` → `ROLE_APIBOT`。
- 这样 checkAuthority 已有逻辑无需改动，自动完成鉴权。

---

## 10. 安全加固建议

### 10.1 Key 的安全传输

- API Key 只能通过 HTTPS 传输（Gateway 生产环境应强制 TLS）。
- 禁止将 API Key 放在 URL Query String 中（容易被日志记录），只允许放在请求 Header `X-API-KEY`。

### 10.2 防枚举攻击

- 连续 5 次 API Key 验证失败（相同来源 IP），触发临时封禁（写入 Redis 黑名单，TTL 15 分钟）。
- 黑名单 key 格式：`api_keys:blocked_ip:{ip}` → "1"（TTL 15 分钟）。
- Gateway `ApiKeyGlobalFilter` 在进入验证之前先查黑名单。

### 10.3 哈希算法选择

- 使用 `SHA-256` 对原始 Key 做哈希后存库，不使用 MD5 / SHA-1。
- 不使用加盐 bcrypt，原因：API Key 验证需要在 Gateway 高频执行，bcrypt 的计算开销不适合这个场景；而 API Key 本身已经有足够的随机性（256 bit），抗暴力破解能力足够。

### 10.4 Key 生命周期建议

| 场景 | 建议 TTL |
|---|---|
| 生产环境全功能 Key | 90 天，到期前 7 天发邮件/飞书提醒 |
| CI/CD 自动化测试 Key | 7 天 |
| 临时调试 Key | 24 小时 |
| 永不过期 Key | 仅 manager 用户可创建，并强制要求设置 IP 白名单 |

---

## 11. 实施路径

### 11.1 分 3 期落地

#### 第一期：最小可用（1～2 周）

目标：能创建 Key、Gateway 能验证、Agent 能调用。

1. 新增 `db_admin.t_sys_api_key` 表（不含日志表）。
2. 新增 `wimoor-admin` 的 `SysApiKeyController` + `SysApiKeyService`（CRUD + 生成 Key）。
3. 新增 `wimoor-gateway` 的 `ApiKeyGlobalFilter`（支持 Redis 校验、scope 匹配、Bot UserInfo 注入）。
4. 新增 `UserInfo.apiKeyId` / `UserInfo.externalSource` 字段。
5. 前端 API Key 管理基础页面（增删改查 + 一次性展示 Key 弹窗）。
6. 不做限频、不做 IP 白名单，放第二期。

**最小 SQL（第一期只建主表，去掉可选字段）：**

| 必选字段 | 说明 |
|---|---|
| id, shopid, name, key_hash, key_hint | 核心必选 |
| scopes, is_enabled | 控制访问范围 |
| creator, create_time | 审计最小集 |

#### 第二期：安全加固（1 周）

1. 增加 `rate_limit` 限频（Gateway Redis 滑动窗口）。
2. 增加 `ip_whitelist` IP 白名单校验。
3. 增加 `expire_time` 过期校验与提醒机制（Quartz 每天检查，飞书通知）。
4. 增加防枚举黑名单机制。
5. 前端调用统计页面。

#### 第三期：完整审计（1 周）

1. 新增 `db_admin.t_sys_api_key_log` 日志表。
2. Gateway 异步写调用日志（建议走消息队列或异步线程池，不影响响应延迟）。
3. 前端日志查询页 + 调用统计图表。
4. 告警机制：调用量异常、成功率下降时飞书推送。

### 11.2 受影响的模块

| 模块 | 改动量 | 说明 |
|---|---|---|
| wimoor-gateway | 中 | 新增 `ApiKeyGlobalFilter` 约 150 行 |
| wimoor-admin | 中 | 新增 apikey CRUD service + controller 约 300 行 |
| wimoor-common/common-core | 小 | `UserInfo` 新增 2 个字段 |
| wimoorui | 中 | 新增管理页面 2 个 vue 文件 |
| init-config/mysql/数据库结构/db_admin | 小 | 新增 2 张表的 DDL SQL |
| 下游业务服务 | 极小 | 无需改动，按需加 usertype 判断 |

---

## 12. 与文档 13 的接口关系

文档 13 中规划的 Agent facade 接口（`/erp/api/v1/agent/**`、`/amazon/api/v1/agent/**`）与本文的 API Key 认证的结合方式是：

```
API Key 认证（本文）    +    Agent facade 接口（文档13）
         ↓                           ↓
  Gateway 校验 Key             后端 Facade Controller
  注入 apibot UserInfo         复用现有 service 层能力
  scope 限制只能打到           追加 externalSource 日志记录
  /agent/** 路径前缀
```

两个文档合并起来，构成完整的"外部系统安全对接"方案：

- **本文**解决"谁在调用、有没有权限调用"。
- **文档 13**解决"调用了什么接口、接口内部怎么编排"。

---

## 13. 不建议采用的方案（附原因）

### 方案 X：复用 t_erp_thirdparty_api 表

现有表 `t_erp_thirdparty_api` 存放的是"Wimoor 调外部服务的密钥"，语义方向相反，而且在 ERP 库而不是 Admin 库，不适合做统一认证管理。不建议复用。

### 方案 Y：引入 OAuth2 / JWT

将当前认证体系整体替换成 OAuth2 + JWT 是正确的长期方向，但短期内改动量极大（所有服务、所有前端路由全部联动）。考虑到本系统当前的规模和改动风险，建议先用 API Key 解决外部系统对接的最迫切需求，等系统整体架构升级时再统一迁移到 OAuth2。

### 方案 Z：直接用 jsessionid + 机器账号

为 Agent 专门创建一个机器用户，用账号密码换取 jsessionid 来调用。问题：

1. session 只有 3 小时，需要外部系统自行续签，容易出错。
2. 机器账号拥有该用户的全部权限，没有细粒度 scope 控制。
3. 无法从日志中区分"人工操作"和"机器操作"。
4. 账号泄露风险等同于真实用户账号泄露。
