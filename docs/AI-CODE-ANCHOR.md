# 🔗 AI 代码-文档锚点映射

> **⚠️ 本文件是代码文件与技术文档章节的双向索引。**
> **AI 产出代码时必须在文件头标注锚点，AI 审查代码时必须通过锚点溯源到文档。**
> **没有锚点的代码 = 无源之水 = 必须重写。**

**文档版本**：v1.0
**创建日期**：2026-04-16
**最后更新**：2026-04-16

---

## 1. 锚点ID编码规范

### 格式：`{文档代号}-{章节号或表名}`

| 文档代号 | 对应文档 | 锚点示例 |
|----------|---------|---------|
| ARCH | 02-技术架构 | ARCH-WS（WebSocket契约） |
| DB | 03-数据库设计 | DB-users、DB-jobs、DB-applications |
| UI | 04-页面设计与交互 | UI-4.3（首页）、UI-4.4（职位详情） |
| FLOW | 05-功能流程图 | FLOW-5.1（注册）、FLOW-5.4（报名） |
| API | 06-API接口规范 | API-6.3.1（注册接口）、API-6.6.1（职位列表） |
| SEC | 08-安全与性能 | SEC-8.1（数据安全）、SEC-8.2（接口安全） |
| DEPLOY | 09-部署说明 | DEPLOY-9.2（部署步骤） |
| CODE | 10-开发规范与编码约定 | CODE-10.4（Controller）、CODE-10.5（Service） |

---

## 2. 后端代码文件 ↔ 文档锚点映射

### 2.1 认证模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `AuthController.php` | API-6.3.1, API-6.3.2, API-6.3.3, FLOW-5.1, FLOW-5.2, CODE-10.4 |
| `AuthService.php` | API-6.3.1~6.3.3, DB-users, DB-login_sessions, FLOW-5.2, CODE-10.5 |
| `RegisterRequest.php` | API-6.3.1, DB-users, CODE-10.7 |
| `LoginRequest.php` | API-6.3.2, CODE-10.7 |
| `UserResource.php` | DB-users, CODE-10.8, SEC-8.1（脱敏） |

### 2.2 用户信息模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `UserController.php` | API-6.3.4~6.3.10, CODE-10.4 |
| `UserService.php` | DB-users, DB-resumes, FLOW-5.12（注销）, FLOW-5.20（改密）, CODE-10.5 |
| `UpdateUserRequest.php` | API-6.3.5, DB-users, CODE-10.7 |
| `ChangePhoneRequest.php` | API-6.3.8, FLOW-5.23.2, CODE-10.7 |
| `ChangePasswordRequest.php` | API-6.3.9, FLOW-5.20, CODE-10.7 |
| `DestroyAccountRequest.php` | API-6.3.10, FLOW-5.12, CODE-10.7 |
| `ResumeController.php` | API-6.5.1~6.5.5, CODE-10.4 |
| `ResumeService.php` | DB-resumes, DB-users, CODE-10.5 |
| `ResumeResource.php` | DB-resumes, CODE-10.8 |

### 2.3 企业认证模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `CompanyController.php` | API-6.4.1~6.4.4, CODE-10.4 |
| `CompanyService.php` | DB-companies, DB-company_audit_histories, FLOW-5.9, FLOW-5.19, CODE-10.5 |
| `CompanyCertRequest.php` | API-6.4.2, DB-companies, CODE-10.7 |
| `CompanyResource.php` | DB-companies, CODE-10.8, SEC-8.1（脱敏） |

### 2.4 职位模块（用户端）

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `JobController.php` | API-6.6.1~6.6.8, CODE-10.4 |
| `JobService.php` | DB-jobs, DB-job_levels, DB-job_categories, FLOW-5.3, FLOW-5.11, CODE-10.5 |
| `JobResource.php` | DB-jobs, DB-job_levels, CODE-10.8 |
| `CategoryController.php` | API-6.6.9, DB-job_categories, CODE-10.4 |
| `CategoryResource.php` | DB-job_categories, CODE-10.8 |

### 2.5 报名模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `ApplicationController.php` | API-6.8.1~6.8.5, CODE-10.4 |
| `ApplicationService.php` | DB-applications, DB-job_levels, DB-jobs, FLOW-5.4, FLOW-5.6, CODE-10.5 |
| `ApplicationResource.php` | DB-applications, CODE-10.8 |

### 2.6 消息/通知模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `MessageController.php` | API-6.9.1~6.9.6, CODE-10.4 |
| `MessageService.php` | DB-messages_private, DB-conversations, FLOW-5.7, ARCH-WS, CODE-10.5 |
| `NotificationController.php` | API-6.10.1~6.10.4, DB-notifications, CODE-10.4 |
| `NotificationService.php` | DB-notifications, FLOW-5.28, CODE-10.5 |
| `MessageResource.php` | DB-messages_private, CODE-10.8 |
| `ConversationResource.php` | DB-conversations, CODE-10.8 |
| `NotificationResource.php` | DB-notifications, CODE-10.8 |

### 2.7 雇主端模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `EmployerJobController.php` | API-6.6.1~6.6.8, CODE-10.4 |
| `EmployerApplicationController.php` | API-6.8.6~6.8.8, FLOW-5.5, CODE-10.4 |
| `EmployerTalentPoolController.php` | API-6.10.5~6.10.8, DB-talent_pool, FLOW-5.27, CODE-10.4 |
| `EmployerStatsController.php` | API-6.10.9, CODE-10.4 |
| `TalentPoolService.php` | DB-talent_pool, FLOW-5.27, FLOW-5.36, CODE-10.5 |

### 2.8 管理端模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `AdminAuthController.php` | API-6.11.1~6.11.2, FLOW-5.29, CODE-10.4 |
| `AdminService.php` | DB-admin_users, DB-admin_user_permissions, FLOW-5.29, FLOW-5.30, FLOW-5.31, CODE-10.5 |
| `AdminUserController.php` | API-6.11.3~6.11.5, CODE-10.4 |
| `AdminCompanyController.php` | API-6.11.6~6.11.8, FLOW-5.9.1, CODE-10.4 |
| `AdminJobController.php` | API-6.11.9~6.11.13, FLOW-5.9.2, CODE-10.4 |
| `AdminReportController.php` | API-6.11.14~6.11.16, FLOW-5.8, CODE-10.4 |
| `AdminBlacklistController.php` | API-6.11.17~6.11.22, FLOW-5.10, FLOW-5.41, CODE-10.4 |
| `AdminAnnouncementController.php` | API-6.11.23~6.11.25, CODE-10.4 |
| `AdminBannedWordController.php` | API-6.11.26~6.11.28, FLOW-5.14, CODE-10.4 |
| `AdminSubAdminController.php` | API-6.11.29~6.11.32, FLOW-5.31, CODE-10.4 |
| `AdminSystemConfigController.php` | API-6.11.33~6.11.34, FLOW-5.32, CODE-10.4 |
| `AdminOperationLogController.php` | API-6.11.35, DB-operation_logs, CODE-10.4 |
| `AdminTopApplicationController.php` | API-6.11.36~6.11.37, FLOW-5.13, CODE-10.4 |
| `AdminExportController.php` | API-6.11.38~6.11.39, FLOW-5.38, DB-export_tasks, CODE-10.4 |
| `AdminBroadcastController.php` | API-6.11.40~6.11.42, FLOW-5.39, DB-system_broadcasts, CODE-10.4 |
| `AdminHelpController.php` | API-6.11.43~6.11.50, FLOW-5.42, CODE-10.4 |

### 2.9 通用/基础模块

| 代码文件 | 必须声明的锚点 |
|----------|---------------|
| `FileService.php` | DB-files, SEC-8.1, ARCH-2.5 |
| `BannedWordService.php` | DB-banned_words, FLOW-5.14, SEC-8.3 |
| `SystemConfigService.php` | DB-system_config, DB-config_change_logs, FLOW-5.32 |
| `ExportService.php` | DB-export_tasks, FLOW-5.38 |

---

## 3. 数据库迁移文件 ↔ 文档锚点映射

| 迁移文件 | 必须声明的锚点 |
|----------|---------------|
| `create_users_table` | DB-users |
| `create_companies_table` | DB-companies |
| `create_company_audit_histories_table` | DB-company_audit_histories |
| `create_job_categories_table` | DB-job_categories |
| `create_files_table` | DB-files |
| `create_jobs_table` | DB-jobs |
| `create_job_levels_table` | DB-job_levels |
| `create_resumes_table` | DB-resumes |
| `create_applications_table` | DB-applications |
| `create_conversations_table` | DB-conversations |
| `create_messages_private_table` | DB-messages_private |
| `create_messages_public_table` | DB-messages_public |
| `create_comments_table` | DB-comments |
| `create_comment_likes_table` | DB-comment_likes |
| `create_reports_table` | DB-reports |
| `create_blacklist_table` | DB-blacklist |
| `create_favorites_table` | DB-favorites |
| `create_notifications_table` | DB-notifications |
| `create_system_config_table` | DB-system_config |
| `create_admin_users_table` | DB-admin_users |
| `create_admin_user_permissions_table` | DB-admin_user_permissions |
| `create_admin_login_logs_table` | DB-admin_login_logs |
| `create_operation_logs_table` | DB-operation_logs |
| `create_config_change_logs_table` | DB-config_change_logs |
| `create_banned_words_table` | DB-banned_words |
| `create_announcements_table` | DB-announcements |
| `create_job_views_table` | DB-job_views |
| `create_category_applications_table` | DB-category_applications |
| `create_job_top_applications_table` | DB-job_top_applications |
| `create_talent_pool_table` | DB-talent_pool |
| `create_export_tasks_table` | DB-export_tasks |
| `create_system_broadcasts_table` | DB-system_broadcasts |
| `create_help_categories_table` | DB-help_categories |
| `create_help_articles_table` | DB-help_articles |
| `create_help_article_feedbacks_table` | DB-help_article_feedbacks |
| `create_help_search_synonyms_table` | DB-help_search_synonyms |
| `create_help_article_relations_table` | DB-help_article_relations |
| `create_help_search_logs_table` | DB-help_search_logs |
| `create_login_sessions_table` | DB-login_sessions |
| `create_password_reset_requests_table` | DB-password_reset_requests |

---

## 4. 已实现代码登记区

> **AI 每次产出代码后必须在此登记。**
> **格式：`| 产出日期 | 文件路径 | 任务编号 | 锚点列表 |`**

| 产出日期 | 文件路径 | 任务编号 | 锚点列表 |
|----------|----------|----------|----------|
| - | - | - | - |

---

## 5. 锚点校验命令

AI 在编码完成后，对每个文件执行以下校验：

```
对每个产出的代码文件，逐项检查：

□ 文件头部是否有文档锚点注释？
□ 声明的锚点是否与本映射表一致？
□ DB类锚点：表名/字段名/类型是否与03-数据库设计完全一致？
□ API类锚点：路径/参数/响应是否与06-API接口规范完全一致？
□ FLOW类锚点：业务逻辑是否与05-功能流程图完全一致？
□ CODE类锚点：代码结构是否与10-开发规范模板完全一致？
□ SEC类锚点：安全处理是否与08-安全与性能完全一致？

任一不通过 → 修改代码，不得跳过
```

---

**本文件结束 —— AI 代理必须在每次产出代码后更新 §4 已实现代码登记区**
