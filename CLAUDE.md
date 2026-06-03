# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 个人配置

- 每次回答我加上我的名字 TDX
- 

## 项目概述

Pig 是基于 Spring Cloud + Spring Boot 的企业级 RBAC 快速开发平台，支持**微服务**和**单体**两种部署模式。使用 Spring Authorization Server 提供 OAuth2 认证授权，Nacos 作为服务注册与配置中心。

- Java 17（本地路径在 `/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home`），Spring Boot 3.5.12，Spring Cloud 2025.0.1
- Maven（本地路径在 `/opt/homebrew/Cellar/maven/3.9.14`） 多模块，根 `groupId:artifactId` = `com.pig4cloud:pig`，版本 3.9.2
- 前端项目在 [pig-ui/](pig-ui/)，独立的 Vue 3 + TypeScript + Vite 项目，详见 [pig-ui/CLAUDE.md](pig-ui/CLAUDE.md)
- 用户全局 Java 规则在 `~/.claude/rules/java/` 下（编码风格、测试、安全、模式），已作为背景规则加载，无需重复

## 常用命令

```bash
# 代码格式化（提交前必须执行）
mvn spring-javaformat:apply

# 代码格式校验（CI 中运行）
mvn spring-javaformat:validate

# 构建微服务版本（默认 cloud profile）
mvn clean install

# 构建单体版本（启用 pig-boot 模块）
mvn clean install -Pboot

# 跳过测试构建
mvn clean install -DskipTests

# 运行单个模块的测试
mvn test -pl pig-upms/pig-upms-biz

# 运行单个测试类
mvn test -pl pig-upms/pig-upms-biz -Dtest=UserServiceTest

# Docker 构建所有镜像
mvn clean install docker:build

# 本地开发：Docker Compose 启动基础服务（MySQL、Redis、Nacos）
docker compose up -d pig-mysql pig-redis pig-register

# 前端启动（在 pig-ui/ 目录下）
cd pig-ui && npm run dev
```

## 架构概览

### 微服务模式（默认）

```
pig-ui (Vue 3, 端口 8888)
    │
    ▼
pig-gateway (Spring Cloud Gateway, 端口 9999)
    │ 路由、限流（Redis + Lua）、认证校验
    ├──→ pig-auth (认证中心, 端口 3000) — Spring Authorization Server
    ├──→ pig-upms-biz (用户权限管理, 端口 4000)
    ├──→ pig-monitor (Spring Boot Admin 监控, 端口 5001)
    ├──→ pig-codegen (代码生成器, 端口 5002)
    └──→ pig-quartz (定时任务管理, 端口 5007)
         │
         └──→ pig-register (内嵌 Nacos, 端口 8848/9848) — 服务发现 + 配置中心
```

### 单体模式（`-Pboot` profile）

[pig-boot/](pig-boot/) 将 auth、upms-biz、codegen、quartz 聚合为单个应用，端口 9999、上下文路径 `/admin`，直接连接 MySQL/Redis，**禁用 Nacos 服务发现和配置**。适用于单 JAR 部署。

### Maven 模块结构

| 模块 | 用途 |
|---|---|
| `pig-register` | 内嵌 Nacos 服务注册与配置中心 |
| `pig-gateway` | Spring Cloud Gateway 网关（路由、限流、鉴权） |
| `pig-auth` | OAuth2 认证授权服务（Spring Authorization Server） |
| `pig-upms/pig-upms-api` | 通用 API：DTO、VO、实体、Feign 接口定义 |
| `pig-upms/pig-upms-biz` | UPMS 业务逻辑：用户、角色、菜单、部门、字典等 |
| `pig-boot` | 单体模式启动器，聚合各业务模块 |
| `pig-common/pig-common-*` | 通用模块（13 个子模块）：安全、数据源、MyBatis、日志、Feign、Swagger 等 |
| `pig-visual/pig-*` | 辅助服务：监控、代码生成、定时任务 |

## 关键架构模式

### 认证与鉴权流程

1. **登录**：前端调用 `/auth/oauth2/token`（OAuth2 密码模式），获取 access_token
2. **请求传递**：前端每次请求在 Header 中携带 `Authorization: Bearer <token>`
3. **网关校验**：[pig-gateway](pig-gateway/) 全局过滤器 (`PigRequestGlobalFilter`) 校验 token 和权限
4. **资源服务器**：[pig-common-security](pig-common/pig-common-security/) 的 `PigResourceServerConfiguration` 配置了 `@EnableMethodSecurity`，各服务通过 `@PreAuthorize` 注解控制权限
5. **Feign 调用**：服务间通过 Feign 调用，[pig-common-feign](pig-common/pig-common-feign/) 的 `PigFeignRequestInterceptor` 自动透传 OAuth2 token

### 动态数据源

[pig-common-datasource](pig-common/pig-common-datasource/) 支持运行时动态切换数据源，通过 `@Ds` 注解声明数据源。

### API 响应格式

统一使用 `com.pig4cloud.pig.common.core.util.R<T>` 作为 API 响应体：
- `R.ok(data)` — 成功响应
- `R.failed(msg)` — 失败响应

### 服务发现与配置

- 微服务模式：所有配置和注册信息由 Nacos（`pig-register`）管理
- 各服务的 `application.yml` 仅包含最基础配置（端口、服务名），其余配置在 Nacos 中
- 单体模式：直接在 `application.yml` 和 `application-dev.yml` 中配置

### 配置文件加密

使用 Jasypt 对敏感配置加密（数据库密码、Redis 密码等），加密密钥通过启动参数或环境变量 `JASYPT_ENCRYPTOR_PASSWORD` 传入。加密值格式为 `ENC(xxx)`。

## 开发注意事项

### 代码格式

项目使用 `spring-javaformat`，CI 中会自动校验。不规范的格式会导致构建失败，提交前务必运行 `mvn spring-javaformat:apply`。

### Profile 切换

- `cloud`（默认）— 微服务模式，激活 `dev` 环境
- `boot` — 额外编译 `pig-boot` 模块
- JDK 25+ 自动激活 `jdk25-annotation-processor` profile

### 多 Java 版本兼容

CI 在 JDK 17/21/25 上运行。新增代码需兼容 Java 17 LTS，如需使用更高版本 API（如虚拟线程），需做兼容处理。

### 数据库初始化

微服务模式首次运行需要初始化两个数据库：
- `pig.sql` — 业务数据库（用户、角色、菜单等）
- `pig_config.sql` — Nacos 配置数据库

SQL 文件在 [db/](db/) 目录下，Docker Compose 会通过 [db/Dockerfile](db/Dockerfile) 自动执行。

### 前端模块独立

[pig-ui/](pig-ui/) 是独立的前端项目，有自己的 `CLAUDE.md`。修改前后端关联功能时，需同时参考前端的 [pig-ui/CLAUDE.md](pig-ui/CLAUDE.md)。

### 依赖更新

项目配置了 Renovate（`.github/renovate.json`）自动管理依赖更新，修改 `pom.xml` 依赖版本时注意与 Renovate 规则一致。

### 🤖 全局行为军规 (Global Core Rules)
1. **Think Before Coding**: 明确假设，不确定先问，避免过度设计。
2. **Simplicity First**: 极简优先，不要写用不到的抽象。
3. **Surgical Changes**: 外科手术式修改，不乱动现有代码。
4. **Goal-Driven Execution**: 目标导向，主动通过测试日志自循环修复。
5. **Token budgets**: 注意 Token 消耗，不静默失败。
   *(Note: 严格遵循 tdx 的 12 条工程原则，遇到冲突大声报错 Fail Loud)*

### 🛠️ AI 技能路由 (Skills Routing)
当 tdx 向你下达任务时，请**首先评估当前所处的阶段**，然后去读取对应路径下的 Skill 文件，严格按照该文件内的标准执行操作，**不要加载不需要的技能**。

- **阶段一：需求分析与设计** -> 读取 `.harness/skills/01-requirement.md`
- **阶段二：数据库设计与建表** -> 读取 `.harness/skills/02-database.md`
- **阶段三：前后端业务代码编写** -> 读取 `.harness/skills/03-coding.md`
- **阶段四：测试与交付验收** -> 读取 `.harness/skills/04-testing.md`
- **阶段五：变更记录 (必须在任务结束前执行)** -> 读取 `.harness/skills/05-changelog.md`