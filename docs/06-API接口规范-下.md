第六章：API接口规范（下）
================================================================================

索引
================================================================================
6.12 人才库相关接口
6.13 其他补充接口

【6.11.6 企业认证详情】
GET /api/admin/company-certifications/{id}/detail

请求参数：
- id: 认证ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "user_id": 10087,
    "company_name": "XX建设集团",
    "unified_social_code": "91310000XXXXXXXXXX",
    "company_type": "enterprise",
    "business_license_file": {
      "file_id": 201,
      "disk": "private",
      "path": "company/2024/03/license.jpg",
      "access": "private",
      "url": null,
      "temporary_url": "https://.../license.jpg?sign=xxx&expires=1709730000",
      "mime_type": "image/jpeg",
      "size": 512345,
      "original_name": "business-license.jpg"
    },  // 通过 temporary_url 访问，需 admin:cert:view 权限
    "id_card_front_file": null,
    "id_card_back_file": null,
    "legal_person_name": "张经理",
    "legal_person_id_card": "310***********1234",  // 脱敏显示，如需明文需独立授权接口
    "legal_person_id_card_front_file": {
      "file_id": 202,
      "disk": "private",
      "path": "company/2024/03/legal-front.jpg",
      "access": "private",
      "url": null,
      "temporary_url": "https://.../legal-front.jpg?sign=xxx&expires=1709730000",
      "mime_type": "image/jpeg",
      "size": 345678,
      "original_name": "legal-front.jpg"
    },
    "legal_person_id_card_back_file": {
      "file_id": 203,
      "disk": "private",
      "path": "company/2024/03/legal-back.jpg",
      "access": "private",
      "url": null,
      "temporary_url": "https://.../legal-back.jpg?sign=xxx&expires=1709730000",
      "mime_type": "image/jpeg",
      "size": 356789,
      "original_name": "legal-back.jpg"
    },
    "contact_person": "李人事",
    "contact_phone": "13800138000",
    "audit_status": 0,
    "audit_operator_id": null,
    "audit_time": null,
    "audit_reason": null,
    "audit_version": 2,
    "reapply_count": 1,
    "created_at": "2024-03-06 09:00:00",
    "user_info": {  // 申请人信息
      "nickname": "张经理",
      "phone": "13800138000",
      "register_time": "2024-01-15",
      "job_count": 15,
      "application_count": 128
    },
    "history": [  // 历史认证记录（含材料快照）
      {
        "version": 1,
        "audit_status": 2,
        "audit_reason": "证件不清晰",
        "audit_time": "2024-03-01 10:00:00",
        "submitted_at": "2024-03-01 09:30:00",
        "materials": {
          "business_license_file": {
            "file_id": 301,
            "disk": "private",
            "path": "company/history/v1/license.jpg",
            "access": "private",
            "url": null,
            "temporary_url": "https://.../history-license-v1.jpg?sign=xxx&expires=1709730000",
            "mime_type": "image/jpeg",
            "size": 498765,
            "original_name": "history-license-v1.jpg"
          },
          "id_card_front_file": null,
          "id_card_back_file": null,
          "legal_person_name": "张经理",
          "legal_person_id_card": "310***********1234",
          "legal_person_id_card_front_file": {
            "file_id": 302,
            "disk": "private",
            "path": "company/history/v1/legal-front.jpg",
            "access": "private",
            "url": null,
            "temporary_url": "https://.../history-legal-front-v1.jpg?sign=xxx&expires=1709730000",
            "mime_type": "image/jpeg",
            "size": 287654,
            "original_name": "history-legal-front-v1.jpg"
          },
          "legal_person_id_card_back_file": {
            "file_id": 303,
            "disk": "private",
            "path": "company/history/v1/legal-back.jpg",
            "access": "private",
            "url": null,
            "temporary_url": "https://.../history-legal-back-v1.jpg?sign=xxx&expires=1709730000",
            "mime_type": "image/jpeg",
            "size": 298765,
            "original_name": "history-legal-back-v1.jpg"
          },
          "contact_person": "李人事",
          "contact_phone": "13800138000"
        }
      }
    ]
  }
}

说明：
- 当前认证详情由 companies + users 主查询返回
- 当前认证材料及 history.materials 中的资质文件均为 private 文件，返回的 `temporary_url` 为服务端按需生成的临时签名地址，有效期建议15分钟
- 后台查看当前材料与历史材料时，须校验管理员具备 `admin:cert:view` 权限；校验通过后才返回对应 `temporary_url`
- history 数组及其中的 materials 字段由 company_audit_histories 历史快照组装返回
- 审核详情页可直接使用 history.materials 渲染"查看上次材料"能力，无需额外定义独立历史材料接口

【6.11.7 审核企业认证】
POST /api/admin/company-certifications/{id}/audit

请求参数：
{
  "id": 1,
  "action": "approve",     // approve通过/reject拒绝
  "audit_reason": "证件清晰，信息真实"
}

说明：
- 属于高危后台写操作（审核），`risk_level` 建议按结果区分：`approve` 可记为 `medium`，`reject` 至少记为 `high`
- `audit_reason` 必填，并作为企业认证审核决定的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 仅允许审核 `status = pending` 的企业认证记录，避免重复审核或越权改写已生效结论
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=company_certification`、`request_id`、`result`、`risk_level`、`reason=audit_reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含认证主体摘要、审核前后状态、驳回原因摘要与材料校验结论摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、审核状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "审核完成",
  "data": null
}

【6.11.8 职位列表】
GET /api/admin/job/list

请求参数：
- status: 状态筛选（draft/pending/rejected/active/paused/expired/closed/deleted，可选）
- audit_status: 审核状态（0待审核/1已通过/2已拒绝，可选）
- keyword: 标题/编号/发布人（可选）
- start_date: 发布开始日期（可选）
- end_date: 发布结束日期（可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "job_no": "JOB202403060001",
        "title": "电工招聘",
        "user_id": 10087,
        "user_name": "张经理",
        "company_name": "XX建设集团",
        "work_location": "上海市浦东新区",
        "category_names": ["电工", "焊工"],
        "status": "active",
        "audit_status": 1,
        "is_top": 1,
        "top_expire_time": "2024-03-13 10:00:00",
        "views_count": 1258,
        "applications_count": 15,
        "created_at": "2024-03-06 10:00:00",
        "refreshed_at": "2024-03-06 15:00:00"
      }
    ],
    "total": 5680,
    "page": 1,
    "limit": 20,
    "stats": {
      "total": 5680,
      "active": 3200,
      "pending": 45,
      "expired": 1800,
      "deleted": 635
    }
  }
}

【6.11.9 后台职位详情】
GET /api/admin/job/detail

请求参数：
- id: 职位ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "job_no": "JOB202403060001",
    "title": "电工招聘",
    "user_id": 10087,
    "user_name": "张经理",
    "company_name": "XX建设集团",
    "work_location": "上海市浦东新区",
    "work_period": "3个月",
    "work_time": "8:00-17:00",
    "settlement_type": "daily",
    "accommodation": 1,
    "meals": 1,
    "requirements": "有电工证，3年以上经验",
    "contact_person": "张经理",
    "contact_phone": "13800138000",
    "status": "active",
    "audit_status": 1,
    "audit_reason": null,
    "is_top": 1,
    "top_expire_time": "2024-03-13 10:00:00",
    "views_count": 1258,
    "applications_count": 15,
    "comments_count": 8,
    "created_at": "2024-03-06 10:00:00",
    "updated_at": "2024-03-06 15:00:00",
    "levels": [  // 职位级别明细
      {
        "category_id": 1,
        "category_name": "电工",
        "level": "senior",
        "level_name": "大工",
        "salary": 350.00,
        "recruit_count": 5,
        "applied_count": 3
      }
    ],
    "images": [
      {
        "file_id": 101,
        "disk": "public",
        "path": "jobs/2024/03/06/img_001.jpg",
        "access": "public",
        "url": "https://.../1.jpg",
        "temporary_url": null,
        "mime_type": "image/jpeg",
        "size": 245678,
        "original_name": "job-image1.jpg"
      },
      {
        "file_id": 102,
        "disk": "public",
        "path": "jobs/2024/03/06/img_002.jpg",
        "access": "public",
        "url": "https://.../2.jpg",
        "temporary_url": null,
        "mime_type": "image/jpeg",
        "size": 267890,
        "original_name": "job-image2.jpg"
      }
    ]
  }
}

【6.11.10 审核职位】
PUT /api/admin/job/audit

请求参数：
{
  "id": 1,
  "action": "approve",  // approve/reject
  "reason": "内容合规"
}

说明：
- 属于高危后台写操作（审核），`risk_level` 建议按结果区分：`approve` 可记为 `medium`，`reject` 至少记为 `high`
- `reason` 必填，并作为审核决定的审计原因；批量审核场景必须为整批请求生成统一 `request_id`
- 仅允许审核 `status = pending` 的职位
- `pending` 的触发条件包括：①新认证企业前N条职位（默认10条）需审核；②命中职位审核规则（如违规词、敏感内容、图片异常、联系方式异常、风控策略）；③管理员强制要求审核
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=job`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含审核前后状态、驳回原因摘要与命中规则摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或审核状态不允许变更，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "审核完成",
  "data": null
}

【6.11.11 删除职位】
DELETE /api/admin/job/delete

请求参数：
{
  "id": 1,
  "reason": "违规内容"
}

说明：
- 属于高危后台写操作（删除），`risk_level` 至少为 `high`
- `reason` 必填，并作为删除职位的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=job`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含职位状态、发布主体摘要、删除原因摘要与删除结果
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "职位已删除",
  "data": null
}

【6.11.12 工种管理】
GET /api/admin/category/list

请求参数：
- is_enabled: 启用状态（可选）
- page, limit

成功响应：（工种列表含排序、状态信息）

【6.11.12.1 新增工种】
POST /api/admin/category/add

请求参数：
{
  "name": "数控车工",
  "icon": "cnc",
  "sort_order": 10
}

成功响应：
{
  "code": 200,
  "msg": "工种已添加",
  "data": {"id": 9}
}

【6.11.12.2 编辑工种】
PUT /api/admin/category/update

请求参数：
{
  "id": 1,
  "name": "电工",
  "icon": "electric",
  "sort_order": 1,
  "is_enabled": 1
}

成功响应：
{
  "code": 200,
  "msg": "工种已更新",
  "data": null
}

【6.11.12.3 删除工种】
DELETE /api/admin/category/delete

请求参数：
{
  "id": 1,
  "reason": "工种已废弃且无继续维护必要"
}

说明：
- 属于高危后台写操作（删除），`risk_level` 至少为 `high`
- `reason` 必填，并作为工种删除的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 删除前必须明确工种名称、启用状态与关联职位/申请数量，避免误删仍在使用中的工种配置
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=category`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含工种名称、排序、状态与关联数量摘要，`after_snapshot` 记录删除结果摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、存在关联职位/申请或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

错误响应（有关联职位时）：
{
  "code": 4003,
  "msg": "该工种下存在职位，无法删除",
  "data": null
}

【6.11.12.4 工种申请列表】
GET /api/admin/category/applications

请求参数：
- status: 状态（pending/approved/rejected，默认pending）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "user_id": 10086,
        "user_name": "张三",
        "name": "数控车工",
        "reason": "市场需求大，建议添加",
        "status": "pending",
        "created_at": "2024-03-06 10:00:00"
      }
    ],
    "total": 3,
    "page": 1,
    "limit": 20
  }
}

【6.11.12.5 处理工种申请】
PUT /api/admin/category/handle-application

请求参数：
{
  "id": 1,
  "action": "approve",    // approve批准/reject拒绝
  "handle_reason": "需求合理，已添加"
}

成功响应：
{
  "code": 200,
  "msg": "已处理",
  "data": null
}

【6.11.14 举报列表】
GET /api/admin/report/list

请求参数：
- status: 状态（pending待处理/processing处理中/resolved已处理/rejected已驳回，可选）
- report_type: 举报类型（job职位/comment评论/user用户/company企业，可选）
- start_date: 开始日期（可选）
- end_date: 结束日期（可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "report_type": "job",
        "target_id": 123,
        "target_title": "电工招聘",  // 被举报内容标题
        "reason": "虚假招聘",
        "reason_detail": "联系后要求先交押金",
        "evidence_files": [
          {
            "file_id": "f_20240306_evidence003",
            "disk": "private",
            "path": "reports/2024/03/06/evidence003.jpg",
            "access": "private",
            "url": null,
            "temporary_url": "https://...",
            "mime_type": "image/jpeg",
            "size": 204800,
            "original_name": "screenshot1.jpg"
          }
        ],
        "reporter_id": 10086,
        "reporter_name": "张三",
        "reporter_phone": "138****8888",
        "status": "pending",
        "handler_id": null,
        "handler_name": null,
        "handle_result": null,
        "handle_time": null,
        "created_at": "2024-03-06 10:00:00"
      },
      {
        "id": 2,
        "report_type": "comment",
        "target_id": 456,
        "target_title": "评论内容预览...",
        "reason": "辱骂他人",
        "reason_detail": "评论中包含人身攻击",
        "evidence_files": [],
        "reporter_id": 10087,
        "reporter_name": "李四",
        "reporter_phone": "139****9999",
        "status": "resolved",
        "handler_id": 1,
        "handler_name": "管理员A",
        "handle_result": "已删除违规评论，警告用户",
        "handle_time": "2024-03-06 14:00:00",
        "created_at": "2024-03-06 09:00:00"
      }
    ],
    "total": 156,
    "page": 1,
    "limit": 20,
    "stats": {
      "pending": 12,
      "processing": 5,
      "resolved": 128,
      "rejected": 11
    }
  }
}

【6.11.14.1 举报详情】
GET /api/admin/report/detail

请求参数：
- id: 举报ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "report_type": "job",
    "target_id": 123,
    "target_content": {  // 被举报内容详情
      "job_no": "JOB202403060001",
      "title": "电工招聘",
      "content": "...",
      "user_name": "张经理",
      "user_phone": "13800138000"
    },
    "reason": "虚假招聘",
    "reason_detail": "联系后要求先交押金",
    "evidence_files": [
      {
        "file_id": "f_20240306_evidence004",
        "disk": "private",
        "path": "reports/2024/03/06/evidence004.jpg",
        "access": "private",
        "url": null,
        "temporary_url": "https://...",
        "mime_type": "image/jpeg",
        "size": 215040,
        "original_name": "screenshot1.jpg"
      }
    ],
    "reporter": {
      "id": 10086,
      "name": "张三",
      "phone": "138****8888"
    },
    "status": "pending",
    "handler": null,
    "handle_result": null,
    "handle_time": null,
    "created_at": "2024-03-06 10:00:00"
  }
}

【6.11.15 处理举报】
PUT /api/admin/report/handle

请求参数：
{
  "id": 1,
  "action": "resolve",        // resolve已处理/reject已驳回
  "handle_result": "已删除违规内容，用户已警告",
  "measures": ["delete_content", "warn_user"]  // 执行的措施
}

说明：
- 属于高危后台写操作（审核/处置），`risk_level` 至少为 `high`
- `handle_result` 必填，并作为举报处理决定的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- `measures` 需与 `action`、`handle_result` 保持一致；如涉及删除内容、封禁主体、加入黑名单等联动处置，必须通过同一 `request_id` 串联主操作与后续明细结果
- 仅允许处理 `status = pending` 的举报记录，避免重复处理或覆盖既有处置结论
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=report`、`request_id`、`result`、`risk_level`、`reason=handle_result`、`before_snapshot`、`after_snapshot`；其中快照至少包含举报状态、举报类型、处置措施摘要、关联对象摘要与处理前后结果摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、举报状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "举报已处理",
  "data": null
}

【6.11.16 黑名单列表】
GET /api/admin/blacklist/list

请求参数：
- blacklist_type: 类型（user用户/company企业/labor中介，可选）
- is_lifted: 是否已解除（0否/1是，可选）
- is_public: 是否公示（0否/1是，可选）
- keyword: 关键词搜索（昵称/手机号/企业名称，可选）
- start_date: 开始日期（可选）
- end_date: 结束日期（可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "blacklist_type": "user",
        "target_id": 10086,
        "target_name": "张三",
        "target_phone": "138****8888",
        "reason": "多次虚假发布",
        "reason_detail": "发布虚假招聘信息，骗取求职者钱财",
        "evidence_files": [
          {
            "file_id": "f_20240306_evidence003",
            "disk": "private",
            "path": "reports/2024/03/06/evidence003.jpg",
            "access": "private",
            "url": null,
            "temporary_url": "https://...",
            "mime_type": "image/jpeg",
            "size": 204800,
            "original_name": "screenshot1.jpg"
          }
        ],
        "is_public": 1,
        "public_level": "all",
        "public_reason": "虚假招聘",
        "expire_at": null,  // null表示永久封禁
        "is_lifted": 0,
        "lifted_at": null,
        "lift_reason": null,
        "operator_id": 1,
        "operator_name": "管理员A",
        "created_at": "2024-03-06 10:00:00"
      },
      {
        "id": 2,
        "blacklist_type": "user",
        "target_id": 10087,
        "target_name": "李四",
        "target_phone": "139****9999",
        "reason": "骚扰行为",
        "reason_detail": "多次骚扰其他用户",
        "evidence_files": [],
        "is_public": 0,
        "public_level": null,
        "public_reason": null,
        "expire_at": "2024-04-06 10:00:00",  // 限时封禁
        "is_lifted": 0,
        "lifted_at": null,
        "lift_reason": null,
        "operator_id": 2,
        "operator_name": "管理员B",
        "created_at": "2024-03-06 09:00:00"
      }
    ],
    "total": 256,
    "page": 1,
    "limit": 20,
    "stats": {
      "total": 256,
      "active": 180,  // 生效中
      "lifted": 56,   // 已解除
      "public": 120   // 公示中
    }
  }
}

【6.11.16.1 黑名单详情】
GET /api/admin/blacklist/detail

请求参数：
- id: 黑名单记录ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "blacklist_type": "user",
    "target": {
      "id": 10086,
      "name": "张三",
      "phone": "138****8888",
      "avatar_file": {
        "file_id": "f_20240306_avatar003",
        "disk": "public",
        "path": "avatars/2024/03/06/avatar003.jpg",
        "access": "public",
        "url": "https://...",
        "temporary_url": null,
        "mime_type": "image/jpeg",
        "size": 101376,
        "original_name": "avatar.jpg"
      },
      "role": "employer"
    },
    "reason": "多次虚假发布",
    "reason_detail": "发布虚假招聘信息，骗取求职者钱财",
    "evidence_files": [
      {
        "file_id": "f_20240306_evidence004",
        "disk": "private",
        "path": "reports/2024/03/06/evidence004.jpg",
        "access": "private",
        "url": null,
        "temporary_url": "https://...",
        "mime_type": "image/jpeg",
        "size": 215040,
        "original_name": "screenshot1.jpg"
      }
    ],
    "is_public": 1,
    "public_level": "all",
    "public_reason": "虚假招聘",
    "public_time": "2024-03-06 10:00:00",
    "expire_at": null,
    "is_lifted": 0,
    "lifted_at": null,
    "lift_reason": null,
    "lift_operator": null,
    "operator": {
      "id": 1,
      "name": "管理员A"
    },
    "created_at": "2024-03-06 10:00:00",
    "related_jobs": [  // 关联的违规职位
      {
        "id": 123,
        "title": "高薪电工招聘"
      }
    ]
  }
}

【6.11.16.2 设置黑名单公示状态】
POST /api/admin/blacklist/{id}/set-public

请求参数：
{
  "id": 1,
  "is_public": 1,
  "public_level": "all",
  "public_reason": "虚假招聘"
}

说明：
- 属于高危后台写操作（公示状态调整），`risk_level` 至少为 `high`；如从非公开改为全站公开，建议记为 `critical`
- 用于黑名单创建后，单独调整是否公示、公示级别与公示原因
- `public_reason` 必填，并作为本次公示状态调整的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 当 `is_public = 1` 时，`public_level` 与 `public_reason` 必填，服务端写入 `public_time = 当前时间`
- 当 `is_public = 0` 时，服务端应清空 `public_level`、`public_time`、`public_reason`
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=blacklist`、`request_id`、`result`、`risk_level`、`reason=public_reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含公示前后状态、公示级别、公示时间与对外展示范围摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、黑名单状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "公示设置已更新",
  "data": {
    "id": 1,
    "is_public": 1,
    "public_level": "all",
    "public_reason": "虚假招聘",
    "public_time": "2024-03-06 10:00:00"
  }
}

【6.11.16.3 编辑黑名单】
PUT /api/admin/blacklist/update

请求参数：
{
  "id": 1,
  "reason": "多次虚假发布并骚扰求职者",
  "evidence_file_ids": ["f_20240306_evidence005"],
  "expire_at": "2024-12-31 23:59:59"
}

说明：
- 属于高危后台写操作（处罚依据调整），`risk_level` 至少为 `high`
- 用于修改封禁原因、补充证据、调整封禁期限等基础信息
- `reason` 必填，并作为本次黑名单编辑的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 公示状态、公示级别、公示原因由 `POST /api/admin/blacklist/{id}/set-public` 单独维护
- 不允许通过本接口修改 `blacklist_type` 与 `target_id`
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=blacklist`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含封禁原因摘要、证据摘要、有效期与关联主体状态摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、黑名单状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "黑名单信息已更新",
  "data": {
    "id": 1,
    "reason": "多次虚假发布并骚扰求职者",
    "expire_at": "2024-12-31 23:59:59"
  }
}

【6.11.20 公告列表】
说明：
- 公告管理仅维护 announcements 公共公告资源
- 定向发送、已读统计、send_count/read_count 等广播能力请使用 §6.11.28 系统广播接口（POST /api/admin/broadcast/send 等）

【6.11.20.1 公告列表】
GET /api/admin/announcement/list

请求参数：
- is_enabled: 启用状态（0禁用/1启用，可选）
- announcement_type: 类型（system系统/activity活动/notice通知，可选）
- keyword: 关键词搜索（标题，可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "title": "系统维护通知",
        "announcement_type": "system",
        "announcement_type_name": "系统",
        "is_top": 1,
        "is_enabled": 1,
        "start_time": "2024-03-06 00:00:00",
        "end_time": "2024-03-10 23:59:59",
        "admin_id": 1,
        "admin_name": "管理员A",
        "created_at": "2024-03-06 09:00:00",
        "updated_at": "2024-03-06 09:00:00"
      }
    ],
    "total": 12,
    "page": 1,
    "limit": 20
  }
}

【6.11.20.2 公告详情】
GET /api/admin/announcement/detail

请求参数：
- id: 公告ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "title": "系统维护通知",
    "content": "系统将于今晚22:00-24:00进行维护...",
    "announcement_type": "system",
    "is_top": 1,
    "is_enabled": 1,
    "start_time": "2024-03-06 00:00:00",
    "end_time": "2024-03-10 23:59:59",
    "admin_id": 1,
    "admin_name": "管理员A",
    "created_at": "2024-03-06 09:00:00",
    "updated_at": "2024-03-06 09:00:00"
  }
}

【6.11.20.3 新增公告】
POST /api/admin/announcement/add

请求参数：
{
  "title": "系统维护通知",
  "announcement_type": "system",
  "content": "系统将于今晚22:00-24:00进行维护...",
  "is_top": 1,
  "is_enabled": 1,
  "start_time": "2024-03-06 00:00:00",
  "end_time": "2024-03-10 23:59:59"
}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "title": "系统维护通知",
    "is_enabled": 1
  }
}

【6.11.20.4 编辑公告】
PUT /api/admin/announcement/update

请求参数：
{
  "id": 1,
  "title": "系统维护通知（更新）",
  "announcement_type": "system",
  "content": "系统将于今晚22:00-24:00进行维护，请提前保存数据...",
  "is_top": 1,
  "is_enabled": 1,
  "start_time": "2024-03-06 00:00:00",
  "end_time": "2024-03-11 23:59:59"
}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "title": "系统维护通知（更新）"
  }
}

【6.11.20.5 删除公告】
DELETE /api/admin/announcement/delete

请求参数：
- id: 公告ID（必填）
- reason: 删除原因（必填）

说明：
- 属于高危后台写操作（删除），`risk_level` 至少为 `high`
- `reason` 必填，并作为公告删除的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 删除前必须明确公告标题、启用状态、是否置顶与影响范围，避免误删正在生效或对外展示中的公告
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=announcement`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含公告标题、类型、启用状态、置顶状态摘要，`after_snapshot` 记录删除结果摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、公告状态不允许删除或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "success"
}

【6.11.20.6 启用/停用公告】
PUT /api/admin/announcement/toggle

请求参数：
{
  "id": 1,
  "is_enabled": 0,
  "reason": "活动已结束，停用展示"
}

说明：
- 属于高危后台写操作（公告状态切换），`risk_level` 至少为 `high`
- `reason` 必填，并作为公告启停的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 启用/停用前必须明确公告标题、类型、是否置顶与当前展示状态，避免错误下线或误上线公告
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=announcement`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含公告标题、类型、置顶状态与启停前后状态摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、公告状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "is_enabled": 0
  }
}

【6.11.21 违禁词列表】
GET /api/admin/banned-word/list

请求参数：
- category: 分类（sensitive敏感词/porn色情/violence暴力/politics政治，可选）
- match_type: 匹配方式（exact精确/fuzzy模糊，可选）
- is_enabled: 是否启用（0否/1是，可选）
- keyword: 关键词搜索（违禁词，可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "word": "赌博",
        "category": "sensitive",
        "category_name": "敏感词",
        "match_type": "exact",
        "match_type_name": "精确匹配",
        "is_enabled": 1,
        "created_by": 1,
        "created_by_name": "管理员A",
        "created_at": "2024-03-06 10:00:00"
      },
      {
        "id": 2,
        "word": "*博",
        "category": "sensitive",
        "category_name": "敏感词",
        "match_type": "fuzzy",
        "match_type_name": "模糊匹配",
        "is_enabled": 1,
        "created_by": 1,
        "created_by_name": "管理员A",
        "created_at": "2024-03-06 09:00:00"
      }
    ],
    "total": 568,
    "page": 1,
    "limit": 20
  }
}

说明：
- `word` 字段接口返回原文，供管理员编辑时使用；前端列表展示时须将 word 脱敏处理（显示为 ****），编辑弹窗中还原显示原文
- 此接口仅限管理员后台使用，需权限校验

【6.11.21.1 新增违禁词】
POST /api/admin/banned-word/add

请求参数：
{
  "word": "赌博",
  "category": "sensitive",
  "match_type": "exact"
}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "word": "赌博",
    "is_enabled": 1
  }
}

【6.11.21.2 编辑违禁词】
PUT /api/admin/banned-word/update

请求参数：
{
  "id": 1,
  "word": "赌博网站",
  "category": "sensitive",
  "match_type": "fuzzy"
}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "word": "赌博网站"
  }
}

【6.11.21.3 删除违禁词】
DELETE /api/admin/banned-word/delete

请求参数：
- id: 违禁词ID（必填）
- reason: 删除原因（必填）

说明：
- 属于高危后台写操作（内容安全规则删除），`risk_level` 至少为 `high`
- `reason` 必填，并作为违禁词删除的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 删除前必须明确违禁词内容、分类、匹配方式与启用状态，避免误删正在生效的内容安全规则
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=banned_word`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含违禁词内容、分类、匹配方式、启用状态摘要，`after_snapshot` 记录删除结果摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、规则状态不允许删除或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "success"
}

【6.11.21.4 切换违禁词状态】
PUT /api/admin/banned-word/toggle

请求参数：
{
  "id": 1,
  "is_enabled": 0,
  "reason": "误命中过多，先停用复核"
}

说明：
- 属于高危后台写操作（内容安全规则启停），`risk_level` 至少为 `high`
- `reason` 必填，并作为违禁词启停的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 启用/停用前必须明确违禁词内容、分类、匹配方式与当前状态，避免错误放开或误拦截内容
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=banned_word`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含违禁词内容、分类、匹配方式与启停前后状态摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、规则状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "is_enabled": 0
  }
}

【6.11.22 子管理员列表】
GET /api/admin/sub-admin/list

请求参数：
- status: 状态（enabled启用/disabled禁用，可选）
- keyword: 关键词搜索（用户名/姓名/手机号，可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 2,
        "username": "admin_audit",
        "name": "审核专员",
        "phone": "13800138001",
        "email": "audit@example.com",
        "role_name": "审核专员",
        "status": "enabled",
        "last_login_time": "2024-03-06 09:00:00",
        "last_login_ip": "192.168.1.100",
        "created_by": 1,
        "created_by_name": "超级管理员",
        "created_at": "2024-01-15 10:00:00"
      }
    ],
    "total": 8,
    "page": 1,
    "limit": 20
  }
}

【6.11.22.1 子管理员详情】
GET /api/admin/sub-admin/detail

请求参数：
- id: 管理员ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 2,
    "username": "admin_audit",
    "name": "审核专员",
    "phone": "13800138001",
    "email": "audit@example.com",
    "role_name": "审核专员",
    "status": "enabled",
    "admin_user_permissions": [
      {
        "permission_code": "user:view",
        "name": "用户查看"
      },
      {
        "permission_code": "job:audit",
        "name": "职位审核"
      },
      {
        "permission_code": "company:audit",
        "name": "企业认证审核"
      },
      {
        "permission_code": "help:article:manage",
        "name": "帮助文章管理"
      },
      {
        "permission_code": "help:stats:view",
        "name": "帮助搜索统计查看"
      }
    ],
    "last_login_time": "2024-03-06 09:00:00",
    "last_login_ip": "192.168.1.100",
    "created_at": "2024-01-15 10:00:00"
  }
}

说明：
- `role_name` 为展示用角色标签或权限模板名称（如"审核专员""客服""运营"），不等同于 admin_users.role 的枚举值
- `admin_user_permissions` 字段为聚合结果，来自 admin_user_permissions 表；超级管理员权限不依赖该字段判断

【6.11.22.2 新增子管理员】
POST /api/admin/sub-admin/add

请求参数：
{
  "username": "admin_audit",
  "password": "Initial@123",
  "name": "审核专员",
  "phone": "13800138001",
  "email": "audit@example.com",
  "permissions": ["user:view", "job:audit", "company:audit"]
}

说明：
- `permissions` 请求参数为 permission_code 数组，后端将逐条写入 admin_user_permissions 表
- 帮助中心相关可选值包括：help:category:manage、help:article:manage、help:relation:manage、help:synonym:manage、help:stats:view

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 2,
    "username": "admin_audit",
    "name": "审核专员",
    "status": "enabled",
    "admin_user_permissions": [
      { "permission_code": "user:view", "name": "用户查看" },
      { "permission_code": "job:audit", "name": "职位审核" },
      { "permission_code": "company:audit", "name": "企业认证审核" }
    ]
  }
}

【6.11.22.3 编辑子管理员权限】
PUT /api/admin/sub-admin/update-permissions

请求参数：
{
  "id": 2,
  "permissions": ["user:view", "job:audit", "company:audit", "report:handle", "help:article:manage", "help:stats:view"],
  "reason": "调整审核与帮助中心管理职责"
}

说明：
- 属于高危后台写操作（权限变更），`risk_level` 至少为 `critical`
- `reason` 必填，并作为本次授权调整的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 仅允许具备更高权限范围的管理员执行，禁止越权授予自身不具备的权限
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=admin`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含授权前后权限差异摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 2,
    "permissions": ["user:view", "job:audit", "company:audit", "report:handle", "help:article:manage", "help:stats:view"]
  }
}

【6.11.22.4 删除子管理员】
DELETE /api/admin/sub-admin/delete

请求参数：
{
  "id": 2,
  "reason": "岗位调整，回收后台权限"
}

说明：
- 属于高危后台写操作（删除管理员账号），`risk_level` 至少为 `critical`
- `reason` 必填，并作为删除子管理员的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 删除前需校验目标账号不是当前登录管理员、不是系统保留超级管理员，且完成必要的职责交接检查
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=admin`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含账号状态、角色与权限摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "success"
}

【6.11.22.5 切换子管理员状态】
PUT /api/admin/sub-admin/toggle-enabled

请求参数：
{
  "id": 2,
  "status": "disabled"
}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 2,
    "status": "disabled"
  }
}

【6.11.23 系统配置】

【6.11.23.1 获取系统配置列表】
GET /api/admin/system-config/list

请求参数：
- category: 配置分类（basic/function/business/security，可选）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "basic": [
      {
        "config_key": "site_name",
        "config_value": "自动化装备招聘系统",
        "config_name": "平台名称",
        "config_desc": "显示在页面顶部的平台名称",
        "value_type": "string",
        "is_editable": 1
      },
      {
        "config_key": "site_logo",
        "config_value": "f_20240306_logo001",
        "config_name": "平台Logo",
        "config_desc": "平台Logo文件标识（公开接口输出为 site_logo_file）",
        "value_type": "image",
        "is_editable": 1
      }
      {
        "config_key": "maintenance_mode",
        "config_value": "0",
        "config_name": "维护模式",
        "config_desc": "是否开启平台维护模式",
        "value_type": "boolean",
        "is_editable": 1
      }
    ],
    "function": [
      {
        "config_key": "chat_enabled",
        "config_value": "1",
        "config_name": "聊天功能开关",
        "config_desc": "是否启用私聊与消息中心",
        "value_type": "boolean",
        "is_editable": 1
      },
      {
        "config_key": "public_chat_enabled",
        "config_value": "1",
        "config_name": "公共聊天室开关",
        "config_desc": "是否启用公共聊天室",
        "value_type": "boolean",
        "is_editable": 1
      },
      {
        "config_key": "captcha_enabled",
        "config_value": "1",
        "config_name": "图形验证码开关",
        "config_desc": "是否启用图形验证码",
        "value_type": "boolean",
        "is_editable": 1
      }
    ],
    "business": [
      {
        "config_key": "job_top_duration_days",
        "config_value": "7",
        "config_name": "置顶时长",
        "config_desc": "职位置顶默认时长（天）",
        "value_type": "number",
        "is_editable": 1
      },
      {
        "config_key": "refresh_limit_per_day",
        "config_value": "5",
        "config_name": "职位刷新每日限额",
        "config_desc": "单个雇主每天可刷新职位的最大次数",
        "value_type": "number",
        "is_editable": 1
      },
      {
        "config_key": "reapply_limit_hours",
        "config_value": "72",
        "config_name": "重新报名限制",
        "config_desc": "被拒绝后再次报名的限制时长（小时）",
        "value_type": "number",
        "is_editable": 1
      },
      {
        "config_key": "first_jobs_audit_limit",
        "config_value": "10",
        "config_name": "首单审核数量",
        "config_desc": "新认证企业需审核的前置职位数量",
        "value_type": "number",
        "is_editable": 1
      }
    ],
    "security": [
      {
        "config_key": "login_fail_max",
        "config_value": "5",
        "config_name": "登录失败限制",
        "config_desc": "连续登录失败多少次后锁定账号",
        "value_type": "number",
        "is_editable": 1
      },
      {
        "config_key": "login_lock_minutes",
        "config_value": "30",
        "config_name": "登录锁定时长",
        "config_desc": "账号锁定持续分钟数",
        "value_type": "number",
        "is_editable": 1
      },
      {
        "config_key": "token_expire_days",
        "config_value": "7",
        "config_name": "Token有效期",
        "config_desc": "新签发登录Token的有效期（天）",
        "value_type": "number",
        "is_editable": 1
      }
    ]
  }
}

说明：
- 管理端配置列表分组必须与系统配置页的 `basic / function / business / security` 四个Tab保持一致
- 页面展示名称可不同，但底层 `config_key` 与分组归属必须唯一

【6.11.23.2 更新系统配置】
PUT /api/admin/system-config/update

请求参数：
{
  "configs": [
    {"config_key": "chat_enabled", "config_value": "1"},           // boolean类型
    {"config_key": "reapply_limit_hours", "config_value": "72"},  // number类型
    {"config_key": "site_name", "config_value": "自动化装备招聘系统"}, // string类型
    {"config_key": "site_logo", "config_value": "f_20240306_logo001"}, // image类型（file_id）
    {"config_key": "search_keywords", "config_value": "[\"电工\",\"焊工\"]"} // json类型
  ],
  "change_reason": "根据业务需求调整配置"
}

configs数组元素字段说明：
- config_key：配置键（必填，参见【6.11.23.1 系统配置列表】获取可用键）
- config_value：配置值（必填，类型由列表接口返回的 value_type 决定）
  * string：普通文本，如 "自动化装备招聘系统"
  * number：数字字符串，如 "72"
  * boolean：0/1，如 "1"
  * image：文件ID（file_id），如 "f_20240306_logo001"
  * json：JSON字符串，如 "[\"item1\",\"item2\"]"
- value_type：值类型（可选，若传则校验格式一致性）
- options：选项约束（可选，用于前端下拉选择）
- is_editable：是否可编辑（列表接口返回，更新时可不传）

说明：
- 属于高危后台写操作（系统配置修改），`risk_level` 至少为 `critical`
- `change_reason` 必填，并作为配置变更的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 必须同时写入 `config_change_logs` 与 `operation_logs`，确保配置变更日志与后台操作审计口径一致
- `operation_logs` 中需记录 `operator_id`、`target_type=config`、`request_id`、`result`、`risk_level`、`reason=change_reason`、`before_snapshot`、`after_snapshot`；快照至少包含变更项列表、变更前后值摘要与生效方式
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "配置已保存",
  "data": {
    "updated_count": 2,
    "effect_type": "immediate"
  }
}

【6.11.24 操作日志列表】
GET /api/admin/operation-log/list

请求参数：
- operator_type: 操作者类型（admin管理员/user用户，可选）
- operator_id: 操作者ID（可选）
- operation_type: 操作类型（create新增/update更新/delete删除/audit审核，可选）
- target_type: 操作对象类型（user/job/company/report/announcement/broadcast/config，可选）
- start_date: 开始日期（可选）
- end_date: 结束日期（可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "operator_type": "admin",
        "operator_type_name": "管理员",
        "operator_id": 1,
        "operator_name": "管理员A",
        "operation_type": "audit",
        "operation_type_name": "审核",
        "operation_desc": "审核通过企业认证",
        "target_type": "company",
        "target_type_name": "企业认证",
        "target_id": 1,
        "target_name": "XX建设集团",
        "params": "{\"id\":1,\"audit_status\":1}",
        "ip": "192.168.1.100",
        "result": "success",
        "created_at": "2024-03-06 10:00:00"
      },
      {
        "id": 2,
        "operator_type": "user",
        "operator_type_name": "用户",
        "operator_id": 10086,
        "operator_name": "张三",
        "operation_type": "create",
        "operation_type_name": "新增",
        "operation_desc": "发布职位",
        "target_type": "job",
        "target_type_name": "职位",
        "target_id": 123,
        "target_name": "电工招聘",
        "params": "{\"title\":\"电工招聘\",\"category_ids\":[1,2]}",
        "ip": "192.168.1.101",
        "result": "success",
        "created_at": "2024-03-06 09:30:00"
      }
    ],
    "total": 56800,
    "page": 1,
    "limit": 20
  }
}

【6.11.25 置顶申请列表】
GET /api/admin/top-application/list

请求参数：
- status: 状态（pending待审核/approved已通过/rejected已拒绝/expired已过期/cancelled已取消，默认pending）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "job_id": 123,
        "job_title": "电工招聘",
        "user_id": 10087,
        "user_name": "张经理",
        "company_name": "XX建设集团",
        "apply_days": 7,
        "apply_time": "2024-03-06 10:00:00",
        "status": "pending"
      }
    ],
    "total": 5,
    "page": 1,
    "limit": 20
  }
}

【6.11.26 置顶申请审核】
PUT /api/admin/top-application/audit

请求参数：
{
  "id": 1,
  "action": "approve",
  "reason": "符合置顶条件"
}

成功响应：
{
  "code": 200,
  "msg": "审核完成",
  "data": null
}

说明：
- 属于高危后台写操作（审核），`risk_level` 建议按结果区分：`approve` 可记为 `medium`，`reject` 至少记为 `high`
- `reason` 必填，并作为置顶申请审核决定的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 仅允许处理 `status = pending` 的置顶申请，避免重复审核
- 审核通过后，系统自动更新 `job_top_applications` 状态，并同步更新 `jobs.is_top=1`、`top_expire_time=当前时间+申请天数`
- 审核拒绝后，需更新 `job_top_applications` 状态并通知申请人
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=top_application`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含申请状态、职位ID、申请天数、审核结果与职位置顶状态摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、申请状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

【6.11.13 设置置顶】
POST /api/admin/job/toggle-top

请求头：Authorization: Bearer {admin_token}

请求参数：
{
  "job_id": 1,
  "action": "set",        // set设置置顶/cancel取消置顶
  "top_days": 7,          // 置顶天数（action=set时必填）
  "reason": "优质职位推荐"  // 操作原因
}

成功响应：
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "is_top": 1,
    "top_expire_time": "2024-03-13 10:00:00"
  }
}

说明：
- 属于高危后台写操作（直接设置/取消置顶），`risk_level` 至少为 `high`
- `reason` 必填，并作为直接置顶或取消置顶的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 超级管理员可直接操作，无需申请审核流程；非超级管理员如需使用该能力，必须受独立高权限控制
- `action=set` 时 `top_days` 必填且必须大于 0；`action=cancel` 时忽略 `top_days`
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=job_id`、`target_type=job`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含职位当前置顶状态、置顶剩余时长摘要与本次操作结果
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、参数不合法或二次确认未通过，必须返回 403 或 422，并写入失败/拒绝审计日志
- 管理员设置置顶前，系统会自动检查该职位是否有待审核的置顶申请（job_top_applications 中 status=pending 的记录）：
  ├─ 若有待审核的申请 → 返回提示，引导管理员先去审核用户申请
  │   错误响应示例：
  │   {
  │     "code": 4002,
  │     "msg": "该职位有待审核的置顶申请，请先去审核",
  │     "data": {
  │       "pending_application_id": 1,
  │       "applicant_name": "张经理",
  │       "company_name": "XX建设集团",
  │       "apply_days": 7,
  │       "created_at": "2024-03-06 10:00:00"
  │     }
  │   }
  └─ 若无待审核申请 → 管理员可正常设置/取消置顶

【6.11.27 数据导出】

【6.11.27.1 创建导出任务】
POST /api/admin/export/create

请求头：Authorization: Bearer {admin_token}

请求参数：
{
  "export_type": "users",  // 导出类型：users用户/jobs职位/applications报名/companies企业
  "filter": {
    "start_date": "2024-01-01",
    "end_date": "2024-03-06",
    "status": "active"
  },
  "fields": ["id", "nickname", "phone", "role", "created_at"],  // 指定导出字段，null表示全部
  "format": "xlsx",  // 格式：csv/xlsx，默认xlsx
  "reason": "运营复盘，导出活跃用户样本"
}

说明：
- `fields`参数用于指定导出字段，对应数据库`export_tasks.selected_fields`字段，格式为JSON数组；传`null`或空数组表示导出全部可用字段
- 属于高危后台写操作（数据导出），`risk_level` 至少为 `high`；涉及手机号、邮箱、身份证等高敏字段时至少为 `critical`
- `reason` 必填，并作为导出审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 导出默认遵循最小字段原则；如 `fields` 为空或申请导出高敏字段，必须额外校验导出权限与数据范围
- 必须写入 `export_tasks` 与 `operation_logs`；`operation_logs` 中需记录 `operator_id`、`target_type=export`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`，并附带导出类型、筛选条件、字段范围、数据范围、任务状态、文件过期时间与下载时间等摘要
- 小数据量同步下载、大数据量异步导出都必须留痕；同一导出请求的创建、生成、下载应通过 `request_id` 串联
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、敏感字段限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应（小数据量直接下载）：
文件下载（Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet）

成功响应（大数据量异步导出）：
{
  "code": 200,
  "msg": "导出任务已创建",
  "data": {
    "task_id": "EXP202403060001",
    "request_id": "req_export_20260403_000001",
    "status": "pending"
  }
}

补充说明：
- 异步导出任务不承诺精确完成时间，前端应基于 `task_id` 轮询状态接口或结合站内通知获取完成结果，不得依赖固定秒数倒计时作为完成依据
- `progress` 仅用于表达任务处理进度，最终是否可下载以 `status=completed` 且返回有效 `file_url` 为准
- 如导出任务失败，应通过状态接口返回失败状态，并由前端提示用户重试或调整筛选条件/导出字段后重新发起导出任务

【6.11.27.2 查询导出任务状态】
GET /api/admin/export/status

请求参数：
- task_id: 任务ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "task_id": "EXP202403060001",
    "request_id": "req_export_20260403_000001",
    "status": "completed",  // pending待处理/processing处理中/completed已完成/failed失败
    "progress": 100,  // 进度百分比
    "file_url": "https://.../exports/users_20240306.xlsx",
    "file_size": 1024567,
    "record_count": 5680,
    "created_at": "2024-03-06 10:00:00",
    "completed_at": "2024-03-06 10:01:30",
    "downloaded_at": "2024-03-06 10:05:00",
    "expire_at": "2024-03-13 10:00:00"
  }
}

【6.11.27.3 导出记录列表】
GET /api/admin/export/list

请求参数：
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "task_id": "EXP202403060001",
        "request_id": "req_export_20260403_000001",
        "export_type": "users",
        "export_type_name": "用户数据",
        "status": "completed",
        "file_url": "https://.../exports/users_20240306.xlsx",
        "file_size": 1024567,
        "record_count": 5680,
        "created_at": "2024-03-06 10:00:00",
        "completed_at": "2024-03-06 10:01:30",
        "expire_at": "2024-03-13 10:00:00"
      }
    ],
    "total": 15,
    "page": 1,
    "limit": 20
  }
}

说明：
- 小数据量（<1000条）：同步导出，直接下载
- 大数据量（>=1000条）：异步导出，创建任务后通过task_id查询状态
- 导出文件有效期7天
- 敏感字段（手机号、身份证号）自动脱敏处理

【6.11.28 系统广播】
说明：用户侧接收、未读统计与已读处理统一复用通知中心接口（/api/notifications*）与 notifications 表；系统广播接口仅负责管理端创建、发布、查询广播资源，不承载单条审核/报名/认证等业务通知。广播统计字段统一使用 `send_count` 与 `read_count`，不再使用 `sent_count` 命名。广播封面图对外统一返回 `cover_image_file`（File Resource），请求写入统一提交 `cover_image_file_id`；`link_url` 仍保持 string。

【6.11.28.1 广播列表】
GET /api/admin/broadcast/list

请求参数：
- status: 状态（draft草稿/published已发布，可选）
- type: 类型（notice通知/announcement公告/warning警告，可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "title": "系统维护通知",
        "type": "notice",
        "type_name": "通知",
        "content": "系统将于今晚22:00-24:00进行维护...",
        "cover_image_file": {
          "file_id": "f_20240306_cover001",
          "disk": "public",
          "path": "broadcasts/2024/03/06/cover001.jpg",
          "access": "public",
          "url": "https://.../cover.jpg",
          "temporary_url": null,
          "mime_type": "image/jpeg",
          "size": 153600,
          "original_name": "cover.jpg"
        },
        "target_type": "all",
        "target_type_name": "全部用户",
        "status": "published",
        "send_count": 5680,
        "read_count": 3200,
        "send_time": "2024-03-06 10:00:00",
        "created_by": 1,
        "created_by_name": "管理员A",
        "created_at": "2024-03-06 09:00:00"
      }
    ],
    "total": 56,
    "page": 1,
    "limit": 20
  }
}

【6.11.28.2 广播详情】
GET /api/admin/broadcast/detail

请求参数：
- id: 广播ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "title": "系统维护通知",
    "type": "notice",
    "content": "系统将于今晚22:00-24:00进行维护...",
    "cover_image_file": {
      "file_id": "f_20240306_cover001",
      "disk": "public",
      "path": "broadcasts/2024/03/06/cover001.jpg",
      "access": "public",
      "url": "https://.../cover.jpg",
      "temporary_url": null,
      "mime_type": "image/jpeg",
      "size": 153600,
      "original_name": "cover.jpg"
    },
    "link_url": "https://.../detail",
    "target_type": "all",
    "target_value": null,
    "status": "published",
    "send_count": 5680,
    "read_count": 3200,
    "send_time": "2024-03-06 10:00:00",
    "created_by": 1,
    "created_by_name": "管理员A",
    "created_at": "2024-03-06 09:00:00"
  }
}

【6.11.28.3 发送系统广播】
POST /api/admin/broadcast/send

请求头：Authorization: Bearer {admin_token}

请求参数：
{
  "title": "系统维护通知",
  "content": "系统将于今晚12点进行维护...",
  "type": "notice",
  "cover_image_file_id": "f_20240306_cover001",
  "link_url": "https://.../detail",
  "target_type": "all",  // all全部/seeker求职者/employer雇主/region区域/users指定用户
  "target_value": null,  // target_type=region时填写区域代码，target_type=users时填写用户ID集合
  "send_time": null,     // null立即发送，否则为定时发送时间
  "send_push": true      // 是否发送推送通知
}

成功响应：
{
  "code": 200,
  "msg": "广播已发送",
  "data": {
    "broadcast_id": 1,
    "target_count": 12580,
    "send_time": "2024-03-06 10:00:00"
  }
}

【6.11.28.4 删除广播】
DELETE /api/admin/broadcast/delete

请求参数：
- id: 广播ID（必填）
- reason: 删除原因（必填）

说明：
- 属于高危后台写操作（删除），`risk_level` 至少为 `high`
- `reason` 必填，并作为广播删除的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 删除前必须明确广播标题、发送状态、目标范围与已发送规模，避免误删已执行或待发送广播
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=broadcast`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含广播标题、发送状态、目标范围、发送时间与目标人数摘要，`after_snapshot` 记录删除结果摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、广播状态不允许删除或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "success"
}

【6.11.29 帮助中心管理接口】
说明：帮助中心管理统一由管理员端维护分类、文章、相关文章、同义词与搜索统计；后台入口统一为"帮助中心管理"。
权限要求：分类管理使用 help:category:manage，文章管理使用 help:article:manage，相关文章配置使用 help:relation:manage，同义词管理使用 help:synonym:manage，搜索统计查看使用 help:stats:view。

【6.11.29.1 分类列表】
GET /api/admin/help/categories

请求参数：
- keyword: 关键词（可选）
- status: 状态（可选）
- page, limit

【6.11.29.2 新增分类】
POST /api/admin/help/categories

请求参数：
{
  "name": "新手指南",
  "icon": "guide",
  "sort_order": 1,
  "status": "enabled"
}

说明：
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_category`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含分类名称、排序与状态摘要

【6.11.29.3 编辑分类】
PUT /api/admin/help/categories/{id}

请求参数：
- id: 分类ID（必填）
- name: 分类名称（可选）
- icon: 图标（可选）
- sort_order: 排序（可选）
- status: 状态（可选）

说明：
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_category`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含分类名称、排序与状态摘要，`after_snapshot` 记录变更后状态摘要

【6.11.29.4 删除分类】
DELETE /api/admin/help/categories/{id}

请求参数：
- id: 分类ID（必填）

说明：
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_category`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含分类名称、排序、状态与关联文章数量摘要，`after_snapshot` 记录删除结果摘要
- 删除前必须检查该分类下是否有文章，如有文章应先转移或提示管理员

【6.11.29.5 文章列表】
GET /api/admin/help/articles

请求参数：
- category_id: 分类ID（可选）
- keyword: 关键词（可选）
- status: 状态（可选）
- page, limit

【6.11.29.6 文章详情】
GET /api/admin/help/articles/{id}

【6.11.29.7 新增文章】
POST /api/admin/help/articles

请求参数：
{
  "category_id": 1,
  "title": "如何发布职位",
  "content": "...",
  "tags": ["发布职位", "招聘"],
  "keywords": ["发布职位", "招工"],
  "is_hot": 1,
  "status": "enabled"
}

【6.11.29.8 编辑文章】
PUT /api/admin/help/articles/{id}

【6.11.29.9 删除文章】
DELETE /api/admin/help/articles/{id}

请求参数：
{
  "reason": "内容已过期，删除避免误导"
}

说明：
- 属于高危后台写操作（删除），`risk_level` 至少为 `high`
- `reason` 必填，并作为帮助文章删除的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 删除前必须明确文章标题、分类、发布状态与关联推荐配置，避免误删仍在前台展示或被关联引用的内容
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_article`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含文章标题、分类、状态与关联配置摘要，`after_snapshot` 记录删除结果摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、文章状态不允许删除或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

【6.11.29.10 切换文章状态】
PUT /api/admin/help/articles/{id}/toggle

请求参数：
{
  "status": "disabled",
  "reason": "内容待修订，先下线"
}

说明：
- 属于高危后台写操作（文章状态切换），`risk_level` 至少为 `high`
- `reason` 必填，并作为帮助文章启停的审计原因；前端必须展示二次确认信息，后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 启用/停用前必须明确文章标题、分类、当前状态与前台展示范围，避免错误下线或误上线内容
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_article`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含文章标题、分类与启停前后状态摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制、文章状态不允许变更或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志

【6.11.29.11 获取相关文章配置】
GET /api/admin/help/articles/{id}/relations

【6.11.29.12 保存相关文章配置】
PUT /api/admin/help/articles/{id}/relations

请求参数：
{
  "related_article_ids": [2, 3, 5]
}

【6.11.29.13 同义词列表】
GET /api/admin/help/synonyms

请求参数：
- keyword: 关键词（可选）
- status: 状态（可选）
- page, limit

【6.11.29.14 新增同义词】
POST /api/admin/help/synonyms

请求参数：
{
  "source_word": "招工",
  "target_word": "招聘",
  "status": "enabled",
  "sort_order": 1
}

说明：
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_synonym`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含源词、目标词与状态摘要

【6.11.29.15 编辑同义词】
PUT /api/admin/help/synonyms/{id}

请求参数：
- id: 同义词ID（必填）
- source_word: 源词（可选）
- target_word: 目标词（可选）
- status: 状态（可选）
- sort_order: 排序（可选）

说明：
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_synonym`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含源词、目标词与状态摘要，`after_snapshot` 记录变更后状态摘要

【6.11.29.16 删除同义词】
DELETE /api/admin/help/synonyms/{id}

请求参数：
- id: 同义词ID（必填）

说明：
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=id`、`target_type=help_synonym`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` 至少包含源词、目标词与状态摘要，`after_snapshot` 记录删除结果摘要

【6.11.29.17 搜索统计】
GET /api/admin/help/search-stats

请求参数：
- start_date: 开始日期（可选）
- end_date: 结束日期（可选）
- type: 统计类型（hot/no_result/trend，可选）

================================================================================
6.12 人才库相关接口
================================================================================

【6.12.1 人才库列表】
GET /api/employer/talent-pool

请求头：Authorization: Bearer {token}（雇主身份）

请求参数：
- keyword: 搜索关键词（姓名/手机号后4位，可选）
- category_id: 工种筛选（可选）
- level: 技能级别筛选（senior/medium/junior，可选）
- start_date: 添加时间开始（可选）
- end_date: 添加时间结束（可选）
- sort: 排序方式（latest最近添加/oldest最早添加，可选）
- page: 页码
- limit: 每页数量

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "seeker_id": 10086,
        "seeker_name": "张三",
        "avatar_file": {
          "file_id": "f_20240306_avatar002",
          "disk": "public",
          "path": "avatars/2024/03/06/avatar002.jpg",
          "access": "public",
          "url": "https://...",
          "temporary_url": null,
          "mime_type": "image/jpeg",
          "size": 99840,
          "original_name": "avatar.jpg"
        },
        "category_id": 1,
        "category_name": "电工",
        "level": "senior",
        "level_name": "大工",
        "work_years": 5,
        "age": 30,
        "education": "高中",
        "added_at": "2024-03-06 10:00:00",
        "source_application_id": 12345,
        "source_job_id": 1,
        "source_job_title": "电工招聘"
      }
    ],
    "total": 15,
    "page": 1,
    "limit": 20
  }
}

说明：
- 人才库列表与搜索统一复用 `GET /api/employer/talent-pool`，不再单独保留 `/api/employer/talent-pool/search`
- 同一求职者可能以不同工种存在于人才库中，分别展示为独立记录
- `category_name` 来自 `talent_pool.category_name` 快照字段，不再关联查询【修订】
- `level` 与 `level_name` 来自加入人才库时固化的 `talent_pool.level` 快照
- 列表默认按 `created_at DESC` 排序

【6.12.2 加入人才库】
POST /api/employer/talent-pool/add

请求头：Authorization: Bearer {token}

请求参数：
{
  "source_application_id": 12345,
  "remark": "技术好，态度佳"
}

Laravel 实现说明：
- 服务端根据 `source_application_id` 反查报名记录，自动写入 `seeker_id`、`source_job_id`、`category_id`、`category_name`、`level`【修订】
- 仅允许将当前雇主自己发布职位下的报名记录加入人才库
- 同一雇主对同一求职者的同一工种仅保留一条人才库记录【修订】
- 若同一求职者以不同工种报名，可分别加入人才库（不影响已有工种记录）

成功响应：
{
  "code": 200,
  "msg": "已加入人才库",
  "data": {
    "id": 1,
    "added_at": "2024-03-06 10:00:00"
  }
}

错误响应：
{
  "code": 4002,
  "msg": "该求职者的[工种名称]工种已加入人才库，无法重复添加",
  "data": {
    "existing_id": 1,
    "category_name": "电工"
  }
}

【6.12.3 移除人才库】
DELETE /api/employer/talent-pool/remove

请求头：Authorization: Bearer {token}

请求参数：
{
  "id": 1  // 人才库记录ID
}

成功响应：
{
  "code": 200,
  "msg": "已移除",
  "data": null
}

【6.12.4 查看人才库简历】
GET /api/employer/talent-pool/resume

请求头：Authorization: Bearer {token}

请求参数：
- seeker_id: 求职者ID

成功响应：（同简历详情，包含完整工作经历和技能）

================================================================================
6.13 其他补充接口
================================================================================

【6.13.1 删除单条通知】
DELETE /api/notifications/{id}

请求头：Authorization: Bearer {token}

请求参数：
- id: 通知ID（必填）

成功响应：
{
  "code": 200,
  "msg": "删除成功",
  "data": null
}

【6.13.2 批量删除通知/清空全部】
DELETE /api/notifications

请求头：Authorization: Bearer {token}

请求参数：
{
  "ids": [1, 2, 3]
}
或
{
  "all": true
}

成功响应：
{
  "code": 200,
  "msg": "删除成功",
  "data": {
    "deleted_count": 3
  }
}

说明：
- 传 `ids` 表示批量删除指定通知
- 传 `all=true` 表示清空当前用户全部通知
⚠️ 参数互斥：ids和all不能同时传递，只能选择其中一种方式
⚠️ 边界情况：
   ├─ ids为空数组时：返回成功但不删除任何通知
   └─ ids中包含不存在的通知ID时：跳过不存在项，只删除存在的通知

【6.13.3 清空聊天记录】
DELETE /api/message/conversation/clear

请求头：Authorization: Bearer {token}

请求参数：
{
  "conversation_key": "10086_10087"  // 会话标识
}

成功响应：
{
  "code": 200,
  "msg": "聊天记录已清空",
  "data": null
}

说明：
- 仅标记当前用户删除，对方记录保留
- 清空后会话仍保留在列表中

【6.13.4 标记所有私聊已读】
PUT /api/message/read-all

请求头：Authorization: Bearer {token}

请求参数：无

成功响应：
{
  "code": 200,
  "msg": "已标记为已读",
  "data": {
    "marked_count": 15
  }
}

说明：
- 该接口仅用于一键标记当前用户全部私聊会话为已读
- 不再支持传 conversation_key 标记单个会话；单会话已读请使用 PUT /api/message/read
- 服务端需批量更新当前用户在 messages_private 中的未读消息状态，并同步清零 conversations 表中的对应未读计数

【6.13.5 获取草稿列表】
GET /api/job/draft-list

请求头：Authorization: Bearer {token}

请求参数：
- page: 页码
- limit: 每页数量

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "title": "电工招聘（草稿）",
        "last_edit_time": "2024-03-06 10:00:00",
        "completion_percent": 60  // 完成度百分比
      }
    ],
    "total": 3,
    "page": 1,
    "limit": 20
  }
}

【6.6.7.1 获取草稿详情】
GET /api/job/draft-detail

请求头：Authorization: Bearer {token}

请求参数：
- id: 草稿职位ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "status": "draft",
    "title": "电工招聘（草稿）",
    "work_location": "北京",
    "updated_at": "2024-03-06 10:00:00"
  }
}

【6.13.6 标记职位已招满】
PUT /api/job/mark-filled

请求头：Authorization: Bearer {token}

请求参数：
{
  "id": 1  // 职位ID
}

成功响应：
{
  "code": 200,
  "msg": "职位已标记为招满",
  "data": {
    "status": "closed",
    "close_reason": "filled"
  }
}

说明：
- 雇主可手动标记职位已招满，标记后不再接受报名
- 本接口必须写入 `status=closed` 且 `close_reason=filled`
- 后续若有求职者取消报名，系统自动判断是否满足恢复条件【明确触发方式】：
│   ├─ 触发方式：【系统自动恢复】（在取消报名同一事务中处理）
│   ├─ 恢复条件：
│   │   ├─ 职位当前 status = closed
│   │   ├─ close_reason = filled（已招满关闭）
│   │   └─ 取消后至少一个启用中的工种/级别未满员
│   ├─ 满足条件 → 自动恢复为 active，close_reason 置回 null
│   └─ 不满足条件（close_reason = manual / admin）→ 不恢复，保持关闭状态

【6.13.7 获取相关职位推荐】
GET /api/job/related

请求参数：
- job_id: 当前职位ID（必填）
- limit: 数量（默认5）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 2,
        "title": "焊工招聘",
        "company_name": "YY机械",
        "work_location": "上海浦东",
        "salary_range": "400-600"
      }
    ]
  }
}

说明：基于相同工种、相近地点推荐

【6.13.8 获取我的工种申请记录】
GET /api/category/application/my-list

请求头：Authorization: Bearer {token}

请求参数：
- status: 状态筛选（pending/approved/rejected）
- page: 页码
- limit: 每页数量

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "name": "数控车工",
        "reason": "市场需求大",
        "status": "pending",
        "created_at": "2024-03-06 10:00:00"
      }
    ],
    "total": 3,
    "page": 1,
    "limit": 20
  }
}

================================================================================
