## Context

后端已有 `Utils.Log` 静态类（封装 Serilog），支付链路（PaymentService、WechatPayClient、PaymentBackgroundService）和文件审核链路（FileService、FileAuditBackgroundService）日志覆盖较完善。但核心业务链路（登录、抽奖、退款、会员管理、商品管理、活动管理）几乎零日志。

现有日志风格统一：`Log.LogInfo($"中文描述: 参数A={a}, 参数B={b}")`。新增日志保持一致。

## Goals / Non-Goals

**Goals:**
- 在 6 个文件、24 个关键节点添加日志调用
- 覆盖认证、抽奖、活动管理、退款、会员管理、商品管理六条链路
- 日志级别遵循统一策略：正常流程用 Info，预期内拒绝用 Warning，异常/数据不一致用 Error

**Non-Goals:**
- 不修改任何业务逻辑，仅添加 `Log.LogInfo/LogWarning/LogError` 调用
- 不重构 Controller 内联逻辑到 Service 层（退款、会员调整保持现状）
- 不给 TokenService 加日志（纯机械操作，无业务决策）
- 不给只读查询接口加日志
- 不引入结构化日志框架或日志中间件
- 不调整现有日志配置（Serilog 输出模板、文件滚动策略等）

## Decisions

### 1. 日志放在 Service 层而非 Controller 层

**决策**: 业务逻辑日志放 Service 层（ActivityService、ProductService、WxAuthService），Controller 层仅记录其内联业务逻辑的日志（PaymentController 退款、MembershipController 会员调整）。

**原因**: Service 层是业务决策所在地，可被多个入口调用。Controller 是薄包装，仅做路由转发和 HTTP 特定处理。

**例外**: PaymentController 的退款三件套和 MembershipController 的会员调整逻辑直接写在 Controller 里没走 Service，日志跟着逻辑放 Controller。

### 2. 日志级别策略

| 级别 | 适用场景 | 本变更中的例子 |
|------|---------|---------------|
| Info | 正常业务流程关键节点 | 登录成功、抽奖结果、活动创建、退款操作 |
| Warning | 预期内拒绝、业务规则触发 | 禁用账号登录、活动未激活、参与人数上限、微信 errcode |
| Error | 异常、不应发生的情况 | 库存递减失败、微信 API 异常 |

**关于防重复中奖和库存降级**: 这两个点虽然每次抽奖都可能触发，但记录为 Info 是因为它们是排查"为什么抽到空奖"的关键证据。不降级为 Debug。

### 3. 日志参数选择

日志包含的关键参数按链路区分：

- **认证**: userId, openId（前8位）, isNew
- **抽奖**: userId, activityId, prizeId, isWin, 当前次数/上限
- **退款**: adminId, orderId, outTradeNo
- **会员**: adminId, targetUserId, oldLevel, newLevel
- **商品**: productId, name
- **活动**: userId, activityId, name

**不记录敏感信息**: 不记录完整 OpenId、不记录 Token、不记录用户密码。

### 4. using Utils 补充

`LotteryController`、`ProductController`、`MembershipController` 等文件当前没有 `using Utils;`，添加日志时需补上。`AuthController` 已有。

## Risks / Trade-offs

- **日志量增加**: 抽奖链路每次请求会多出 1-2 条 Info 日志（防重复中奖、库存降级在正常路径也会触发）。在当前用户量下可接受。如未来量级增长，可考虑将防重复中奖和库存降级降级为 Debug。
- **字符串插值 vs 结构化日志**: 现有代码全部使用 `$""` 字符串插值，新日志保持一致。Serilog 对 `$""` 的处理不如模板 `{Property}` 结构化，但一致性优先。
- **Controller 内联逻辑的日志位置**: 退款和会员调整的日志放 Controller 层。如果将来重构到 Service 层，日志需要跟着迁移。这是一个已知的技术债，但不影响当前功能。
