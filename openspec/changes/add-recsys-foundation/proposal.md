# Proposal

## Why

推荐系统的所有后续阶段（排序、帖找人、想去清单匹配、API）都依赖三样东西：统一的兴趣标签体系、结构化的数据模型，以及可以用来训练和评估的数据。平台目前没有任何真实用户数据，因此必须先建好这三块地基，后续算法才有东西可算、有标准可比。

## What Changes

- 新建 Python 包 `dazi/` 和项目工程骨架（`pyproject.toml`、`.venv`、pytest）。
- 新增**兴趣标签树**：大类 → 子类 → 具体的三层结构、相近兴趣关系，以及两个标签之间匹配层级的计算。初始覆盖新加坡场景下的旅行、运动、桌游、学习、吃喝、文艺、夜生活。
- 新增**领域数据模型**：用户（User）、搭子帖（Post）、想去清单（Wish）、行为事件（Interaction），包括帖子状态流转和字段校验，以及表格化导入导出（pandas DataFrame / Parquet 或 CSV）。
- 新增**模拟器**：用随机种子生成可复现的虚拟新加坡世界，包括带有隐藏真实偏好的用户、帖子、想去清单，以及按时间推进的完整行为链路（曝光 → 浏览 → 申请 → 同意或拒绝 → 成行 → 互评或放鸽子）。同时输出隐藏的真实偏好，供后续阶段做离线评估。
- 提供一个命令行入口，可以一键生成数据集并打印概览统计。

本阶段不包含任何推荐算法、评估指标或 API，它们分别属于 Phase 2–5（见 `docs/DESIGN.md` 第 10 节）。

## Capabilities

### New Capabilities
- `interest-taxonomy`: 兴趣标签树的结构、相近兴趣关系和匹配层级计算，是全系统共享的兴趣词表。
- `domain-model`: 用户、搭子帖、想去清单、行为事件的结构、校验规则、帖子状态流转，以及表格化读写。
- `world-simulator`: 可复现的虚拟世界生成器和行为模拟，输出数据集与隐藏的真实偏好。

### Modified Capabilities
（无，这是项目的第一个 change）

## Impact

- 新增代码：`dazi/taxonomy/`、`dazi/schema/`、`dazi/sim/`、`tests/`、`pyproject.toml`。
- 新增依赖：numpy、pandas、pytest（以及可选的 pyarrow，用于 Parquet）。
- 后续 Phase 2–5 的所有 change 都会基于这三个 capability 构建。
