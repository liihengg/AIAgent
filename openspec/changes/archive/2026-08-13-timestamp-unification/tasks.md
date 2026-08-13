## 1. Clock 重写（基础层）

- [x] 1.1 重写 `Utils/Clock.cs`：`Now` 返回 `long`（`DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()`），新增 `ToUnixSeconds(long ms)`、`FromUnixSeconds(long s)`、`ToWechatExpire(long ms)`、`ParseWechatTime(string)` 方法
- [x] 1.2 删除旧 `Clock.FromUtc(DateTime)` 方法

## 2. Models 层改造

- [x] 2.1 `User.cs`: CreateTime、LastLoginTime → `long`
- [x] 2.2 `UserAddress.cs`: CreateTime、UpdateTime → `long`
- [x] 2.3 `LotteryActivity.cs`: StartTime、EndTime、CreateTime、UpdateTime → `long`
- [x] 2.4 `DrawRecord.cs`: DrawTime → `long`
- [x] 2.5 `MembershipConfig.cs`: UpdateTime → `long`
- [x] 2.6 `MembershipRecord.cs`: StartTime、EndTime、CreateTime → `long`
- [x] 2.7 `UserMembership.cs`: ExpireTime → `long?`  (注: ExpireTime 改为 long, PermanentExpireTime 改为 long 常量)
- [x] 2.8 `PaymentOrder.cs`: CreateTime → `long`、PayTime → `long?`
- [x] 2.9 `Product.cs`: CreateTime、UpdateTime → `long`

## 3. DTOs 层改造

- [x] 3.1 `LoginResponseDto.cs`: ExpiresAt、MembershipExpireTime → `long` / `long?`
- [x] 3.2 `UserMembershipResponseDto.cs`: ExpireTime → `long?`
- [x] 3.3 `ActivityDetailResponseDto.cs`: StartTime、EndTime → `long`
- [x] 3.4 `ActivitySquareDto.cs`: StartTime、EndTime → `long`
- [x] 3.5 `DrawRecordResponseDto.cs`: DrawTime → `long`
- [x] 3.6 `CreateActivityDto.cs`: StartTime、EndTime → `long`
- [x] 3.7 `ShopDto.cs`: CreateTime → `long`  (注: ShopDto 不存在, 跳过)
- [x] 3.8 `AddressResponseDto.cs`: CreateTime、UpdateTime → `long`
- [x] 3.9 `PaymentQueryResultDto.cs`: CreateTime → `long`、PayTime → `long?`
- [x] 3.10 `PaymentOrderResponseDto.cs`: CreateTime → `long`
- [x] 3.11 `MembershipInfoDto.cs`: MembershipExpireTime → `long?`
- [x] 3.12 `SetUserMembershipDto.cs`: ExpireTime → `long?`
- [x] 3.13 `ProductResponseDto.cs`: CreateTime、UpdateTime → `long`
- [x] 3.14 `ActivityDrawRecordResponseDto.cs`: DrawTime → `long`  (补充: 原任务列表遗漏)
- [x] 3.15 `MembershipRecordResponseDto.cs`: StartTime、EndTime、CreateTime → `long`  (补充: 原任务列表遗漏)

## 4. Services 层改造

- [x] 4.1 `ActivityService.cs`: Clock.Now 赋值改为 long；UpdateTime 赋值；日期加减改为 long 算术
- [x] 4.2 `PaymentService.cs`: CreateTime 赋值；time_expire 改用 `Clock.ToWechatExpire()`；PayTime 赋值改用 `Clock.ParseWechatTime()`；会员到期计算改为 long 算术；JSAPI timeStamp 改用 `Clock.ToUnixSeconds(Clock.Now)`；防重复下单 5 分钟窗口改为 long 比较
- [x] 4.3 `PaymentBackgroundService.cs`: cutoff 计算改为 long 算术（`Clock.Now - 10 * 60_000L`）
- [x] 4.4 `ActivityBackgroundService.cs`: 时间比较不变（long 比较），确认无 DateTime 用法残留
- [x] 4.5 `TokenService.cs`: JWT exp 改用 `Clock.Now + expireMinutes * 60_000L`，内部用 `Clock.ToUnixSeconds()` 转秒
- [x] 4.6 `WechatPayClient.cs`: `_certDownloadTime` 改为 `long`；签名 timestamp 改用 `Clock.ToUnixSeconds(Clock.Now)`；证书刷新判断改为 long 比较
- [x] 4.7 `AuthController.cs`: membershipExpireTime → long, jwt.ValidTo → long  (补充: 原任务列表遗漏)
- [x] 4.8 `MembershipController.cs`: 所有 DateTime 变量和运算改为 long  (补充: 原任务列表遗漏)
- [x] 4.9 `AddressService.cs`: DateTime now → long now  (补充: 原任务列表遗漏)
- [x] 4.10 `ProductService.cs`: DateTime now → long now  (补充: 原任务列表遗漏)
- [x] 4.11 `ProductSeeder.cs`: DateTime now → long now, SeedProductDto.UpdateTime → long  (补充: 原任务列表遗漏)

## 5. 数据库重建

- [x] 5.1 `AppDbContext.cs`: 种子数据 `seedTime` 从 `DateTime` 改为 `long` 常量
- [x] 5.2 删除 `Migrations/` 目录下所有迁移文件
- [x] 5.3 删除 `lottery.db` 文件
- [x] 5.4 执行 `dotnet ef migrations add InitialTimestamp` 生成新迁移
- [x] 5.5 执行 `dotnet build` 确认编译通过
- [x] 5.6 执行 `dotnet run` 确认自动迁移和种子数据正常

## 6. 前端 util.js 重写

- [x] 6.1 `formatDate(ts)`: 改为 `new Date(ts)` 构造，移除字符串解析
- [x] 6.2 `formatDateTime(ts)`: 改为 `new Date(ts)` 构造，移除字符串解析
- [x] 6.3 `formatExpireText(ts)`: 改为 `new Date(ts)` 构造，10 年判断改为时间戳比较
- [x] 6.4 `isMembershipExpired()`: 改为直接数值比较 `ts <= Date.now()`，移除 `new Date()` 构造
- [x] 6.5 `formatTime(date)`: 确认此函数仅用于日志页（`new Date(log)` 本地时间），不涉及 API 数据，保持不变

## 7. 前端时间提交改造

- [x] 7.1 `activity-create.js`: startTime/endTime 从拼字符串改为 `new Date(date + 'T00:00:00+08:00').getTime()` / `new Date(date + 'T23:59:59+08:00').getTime()`
- [x] 7.2 `membership-list.js`: expireTime 从拼字符串改为 `new Date(form.expireTime + 'T23:59:59+08:00').getTime()`

## 8. 前端页面验证

- [x] 8.1 检查所有页面 JS 中 `formatDate` / `formatDateTime` / `formatExpireText` 调用处，确认参数从 string 变 long 后无需改动（函数内部已处理）
- [x] 8.2 检查 `membership.js`、`mine.js`、`membership-records.js` 中 `membershipExpireTime` 的本地存储和读取逻辑，确认直接传递 long
- [x] 8.3 检查 `order-detail.js`、`payment-orders.js` 中 `payTime` 可空判断逻辑（`o.payTime ? formatDateTime(o.payTime) : ''`），确认 long 的 falsy 判断正确

## 9. 编译与集成验证

- [x] 9.1 后端 `dotnet build` 通过
- [x] 9.2 后端 `dotnet run` 启动正常，自动迁移执行成功
- [ ] 9.3 用 `LotteryService.http` 测试关键 API：微信登录、创建活动、抽奖、创建支付订单，确认时间字段返回 JSON 数字
- [ ] 9.4 前端微信开发者工具预览，确认活动列表、抽奖记录、订单详情页面时间显示正常
