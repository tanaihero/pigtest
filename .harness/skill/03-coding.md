# Skill 3: 代码生成与模块规范 (Code Generation Standard)

**触发条件 (When to use):** 当数据库设计确认完毕，准备开始编写前端和后端业务代码时。

**执行标准 (Standards):**
1. **微服务目录划分**：如果是全新的业务模块，**必须**在与 `pig-upms-biz` 同级的目录下新建包（例如：订单模块建立为 `pig-upms-order` 目录）。
2. **后端 (Java)**：
    - 严格遵循 Controller -> Service -> Mapper 分层。
    - 统一使用 Pig 框架内置的 `R<T>` 进行 API 结果包装。
    - 遵循 Simplicity First，不要过度设计抽象类。
3. **前端 (Vue 3 / TypeScript)**：
    - **强制要求**：每次新增调用后端的 API，**必须**先编写与之对应的 TypeScript Interface 声明文件，不能使用 `any`。
    - 使用 `<script setup lang="ts">` 语法。
    - 样式优先使用 Tailwind CSS 实用类，UI 组件使用 Element Plus。