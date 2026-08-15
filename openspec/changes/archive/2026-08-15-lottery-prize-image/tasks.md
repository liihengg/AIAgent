## 1. WXML 修改

- [x] 1.1 在每个 `prize-cell` 中添加 `<image>` 标签，`src` 绑定 `prizes[N].imageUrl`，`mode="aspectFit"`，加 `wx:if="{{prizes[N].imageUrl}}"` 条件渲染
- [x] 1.2 给每个 `<image>` 标签添加 `binderror` 事件绑定，用于图片加载失败时隐藏图片

## 2. WXSS 修改

- [x] 2.1 将 `.prize-cell` 的 `flex-direction` 改为 `column`，使图片和文字垂直排列
- [x] 2.2 添加 `.prize-cell .prize-image` 样式：`width: 100rpx; height: 100rpx; margin-bottom: 8rpx;` 控制图片尺寸
- [x] 2.3 调整 `.prize-name` 样式，确保有图片时文字在底部、无图片时居中（可利用 `flex: 1` 或 `margin-top: auto`）

## 3. JS 修改

- [x] 3.1 添加 `onPrizeImageError` 事件处理函数，接收 `e.currentTarget.dataset.index`，将对应 `prizes[index].imageUrl` 置空并 `setData` 更新
- [x] 3.2 在每个 `<image>` 标签上添加 `data-index` 属性绑定对应奖品索引（0-7）

## 4. 验证

- [x] 4.1 有图片的奖品在转盘格子中正确显示图片+名称
- [x] 4.2 无图片的奖品仅显示名称，文字居中
- [x] 4.3 空奖品格子正常显示名称（空奖通常无图片）
- [x] 4.4 抽奖动画过程中高亮状态正常（active 样式不受图片影响）
- [x] 4.5 图片加载失败时自动降级为纯文字显示
