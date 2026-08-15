## Context

抽奖转盘页面 `pages/lottery/lottery.wxml` 使用九宫格布局，8 个奖品格子环绕中间的"开始抽奖"按钮。当前每个 `prize-cell` 内只有一个 `<text>` 标签显示 `prizes[N].name`，未渲染 `prizes[N].imageUrl`。

后端 `ActivityDetailResponseDto.Prizes` 中的 `ActivityPrizeDto.ImageUrl` 已返回图片 URL，前端 `lottery.js` 中 `fetchActivity` 已将 `activity.prizes` 存入 `data.prizes`，数据链路完整。

格子尺寸为 190rpx × 190rpx，有 4rpx border。

## Goals / Non-Goals

**Goals:**
- 在奖品格子中渲染 `imageUrl` 图片，图片在上、文字在下
- 无图片时保持纯文字居中显示
- 图片加载失败时优雅降级，不影响文字展示

**Non-Goals:**
- 不修改后端 API 或数据模型
- 不修改中奖结果区和中奖弹窗的图片展示（已正常工作）
- 不添加默认占位图（空奖不需要图片，非空奖品无图时保持文字展示）
- 不修改其他页面

## Decisions

### 1. 格子布局改为 `flex-direction: column`

**选择**：将 `prize-cell` 的 `display: flex` 从默认水平排列改为垂直排列。

**理由**：图片需要和名称垂直排列，column 布局最自然。

**替代方案**：保持水平排列，图片在左文字在右——但 190rpx 宽度太小，水平排列会导致文字被挤压。

### 2. 图片尺寸 100rpx × 100rpx

**选择**：图片占据格子上半部分，约 100rpx 见方，留出底部空间给名称。

**理由**：格子总高 190rpx，减去 border 和 padding，100rpx 图片 + 名称文字刚好填满，视觉平衡。

### 3. 使用 `binderror` 处理图片加载失败

**选择**：给 `<image>` 标签绑定 `binderror` 事件，失败时隐藏图片，仅保留文字。

**理由**：避免显示破损图标，优雅降级。需要在 `lottery.js` 中添加 error handler，用 `setData` 标记对应格子的图片加载失败状态。

**替代方案**：纯 CSS 隐藏 `::before` 破损图标——小程序不支持伪元素的 `content` 图片，不可行。

## Risks / Trade-offs

- **[图片比例不一致] → 使用 `aspectFit` 模式**：不同奖品图片宽高比可能不同，`aspectFit` 保证完整显示但可能有留白。这是可接受的 trade-off。
- **[8 个格子同时加载图片] → 图片由后端管理**：转盘通常只有 8 个格子，图片量小，性能影响可忽略。
- **[空奖格子] → 不渲染图片**：空奖通常没有 `imageUrl`，自然走到文字 fallback，无需特殊处理。
