# 缺失功能与优化清单

> 2026-08-13 全面审查产出。按优先级排列，供后续迭代参考。

## P0 — 不补无法上线

### 1. 图片上传

- **现状**：奖品图片、用户头像仅支持 URL 手动填写
- **需要**：
  - 后端 `POST /api/upload` 接口（multipart/form-data）
  - 图片存储方案（本地文件 / 对象存储 / CDN）
  - 前端 `wx.uploadFile` 调用
- **影响范围**：`activity-create`（奖品图片）、`profile-setup`（头像）、`product-edit`（商品图片）

### 2. 分页

- **现状**：所有列表接口返回全量数据
- **需要**：
  - 后端：`ActivityService.GetActivitiesAsync`、`GetDrawRecordsAsync`、`PaymentController.GetOrders`、`MembershipController.GetMembershipRecords` 等加 `pageIndex` / `pageSize` 参数
  - 前端：各列表页实现上拉加载更多
- **风险**：数据量增长后接口响应缓慢、前端渲染卡顿、内存溢出

### 3. 敏感信息从 appsettings.json 移除

- **现状**：`Jwt:SecretKey`、`WxMiniProgram:AppSecret` 已提交到 Git
- **需要**：
  - 迁移到环境变量 / User Secrets / 密钥管理服务
  - 轮换已泄露的密钥
  - 清理 Git 历史（BFG / git filter-branch）
  - `appsettings.json` 中敏感字段改为占位符或删除

### 4. 前端 BASE_URL 生产化

- **现状**：`request.js` 硬编码 `http://localhost:5297/api`
- **需要**：
  - 根据环境（dev/prod）动态切换 BASE_URL
  - 生产环境必须使用 HTTPS

## P1 — 核心体验缺失

### 5. 删除 / 结束活动

- **现状**：创建者只能创建、编辑、复制活动，无法主动结束或删除
- **需要**：
  - 后端：`DELETE /api/lottery/activities/{id}`（软删除）、`POST /api/lottery/activities/{id}/end`（提前结束）
  - 前端：活动列表页操作菜单增加"结束"和"删除"按钮
  - 业务规则：已结束活动不可删除（或有数据保留期）；有中奖记录的活动不可物理删除

### 6. 订阅消息通知

- **现状**：无任何推送通知机制
- **需要**：
  - 中奖通知：用户中奖后推送订阅消息
  - 活动状态变更通知：活动开始/结束通知创建者
  - 会员到期提醒：到期前 N 天推送提醒
  - 前端：调用 `wx.requestSubscribeMessage` 获取授权
  - 后端：调用微信订阅消息 API 发送

### 7. 隐私政策 / 用户协议

- **现状**：无隐私政策页面
- **需要**：微信小程序审核必需项，需在提审前添加
  - 隐私政策页面（收集了哪些用户数据、如何使用）
  - 用户协议页面

### 8. 退款流程完善

- **现状**：`PaymentController.ApproveRefund` 直接改状态，未调用微信退款 API，未撤销会员
- **需要**：
  - 调用微信支付退款 API
  - 撤销 `UserMembership`（回退等级 + 到期时间）
  - 记录 `MembershipRecord`（状态=Refunded）
  - 恢复 `PaymentOrder.Status` 为 Refunded

### 9. API 版本控制

- **现状**：所有路由为 `/api/xxx`，无版本前缀
- **需要**：
  - 引入 API 版本控制（如 `aspnet-api-versioning` 库或手动路由前缀 `/api/v1/xxx`）
  - 首次上线前确定版本策略，避免后续破坏性变更无回退路径
  - 支持多版本并存（如 `v1` 和 `v2` 共存过渡期）

## P2 — 功能增强

### 10. 活动搜索

- 按名称搜索活动，支持活动广场页搜索框

### 11. 中奖者公示

- 活动详情页公开展示中奖名单（用户头像 + 昵称 + 奖品名）

### 12. 数据导出

- 抽奖记录、支付订单导出 CSV/Excel（管理员功能）

### 13. 用户管理（Admin）

- 后端：用户列表、封禁/解封用户
- 前端：管理中心增加用户管理入口

### 14. 奖品领取截止

- 活动设置奖品领取截止时间，超时未领奖自动作废

### 15. 活动数据统计

- 参与人数、中奖率、剩余库存、每日抽奖趋势图表

## P3 — 锦上添花

### 16. 活动模板

- 预设几套模板（节日抽奖、新品试用、粉丝福利），快速创建

### 17. 评价 / 反馈

- 用户反馈渠道（意见反馈页面）

### 18. 深色模式

- 系统级深色模式适配

---

## 技术债务与架构优化

| 项目 | 现状 | 建议 |
|------|------|------|
| 数据库 | SQLite 单文件 | 生产环境迁移到 PostgreSQL / MySQL |
| 支付回调队列 | 内存 Channel，重启丢消息 | 持久化队列（DB-backed / Redis） |
| 支付状态更新 | 无事务包裹 | `BeginTransactionAsync` 包裹 order + membership 更新 |
| 缓存 | 无 | `IMemoryCache` 缓存 `MembershipConfig` 等低频变更数据 |
| 速率限制 | 无 | `AspNetCoreRateLimit` 或内置 RateLimiter |
| 单元测试 | 无 | xUnit + Moq + EF InMemory |
| CORS | 未配置 | 如前端非同域需配置 |
| 输入验证 | DTO 无 `[Required]` / `[Range]` 等 | 添加数据注解 |
| 依赖包漏洞 | `Microsoft.OpenApi`、`SQLitePCLRaw` 有已知漏洞 | 升级到安全版本 |
