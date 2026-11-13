# Design

## Context

项目目前是空的，只建好了 `.venv`（Python 3.12）。动机见 proposal.md 的 Why 部分，整体架构见 `docs/DESIGN.md`。本 change 产出的三个 capability（见 `specs/`）会被后续每个阶段使用，所以设计上优先考虑：**接口稳定、数据可复现、行为可解释**，而不是性能或功能的完整性。

## Goals / Non-Goals

**Goals:**
- 后续阶段只依赖 `dazi.taxonomy`、`dazi.schema`、`dazi.sim` 的公开接口，不依赖它们的内部实现。
- 模拟器的「真实世界规则」简单、可读、集中在一个地方，便于以后调整和审视。
- 默认规模（2000 个用户、84 天）的完整模拟在笔记本上 1 分钟内跑完。

**Non-Goals:**
- 不做任何推荐、排序或评估指标（Phase 2 才做）。
- 不追求逼真到可以拿来预测真实市场；模拟器只需要「方向正确、有足够信号」。
- 不做数据库存储；Phase 5 做 API 时再引入。

## Decisions

### 1. 包结构

```
dazi/
  taxonomy/   data/interests.toml, loader.py, match.py
  schema/     enums.py, models.py, dataset.py, consistency.py
  geo.py      新加坡五大区、区域中心坐标、区域间通勤时间矩阵、少量地铁站
  sim/        config.py, truth.py, world.py, engine.py, policy.py, __main__.py
tests/
```
`geo.py` 单独放在包的顶层，因为后续的召回和排序也要用通勤时间。

### 2. 词表用 TOML 文件，标准库 `tomllib` 读取
- 选择理由：TOML 适合手工编辑中文，支持注释，并且 `tomllib` 是 Python 3.11 起的标准库，不需要额外依赖。
- 备选方案：JSON 不能写注释、手改容易出错；YAML 需要额外的 pyyaml 依赖。
- 节点 `id` 使用英文 slug 路径，例如 `travel.sea.chiang_mai`、`sport.badminton`。这样 `id` 本身就能看出层级，日志里也好读；但层级关系仍以 `parent` 字段为准，不靠解析 `id` 字符串。

### 3. 领域模型：单条记录用 pydantic，数据集用 DataFrame
- 用户、帖子、清单这些数量少、结构复杂的记录用 pydantic v2 模型做单条校验。Phase 5 的 FastAPI 也基于 pydantic，可以直接复用这些模型。
- 行为事件数量很大（默认规模下约 100 万条以上），逐条构造 pydantic 对象太慢。所以事件以行字典的形式累积，最后转成 DataFrame，再用**向量化校验**（枚举、按类型必填的字段、取值范围）。
- `Dataset` 对象持有 `users / posts / wishes / interactions` 四张 DataFrame，是全系统的标准数据形态。
- 需要把 pydantic 加入允许的依赖（FastAPI 本来就依赖它），并同步更新 `openspec/config.yaml`。
- 备选方案：标准库 dataclasses 加手写校验。这样到 Phase 5 还得再写一套 API 模型，等于重复劳动。

### 4. 存储用 Parquet（pyarrow）
- 选择理由：保留类型和时区信息，读写快，文件小。
- 嵌套字段（兴趣列表、要求、可用时段）在落盘时序列化为 JSON 字符串列，读回时再解析。这样可以保证往返一致，不受 pyarrow 对嵌套类型处理差异的影响。
- 备选方案：CSV 会丢失类型和时区；pickle 不跨版本，也不便于其他工具查看。

### 5. 模拟器：按天推进的离散时间循环

每个模拟日依次执行：
1. **生成供给**：每个用户按活跃度和真实喜好度，以泊松过程发帖、登记清单。
2. **曝光**：按活跃度抽取当天的活跃用户，由曝光策略为每人选出 K 个招募中的帖子。
3. **用户反应**：用真实申请概率抽样，决定是否浏览、收藏、申请。
4. **发帖人处理**：发帖人在 0–2 天内按真实兼容度和挑剔程度同意或拒绝申请；满员后转为 `full`。
5. **结算**：对当天开始的活动，按放鸽子倾向抽样到场情况，再判定 `ongoing` 或 `expired`；对当天结束的 `ongoing` 活动，置为 `completed`。多天活动（如旅行）在开始与结束之间保持 `ongoing`。
6. **互评**：活动**结束**后的第二天，到场者两两按真实兼容度加噪声打分。

随机性全部来自同一个 `numpy.random.Generator(seed)`，按固定顺序消耗，从而保证可复现。

### 6. 真实世界规则集中放在 `truth.py`
- **兴趣喜好度**：每个用户先用 Dirichlet 分布选 2–4 个偏好的大类，再逐层向下分配子类和具体节点的喜好度；相近兴趣按权重扩散一部分喜好度。
- **申请概率**：`sigmoid(b0 + w_int·喜好度 + w_time·时间契合 − w_dist·通勤 + w_budget·预算匹配 + w_style·风格匹配 + 帖子吸引力 − 疲劳)`，所有权重集中写成常量。
- **两两兼容度**：由风格距离、年龄段、语言、共同兴趣，再加一个由 `hash(seed, min(u,v), max(u,v))` 决定的确定性噪声组成。这样对同一对用户的查询结果总是相同。
- 查询接口 `Oracle` 从隐藏数据目录加载，只给评估代码使用。

### 7. 曝光策略接口
```python
class ExposurePolicy(Protocol):
    def select(self, user_id: str, now: datetime, candidates: pd.DataFrame, k: int) -> list[Exposure]: ...
```
`candidates` 只包含可观测字段，防止推荐算法偷看隐藏数据。`Exposure` 包含帖子 `id`、位置和来源。默认实现是 `RandomPolicy`。

### 8. 输出目录结构
```
data/sim_seed42/
  users.parquet  posts.parquet  wishes.parquet  interactions.parquet  meta.json
  _truth/        affinity.parquet  user_latent.parquet  config.json
```
`data/` 加入 `.gitignore`。

## Risks / Trade-offs

- [后续算法拟合的是模拟器的假设，而不是真实用户] → 真实规则保持简单并写进文档；推荐算法不允许导入 `truth.py`；评估时使用多个种子和配置；上线后用真实数据重新校准。
- [合理性范围靠手动调常量才能满足] → 用合理性测试锁定范围，调参只改 `truth.py` 里的常量。
- [默认规模下模拟太慢] → 用户反应一步做向量化；如果仍然太慢，就降低每天的曝光数。规模参数都可以配置。
- [嵌套字段存成 JSON 字符串后，用 pandas 查询不方便] → `Dataset` 提供辅助函数，返回已解析的视图。

## Open Questions

- 地铁站是否需要覆盖全部站点？目前只放少量代表站，后续做召回时如果需要更细的距离，再补充数据，不影响本阶段的规格。
