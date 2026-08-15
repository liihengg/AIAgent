## Why

后端核心业务链路（登录认证、抽奖、退款、会员管理、商品管理、活动管理）在关键节点缺乏日志记录。当出现"用户说抽到了但没记录"、"库存对不上"、"谁操作了退款"等问题时，排查无据可依。支付链路和文件审核链路日志已较完善，但核心业务逻辑几乎是日志盲区。

## What Changes

- 在 `WxAuthService` 中为微信 `code2session` 调用添加异常和错误码日志
- 在 `AuthController` 中为新用户创建、禁用账号登录、登录成功添加日志
- 在 `ActivityService.DrawAsync` 中为活动不存在、活动未激活、每日次数用尽、参与人数上限、防重复中奖触发、库存降级、库存递减失败、抽奖结果添加日志
- 在 `ActivityService` 中为活动创建、活动更新、中奖记录设地址添加日志
- 在 `PaymentController` 中为发起退款、同意退款、拒绝退款添加管理员操作日志
- 在 `MembershipController` 中为管理员调整会员、管理员重置会员添加操作日志
- 在 `ProductService` 中为商品创建、更新、删除添加日志

## Capabilities

### New Capabilities

- `api-logging`: 后端 API 关键节点的日志记录规范，定义认证、抽奖、退款、会员管理、商品管理、活动管理各链路在哪些节点记录什么级别的日志

### Modified Capabilities

（无）

## Impact

- **涉及文件**: 6 个文件，共 24 个日志点
  - `Services/Auth/WxAuthService.cs` — 2 个日志点
  - `Controllers/AuthController.cs` — 3 个日志点
  - `Services/Activity/ActivityService.cs` — 11 个日志点
  - `Controllers/PaymentController.cs` — 3 个日志点
  - `Controllers/MembershipController.cs` — 2 个日志点
  - `Services/Product/ProductService.cs` — 3 个日志点
- **不涉及逻辑变更**: 仅添加 `Log.LogInfo/LogWarning/LogError` 调用，不修改业务逻辑
- **新增引用**: 部分文件需补充 `using Utils;`
- **依赖**: 使用现有 `Utils.Log` 静态类，无新依赖
