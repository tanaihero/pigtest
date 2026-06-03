# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

pig-ui 是 PIGCLOUD 微服务开发平台的前端项目，基于 Vue 3 + TypeScript + Vite + Element Plus + Tailwind CSS。

## 常用命令

```bash
npm run dev          # 启动开发服务器（--force 强制重新构建）
npm run build        # 生产构建 (NODE_OPTIONS=--max-old-space-size=4096)
npm run build:docker # Docker 构建，输出到 ./docker/dist/
npm run lint:eslint  # ESLint 检查并自动修复
npm run prettier     # Prettier 格式化
```

## 路径别名

`/@` → `src/`（在 `vite.config.ts` 和 `tsconfig.json` 中配置）。

## 架构概览

### 路由（src/router/）

**核心模式：后端控制路由。** `themeConfig.isRequestRoutes` 默认为 `true`，路由菜单由后端 `/admin/menu` 接口动态返回，前端通过 `import.meta.glob` 自动匹配 `views/` 和 `layout/routerView/` 下的 `.vue/.tsx` 文件作为组件。

关键文件：
- `router/index.ts` — 路由实例创建、守卫逻辑、工具函数
- `router/route.ts` — 静态路由定义（login、home、personal、404、401、baseRoutes 根布局）
- `router/backEnd.ts` — 后端控制路由初始化（`initBackEndControlRoutes`），处理 component 转换

路由生命周期：`beforeEach` 守卫 → 无 token 跳转 `/login` → 已有 token 则调用 `initBackEndControlRoutes()` → 拉取用户信息 + 菜单数据 → 动态 `addRoute`。

### API 层（src/api/ + src/utils/request.ts）

- 所有接口调用放在 `src/api/` 下，按模块分目录（admin、login、daemon、gen）
- Axios 实例 `src/utils/request.ts`：
  - 自动从 Session(Cookie) 注入 `Authorization: Bearer <token>`
  - 自动注入 `TENANT-ID` 请求头
  - 支持请求报文加密（`Enc-Flag` 头）和响应解密
  - 自动适配微服务/单体架构 URL（`other.adaptationUrl`）
  - 424 状态码 → 弹出确认框，清除缓存并跳转登录页
- **调用模板**：参考 `src/api/admin/` 下的文件，使用 `await` + 解构的同步写法
- OAuth2 密码模式认证：登录接口为 `/auth/oauth2/token`，form 格式提交

### 状态管理（src/stores/）

- `userInfo` — 用户信息、登录/登出/刷新 token
- `routesList` — 后端返回的动态路由列表
- `themeConfig` — 全局主题/布局配置（持久化到 localStorage）
- `tagsViewRoutes` — 已打开页签的路由
- `keepAliveNames` — 缓存组件名称列表
- 使用 `pinia-plugin-persist` 做持久化

### 国际化（src/i18n/）

- Vue I18n（legacy 模式关闭），默认语言 `zh-cn`
- 通过 `import.meta.glob` 自动收集：
  - `src/i18n/lang/` 下框架级文案
  - `src/i18n/pages/` 下页面级文案
  - 任意 `views/**/i18n/` 目录下的 `*.ts` 文件
- 组件中使用：`const { t } = useI18n()`，文件命名用驼峰
- Element Plus 国际化自动集成

### 布局系统（src/layout/）

支持多种布局模式（defaults / classic / transverse / columns），通过 `themeConfig.layout` 切换。核心目录：
- `layout/index.vue` — 布局主入口
- `layout/navMenu/` — 侧边栏/顶部导航菜单
- `layout/navBars/` — 顶栏（面包屑、标签页、设置抽屉）
- `layout/routerView/` — 路由视图容器（parent.vue 缓存、iframes.vue 内嵌、link.vue 外链）
- `layout/lockScreen/` — 自动锁屏

### 主题（src/theme/）

- SCSS 变量定制 Element Plus 主题（`element.scss`）
- 暗黑模式（`dark.scss`）：使用 CSS 自定义属性（`--next-*`），Twitter 风格配色
- Tailwind CSS 通过 `tailwind.config.js` 与 Element Plus 主题统一
- 支持灰色模式/色弱模式

### 全局组件（在 main.ts 中注册）

`DictTag`、`Pagination`、`RightToolbar`、`UploadExcel`、`UploadFile`、`UploadImg`、`Editor`、`Tip`、`DelWrap`、`Splitpanes`/`Pane`。

### 全局属性

- `parseTime` / `parseDate` / `dateTimeStr` / `dateStr` / `timeStr` — 时间格式化
- `baseURL` — `VITE_API_URL` 环境变量值

### 环境变量

- `.env` — 通用配置（微服务开关、API 前缀、OAuth2 客户端信息、加密密钥等）
- `.env.development` — 开发环境（端口 8888，代理到 `http://127.0.0.1:9999`）

### 存储策略

- `Session`（`src/utils/storage.ts`）：token/refresh_token 用 Cookie 存储，其他用 sessionStorage
- `Local`：localStorage，key 自动加项目名前缀（`__NEXT_NAME__`）

### 自动导入

`unplugin-auto-import` 自动导入 `vue`、`vue-router`、`pinia` 的 API，无需手动 import `ref`、`computed`、`useRoute` 等。
