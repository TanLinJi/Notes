# 点云 Test-Time Adaptation 方向深度调研

日期：2026-05-25  
输出位置：`/root/autodl-tmp/Test/point_cloud_tta_deep_research.md`  
调研模式：deep-research / literature review + research-roadmap hybrid  
本地上下文：已用 qmd 在 `autodl-workspace` 中检索，并结合 `/root/autodl-tmp/MCP-Point-Cache` 的 Point-Cache/MCP-3D 计划。

## 1. 执行摘要

点云 TTA 正在从“把 2D TTA 的熵最小化/BN 更新直接搬到 3D”转向更 3D-specific 的方向：几何结构修复、局部-全局层次缓存、原型/记忆库、开词表多模态适配、以及低成本或无反向传播的在线适配。对当前 `/root/autodl-tmp/MCP-Point-Cache` 的研究线来说，最值得押注的是：

1. 以 Point-Cache 为强基线，做 open-vocabulary / training-free 的点云识别 TTA。
2. 必须纳入 2026 AAAI 的 Uni-Adapter 作为新近强相关基线。它已经做了 3D VLFM 的 training-free online prototype cache，因此后续创新不能只停留在“动态原型缓存”。
3. 将 MCP 的 entropy / align / negative multi-cache 思路迁移到 3D，但不能只做“文本换点云特征”的平移，需要加入几何可靠性、局部结构和 corruptions 类型差异。
4. 把“低熵样本不一定可靠”作为核心问题，用 3D compactness、ICP/Chamfer residual、局部几何一致性、文本锚点一致性来改进 cache admission、prototype update 和 prediction calibration。
5. 优先保持无训练或轻量在线更新，因为 T4 机器和大 3D-VLM backbone 对全量反传不友好。

一句话选题建议：

> Geometry-Aware Multi-Cache Test-Time Adaptation for Open-Vocabulary Point Cloud Recognition

它与 Point-Cache、MCP 和当前本地计划的交集最大，技术风险可控，实验可以从 ModelNet-C / ScanObjectNN-C 快速闭环。

## 2. 调研问题与范围

研究问题：

1. 点云 TTA 的主要方法谱系是什么？
2. 当前分类、语义分割、检测、注册、补全等任务中，哪些方向已经拥挤，哪些仍有空间？
3. 在 open-vocabulary 3D recognition 和 3D-VLM 场景下，Point-Cache + MCP-style multi-cache 是否有研究价值？
4. 面向 `/root/autodl-tmp/MCP-Point-Cache`，最现实的实验路线和风险门是什么？

范围说明：

- 重点：3D point cloud recognition / classification / open-vocabulary recognition。
- 次重点：点云 semantic segmentation、LiDAR scene understanding、registration、upsampling。
- 只作为基础背景：通用 2D TTA、CLIP/VLM TTA。
- 不做严格 PRISMA 系统综述，也不做 meta-analysis。本文是用于选题、Related Work 和实验规划的深度调研。

## 3. 本地项目上下文

qmd 和文件阅读显示，本机已有三条直接相关线索：

- `/root/autodl-tmp/MCP-Point-Cache/Point-Cache/README.md`：Point-Cache 官方实现，本地说明其为 CVPR 2025 论文 “Point-Cache: Test-time Dynamic and Hierarchical Cache for Robust and Generalizable Point Cloud Analysis”，训练自由、层次 cache、面向 open-vocabulary point cloud recognition，支持 ULIP、ULIP-2、OpenShape、Uni3D，并在 ModelNet-C、ScanObjectNN-C、OmniObject3D、Objaverse-LVIS 等 benchmark 上验证。
- `/root/autodl-tmp/MCP-Point-Cache/MCP/README.md`：MCP 官方实现，本地说明其为 ICCV 2025 “Multi-Cache Enhanced Prototype Learning for Test-Time Generalization of Vision-Language Models”，核心是 entropy cache、align cache、negative cache 和 MCP++ residual tuning。
- `/root/autodl-tmp/MCP-Point-Cache/progress.txt`：已有 AAAI 2027 到 CVPR 2027 的实验计划，包括 Point-Cache 复现、OpenShape PointBERT-vitg14-rgb、ModelNet-C/ScanObjectNN-C、3D-MCP/TDA/TPT、vMF text anchor、ICP+Chamfer、6-cell memory、compactness-performance diagnosis、clean control、多 seed 和效率分析。

这意味着调研结论不应停留在“点云 TTA 有哪些论文”，而应回答：怎样把 MCP-style cache 变成一个 3D-specific、可复现、可投稿的 open-vocabulary point cloud TTA 方法。

## 4. 文献地图

### 4.1 通用 TTA 基础

| 方法 | 核心思想 | 对点云 TTA 的启发 | 局限 |
| --- | --- | --- | --- |
| TTT, ICML 2020 | 测试时用自监督任务更新模型 | 点云可用重建、旋转、遮蔽补全等辅助任务 | 需要训练阶段设计辅助头，source model 不一定可改 |
| Tent, ICLR 2021 | 测试时熵最小化，主要更新 BN/归一化参数 | 简单强基线，可用于 closed-set 分类 | 小 batch、类别偏移、open-vocab 和 corruptions 下可能自信错误 |
| CoTTA, CVPR 2022 | 连续测试时适配，teacher-student 和 stochastic restoration | 对 online stream / non-stationary 点云场景有价值 | 维护和调参成本高，可能牺牲干净分布表现 |
| SAR, ICLR 2023 | 可靠熵最小化 + sharpness-aware 策略 | 可作为“低熵不可靠”的理论和实验对照 | 仍偏梯度更新，和 training-free 3D-VLM 不完全匹配 |

通用结论：entropy minimization 是必要基线，但不是足够的 3D 方案。点云 corruptions 常改变局部密度、姿态、缺失、噪声和采样分布，低熵样本可能只是模型过度自信。后续 3D 方法大多在解决这个问题。

### 4.2 点云分类 / open-vocabulary recognition

| 方法 | 年份/ venue | 主要方向 | 价值判断 |
| --- | --- | --- | --- |
| MATE | ICCV 2023 | masked autoencoder 作为 online 3D test-time learner | 代表“自监督重建/MAE”路线，适合 closed-set 3D backbone，但对 OpenShape/ULIP 这类 3D-VLM 的直接迁移成本较高 |
| CloudFixer | ECCV 2024 | diffusion-guided geometric transformation，在输入空间修复点云 | 很 3D-specific，适合 corruption 修复；但引入 diffusion/score model 后效率和实现复杂度上升 |
| Point-Cache | CVPR 2025 | dynamic hierarchical cache，global + local part clues，training-free，支持 large multimodal 3D models | 与本地计划最吻合，是必须复现和比较的主基线 |
| Uni-Adapter / Adapt-As-You-Walk Through the Clouds | AAAI 2026 | 3D VLFM 的 training-free online TTA，动态 cluster prototype cache + graph label smoothing + entropy-weighted aggregation | 与本项目高度重叠，是必须正面对比的新强基线 |
| Test-Time Adaptation in Point Clouds: Leveraging Sampling Variation with Weight Averaging | WACV 2025 | 利用采样变化和权重平均稳定适配 | 可作为轻量、效率友好的对照；适合检验“随机采样本身带来 test-time ensemble/adaptation” |
| Purge-Gate | 2025 preprint | backprop-free point-cloud classification TTA | 与 training-free/高效方向一致，值得作为 Related Work 和效率对照 |
| SMART-PC | ICML 2025 | skeleton model adaptation for robust test-time training in point clouds | 偏模型结构/骨架适配，说明 2025 后 3D TTA 开始关注结构先验 |
| GSDTTA | 2025 preprint | graph spectral driven point shift | 属于几何/图谱驱动的输入或特征修复路线，和 ICP/Chamfer 思路可对话 |

判断：分类方向已经有强基线，但 open-vocabulary + multimodal + training-free + geometry-aware multi-cache 仍有空间。Point-Cache 解决了“层次 cache”，Uni-Adapter 解决了“3D VLFM 动态原型 cache”，MCP 解决了“低熵 cache 不可靠、需要 align/negative cache”。后续关键新意应落在 3D-specific cache quality 和 negative/alignment calibration，而不是简单增加 cache 数量或重做动态 prototype。

### 4.3 点云语义分割 / LiDAR scene understanding

| 方法 | 任务 | 核心思想 | 对本项目的启发 |
| --- | --- | --- | --- |
| HGL: Hierarchical Geometry Learning | 3D point cloud semantic segmentation | 利用层次几何学习做 TTA | 说明 segmentation 中局部-全局几何更关键，可借鉴到 hierarchical cache 的局部 part 表征 |
| APCoTTA | airborne LiDAR semantic segmentation | 针对 geospatial/airborne LiDAR 的 continual TTA | 说明真实 LiDAR shift 与 ModelNet-C 类 synthetic corruption 不同，后续可做泛化论证 |
| PCoTTA | multi-task point cloud understanding | continual TTA + 多任务 | 如果将来从分类扩到 segmentation/detection，可以用它作为 online continual protocol 参考 |

判断：语义分割方向更贴近真实自动驾驶和机器人，但实验成本高、数据复杂、评价慢。当前更适合先把 object-level point cloud recognition 做扎实，再把局部几何/scene stream 作为 CVPR phase extension。

### 4.4 生成式或输入空间适配

CloudFixer、3DD-TTA、GSDTTA 等方法共同说明：点云 TTA 不能只更新模型参数，直接在输入点集上做几何修复、位移、denoising、diffusion-guided transformation 也有效。

对本地计划的启发：

- ICP + Chamfer residual 可以作为 cache quality 或 failure detection，而不一定直接作为主修复器。
- 输入空间修复对 rotation、jitter、scale、dropout 的收益可能差异很大，必须 per-corruption 统计。
- 如果 diffusion 类方法作为 baseline，可能会造成环境和效率负担。建议先做“轻量几何可靠性分数”，再考虑复杂修复。

### 4.5 多模态 / 原型 / cache 适配

2D VLM TTA 中，TPT、TDA、MCP 等方法把 CLIP 的 text/image alignment、prompt 或 cache prototype 用于 source-free test-time generalization。Point-Cache 已把 cache 思路带到 open-vocabulary 3D recognition，但 MCP 提出的一个核心批评对 3D 仍成立：低熵样本不一定形成 compact 和可靠的类内原型。

可迁移到 3D 的 MCP 组件：

1. Entropy cache：用低熵样本快速启动，但不能单独作为可靠性标准。
2. Align cache：在 3D point feature 与 text feature 之间做类内 compactness 和跨模态一致性约束。
3. Negative cache：用高熵或冲突样本做 prediction calibration，抑制污染 cache 的错误类别。
4. Residual tuning：可以作为高风险 extension，不建议第一阶段依赖它。

3D-specific 改造点：

- 低熵 + 高 text similarity 不足以判断可靠性，要加入几何质量。
- 全局点云特征和局部 part 特征应分开维护 cache。
- rotation/dropout/jitter 等 corruption 对 cache admission 的影响不同，应有 corruption-aware 或 self-diagnosed 的权重。
- open-vocabulary 类别的文本锚点可用多模板、LLM paraphrase、vMF MAP 或 Bayes-style text aggregation 做稳定化。

## 5. 方法谱系总结

| 谱系 | 代表方法 | 适用场景 | 对本地项目的优先级 |
| --- | --- | --- | --- |
| Entropy / BN / normalization update | Tent, SAR | closed-set, batch 足够, 可反传 | 中。必须作为思想背景和部分基线，但不应作为主方法 |
| Self-supervised auxiliary task | TTT, MATE | 有辅助头或可重建的 3D backbone | 中。Related Work 必须覆盖，复现成本看代码兼容性 |
| Continual online adaptation | CoTTA, PCoTTA, APCoTTA | 数据流、non-stationary shift、segmentation | 中低。可作为 protocol extension |
| Input-space geometric repair | CloudFixer, 3DD-TTA, GSDTTA | corruption 修复，尤其噪声/缺失/旋转 | 中。可借几何可靠性，不建议一开始引入重 diffusion |
| Cache / prototype / memory | Point-Cache, TDA, MCP | open-vocabulary、training-free、VLM | 最高。本地项目主线 |
| Backprop-free / efficient TTA | Purge-Gate, BFTT3D, Point-Cache | 边缘设备、T4、实时在线 | 高。能强化方法卖点 |
| Multi-modal prompt/text anchor | TPT, MCP, 3D-VLM methods | OpenShape/ULIP/Uni3D | 高。适合 vMF text anchor 和 negative cache |

## 6. Benchmark 与协议建议

### 6.1 数据集

优先级 1：

- ModelNet-C / ModelNet40-C：object-level synthetic corruption，适合快速复现和 per-corruption 分析。
- ScanObjectNN-C：更接近真实扫描噪声，能检验是否只对 synthetic CAD 有效。
- ModelNet40 / ScanObjectNN clean：必须做 clean control，防止 TTA 损害正常分布。

优先级 2：

- OmniObject3D：类别和形状更丰富，可做泛化。
- Objaverse-LVIS：更适合 open-vocabulary long-tail，但实验规模和文本类别处理更重。
- PointDA-10 / Sim2Real-SONN：可用于 domain generalization 或 sim-to-real 论证。

优先级 3：

- SemanticKITTI、nuScenes、SynLiDAR：适合 LiDAR segmentation/detection extension。
- ShapeNet-C：可作为额外 corruption benchmark。

### 6.2 协议必须写清

| 协议项 | 建议 |
| --- | --- |
| Source-free | 不访问 source training data，不用 source labels |
| Online stream | 明确是否单样本顺序到达，cache 是否跨样本累积 |
| Episodic vs continual | Episodic 每个 corruption reset；continual 不 reset。两者不能混表 |
| Open-vocabulary | 明确 seen/unseen class split，文本 prompt 是否只含 class name |
| Transductive batch | 如果一次看到整个 test set，必须单独标注，不能和 online 混同 |
| Random order | online cache 对顺序敏感，至少 3 个 stream order seed |
| Clean control | 每个方法都要报告 clean accuracy drop |
| Efficiency | latency、GPU memory、cache size、每样本额外计算 |

### 6.3 指标

- Accuracy / mean accuracy。
- mCE 或 per-corruption relative error，如果 benchmark 支持。
- Clean drop：clean accuracy 相比 zero-shot 的下降。
- Per-corruption gain：add_global、add_local、dropout_global、dropout_local、rotate、scale、jitter 等分别看。
- Calibration：ECE / NLL / entropy distribution。
- Cache quality：cache hit rate、admission rate、purity proxy、prototype compactness Phi。
- Geometry diagnostics：ICP success rate、Chamfer residual、PCA alignment residual。
- Efficiency：ms/sample、peak VRAM、cache memory、是否需要 backward。

## 7. 研究空白与机会

### 7.1 低熵不等于可靠

MCP 在 2D VLM 中已经指出低熵样本可能不产生 compact 类内分布。点云中这个问题更严重，因为 rotation、partial scan、dropout、non-uniform density 可能让模型对错误类别非常自信。机会在于构造 3D-specific reliability：

- Entropy reliability：预测熵、top-1/top-2 margin。
- Text alignment reliability：point feature 与 class text anchor 的余弦一致性。
- Geometry reliability：局部密度、法向/曲率稳定性、PCA/ICP residual、Chamfer residual。
- Cache compactness reliability：样本加入后是否提高类内 compactness，而不是只看当前置信度。

### 7.2 局部-全局 cache 还可以更细

Point-Cache 已有 hierarchical cache，但 MCP-style multi-cache 可以进一步拆：

- global positive cache：全局形状语义。
- local positive cache：局部部件或 patch token。
- align cache：文本锚点一致的高质量样本。
- negative cache：高熵/冲突/几何异常样本。
- corruption-aware cache：对 rotation、dropout、jitter 等维护不同可靠性统计。

关键不是“cache 越多越好”，而是每个 cache 解决一个可证伪的问题，并通过 leave-one-out ablation 证明。

### 7.3 Open-vocabulary 3D TTA 仍不成熟

open-vocabulary 3D 模型依赖文本 prompt、CLIP-style text encoder 和点云 encoder 对齐。测试时类别文本锚点的质量会影响 cache 和 prototype。可研究：

- 多模板 text prompt 聚合。
- LLM paraphrase 生成 class descriptions。
- vMF MAP text anchor，把多文本嵌入看作球面分布样本。
- 用 negative cache 校准容易混淆的文本类别。

风险：文本增强的收益可能很小，必须在 W3 之前用小实验判断，避免主方法依赖弱增益。

### 7.4 真实 3D shift 与 synthetic corruption 不同

ModelNet-C 很适合方法开发，但真实扫描的 partial observation、background clutter、sensor pattern、domain shift 更复杂。ScanObjectNN-C、OmniObject3D、Objaverse-LVIS 和 Sim2Real-SONN 是重要扩展。论文叙事应避免只说“解决真实世界”，要用实验证明从 CAD corruption 到 real scan shift 的迁移。

### 7.5 高效 TTA 是卖点

Point-Cache 的 training-free 和 Purge-Gate/BFTT3D 的 backprop-free 趋势说明，3D TTA 不能只看 accuracy。在线机器人/自动驾驶场景关心 latency、VRAM 和稳定性。本地 T4 环境也天然要求方法轻量。

## 8. 与 MCP-Point-Cache 的具体结合

### 8.1 最推荐的主线

主方法雏形：

> 3D-MCP: Geometry-Aware Multi-Cache Prototype Adaptation for Open-Vocabulary Point Cloud Recognition

核心组件：

1. 以 OpenShape / ULIP-2 / Uni3D 的 point feature 和 text feature 为基础。
2. 继承 Point-Cache 的 global/local hierarchical cache。
3. 引入 MCP-style entropy / align / negative cache。
4. 相比 Uni-Adapter 的动态 cluster prototype，增加 geometry-aware reliability、negative cache 和 cross-modal align cache。
5. 用 3D geometry reliability 改造 cache admission 和 logit fusion。
6. 用 vMF text anchor 稳定 class prototype。
7. 用 ICP/Chamfer residual 作为 failure detection 或 reliability score，而不是一开始就作为重计算主模块。

### 8.2 可发表的最小创新闭环

最小可行贡献：

- 提出一个 3D-specific cache reliability score：
  - `R = alpha * confidence + beta * text_alignment + gamma * geometry_consistency + delta * compactness_gain`
- 用它统一控制：
  - positive cache admission
  - align cache update
  - negative cache calibration
  - global/local cache fusion weight
- 证明它比 entropy-only、Point-Cache-only 和 naive 3D-MCP 更稳。

如果实验结果好，可以把方法命名为 Geometry-aware Multi-Cache Adaptation。若 ICP/Chamfer 计算过重，可把 geometry score 简化为 PCA residual、local density statistics 或 feature-space local consistency。

### 8.3 低风险、中风险、高风险选题

| 选题 | 风险 | 预期价值 | 建议 |
| --- | --- | --- | --- |
| Cache quality study for point cloud TTA | 低 | 可快速产出分析图和 workshop 级结果 | 可作为保底论文或主论文诊断章节 |
| Geometry-aware multi-cache for open-vocabulary point cloud TTA | 中 | 与 Point-Cache/MCP 差异明确，最适合主线 | 推荐主攻 |
| vMF text anchor for 3D open-vocabulary TTA | 中 | 如果文本 prompt 确实影响大，会成为亮点 | 先做小规模门控实验 |
| ICP/Chamfer-guided cache admission | 中高 | 3D-specific 强，但效率和鲁棒性有风险 | 作为 ablation/extension，避免主方法完全依赖 |
| Diffusion-guided point repair + cache | 高 | 可能效果强，但实现重且 baseline 压力大 | 不建议第一阶段主攻 |
| Continual scene-level 3D TTA | 高 | 真实价值高，实验复杂 | 放到 CVPR phase 或后续工作 |

## 9. 实验路线图

### Week 1：复现和协议锁定

- 跑通 OpenShape PointBERT-vitg14-rgb。
- 准备 ModelNet-C、ScanObjectNN-C、ModelNet40、ScanObjectNN clean。
- 复现 zero-shot、Point-Cache global、Point-Cache hierarchical。
- 记录每个 corruption 的 accuracy、latency、VRAM。
- 固定 prompt、输入点数、采样策略、stream order、reset 策略。

风险门：

- Point-Cache 复现误差如果超过 2pp，先 fork official code unchanged，不要急着改方法。
- T4 无法同时加载 OpenShape + text encoder 时，优先 feature pre-encoding 或换更小 backbone。

### Week 2：诊断实验

- Entropy vs correctness：低熵样本在各 corruption 下是否可靠。
- Compactness vs performance：类内 compactness Phi 与 accuracy gain 是否相关。
- Text alignment vs correctness：point-text similarity 是否能过滤错误样本。
- Geometry residual vs correctness：PCA/ICP/Chamfer residual 是否能识别 rotate/dropout/jitter 失败样本。
- Cache contamination：错误样本进入 cache 后对后续 stream 的影响。

输出：

- 1 张 entropy reliability 曲线。
- 1 张 compactness-performance scatter。
- 1 张 per-corruption failure heatmap。

### Week 3：Naive 3D-MCP 与轻量多 cache

- 实现 naive 3D-MCP：
  - entropy cache
  - align cache
  - negative cache
  - prototype/logit fusion
- 与 Point-Cache global/hierarchical 做公平对比。
- 加入 vMF text anchor 的开关。
- 加入 geometry reliability 的轻量版本。

风险门：

- naive 3D-MCP 如果低于 Point-Cache，仍可作为“entropy-only cache 不足”的反例。
- vMF text anchor 增益小于 0.1pp 时，降级为 appendix。

### Week 4：主实验和 ablation

- ModelNet-C 7 corruption x severity。
- ScanObjectNN-C 3 split 或至少 obj_only/hardest。
- Clean control。
- Multi-seed 和 stream-order seed。
- Leave-one-out cache ablation。
- Reliability score component ablation。
- Efficiency table。

### 后续扩展

- Cross-backbone：OpenShape -> ULIP-2 -> Uni3D。
- Long-tail open-vocab：Objaverse-LVIS。
- Real scan / sim-to-real：Sim2Real-SONN。
- Residual fine-tuning：MCP++ style，但需要严格 clean drop 和 efficiency 控制。

## 10. Baseline 矩阵

必须 baseline：

| Baseline | 目的 |
| --- | --- |
| Zero-shot OpenShape / ULIP-2 / Uni3D | 基准线 |
| Point-Cache global | 直接 cache 基线 |
| Point-Cache hierarchical | 最强本地相关基线 |
| Uni-Adapter | 最新 3D VLFM training-free online prototype cache 强基线 |
| Naive 3D-TDA | 单正 cache 基线 |
| Naive 3D-MCP | 证明 multi-cache 从 2D 迁移的直接效果 |

建议 baseline：

| Baseline | 目的 |
| --- | --- |
| Tent / SAR | 通用 TTA 对比，若 backbone 可反传且协议公平 |
| MATE | 3D MAE/self-supervised TTA 对比 |
| CloudFixer / 3DD-TTA | 输入空间几何修复对比 |
| WACV 2025 sampling variation + weight averaging | 轻量适配对比 |
| Purge-Gate / BFTT3D | backprop-free 高效路线对比 |

注意：不是所有 baseline 都必须完整复现。主论文至少要有强 Point-Cache 对比、一个通用 TTA 对比、一个 3D-specific TTA 对比、一个 cache/prototype 类对比。

## 11. 可能的论文贡献写法

### Contribution 1：诊断

We reveal that entropy-only cache admission is unreliable for open-vocabulary point cloud TTA under geometry corruptions, because low-entropy samples may be geometrically inconsistent or cross-modally misaligned.

### Contribution 2：方法

We propose a geometry-aware multi-cache prototype adaptation framework that jointly maintains global/local positive caches, cross-modal alignment cache, and negative cache, with a 3D reliability score for cache admission and logit calibration.

### Contribution 3：验证

We evaluate on ModelNet-C, ScanObjectNN-C, clean controls, and open-vocabulary benchmarks across large 3D models, showing better robustness, lower cache contamination, and comparable test-time efficiency.

中文落点：

- “熵置信度不足”是问题。
- “几何可靠性 + 文本锚点 + 多缓存校准”是方法。
- “open-vocabulary 3D recognition 的 training-free TTA”是场景。

## 12. Devil's Advocate：主要风险

1. 新意风险：Point-Cache 已经是 hierarchical cache，MCP 已经是 multi-cache。必须证明 3D geometry-aware reliability 是必要且有效的，而不是把两个方法拼起来。
2. 最新基线风险：Uni-Adapter 已经覆盖 3D VLFM 的 training-free online dynamic prototype cache。主方法需要明确回答：为什么 dynamic prototype + graph smoothing 不够，geometry reliability / negative cache / text-anchor calibration 带来什么额外收益。
3. 效率风险：ICP/Chamfer 如果每样本都跑，可能破坏 training-free / real-time 叙事。建议把它作为可选高置信诊断或低频校准。
4. 文本锚点风险：如果 OpenShape/ULIP 的文本 prompt 已很稳定，vMF/LLM paraphrase 增益可能很小。
5. 协议风险：online TTA 很容易因为 test set 顺序、batch transduction、hyperparameter tuning 被质疑。必须预注册式固定协议。
6. Clean accuracy 风险：TTA 提升 corruptions 但损害 clean，是审稿人常抓问题。
7. Benchmark 风险：只在 ModelNet-C 上有效会显得过拟合 synthetic corruptions，ScanObjectNN-C 和 real scan 类数据很重要。

## 13. 写 Related Work 的组织方式

建议分 4 段：

1. Test-time adaptation generalization：TTT、Tent、CoTTA、SAR，强调通用方法依赖 entropy/online update，但忽略 3D geometry。
2. Point cloud TTA：MATE、CloudFixer、HGL、BFTT3D、PCoTTA、APCoTTA、3DD-TTA、SMART-PC、GSDTTA、Uni-Adapter，强调 3D 方法开始使用 reconstruction、geometry repair、scene-level continual adaptation 和 3D-VLFM online prototype cache。
3. Cache/prototype and VLM TTA：TPT、TDA、MCP，说明 cache/prototype 对 VLM source-free generalization 有效，但 2D compactness assumption 不能直接搬到 3D。
4. Open-vocabulary point cloud recognition：ULIP、OpenShape、Uni3D、Point-Cache，最后引出本文：需要 training-free、geometry-aware、multi-cache 的 open-vocabulary point cloud TTA。

## 14. 注释书目

| 文献 | 类型 | 关键贡献 | 本项目用途 |
| --- | --- | --- | --- |
| Sun et al., 2020, Test-Time Training with Self-Supervision | ICML | 测试时自监督更新 | 通用 TTA 起点 |
| Wang et al., 2021, Tent | ICLR | 熵最小化和 BN/归一化更新 | 通用强基线 |
| Wang et al., 2022, CoTTA | CVPR | 连续 TTA，teacher-student 和 restoration | online/continual 协议参考 |
| Niu et al., 2023, SAR | ICLR | reliable entropy + sharpness-aware | “低熵不可靠”的通用基线 |
| Mirza et al., 2023, MATE | ICCV | MAE online 3D test-time learner | 点云 self-supervised TTA |
| CloudFixer, 2024 | ECCV | diffusion-guided geometric transformation | 输入空间修复 baseline |
| HGL, 2024 | ECCV | 3D segmentation TTA 的 hierarchical geometry learning | 局部-全局几何叙事 |
| BFTT3D, 2024 | CVPR | backpropagation-free network for 3D TTA | 高效无反传路线 |
| 3DD-TTA, 2025 | WACV | denoising diffusion for point cloud TTA | 生成式修复对比 |
| WACV 2025 sampling variation TTA | WACV | 采样变化 + weight averaging | 轻量点云 TTA baseline |
| Point-Cache, 2025 | CVPR | dynamic hierarchical cache, training-free, open-vocabulary 3D recognition | 主基线 |
| MCP, 2025 | ICCV | entropy/align/negative multi-cache for VLM TTA | 主要迁移来源 |
| PCoTTA, 2024/2025 | NeurIPS / OpenReview | continual TTA for multi-task point cloud understanding | scene-level extension |
| APCoTTA, 2025/2026 | journal/preprint | airborne LiDAR continual TTA | 真实 LiDAR shift 参考 |
| SMART-PC, 2025 | ICML | skeleton model adaptation for robust point cloud TTT | 结构先验路线 |
| Purge-Gate, 2025 | preprint | backprop-free point cloud TTA | 高效无反传对照 |
| GSDTTA, 2025 | preprint | graph spectral driven point shift | 几何/图谱修复路线 |
| Uni-Adapter, 2026 | AAAI | training-free online TTA for 3D VLFMs via dynamic prototype cache | 最新最相关 baseline |
| PointWOLF, 2021 | ICCV | weighted local transformations, ModelNet-C 相关 corruption 评测 | corruption benchmark 背景 |

## 15. 主要来源链接

通用 TTA：

- TTT, ICML 2020: https://proceedings.mlr.press/v119/sun20b.html
- Tent, ICLR 2021: https://openreview.net/forum?id=uXl3bZLkr3c
- CoTTA, CVPR 2022: https://openaccess.thecvf.com/content/CVPR2022/html/Wang_Continual_Test-Time_Domain_Adaptation_CVPR_2022_paper.html
- SAR, ICLR 2023: https://openreview.net/forum?id=g2YraF75Tj

点云 TTA / 3D robustness：

- MATE, ICCV 2023: https://openaccess.thecvf.com/content/ICCV2023/html/Mirza_MATE_Masked_Autoencoders_are_Online_3D_Test-Time_Learners_ICCV_2023_paper.html
- CloudFixer: https://arxiv.org/abs/2407.16193
- HGL: https://arxiv.org/abs/2407.12387
- BFTT3D: https://openaccess.thecvf.com/content/CVPR2024/html/Wang_Backpropagation-free_Network_for_3D_Test-time_Adaptation_CVPR_2024_paper.html
- 3DD-TTA: https://arxiv.org/abs/2411.14495
- Test-Time Adaptation in Point Clouds via Sampling Variation and Weight Averaging: https://openaccess.thecvf.com/content/WACV2025/html/Bahri_Test-Time_Adaptation_in_Point_Clouds_Leveraging_Sampling_Variation_with_Weight_WACV_2025_paper.html
- Point-Cache: https://arxiv.org/abs/2503.12150
- Point-Cache code: https://github.com/auniquesun/Point-Cache
- PCoTTA: https://openreview.net/forum?id=739jAzUXk7
- APCoTTA: https://arxiv.org/abs/2505.09971
- SMART-PC: https://proceedings.mlr.press/v267/bahri25a.html
- Purge-Gate: https://openaccess.thecvf.com/content/ICCV2025/html/Yazdanpanah_Purge-Gate_Backpropagation-Free_Test-Time_Adaptation_for_Point_Clouds_Classification_via_Token_ICCV_2025_paper.html
- GSDTTA: https://openaccess.thecvf.com/content/ICCV2025/html/Wei_3D_Test-time_Adaptation_via_Graph_Spectral_Driven_Point_Shift_ICCV_2025_paper.html
- Uni-Adapter / Adapt-As-You-Walk Through the Clouds: https://ojs.aaai.org/index.php/AAAI/article/view/37888
- Point-TTA for registration: https://openaccess.thecvf.com/content/ICCV2023/html/Li_Point-TTA_Test-Time_Adaptation_for_Point_Cloud_Registration_Using_Multitask_Meta-Auxiliary_ICCV_2023_paper.html

Cache / VLM TTA / 3D foundation models：

- MCP: https://arxiv.org/pdf/2508.01225
- MCP project: https://zhaihaotian.github.io/MCP-ICCV25/
- TPT: https://arxiv.org/abs/2209.07511
- TDA: https://arxiv.org/abs/2403.18293
- OpenShape: https://arxiv.org/abs/2305.10764
- ULIP-2: https://arxiv.org/abs/2305.08275
- Uni3D: https://arxiv.org/abs/2310.06773

Benchmarks / corruption：

- PointWOLF, ICCV 2021: https://openaccess.thecvf.com/content/ICCV2021/html/Kim_Point_Cloud_Augmentation_With_Weighted_Local_Transformations_ICCV_2021_paper.html
- PointDA-10: https://arxiv.org/abs/1912.12577

## 16. 最终建议

最推荐执行路径：

1. 先复现 Point-Cache，不要先写新模块。
2. 做 entropy reliability / compactness / geometry residual 三个诊断图，证明问题存在。
3. 实现 naive 3D-MCP，作为从 2D MCP 到 3D 的直接迁移基线。
4. 在 naive 3D-MCP 上加入 geometry-aware reliability 和 global/local/negative cache。
5. 以 clean drop、efficiency、multi-seed、per-corruption gain 为核心质量门。

如果只能选择一个主问题：

> 在 open-vocabulary point cloud TTA 中，如何避免 entropy-only cache 被几何 corruption 污染？

这个问题足够具体，和 Point-Cache/MCP 直接相关，也能自然导出方法、实验和贡献。
