## Context

当前项目通过 `Utils.Clock` 统一使用 UTC+8 `DateTime`（Kind=Unspecified）。时间字段遍布 Models（10 文件，25+ 属性）、DTOs（14+ 文件，20+ 属性）、Services（6 文件）。前端微信小程序通过 `new Date(str)` 解析后端返回的 ISO 8601 字符串。数据库为 SQLite，DateTime 映射为 TEXT 列。项目处于测试阶段，可直接删库重建。

## Goals / Non-Goals

**Goals:**
- 前后端时间表示统一为 `long`（毫秒级 Unix 时间戳）
- 消除 `Kind=Unspecified` 带来的时区歧义
- 前端 `new Date(ts)` 零转换，消除 `*1000` / `/1000` bug 源
- 微信支付时间转换集中在 Clock 内

**Non-Goals:**
- 不改 JWT exp claim 的处理方式（JWT 标准要求秒级，内部转换即可）
- 不改微信支付签名机制（已经是秒级 Unix timestamp）
- 不做数据库数据迁移（删库重建）
- 不改 EF Core 查询过滤器或索引结构（long 替换 DateTime 后索引照常工作）

## Decisions

### 1. 毫秒而非秒

**选择**: 毫秒级时间戳（13 位）

**理由**: JavaScript `Date.now()` 和 `new Date(ts)` 原生使用毫秒。用毫秒时前端零转换，用秒则前端 6 处需 `*1000`，每处遗漏都是灾难性 bug。后端仅 3 处需 `/1000`（微信支付签名、JWT exp、微信 time_expire），全部封装在 Clock 内，业务代码无感知。

**备选**: 秒级时间戳——微信支付和 JWT 天然是秒，但前端转换点更多且更分散。

### 2. Clock 新 API 设计

```
Clock.Now              → long (UTC 毫秒)
Clock.ToUnixSeconds(long ms) → long (秒, 供微信支付/JWT 内部使用)
Clock.FromUnixSeconds(long s) → long (毫秒, 解析微信返回的秒级时间)
Clock.ToWechatExpire(long ms) → string "yyyy-MM-ddTHH:mm:ss+08:00"
Clock.ParseWechatTime(string) → long (毫秒)
```

**理由**: 业务代码只接触 `Now` 和比较运算。微信支付的秒级/字符串转换全在 Clock 内部，Service 层调用 Clock 方法而不直接做算术。

### 3. 日期加减运算改写

DateTime 的 `.AddMinutes(N)` / `.AddDays(N)` 改为 long 算术：

```
Clock.Now.AddMinutes(30)  →  Clock.Now + 30 * 60_000L
Clock.Now.AddDays(30)     →  Clock.Now + 30 * 86_400_000L
Clock.Now.AddMinutes(-5)  →  Clock.Now - 5 * 60_000L
```

**备选**: 定义常量辅助——`TimeSpan.FromMinutes(30)` 转 long。但直接算术更清晰，且项目内加减场景有限（PaymentService 5 分钟窗口、30 分钟超时；MembershipRecord 按天加）。比较运算（`<=`、`>=`、`>`、`<`）完全不变。

### 4. 前端 util.js 重写策略

```
formatDate(ts)       → new Date(ts) → YYYY/MM/DD
formatDateTime(ts)   → new Date(ts) → YYYY-MM-DD HH:MM
formatExpireText(ts) → new Date(ts) → YYYY-MM-DD 或 "永久"
isMembershipExpired() → ts <= Date.now()  (直接数值比较)
```

调用处参数类型从 string 变 long，但函数签名不变，大部分页面调用处**不用改**。仅 `activity-create.js` 和 `membership-list.js` 需要改时间提交逻辑：从拼字符串改为 `new Date(...).getTime()`。

### 5. 删库重建而非迁移

删除 `Migrations/` 下所有文件 + `lottery.db`，执行 `dotnet ef migrations add InitialTimestamp`。

**理由**: 项目处于测试阶段，无生产数据。EF Core 迁移文件中 DateTime→long 的 column 变更会产生复杂的 ALTER TABLE，SQLite 对 ALTER 支持有限，不如重建干净。

### 6. TokenService 和 WechatPayClient 的例外处理

```
TokenService:
  旧: DateTime.UtcNow.AddMinutes(expireMinutes)
  新: Clock.Now + expireMinutes * 60_000L  (JWT exp claim 内部用 Clock.ToUnixSeconds 转秒)

WechatPayClient:
  旧: _certDownloadTime: DateTime
  新: _certDownloadTime: long (Clock.Now)
  旧: DateTimeOffset.UtcNow.ToUnixTimeSeconds()
  新: Clock.ToUnixSeconds(Clock.Now)
```

**理由**: 这两处是唯一使用 UTC 的代码。改为通过 Clock 统一，保持一致性。JWT exp 最终是秒级 Unix timestamp，这是 JWT 标准，不可改变。

## Risks / Trade-offs

- **[SQLite CLI 调试需 /1000]** → 可接受。`datetime(ts/1000, 'unixepoch')` 仅手动调试时使用，不进入代码路径。
- **[long 默认值 0 的语义]** → `0` 代表 1970-01-01，不像 `DateTime.MinValue` 那么明显。非空字段在创建时必须显式赋值 `Clock.Now`，可空字段用 `long?`。Service 层校验时检查 `> 0`。
- **[前后端需同步发布]** → API 时间字段从字符串变为数字，前后端不同步会导致解析失败。部署时需同时更新。
- **[日志可读性下降]** → long 数字不可读。可选在 Clock 中提供 `Format(long)` 方法用于日志，但非必须。
