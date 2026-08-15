## Why

抽奖转盘页面的 8 个奖品格子只显示奖品名称文字，没有渲染奖品图片。后端 `ActivityPrizeDto.ImageUrl` 已经返回了图片 URL，前端 `lottery.wxml` 的 `prize-cell` 中只绑定了 `prizes[N].name`，缺少 `<image>` 标签。这是抽奖页面的核心视觉区域，缺少图片严重影响用户体验。

## What Changes

- 在 `lottery.wxml` 的每个 `prize-cell` 中添加 `<image>` 标签渲染 `prizes[N].imageUrl`
- 当 `imageUrl` 为空时，显示文字奖品名称作为 fallback（保持现有行为）
- 当 `imageUrl` 存在时，图片在上、名称在下，形成图文混排布局
- 调整 `lottery.wxss` 中 `prize-cell` 的布局为 `flex-direction: column`，适配图片+文字的垂直排列

## Capabilities

### New Capabilities

- `lottery-draw`: 抽奖转盘页面的奖品展示行为

### Modified Capabilities

（无）

## Impact

- **前端文件**: `pages/lottery/lottery.wxml`、`pages/lottery/lottery.wxss`
- **后端**: 无需改动，数据已就绪
- **无 breaking change**：纯 UI 增强，不影响现有 API 和数据流
