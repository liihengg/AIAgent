## Context

当前项目是一个微信小程序抽奖平台，前端为微信小程序（CommonJS），后端为 ASP.NET Core 10 + EF Core + SQLite。后端已有 JWT 认证、微信支付、会员体系等模块，使用 `Result<T>` 统一返回格式，服务层通过 DI 注入，直接使用 `AppDbContext`。时间统一使用毫秒 Unix 时间戳（`Clock.Now`）。项目无 `wwwroot` 目录，未启用静态文件服务。前端 `request.js` 仅封装 JSON 请求，无文件上传能力。

## Goals / Non-Goals

**Goals:**
- 实现本地文件存储的上传/管理/访问能力
- 接入微信 `mediaCheckAsync 2.0` 异步内容安全审核
- 前端 `image-picker` 自定义组件，支持图库选择和上传
- CDN 友好的相对路径设计，未来可无缝迁移到云存储
- 乐观审核策略：上传后立即可用，审核失败后下线

**Non-Goals:**
- 商品图片上传（Admin 管理的会员商品，暂不改造）
- 审核失败后通知用户（标记为 todo，后续迭代）
- 人工审核流程（`NeedsReview` 状态暂不处理）
- 接入云存储/OSS（本地存储为主，架构上预留迁移能力）

## Decisions

### D1: 本地磁盘存储 + 后端拼接完整 URL

**选择**: 文件存储到 `wwwroot/uploads/{type}/{yyyyMM}/{guid}.{ext}`，数据库记录相对路径（如 `/uploads/prizes/202608/abc.jpg`），API 返回时通过后端 `BuildPublicUrl()` 方法拼接为完整 URL（如 `http://localhost:5297/uploads/prizes/202608/abc.jpg`）。完整 URL 的域名部分由 `appsettings.json` 的 `FileUpload:PublicBaseUrl` 配置。

**理由**: 项目当前使用 SQLite 单文件数据库 + 自托管后端，属于单机小规模架构。ASP.NET Core `UseStaticFiles` 零依赖即可提供文件访问。DB 存相对路径保证 CDN 迁移时无需改数据库；API 返回完整 URL 使前端无需任何路径拼接逻辑，`<image src>` 直接可用。迁移 CDN 时只需修改 `PublicBaseUrl` 一处配置，前端零改动。

**替代方案**: 前端用 `FILE_BASE_URL` 常量拼接。排除原因：每个展示图片的地方都要调用拼接函数，容易遗漏。最终选择后端统一拼接，前端直接用。

### D1a: wwwroot 目录需启动前存在

**选择**: 在 `wwwroot/` 目录下放置 `.gitkeep` 文件，确保目录被 Git 跟踪且在应用启动前就存在。

**理由**: `UseStaticFiles` 初始化 `PhysicalFileProvider` 时会检查 `Directory.Exists(root)`，如果 `wwwroot` 不存在则 `ExistentDirectory=false`，之后所有静态文件请求永久返回 404，即使文件已写入磁盘。上传代码运行时创建的 `uploads/` 子目录不受此限制（`PhysicalFileProvider` 只检查根目录是否存在）。

### D2: FileRecord 表独立记录文件元数据

**选择**: 新增 `UploadedFile` 表，记录文件 ID、上传者、路径、大小、类型、审核状态等。现有的 `User.AvatarUrl`、`LotteryPrize.ImageUrl`、`Product.ImageUrl` 继续存储 URL 字符串，不变。

**理由**: 独立表提供孤儿文件清理能力（知道哪个文件还在被引用）、CDN 迁移时可批量更新路径、完整的审计追踪和用量统计。不修改现有表结构避免迁移风险。

**替代方案**: 只存 URL 不建表。排除原因：无法追踪文件归属、无法清理孤儿文件、无法迁移。

### D3: 乐观审核策略 — 先用后审

**选择**: 文件上传后立即可用（审核状态 `Checking`），审核结果返回后再更新状态。审核结果为 `risky` 时删除文件。

**理由**: `mediaCheckAsync 2.0` 是异步接口，结果最多 30 分钟才返回。如果等审核通过才可用，用户体验不可接受。抽奖平台用户都是微信登录用户（openid 可追溯），图片仅在小程序内展示，不是公开传播，窗口期风险可控。

**替代方案**: 先审后用（Pending → Approved 才可用）。排除原因：30 分钟等待对用户体验致命。

### D4: 审核失败后前端 binderror 兜底

**选择**: 审核失败后删除文件 + 标记数据库记录。不主动更新引用方（`User.AvatarUrl`、`LotteryPrize.ImageUrl` 等）的 URL。前端所有 `<image>` 组件绑定 `binderror`，加载失败时替换为默认图片。

**理由**: URL 被复制到多个表的字段中（非外键引用），追踪所有引用方需跨表扫描，复杂且易漏。`binderror` 是微信小程序处理图片加载失败的标准方式，覆盖所有异常场景（审核拒绝、网络超时、文件损坏）。

**替代方案**: 后端追踪引用并批量更新为默认图 URL。排除原因：跨表查询复杂、漏更新风险、且只能覆盖审核拒绝场景。

### D5: 审核超时重试 — 最多 3 次后乐观放行

**选择**: 后台服务（`FileAuditBackgroundService`）每 5 分钟扫描 `Checking` 状态超过 30 分钟的记录，重新调用 `mediaCheckAsync`。最多重试 3 次，之后标记为 `Approved`。

**理由**: 审核超时大概率是微信侧问题而非内容问题，3 次重试已足够。乐观放行避免文件永久卡在 `Checking` 状态。

### D6: access_token 缓存 — 稳定版接口

**选择**: 使用微信稳定版 access_token 接口（`https://api.weixin.qq.com/cgi-bin/stable_token`），内存缓存，过期前 5 分钟刷新。

**理由**: 稳定版接口不会因并发请求产生冲突，适合服务端调用。有效期 2 小时，缓存避免频繁调用。access_token 获取失败不影响文件上传，审核延迟到后台重试。

**替代方案**: 使用标准 `client_credential` 接口。排除原因：并发获取可能冲突，需额外处理锁。

### D7: Mock 模式复用 WeChatPay:MockEnabled

**选择**: 复用现有 `WeChatPay:MockEnabled` 配置项控制审核行为。`true` 时跳过 `mediaCheckAsync` 调用，直接标记文件为 `Approved`；`false` 时正常走审核流程。

**理由**: 开发环境没有公网回调地址，`mediaCheckAsync` 无法收到结果。复用已有 Mock 配置避免新增配置项，且语义一致 — Mock 模式下跳过所有微信外部依赖。

### D8: 审核调用同步 await（不使用 Task.Run）

**选择**: 上传时 `await TriggerMediaCheckAsync()`，不使用 fire-and-forget 的 `Task.Run`。

**理由**: `FileService` 是 Scoped 生命周期，绑定到 HTTP 请求。`Task.Run` 中的异步任务在请求结束后 `_db` 可能被 dispose，导致 `ObjectDisposedException`。`mediaCheckAsync` 本身是异步 API（调用后立即返回 `trace_id`），`await` 仅多等几百毫秒（获取 access_token + 调 API），用户无感知。

**替代方案**: 上传时只写 DB（Pending 状态），由后台服务扫描 Pending 记录调用审核。排除原因：需缩短后台扫描间隔到 30 秒以内，否则用户上传后审核延迟明显，且增加后台服务复杂度。

### D9: 保存时上传（避免孤儿文件）

**选择**: 头像在用户点击保存时上传，而非 `chooseAvatar` 时立即上传。

**理由**: 如果用户选择头像后取消编辑，立即上传会产生孤儿文件。保存时上传确保只有用户确认的文件才被存储。

### D10: 目录结构按类型 + 年月分区

**选择**: `uploads/{avatars|prizes}/{yyyyMM}/{guid}.{ext}`

**理由**: 按类型分目录便于管理（可独立设置清理策略）。按年月分区避免单目录文件过多影响文件系统性能。

### D11: 文件命名使用 GUID

**选择**: `{GUID}.{ext}`

**理由**: 全局唯一无碰撞，不泄露用户信息，防止路径穿越攻击。原始文件名记录在数据库中供审计。

### D12: wx.uploadFile Promise 封装

**选择**: 在 `request.js` 中新增 `upload` 方法，手动包装 `wx.uploadFile` 回调为 Promise。复用现有的 401 拦截逻辑。

**理由**: `wx.uploadFile` 原生不支持 Promise 风格调用。封装后与现有 `get`/`post`/`put`/`del` 方法风格一致，调用方代码更清晰。

## Risks / Trade-offs

- **[审核窗口期风险]** → 乐观策略下 risky 内容有最多 30 分钟可见窗口。缓解：图片仅在小程序内展示，用户身份可追溯（openid），非公开传播。
- **[access_token 限额]** → 微信 access_token 每天获取次数有限（默认 2000 次）。缓解：内存缓存 + 过期前刷新，正常使用每天仅需约 12 次刷新。
- **[openid 2 小时过期]** → `mediaCheckAsync` 要求用户近 2 小时访问过小程序，否则返回 errcode 61010。缓解：后台重试机制，且不影响文件上传本身。
- **[磁盘空间增长]** → 用户上传的文件持续增长。缓解：`UploadedFile` 表记录元数据，可定期清理 `IsDeleted=true` 的文件。未来可加单用户上传限制。
- **[微信回调端点安全]** → 回调端点无 JWT 认证，通过签名验证确认请求来自微信。缓解：严格实现签名验证算法，拒绝无效签名。
- **[本地存储单点故障]** → 磁盘损坏导致文件丢失。缓解：定期备份。未来迁移到云存储可解决。
- **[审核回调 URL 需公网可访问]** → `mediaCheckAsync` 要求 `media_url` 可被微信服务器下载。开发环境需内网穿透或公网地址。缓解：开发环境可跳过审核调用（配置开关）。
