# OppScout 评测方法与结果

## 为什么拆成多层评测

检索系统不能用单一“准确率”代表完整质量。OppScout 分别评估候选召回、来源组覆盖、排序、证据准入、端到端运行和工程回归，并明确每个数字不能证明什么。

## 1. 500-query 历史已知来源评测

### 数据合同

- 500 条历史查询，确定性拆分为 `300 calibration / 100 validation / 100 frozen holdout`。
- 覆盖成人继续教育、职业教育实习和微型电商三个领域。
- 对应 111 个独立已知来源和 2,256 个 query-source 关系。
- 标签只表示“该历史查询曾观察到这个来源”，不是穷举式相关性或负例标签。

### 生产等权 RRF

| 阶段 | Hit@50 | Query-macro known-source recall | MRR |
| --- | ---: | ---: | ---: |
| FTS Top-50 | 36.4% | 11.72% | 6.07% |
| Vector Top-50 | 83.0% | 45.28% | 31.37% |
| RRF Top-50 | **83.8%** | **47.56%** | 22.56% |

RRF 相对 vector-only 的 query-macro recall 提升 `2.28` 个百分点，paired bootstrap 95% CI 为 `[0.98, 3.76]`；但 MRR 下降 `8.81` 个百分点。结论是等权融合扩大了候选覆盖，却削弱了进入 Reranker 前的顺序，不能只展示 Hit@50 而忽略排序损失。

候选召回层 P95 总耗时为 `816.4 ms`，其中向量检索是主要开销；RRF 本身 P95 为 `2.05 ms`。

### 离线权重消融

只在 calibration 上选择出的 FTS/vector 权重为 `0.1/1.0`。它在未触碰的 100-query holdout 上达到：

- Hit@50：`82.0%`
- Macro known-source recall：`47.25%`
- MRR：`34.09%`

该配置仍是离线候选，没有直接替换生产配置；晋升前必须重放人工双审池和同 revision GPU 验收。

## 2. 36-query 人工双标签质量门

### 数据合同

- 36 条锁定 query。
- 1,412 个真实候选，901 个 canonical source group。
- 六类证据：`user_voice / policy / competitor / payment_proxy / compliance / data_source`，每类 6 个 case。
- 每个候选都有 relevance、admission、error 和 note 决策；未审核候选为 0。

### 结果

| 质量门 | 实测 |
| --- | ---: |
| RRF Top-50 macro source-group coverage | **100.00%** |
| BGE Reranker Top-10 macro source-group coverage | **94.33%** |
| Evidence-fit admissible group coverage | **87.98%** |
| Evidence-fit micro conditional precision | **100.00%** |
| Wrong-domain false accept | **0** |
| Authority-mismatch false accept | **0** |

逐候选 recall 为 RRF Top-50 `86.01%`、Reranker Top-10 `69.80%`、evidence-fit 对 admissible gold `74.94%`。验收使用容量归一化的来源组覆盖，避免把同一来源的多个 chunk 或评论重复计算成多份独立证据。

### 延迟边界

在 CPU 环境中，Reranker P95 为 `58.988 s`，不满足交互式延迟要求。CPU 配置只通过质量和离线/批处理稳定性验收；交互式生产需要在 GPU 上保持同一模型及 Top-50/Top-10 合同后重新测量，而不是删除 Reranker 或降低质量门。

## 3. 真实运行与性能归因

在不减少帖子/评论目标和研究合同的前提下，两次同时取得抖音与 B 站正输出的真实运行中，retrieval 阶段相对历史基线 `1127.899 s` 分别变为：

| 运行 | 帖子 / 评论 | Retrieval | 相对基线 |
| --- | ---: | ---: | ---: |
| 历史基线 | 72 / 922 | 1127.899 s | — |
| 验证 A | 87 / 1166 | 556.567 s | **-50.7%** |
| 验证 B | 87 / 838 | 517.636 s | **-54.1%** |

外部平台返回量会变化，因此只把两次双平台正输出视为调度优化的实证，不把不同样本间的 wall-clock 差异外推为稳定 SLA。

最近一次闭合完整业务合同的运行：

- 完整任务耗时 `67m15.3s`，其中包含一次领域校准超时及正常重试。
- 成功重试的候选链为 `28m30.3s`。
- 22 次 LLM 调用，共 668,416 tokens。
- 完成两轮、五位专家、共 10 次专家发言。
- 领域专家质量为 L3 `40/40`，citation/eval passed，`can_promote=true`。
- 通过最终门禁并生成一份 `product_opportunity_card_v2`。

这组数据证明链路能够恢复并闭合，也显示当前成本瓶颈已经从检索调度转移到专家阶段的重复上下文和生成。

## 4. 工程质量

最近一次完整仓库回归为：

- `1,197 passed`
- `17 skipped`
- 决策产物定向回归 `17/17`
- reporting、decision 和 Prompt 相关回归 `192/192`

此外持续执行 Python compile、复杂度边界、依赖边界、Compose 配置和 whitespace 检查。真实 PostgreSQL、浏览器、外部平台和模型调用使用单独的 opt-in/验收门，不与默认快速单元测试混为同一统计口径。

## 5. 指标解释边界

- `Hit@50` 不是 Top-10 准确率，也不是证据准入准确率。
- known-source recall 不把未观察到的结果当作负例，因此不能用于声称全语料 precision。
- Reranker 只负责排序；最终 evidence-fit 才拥有证据准入权。
- 完整任务的 67 分钟和成功候选链的 28 分钟是两个不同窗口，不能互相替换。
- 外部平台状态、模型采样和缓存都会影响 wall-clock，性能结论只在报告所述对照边界内成立。
