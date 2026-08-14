## 1. 后端数据模型与配置

- [x] 1.1 创建 `Models/File/FileType.cs` 枚举（Avatar=0, Prize=1, Other=99）
- [x] 1.2 创建 `Models/File/AuditStatus.cs` 枚举（Pending=0, Checking=1, Approved=2, Rejected=3, NeedsReview=4）
- [x] 1.3 创建 `Models/File/UploadedFile.cs` 模型（Id, UserId, OriginalFileName, StoredFileName, FilePath, FileSize, ContentType, FileType, AuditStatus, AuditTraceId, AuditTime, AuditRetryCount, UploadTime, IsDeleted）
- [x] 1.4 修改 `AppDbContext.cs`：添加 `DbSet<UploadedFile>`，配置索引（UserId, AuditStatus + AuditTime）
- [x] 1.5 生成 EF Core migration：`dotnet ef migrations add AddUploadedFile`
- [x] 1.6 修改 `appsettings.json`：添加 `FileUpload` 配置节（MaxFileSize, AllowedTypes, UploadPath）和 `WxCallback` 配置节（Token, EncodingAESKey）
- [x] 1.7 修改 `Program.cs`：添加 `app.UseStaticFiles()` 配置（在 UseAuthorization 之前），配置请求体大小限制，注册 DI 服务

## 2. 后端文件服务与控制器

- [x] 2.1 创建 `DTOs/File/UploadResponseDto.cs`（Url, OriginalFileName, StoredFileName, Size, ContentType, FileId, AuditStatus）
- [x] 2.2 创建 `DTOs/File/FileListResponseDto.cs`（Id, Url, UploadTime, AuditStatus, FileType）
- [x] 2.3 创建 `Services/File/IFileService.cs` 接口（UploadAsync, GetUserFilesAsync, GetUserFileAsync, DeleteFileAsync）
- [x] 2.4 创建 `Services/File/FileService.cs` 实现：文件校验（类型+大小）、GUID 命名、按类型+年月分区存储、磁盘写入、数据库记录、物理删除
- [x] 2.5 创建 `Controllers/UploadController.cs`：`POST /api/upload`（[Authorize], IFormFile + type 参数）、`GET /api/upload/files`（[Authorize], type 过滤）、`GET /api/upload/files/{id}`、`DELETE /api/upload/files/{id}`（校验归属权）
- [x] 2.6 在 `Program.cs` 注册 `IFileService` 和 `FileService` 到 DI

## 3. 后端微信 access_token 服务

- [x] 3.1 创建 `Services/Auth/IAccessTokenService.cs` 接口（GetAccessTokenAsync）
- [x] 3.2 创建 `Services/Auth/AccessTokenService.cs` 实现：调用微信稳定版 token 接口、内存缓存、过期前 5 分钟刷新
- [x] 3.3 在 `Program.cs` 注册 `IAccessTokenService` 和 `AccessTokenService`（Singleton）
- [x] 3.4 在 `IFileService.UploadAsync` 中集成审核调用：获取 access_token → 调用 `mediaCheckAsync` → 记录 trace_id → 设置 AuditStatus=Checking。access_token 获取失败时设为 Pending 不影响上传

## 4. 后端微信回调与审核

- [x] 4.1 创建 `Controllers/WxCallbackController.cs`：`GET /api/wx/callback`（签名验证 + 返回 echostr）、`POST /api/wx/callback`（解析回调 XML/JSON，提取 trace_id + result.suggest，更新审核状态）
- [x] 4.2 实现审核结果为 risky 时的处理：更新 AuditStatus=Rejected → 标记 IsDeleted → 删除物理文件
- [x] 4.3 实现审核结果为 pass 时的处理：更新 AuditStatus=Approved

## 5. 后端审核超时重试后台服务

- [x] 5.1 创建 `Services/File/FileAuditBackgroundService.cs`（HostedService）：每 5 分钟扫描 AuditStatus=Checking 且 AuditTime 超过 30 分钟的记录
- [x] 5.2 实现重试逻辑：重新调用 mediaCheckAsync，AuditRetryCount +1，更新 AuditTime
- [x] 5.3 实现 3 次重试上限：超过 3 次标记为 Approved
- [x] 5.4 在 `Program.cs` 注册 `FileAuditBackgroundService`（AddHostedService）

## 6. 后端构建验证

- [x] 6.1 运行 `dotnet build` 确认编译通过
- [x] 6.2 运行 `dotnet ef migrations list` 确认 migration 正常
- [x] 6.3 启动后端服务，测试 `POST /api/upload` 上传一张测试图片，验证文件存储和返回
- [x] 6.4 测试 `GET /api/upload/files` 返回文件列表
- [ ] 6.5 测试 `DELETE /api/upload/files/{id}` 删除文件
- [ ] 6.6 测试 `GET /uploads/...` 静态文件访问

## 7. 前端基础设施

- [x] 7.1 修改 `utils/request.js`：新增 `upload(url, filePath, formData)` 方法，包装 `wx.uploadFile` 为 Promise，自动注入 Authorization 头，复用 401 拦截逻辑
- [x] 7.2 修改 `utils/api.js`：新增 `uploadFile(filePath, type)` 调用 `upload('/upload', filePath, { type })`
- [x] 7.3 修改 `utils/api.js`：新增 `getUserFiles(type)` 调用 `get('/upload/files', { type })`
- [x] 7.4 修改 `utils/api.js`：新增 `deleteUserFile(id)` 调用 `del('/upload/files/' + id)`
- [x] 7.5 修改 `utils/constants.js`：新增 `AuditStatus` 枚举（`FILE_BASE_URL` 常量已删除，URL 拼接改由后端 `PublicBaseUrl` 配置实现）

## 8. 前端 image-picker 自定义组件

- [x] 8.1 创建 `components/image-picker/image-picker.json`（自定义组件配置）
- [x] 8.2 创建 `components/image-picker/image-picker.wxml`：图片网格布局，包含上传按钮、已上传图片缩略图、选中高亮、上传 loading
- [x] 8.3 创建 `components/image-picker/image-picker.wxss`：网格样式、缩略图样式、选中高亮、loading 蒙层
- [x] 8.4 创建 `components/image-picker/image-picker.js`：properties（selectedUrl, fileType）、loadUserFiles 方法、chooseAndUpload 方法、selectImage 方法、triggerEvent('select', {url})
- [x] 8.5 在 `app.json` 中注册组件为全局组件（或在使用页面 json 中引入）

## 9. 前端头像上传改造

- [x] 9.1 修改 `pages/profile-setup/profile-setup.js`：onSubmit 方法中，如果 avatarUrl 是临时路径（wxfile:// 开头），先调用 `api.uploadFile(avatarUrl, 'avatar')` 获取持久 URL，再调用 `api.updateProfile`
- [x] 9.2 修改 `pages/profile-setup/profile-setup.wxml`：image 组件添加 binderror 兜底处理
- [ ] 9.3 准备默认头像图片资源 `/images/default-avatar.png`

## 10. 前端奖品图片选择改造

- [x] 10.1 修改 `pages/activity-create/activity-create.json`：引入 image-picker 组件
- [x] 10.2 修改 `pages/activity-create/activity-create.wxml`：奖品编辑弹窗中将 imageUrl 文本输入替换为 `<image-picker>` 组件，绑定 `bind:select` 事件
- [x] 10.3 修改 `pages/activity-create/activity-create.js`：新增 onImageSelect 方法，将选中 URL 写入 editDraft.imageUrl
- [ ] 10.4 准备默认奖品图片资源 `/images/default-prize.png`
- [ ] 10.5 在活动详情页、抽奖转盘页等展示奖品图片的地方添加 binderror 兜底

## 11. 集成测试与验证

- [ ] 11.1 前端完整流程测试：登录 → 完善资料（上传头像）→ 创建活动 → 编辑奖品（选择/上传图片）→ 提交活动
- [ ] 11.2 验证图片 URL 持久化：重新打开小程序，确认头像和奖品图片正常显示
- [ ] 11.3 测试图片加载失败兜底：手动删除一个文件，验证前端 binderror 替换为默认图
- [ ] 11.4 验证文件删除：用户删除已上传文件，确认磁盘文件和数据库记录均被处理
- [x] 11.5 后端 `dotnet build` 最终确认
