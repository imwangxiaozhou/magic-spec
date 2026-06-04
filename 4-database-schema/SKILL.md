---
name: magic-4-database-schema
description: >
  基于前序文档（技术方案、详细设计、操作流程），生成完整的数据库表结构文档。
  标注每个表的作用、每个字段的描述、类型建议、约束和表间关联关系。
metadata:
  type: spec
  version: "1.0"
  author: wangxz
---

基于前序文档中出现的所有数据库表，提取并整理为一份完整的数据库表结构文档，标注每个表的作用和每个字段的描述。

---

**前置条件**：
- `magic-align-spec/3-operation-flow/overview.md` 必须已生成

**输入**：前序文档（技术方案、详细设计、操作流程）中出现的所有表名和字段。

**输出**：数据库表结构文档（Markdown 格式），按模块拆分为多个文件，写入当前项目目录的 `magic-align-spec/4-database-schema/` 目录下：

```
magic-align-spec/4-database-schema/
├── overview.md              # 文档信息、数据库概览表、ER 关系图
├── M-01-<表英文名>.md       # 单表的字段、索引、关联关系
├── M-02-<表英文名>.md
└── ...
```

- `overview.md` 包含全局信息（数据库概览、ER 关系图）
- 每个 `M-XX-<表英文名>.md` 包含该表的字段列表、索引、关联关系

---

## 执行步骤

### 1. 前置检查

检查 `magic-align-spec/3-operation-flow/overview.md` 是否存在：

- **存在**：读取该目录下的 overview.md 和各 M-XX-*-operation.md 文件，提取所有表名和字段作为输入
- **不存在**：告知用户「尚未生成操作流程文档，请先使用 3-operation-flow 生成后再执行本步骤」，终止流程

### 2. 技术方案确认状态检查

读取 `magic-align-spec/1-tech-implementation-plan/` 目录下 overview.md 中每个技术方案（T-XX）的"确认状态"字段：

- **全部已确认**：正常继续
- **存在待确认或存疑的方案**：使用 **AskUserQuestion** 逐项向用户确认：
  - 待确认的方案：询问"T-XX {方案名称} 的技术选型 {当前选型} 是否确认？如有调整请说明"
  - 存疑的方案：询问"T-XX {方案名称} 当前存疑，原因：{存疑原因}。请确认最终方案或提供替代方案"
  - 根据用户回复，**更新 `magic-align-spec/1-tech-implementation-plan/` 目录下对应方案文件** 中对应方案的确认状态
- 确认完成后继续执行

### 3. 提取表结构信息

从操作流程文档的 INSERT/UPDATE/SELECT 语句、后端影响描述中提取：

- 所有表名
- 每个表的字段名
- 字段的约束信息（NOT NULL、UNIQUE、默认值等）

同时参考技术方案文档和详细设计文档，补充表的作用说明。

### 3. 推断字段类型

根据字段名和业务语义，推断合理的数据库字段类型。例如：
- `id` → UUID 或自增整数
- `name` → VARCHAR(255)
- `description` / `content` → TEXT
- `status` → VARCHAR(50) 或 ENUM
- `*_id` 关联字段 → 与引用表主键同类型（仅逻辑关联，不创建物理外键）
- `*_at` / `*_date` → TIMESTAMP
- `sort_order` → INTEGER
- `visible` → BOOLEAN
- JSON 字段 → JSONB

### 4. 输出

将文档按模块拆分为多个文件，写入 `magic-align-spec/4-database-schema/` 目录：

1. 先写 `overview.md`，包含文档信息、数据库概览表、ER 关系图
2. 再逐表写 `M-XX-<表英文名>.md`，每个文件包含该表的字段列表、索引、关联关系

---

## 文档框架

### overview.md

#### 1. 文档信息

| 字段 | 值 |
|------|-----|
| 项目名称 | （继承自 tech-implementation-plan） |
| 文档版本 | |
| 创建日期 | |
| 前置文档 | magic-align-spec/1-tech-implementation-plan/overview.md, magic-align-spec/2-detailed-design/overview.md, magic-align-spec/3-operation-flow/overview.md |
| 评审状态 | 草稿 / 评审中 / 已通过 |

#### 2. 数据库概览

列出所有表及其作用概要：

| 表名 | 作用 | 关联模块 | 详细文件 |
|------|------|---------|---------|
| agent | | M-01, M-02, M-03 | M-01-agent.md |
| post_tag | | M-04 | M-04-post-tag.md |
| agent_post_tag | | M-01, M-04 | M-01-03-agent-post-tag.md |
| ... | | | |

#### 3. ER 关系图

用 ASCII 图画出表间的逻辑关系（不创建物理外键，关系由应用层维护）：

```
┌──────────┐     ┌──────────────────┐     ┌──────────┐
│  agent   │── .. →│ agent_post_tag   │← .. ──│ post_tag │
└──────────┘     └──────────────────┘     └──────────┘
     │
     ├── .. → agent_skill ← .. ── skill
     │
     └── .. → wxwork_user
```

注：`-- .. →` 表示逻辑关联，不创建物理外键约束。

### M-XX-<表英文名>.md（每表一个文件）

#### 表名（英文表名）

**表作用**：描述该表在系统中的用途

**关联模块**：M-XX, M-XX

**字段列表**：

| 字段名 | 类型 | 约束 | 描述 |
|--------|------|------|------|
| id | UUID / SERIAL | PK, NOT NULL | 主键 |
| name | VARCHAR(255) | NOT NULL, UNIQUE | Agent 名称 |
| description | TEXT | NOT NULL | Agent 能力描述，用于管道路由的语义匹配 |
| status | VARCHAR(50) | NOT NULL, DEFAULT 'enabled' | 状态：enabled=已上架，disabled=已停用 |
| wxwork_user_id | VARCHAR(100) | NOT NULL, UNIQUE | 绑定的企微员工 ID |
| created_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMP | NOT NULL, DEFAULT NOW() | 更新时间 |

**索引**：

| 索引名 | 字段 | 类型 |
|--------|------|------|
| idx_agent_status | status | BTREE |
| idx_agent_wxwork_user_id | wxwork_user_id | UNIQUE |

**关联关系**（逻辑关联，不创建物理外键）：

| 关联表 | 关联字段 | 关系 |
|--------|---------|------|
| agent_post_tag | agent_id | 1:N |
| agent_skill | agent_id | 1:N |
| website_agent_card | agent_id | 1:N |

---

## 编写原则

- **从代码中提取**：所有表名和字段必须来自前序文档中实际出现的内容，不凭空添加
- **类型推断合理**：字段类型基于命名规范和业务语义推断，标注为"建议类型"
- **约束标注清楚**：PRIMARY KEY、NOT NULL、UNIQUE、DEFAULT 等约束必须标注
- **表作用一句话说清**：每个表一句话描述其业务用途
- **关联关系完整**：表间逻辑关联关系（1:N、N:M 等）要画出，但不创建物理外键约束，关联由应用层维护
- **编号一致**：关联模块编号（M-XX）与前序文档保持一致
- **按模块拆分文件**：每张表一个独立文件，overview.md 放概览和 ER 图
