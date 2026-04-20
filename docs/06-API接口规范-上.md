第六章：API接口规范（上）
================================================================================

索引
================================================================================
6.1 接口通用规范
6.2 错误代码规范
6.3 用户相关接口
6.4 企业认证相关接口【新增】
6.5 简历相关接口
6.6 职位相关接口
6.7 评论相关接口【新增】
6.8 报名相关接口
6.9 消息相关接口
6.10 其他功能接口
6.11 管理员后台接口【新增】

6.1 接口通用规范
--------------------------------------------------

【请求格式】
- 请求方法：GET, POST, PUT, DELETE
- Content-Type: application/json（文件上传接口除外，使用 multipart/form-data）
- 字符编码：UTF-8
- 请求头：
  Authorization: Bearer {token}
  User-Agent: {客户端信息}
  Accept: application/json
  X-Requested-With: XMLHttpRequest

【响应格式】
统一 JSON 格式：
{
  "code": 200,
  "msg": "success",
  "data": {}
}

【HTTP状态码与业务code约定】【新增】【P0】
- HTTP 状态码用于表达协议/传输语义；响应体中的业务 `code` 用于表达具体业务结果，两者需同时使用，不互相替代
- 成功响应：HTTP 200，业务 `code` = 200，`msg` = success
- 参数错误：HTTP 400，对应业务 `code` 使用 2000-2999 区间
- 未认证/登录失效：HTTP 401，对应业务 `code` 使用 3000/3001/3008 等认证错误码
- 已认证但无权限：HTTP 403，对应业务 `code` 使用 3003 或具体权限类错误码
- 资源不存在：HTTP 404，对应业务 `code` 优先使用 4001 等资源不存在类错误码
- 请求过于频繁：HTTP 429，对应业务 `code` 使用 6000-6999 区间
- 服务端异常：HTTP 500，对应业务 `code` 使用 1000-1999 区间
- 数据验证失败：HTTP 400，对应业务 `code` 可使用 5000-5999 区间；如使用 Laravel FormRequest，需将校验异常统一转换为本文档定义的 JSON 结构
- 默认要求：除成功场景外，禁止一律返回 HTTP 200 再仅依赖业务 `code` 表达失败
- 所有接口示例若仅展示成功响应，实际实现仍必须遵循上述 HTTP 状态码映射规则

【分页参数】
- page: 页码（默认1）
- limit: 每页数量（默认20）

【分页响应】
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [],
    "total": 100,
    "page": 1,
    "limit": 20,
    "total_pages": 5
  }
}

【Laravel 11 API 实施规范】
- 路由层：负责路由组织、鉴权中间件、频率限制中间件、路由模型绑定
- FormRequest：负责参数校验、字段白名单、条件校验、授权预检查
- Controller：负责调用 Service / Action，组织事务边界，返回 Resource / JsonResponse，不直接堆积复杂业务逻辑
- Service / Action：负责职位发布、企业认证审核、消息发送、报名审核、黑名单处理等核心业务状态流转
- Policy / Gate：负责资源级权限判断，如"是否可编辑职位""是否可审核企业""是否可查看敏感资质文件"
- Resource：统一响应字段结构、脱敏逻辑、枚举值转换、嵌套资源输出
- Job / Event / Listener / Notification：负责异步通知、日志记录、统计刷新、违规审核、导出生成等耗时任务
- Exception Handler：统一把验证异常、鉴权异常、业务异常转换为本文档定义的 code/msg/data 格式
- 数据事务：涉及多表写入的接口（如发布职位、审核报名、认证审核、消息写入+会话更新）必须使用数据库事务
- 幂等控制：报名、消息发送、职位刷新、置顶申请、批量审核等写接口必须落实幂等控制，结合业务唯一约束、幂等键、`request_id`、`client_msg_id` 或 Redis 短期去重避免重复提交
- 幂等结果约定：同一幂等键或同一业务唯一键命中重复请求时，必须返回首次处理结果或明确返回"记录已存在/请勿重复提交"的标准化业务响应，不得再次产生重复业务数据
- 批量操作要求：批量审核、批量上下架、批量导出等批量写操作必须记录批次号或 `request_id`，用于审计、失败补偿与重复请求去重
- 路由模型绑定：如 `jobs/{job}`、`applications/{application}`，优先使用 Laravel 路由模型绑定减少重复查询与空值判断
- 限流：登录、注册、上传、聊天、评论、职位刷新、验证码、WebSocket 握手等高频入口统一通过 Laravel `RateLimiter::for()` 管理策略
- 限流维度：必须按用户ID、IP、设备标识、接口类型等维度组合限流；未登录场景至少按 IP 与设备标识限流
- 限流响应：触发频率限制时必须返回 HTTP 429、业务 `code`=6000-6999，并在响应头返回 `Retry-After`，响应体返回 `retry_after`（单位：秒）

【Laravel 11.x 路由与控制器规范】
- API 路由定义在 routes/api.php；如需页面路由再使用 routes/web.php
- 使用 Route::apiResource()、Route::prefix()、Route::middleware() 组织 RESTful 资源接口
- 控制器位于 app/Http/Controllers/Api/ 目录，配合 FormRequest、API Resources、Policy、Service 类保持职责清晰
- Laravel 11 的中间件注册与别名维护通过 bootstrap/app.php 统一装配，也可在路由中直接声明 middleware
- 用户端与管理员端均使用 Bearer Token 鉴权；可通过自定义 guard / 中间件区分 user、admin、ws-token 等上下文

【路由示例 - Laravel】
```php
// routes/api.php
use App\Http\Controllers\Api\JobController;
use Illuminate\Support\Facades\Route;

Route::middleware(['api'])->group(function () {
    Route::middleware(['auth.jwt:user'])->group(function () {
        Route::apiResource('jobs', JobController::class);
        Route::post('jobs/{job}/refresh', [JobController::class, 'refresh']);
        Route::post('jobs/{job}/apply-top', [JobController::class, 'applyTop']);
    });

    Route::prefix('admin')->middleware(['auth.jwt:admin'])->group(function () {
        Route::get('job/list', [JobController::class, 'adminList']);
    });
});
```

【控制器示例 - Laravel】
```php
// app/Http/Controllers/Api/JobController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StoreJobRequest;
use App\Http\Resources\JobResource;
use App\Models\Job;
use App\Services\JobService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class JobController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        $jobs = Job::query()
            ->with(['categories', 'company'])
            ->latest('id')
            ->paginate((int) $request->input('limit', 20));

        return response()->json([
            'code' => 200,
            'msg' => 'success',
            'data' => [
                'list' => JobResource::collection($jobs->items()),
                'total' => $jobs->total(),
                'page' => $jobs->currentPage(),
                'limit' => $jobs->perPage(),
                'total_pages' => $jobs->lastPage(),
            ],
        ]);
    }

    public function store(StoreJobRequest $request, JobService $jobService): JsonResponse
    {
        $this->authorize('create', Job::class);

        $job = DB::transaction(function () use ($request, $jobService) {
            return $jobService->createByEmployer($request->user(), $request->validated());
        });

        return response()->json([
            'code' => 200,
            'msg' => '职位发布成功',
            'data' => new JobResource($job->load(['categories', 'company'])),
        ]);
    }
}
```

说明：
- 控制器仅负责鉴权、授权、事务边界与响应封装
- 复杂业务放入 Service 层；验证放入 FormRequest；输出放入 Resource；权限走 Policy
- 如需发送通知、刷新统计、写操作日志，可在 Service 内派发 Job / Event 异步处理

6.2 错误代码规范
--------------------------------------------------

【错误响应映射规则】【新增】【P0】
- 1000-1999：系统级错误，默认对应 HTTP 500
- 2000-2999：参数错误，默认对应 HTTP 400
- 3000-3999：认证与权限错误，其中未登录/登录失效默认对应 HTTP 401，无权限默认对应 HTTP 403
- 4000-4999：业务错误；资源不存在默认对应 HTTP 404，其余按具体业务语义通常对应 HTTP 400 / 409 / 422，若无特殊说明统一按 HTTP 400 处理
- 5000-5999：数据验证错误，默认对应 HTTP 400
- 6000-6999：频率限制错误，默认对应 HTTP 429
- 7000-7999：内容安全错误，默认对应 HTTP 400
- 8000-8999：管理员权限错误，默认对应 HTTP 403
- 9000-9999：WebSocket 错误，仅用于 WebSocket 通道的事件响应或握手失败反馈；若通过 HTTP 接口签发/校验相关参数失败，仍按 HTTP 接口状态码规则返回

【预留码段说明】【新增】
- 1010-1099：系统级预留（如系统维护、计划任务异常等）
- 2010-2099：参数验证预留
- 3010-3099：认证预留（如设备绑定、异地登录等）
- 4018-4099：业务错误预留
- 5010-5099：数据验证预留
- 6010-6099：限流预留
- 7010-7099：内容安全预留
- 8010-8099：管理员权限预留
- 9010-9099：WebSocket预留

【错误码管理规范】【新增】
- 新增错误码需评审维度：区间占用（是否与现有码段冲突）、语义唯一性（是否已有相同语义的错误码）
- 错误码命名格式：`{错误码}: {简短描述}`，描述应清晰表达错误原因，避免歧义
- 错误码使用优先级：优先使用已有错误码，其次使用预留码段，最后申请新增码段
- 禁止事项：禁止占用未定义码段、禁止为相同语义定义多个错误码、禁止删除已发布的错误码（可标记为废弃）
- 废弃错误码处理：废弃错误码保留定义，注释说明废弃原因，不再新用

【系统级错误 1000-1999】
1000: 系统错误
1001: 服务不可用
1002: 数据库错误
1003: 缓存错误
1004: 第三方服务错误

【参数错误 2000-2999】
2000: 参数错误
2001: 缺少必填参数
2002: 参数格式错误
2003: 参数值超出范围
2004: 参数类型错误

【认证错误 3000-3999】
3000: 未登录
3001: Token无效或过期
3002: Token即将过期（续期提示）
3003: 无权限访问
3004: 账号被封禁
3005: 账号已注销
3006: 登录失败次数过多，账号已临时锁定【新增】
3007: 当前密码错误【新增】
3008: Token已在黑名单（用户已退出但Token仍被使用）【新增】【P1】
3009: 新密码与旧密码相同，不允许重复设置【新增】【P2】

【WebSocket错误 9000-9999】【新增】【P0】
9000: WebSocket连接错误
9001: WebSocket Token无效或过期（鉴权失败）
9002: WebSocket消息格式错误
9003: WebSocket发送消息过于频繁
9004: WebSocket消息内容违规
9005: WebSocket对方已拉黑你
9006: WebSocket对方不在线（仅用于特定场景）
9007: WebSocket会话不存在或已关闭
9008: WebSocket消息撤回超时（超过2分钟）
9009: WebSocket无权执行此操作

【业务错误 4000-4999】
4000: 业务处理失败
4001: 记录不存在
4002: 记录已存在
4003: 状态不允许此操作
4004: 名额已满
4005: 已报名过
4006: 简历未完善
4007: 未通过认证
4008: 审核中，请等待
4009: 重新报名时间限制未到（需等待至can_reapply_time）【新增】
4010: 职位已过期【新增】
4011: 职位已暂停【新增】
4012: 草稿内容不完整，无法发布【新增】
4013: 不能报名自己发布的职位【新增】
4014: 会话不存在或已关闭【新增】
4015: 新手机号已被其他账号注册【新增】【P1】
4016: 企业名称已被认证占用【新增】【P2】
4017: 工种不存在或已禁用【新增】【P2】

【数据验证错误 5000-5999】
5000: 验证失败
5001: 手机号格式错误
5002: 邮箱格式错误
5003: 密码格式错误
5004: 验证码错误
5005: 验证码已过期
5006: 图片格式错误
5007: 文件大小超限
5008: 手机号未注册【新增】
5009: 邮箱未绑定【新增】

【频率限制错误 6000-6999】
6000: 请求过于频繁
6001: 超出每日限额
6002: 超出每小时限额
6003: IP被限制
6004: 职位刷新次数超限（达到 system_config.refresh_limit_per_day 每日限额）【新增】
6005: 公共聊天发言过于频繁【新增】

【内容安全错误 7000-7999】
7000: 内容包含违禁词
7001: 内容包含敏感信息
7002: 内容涉嫌违规
7003: 图片涉嫌违规
7004: 联系方式被过滤拦截（消息中含电话/微信被系统过滤）【新增】

【管理员权限错误 8000-8999】【新增】
8000: 管理员权限不足
8001: 该操作需要超级管理员权限
8002: 无该模块操作权限
8003: 操作已被其他管理员处理
8004: 管理员账号已被禁用【新增】

6.3 用户相关接口
--------------------------------------------------

【Laravel 模块实现建议】
- 推荐 FormRequest：`RegisterRequest`、`LoginRequest`、`CompleteProfileRequest`、`UpdateUserRequest`、`ChangePasswordRequest`、`ResetPasswordRequest`、`BindEmailRequest`
- 推荐 Service / Action：`AuthService`、`UserProfileService`、`PasswordService`、`DeviceSessionService`、`CaptchaService`
- 推荐模型与关系：`User`、`LoginSession`、`Resume`、`Company`；用户与设备会话、简历、企业认证建议通过 Eloquent 关系统一访问
- 推荐中间件：`auth.jwt:user`、`token.refresh`、`verified.email`（如启用）、`throttle:*`
- 可异步化任务：发送欢迎通知、邮箱验证码、找回密码邮件、异地登录提醒、设备踢出通知
- 敏感字段处理：手机号、邮箱、身份证、设备信息在 Resource 层统一脱敏；密码修改/重置必须写入审计日志并使旧 token 失效
- 事务边界建议：注册成功后的"用户创建 + 默认资料初始化 + 登录会话写入"可在事务内完成；邮件通知、欢迎消息、统计写入可异步派发
- 路由模型绑定建议：设备移除接口可使用 `loginSessions/{loginSession}` 或内部通过当前用户 + device_id 解析，避免越权删除他人设备
- 登录态管理建议：登录成功后统一写入 `login_sessions`，退出登录/踢出设备/修改密码时统一通过 `DeviceSessionService` 回收会话与 token 黑名单；**设备数量限制**：同一账号最多允许 3 个设备同时登录，超过限制时自动顶替最早且不活跃的会话【新增】【P0】
- 设备会话建议：为每次登录生成唯一 `device_session_id`，JWT、ws_token、设备管理、自动续期、强制下线均围绕 `device_session_id` 统一建模；服务端校验 JWT 时除验证签名与过期时间外，还需校验 `device_session_id` 是否仍处于有效状态【新增】【P0】
- 强制失效联动规则：退出登录、退出其他设备、修改密码、注销账号、账号被封禁、管理员强制下线时，必须同时使相关 JWT 失效并加入 Redis 黑名单，且主动断开对应 WebSocket 连接，未使用的 ws_token 也必须立即失效【新增】【P0】
- 自动续期规则：当响应头返回 `X-New-Token` 时，客户端必须使用新 token 覆盖旧 token；旧 token 仅允许保留 60 秒并发宽限期，宽限期结束后必须失效；若旧 token 已进入黑名单，则不受宽限期保护并立即失效【新增】【P1】
- ws_token 规则：ws_token 仅用于 WebSocket 握手鉴权，不得作为 HTTP 登录态凭证，也不得脱离对应 `device_session_id` 单独长期代表登录状态【新增】【P0】

【6.3.1 获取图形验证码】【新增】
GET /api/user/captcha

请求参数：无（后端基于Session或临时key生成）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "captcha_key": "abc123",    // 验证码key（提交时携带）
    "captcha_img": "data:image/png;base64,..."  // base64图片
  }
}

说明：验证码有效期5分钟，每次请求生成新的

【6.3.2 用户注册】
POST /api/user/register

请求参数：
{
  "phone": "13800138000",
  "password": "123456",
  "password_confirm": "123456",
  "role": "seeker",
  "captcha_key": "abc123",    // 验证码key
  "captcha": "abcd"
}

（响应同原文）

【6.3.3 用户登录】
POST /api/user/login

请求参数：
{
  "phone": "13800138000",
  "password": "123456",
  "captcha_key": "abc123",
  "captcha": "abcd"
}

成功响应：
{
  "code": 200,
  "msg": "登录成功",
  "data": {
    "user_id": 10086,
    "phone": "13800138000",
    "nickname": "张三",
    "avatar_file": {
      "file_id": "f_20240306_avatar001",
      "disk": "public",
      "path": "avatars/2024/03/06/avatar001.jpg",
      "access": "public",
      "url": "https://...",
      "temporary_url": null,
      "mime_type": "image/jpeg",
      "size": 102400,
      "original_name": "avatar.jpg"
    },
    "role": "seeker",
    "is_certified": 2,
    "token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "token_expire": "2024-03-13 10:30:00"
  }
}

说明：
- 登录接口仅返回登录态信息；涉及头像等文件语义字段时，统一返回 File Resource 对象（如 `avatar_file`），不再返回裸 URL 字符串
- WebSocket 连接地址与 ws_token 请在登录成功后按需调用 GET /api/ws/config 获取，不在登录接口中直接下发

错误响应（账号锁定）：
{
  "code": 3006,
  "msg": "登录失败次数过多，账号已锁定",
  "data": {
    "locked_until": "2024-03-06 11:00:00",
    "lock_minutes_left": 28
  }
}

【6.3.4 完善用户信息（引导页）】【新增】
POST /api/user/complete-profile

请求头：Authorization: Bearer {token}

请求参数（求职者）：
{
  "nickname": "张三",
  "avatar_file_id": "f_20240306_abc123",  // 头像文件ID，通过上传接口获得；不传则不修改头像
  "real_name": "张三",
  "gender": "male",
  "birth_date": "1990-01-01",
  "education": "高中",
  "work_years": 5,
  "major_category": 1
}

请求参数（雇主）：
{
  "nickname": "张经理",
  "avatar_file_id": "f_20240306_abc123"  // 头像文件ID，通过上传接口获得；不传则不修改头像
}

成功响应：
{
  "code": 200,
  "msg": "信息完善成功",
  "data": {
    "user_id": 10086,
    "is_completed": true
  }
}

说明：
- 注册成功后调用此接口完善用户信息
- 求职者需填写基本信息并选择主要工种（单选）
- 雇主只需填写昵称和头像，后续需进行企业认证
- 此接口可复用简历保存接口，但需标记为首次完善，并与 resumes 的 birth_date / education / major_category 字段保持一致
- 【接口区分】complete-profile用于注册后首次完善（可选字段，引导用户填写）；resume/save用于简历管理页完整编辑（字段校验更严格）【新增】【P1】

【6.3.5 获取用户信息】
GET /api/user/info

请求头：
Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 10086,
    "phone": "138****8000",
    "nickname": "张三",
    "avatar_file": {
      "file_id": "f_20240306_avatar001",
      "disk": "public",
      "path": "avatars/2024/03/06/avatar001.jpg",
      "access": "public",
      "url": "https://...",
      "temporary_url": null,
      "mime_type": "image/jpeg",
      "size": 102400,
      "original_name": "avatar.jpg"
    },
    "email": "zhang****@qq.com",
    "role": "seeker",
    "is_certified": 2,
    "created_at": "2024-01-15 10:30:00"
  }
}

【6.3.6 修改个人信息】【编号修正】
PUT /api/user/update

请求参数：
{
  "nickname": "李四",
  "avatar_file_id": "f_20240306_abc123",  // 头像文件ID，通过上传接口获得；不传则不修改头像
  "email": "lisi@qq.com"
}

成功响应：
{
  "code": 200,
  "msg": "修改成功",
  "data": null
}

【6.3.7 修改密码】
PUT /api/user/change-password

请求参数：
{
  "old_password": "123456",
  "new_password": "654321",
  "new_password_confirm": "654321"
}

成功响应：
{
  "code": 200,
  "msg": "密码修改成功，请重新登录",
  "data": null
}

错误响应：
{
  "code": 3007,
  "msg": "当前密码错误",
  "data": null
}

【6.3.8 账号注销】
POST /api/user/delete

请求参数：
{
  "password": "123456"
}

成功响应：
{
  "code": 200,
  "msg": "注销申请已提交，{account_delete_days}天后生效",  // 天数由系统配置项 account_delete_days 决定，默认7天
  "data": {
    "delete_apply_time": "2024-03-06 10:00:00",
    "delete_effective_time": "2024-03-13 10:00:00"
  }
}

【6.3.9 取消注销申请】【新增】
POST /api/user/cancel-delete

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "已成功取消注销申请",
  "data": null
}

【6.3.10 退出登录】【新增】【P0】
POST /api/user/logout

请求头：Authorization: Bearer {token}

请求参数：无

成功响应：
{
  "code": 200,
  "msg": "退出成功",
  "data": null
}

说明：
- 服务端将当前Token加入Redis黑名单
- 清除客户端登录状态
- 必须同时将当前 `device_session_id` 标记为失效，并主动断开该设备对应的 WebSocket 连接【新增】【P0】
- 该设备对应的未使用 ws_token 必须立即失效，不得继续用于建立新连接【新增】【P0】
- 可选：清除设备关联记录
- 其他设备保持在线；若需踢出其他设备，应通过设备管理接口单独处理【新增】【P1】

【6.3.11 获取未读消息总数】【新增】【P1】
GET /api/user/unread-count

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "total": 15,
    "private_message": 8,      // 私聊未读
    "notification": 5,         // 系统通知未读
    "application": 2           // 报名状态变更未读
  }
}

说明：用于底部导航消息角标显示

【6.3.12 找回密码申请】【修正】
POST /api/user/forgot-password

说明：用户提交找回密码申请，由客服人工处理

请求参数：
{
  "phone": "13800138000",
  "captcha_key": "abc123",
  "captcha": "abcd"
}

成功响应：
{
  "code": 200,
  "msg": "申请已提交，请联系客服完成身份验证",
  "data": {
    "service_phone": "400-xxx-xxxx",
    "service_hours": "9:00-18:00"
  }
}

错误响应：
{
  "code": 5008,
  "msg": "手机号未注册",
  "data": null
}

【6.3.12.1 找回密码（邮箱申请）】【新增】
POST /api/user/forgot-password-email

说明：已绑定邮箱用户通过邮箱验证码/邮件链接找回密码

请求参数：
{
  "email": "zhangsan@example.com",
  "captcha_key": "abc123",
  "captcha": "abcd"
}

成功响应：
{
  "code": 200,
  "msg": "邮件已发送，请在30分钟内完成验证",
  "data": {
    "email": "zhangsan@example.com"
  }
}

错误响应：
{
  "code": 5009,
  "msg": "邮箱未绑定",
  "data": null
}

【6.3.12.2 找回密码（邮箱设置新密码）】【新增】
POST /api/user/reset-password-email

请求参数：
{
  "reset_token": "reset_token_xxx",
  "new_password": "654321",
  "new_password_confirm": "654321"
}

成功响应：
{
  "code": 200,
  "msg": "密码重置成功，请重新登录",
  "data": null
}

错误响应：
{
  "code": 3001,
  "msg": "重置令牌无效或已过期",
  "data": null
}

【6.3.13 修改手机号】【新增】
PUT /api/user/change-phone

请求参数（步骤1）：
{
  "step": 1,
  "password": "123456"
}

请求参数（步骤2）：
{
  "step": 2,
  "verify_token": "verify_token_xxx",  // 步骤1返回的验证令牌（必填）
  "new_phone": "13900139000",
  "captcha_key": "abc123",
  "captcha": "abcd"
}

成功响应（步骤1）：
{
  "code": 200,
  "msg": "身份验证成功",
  "data": {
    "verify_token": "verify_token_xxx"  // 有效期10分钟，一次性使用
  }
}

成功响应（步骤2）：
{
  "code": 200,
  "msg": "手机号修改成功，请重新登录",
  "data": null
}

错误响应（步骤2 verify_token无效）：
{
  "code": 3001,
  "msg": "验证令牌无效或已过期，请重新验证身份",
  "data": null
}

说明：
- 步骤1验证当前密码，返回 verify_token（存储于Redis，TTL=10分钟）
- 步骤2必须携带步骤1返回的 verify_token，服务端校验通过后才允许修改手机号
- verify_token 一次性使用，修改成功后立即失效
- 修改手机号成功后，当前Token失效，需使用新手机号重新登录

【6.3.14 获取设备列表】【新增】
GET /api/user/devices

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "current_device": {
      "device_id": "device_001",
      "device_name": "iPhone 15 Pro",
      "device_type": "ios",
      "login_time": "2024-03-06 10:00:00",
      "ip_address": "192.168.1.100"
    },
    "other_devices": [
      {
        "device_id": "device_002",
        "device_name": "Windows PC",
        "device_type": "web",
        "browser": "Chrome 122",
        "login_time": "2024-03-05 14:30:00",
        "last_active": "2024-03-05 18:00:00",
        "ip_address": "192.168.1.50"
      }
    ]
  }
}
说明：
- current_device 由当前请求携带的 token/session_token 对应的 login_sessions 记录动态识别
- current_device / other_devices 的划分以 `device_session_id` 为准，同一 `device_session_id` 续签产生的新旧 token 视为同一设备会话【新增】【P1】
- other_devices 返回同一用户下除当前会话外、未撤销且未过期的其他登录设备
- **设备数量限制**：同一账号最多允许 **3 个设备**同时登录；超过限制时，最早登录且当前不活跃的设备会话将被自动顶替下线

【6.3.15 移除设备】【新增】
DELETE /api/user/device/{device_id}

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "设备已移除",
  "data": null
}
说明：
- 仅允许移除 other_devices，不可移除当前请求对应的 current_device
- 移除后将对应 login_sessions 记录标记为 is_revoked=1，并使该设备当前 Token 失效
- 被移除设备对应的 `device_session_id` 必须整体失效；该设备已建立的 WebSocket 连接需被主动断开，未使用的 ws_token 也必须立即作废【新增】【P0】
- 当前设备保持登录状态，不受影响【新增】【P1】

【6.3.16 退出所有其他设备】【新增】
POST /api/user/logout-others

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "已退出所有其他设备",
  "data": {
    "logout_count": 2
  }
}
说明：
- logout_count 表示除当前请求对应会话外，被撤销的其他设备会话数量
- 当前请求对应设备保持登录状态，其余同用户设备会话统一标记为 is_revoked=1 并使 Token 失效
- 所有被退出的其他设备，其对应 `device_session_id` 的现存 WebSocket 连接需被主动断开，未使用的 ws_token 必须立即失效【新增】【P0】
- 当前设备对应 `device_session_id` 不受影响，当前请求使用的 token 可继续使用或按自动续期规则续签【新增】【P1】

【6.3.17 邮件通知设置】【新增】
PUT /api/user/email-notification

请求头：Authorization: Bearer {token}

请求参数：
{
  "email_notification_enabled": true  // 邮件通知开关：true开启/false关闭（必填）
}

成功响应：
{
  "code": 200,
  "msg": "设置成功",
  "data": {
    "email_notification_enabled": true
  }
}

错误响应：
{
  "code": 4001,
  "msg": "请先绑定邮箱后再设置邮件通知",
  "data": null
}

说明：
- 邮件通知开关用于控制私聊回复与系统通知是否同步发送邮件
- 开启邮件通知前必须已绑定邮箱，未绑定邮箱时返回错误码4001
- 该设置为用户偏好，应在用户表中增加相应字段存储

6.4 企业认证相关接口【新增】
--------------------------------------------------

【Laravel 模块实现建议】
- 推荐 FormRequest：`SubmitCompanyRequest`、`UpdateCompanyRequest`、`AuditCompanyRequest`
- 推荐 Service / Action：`CompanyCertificationService`、`CompanyAuditService`、`CompanyFileService`
- 推荐 Policy：`CompanyPolicy`（本人查看/修改）、`AdminCompanyPolicy`（审核、查看敏感资质、解密信息）
- 推荐 Job / Notification：`SyncCompanyAuditTodoJob`、`SendCompanyAuditResultNotification`、`ArchiveSensitiveFileJob`
- 事务边界建议：企业认证提交、管理员审核（含 users.is_certified 更新、companies 状态更新、通知写入、日志写入）必须放在事务内
- 【is_certified状态机】状态定义：0未认证/1审核中/2已认证/3认证被拒绝；首次提交认证→is_certified=1；审核通过→is_certified=2；审核拒绝→is_certified=3；重新提交认证→is_certified=1（审核中）【新增】【P0】
- 敏感字段处理：法人身份证号通过 Service 层 AES-256 加密后写入数据库；资质文件在业务提交阶段统一使用 file_id 引用，落库时内部可解析为 private disk 相对路径，不加密，通过签名 URL 机制控制访问；统一社会信用代码无需加密，但接口输出时可酌情脱敏；不在控制器直接拼接任何敏感字段【更新】【P0】
- 路由模型绑定建议：`/api/company/{company}` 适合后台接口；前台"我的企业认证详情"仍可保持 `/api/company/detail`，由当前登录用户反查
- 审核事件建议：审核通过/拒绝后派发领域事件，统一触发通知、消息中心、后台待办刷新、统计更新
- 建议补充 Query Scope：`pending()`、`approved()`、`rejected()`，便于后台审核列表复用查询条件

【6.4.1 提交企业认证】
POST /api/company/submit

请求头：Authorization: Bearer {token}
Content-Type: application/json

说明：资质图片须先通过 POST /api/upload/image（type=cert）上传，获得 file_id 后再提交本接口；本接口不接受文件二进制，只接受 file_id 引用。

请求参数：
- company_type: enterprise / labor / individual（必填）
- company_name: 企业名称（企业/劳务必填）
- unified_social_code: 统一社会信用代码（企业/劳务建议必填）
- business_license_file_id: 营业执照文件ID（企业/劳务必填，通过上传接口获得）
- legal_person_name: 法人姓名（企业/劳务必填）
- legal_person_id_card: 法人身份证号明文（企业/劳务必填，Service层加密后存储）
- legal_person_id_card_front_file_id: 法人身份证正面文件ID（企业/劳务必填）
- legal_person_id_card_back_file_id: 法人身份证反面文件ID（企业/劳务必填）
- id_card_front_file_id: 身份证正面文件ID（个人雇主必填）
- id_card_back_file_id: 身份证反面文件ID（个人雇主必填）
- contact_person: 联系人（必填）
- contact_phone: 联系电话（必填）

Laravel 实现说明：
- 使用 FormRequest 校验企业类型差异字段
- Service 层根据 file_id 查询 files 表，校验文件归属当前用户且 type=cert，**直接将 file_id 写入 companies 对应字段**【更新】【P0】
- 对外接口层统一使用 File Resource 对象返回资质材料【更新】【P0】
- **资质图片统一以 file_id 存入数据库**，Service 层创建 companies 记录【更新】【P0】
- Service 层需根据 company_type 写入对应材料字段，并同步将完整材料快照落到 company_audit_histories
- 提交成功后应派发审核通知/后台待办刷新事件，可异步写入通知与操作日志
- 涉及同一用户重复提交、企业名称占用、统一社会信用代码重复等情况，优先通过唯一约束 + 业务校验双层保障

成功响应：
{
  "code": 200,
  "msg": "认证信息已提交，等待管理员审核",
  "data": {
    "company_id": 1,
    "audit_status": 0
  }
}

【6.4.2 获取企业认证详情】
GET /api/company/detail

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "company_name": "XX建设集团",
    "company_type": "enterprise",
    "unified_social_code": "91310000XXXXXXXXXX",
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
    },
    "id_card_front_file": null,
    "id_card_back_file": null,
    "legal_person_name": "张经理",
    "legal_person_id_card": "310***********1234",
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
    "contact_person": "张经理",
    "contact_phone": "138****8888",
    "audit_status": 1,
    "audit_time": "2024-03-06 12:00:00",
    "audit_reason": null,
    "reapply_count": 0,
    "created_at": "2024-03-06 10:00:00"
  }
}

说明：
- 资质材料为 private 文件，返回的 `temporary_url` 为服务端按需生成的临时签名地址，有效期建议15分钟
- 仅当前登录用户可查看自己的企业认证详情及对应资质材料，不返回明文存储路径，也不得自行拼接访问地址
- `business_license_file`、`id_card_front_file`、`id_card_back_file`、`legal_person_id_card_front_file`、`legal_person_id_card_back_file` 均按 File Resource 结构返回；若材料不存在则返回 null

6.5 简历相关接口
--------------------------------------------------

【Laravel 模块实现建议】
- 推荐 FormRequest：`SaveResumeRequest`、`UpdateResumePrivacyRequest`
- 推荐 Service / Action：`ResumeService`、`ResumeCompletionService`
- 推荐 Policy：`ResumePolicy`（本人编辑、雇主查看、管理员查看）
- 推荐 Resource：`ResumeResource`、`ResumeDetailResource`、`ResumeWorkExperienceResource`
- 数据组织建议：工作经历如需高频增删改查，建议拆分独立表 `resume_experiences`；若仅简化存储，也可保留 JSON 字段并通过 `$casts` 管理
- 事务边界建议：保存简历时涉及主表更新 + 工作经历明细写入时应使用事务；主工种字段统一为单值 major_category
- 可异步化任务：简历完善度重算、搜索索引刷新、人才库匹配推荐
- 敏感字段处理：真实姓名、联系方式、年龄、工作经历对雇主端展示需按权限脱敏；公开简历与私密简历应由 Policy + Query Scope 双重控制
- 查询建议：简历列表/详情对外输出统一通过 `ResumeResource`，避免在多个接口重复计算年龄、工龄、公开状态

【6.5.1 获取简历】
GET /api/resume/detail

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "user_id": 10086,
    "real_name": "张三",
    "gender": "male",
    "birth_date": "1990-01-01",
    "work_years": 5,
    "education": "高中",
    "major_category_id": 1,
    "major_category_name": "电工",
    "work_experience": [
      {
        "company": "XX建设公司",
        "position": "电工",
        "start_date": "2018-01",
        "end_date": "2023-12",
        "description": "负责工地电路安装维护"
      }
    ],
    "skills": "熟练掌握低压电工技能...",
    "is_public": 1
  }
}

【6.5.2 保存简历】
POST /api/resume/save

请求参数：
{
  "real_name": "张三",
  "gender": "male",
  "birth_date": "1990-01-01",
  "work_years": 5,
  "education": "高中",
  "major_category_id": 1,
  "work_experience": [
    {
      "company": "XX建设公司",
      "position": "电工",
      "start_date": "2018-01",
      "end_date": "2023-12",
      "description": "负责工地电路安装维护"
    }
  ],
  "skills": "熟练掌握低压电工技能...",
  "is_public": 1
}

成功响应：
{
  "code": 200,
  "msg": "简历保存成功",
  "data": null
}


【6.5.3 雇主查看求职者简历】【新增】
GET /api/resume/view

请求头：Authorization: Bearer {token}（雇主身份）

请求参数：
- user_id: 求职者用户ID（必填）
- application_id: 关联报名ID（必填，验证报名关系）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "real_name": "张三",
    "gender": "male",
    "age": 30,
    "work_years": 5,
    "education": "高中",
    "major_category_name": "电工",
    "work_experience": [...],
    "skills": "熟练掌握低压电工技能..."
  }
}

6.6 职位相关接口
--------------------------------------------------

【Laravel 模块实现建议】
- 推荐 FormRequest：`StoreJobRequest`、`UpdateJobRequest`、`RefreshJobRequest`、`ApplyTopJobRequest`、`JobSearchRequest`
- 推荐 Service / Action：`JobService`、`JobDraftService`、`JobRefreshService`、`JobTopApplicationService`、`JobSearchService`
- 推荐 Policy：`JobPolicy`（创建、编辑、删除、刷新、置顶申请、查看后台详情）
- 推荐 Query Scope：`active()`、`approved()`、`notExpired()`、`visibleForList()`、`ownedByEmployer()`
- 事务边界建议：职位发布/保存草稿/复制职位/置顶申请/刷新次数扣减等涉及多表或计数写入时必须使用事务
- 可异步化任务：职位审核通知、搜索索引刷新、统计聚合、浏览量校准、推荐位更新
- 路由模型绑定建议：优先使用 `jobs/{job}`，并结合 Policy 避免雇主操作他人职位
- 搜索实现建议：列表接口可先基于 MySQL 8 索引 + 条件查询实现，后续如接入 Elasticsearch/Meilisearch 再抽象 SearchService

【6.6.1 职位列表】
GET /api/job/list

请求头：Authorization: Bearer {token}（可选，登录后传token可获取收藏状态）

请求参数：
- page: 页码（默认1）
- limit: 每页数量（默认20）
- category_id: 工种ID（可选）
- location: 工作地点（可选）
- min_salary: 最低工价（可选）
- max_salary: 最高工价（可选）
- work_period: 工期（可选）
- settlement_type: 结算方式（可选）
- accommodation: 是否包住（可选，0/1）
- meals: 是否包吃（可选，0/1）
- level: 招聘级别（可选，senior/medium/junior）
- keyword: 搜索关键词（可选）
- sort: 排序方式（默认latest，可选：latest最新/views浏览最多/salary工资最高）

排序规则：
- **置顶优先**：所有排序方式下，`is_top=1` 且 `top_expire_time > 当前时间` 的职位始终排在最前
- 置顶职位之后，按选定的排序方式排序（latest按刷新时间/ views按浏览次数/ salary按薪资）
- `is_top=1` 但 `top_expire_time <= 当前时间` 的职位视为普通职位

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
        "company_name": "XX建设集团",
        "is_certified": 1,
        "work_location": "北京朝阳",
        "categories": [
          {
            "category_id": 1,
            "category_name": "电工",
            "levels": [
              {"level": "senior", "salary": 500, "recruit_count": 3, "applied_count": 1},
              {"level": "medium", "salary": 350, "recruit_count": 5, "applied_count": 2},
              {"level": "junior", "salary": 200, "recruit_count": 2, "applied_count": 0}
            ]
          }
        ],
        "work_period": "3个月",
        "settlement_type": "monthly",
        "accommodation": 1,
        "meals": 1,
        "tags": ["长白班"],
        "is_top": 1,
        "views_count": 128,
        "created_at": "2024-03-06 08:00:00",
        "refreshed_at": "2024-03-06 10:00:00",
        "is_favorited": false  // 是否已收藏（登录用户返回，未登录不返回）【新增】【P1】
      }
    ],
    "total": 100,
    "page": 1,
    "limit": 20,
    "total_pages": 5
  }
}

说明：
- 登录用户请求时，返回is_favorited字段标识是否已收藏该职位
- 未登录用户请求时，不返回is_favorited字段

【6.6.2 热门搜索词】【新增】
GET /api/job/hot-words

请求参数：无

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "hot_words": [
      {"word": "电工", "count": 1580},
      {"word": "焊工", "count": 1230},
      {"word": "钳工", "count": 980},
      {"word": "车工", "count": 850},
      {"word": "北京", "count": 720},
      {"word": "上海", "count": 680}
    ],
    "updated_at": "2024-03-06 10:00:00"
  }
}

说明：
- 热门搜索词每小时更新一次（Redis缓存）
- 首次进入搜索页时调用此接口获取热门词
- count为近7天搜索次数

【6.6.3 职位搜索】【新增】
GET /api/job/search

请求参数：
- keyword: 搜索关键词（可选，为空时返回热门职位）
- category_id: 工种ID（可选）
- location: 工作地点（可选）
- min_salary: 最低工价（可选）
- max_salary: 最高工价（可选）
- sort: 排序：latest/salary/match（默认latest）
- page: 页码
- limit: 每页数量

排序规则：
- **置顶优先**：所有排序方式下，`is_top=1` 且 `top_expire_time > 当前时间` 的职位始终排在最前
- 置顶职位之后，按选定的排序方式排序（latest按刷新时间/ salary按薪资/ match按匹配度）
- `is_top=1` 但 `top_expire_time <= 当前时间` 的职位视为普通职位

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [...],
    "total": 36,
    "page": 1,
    "limit": 20,
    "total_pages": 2
  }
}

【6.6.4 职位详情】
GET /api/job/detail

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
    "company_id": 1,
    "company_name": "XX建设集团",
    "company_type": "enterprise",
    "is_certified": 1,
    "work_location": "北京市朝阳区建国门外大街",
    "categories": [
      {
        "category_id": 1,
        "category_name": "电工",
        "icon": "electric",
        "levels": [
          {
            "id": 1,
            "level": "senior",
            "level_name": "大工",
            "salary": 500,
            "recruit_count": 3,
            "applied_count": 1
          },
          {
            "id": 2,
            "level": "medium",
            "level_name": "中工",
            "salary": 350,
            "recruit_count": 5,
            "applied_count": 2
          },
          {
            "id": 3,
            "level": "junior",
            "level_name": "小工",
            "salary": 200,
            "recruit_count": 2,
            "applied_count": 0
          }
        ]
      }
    ],
    "work_period": "3个月",
    "work_time": "8:00-18:00",
    "settlement_type": "monthly",
    "accommodation": 1,
    "meals": 1,
    "requirements": "1. 有电工证\n2. 3年以上工作经验\n3. 能吃苦耐劳",
    "education": "不限",
    "age_range": "18-50岁",
    "tags": ["长白班"],
    "images": [
      {
        "file_id": 101,
        "disk": "public",
        "path": "jobs/2024/03/06/img_001.jpg",
        "access": "public",
        "url": "https://example.com/storage/jobs/2024/03/06/img_001.jpg",
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
        "url": "https://example.com/storage/jobs/2024/03/06/img_002.jpg",
        "temporary_url": null,
        "mime_type": "image/jpeg",
        "size": 267890,
        "original_name": "job-image2.jpg"
      }
    ],
    "contact_person": "张经理",
    "contact_phone": "138****8888",  // 未报名时脱敏
    "views_count": 128,
    "status": "active",
    "is_top": 1,
    "created_at": "2024-03-06 08:00:00",
    "refreshed_at": "2024-03-06 10:00:00",
    "user_application": null,  // 当前用户的报名信息（如果已报名）
    "is_favorited": false,  // 是否已收藏（登录用户返回）【新增】【P1】
    "top_application": null  // 置顶申请状态（仅雇主查看自己职位时返回）【新增】【P1】
  }
}

说明：
- 调用此接口时自动写入浏览记录（job_views表），防刷规则见5.16
- 登录用户返回is_favorited字段标识是否已收藏
- 雇主查看自己职位时，返回top_application置顶申请状态
- 浏览记录写入采用异步队列，不影响接口响应速度

【6.6.5 雇主获取自己的职位列表】【新增】【编号修正】
GET /api/job/my-list

请求头：Authorization: Bearer {token}（雇主身份）

请求参数：
- status: 状态筛选（可选：draft/pending/rejected/active/paused/expired/closed/deleted）
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
        "job_no": "JOB202403060001",
        "title": "电工招聘",
        "status": "active",
        "is_top": 1,
        "views_count": 128,
        "apply_total": 15,
        "categories_summary": "电工、焊工",
        "created_at": "2024-03-06 08:00:00",
        "refreshed_at": "2024-03-06 10:00:00"
      }
    ],
    "total": 5,
    "page": 1,
    "limit": 20
  }
}

【6.6.6 发布职位】【编号修正】
POST /api/job/publish

请求头：
Authorization: Bearer {token}

请求参数：
{
  "title": "电工招聘",
  "work_location": "北京市朝阳区建国门外大街",
  "work_period": "3个月",
  "work_time": "8:00-18:00",
  "categories": [
    {
      "category_id": 1,
      "levels": [
        {"level": "senior", "salary": 500, "recruit_count": 3},
        {"level": "medium", "salary": 350, "recruit_count": 5},
        {"level": "junior", "salary": 200, "recruit_count": 2}
      ]
    },
    {
      "category_id": 2,
      "levels": [
        {"level": "senior", "salary": 600, "recruit_count": 2},
        {"level": "medium", "salary": 400, "recruit_count": 3},
        {"level": "junior", "salary": 250, "recruit_count": 1}
      ]
    }
  ],
  "settlement_type": "monthly",
  "accommodation": 1,
  "meals": 1,
  "requirements": "1. 有电工证\n2. 3年以上工作经验",
  "education": "不限",
  "age_range": "18-50岁",
"tags": ["长白班"],
    "image_file_ids": [101, 102],
  "contact_person": "张经理",
  "contact_phone": "13800138000"
}

Laravel 实现说明：
- 工作环境图片建议先通过统一上传接口或 multipart/form-data 上传，发布接口统一提交 `image_file_ids`（`file_id[]`）引用，不再传文件路径或以 base64 作为主方案
- `categories`、`tags` 等复杂结构建议由 FormRequest + DTO / Service 层统一解码和校验
- 仅企业认证通过（`users.is_certified = 2`）的雇主可调用正式发布；未认证/审核中仅允许保存草稿
- 发布审核规则（由 JobService 统一处理）：
  - 命中职位审核规则（如违规词、敏感内容、图片异常、联系方式异常、风控策略）→ status = pending、audit_status = 0（待审核）
  - 企业认证时间超过30天 → status = active、audit_status = 1（直接发布，视为系统自动审核通过）
  - 企业认证时间≤30天：
    ├─ 统计该企业认证通过后发布的职位数量（status != 'draft'）【明确计数规则】
    │   ├─ 排序规则：按 created_at ASC（按发布时间升序，取最早发布的职位）
    │   ├─ 统计范围：包括所有非草稿状态的职位（active/paused/closed），草稿不计入
    │   ├─ 不包括：已删除的职位
    │   └─ 说明："前N条" = 按发布时间最早发布的N条职位，这些需要审核
    ├─ 数量 < first_jobs_audit_limit（默认10） → status = pending、audit_status = 0（待审核）
    └─ 数量 >= first_jobs_audit_limit → status = active、audit_status = 1（直接发布，视为系统自动审核通过）
    └─ 配置项 first_jobs_audit_limit 可在管理员后台系统配置中调整
- 服务层需在事务内完成：职位写入 + 认证时间判断 + 数量统计 + 状态设置 + audit_status 设置 + 审核队列派发

成功响应：
{
  "code": 200,
  "msg": "职位发布成功",  // 或"职位已提交审核"
  "data": {
    "job_id": 1,
    "job_no": "JOB202403060001",
    "status": "active"  // 或"pending（职位审核中）"
  }
}
说明：
- `status = pending` 表示职位待审核，可能原因包括：新认证企业前N条职位需审核、命中职位审核规则、管理员强制要求审核等
- `status = pending` 不表示企业认证审核中
- 直接发布成功时，服务端应同时写入 `audit_status = 1`，表示该职位视为系统自动审核通过
- 企业认证未通过时，发布接口应返回业务错误并提示用户先完成认证或继续保存草稿

【6.6.7 保存草稿】【新增】【编号修正】
POST /api/job/save-draft

请求头：Authorization: Bearer {token}

请求参数：（同发布职位，但所有字段均为可选）
{
  "id": null,         // null表示新建草稿，有值表示更新已有草稿
  "title": "电工招聘（填写中）",
  "work_location": "北京",
  ...
}

成功响应：
{
  "code": 200,
  "msg": "草稿已保存",
  "data": {
    "job_id": 1,
    "updated_at": "2024-03-06 10:00:00"
  }
}

【6.6.8 编辑职位】【编号修正】
PUT /api/job/update

请求参数：
{
  "id": 1,  // 职位ID（必填）
  // 其他参数同发布职位
}

成功响应：
{
  "code": 200,
  "msg": "职位更新成功",
  "data": null
}

【6.6.9 刷新职位】【编号修正】
PUT /api/job/refresh

请求参数：
{
  "id": 1  // 职位ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "刷新成功",
  "data": {
    "refreshed_at": "2024-03-06 12:00:00"
  }
}

【6.6.10 删除职位】【编号修正】
DELETE /api/job/delete

请求参数：
{
  "id": 1  // 职位ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "职位已删除",
  "data": null
}


【6.6.11 切换职位状态（暂停/继续）】【新增】【编号修正】
PUT /api/job/toggle-status

请求头：Authorization: Bearer {token}

请求参数：
{
  "id": 1,             // 职位ID（必填）
  "action": "pause"    // pause暂停/resume继续（必填）
}

成功响应：
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "status": "paused"
  }
}

【6.6.12 复制职位】【新增】【编号修正】
POST /api/job/copy

请求头：Authorization: Bearer {token}

请求参数：
{
  "id": 1  // 源职位ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "职位已复制为草稿",
  "data": {
    "new_job_id": 2,
    "status": "draft"
  }
}

【6.6.13 申请置顶】【新增】【编号修正】
POST /api/job/apply-top

请求头：Authorization: Bearer {token}

请求参数：
{
  "job_id": 1,  // 职位ID（必填）
  "apply_days": 7  // 申请置顶天数（可选，默认7天，与表设计一致）【修正】【P1】
}

成功响应：
{
  "code": 200,
  "msg": "置顶申请已提交，等待管理员审核",
  "data": {
    "application_id": 1,
    "status": "pending",  // pending待审核/approved已通过/rejected已拒绝/expired已过期/cancelled已取消
    "apply_days": 7
  }
}

status状态说明：
- pending：待审核（申请已提交，等待管理员处理）
- approved：已通过（管理员审核通过，职位已置顶）
- rejected：已拒绝（管理员审核拒绝，可查看reason了解原因）
- expired：已过期（置顶有效期到期，自动失效）
- cancelled：已取消（用户主动取消或职位关闭时取消）

补充说明（新增）：
- 用户申请提交后，管理员会在后台看到申请并审核
- 管理员审核通过后，职位才会正式置顶
- 管理员可直接设置置顶，但系统会先检查是否有待审核的用户申请，如有则提示管理员去审核

【6.6.13.1 查询置顶申请状态】【新增】【P1】【编号修正】
GET /api/job/top-application-status

请求头：Authorization: Bearer {token}

请求参数：
- job_id: 职位ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "has_application": true,  // 是否有申请记录
    "application": {
      "id": 1,
      "job_id": 1,
      "apply_days": 7,
      "status": "pending",  // pending待审核/approved已通过/rejected已拒绝/expired已过期/cancelled已取消
      "reason": null,  // 拒绝原因（status=rejected时返回）
      "created_at": "2024-03-06 10:00:00",
      "handled_at": null  // 处理时间（status!=pending时返回）
    }
  }
}

错误响应（无申请记录）：
{
  "code": 200,
  "msg": "success",
  "data": {
    "has_application": false,
    "application": null
  }
}

说明：
- 雇主可在职位管理页调用此接口查看置顶申请进展
- status=applied时显示"待审核"，status=accepted时显示"已通过"，status=rejected时显示"已拒绝"及原因，status=cancelled时显示"已取消"

【职位模块补充实现建议】
- 针对职位评论子模块，建议补充 `StoreJobCommentRequest`、`ReplyJobCommentRequest`、`ToggleCommentLikeRequest`、`DeleteCommentRequest`
- 推荐 Service / Action：`JobCommentService`、`CommentLikeService`、`CommentModerationService`
- 推荐 Policy：`CommentPolicy`（删除本人评论、管理员删除、雇主举报查看）
- 评论发表/回复/点赞/删除涉及 comments、comment_likes、jobs.comments_count 等写入时建议使用事务或原子更新
- 评论内容应复用违禁词/联系方式过滤规则，并通过统一内容安全服务拦截；被举报评论可异步派发审核任务
- 职位统计接口建议由 `JobStatisticsService` 汇总，复杂聚合查询与图表数据整形不要放在控制器中

【6.6.14 职位数据统计（雇主端）】【新增】【编号修正】
GET /api/job/statistics

请求头：Authorization: Bearer {token}

请求参数：
- id: 职位ID（必填）
- days: 统计天数（7/30/all，默认7）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "summary": {
      "views_total": 128,
      "apply_total": 15,
      "approved_count": 8,
      "rejected_count": 5,
      "pass_rate": 0.533
    },
    "daily_views": [
      {"date": "2024-03-01", "count": 20},
      ...
    ],
    "daily_applies": [
      {"date": "2024-03-01", "count": 3},
      ...
    ],
    "level_distribution": [
      {"category_name": "电工", "level": "senior", "level_name": "大工", "count": 3},
      ...
    ]
  }
}

说明：【新增】
- 浏览量统计基于 job_views 表，按日聚合
- views_total 为统计周期内累计浏览次数，非实时在线人数
- daily_views 数组按日期倒序排列
- 统计接口每小时刷新一次聚合数据

【6.6.15 雇主整体数据统计】【新增】【编号修正】
GET /api/employer/statistics

请求头：Authorization: Bearer {token}

请求参数：
- days: 统计天数（7/30/all，默认7）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "summary": {
      "job_total": 5,
      "job_active": 3,
      "job_expired": 2,
      "views_total": 1280,
      "apply_total": 150,
      "approved_count": 80,
      "rejected_count": 50,
      "pending_count": 20,
      "pass_rate": 0.533
    },
    "daily_views": [
      {"date": "2024-03-01", "count": 200},
      ...
    ],
    "daily_applies": [
      {"date": "2024-03-01", "count": 25},
      ...
    ],
    "job_ranking": [
      {"job_id": 1, "job_title": "电工招聘", "views": 500, "applies": 50},
      {"job_id": 2, "job_title": "焊工招聘", "views": 300, "applies": 30},
      ...
    ],
    "category_distribution": [
      {"category_name": "电工", "apply_count": 80},
      {"category_name": "焊工", "apply_count": 50},
      ...
    ]
  }
}

6.7 评论相关接口【新增】
--------------------------------------------------

【6.7.1 获取职位评论列表】
GET /api/job/comments

请求参数：
- job_id: 职位ID（必填）
- sort: 排序：latest最新/hot最热（默认latest）
- page: 页码
- limit: 每页数量（默认20）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "user_id": 10086,
        "user_name": "用户A",
        "avatar_file": {
          "file_id": "f_20240306_avatar005",
          "disk": "public",
          "path": "avatars/2024/03/06/avatar005.jpg",
          "access": "public",
          "url": "https://...",
          "temporary_url": null,
          "mime_type": "image/jpeg",
          "size": 99840,
          "original_name": "avatar.jpg"
        },
        "content": "工资按时吗？",
        "likes_count": 5,
        "is_liked": false,    // 当前用户是否已点赞
        "is_top": 0,
        "parent_id": 0,
        "reply_to_user_id": null,
        "reply_to_user_name": null,
        "created_at": "2024-03-06 09:00:00",
        "replies": [
          {
            "id": 2,
            "user_id": 10087,
            "user_name": "雇主",
            "reply_to_user_id": 10086,
            "reply_to_user_name": "用户A",
            "content": "按时发放",
            "likes_count": 1,
            "is_liked": false,
            "created_at": "2024-03-06 09:30:00"
          }
        ]
      }
    ],
    "total": 23,
    "page": 1,
    "limit": 20
  }
}

【6.7.2 发表评论】
POST /api/job/comment

请求头：Authorization: Bearer {token}

请求参数：
{
  "job_id": 1,           // 职位ID（必填）
  "content": "工资按时吗？",  // 评论内容（必填）
  "parent_id": 0         // 父评论ID（0为一级评论，必填）
}

成功响应：
{
  "code": 200,
  "msg": "评论成功",
  "data": {
    "comment_id": 1,
    "created_at": "2024-03-06 10:00:00"
  }
}

【6.7.3 回复评论】
POST /api/job/comment/reply

请求头：Authorization: Bearer {token}

请求参数：
{
  "job_id": 1,
  "parent_id": 1,             // 父评论ID（必填）
  "content": "按时发放",       // 回复内容（必填）
  "reply_to_user_id": 10086   // 回复目标用户ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "回复成功",
  "data": {
    "comment_id": 2,
    "created_at": "2024-03-06 10:05:00"
  }
}

【6.7.4 点赞/取消点赞评论】
POST /api/job/comment/like

请求头：Authorization: Bearer {token}

请求参数：
{
  "comment_id": 1,     // 评论ID（必填）
  "action": "like"     // like点赞/unlike取消点赞（必填）
}

成功响应：
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "likes_count": 6,
    "is_liked": true
  }
}

【6.7.5 删除评论】
DELETE /api/job/comment/delete

请求头：Authorization: Bearer {token}

请求参数：
{
  "comment_id": 1  // 评论ID（必填，只能删除自己的评论）
}

成功响应：
{
  "code": 200,
  "msg": "评论已删除",
  "data": null
}

6.8 报名相关接口
--------------------------------------------------

【Laravel 模块实现建议】
- 推荐 FormRequest：`ApplyJobRequest`、`CancelApplicationRequest`、`AuditApplicationRequest`、`BatchAuditApplicationRequest`
- 推荐 Service / Action：`ApplicationService`、`ApplicationAuditService`、`ReapplyRuleService`
- 推荐 Policy：`ApplicationPolicy`（本人取消、雇主审核、管理员查看）
- 推荐 Job / Notification：`SendApplicationStatusNotification`、`SyncApplicationStatsJob`、`RecoverJobQuotaJob`
- 事务边界建议：报名、取消报名、审核通过/拒绝、批量审核必须放在事务内，尤其要保证名额、状态、通知、联系方式开放状态一致更新
- 幂等控制：报名接口必须基于唯一索引（user_id + job_id + category_id + level + 有效状态）与客户端幂等键双保险；命中重复报名时必须返回标准化业务响应，不得生成重复有效报名记录
- 批量审核必须防止重复提交，并记录批次号或 `request_id`，用于审计、失败补偿与重复请求去重
- 路由模型绑定建议：`applications/{application}` 并结合 Policy 校验归属与职位归属
- 规则建议：重新报名时间限制、不能报名自己职位、职位状态校验、简历完整度校验应集中放在 Service/Rule 类，不散落控制器
- 报名接口限流：必须通过 Laravel `RateLimiter::for()` 按用户ID + IP + 职位ID 组合限流；命中限流时返回 HTTP 429、业务 `code`=6000 或具体 6000-6999 错误码，并返回 `Retry-After` / `retry_after`

【报名审核子模块补充建议】
- 审核报名建议拆分 `ReviewApplicationAction`、`BatchReviewApplicationAction`，统一处理 approve/reject 两种流转
- 雇主留言、拒绝原因、联系方式开放状态建议在 Resource 中统一输出，避免不同接口字段含义不一致
- 报名通过后可异步触发 `SendApplicationApprovedNotification`、`OpenContactInfoJob`；拒绝后可触发 `SendApplicationRejectedNotification`
- 批量审核接口必须记录批次号/request_id，便于审计、失败补偿与重复请求去重

【6.8.1 报名职位】
POST /api/application/apply

请求参数：
{
  "job_id": 1,           // 职位ID（必填）
  "category_id": 1,      // 工种ID（必填）
  "level": "medium",     // 级别：senior/medium/junior（必填）
  "message": "有电工证，经验丰富",  // 留言（可选）
  "client_apply_id": "uuid-xxx-xxx"  // 客户端幂等键（必填，用于防止重复报名）【新增】
}

成功响应：
{
  "code": 200,
  "msg": "报名成功",
  "data": {
    "application_id": 1,
    "status": "applied"
  }
}

说明：
- `status = applied` 表示报名记录已创建成功；前端展示文案可显示为"待审核"，但不得将展示文案当作状态值回传或存储
- API参数设计说明：
  * API使用 `job_id + category_id + level` 组合定位，而非直接传 `job_level_id`，这是有意设计：
    - 用户只知道职位、工种、级别，不知道内部 `job_level_id`
    - 服务端根据三个参数自动关联到 `job_levels` 表获取 `job_level_id`
    - 这样做简化了前端调用，降低了前端与表结构的耦合
  * 数据库 `applications.job_level_id` 字段由服务端根据上述三参数自动填充

【6.8.2 取消报名】
PUT /api/application/cancel

请求参数：
{
  "id": 1  // 报名ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "已取消报名",
  "data": null
}

【6.8.3 我的报名列表】
GET /api/application/my-list

请求参数：
- page: 页码
- limit: 每页数量
- status: 状态筛选（可选：applied/accepted/rejected/cancelled）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "job_id": 1,
        "job_title": "电工招聘",
        "company_name": "XX建设集团",
        "work_location": "北京朝阳",
        "category_name": "电工",
        "level": "medium",
        "level_name": "中工",
        "salary": 350,
        "message": "有电工证，经验丰富",
        "status": "applied",
        "reject_reason": null,
        "can_reapply_time": null,
        "created_at": "2024-03-06 10:30:00"
      }
    ],
    "total": 10,
    "page": 1,
    "limit": 20,
    "total_pages": 1
  }
}

【6.8.4 报名管理（雇主端）】
GET /api/application/manage-list

请求参数：
- job_id: 职位ID（必填）
- category_id: 工种ID（可选）
- level: 级别（可选）
- status: 状态筛选（可选）
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
        "user_id": 10086,
        "user_name": "张三",
        "avatar_file": {
          "file_id": "f_20240306_avatar005",
          "disk": "public",
          "path": "avatars/2024/03/06/avatar005.jpg",
          "access": "public",
          "url": "https://...",
          "temporary_url": null,
          "mime_type": "image/jpeg",
          "size": 99840,
          "original_name": "avatar.jpg"
        },
        "age": 30,
        "work_years": 5,
        "category_name": "电工",
        "level": "medium",
        "level_name": "中工",
        "salary": 350,
        "message": "有电工证，经验丰富",
        "status": "applied",
        "created_at": "2024-03-06 10:30:00"
      }
    ],
    "total": 15,
    "page": 1,
    "limit": 20,
    "total_pages": 1
  }
}

【6.8.5 审核报名】
PUT /api/application/review

请求参数：
{
  "id": 1,
  "action": "approve",
  "employer_message": "明天上午9点来面试",  // 雇主留言（approve时可填）
  "reject_reason": "年龄不符合要求"          // 拒绝原因（reject时必填）
}

成功响应：
{
  "code": 200,
  "msg": "审核成功",
  "data": null
}

【6.8.6 批量审核】
PUT /api/application/batch-review

请求参数：
{
  "ids": [1, 2, 3],     // 报名ID数组（必填）
  "action": "approve",  // 操作：approve/reject（必填）
  "reject_reason": ""   // 批量拒绝原因（可选）
}

成功响应：
{
  "code": 200,
  "msg": "批量审核成功",
  "data": {
    "success_count": 3,
    "fail_count": 0
  }
}


6.9 消息相关接口
--------------------------------------------------

【Laravel 模块实现建议】
- 推荐 FormRequest：`SendPrivateMessageRequest`、`ReadConversationRequest`、`DeleteMessageRequest`、`ReportMessageRequest`
- 推荐 Service / Action：`ChatService`、`ConversationService`、`WsTokenService`、`MessageModerationService`
- 推荐 Policy：`ConversationPolicy`、`MessagePolicy`
- 推荐 Job / Event / Notification：`PrivateMessageSent`、`ConversationRead`、`SyncUnreadCountJob`、`PushWsEventJob`、`StoreModerationLogJob`
- WebSocket 实现建议：HTTP 接口负责签发 ws_token、拉取历史消息、补偿查询；实时发送由 Workerman/GatewayWorker 承担，业务逻辑通过 Laravel 容器服务复用
- 事务边界建议：私聊发送时需保证"消息表写入 + 会话更新 + 未读数更新 + 推送任务派发"一致完成
- 消息保留期建议：私聊与公共聊天室消息在入库时均由 MessageService / ChatService 按 `system_config.message_retention_days` 统一计算 `expire_at`；定时清理任务仅基于 `expire_at` 删除，不再重复推导保留期
- 幂等控制：客户端 `client_msg_id` 必须落库并建立唯一约束/去重逻辑；服务端必须结合 `client_msg_id`、`request_id` 或发送者+业务唯一键去重，防止弱网重发导致重复消息
- 消息发送限流：私聊发送、已读回执、撤回、删除等写操作必须纳入限流策略；至少按发送者ID、设备标识、会话ID 组合限流，命中限流时返回 HTTP 429 或 `system.error`，并携带 `code`、`msg`、`request_id`、`retry_after`
- 内容安全建议：违禁词、联系方式过滤、图片/文件审核应在 MessageModerationService 中统一编排，并保留审核日志

【消息子模块补充实现建议】
- 私聊消息建议拆分 `MessageSendAction`、`MessageReadAction`、`MessageRecallAction`、`MessageDeleteAction`
- 会话列表、未读数、最后消息摘要建议由 `ConversationService` 统一维护，并通过 Query Object / Scope 复用查询
- WebSocket 握手成功后，连接与用户绑定、会话订阅、在线状态广播应封装到独立网关服务，不直接写在控制器或单个事件处理器中
- 消息入库后建议派发领域事件，再由 Listener / Job 负责推送、通知中心写入、未读数增减、敏感词审计日志
- 文件/图片消息建议复用统一上传服务，消息发送阶段只提交 file_id；消息表保存 file_id 或受控的结构化文件元数据引用，消息输出统一转换为 File Resource 对象，不直接持久化完整 URL【更新】【P0】

【6.9.1 WebSocket连接配置】【新增】【P0】
GET /api/ws/config

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "ws_url": "ws://m.mokag.top/ws",
    "wss_url": "wss://m.mokag.top/ws",
    "ws_token": "ws_token_xxx",
    "expire_time": "2024-03-06 12:00:00",
    "heartbeat_interval": 30
  }
}

错误响应（Token无效）：
{
  "code": 3001,
  "msg": "Token无效或过期",
  "data": null
}

说明：
- ws_token 用于 WebSocket 连接鉴权，有效期较短（建议5分钟），过期后需重新调用本接口获取新 ws_token
- ws_token 必须绑定到当前登录态对应的 `device_session_id`，仅作为握手凭证使用，不能替代 HTTP Bearer Token，也不能脱离 `device_session_id` 单独长期代表登录状态【新增】【P0】
- 已过期、已失效、已被加入黑名单的登录 token 不允许继续换取新的 ws_token；若账号已注销、被封禁、被管理员强制下线，或对应 `device_session_id` 已失效，本接口必须返回认证失败【新增】【P0】
- 当用户退出登录、退出其他设备、修改密码、注销账号、账号被封禁、管理员强制下线时，相关 `device_session_id` 下已建立的 WebSocket 连接必须被主动断开，尚未使用的 ws_token 也必须立即失效【新增】【P0】
- 连接时必须使用 URL 参数方式携带 `ws_token`：
  * HTTP环境：ws://m.mokag.top/ws?ws_token=ws_token_xxx
  * HTTPS环境：wss://m.mokag.top/ws?ws_token=ws_token_xxx
- 对外统一走 Nginx 暴露的 `/ws` 入口，内部再反向代理到 Workerman / GatewayWorker 实际监听端口
- 服务端在连接建立时自动验证 token，不再支持首条消息鉴权
- 鉴权失败时服务端立即断开连接，返回 WebSocket 错误码 9001
- 此接口为 WebSocket 连接的前置步骤，必须先调用获取 ws_token 后才能建立 WebSocket 连接

【6.9.2 获取公共聊天室在线人数】【新增】
GET /api/chat/online-count

请求参数：无

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "online_count": 328,
    "updated_at": "2024-03-06 10:30:00"
  }
}

说明：
- HTTP轮询备用方案（WebSocket不可用时使用）
- 建议轮询间隔：30秒

【3-11. 私聊/公聊/通知接口】
--------------------------------------------------

【6.9.3 私聊消息列表】
GET /api/message/private-list

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
        "conversation_key": "10086_10087",
        "other_user_id": 10087,
        "other_user_name": "XX建设集团",
        "avatar_file": {
          "file_id": "f_20240306_avatar006",
          "disk": "public",
          "path": "avatars/2024/03/06/avatar006.jpg",
          "access": "public",
          "url": "https://...",
          "temporary_url": null,
          "mime_type": "image/jpeg",
          "size": 100352,
          "original_name": "avatar.jpg"
        },
        "first_job_id": 1,
        "first_job_title": "电工招聘",
        "last_message": "你好，明天可以来面试吗？",
        "last_message_time": "2024-03-06 10:30:00",
        "unread_count": 3
      }
    ],
    "total": 5,
    "page": 1,
    "limit": 20,
    "total_pages": 1
  }
}

【6.9.4 私聊消息详情】
GET /api/message/private-detail

请求参数：
- conversation_key: 会话标识（必填，如 10086_10087）
- page: 页码
- limit: 每页数量（默认50）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "conversation_key": "10086_10087",
    "list": [
      {
        "id": 1,
        "from_user_id": 10086,
        "to_user_id": 10087,
        "job_id": 1,
        "client_msg_id": "msg_123456",
        "content": "你好，我已报名",
        "message_type": "text",
        "is_read": 1,
        "created_at": "2024-03-06 10:00:00"
      }
    ],
    "total": 10,
    "page": 1,
    "limit": 50
  }
}

说明：
- 私聊按双方用户维度聚合为一个会话，职位仅作为消息上下文来源展示，不再通过 `other_user_id + job_id` 组合定位会话

【6.9.5 发送私聊消息】
POST /api/message/send-private

请求参数：
{
  "to_user_id": 10087,   // 接收者ID（必填）
  "job_id": 1,            // 关联职位ID（可选，建议首次发起聊天时传入）
  "client_msg_id": "msg_123456",  // 客户端幂等消息ID（必填）
  "content": "你好",      // 消息内容（必填）
  "message_type": "text"  // 消息类型：text/emoji/image/file（必填）
}

成功响应：
{
  "code": 200,
  "msg": "发送成功",
  "data": {
    "message_id": 1,
    "conversation_key": "10086_10087",
    "client_msg_id": "msg_123456",
    "created_at": "2024-03-06 10:35:00"
  }
}

【6.9.6 标记私聊消息已读】【新增】【P0】
PUT /api/message/read

请求头：Authorization: Bearer {token}

请求参数：
{
  "conversation_key": "10086_10087",  // 会话标识（必填）
  "message_ids": [1, 2, 3]              // 消息ID数组（可选，仅限当前会话；为空或不传表示标记该会话全部消息已读）
}

成功响应：
{
  "code": 200,
  "msg": "已标记为已读",
  "data": {
    "marked_count": 3,        // 标记已读的消息数量
    "unread_remaining": 0     // 该会话剩余未读数
  }
}

说明：
- 用户进入私聊界面时调用此接口标记当前会话消息已读
- 此接口仅用于单个 conversation_key 对应会话的已读同步
- 若传 message_ids 则只标记当前会话内指定消息，否则标记该会话全部消息
- 【空数组行为】空数组[]与不传message_ids参数等价，均表示标记该会话全部消息已读【新增】
- 【不存在消息ID处理】若message_ids中包含不存在或已删除的消息ID，服务端忽略这些ID，仅处理存在的消息ID【新增】
- 服务端更新 messages_private 表的 is_read 字段
- 同时更新 conversations 表的 unread 计数
- 通过 WebSocket 推送已读回执给消息发送方
- 若需一键标记当前用户全部私聊会话已读，请使用 PUT /api/message/read-all

【6.9.7 公共聊天室消息】
GET /api/message/public-list

请求参数：
- last_id: 最后一条消息ID（可选，用于增量获取）
- limit: 数量（默认50）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "user_id": 10086,
        "user_name": "用户A",
        "avatar_file": {
          "file_id": "f_20240306_avatar005",
          "disk": "public",
          "path": "avatars/2024/03/06/avatar005.jpg",
          "access": "public",
          "url": "https://...",
          "temporary_url": null,
          "mime_type": "image/jpeg",
          "size": 99840,
          "original_name": "avatar.jpg"
        },
        "content": "有没有人在北京找工？",
        "created_at": "2024-03-06 10:00:00"
      }
    ],
    "online_count": 328
  }
}

【6.9.8 发送公共消息】
POST /api/message/send-public

请求参数：
{
  "content": "有没有人在北京找工？"  // 消息内容（必填）
}

成功响应：
{
  "code": 200,
  "msg": "发送成功",
  "data": {
    "message_id": 1,
    "created_at": "2024-03-06 10:00:00"
  }
}

【6.9.9 系统通知列表】
GET /api/notifications

请求参数：
- type: 通知类型（可选：system/application/message/audit）
- is_read: 是否已读（可选：0未读/1已读）
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
        "title": "报名通知",
        "content": "您报名的《电工招聘》已通过",
        "notification_type": "application",
        "related_type": "application",
        "related_id": 1,
        "is_read": 0,
        "created_at": "2024-03-06 10:30:00"
      }
    ],
    "total": 20,
    "page": 1,
    "limit": 20,
    "unread_count": 5
  }
}

【6.9.10 标记通知已读】
PUT /api/notifications/read

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
  "msg": "已标记为已读",
  "data": null
}
说明：
- 传 `ids` 表示批量标记指定通知为已读
- 传 `all=true` 表示将当前用户全部未读通知标记为已读
⚠️ 参数互斥：ids和all不能同时传递，只能选择其中一种方式


【6.9.11 撤回私聊消息】【新增】
POST /api/message/private/recall

请求头：Authorization: Bearer {token}

请求参数：
{
  "id": 1  // 消息ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "消息已撤回",
  "data": null
}

错误响应：
{
  "code": 9008,
  "msg": "消息撤回超时（超过2分钟）",
  "data": null
}

说明：
- 仅发送方可以撤回自己发送的消息
- 消息发送超过2分钟后不可撤回（返回错误码9008）
- 撤回后设置 is_recalled=1 并推送 message.recalled 事件
- 撤回后双方都看到"[消息已撤回]"，内容不可见


【6.9.12 删除私聊消息】【新增】
DELETE /api/message/private/delete

请求头：Authorization: Bearer {token}

请求参数：
{
  "id": 1  // 消息ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "消息已删除",
  "data": null
}

说明：
- 【软删除】删除仅对本方有效（is_deleted_sender/receiver=1），对方仍可查看原文
- 【物理删除】当发送方和接收方都已软删除同一消息时（is_deleted_both=1），消息从数据库物理删除，不再保留任何痕迹

6.10 其他功能接口
--------------------------------------------------

【6.10.1 工种列表】
GET /api/category/list

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": [
    {
      "id": 1,
      "name": "电工",
      "icon": "electric",
      "sort_order": 1
    },
    {
      "id": 2,
      "name": "焊工",
      "icon": "welding",
      "sort_order": 2
    }
  ]
}

【6.10.2 申请新工种】
POST /api/category/apply

请求参数：
{
  "name": "数控车工",            // 工种名称（必填）
  "reason": "市场需求大，建议添加"  // 申请理由（必填）
}

成功响应：
{
  "code": 200,
  "msg": "申请已提交，等待管理员审核",
  "data": null
}

【6.10.3 收藏职位】
POST /api/favorite/add

请求参数：
{
  "favorite_type": "job",  // 收藏类型：job职位/talent人才
  "target_id": 1            // 目标ID（必填）
}

成功响应：
{
  "code": 200,
  "msg": "收藏成功",
  "data": null
}

【6.10.4 取消收藏】
DELETE /api/favorite/remove

请求参数：
{
  "favorite_type": "job",
  "target_id": 1
}

成功响应：
{
  "code": 200,
  "msg": "已取消收藏",
  "data": null
}

【6.10.5 我的收藏列表】
GET /api/favorite/list

请求参数：
- favorite_type: 收藏类型
- status: all/active/closed（可选，默认all）
- page: 页码
- limit: 每页数量

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      // 职位信息或人才信息
    ],
    "total": 10,
    "page": 1,
    "limit": 20,
    "total_pages": 1,
    "stats": {
      "all": 10,
      "active": 7,
      "closed": 3
    }
  }
}

说明：
- `status=active` 表示仅返回仍处于招聘中的收藏职位
- 此处 `status=closed` 为收藏页前端筛选分组口径，表示返回非 active 职位集合（含 paused、closed、expired、deleted 等），并非数据库真实状态值仅等于 `closed`；如需按数据库真实状态精确筛选，应传具体状态枚举【更新】【P0】
- `total_pages` 表示总页数，与其他列表接口保持一致；前端可通过 page < total_pages 判断是否还有更多数据
- `stats` 用于收藏页顶部状态筛选统计展示

【6.10.6 举报】
POST /api/report/submit

请求参数：
{
  "report_type": "job",         // 举报类型：job/comment/user/company（必填）
  "target_id": 1,                // 被举报对象ID（必填）
  "reason": "虚假职位",          // 举报原因（必填）
  "description": "工资与实际不符",  // 详细描述（必填）
  "evidence_file_ids": ["f_20240306_evidence001"]     // 证据文件ID列表（可选，文件需先通过上传接口获取）
}

成功响应：
{
  "code": 200,
  "msg": "举报已提交",
  "data": null
}

【6.10.7 公告列表】
GET /api/announcement/list

请求参数：
- page: 页码
- limit: 每页数量
- announcement_type: 公告类型（system系统/activity活动/notice通知，可选）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "title": "系统升级通知",
        "content_summary": "系统将于...",
        "announcement_type": "system",
        "is_top": 1,
        "start_time": "2024-03-06 00:00:00",
        "end_time": "2024-03-10 23:59:59",
        "created_at": "2024-03-05 18:00:00"
      }
    ],
    "total": 5,
    "page": 1,
    "limit": 20
  }
}

说明：
- 仅返回 is_enabled=1 且当前时间位于 start_time/end_time 有效期内的公告
- 列表接口中的 `content_summary` 用于列表摘要展示；进入详情页后应调用 `GET /api/announcement/{id}` 获取完整正文
- 该接口返回 announcements 资源，不返回系统广播统计字段

【6.10.7.1 公告详情】
GET /api/announcement/{id}

请求参数：
- id: 公告ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "title": "系统升级通知",
    "content": "系统将于2024-03-06 22:00至24:00进行升级维护，期间部分功能可能短暂不可用，请提前做好安排。",
    "announcement_type": "system",
    "is_top": 1,
    "start_time": "2024-03-06 00:00:00",
    "end_time": "2024-03-10 23:59:59",
    "admin_name": "平台运营团队",
    "created_at": "2024-03-05 18:00:00"
  }
}

说明：
- 公告详情页统一通过该接口单独加载完整内容
- 仅返回 is_enabled=1 且当前时间位于 start_time/end_time 有效期内的公告详情

【6.10.8 文件上传】
POST /api/upload/image

请求参数：
- Content-Type: multipart/form-data
- file: 图片文件（支持 jpg/png/webp，大小限制由系统配置 max_upload_size 决定，默认最大 5MB）
- type: 上传用途类型（可选，枚举：avatar / job_image / cert / resume，用于分目录存储和权限校验）

成功响应：
{
  "code": 200,
  "msg": "上传成功",
  "data": {
    "file_id": "f_20240306_abc123",
    "disk": "public",
    "path": "uploads/202403/abc123.jpg",
    "access": "public",
    "url": "https://example.com/storage/uploads/202403/abc123.jpg",
    "temporary_url": null,
    "mime_type": "image/jpeg",
    "size": 2048576,
    "original_name": "avatar.jpg"
  }
}

说明：上传接口返回结构即标准 File Resource 对象；若上传结果为 private 文件，则 `url` 返回 null，改为返回 `temporary_url`。前端应直接消费该对象，不得自行拼接访问地址。【新增】【P0】

统一文件引用规范（重要）：
- 业务表（jobs.images、users.avatar_file_id、companies.business_license_file_id 等）默认统一存储 file_id，不存储完整 URL；历史字段若暂存相对 path，仅允许作为内部过渡字段使用【更新】【P0】
- 原因：完整 URL 包含域名/CDN前缀，域名变更、CDN切换、私有存储迁移时会批量失效
- 展示层由服务端统一将 file_id / path 转换为 File Resource 对象（Resource 层或 Presenter 层处理），前端不得自行拼接路径【新增】【P0】
- File Resource 对象字段统一为：file_id、disk、path、access、url、temporary_url、mime_type、size、original_name【新增】【P0】
- 私有文件（资质证件等）：disk=private，access=private，返回 `temporary_url`，有效期建议15分钟
- 公开文件（职位图片、头像等）：disk=public，access=public，返回 `url`，可直接访问或走 CDN
- private 文件的 `url` 应返回 null；public 文件的 `temporary_url` 默认返回 null，除非业务明确要求临时签名公开链路【新增】【P1】

Laravel 实现建议：
- 上传后在 files 表记录文件元数据（file_id、path、disk、mime_type、size、uploader_id、type、created_at）
- 业务接口提交时只传 file_id，由 Service 层校验 file_id 归属与合法性
- 资源输出使用 Storage::url($path) 或 Storage::temporaryUrl($path, $expiry) 统一生成访问链接
- 文件相关 Resource 建议封装统一 `FileResource` / `FileResourceCollection`，避免头像、资质材料、简历附件、消息图片在多个接口中重复定义返回结构【新增】【P1】
- 职位图片、头像、简历附件、企业资质、消息文件等场景均遵循同一文件模型：业务层传 file_id，输出层返 File Resource 对象【新增】【P0】
- 对外返回命名统一使用 `*_file` / `*_files` 表达文件语义字段，如 `avatar_file`、`cover_image_file`、`site_logo_file`；页面跳转链接、下载链接等纯链接字段继续使用 string，如 `link_url`、`android_download_url`、`ios_download_url`【新增】【P0】

【6.10.9 黑名单公示】
GET /api/blacklist/public

请求参数：
- page: 页码
- limit: 每页数量
- type: 类型筛选（user/company/labor/all）
- keyword: 搜索关键词

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "display_name": "XX骗子公司",
        "blacklist_type": "company",
        "public_reason": "虚假招聘",
        "public_time": "2024-03-01 10:00:00",
        "expire_at": "2025-03-01 10:00:00"
      },
      {
        "id": 2,
        "display_name": "张三",
        "blacklist_type": "user",
        "public_reason": "骚扰行为",
        "public_time": "2024-03-02 15:30:00",
        "expire_at": null
      }
    ],
    "total": 10,
    "page": 1,
    "limit": 20
  }
}

说明：
- 用户端黑名单公示统一展示 `public_reason`，不直接返回后台内部处理字段 `reason`
- 页面仅支持类型筛选（all/company/labor/user）与关键词搜索，不提供统计卡片和额外违规类型筛选
- **申诉说明**：被公示的企业/中介/用户可通过平台客服渠道提交申诉，管理员审核后决定是否撤销公示。【新增】

【6.10.10 检查黑名单状态】【新增】
GET /api/blacklist/check

请求参数：
- user_id: 用户ID（可选）
- company_id: 企业ID（可选）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "is_blacklisted": false,
    "blacklist_info": null  // 如果在黑名单中，返回相关信息
  }
}

【6.10.11 浏览历史列表】【新增】
GET /api/user/browse-history

请求头：Authorization: Bearer {token}

请求参数：
- page: 页码
- limit: 每页数量（默认20）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "job_id": 1,
        "job_title": "电工招聘",
        "company_name": "XX建设集团",
        "work_location": "北京朝阳",
        "viewed_at": "2024-03-06 10:30:00",
        "job_status": "active"
      }
    ],
    "total": 30,
    "page": 1,
    "limit": 20
  }
}

【6.10.12 清空浏览历史】【新增】
DELETE /api/user/browse-history/clear

请求头：Authorization: Bearer {token}

成功响应：
{
  "code": 200,
  "msg": "浏览历史已清空",
  "data": null
}

【6.10.13 获取系统配置】【新增】【P1】
GET /api/system/config

请求参数：无（公开接口，无需登录）

Laravel 实现说明：
- 优先从 `system_config` 表读取可运营动态调整项，再与 `config/system.php` 中的默认值合并输出
- 公开接口默认返回字段名应与 `system_config.config_key` 保持一致；但文件语义配置项可在输出层统一转换为 `*_file` 形式（如 `site_logo` -> `site_logo_file`），以保持对外文件返回结构一致，仅返回允许公开的配置子集，不返回仅后台使用的安全类配置
- 建议通过缓存键如 `system:public_config` 做 5~30 分钟缓存，后台更新配置后主动清理缓存
- 对下载地址、客服联系方式、开关项等公共配置，可通过 Resource 层统一做字段过滤与类型转换；其中图片类配置（如 `site_logo`）统一转换为 File Resource 并按 `site_logo_file` 输出
- 公开接口建议返回的配置子集包括：basic 中的公共项、function 中对前端可见的功能开关、business 中影响前端交互的限制项；`token_expire_days`、`login_fail_max`、`login_lock_minutes`、`contact_filter_pattern`、`operation_log_retention_days` 等安全配置不对外返回

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "site_name": "自动化装备招聘系统",
    "site_logo_file": {
      "file_id": "f_20240306_logo001",
      "disk": "public",
      "path": "system/2024/03/logo001.png",
      "access": "public",
      "url": "https://...",
      "temporary_url": null,
      "mime_type": "image/png",
      "size": 24576,
      "original_name": "logo.png"
    },
    "contact_email": "support@mokag.top",
    "contact_phone": "400-123-4567",
    "maintenance_mode": false,
    "maintenance_message": null,
    "android_download_url": "https://...",
    "ios_download_url": "https://...",
    "real_name_auth": true,
    "chat_enabled": true,
    "public_chat_enabled": true,
    "application_enabled": true,
    "comment_enabled": true,
    "captcha_enabled": true,
    "contact_filter_enabled": false,
    "public_chat_interval": 10,
    "reapply_limit_hours": 72,
    "refresh_limit_per_day": 5,
    "max_upload_size": 5,
    "account_delete_days": 7
  }
}

说明：
- 此接口用于获取前端所需的系统公共配置
- 无需登录即可访问
- 建议在应用启动时调用获取配置
- 配置项根据实际需求返回，可能动态调整

【6.10.14 获取帮助中心首页数据】【新增】【P1】
GET /api/help/home

说明：用于帮助中心首页加载分类导航、热门问题与推荐文章。

请求参数：无

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "categories": [],
    "hot_articles": [],
    "recommended_articles": []
  }
}

【6.10.15 帮助分类列表】【新增】【P1】
GET /api/help/categories

请求参数：无

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": [
    {"id": 1, "name": "新手指南", "icon": "guide", "sort_order": 1}
  ]
}

【6.10.16 帮助文章列表】【新增】【P1】
GET /api/help/articles

请求参数：
- category_id: 分类ID（可选）
- sort: 排序（latest最新/hot最热，可选）
- page: 页码
- limit: 每页数量

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [],
    "total": 0,
    "page": 1,
    "limit": 20
  }
}

【6.10.17 帮助文章详情】【新增】【P1】
GET /api/help/articles/{id}

请求参数：
- id: 文章ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "title": "如何发布职位",
    "content": "...",
    "category": {"id": 1, "name": "新手指南"},
    "view_count": 120,
    "helpful_count": 56,
    "not_helpful_count": 3,
    "related_articles": []
  }
}

【6.10.18 帮助中心搜索】【新增】【P1】
GET /api/help/search

请求参数：
- keyword: 搜索关键词（必填）
- page: 页码
- limit: 每页数量

Laravel 实现说明：
- 搜索前先按 `help_search_synonyms` 对关键词做归一化处理
- 每次搜索都写入 `help_search_logs`，用于热门词、无结果词和趋势统计
- 搜索结果可基于标题、正文、tags、keywords 综合排序

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [],
    "total": 0,
    "suggest": [],
    "related": []
  }
}

【6.10.19 热门问题列表】【新增】【P1】
GET /api/help/hot

请求参数：
- limit: 数量（默认10）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": []
  }
}

【6.10.20 提交帮助文章反馈】【新增】【P1】
POST /api/help/articles/{id}/feedback

请求头：Authorization: Bearer {token}

请求参数：
{
  "is_helpful": 0,
  "reason": "内容不完整"
}

成功响应：
{
  "code": 200,
  "msg": "反馈已提交",
  "data": null
}

6.11 管理员后台接口【新增】
--------------------------------------------------

【Laravel 模块实现建议】
- 推荐 FormRequest：`AdminLoginRequest`、`BlacklistUserRequest`、`AuditCompanyRequest`、`HandleReportRequest`、`UpdateSystemConfigRequest`、`CreateAnnouncementRequest`、`SendBroadcastRequest`
- 推荐 Service / Action：`AdminAuthService`、`BlacklistService`、`ReportHandleService`、`AnnouncementService`、`BroadcastService`、`SystemConfigService`、`SubAdminPermissionService`
- 推荐 Gate / 中间件：`auth.jwt:admin`、`*:manage`、`*:view`、`*:audit`、`super-admin-only`、`operation.lock`（防并发处理）【更新】【P1】
- 推荐审计能力：所有写操作通过统一 `OperationLogService` 记录请求人、目标资源、前后变更摘要、IP、设备、请求ID
- 事务边界建议：审核、拉黑、解封、删除、权限变更、配置发布等接口应以事务保证"主业务更新 + 审计日志 + 通知/待办刷新"一致
- 可异步化任务：批量导出、系统广播下发、统计报表刷新、敏感内容复检、黑名单公示缓存刷新
- 资源授权建议：管理员不仅要校验模块权限，还要校验资源级权限和数据范围，例如只允许某些子管理员审核指定模块

所有管理员接口均需：
- 请求头：Authorization: Bearer {admin_token}
- 服务端验证管理员身份及对应模块权限
- 操作自动写入 operation_logs 表

Laravel 实现说明：
- 管理端建议独立 `auth.jwt:admin` 中间件或独立 guard，避免与普通用户 token 混用
- 模块级权限使用 Policy / Gate / 自定义权限中间件组合实现，格式与 permission_code 一致，例如 `job:audit`、`user:view`、`help:article:manage`、`help:stats:view`【更新】【P1】
- 涉及查看敏感资质、解密身份证、拉黑/解封、删除内容、审核企业等操作时，除接口权限外还应追加资源级授权校验与审计日志

【管理员子模块补充实现建议】
- 用户管理、企业审核、举报处理、黑名单、公示、公告、违禁词、系统配置、子管理员建议分别拆成独立 Service，避免 `AdminController` 过度膨胀
- 涉及"只能处理一次"的资源（举报单、企业审核单、置顶申请）建议增加乐观锁/处理中状态/请求锁，避免多个管理员并发处理
- 敏感查看类接口（如查看完整身份证、营业执照原图）建议单独设计授权点与查看日志，不与普通详情接口混用
- 子管理员权限变更后应主动刷新权限缓存，并使其旧 token 在下次请求时重新拉取权限或直接失效
- 后台导出类接口建议返回导出任务 ID，由队列异步生成文件，再通过下载接口获取结果，避免长请求阻塞

【6.11.1 管理员登录】
POST /api/admin/login

请求参数：
{
  "username": "admin",          // 用户名/邮箱/手机号
  "password": "123456",
  "captcha_key": "abc123",
  "captcha": "a7Kd"
}

成功响应：
{
  "code": 200,
  "msg": "登录成功",
  "data": {
    "admin_id": 1,
    "username": "admin",
    "role": "super",
    "admin_user_permissions": [
      { "permission_code": "user:view", "name": "用户查看" },
      { "permission_code": "job:audit", "name": "职位审核" },
      { "permission_code": "help:article:manage", "name": "帮助文章管理" },
      { "permission_code": "help:stats:view", "name": "帮助搜索统计查看" }
    ],
    "token": "eyJ...",
    "token_expire": "2024-03-06 12:00:00"
  }
}

错误响应（账号锁定）：
{
  "code": 3006,
  "msg": "登录失败次数过多，账号已锁定",
  "data": {
    "locked_until": "2024-03-06 11:00:00",
    "lock_minutes_left": 28
  }
}

说明：
- 管理员登录流程需校验图形验证码，与 5.29 管理员登录流程保持一致
- 管理员登录失败次数与锁定时长建议读取 system_config 中的 `login_fail_max`、`login_lock_minutes`
- 登录成功后返回管理员 token、权限列表和过期时间；普通用户 token 不可用于后台接口

【6.11.1.1 管理员登出】
POST /api/admin/logout

请求头：
Authorization: Bearer {token}

说明：
- 使用 admin guard 验证 Token 有效性
- 将当前使用的管理员 Token 加入 Redis 黑名单，使其立即失效
- 清除该管理员在服务端的权限缓存数据
- 写入一条登出日志到 admin_login_logs 表，记录登出时间与 IP

成功响应：
{
  "code": 200,
  "msg": "已安全退出",
  "data": null
}

【6.11.2 数据统计概览】
GET /api/admin/stat/overview

请求参数：
- days: 统计天数（7/30，默认7）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "user_total": 12580,
    "user_today": 32,
    "job_total": 5680,
    "job_today": 18,
    "apply_total": 28900,
    "apply_today": 156,
    "pending_jobs": 8,
    "pending_companies": 5,
    "pending_reports": 12,
    "pending_categories": 3,
    "daily_users": [...],
    "daily_jobs": [...]
  }
}

【6.11.3 用户列表】
GET /api/admin/user/list

请求参数：
- role: 角色筛选（可选：seeker/employer）
- is_blacklist: 黑名单筛选（可选：0/1）
- keyword: 手机号/昵称/ID（可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 10086,
        "nickname": "张三",
        "phone": "138****8888",
        "role": "seeker",
        "is_certified": 2,
        "is_blacklist": 0,
        "is_deleted": 0,
        "created_at": "2024-01-15",
        "last_login_time": "2024-03-06 10:00:00"
      }
    ],
    "total": 12580,
    "page": 1,
    "limit": 20
  }
}

【6.11.4 用户详情】
GET /api/admin/user/detail

请求参数：
- id: 用户ID（必填）

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 10086,
    "nickname": "张三",
    "phone": "13800138000",
    "email": "zhangsan@example.com",
    "role": "seeker",
    "avatar_file": {
      "file_id": "f_20240306_avatar001",
      "disk": "public",
      "path": "avatars/2024/03/06/avatar001.jpg",
      "access": "public",
      "url": "https://...",
      "temporary_url": null,
      "mime_type": "image/jpeg",
      "size": 102400,
      "original_name": "avatar.jpg"
    },
    "is_certified": 2,
    "is_blacklist": 0,
    "is_deleted": 0,
    "id_card": "310***********1234",  // 脱敏显示
    "device_id": "device_xxx",
    "ip_address": "192.168.1.1",
    "last_login_time": "2024-03-06 10:00:00",
    "last_login_ip": "192.168.1.100",
    "login_fail_count": 0,
    "created_at": "2024-01-15 08:30:00",
    "resume": {  // 求职者简历信息
      "real_name": "张三",
      "gender": "男",
      "age": 30,
      "work_years": 5,
      "skills": ["电工", "焊工"],
      "self_introduction": "..."
    },
    "company": {  // 雇主企业信息
      "id": 1,
      "company_name": "XX建设集团",
      "company_type": "enterprise",
      "audit_status": 1
    },
    "stats": {
      "job_count": 10,  // 发布职位数
      "application_count": 5,  // 报名数
      "message_count": 128  // 消息数
    }
  }
}

【6.11.4.1 管理员修改用户密码】
POST /api/admin/user/reset-password

说明：客服验证用户身份后，通过管理后台直接修改用户密码

请求头：Authorization: Bearer {admin_token}

请求参数：
{
  "user_id": 10086,
  "new_password": "NewPass@123",      // 管理员直接设置的新密码
  "verify_remark": "已电话核实身份"  // 客服验证备注
}

成功响应：
{
  "code": 200,
  "msg": "密码修改成功",
  "data": null
}

说明：
- 权限要求：`user:edit`
- 该接口仅用于客服/管理员核验身份后的人工改密链路；邮箱找回走 `POST /api/user/reset-password-email`，两条链路分开处理
- 属于高危后台写操作，`risk_level` 至少为 `high`
- `verify_remark` 必填，并作为本次人工改密的核验说明与审计原因摘要
- 前端必须展示高危操作确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 管理员直接提交 `new_password` 作为用户新密码，服务端按密码存储规则加密保存，接口响应、审计日志与快照中都不得返回或记录明文密码
- 修改成功后应使用户现有登录 token / 会话失效，用户需使用新密码重新登录
- 如命中安全策略，可同时设置 `must_change_password = 1`，要求用户首次登录后再次修改密码
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=user_id`、`target_type=user`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；`before_snapshot` / `after_snapshot` 仅记录密码状态、会话失效状态、强制改密标记等脱敏摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须拒绝执行并写入失败/拒绝审计日志
- 建议从【用户详情页】或客服核验场景进入，完成身份核验后再执行

【6.11.17 新增黑名单】
POST /api/admin/blacklist/add

请求参数：
{
  "blacklist_type": "user",   // user用户/company企业/labor中介
  "target_id": 10086,
  "reason": "多次虚假发布",
  "evidence_file_ids": ["f_20240306_evidence002"],
  "is_public": 1,
  "public_level": "all",       // all全部可见/login_only仅登录可见
  "public_reason": "虚假招聘",
  "expire_at": null   // null为永久，填日期为临时
}

说明：
- 属于高危后台写操作（封禁），`risk_level` 至少为 `high`
- `reason` 必填；当 `is_public = 1` 时，`public_level` 与 `public_reason` 必填
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- blacklist_type=user 时，target_id 表示 user_id
- blacklist_type=company/labor 时，target_id 表示 company_id；服务端需反查并写入其主账号 user_id
- 企业/中介主体被加入黑名单时，需连带更新其主账号 `users.is_blacklist = 1`
- 支持在拉黑时同步写入公示配置
- 当 is_public = 0 时，public_level / public_reason 可为空，服务端应清空对应公示字段
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id`、`target_type`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中 `before_snapshot` / `after_snapshot` 至少包含黑名单状态、有效期、公示状态及关联主账号状态等摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "已加入黑名单",
  "data": {
    "blacklist_id": 1,
    "blacklist_type": "user",
    "target_id": 10086,
    "linked_user_id": 10086
  }
}

【6.11.18 移除黑名单】
POST /api/admin/blacklist/lift

请求参数：
{
  "blacklist_id": 1,
  "lift_reason": "已整改"
}

说明：
- 属于高危后台写操作（解封），`risk_level` 至少为 `high`
- `lift_reason` 必填，并作为解除黑名单的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 若黑名单记录为 company/labor，且存在关联主账号 user_id，则解除时需同步恢复该账号的黑名单状态
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=blacklist_id`、`target_type=blacklist`、`request_id`、`result`、`risk_level`、`reason=lift_reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含黑名单状态、关联对象状态与解除时间摘要
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "黑名单已解除",
  "data": null
}

【6.11.19 解封用户】
DELETE /api/admin/user/delete

请求参数：
{
  "user_id": 10086,
  "reason": "严重违规"
}

说明：
- 属于高危后台写操作（删除用户），`risk_level` 至少为 `critical`
- `reason` 必填，并作为删除用户的审计原因
- 前端必须展示二次确认信息；后端需校验统一的高危操作二次确认状态，未完成确认时拒绝执行
- 删除前需校验数据范围、关联业务约束与不可逆影响，必要时转为逻辑删除或冻结流程，避免误删核心数据
- 必须写入 `operation_logs`，记录 `operator_id`、`target_id=user_id`、`target_type=user`、`request_id`、`result`、`risk_level`、`reason`、`before_snapshot`、`after_snapshot`；其中快照至少包含账号状态、角色、认证状态、关联主体摘要与删除结果
- 成功、失败、拒绝三类结果都必须留痕；如命中权限不足、数据范围限制或二次确认未通过，必须返回 403 并写入失败/拒绝审计日志

成功响应：
{
  "code": 200,
  "msg": "用户已删除",
  "data": null
}

【6.11.5 企业认证列表】
GET /api/admin/company-certifications

请求参数：
- audit_status: 审核状态（0/1/2，默认0）
- keyword: 搜索关键词（企业名称/联系人/手机号，可选）
- page, limit

成功响应：
{
  "code": 200,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "user_id": 10087,
        "company_name": "XX建设集团",
        "company_type": "enterprise",
        "contact_person": "张经理",
        "contact_phone": "13800138000",
        "applicant_name": "张经理",
        "applicant_phone": "13800138000",
        "material_preview_type": "business_license",
        "material_preview_url": "https://.../license-thumb.jpg?sign=xxx&expires=1709730000",  // 签名临时URL（15分钟有效）
        "audit_status": 0,
        "created_at": "2024-03-06 09:00:00"
      }
    ],
    "total": 5,
    "page": 1,
    "limit": 20,
    "stats": {
      "pending": 5,
      "approved": 120,
      "rejected": 8
    }
  }
}

说明：
- 列表接口同时返回顶部统计卡片所需的 stats 数据
- material_preview_type / material_preview_url 用于列表页材料缩略图展示
- 企业/劳务默认返回营业执照缩略图，个人雇主默认返回身份证正面缩略图

