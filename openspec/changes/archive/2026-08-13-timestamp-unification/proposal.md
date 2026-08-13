## Why

当前项目所有时间字段使用 `DateTime` (UTC+8, Kind=Unspecified)，存在时区歧义、JSON 序列化跨时区错误、SQLite TEXT 范围查询低效等问题。前端 JS `new Date(str)` 依赖字符串解析行为，非东八区设备会显示错误时间。改为毫秒级 Unix 时间戳后，时间成为绝对值，前后端零歧义。

## What Changes

- **BREAKING**: 所有 Model 实体时间字段 `DateTime` / `DateTime?` → `long` / `long?`（毫秒级 Unix 时间戳）
- **BREAKING**: 所有 DTO 时间字段 `DateTime` / `DateTime?` → `long` / `long?`
- **BREAKING**: API 输入输出时间格式从 ISO 8601 字符串变为 JSON 数字（如 `1739430600000`）
- 重写 `Utils/Clock`：`Now` 返回 `long`（毫秒），新增微信支付专用转换方法
- 所有 Service 层日期加减运算改为 long 算术（比较运算不变）
- 前端 `utils/util.js` 时间格式化函数改为接收 `long`，内部 `new Date(ts)` 直接使用
- 删除旧 EF Core 迁移文件和 `lottery.db`，重建迁移
- `AppDbContext` 种子数据改为 long 常量
- 微信支付 `time_expire`、签名 timestamp、JWT exp 保持秒级，通过 Clock 封装转换

## Capabilities

### New Capabilities
- `timestamp-unification`: 统一时间表示为毫秒级 Unix 时间戳，覆盖后端 Model/DTO/Service、前端工具函数、API 契约

### Modified Capabilities
（无现有 spec）

## Impact

- **后端 Models**（10 文件）: 25+ 个 DateTime 属性改为 long
- **后端 DTOs**（14+ 文件）: 20+ 个 DateTime 属性改为 long
- **后端 Services**（6 文件）: Clock.Now 赋值、日期加减运算重写
- **后端 Utils/Clock.cs**: 完全重写
- **后端 Migrations**: 全部删除重建
- **后端 AppDbContext**: 种子数据改为 long
- **前端 utils/util.js**: 4 个格式化函数重写
- **前端 activity-create.js**: 时间拼装改为 getTime() 输出
- **前端 membership-list.js**: 时间拼装改为 getTime() 输出
- **前端 ~10 个页面 JS**: 调用格式化函数处参数类型从 string 变 long（函数内部已处理，调用处基本不变）
- **API 契约**: 时间字段从字符串变为数字，前后端需同步发布
- **数据库**: 删库重建，无数据迁移
