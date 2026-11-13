# Spec Delta

## Purpose

定义推荐系统使用的核心领域数据：用户、搭子帖、想去清单、行为事件。包括字段、校验规则、帖子状态流转、数据集一致性检查，以及数据集的表格化读写。

## ADDED Requirements

### Requirement: 用户
用户记录 SHALL 包含：唯一 `id`、年龄段、身份（学生 / 工作）、常住区域（新加坡五大区：Central、East、North、North-East、West）、语言偏好、兴趣列表（兴趣节点 `id` + 可选的水平：入门 / 中级 / 进阶）、每周可用时段（7 天 × 早 / 午 / 晚 / 夜）、预算档位、社交风格（内外向 1–5、节奏 1–5）。性别和「只接受同性别」偏好 MUST 是可选字段。

#### Scenario: 有效用户
- **WHEN** 创建一个所有必填字段合法的用户
- **THEN** 创建成功

#### Scenario: 非法兴趣引用
- **WHEN** 用户兴趣中包含词表里不存在的节点 `id`
- **THEN** 校验失败，错误信息指出该 `id`

#### Scenario: 风格分值越界
- **WHEN** 内外向或节奏的分值不在 1–5 范围内
- **THEN** 校验失败

### Requirement: 搭子帖
搭子帖 SHALL 包含：唯一 `id`、发帖人 `id`、主兴趣节点（可附加次兴趣）、时间、地点、总人数与还缺人数、对搭子的要求（分为硬性和软性）、费用方式、是否室外、语言、描述文本、创建时间、报名截止时间。时间 MUST 是二选一：固定时间段（开始、结束），或弹性窗口（窗口开始、窗口结束、持续天数）。地点 MUST 是二选一：本地（区域 + 可选地铁站），或旅行（目的地兴趣节点 + 出发地）。

#### Scenario: 固定时间的本地帖
- **WHEN** 创建一个羽毛球帖：固定时间段、本地地点、总 4 人、还缺 3 人
- **THEN** 创建成功

#### Scenario: 弹性窗口的旅行帖
- **WHEN** 创建一个清迈旅行帖：窗口为 12 月 10 日到 12 月 25 日、持续 5 天
- **THEN** 创建成功

#### Scenario: 非法时间
- **WHEN** 窗口结束早于窗口开始，或持续天数超过窗口长度，或报名截止晚于活动开始
- **THEN** 校验失败

#### Scenario: 非法人数
- **WHEN** 还缺人数大于等于总人数，或总人数小于 2
- **THEN** 校验失败

### Requirement: 帖子状态流转
搭子帖 SHALL 具有以下状态：`recruiting`（招募中）、`full`（已满）、`completed`（已成行）、`expired`（未凑齐人而过期）、`cancelled`（已取消）。只允许以下流转：`recruiting→full`、`full→recruiting`（有人退出）、`recruiting→expired`、`recruiting→cancelled`、`full→cancelled`、`full→completed`、`recruiting→completed`（人数已达成行下限）。`completed`、`expired`、`cancelled` 为终态。

#### Scenario: 合法流转
- **WHEN** 一个 `recruiting` 的帖子被申请满员
- **THEN** 状态变为 `full`

#### Scenario: 非法流转
- **WHEN** 尝试把 `completed` 的帖子改回 `recruiting`
- **THEN** 操作被拒绝并报错，帖子状态保持不变

### Requirement: 想去清单
想去清单 SHALL 包含：唯一 `id`、用户 `id`、兴趣节点、时间窗口、可选的地点或目的地、预算档位、创建时间、是否有效。它表达的是没有发帖承诺的意向。

#### Scenario: 创建清单
- **WHEN** 用户登记「12 月想去巴厘岛，预算中等」
- **THEN** 生成一条有效的想去清单，兴趣节点指向巴厘岛

### Requirement: 行为事件
行为事件 SHALL 包含：唯一 `id`、类型、时间戳、行为人用户 `id`，并按类型附带帖子 `id` 或目标用户 `id`。类型 MUST 包括：`impression`、`view`、`save`、`apply`、`accept`、`reject`、`withdraw`、`attend`、`no_show`、`review`、`report`、`block`。`impression` MUST 带展示位置和来源（`feed` / `search` / `invite` / `wish`）；`view` MUST 带停留秒数；`review` MUST 带 1–5 分的评分和被评人。

#### Scenario: 曝光事件
- **WHEN** 记录一条缺少展示位置的 `impression`
- **THEN** 校验失败

#### Scenario: 评分越界
- **WHEN** 记录一条评分为 6 的 `review`
- **THEN** 校验失败

### Requirement: 数据集一致性检查
系统 SHALL 提供对整个数据集（用户、帖子、清单、事件）的一致性检查，并返回全部违规项。检查项包括：引用的 `id` 都存在；`accept` / `reject` 的行为人是该帖发帖人；`attend` / `no_show` 只针对被同意的申请人或发帖人本人；`review` 只发生在同一个 `completed` 帖子的实际到场者之间，并且晚于活动结束时间。

#### Scenario: 一致的数据集
- **WHEN** 对模拟器生成的数据集执行一致性检查
- **THEN** 返回零个违规项

#### Scenario: 非发帖人同意申请
- **WHEN** 数据集中存在一条 `accept`，其行为人不是该帖的发帖人
- **THEN** 检查结果列出这条违规，并包含事件 `id`

### Requirement: 时区
所有时间戳 SHALL 是带时区信息的 `Asia/Singapore` 时间。

#### Scenario: 无时区时间
- **WHEN** 用不带时区的时间创建帖子或事件
- **THEN** 校验失败

### Requirement: 数据集读写
系统 SHALL 能把一个数据集（用户、帖子、清单、事件）写入目录，再完整读回。读回的内容 MUST 与写入前一致，并且每张表 MUST 能以 pandas DataFrame 的形式获取，供后续特征计算使用。

#### Scenario: 往返一致
- **WHEN** 把一个数据集写入目录，再从该目录读回
- **THEN** 读回的四张表与原数据集逐条一致（包括时区）
