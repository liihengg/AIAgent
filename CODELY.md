# CODELY.md

> 本文件为 Codely CLI 提供项目上下文，帮助 AI 快速理解代码库并遵循项目约定。

## 项目概述

LotteryProject 是一个微信小程序抽奖平台，包含前后端两个子项目：

| 子项目 | 路径 | 技术栈 |
|--------|------|--------|
| **LotteryClient** | `Lottery/LotteryClient/` | 微信小程序（WXML / WXSS / JS） |
| **LotteryService** | `LotteryService/LotteryService/` | ASP.NET Core 10 + EF Core + SQLite |

核心功能：用户通过微信小程序登录，创建抽奖活动并分享给好友参与抽奖；支持会员体系、微信支付购买会员商品、收货地址管理、中奖记录管理等。

## 项目结构

```
LotteryProject/
├── Lottery/
│   └── LotteryClient/              # 微信小程序前端
│       ├── app.js / app.json / app.wxss
│       ├── pages/                  # 页面目录
│       │   ├── index/              # 首页（TabBar）
│       │   ├── membership/         # 会员页（TabBar）
│       │   ├── mine/               # 个人中心（TabBar）
│       │   ├── login/              # 微信登录
│       │   ├── profile-setup/      # 资料完善
│       │   ├── activity-create/    # 创建抽奖活动
│       │   ├── activity-list/      # 活动列表
│       │   ├── activity-detail/    # 活动详情
│       │   ├── lottery/            # 抽奖转盘页
│       │   ├── draw-records/      # 抽奖记录
│       │   ├── address-list/      # 收货地址列表
│       │   ├── address-edit/      # 收货地址编辑
│       │   ├── product-list/      # 商品列表
│       │   ├── product-edit/      # 商品编辑
│       │   ├── management-center/ # 管理中心
│       │   └── logs/              # 日志页
│       ├── utils/
│       │   ├── request.js          # HTTP 请求封装（统一加 token、401 拦截）
│       │   ├── api.js              # API 接口定义
│       │   ├── constants.js         # 枚举常量
│       │   ├── util.js             # 工具函数（登录检查、时间格式化）
│       │   └── share.js            # 分享配置
│       └── images/                 # 图标资源
│
└── LotteryService/                 # ASP.NET Core 后端
    ├── LotteryService.sln
    └── LotteryService/
        ├── Program.cs              # 入口：服务注册、中间件、自动迁移
        ├── LotteryService.csproj    # 项目文件（.NET 10）
        ├── appsettings.json        # 配置（连接串、JWT、微信小程序、微信支付）
        ├── LotteryService.http      # API 测试文件
        ├── Controllers/            # API 控制器
        │   ├── AuthController.cs          # 微信登录、资料更新
        │   ├── LotteryController.cs        # 活动管理、抽奖
        │   ├── AddressController.cs        # 收货地址
        │   ├── ProductController.cs       # 商品管理
        │   ├── MembershipController.cs     # 会员配置（Admin）
        │   ├── PaymentController.cs       # 微信支付
        │   └── HelloController.cs
        ├── Models/                 # 数据模型（映射数据库表）
        │   ├── User.cs                    # 用户（含 OpenId/UnionId）
        │   ├── LotteryActivity.cs         # 抽奖活动
        │   ├── LotteryPrize.cs            # 奖品（含权重、库存）
        │   ├── DrawRecord.cs              # 抽奖记录（含收货信息）
        │   ├── UserAddress.cs             # 收货地址
        │   ├── Product.cs                 # 商品
        │   ├── MembershipProduct.cs       # 会员商品（1:1 Product）
        │   ├── MembershipConfig.cs        # 会员等级配置
        │   ├── UserMembership.cs          # 用户会员（1:1 User）
        │   ├── MembershipRecord.cs       # 会员购买记录
        │   ├── PaymentOrder.cs            # 支付订单
        │   ├── ProductType.cs             # 枚举：商品类型
        │   └── ActivityStatus.cs          # 枚举：活动状态
        ├── DTOs/                   # 数据传输对象
        │   ├── Result.cs                  # 统一返回 Result<T>
        │   └── ...（请求/响应 DTO）
        ├── Services/               # 业务服务层
        │   ├── ActivityService.cs         # 活动管理、抽奖逻辑
        │   ├── AddressService.cs         # 地址管理
        │   ├── ProductService.cs          # 商品管理
        │   ├── PaymentService.cs          # 支付订单、回调处理
        │   ├── PaymentNotifyQueue.cs     # 支付回调异步队列
        │   ├── PaymentBackgroundService.cs # 后台轮询查单兜底
        │   ├── TokenService.cs           # JWT 生成
        │   ├── WxAuthService.cs          # 微信 code2session
        │   ├── WechatPayClient.cs        # 微信支付 V3 API 客户端
        │   └── I*Service.cs               # 接口定义
        ├── Data/
        │   └── AppDbContext.cs            # EF Core 上下文（含种子数据）
        ├── Migrations/              # EF Core 迁移文件
        └── Utils/
            ├── Log.cs                      # Serilog 日志工具
            └── WechatPayHelper.cs         # 微信支付签名/验签/解密
```

## 构建与运行

### 后端（LotteryService）

```bash
cd LotteryService

# 还原依赖
dotnet restore

# 运行（启动时自动执行 EF Core 迁移）
dotnet run

# 生成迁移（模型变更时）
dotnet ef migrations add <MigrationName>

# 运行端口
# http://localhost:5297
```

### 前端（LotteryClient）

1. 用微信开发者工具打开 `Lottery/LotteryClient/` 目录
2. AppId: `wx467ba68dccdae782`
3. `utils/request.js` 中的 `BASE_URL` 需指向后端地址（默认 `http://localhost:5297/api`）
4. 开发者工具中关闭"域名校验"或配置合法域名

### 测试

- 后端无单元测试项目；可使用 `LotteryService.http` 文件进行 API 手动测试
- 前端通过微信开发者工具预览/调试

## 代码约定

### 后端（C# / ASP.NET Core）

- **Target Framework**: .NET 10, `Nullable` enable, `ImplicitUsings` enable
- **命名空间**: `LotteryService.{Controllers|Models|DTOs|Services|Data}`，Utils 类在 `Utils` 命名空间
- **文件级命名空间**: 新文件优先使用文件级命名空间（`namespace Foo;`），部分旧文件仍用块级
- **命名风格**: PascalCase（类/属性/方法），私有字段 `_camelCase`
- **控制器约定**:
  - `[ApiController]` + `[Route("api/xxx")]`
  - 非 `auth` 控制器统一加 `[Authorize]`
  - 管理员接口加 `[Authorize(Roles = "Admin")]`
  - 通过 `GetUserId()` 私有方法从 JWT claims 中提取用户 ID
  - 返回 `Result<T>` 包装，失败时返回 `BadRequest`/`NotFound` + `{ code, message }`
- **服务层约定**:
  - 接口 `IXxxService` + 实现 `XxxService`，通过 DI 注入
  - 服务返回 `Result<T>` 表示成功/失败（含 ErrorCode 字符串）
  - 直接使用 `AppDbContext` 进行数据操作（无 Repository 层）
  - 软删除：所有实体含 `IsDeleted` 字段，查询时过滤
  - 时间统一使用 `DateTime.UtcNow`
- **DTO 约定**: 请求和响应各用独立 DTO，避免直接暴露 Model
- **日志**: 使用 `Utils.Log` 静态类（封装 Serilog），不直接引用 `Serilog.Log`
- **JSON 序列化**: 微信支付相关使用 `JsonNamingPolicy.SnakeCaseLower`
- **事务**: 涉及多表写入时使用 `BeginTransactionAsync()` / `ExecuteUpdateAsync` 乐观更新

### 前端（微信小程序）

- **页面结构**: 每个页面一个目录，含 `.js` / `.json` / `.wxml` / `.wxss` 四件套
- **模块系统**: CommonJS（`require` / `module.exports`），不使用 ES Modules
- **缩进**: 2 空格
- **API 调用**: 统一通过 `utils/api.js` 封装的方法调用，不直接使用 `wx.request`
- **HTTP 封装** (`utils/request.js`):
  - 自动注入 `Authorization: Bearer <token>`
  - 401 时自动清除 token 并跳转登录页
  - 导出 `get` / `post` / `put` / `del` 方法
- **登录态**: token 存 `wx.getStorageSync('token')`，用户信息存 `wx.getStorageSync('userInfo')`
- **常量**: 枚举值在 `utils/constants.js` 中定义，前后端需保持一致
- **TabBar**: 首页、会员、我的（3 个 tab）

## API 端点

| 方法 | 路径 | 说明 | 权限 |
|------|------|------|------|
| POST | `/api/auth/wxlogin` | 微信登录 | 公开 |
| PUT | `/api/auth/profile` | 更新用户资料 | 已登录 |
| GET | `/api/lottery/activities` | 获取活动列表 | 已登录 |
| POST | `/api/lottery/activities` | 创建活动 | 已登录 |
| GET | `/api/lottery/activities/{id}` | 获取活动详情 | 已登录 |
| PUT | `/api/lottery/activities/{id}` | 更新活动 | 已登录 |
| POST | `/api/lottery/draw` | 抽奖 | 已登录 |
| GET | `/api/lottery/draw-records` | 获取抽奖记录 | 已登录 |
| PUT | `/api/lottery/draw-records/{id}/address` | 设置中奖记录收货地址 | 已登录 |
| GET/POST/PUT/DELETE | `/api/addresses` | 收货地址 CRUD | 已登录 |
| PUT | `/api/addresses/{id}/default` | 设置默认地址 | 已登录 |
| GET/POST/PUT/DELETE | `/api/products` | 商品 CRUD | GET 公开，写操作 Admin |
| GET | `/api/products/type/{productType}` | 按类型查询商品 | 已登录 |
| GET | `/api/products/category/{category}` | 按分类查询商品 | 已登录 |
| GET/PUT | `/api/membership/configs` | 会员等级配置 | Admin |
| POST | `/api/payment/order` | 创建支付订单 | 已登录 |
| POST | `/api/payment/notify` | 微信支付回调 | 公开（验签） |
| GET | `/api/payment/query/{outTradeNo}` | 查询订单状态 | 已登录 |

## 架构要点

### 认证流程

1. 小程序调用 `wx.login()` 获取 `code`
2. 前端 POST `/api/auth/wxlogin` → 后端调用微信 `code2session` 获取 OpenId
3. 后端查找/创建用户，生成 JWT Token 返回
4. 前端存储 token，后续请求自动携带 `Authorization: Bearer <token>`

### 抽奖算法

- **权重抽奖**: 每个奖品有 `Weight` 字段，按权重比例随机命中
- **库存控制**: `RemainCount` 递减，库存为 0 时降级为空奖
- **防重复中奖**: `AllowRepeatWin` 为 false 时，已中奖用户只能抽到空奖
- **每日限制**: `DailyMaxDrawCount` 限制每日抽奖次数
- **参与人数上限**: 新参与者首次抽奖时原子递增 `CurrentParticipantCount`

### 微信支付

- **支付方式**: JSAPI 支付（小程序内支付）
- **API 版本**: V3（RSA-SHA256 签名）
- **回调处理**: 异步队列模式 — 验签通过后入队，后台服务消费处理
- **兜底机制**: `PaymentBackgroundService` 每 30 秒轮询未支付订单，主动查单，超 10 次未支付则关单
- **防重复下单**: 5 分钟内同用户同商品的 Pending 订单复用 prepay_id
- **证书缓存**: 平台证书每 12 小时刷新，未知序列号触发强制刷新

### 会员体系

- 三级会员: Regular(0) / Member(1) / Premium(2)
- 会员等级决定活动最大参与人数（`MaxParticipantCount`）
- 通过购买会员商品升级（微信支付 → 自动开通）
- `UserMembership` 与 `User` 1:1 关系，UserId 同时为主键和外键

## 数据库

- **数据库**: SQLite（`lottery.db`）
- **ORM**: EF Core 10
- **自动迁移**: `Program.cs` 中 `db.Database.Migrate()` 启动时自动执行
- **种子数据**: `MembershipConfig` 三条初始记录（Regular/Member/Premium）
- **迁移文件**: `Migrations/` 目录
- **连接字符串**: `Data Source=lottery.db`

## 配置

`appsettings.json` 关键配置项：

| 配置节 | 说明 |
|--------|------|
| `ConnectionStrings:DefaultConnection` | SQLite 连接串 |
| `Jwt:SecretKey` | JWT 签名密钥（生产环境应通过环境变量注入） |
| `Jwt:Issuer` / `Jwt:Audience` | JWT 签发者/接收方 |
| `Jwt:ExpireMinutes` | Token 有效期（默认 1440 分钟 = 24 小时） |
| `WxMiniProgram:AppId` / `AppSecret` | 微信小程序凭证 |
| `WeChatPay:MerchantId` | 微信支付商户号 |
| `WeChatPay:ApiV3Key` | 微信支付 APIv3 密钥 |
| `WeChatPay:CertificateSerialNumber` | 商户证书序列号 |
| `WeChatPay:CertificatePrivateKey` | 商户私钥（PEM） |
| `WeChatPay:NotifyUrl` | 支付回调通知地址 |
| `Sentry:Dsn` | Sentry 错误监控 DSN（空则不启用） |

## 安全注意事项

- `appsettings.json` 中的 `SecretKey`、`AppSecret`、`ApiV3Key` 等敏感信息不应提交到版本库
- `*.db` 数据库文件已 gitignore
- JWT `ClockSkew` 设为 `Zero`（不允许时间误差）
- 微信支付回调通过 RSA 验签 + AES-GCM 解密保证安全性
- 支付回调校验：商户号、AppId、金额三重验证 + 幂等检查

## 常用命令

```bash
# 后端
cd LotteryService
dotnet restore                          # 还原依赖
dotnet build                            # 构建
dotnet run                               # 运行（自动迁移）
dotnet ef migrations add <Name>          # 添加迁移
dotnet ef database update                # 更新数据库
dotnet ef migrations list                # 查看迁移列表
dotnet watch run                         # 热重载开发

# Git（两个子项目各自独立 Git 仓库）
cd Lottery && git status                 # 前端
cd LotteryService && git status          # 后端
```

## 注意事项

- **两个独立 Git 仓库**: `Lottery/` 和 `LotteryService/` 各自有独立的 `.git`，根目录 `LotteryProject/` 本身不是 Git 仓库
- **前后端枚举一致性**: `ProductType`、`MembershipLevel`、`ActivityStatus` 等枚举值在后端 C# 和前端 `constants.js` 中需保持一致
- **时间处理**: 后端统一使用 `DateTime.UtcNow`，前端展示时由客户端转换为本地时间
- **EF Core 迁移**: 模型变更后必须生成迁移并提交迁移文件，否则其他环境部署时迁移不生效

## Codely Structured Memories

### User

### Feedback

### Project
- [2026-07-25 15:10:34] seed-products.json drives membership product seeding at startup (Data/ProductSeeder.cs). Products are matched by Id — existing products are updated, new ones are created. Product.Id is manually assigned (ValueGeneratedNever), not auto-increment. Seed product IDs: 高级会员 10001~10999, 企业会员 11001~11999. This allows different deployments to adjust pricing/periods by editing the JSON file without code changes or DB manual operations.

### Reference

