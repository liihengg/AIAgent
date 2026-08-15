## Purpose

抽奖转盘页面的奖品格子展示行为，包括奖品图片渲染和文字 fallback 机制。

## ADDED Requirements

### Requirement: 转盘格子展示奖品图片

当奖品有 `imageUrl` 时，抽奖转盘的每个奖品格子 SHALL 在格子内渲染奖品图片。图片 SHALL 显示在奖品名称上方，形成图片在上、文字在下的垂直布局。图片 SHALL 使用 `aspectFit` 模式以完整展示。

#### Scenario: 奖品有图片 URL

- **WHEN** 活动详情接口返回的奖品数据中 `imageUrl` 不为空
- **THEN** 转盘格子内渲染 `<image>` 标签显示该 URL 的图片，并在图片下方显示奖品名称

#### Scenario: 奖品无图片 URL

- **WHEN** 奖品数据中 `imageUrl` 为空字符串
- **THEN** 转盘格子内不渲染 `<image>` 标签，仅显示奖品名称文字（保持现有行为）

#### Scenario: 奖品图片加载失败

- **WHEN** 奖品 `imageUrl` 不为空但图片加载失败
- **THEN** 图片区域不显示破损图标，奖品名称文字仍正常显示

### Requirement: 转盘格子布局适配图文混排

转盘格子的布局 SHALL 改为垂直方向（`flex-direction: column`），以支持图片在上、文字在下的排列。当有图片时，图片占据格子上部区域，名称在底部；当无图片时，名称居中显示。

#### Scenario: 有图片时格子布局

- **WHEN** 奖品格子内同时包含图片和名称
- **THEN** 图片显示在格子偏上区域，名称显示在格子底部，两者垂直排列不重叠

#### Scenario: 无图片时格子布局

- **WHEN** 奖品格子内只有名称无图片
- **THEN** 名称在格子中居中显示，与当前行为一致
