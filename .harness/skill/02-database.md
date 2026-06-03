# Skill 2: 数据库建表规范 (Database Design Standard)

**触发条件 (When to use):** 当明确需求需要持久化，准备生成 SQL 脚本或修改数据库结构时。

**执行标准 (Standards):**
1. **引擎与字符集**：必须使用 `ENGINE=InnoDB` 和 `CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci`。
2. **命名规范**：表名和字段名全小写，下划线分割（`snake_case`）。每个表和每个字段必须包含 `COMMENT`。
3. **主键设计**：主键统一使用 `bigint`，名称为 `id`。在代码层面，必须配置为 MyBatis-Plus 的**雪花算法**（`@TableId(type = IdType.ASSIGN_ID)`）。
4. **公共审计字段 (必留)**：必须包含以下 5 个字段：
    - `create_by` varchar(64) DEFAULT NULL COMMENT '创建人'
    - `update_by` varchar(64) DEFAULT NULL COMMENT '修改人'
    - `create_time` datetime DEFAULT NULL COMMENT '创建时间'
    - `update_time` datetime DEFAULT NULL COMMENT '修改时间/更新时间'
    - `del_flag` char(1) DEFAULT '0' COMMENT '删除标志'（0为正常，1为删除）
5. **禁止使用外键**：关闭外键检查（`SET FOREIGN_KEY_CHECKS = 0`），由代码保障数据一致性。