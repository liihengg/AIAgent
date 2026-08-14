## Why

当前小程序中用户头像、奖品图片等场景没有文件上传能力。用户头像使用微信 `chooseAvatar` 返回的临时路径（`wxfile://tmp_xxx`），重启后失效；奖品图片和商品图片只能手动输入 URL，用户需要自己找图床。需要增加客户端文件上传功能，实现图片的持久化存储和管理，同时接入微信内容安全审核保障合规。

## What Changes

- 新增后端文件上传 API（`POST /api/upload`），接收 `multipart/form-data` 格式的图片文件，校验类型和大小后存储到本地磁盘
- 新增后端文件管理 API（`GET /api/upload/files`、`GET /api/upload/files/{id}`、`DELETE /api/upload/files/{id}`），用户可查看和删除自己上传的文件
- 新增 `UploadedFile` 数据模型，记录文件元数据（文件名、路径、大小、类型、审核状态等）
- 新增 `FileService` 处理文件校验、存储、删除逻辑
- 新增 `AccessTokenService` 管理微信 `access_token` 缓存（供 `mediaCheckAsync` 调用使用）
- 新增微信内容安全异步审核集成：上传后调用 `mediaCheckAsync 2.0`，通过回调端点接收审核结果
- 新增 `WxCallbackController` 处理微信回调（签名验证 + 审核结果更新）
- 新增 `FileAuditBackgroundService` 后台服务，对超时未返回审核结果的记录重试调用（最多 3 次，之后乐观放行）
- 后端启用 `UseStaticFiles`，通过 `/uploads/` 路径提供文件访问
- 前端 `request.js` 新增 `upload` 方法封装 `wx.uploadFile`
- 前端 `api.js` 新增 `uploadFile`、`getUserFiles`、`deleteUserFile` 方法
- 前端新增 `image-picker` 自定义组件，展示用户已上传图片网格，支持选择已有图片或上传新图片
- 前端 `profile-setup` 页改造：保存时上传头像到服务器，获得持久 URL 后再更新资料
- 前端 `activity-create` 页改造：奖品编辑弹窗中使用 `image-picker` 组件替代手动 URL 输入
- 前端图片展示统一添加 `binderror` 兜底处理，图片加载失败时显示默认图

## Capabilities

### New Capabilities
- `file-upload`: 客户端文件上传、存储、管理及内容安全审核能力

### Modified Capabilities
（无现有 capability 的 spec 级别行为变更）

## Impact

- **后端新增文件**: UploadController.cs、WxCallbackController.cs、FileService.cs + IFileService.cs、AccessTokenService.cs + IAccessTokenService.cs、FileAuditBackgroundService.cs、UploadedFile.cs、FileType.cs、AuditStatus.cs、UploadResponseDto.cs、FileListResponseDto.cs
- **后端修改文件**: AppDbContext.cs（加 DbSet）、Program.cs（UseStaticFiles + 请求体限制 + DI 注册）、appsettings.json（FileUpload + WxCallback 配置节）
- **前端新增**: components/image-picker/ 自定义组件
- **前端修改**: request.js、api.js、constants.js、profile-setup.js/wxml、activity-create.js/wxml
- **数据库**: 新增 UploadedFiles 表（1 个 EF Core migration）
- **微信小程序后台配置**: uploadFile 合法域名、downloadFile 合法域名、消息接收服务器 URL + Token
- **外部依赖**: 微信 `mediaCheckAsync 2.0` API（异步内容审核）、微信 `access_token` 获取接口
- **文件存储**: 本地磁盘 `wwwroot/uploads/{avatars|prizes}/{yyyyMM}/{guid}.{ext}`，CDN 友好的相对路径设计
