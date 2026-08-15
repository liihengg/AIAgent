## Purpose

定义后端 API 关键节点日志记录规范，覆盖认证登录、抽奖核心逻辑、退款操作、会员管理、商品管理、活动管理六条链路，确保业务关键节点可追溯、可排查。

## ADDED Requirements

### Requirement: 认证链路日志

系统 SHALL 在微信登录链路的关键节点记录日志：微信 code2session API 调用异常（Error）、返回错误码（Warning）、新用户创建（Info）、禁用账号尝试登录（Warning）、登录成功（Info）。

#### Scenario: 微信 code2session 调用异常
- **WHEN** `Code2SessionAsync` 调用微信 API 抛出异常
- **THEN** 系统 SHALL 记录 Error 级别日志，包含异常信息

#### Scenario: 微信 code2session 返回错误码
- **WHEN** `Code2SessionAsync` 返回非空 ErrCode
- **THEN** 系统 SHALL 记录 Warning 级别日志，包含 errcode 和 errmsg

#### Scenario: 新用户创建
- **WHEN** 微信登录时数据库中不存在该 OpenId 的用户，系统创建新用户
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId 和 openId 前缀

#### Scenario: 禁用账号尝试登录
- **WHEN** 已标记 IsDeleted 的用户尝试登录
- **THEN** 系统 SHALL 记录 Warning 级别日志，包含 userId

#### Scenario: 登录成功
- **WHEN** 用户登录流程完成，Token 已签发
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId 和 isNew 标记

### Requirement: 抽奖核心链路日志

系统 SHALL 在抽奖核心方法 `DrawAsync` 的每个关键决策点记录日志：活动不存在（Warning）、活动未激活（Warning）、每日次数用尽（Info）、参与人数上限（Warning）、防重复中奖触发（Info）、库存为0降级空奖（Info）、库存递减失败（Error）、抽奖结果（Info）。

#### Scenario: 活动不存在
- **WHEN** 用户请求抽奖但 activityId 对应的活动不存在
- **THEN** 系统 SHALL 记录 Warning 级别日志，包含 userId 和 activityId

#### Scenario: 活动未激活
- **WHEN** 用户请求抽奖但活动状态不是 Active
- **THEN** 系统 SHALL 记录 Warning 级别日志，包含 userId、activityId 和当前状态

#### Scenario: 每日抽奖次数用尽
- **WHEN** 用户当日抽奖次数达到 DailyMaxDrawCount
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId、activityId、当日已抽次数和上限

#### Scenario: 参与人数达上限
- **WHEN** 新参与者首次抽奖时 CurrentParticipantCount 已达 MaxParticipantCount
- **THEN** 系统 SHALL 记录 Warning 级别日志，包含 userId、activityId、当前参与人数和上限

#### Scenario: 防重复中奖触发
- **WHEN** AllowRepeatWin 为 false 且用户已中奖，强制返回空奖
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId 和 activityId

#### Scenario: 库存为0降级空奖
- **WHEN** 抽中非空奖项但 RemainCount <= 0，降级为空奖
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 activityId 和原 prizeId

#### Scenario: 库存递减失败
- **WHEN** 乐观更新库存返回影响行数为 0（并发争抢）
- **THEN** 系统 SHALL 记录 Error 级别日志，包含 userId、activityId 和 prizeId

#### Scenario: 抽奖完成
- **WHEN** 抽奖流程成功完成，事务已提交
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId、activityId、prizeId 和 isWin

### Requirement: 活动管理日志

系统 SHALL 在活动创建、活动更新、中奖记录设置收货地址时记录 Info 级别日志。

#### Scenario: 活动创建
- **WHEN** 用户成功创建抽奖活动
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId、activityId 和活动名称

#### Scenario: 活动更新
- **WHEN** 用户成功更新抽奖活动
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId 和 activityId

#### Scenario: 中奖记录设置收货地址
- **WHEN** 用户成功为中奖记录设置收货地址
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 userId、recordId 和 addressId

### Requirement: 退款操作日志

系统 SHALL 在管理员发起退款、同意退款、拒绝退款时记录 Info 级别日志，包含管理员 ID 和订单号。

#### Scenario: 发起退款
- **WHEN** 管理员对已支付订单发起退款申请
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 adminId、orderId 和 outTradeNo

#### Scenario: 同意退款
- **WHEN** 管理员同意退款申请
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 adminId、orderId 和 outTradeNo

#### Scenario: 拒绝退款
- **WHEN** 管理员拒绝退款申请
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 adminId、orderId 和 outTradeNo

### Requirement: 会员管理操作日志

系统 SHALL 在管理员调整用户会员等级和重置用户会员时记录 Info 级别日志，包含管理员 ID、目标用户 ID 和等级变更信息。

#### Scenario: 管理员调整会员
- **WHEN** 管理员创建或更新用户会员等级
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 adminId、targetUserId、原等级和新等级

#### Scenario: 管理员重置会员
- **WHEN** 管理员将用户会员重置为普通会员
- **THEN** 系统 SHALL 记录 Info 级别日志，包含 adminId、targetUserId 和原等级

### Requirement: 商品管理日志

系统 SHALL 在商品创建、更新、删除时记录 Info 级别日志。

#### Scenario: 创建商品
- **WHEN** 管理员成功创建商品
- **THEN** 系统 SHALL 记录 Info 级别日志，包含商品 ID 和名称

#### Scenario: 更新商品
- **WHEN** 管理员成功更新商品
- **THEN** 系统 SHALL 记录 Info 级别日志，包含商品 ID

#### Scenario: 删除商品
- **WHEN** 管理员成功删除商品
- **THEN** 系统 SHALL 记录 Info 级别日志，包含商品 ID

### Requirement: 日志格式一致性

所有新增日志 SHALL 使用 `Utils.Log` 静态类的 `LogInfo`、`LogWarning`、`LogError` 方法，采用中文描述 + 关键参数插值的格式，与现有支付链路日志风格一致。

#### Scenario: 日志格式验证
- **WHEN** 任意新增日志点被触发
- **THEN** 日志消息 SHALL 以中文描述开头，后接关键参数（如 userId、activityId 等），格式与 `Log.LogInfo($"用户 {order.UserId} 会员升级: {oldLevel} -> {newLevel}")` 风格一致
