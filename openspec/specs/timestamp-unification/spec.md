## Purpose

统一前后端时间表示为毫秒级 Unix 时间戳（long），消除 DateTime 时区歧义，确保跨平台时间一致性。

## ADDED Requirements

### Requirement: 所有业务时间字段使用毫秒级时间戳

后端所有 Model 和 DTO 中的时间字段 SHALL 使用 `long`（非空）或 `long?`（可空）类型，值为 UTC 毫秒级 Unix 时间戳。数据库列 SHALL 映射为 SQLite INTEGER。

#### Scenario: 创建实体时自动写入时间戳
- **WHEN** 系统创建任何带时间字段的实体（如订单、活动、记录）
- **THEN** 时间字段 MUST 被赋值为 `Clock.Now`（当前 UTC 毫秒时间戳）

#### Scenario: 可空时间字段未赋值时为 null
- **WHEN** 实体的可空时间字段（如 PayTime、ExpireTime）尚未发生
- **THEN** 该字段 MUST 为 `null`，在 JSON 中序列化为 `null`

#### Scenario: API 返回时间字段为 JSON 数字
- **WHEN** 前端调用任何返回时间字段的 API
- **THEN** 时间字段 MUST 序列化为 JSON 数字（如 `1739430600000`），而非 ISO 8601 字符串

#### Scenario: 前端接收时间戳并格式化显示
- **WHEN** 前端收到 API 返回的时间戳数字
- **THEN** 前端 SHALL 使用 `new Date(ts)` 直接构造 Date 对象并格式化，MUST NOT 进行任何乘除法转换

### Requirement: 前端提交时间字段使用毫秒级时间戳

前端向后端提交的任何时间字段（如活动开始时间、结束时间、会员到期时间）SHALL 为毫秒级 Unix 时间戳数字。

#### Scenario: 创建活动时提交时间戳
- **WHEN** 用户在创建活动页面选择日期并提交
- **THEN** startTime 和 endTime MUST 为毫秒级时间戳（通过 `new Date(...).getTime()` 获取）

#### Scenario: 编辑会员到期时间提交时间戳
- **WHEN** 管理员设置会员到期时间并提交
- **THEN** expireTime MUST 为毫秒级时间戳

### Requirement: Clock 提供统一时间源

系统 SHALL 通过 `Clock` 静态类提供所有时间操作，业务代码 MUST NOT 直接使用 `DateTime.UtcNow` 或 `DateTimeOffset.UtcNow`。

#### Scenario: 获取当前时间
- **WHEN** 业务代码需要当前时间
- **THEN** `Clock.Now` SHALL 返回 `long`（UTC 毫秒级时间戳）

#### Scenario: 时间比较
- **WHEN** 业务代码比较时间先后
- **THEN** SHALL 使用 long 数值比较（如 `startTime <= Clock.Now`）

### Requirement: 微信支付时间转换封装在 Clock 内

微信支付 API 要求秒级时间戳和 RFC3339 格式字符串。Clock SHALL 提供转换方法，业务代码 MUST NOT手动拼装时间格式字符串。

#### Scenario: 生成微信支付 time_expire
- **WHEN** 创建微信支付订单需要设置过期时间
- **THEN** `Clock.ToWechatExpire(long ms)` SHALL 返回 `yyyy-MM-ddTHH:mm:ss+08:00` 格式字符串

#### Scenario: 解析微信支付回调时间
- **WHEN** 微信支付回调返回 success_time 字符串
- **THEN** `Clock.ParseWechatTime(string)` SHALL 返回毫秒级时间戳 long

#### Scenario: 生成微信支付签名时间戳
- **WHEN** 构造微信支付请求签名
- **THEN** SHALL 使用秒级时间戳，通过 Clock 提供的方法获取

### Requirement: 前端时间格式化函数接收时间戳

前端 `utils/util.js` 中所有时间格式化函数 SHALL 接收 `long` 类型参数，内部使用 `new Date(ts)` 构造 Date 对象，MUST NOT 进行 `* 1000` 或 `/ 1000` 转换。

#### Scenario: 格式化日期
- **WHEN** 调用 `formatDate(ts)` 传入毫秒时间戳
- **THEN** SHALL 返回 `YYYY/MM/DD` 格式字符串

#### Scenario: 格式化日期时间
- **WHEN** 调用 `formatDateTime(ts)` 传入毫秒时间戳
- **THEN** SHALL 返回 `YYYY-MM-DD HH:MM` 格式字符串

#### Scenario: 格式化到期时间
- **WHEN** 调用 `formatExpireText(ts)` 传入毫秒时间戳
- **THEN** SHALL 返回 `YYYY-MM-DD` 格式字符串，若时间超过当前时间 10 年则返回"永久"

#### Scenario: 判断会员是否过期
- **WHEN** 调用 `isMembershipExpired()` 读取本地存储的 membershipExpireTime
- **THEN** SHALL 直接比较时间戳数值与 `Date.now()`，MUST NOT 构造 Date 对象

### Requirement: 数据库删库重建

迁移方式 SHALL 为删除旧迁移文件和数据库文件后重建，不执行数据迁移。

#### Scenario: 重建迁移
- **WHEN** 实施此变更
- **THEN** SHALL 删除 Migrations/ 目录下所有文件和 lottery.db，生成全新迁移
