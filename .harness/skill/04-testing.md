# Skill 4: 测试与验收标准 (Testing Standard)

**触发条件 (When to use):** 当业务逻辑代码编写完成，准备向 tdx 交付验收前。

**执行标准 (Standards):**
1. **验证意图**：后端核心业务逻辑必须包含单元测试，不仅测试主流程，必须包含异常分支测试。
2. **Self-Review**：交付前主动检查代码是否引入了未使用的 Import，是否破坏了项目现有的代码格式（Match existing style）。
3. 确保所有改动严格锁定在最小范围内（Surgical Changes）。