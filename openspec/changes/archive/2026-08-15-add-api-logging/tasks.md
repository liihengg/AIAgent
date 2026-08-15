## 1. 认证链路日志

- [x] 1.1 在 `WxAuthService.Code2SessionAsync` 中添加 try-catch，捕获异常时记录 `Log.LogError($"微信 code2session 调用异常: {ex.Message}")`
- [x] 1.2 在 `WxAuthService.Code2SessionAsync` 中，当 ErrCode 非空且非 "0" 时记录 `Log.LogWarning($"微信 code2session 错误: errcode={result.ErrCode}, errmsg={result.ErrMsg}")`
- [x] 1.3 在 `AuthController.WxLogin` 中，新用户创建后记录 `Log.LogInfo($"新用户创建: userId={user.Id}, openId={wxSession.OpenId.Substring(0, 8)}...")`
- [x] 1.4 在 `AuthController.WxLogin` 中，检测到 IsDeleted 用户时记录 `Log.LogWarning($"禁用账号尝试登录: userId={user.Id}")`
- [x] 1.5 在 `AuthController.WxLogin` 末尾，登录成功后记录 `Log.LogInfo($"登录成功: userId={user.Id}, isNew={isNew}")`

## 2. 抽奖核心链路日志

- [x] 2.1 在 `ActivityService.DrawAsync` 中，活动不存在时记录 `Log.LogWarning($"抽奖失败-活动不存在: userId={userId}, activityId={activityId}")`
- [x] 2.2 在 `ActivityService.DrawAsync` 中，活动未激活时记录 `Log.LogWarning($"抽奖失败-活动未激活: userId={userId}, activityId={activityId}, status={activity.Status}")`
- [x] 2.3 在 `ActivityService.DrawAsync` 中，每日次数用尽时记录 `Log.LogInfo($"抽奖失败-每日次数用尽: userId={userId}, activityId={activityId}, todayDrawCount={todayDrawCount}, max={activity.DailyMaxDrawCount}")`
- [x] 2.4 在 `ActivityService.DrawAsync` 中，参与人数达上限时记录 `Log.LogWarning($"抽奖失败-参与人数上限: userId={userId}, activityId={activityId}, current={activity.CurrentParticipantCount}, max={activity.MaxParticipantCount}")`
- [x] 2.5 在 `ActivityService.DrawAsync` 中，防重复中奖触发时记录 `Log.LogInfo($"防重复中奖触发: userId={userId}, activityId={activityId}")`
- [x] 2.6 在 `ActivityService.DrawAsync` 中，库存为0降级空奖时记录 `Log.LogInfo($"库存为0降级空奖: activityId={activityId}, prizeId={winningPrize.Id}")`
- [x] 2.7 在 `ActivityService.DrawAsync` 中，库存递减失败（prizeUpdated == 0）时记录 `Log.LogError($"库存递减失败: userId={userId}, activityId={activityId}, prizeId={winningPrize.Id}")`
- [x] 2.8 在 `ActivityService.DrawAsync` 中，事务提交后记录 `Log.LogInfo($"抽奖完成: userId={userId}, activityId={activityId}, prizeId={winningPrize.Id}, isWin={isWin}")`

## 3. 活动管理日志

- [x] 3.1 在 `ActivityService.CreateActivityAsync` 中，事务提交后记录 `Log.LogInfo($"活动创建: userId={userId}, activityId={activity.Id}, name={activity.Name}")`
- [x] 3.2 在 `ActivityService.UpdateActivityAsync` 中，事务提交后记录 `Log.LogInfo($"活动更新: userId={userId}, activityId={activityId}")`
- [x] 3.3 在 `ActivityService.SetDrawRecordAddressAsync` 中，保存后记录 `Log.LogInfo($"中奖记录设置地址: userId={userId}, recordId={recordId}, addressId={addressId}")`

## 4. 退款操作日志

- [x] 4.1 在 `PaymentController.InitiateRefund` 中，状态变更后记录 `Log.LogInfo($"发起退款: adminId={GetUserId()}, orderId={id}, outTradeNo={order.OutTradeNo}")`
- [x] 4.2 在 `PaymentController.ApproveRefund` 中，状态变更后记录 `Log.LogInfo($"同意退款: adminId={GetUserId()}, orderId={id}, outTradeNo={order.OutTradeNo}")`
- [x] 4.3 在 `PaymentController.RejectRefund` 中，状态变更后记录 `Log.LogInfo($"拒绝退款: adminId={GetUserId()}, orderId={id}, outTradeNo={order.OutTradeNo}")`

## 5. 会员管理操作日志

- [x] 5.1 在 `MembershipController.UpsertUserMembership` 中，保存后记录 `Log.LogInfo($"管理员调整会员: adminId={GetUserId()}, targetUserId={userId}, oldLevel={oldLevel}, newLevel={dto.MembershipLevel}")`
- [x] 5.2 在 `MembershipController.DeleteUserMembership` 中，保存后记录 `Log.LogInfo($"管理员重置会员: adminId={GetUserId()}, targetUserId={userId}, oldLevel={oldLevel}")`

## 6. 商品管理日志

- [x] 6.1 在 `ProductService.CreateProductAsync` 中，保存后记录 `Log.LogInfo($"商品创建: productId={product.Id}, name={product.Name}")`
- [x] 6.2 在 `ProductService.UpdateProductAsync` 中，保存后记录 `Log.LogInfo($"商品更新: productId={product.Id}")`
- [x] 6.3 在 `ProductService.DeleteProductAsync` 中，保存后记录 `Log.LogInfo($"商品删除: productId={product.Id}")`

## 7. 补充引用与构建验证

- [x] 7.1 确认 `ProductService.cs` 已有 `using Utils;`（已有），`MembershipController.cs` 和 `PaymentController.cs` 补充 `using Utils;`
- [x] 7.2 运行 `dotnet build` 确认编译通过
