# file-upload

## Purpose

提供客户端文件上传、存储、管理及内容安全审核能力，支持用户头像和奖品图片的持久化存储，通过微信内容安全异步审核保障上传内容合规。

## Requirements

### Requirement: 文件上传

系统 SHALL 提供文件上传接口，接收 `multipart/form-data` 格式的图片文件。上传接口 MUST 要求用户已登录（携带有效 JWT Token）。系统 MUST 校验文件大小不超过 2MB，文件类型仅允许 `image/jpeg`、`image/png`、`image/webp`。系统 MUST 将文件存储到本地磁盘，并为每个文件生成全局唯一的存储文件名（GUID + 扩展名）。系统 MUST 记录文件元数据到数据库，包括上传者 ID、原始文件名、存储路径、文件大小、内容类型、文件类型（头像/奖品）和上传时间。数据库存储相对路径（如 `/uploads/prizes/202608/{guid}.jpg`），API 返回时 MUST 拼接为完整 URL（如 `http://localhost:5297/uploads/prizes/202608/{guid}.jpg`），通过后端 `PublicBaseUrl` 配置实现，便于 CDN 迁移时只改一处配置。

#### Scenario: 已登录用户成功上传图片
- **WHEN** 已登录用户通过 `POST /api/upload` 上传一张合规的 JPG 图片（1MB），formData 包含 `type=prize`
- **THEN** 系统返回 `Result<UploadResponseDto>`，包含文件完整 URL（如 `http://localhost:5297/uploads/prizes/202608/{guid}.jpg`）、原始文件名、存储文件名、文件大小和内容类型

#### Scenario: 未登录用户尝试上传
- **WHEN** 未携带 JWT Token 的请求访问 `POST /api/upload`
- **THEN** 系统返回 401 Unauthorized

#### Scenario: 文件超过大小限制
- **WHEN** 用户上传一张 3MB 的图片
- **THEN** 系统返回错误，提示文件大小超过 2MB 限制

#### Scenario: 不支持的文件类型
- **WHEN** 用户上传一个 GIF 或 BMP 格式的文件
- **THEN** 系统返回错误，提示不支持的文件类型

### Requirement: 文件列表查询

系统 SHALL 提供文件列表查询接口，返回当前登录用户上传的文件列表。系统 MUST 支持按文件类型（avatar/prize）过滤。返回列表 MUST 排除已软删除的文件。每个文件项 SHALL 包含文件 ID、URL、上传时间和审核状态。

#### Scenario: 查询用户已上传的奖品图片
- **WHEN** 已登录用户请求 `GET /api/upload/files?type=prize`
- **THEN** 系统返回该用户上传的所有未删除的奖品类型文件列表，按上传时间倒序排列

#### Scenario: 查询全部类型文件
- **WHEN** 已登录用户请求 `GET /api/upload/files`（不传 type 参数）
- **THEN** 系统返回该用户所有类型的未删除文件列表

#### Scenario: 用户只能看到自己的文件
- **WHEN** 用户 A 请求文件列表
- **THEN** 系统仅返回用户 A 上传的文件，不包含其他用户的文件

### Requirement: 文件删除

系统 SHALL 提供文件删除接口，允许用户删除自己上传的文件。系统 MUST 校验文件归属权（只能删除自己的文件）。删除操作 MUST 同时执行软删除（数据库标记）和物理删除（删除磁盘文件）。如果磁盘文件删除失败，系统 MUST 记录日志但不影响数据库软删除的成功。

#### Scenario: 用户删除自己的文件
- **WHEN** 已登录用户请求 `DELETE /api/upload/files/{id}`，且该文件属于当前用户
- **THEN** 系统标记文件为已删除，删除磁盘上的物理文件，返回成功

#### Scenario: 用户尝试删除他人的文件
- **WHEN** 用户 A 请求删除用户 B 上传的文件
- **THEN** 系统返回 404 Not Found（不泄露文件存在性）

### Requirement: 静态文件访问

系统 MUST 通过 HTTP 提供已上传文件的访问能力，URL 路径以 `/uploads/` 开头。静态文件服务 MUST 不要求认证（公开访问）。系统 SHOULD 为静态文件设置缓存头以优化性能。

#### Scenario: 通过 URL 访问已上传的图片
- **WHEN** 任意客户端通过 `GET /uploads/avatars/202608/{guid}.jpg` 请求一个已上传的图片
- **THEN** 系统返回该图片文件内容，Content-Type 对应文件类型

#### Scenario: 访问不存在的文件
- **WHEN** 客户端请求一个不存在的文件路径
- **THEN** 系统返回 404 Not Found

### Requirement: 内容安全审核

系统 MUST 在文件上传成功后调用微信 `mediaCheckAsync 2.0` 接口进行异步内容安全审核。调用审核接口时 MUST 传入文件的可公网访问 URL、媒体类型（2=图片）、版本号（2）、场景值（1=资料）和用户的 OpenId。系统 MUST 记录审核返回的 `trace_id` 用于关联异步结果。上传后文件默认立即可用（乐观策略），审核结果返回后再更新状态。审核调用 MUST 同步 `await` 执行（不使用 fire-and-forget），因为 `FileService` 是 Scoped 生命周期，异步任务中 `_db` 可能被 dispose。当 `WeChatPay:MockEnabled` 配置为 `true` 时，系统 MUST 跳过审核调用，直接将文件审核状态标记为 `Approved`。

#### Scenario: 上传后自动发起审核
- **WHEN** 用户成功上传一张图片且 MockEnabled=false
- **THEN** 系统同步调用 `mediaCheckAsync`，记录返回的 `trace_id`，文件状态设为 `Checking`

#### Scenario: Mock 模式跳过审核
- **WHEN** 配置 `WeChatPay:MockEnabled=true` 且用户成功上传一张图片
- **THEN** 系统跳过审核调用，文件审核状态直接标记为 `Approved`

#### Scenario: 用户的 OpenId 过期导致审核调用失败
- **WHEN** 用户超过 2 小时未与微信交互，`mediaCheckAsync` 返回 errcode 61010
- **THEN** 系统记录错误日志，文件仍保持可用状态（乐观放行），后续由后台服务重试

### Requirement: 审核结果回调处理

系统 SHALL 提供微信回调接收端点（`GET/POST /api/wx/callback`），处理微信消息接收服务器的验证请求和审核结果推送。GET 请求 MUST 验证签名并原样返回 `echostr`。POST 请求 MUST 解析回调数据中的 `trace_id` 和 `result.suggest`，更新对应文件记录的审核状态。当审核结果为 `risky` 时，系统 MUST 将文件标记为已拒绝并执行文件删除（软删除 + 物理删除）。当审核结果为 `pass` 时，系统 MUST 将文件审核状态更新为已通过。

#### Scenario: 微信验证签名
- **WHEN** 微信发送 GET 请求 `/api/wx/callback?signature=xxx&timestamp=xxx&nonce=xxx&echostr=xxx`
- **THEN** 系统验证签名通过后原样返回 `echostr`

#### Scenario: 审核通过回调
- **WHEN** 微信推送审核结果 `result.suggest=pass`
- **THEN** 系统更新对应 `trace_id` 的文件审核状态为 `Approved`

#### Scenario: 审核拒绝回调
- **WHEN** 微信推送审核结果 `result.suggest=risky`
- **THEN** 系统将文件审核状态更新为 `Rejected`，删除物理文件，标记数据库记录为已删除

### Requirement: 审核超时重试

系统 SHALL 提供后台服务定期检查审核状态为 `Checking` 且超过 30 分钟未收到结果的文件记录。系统 MUST 对这些记录重新调用 `mediaCheckAsync`，每次重试更新时间戳。系统 MUST 限制最大重试次数为 3 次，超过后将文件标记为 `Approved`（乐观放行）。

#### Scenario: 审核结果 30 分钟内未返回
- **WHEN** 文件审核状态为 `Checking` 且超过 30 分钟
- **THEN** 后台服务重新调用 `mediaCheckAsync`，重试计数加 1

#### Scenario: 重试 3 次仍无结果
- **WHEN** 文件已重试 3 次仍未收到审核结果
- **THEN** 系统将文件审核状态标记为 `Approved`

### Requirement: access_token 管理

系统 SHALL 提供 access_token 缓存服务，用于调用微信 `mediaCheckAsync` 接口。系统 MUST 缓存 access_token 并在过期前自动刷新。系统 SHOULD 使用微信稳定版 access_token 接口。如果获取 access_token 失败，文件上传仍须成功（审核调用延迟到后台重试）。

#### Scenario: 正常获取 access_token
- **WHEN** 系统需要调用 `mediaCheckAsync` 且缓存中无有效 access_token
- **THEN** 系统调用微信接口获取 access_token 并缓存

#### Scenario: access_token 获取失败不影响上传
- **WHEN** 上传文件时获取 access_token 失败
- **THEN** 文件上传正常返回成功，审核状态设为 `Pending`，由后台服务后续重试

### Requirement: 前端上传封装

前端 SHALL 封装 `wx.uploadFile` 为 Promise 风格的 `upload` 方法，自动注入 `Authorization` 头。前端 SHALL 封装文件管理 API（上传、查询列表、删除）到 `api.js`。后端 API 返回的 URL 为完整地址，前端可直接用于 `<image src>` 展示，无需额外拼接。

#### Scenario: 调用 upload 方法上传文件
- **WHEN** 前端调用 `api.uploadFile(filePath, 'prize')`
- **THEN** 系统通过 `wx.uploadFile` 发送 `multipart/form-data` 请求到 `/api/upload`，返回 Promise 解析为服务器响应

#### Scenario: 上传时 token 过期
- **WHEN** 上传请求返回 401
- **THEN** 系统清除 token 并提示用户重新登录（与现有 request.js 401 拦截逻辑一致）

### Requirement: 图片选择器组件

前端 SHALL 提供自定义组件 `image-picker`，展示当前用户已上传的图片网格。组件 SHALL 支持选择已有图片和上传新图片。组件 MUST 在上传过程中显示 loading 状态。组件 SHALL 通过 `bind:select` 事件向父页面传递选中的图片 URL。

#### Scenario: 打开组件时加载用户图片
- **WHEN** 组件挂载时
- **THEN** 组件调用 `GET /api/upload/files?type={fileType}` 加载用户已上传图片列表并展示为网格

#### Scenario: 从已有图片中选择
- **WHEN** 用户点击网格中的某张图片
- **THEN** 该图片被高亮选中，组件通过 `bind:select` 触发事件返回选中的 URL

#### Scenario: 上传新图片
- **WHEN** 用户点击上传按钮
- **THEN** 调用 `wx.chooseMedia` 选择图片，调用 `api.uploadFile` 上传，上传中显示 loading，完成后刷新列表并自动选中新上传的图片

### Requirement: 头像上传改造

前端 `profile-setup` 页 SHALL 在用户点击保存时上传头像。如果用户选择了新头像（临时路径），MUST 先调用上传接口获得持久 URL，再调用 `PUT /api/auth/profile` 更新资料。如果头像未变更，不执行上传。

#### Scenario: 用户选择新头像后保存
- **WHEN** 用户通过 `chooseAvatar` 选择了新头像并点击保存
- **THEN** 系统先上传头像文件到服务器获得持久 URL，再用该 URL 更新用户资料

#### Scenario: 用户未更换头像直接保存
- **WHEN** 用户未修改头像（仍使用已有的持久 URL）并点击保存
- **THEN** 系统直接更新昵称，不执行文件上传

### Requirement: 奖品图片选择改造

前端 `activity-create` 页的奖品编辑弹窗 SHALL 使用 `image-picker` 组件替代手动 URL 输入。选中的图片 URL 存入 `prizeSlots[index].imageUrl`。

#### Scenario: 编辑奖品时选择图片
- **WHEN** 用户在奖品编辑弹窗中通过 `image-picker` 选择了一张图片
- **THEN** 该图片 URL 被写入 `editDraft.imageUrl`

#### Scenario: 奖品不设置图片
- **WHEN** 用户在奖品编辑弹窗中未选择任何图片
- **THEN** `editDraft.imageUrl` 为空字符串，奖品不显示图片

### Requirement: 图片加载失败兜底

前端所有展示用户上传图片的 `<image>` 组件 MUST 绑定 `binderror` 事件处理。图片加载失败时 SHALL 将 `src` 替换为对应类型的默认图片路径（如 `/images/default-avatar.png`、`/images/default-prize.png`）。

#### Scenario: 审核拒绝导致图片 URL 失效
- **WHEN** 图片 URL 对应的文件已被审核拒绝并删除，`<image>` 加载失败
- **THEN** `binderror` 处理器将 `src` 替换为默认图片

#### Scenario: 网络问题导致图片加载失败
- **WHEN** 图片加载因网络超时失败
- **THEN** `binderror` 处理器将 `src` 替换为默认图片
