# 🤖 AI 开发上下文与强制约束

> **⚠️ AI代理开发本项目的强制约束文件。违反约束的代码=无效=重写。**
> **首次会话必读，后续按需查阅。日常操作以00-AI开发启动入口.md为准。**

**文档版本**：v4.0
**最后更新**：2026-04-17

---

## 1. 强制前提

```
┌──────────────────────────────────────────────────────────────────┐
│  1. 你在为「自动化装备招聘系统」编写代码                              │
│  2. 本项目有完整技术文档，你不是从零设计，是在文档框架内实现          │
│  3. 必须先读文档，再写代码——绝不凭训练数据自行设计                  │
│  4. 文档是唯一的真相来源，训练知识与文档冲突时以文档为准            │
│  5. 本文件约束优先级高于你的任何"默认行为"或"最佳实践建议"        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心约束速查表

### 2.1 数据库约束（🔴最高优先级）

| ID | 约束 | 违反后果 |
|----|------|---------|
| DB-01 | 表名必须与03-数据库设计一致 | 重写 |
| DB-02 | 字段名必须与03-数据库设计一致 | 重写 |
| DB-03 | 字段类型(VARCHAR长度/INT/TINYINT/ENUM)必须一致 | 重写 |
| DB-04 | 状态枚举值必须与状态机定义一致 | 重写 |
| DB-05 | 外键关系必须与03-数据库设计一致 | 重写 |
| DB-06 | 身份证号/银行卡号必须AES-256加密存储 | 重写 |
| DB-07 | Migration的down()必须完整可回滚 | 修复 |
| DB-08 | 初始数据用Seeder，不在Migration中混入 | 修复 |
| DB-09 | 所有表必须有created_at/updated_at | 修复 |
| DB-10 | 软删除表必须有deleted_at+SoftDeletes trait | 修复 |

### 2.2 API约束（🔴最高优先级）

| ID | 约束 | 违反后果 |
|----|------|---------|
| API-01 | 接口路径/HTTP方法/参数名/响应结构与06规范一致 | 重写 |
| API-02 | 不得自创接口，新增需先更新文档 | 重写 |
| API-03 | 响应格式：`{code, msg, data}`，分页含list/total/page/limit/total_pages | 修复 |
| API-04 | 错误码与错误码规范一致 | 重写 |
| API-05 | 必须使用FormRequest校验，不在Controller直接校验 | 重写 |
| API-06 | 必须使用API Resource组装响应，不直接return Model/数组 | 重写 |
| API-07 | 列表接口必须支持分页(page/per_page) | 修复 |
| API-08 | 鉴权使用JWT Bearer Token，用户端/管理端区分guard | 重写 |
| API-09 | 多表写操作必须使用数据库事务 | 重写 |
| API-10 | 写操作必须有幂等控制(client_msg_id/request_id) | 修复 |

### 2.3 代码架构约束

| ID | 约束 | 违反后果 |
|----|------|---------|
| ARCH-01 | 严格Service分层：Controller→Service→Model，不跨层 | 重写 |
| ARCH-02 | Controller不写业务逻辑，仅调度 | 重写 |
| ARCH-03 | Service不依赖Request/Response对象 | 重写 |
| ARCH-04 | 命名遵循10-开发规范(PascalCase/camelCase/kebab-case) | 修复 |
| ARCH-05 | 目录结构遵循10-开发规范 | 修复 |
| ARCH-06 | PHP 8.2类型声明完整 | 修复 |
| ARCH-07 | 不得留TODO/dd()/dump() | 修复 |
| ARCH-08 | 配置值用config()或system_config，不硬编码 | 修复 |

### 2.4 安全约束

| ID | 约束 |
|----|------|
| SEC-01 | 密码用bcrypt/Hash加密，禁止明文/MD5 |
| SEC-02 | 敏感文件用private disk+签名URL，禁止暴露裸路径 |
| SEC-03 | 高危操作(注销/封禁/删除)需二次确认 |
| SEC-04 | 管理员操作必须写入operation_logs |
| SEC-05 | 限流中间件必须配置(参照08-安全与性能) |

### 2.5 WebSocket约束

| ID | 约束 |
|----|------|
| WS-01 | 事件名用点分命名(message.created, system.kickout) |
| WS-02 | 信封格式：客户端{action,data,ts,request_id}；服务端{event,data,ts,request_id} |
| WS-03 | 写操作必须返回回执，回传request_id |
| WS-04 | 鉴权仅通过ws_token(URL参数)，不用首包鉴权 |
| WS-05 | 断线补偿走HTTP，WebSocket不承担离线消息补偿 |

---

## 3. 禁止行为清单（15条）

| # | 禁止 | 原因 |
|---|------|------|
| 1 | 不读文档直接写代码 | 脱离设计规范 |
| 2 | 自创表名/字段名/接口路径 | 与文档不一致 |
| 3 | Controller中写业务逻辑 | 违反分层 |
| 4 | 跳过FormRequest直接校验 | 安全隐患 |
| 5 | 直接return Model或数组 | 必须用Resource |
| 6 | 明文存储密码/身份证号 | 安全违规 |
| 7 | 硬编码配置值 | 维护困难 |
| 8 | 提交dd()/dump()/console.log() | 调试代码 |
| 9 | 自创状态枚举值 | 与状态机冲突 |
| 10 | 不更新进度跟踪就结束 | 记忆断裂 |
| 11 | 不记录开发决策就跳过 | 无法追溯 |
| 12 | 自行修改文档设计决策 | 需人工确认 |
| 13 | 使用ThinkPHP语法 | 已迁移Laravel 11 |
| 14 | WebSocket用旧type字段 | 必须用action/event契约 |
| 15 | 暴露文件裸路径 | 必须用file_id+File Resource |

---

## 4. 术语速查（禁止使用右列词汇）

| 术语 | 代码值 | 禁止使用 |
|------|--------|---------|
| 求职者 | `role=seeker` | 找工者/工人/应聘者 |
| 雇主 | `role=employer` | 招聘方/老板/用工方 |
| 管理员 | `admin_users`表 | 后台人员/运营 |
| 工种 | `job_categories`表 | 岗位类型/职业类别 |
| 大/中/小工 | `level=senior/medium/junior` | 高级/中级/初级 |
| 报名 | `applications`表 | 投递/申请/应聘 |
| 会话 | `conversations`表，`conversation_key` | 聊天/对话 |
| 置顶 | `is_top`字段 | 精选/推荐 |
| 违禁词 | `banned_words`表 | 敏感词/屏蔽词 |
| 文件标识 | `file_id` | 文件路径/文件URL |

---

## 5. 编码后自检表（每文件必过）

### 数据库相关
- [ ] 表名与03-数据库设计完全一致
- [ ] 字段名/类型与03-数据库设计完全一致
- [ ] 状态枚举值与状态机定义一致
- [ ] 敏感字段使用加密存储
- [ ] Migration的down()可正确回滚

### API相关
- [ ] 接口路径/HTTP方法与06-API规范一致
- [ ] 请求参数名与规范一致
- [ ] 响应格式遵循`{code, msg, data}`
- [ ] 使用FormRequest + Resource
- [ ] 多表写操作使用事务

### 代码架构
- [ ] Controller仅调度，逻辑在Service
- [ ] 命名遵循10-开发规范
- [ ] PHP 8.2类型声明完整
- [ ] 无TODO/dd()/dump()
- [ ] 文件头有锚点注释

---

## 6. 任务执行协议

```
收到任务 → 读AI-DEVELOPMENT-LOG(恢复记忆+读取快照摘要) → 读技术文档(获取规范)
→ 声明锚点清单 → 按步骤清单编码(每步勾选) → 自检(§5) → L1/L2/L3校验(§7) → 更新3个文档 → 完成
```

**关键规则**：
- 开始前先在AI-DEVELOPMENT-LOG中记录计划
- 遇到文档冲突先记录，不得自行决定
- 每完成一个步骤就勾选步骤清单，不得等全部完成
- 代码必须可独立运行，不产出片段
- 标记✅已完成前必须通过三级校验（见§7）
- 存在临时方案必须登记技术债务（项目进度跟踪文档§9）

---

## 7. 进度更新强制校验协议

> **📎 AI将任何条目标记为 `✅ 已完成` 前，必须逐一确认以下三级校验。**
> **完整版见项目进度跟踪文档 附录A。**

### L1 基础校验（必须全部✅）

- [ ] 所有步骤清单项已勾选
- [ ] 代码文件实际存在于磁盘
- [ ] 代码无PHP语法错误（`php -l` 检查通过）

> ⚠️ L1未通过 → 只能标记 `🟡 开发中`

### L2 文档一致性校验（必须全部✅）

- [ ] 请求参数：逐字段对照06-API定义，字段名+类型+必填/可选+验证规则全部匹配
- [ ] 响应格式：逐字段对照06-API定义，字段名+类型+File Resource嵌套+脱敏规则全部匹配
- [ ] 数据库操作：对照03-数据库设计，字段名+类型+默认值+索引+约束全部匹配
- [ ] 业务流转：对照05-功能流程图，状态机转换条件+前置校验+后置动作全部实现
- [ ] 错误处理：对照06-API错误码定义，每个可预见的错误场景有对应处理和正确错误码

> ⚠️ L2未通过 → 只能标记 `🟡 开发中`

### L3 交叉追溯校验

- [ ] 关联数据库表已创建且字段匹配
- [ ] 关联前端页面状态已更新
- [ ] 关联功能模块状态已更新
- [ ] 无未记录的技术债务

> ⚠️ L3未通过 → 可标记 `✅` 但须在项目进度跟踪文档§9登记技术债务

---

## 8. 文件产出路径映射

| 模块 | Controller | Service | Model |
|------|-----------|---------|-------|
| 认证 | `Api/User/AuthController` | `AuthService` | `User` |
| 用户 | `Api/User/UserController` | `UserService` | `User` |
| 简历 | `Api/User/ResumeController` | `ResumeService` | `Resume` |
| 职位(用户) | `Api/User/JobController` | `JobService` | `Job` |
| 报名 | `Api/User/ApplicationController` | `ApplicationService` | `Application` |
| 企业认证 | `Api/User/CompanyController` | `CompanyService` | `Company` |
| 消息 | `Api/User/MessageController` | `MessageService` | `MessagePrivate`等 |
| 雇主-职位 | `Api/Employer/EmployerJobController` | `JobService` | `Job` |
| 雇主-人才库 | `Api/Employer/EmployerTalentPoolController` | `TalentPoolService` | `TalentPool` |
| 管理-认证 | `Api/Admin/AdminAuthController` | `AdminService` | `AdminUser` |
| 管理-用户 | `Api/Admin/AdminUserController` | `UserService` | `User` |
| 管理-企业 | `Api/Admin/AdminCompanyController` | `CompanyService` | `Company` |
| 管理-职位 | `Api/Admin/AdminJobController` | `JobService` | `Job` |
| 管理-举报 | `Api/Admin/AdminReportController` | `ReportService` | `Report` |
| 管理-黑名单 | `Api/Admin/AdminBlacklistController` | `BlacklistService` | `Blacklist` |
| 管理-公告 | `Api/Admin/AdminAnnouncementController` | `AnnouncementService` | `Announcement` |
| 管理-违禁词 | `Api/Admin/AdminBannedWordController` | `BannedWordService` | `BannedWord` |
| 管理-子管理员 | `Api/Admin/AdminSubAdminController` | `AdminService` | `AdminUser` |
| 管理-配置 | `Api/Admin/AdminSystemConfigController` | `SystemConfigService` | `SystemConfig` |
| 管理-操作日志 | `Api/Admin/AdminOperationLogController` | - | `OperationLog` |
| 管理-置顶 | `Api/Admin/AdminTopApplicationController` | `JobService` | `JobTopApplication` |
| 管理-导出 | `Api/Admin/AdminExportController` | `ExportService` | `ExportTask` |
| 管理-广播 | `Api/Admin/AdminBroadcastController` | `BroadcastService` | `SystemBroadcast` |
| 管理-帮助 | `Api/Admin/AdminHelpController` | `HelpService` | `HelpCategory`等 |

完整映射（含FormRequest/Resource/Policy）见 `AI-CODE-ANCHOR.md §2`。

---

**本文件结束 —— AI代理必须严格遵守以上所有约束**
