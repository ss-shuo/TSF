# Innovation-Research Report for Cross-Site EV Charging-Demand Probabilistic Forecasting

> 研究日期：2026-08-30（UTC）  
> 固定主数据集：UrbanEVDataset / UrbanEV  
> 执行模式：**[Single-Agent Constrained Review]**  
> 结论性质：桌面研究合同与待验证方案，不是实验结论  
> 证据标签：**[Official Documentation Supported]**、**[User Local Audit]**、**[Literature Report]**、**[Full Text Unverified]**、**[Planning Decision]**、**[Falsifiable Hypothesis]**、**[Research Plan to Validate]**、**[To Verify]**

## 0. Executive Summary

### 0.1 Desk-Research Conclusion and Evidence Ceiling

本报告遵循 b.md 的门禁顺序。原模板只预设数据集名称 UrbanEVDataset；本轮还收到一份 2026-08-18 初审、2026-08-29 独立复核的 dataset.md。该文件是**用户提供的审计档案**，不是本代理重新扫描原始字节所得结果。本工作区没有 6.57 GB 解压数据、项目代码、训练日志、模型检查点或实验结果。因此：

- Dryad、Scientific Data 论文和官方 GitHub 页面给出的身份、版本、许可与字段信息标为 **[Official Documentation Supported]**；
- dataset.md 中的文件哈希、逐站行数、缺失比例和一致性计数标为 **[User Local Audit]**；
- 论文作者报告的性能或现象标为 **[Literature Report]**；
- 对 UrbanEV 上可能发生的基线失败、候选优越性与资源需求全部标为 **[Falsifiable Hypothesis]** 或 **[Research Plan to Validate]**，不写成既成事实。

官方状态门禁结论为 **CONDITIONAL PASS**。理由是：UrbanEV 身份已由 [Dryad DOI 记录](https://datadryad.org/dataset/doi:10.5061/dryad.np5hqc04z)、[Scientific Data 数据论文](https://doi.org/10.1038/s41597-025-04874-4) 和 [作者仓库](https://github.com/IntelligentSystemsLab/UrbanEV) 交叉确认；最新公开完整 Dryad 版本更新于 2026-02-04，许可为 CC0，含 1,682 个站的 5 分钟状态快照和站点容量，结构上足以构造容量有界、严格留站的概率预测任务。条件来自四个未消除风险：官方材料对坐标系、小时目标语义和价格单位有冲突；约五分之一状态快照缺失且缺失原因未知；时间戳时区仍需从原始说明/维护者确认；数据只覆盖深圳同一采集系统的六个月。它们不阻塞一个**不使用坐标、未来天气和未来价格，并保留缺失掩码的深圳站内跨站任务**，但禁止跨城市、跨运营商和电网功率结论。

本地审计把目标进一步收窄为：每小时 HH:55 的占用连接器数 busy，而不是会话到达数、充电功率或电量。**[User Local Audit]** 报告所有有效行满足 busy + idle = charge_count，故 $Y_{s,t}\in\{0,\ldots,C_s\}$；这使“容量有界离散预测”成为有数据依据的研究对象。相反，天气只有事后观测、POI 是 2022-12-01 快照、充电桩明细不覆盖全部站、坐标系不确定，因此它们都不进入主模型。

### 0.2 Recommended Research Contract, Conditional Preferred Method, and Rationale

**[Planning Decision] 主合同**：站点完全隔离的 unseen-site few-shot；每个模拟部署 episode 仅允许一个预冻结的 168 小时目标站适配窗，之后立即预测 1、3、6 小时，12 小时作敏感性分析。主输出是各 horizon 的离散边际 PMF；不声称跨 horizon 或跨站联合情景。零样本以同一站点、同一 forecast origin、空上下文单独报告。

**推荐基线**是 GluonTS 0.17.0 的 DeepAR-NB：全局 RNN、负二项计数似然、站点隔离训练、无站点 ID 嵌入。它有公开论文与维护实现，能处理观测掩码和已知时间特征，且其“非负但无容量上界”的机制缺口与 UrbanEV 目标直接相交。透明必备基线还包括季节经验分布和经验贝叶斯季节 Beta-Binomial；后者防止首选方法只胜过不匹配的负二项似然。

**[Research Plan to Validate] 条件首选方法**：

> **容量锚定单调伪计数神经过程 / Capacity-Anchored Monotone Pseudo-count Neural Process（CAMP-NP）**

其核心不是“把跨站、few-shot 或 Beta-Binomial 首次用于预测”。这些元素分别已有直接先例。CAMP-NP 的窄创新假设是：由源站学得查询时刻先验，再让每个合法上下文观测通过**非负、查询相关的分数成功/失败伪计数**单调增加 Beta 证据，最后以站点物理容量 $C_s$ 为 Binomial 试验数形成 Beta-Binomial PMF。空上下文退化为零样本先验；168 小时上下文只经一次前向传播改变伪计数，不做目标站梯度更新。该结构同时给出精确支持、可追踪的上下文影响和相同接口下的零/少样本行为。

选择它而不是“图网络 + 扩散”的原因是数据约束，而非模型流行度：坐标系冲突、同坐标多 ID、未来天气不可用和六个月时长使图/扩散链路风险高；CAMP-NP 只依赖容量、日历、合法适配窗和缺失掩码。它仍必须击败同容量头但无单调更新的 CNP、同参数量 MLP、截断 NB、经验贝叶斯 Beta-Binomial 和更长历史控制，才可把增益归因给核心机制。

### 0.3 Greatest Risks, First Validation, and Second-Candidate Switch Conditions

最大风险不是网络能否拟合，而是研究问题是否被错误表述。busy 是占用快照；它不能支持“能量采购误差下降”“到达率更准”或其他未经数据和决策模型连接的结论。缺失可能是站点未上线、停运或遥测失败，不能当作随机缺失，也不能填为零。

项目建立后的首个决定性验证按以下顺序执行：

1. 复核冻结版本的哈希、时区、容量字段、有效行支持和 HH:55 整数性；
2. 运行季节经验 PMF、经验贝叶斯 Beta-Binomial 与 DeepAR-NB smoke test；
3. 仅用验证站检验 DeepAR 的越界概率质量、随机 PIT、容量分层 CRPS 和 context-length 曲线；
4. 若 NB 没有越界/尾部失配，且任意 Beta-Binomial 头无稳定收益，则删除容量似然创新主张；
5. 若单一 CAMP-NP 在站点层面呈现系统性多峰残差，而软原型混合在参数量/预算匹配下改善 macro-CRPS 与校准，则切换到第二候选 **容量有界软原型混合（CBSAM）**；
6. 若所有概率模型差别主要来自事后校准，则把方法主张降级为协议/校准研究；不把 conformal 包装器冒充生成模型创新。

一句话核心主张如下，且目前只能是待检验假设：

> **[Falsifiable Hypothesis]** 在冻结的 UrbanEV 严格留站、168 小时 few-shot 协议中，若负二项基线的主要误差确由容量越界与上下文证据使用不当造成，则相对参数量和信息集匹配的 DeepAR-NB、截断 NB、普通 Beta-Binomial CNP 与经验贝叶斯季节模型，CAMP-NP 应同时降低站点宏平均离散 CRPS/NLL、减少高占用阈值的 Brier 损失，并在不显著扩大 80% 区间宽度的前提下改善覆盖；若这些联合条件不成立，核心主张即被证伪。

## 1. Recommended Research Contract

### 1.1 Known Facts, Planning Decisions, Items to Validate, and Adjustment Conditions

| 项目 | 当前状态 | 依据 | 项目建立后验证 | 若不成立的调整 |
|---|---|---|---|---|
| 数据身份 | UrbanEV，Dryad DOI 10.5061/dryad.np5hqc04z | [Official Documentation Supported] | DOI、文件清单、版本日期再请求一次 | 身份/版本无法一致则 Data Not Ready |
| 冻结版本 | 2026-02-04 最新完整 Dryad 版本；用户审计记为 version id 423269、internal 23 | 官方日期/大小 + [User Local Audit] | 哈希与 Dryad manifest 一致 | 任何字节变化则新建 manifest，不静默覆盖 |
| 原始对象 | 1,682 站、5 分钟状态快照，2022-09-01 至 2023-02-28 | 官方论文/README；[User Local Audit] | 最小/最大时间、每站 52,128 行 | 不规则站进入排除日志，不插值修复 |
| 目标语义 | busy = 同时占用连接器数，不是 session、arrival、kW、kWh | README + [User Local Audit] | 抽样核对 busy + idle = capacity | 等式不普遍成立则取消 Beta-Binomial 方案 |
| 主频率 | [Planning Decision] 每小时 HH:55 快照 | 保留整数与精确容量支持 | HH:55 覆盖和相位敏感性 | 若该相位系统性异常，改为预冻结另一相位并重跑合同 |
| 主 horizon | [Planning Decision] 1/3/6 h；12 h 敏感性 | 短期运营且不夸大六个月数据 | 验证站按 horizon 的样本量/失配 | 长 horizon 样本不足则只保留 1/3/6 |
| 主跨站制度 | [Planning Decision] unseen-site few-shot，168 h 冻结窗 | 新站短历史；数据站数充足 | 每 episode 上下文覆盖 | 可用上下文不足则降为 zero-shot 主问题 |
| 次制度 | [Planning Decision] unseen-site zero-shot，$K=0$ | 检验源站先验，而非混在主结果 | 只用容量与日历 | 静态信息不足时明确“弱信息零样本” |
| 主概率输出 | [Planning Decision] 每 horizon 完整离散边际 PMF | count + capacity support | PMF 归一、零/上界概率 | 若支持失效，改为比例/连续任务并撤回当前方法 |
| 推荐基线 | [Planning Decision] DeepAR-NB + 两个透明基线 | 全局概率模型、公开实现、低复现门槛 | smoke test 与资源测量 | RNN 不适合 episode 接口时，以直接 NB-MLP 为公平主基线 |
| 未来外生量 | 日历可知；未来天气/价格不可证明 | [User Local Audit] + 官方说明 | forecast-origin 可用性表 | 不可证明者从主输入删除，只作 oracle 敏感性 |
| 硬件 | [To Verify] | 用户未提供 | 首次 smoke 记录峰值显存/RAM/时间 | 依预设减配顺序缩小隐藏维度/episode 数，不破坏隔离 |

### 1.2 Target, Frequency, Horizon, Forecast Origin, and Legal Information Set

设站点 $s$ 的连接器容量为 $C_s$，时间 $t$ 表示 Asia/Shanghai 候选时区下的整点小时。主目标：

$$
Y_{s,t}=\texttt{busy}_{s,t{:}55},\qquad
Y_{s,t}\in\{0,1,\ldots,C_s\}.
$$

“Asia/Shanghai”是 **[Planning Decision, To Verify]**，因为时间戳无显式时区；深圳在采样期无夏令时，但仍须维护者/原始说明确认，不可凭地理位置偷偷设定。

每个部署 episode 有一个适配开始 $a_e$、168 个时钟小时的上下文 $[a_e,a_e+167]$，forecast origin 为最后一个上下文小时结束后。只预测 origin 后第 1、3、6 小时；12 小时仅敏感性。上下文中的缺失位置保留 $m_{s,t}=0$，不压缩时间轴、不前后填充。few-shot 主队列要求在 origin 时已知的上下文覆盖至少 80%（134/168）；低覆盖 episode 单列，不能按未来 query 是否完整筛站。

合法信息集：

| 变量 | 分类 | 主实验是否使用 | 原因 |
|---|---|---:|---|
| $C_s$ / charge_count | static known | 是 | 物理支持；需与有效 raw 状态一致 |
| 小时、星期、周末、month | future known | 是 | forecast origin 时确定 |
| 站点 ID | identifier | 否 | 会使未见站无法编码 |
| 上下文 busy 与 mask | observed past | few-shot 是；zero-shot 否 | 仅来自冻结 168 h |
| 上下文 fast/slow 容量 | observed/static candidate | 暂否 | pile 表不完整；可作有完整元数据子集敏感性 |
| s_price/e_price | future-known 未证实 | 否 | 记录值不等于提前发布的价格表 |
| 未来天气 | oracle only | 否 | 数据无历史天气预报存档 |
| 过去天气 | observed past | 暂否 | 不是核心机制所需，避免无关复杂度 |
| POI | dated static snapshot | 否 | 2022-12-01 快照对更早 origin 有时间穿越风险 |
| 经纬度/距离图 | static but CRS disputed | 否 | Dryad 与 GitHub 对 WGS84/GCJ-02 冲突 |
| TAZ | static categorical | 主实验否 | 同区源/目标共享可能弱化严格留站；作 leave-TAZ-out 敏感性 |

### 1.3 Primary Cross-Site Regime, Probabilistic Output, Baseline, and Default Budget

| Regime | 训练可见站 | 目标站历史 | episode 与主张 |
|---|---|---|---|
| seen-site | 与评估站相同 | 可用 | 仅作机制诊断，不支持跨站泛化 |
| **unseen-site few-shot（主）** | 与目标站完全不相交 | 每 episode 固定 168 h；无其余目标站标签 | 新站短历史下的深圳站间迁移 |
| **unseen-site zero-shot（次）** | 与目标站完全不相交 | 禁止，$K=0$ | 容量+日历弱信息冷启动 |
| leave-TAZ-out | 训练/测试 TAZ 不相交 | 按 few-shot 规则 | 地理组外推敏感性，不等同跨城市 |
| future-domain | 站可重合 | 历史可用 | 另一个问题，不合并进跨站数字 |

站点按稳定 ID 分组；完全相同坐标的多 ID 不合并目标，但必须进入同一 split group，防止物理共址信息跨集合。对 group ID 做预登记 hash，目标比例 train/validation/blind-test = 60/20/20；实际站数由分组后 manifest 给出，不事后按目标表现平衡。源站训练期为 2022-09-01 至 2022-11-30；验证站使用 2022-12 的预冻结 episodes 选模型；选择冻结后，仅以源站 2022-09-01 至 2022-12-31 重新拟合一次；盲测站 episodes 起点固定为 2023-01-01、01-15、02-01、02-15，各自 168 h context 后立即出预测。每站重复 episode 的依赖在站点块 bootstrap 中处理；另报每站仅首个 episode 的严格固定启动敏感性。

完整 PMF 是 $P(Y_{s,t+h}=k\mid\mathcal I_{s,t}), k=0,\ldots,C_s$。主任务只给边际分布；若论文需要跨 horizon 场景，必须另建联合生成机制与 energy/variogram score，本报告不预先声称。

默认资源合同在硬件未知时写成可测公式：

| 项目 | 最低可执行 | 推荐 Q3 证据预算 | 当前状态 |
|---|---:|---:|---|
| 正式配置 | 7 个核心配置：E0/E1/E3/E4/E5/E6/E9 | 11 个配置 E0–E10；随机模型 3 seeds、确定性 E0/E1 各一次，共 29 runs | [Planning Decision] |
| 外部复现 | DeepAR-NB 1 个 | DeepAR-NB、N-HiTS/点模型+校准各 1 个独立计账 | [To Verify] |
| 统计不确定性 | 最低证据集的随机模型 3 seeds + 站点块 bootstrap | 所有随机模型 3 seeds + 站点/episode 块 bootstrap | [Planning Decision] |
| 总 GPU 时间 | 15 个随机训练的实测时间之和 | 27 个随机训练的实测时间之和；E0/E1 近似不占 GPU | 每种方法时间由 smoke 实测，不假设同一个 $T$ |
| 内存/磁盘 | 流式 Parquet；峰值实测 | RAM/VRAM 容量均 TBD；不预设用户硬件，按 OOM reduction rule 适配 | [To Verify] |
| API/数据费 | 0，使用公开固定数据 | 0；不接入商业天气/地图 API | [Planning Decision] |

减配顺序：先删除图/扩散外部复现，再把第二候选降为单 seed，再减少隐藏宽度和非关键 horizon；始终保留透明基线、DeepAR-NB、CAMP-NP、普通 BB-CNP、同参数量控制、proper score、盲测隔离和至少一种依赖保持的不确定性分析。

## 2. Dataset, Protocol, and Reproducibility

### 2.1 Official Status, Version Resolution, Selected-Release Freeze, and Gate Verdict

| 官方记录 | 身份/状态 | 访问日 | 作用 | 限制/冲突 |
|---|---|---|---|---|
| [Dryad DOI landing](https://datadryad.org/dataset/doi:10.5061/dryad.np5hqc04z) | Published，更新至 2026-02-04，CC0 | 2026-08-30 | 权威版本、文件、变更日志 | landing 未显示用户审计中的内部 version id/hash |
| [Scientific Data 论文](https://doi.org/10.1038/s41597-025-04874-4) | 2025 数据论文 | 2026-08-30 | 数据生成、处理和基准语义 | 论文/后续 Dryad 在占用率与占用数上有修订 |
| [官方 GitHub](https://github.com/IntelligentSystemsLab/UrbanEV) | public，CC0；2026-05-11 又加预处理站级数据 | 2026-08-30 | 代码、README、修订说明 | 坐标系写 GCJ-02，与 Dryad 的 WGS84 表述冲突 |

可发现版本关系：

| Dryad 日期 | 官方显示大小 | 关键变化 | 冻结判定 |
|---|---:|---|---|
| 2025-03-17 | 101.83 MB | 初始处理数据 | 拒绝：不是当前完整 raw 站级版本 |
| 2025-04-25 | 320.23 MB | 以 5 分钟 raw 站级数据替换/补足原 1 h 处理数据，并把区域小时目标从占用率改为占用数 | 拒绝：后续有修复和元数据补充 |
| 2025-09-24 | 350.97 MB | 官方日志称修复压缩归档问题，数据内容关系需按 changelog 复核 | 拒绝：后续补 station_information |
| **2026-02-04** | **320.25 MB** | 增加 station_information、澄清 TAZID 连接 | **选定：最新 active complete** |

冻结记录：

- DOI：10.5061/dryad.np5hqc04z；
- 版本日期：2026-02-04；
- **[User Local Audit]** Dryad version id 423269、internal version 23；
- README：14,914 bytes，SHA-256 <code>ec2aba0a893d2b310b12ddfd006679ae92a85149aa7a767e1734e82d175ac177</code>；
- ZIP：320,234,247 bytes，SHA-256 <code>a041322ed75eab8c49095fe5d0501b05f8d56dae16209586f8e4d7333ad1051f</code>；
- 官方代码固定 commit：<code>44f2aa0c8d89f192bce00bafb0def74a21b39c68</code>；用户审计的 commit archive SHA-256 为 <code>b12c2b6d53020003e5ed6d921cd910aee78e7d350f3b77a76403dbf1c53844e9</code>；
- 后续 GitHub 预处理文件只是补充实现快照，不替换 Dryad 主数据冻结版本。

**门禁：CONDITIONAL PASS。** 非关键条件的处理是从主合同删除坐标图、未来天气/价格、POI 和 pile 明细，并把 missing 作为不可识别的观测机制；若 raw 复核发现 busy 不再是容量有界整数、容量无法稳定对应、时区导致切分跨日错误，门禁立即转为 **FAIL / Data Not Ready**，停止依赖该目标的方法开发。

### 2.2 Fixed UrbanEVDataset Task-Fit Audit and Innovation-Driven Auxiliary-Data Decision

| 审计项 | 当前证据 | 任务适配 | 待验证/降级 |
|---|---|---|---|
| 多站稳定 ID | 1,682 个 CSV，ID 1001–2682 [User Local Audit] | 支持留站 | 检查文件名、id 列一致 |
| 连续时间轴 | 每站 52,128 个 5 分钟槽，六个月 [User Local Audit] | 支持短期 episode | 时区、边界、重复再核 |
| 容量有界目标 | 有效行 busy + idle = charge_count [User Local Audit] | 直接支持 CAMP-NP | 任一系统性违例则撤销 |
| 缺失/零可分 | 六状态字段要么全有要么全缺；有效 busy=0 单列 [User Local Audit] | 可保留 mask | 缺失原因不可识别，不能 MAR 宣称 |
| 跨站样本规模 | 1,682 站；精确 eligible 数取决于部署窗 | 结构上充足 | 分组后站数/覆盖不能预填 |
| 元数据 | capacity、TAZ、坐标、pile、POI | 主模型只需 capacity | pile 240 站缺失/不一致；坐标冲突 |
| 未来外生量 | weather 是实况、无预报档案 | 不可作主未来输入 | oracle 只能单列 |
| 外部有效性 | 深圳、单采集系统、2022-09 至 2023-02 | 仅站内跨站 | 不写跨城市/运营商结论 |

辅助数据决定：**本轮不引入任何新辅助数据集。** 原因是 CAMP-NP 的核心 claim 只需容量、日历和冻结上下文；天气预报、交通、事件或地图数据只会增加法律信息集与空间匹配风险，不能作为“创新点”。法定节假日可在后续用中国政府已公布日历作预登记敏感性，但不影响首轮保留/否决 CAMP-NP。若未来把论文主张改成事件或天气条件下的尾部概率，才触发独立的官方来源、许可、发布时间和历史预报审计。

创新 claim 到数据要求的映射：

| Claim | 必需属性/控制 | UrbanEV 证据 | 定向 raw 审计 | 辅助数据 |
|---|---|---|---|---|
| 精确容量支持改善概率质量 | $Y$ 整数且 $0\le Y\le C_s$ | [User Local Audit] 全部有效行一致 | 全量断言、capacity 版本稳定性 | 不需要 |
| 单调伪计数利用 168 h | context 中零/缺失可分，clock 连续 | [User Local Audit] 支持 | episode 覆盖、同步缺失长度 | 不需要 |
| 跨站而非随机窗口 | 站 ID 稳定、站 split 隔离 | 官方/审计支持 | 共址 group、TAZ 泄漏检查 | 不需要 |
| 校准而非只降 MAE | 完整离散 PMF | 方法定义支持 | evaluator 单元测试 | 不需要 |
| 空间图增益 | 可信 CRS/合法邻接 | 官方冲突 | 暂缓 | 当前不准入 |

### 2.3 Download, License, Schema, Target Construction, and Data-Generating Process

Dryad 与仓库均声明 CC0；研究代码、派生特征和可复现 manifest 原则上可发布，但仍须遵守平台访问条款，并避免把站点坐标重新解释为个人数据。数据是充电设施状态遥测，不含车辆/用户级 ID。CC0 不保证无第三方权利或数据正确性，因此论文仍需保留免责声明。

**[User Local Audit]** 解压结果为 1,701 个文件、6,569,791,800 bytes，未报告 ZIP 完整性错误。核心 raw station 字段：

| raw field | 当前语义 | 主处理 | 输出字段 | 主要风险 |
|---|---|---|---|---|
| time | 5 分钟本地候选时间戳 | 严格 parse、排序、完整 grid join | timestamp, hour | 无显式时区 |
| busy | 占用连接器快照 | 不填充；取 HH:55 | target_y | 不是流量/能量 |
| idle | 空闲连接器 | 一致性检查 | audit only | 维护/故障状态未单列 |
| fast_busy/idle | 快充状态 | 容量一致性审计 | optional | 不是全站 pile 表替代品 |
| slow_busy/idle | 慢充状态 | 同上 | optional | 同上 |
| s_price/e_price | 价格字段 | 保留 raw，不进主未来输入 | oracle/sensitivity | 单位和提前可知性冲突 |
| duration/volume | 文档含义需核 | 不进主 target | audit only | 易被误当会话或能量 |

数据生成过程是平台按 5 分钟记录连接器状态的**快照过程**。同一辆车可跨多个快照；busy 的小时序列因此具有持续性和容量截断。HH:55 选择不把 12 个快照平均成非整数，也避免前向填充制造观测；代价是它代表小时末一个时点而非小时平均。先做 HH:25 的 raw coverage/support/distribution 相位审计；只有差异超过预冻结阈值才触发一套独立模型敏感性，不能看主结果后决定。

### 2.4 Cleaning, Aggregation, Acceptance, Traceable Artifacts, and Raw-Data Audit Checklist

**[User Local Audit]** 的关键测量用于规划而不冒充本轮复核：

- 87,679,296 个 raw 站-5 分钟槽；69,123,930 行六状态完整，18,555,366 行六状态全缺，缺失率 21.162768%；无“部分六状态”行；
- 有效 busy=0 共 19,038,287，证明零与缺失是不同编码；
- HH:55 形成 7,306,608 个站小时，其中 5,760,234 有效、1,546,374 缺失，缺失率 21.164048%；
- 1,661 站至少有一个有效 HH:55，21 站完全没有；全期覆盖 ≥95%/90%/80% 分别为 452/836/878 站；
- 月度 HH:55 观测率从 2022-09 的 84.6037% 到 2023-02 的 56.9218% 波动，说明不能假设 MCAR/MAR；
- 24,798 个连接器、容量范围 1–108、331 个 TAZ；22,650 条 pile 覆盖 1,552 站，另有 130 站无 pile，110 站 pile 数与站容量不一致；
- 1,543 个唯一精确坐标；125 个多 ID 同坐标组、涉及 264 个 ID，故“共址分组但不合并目标”是必要防泄漏规则。

可执行处理顺序：

1. 下载到 content-addressed raw 目录；校验 DOI、日期、大小和 SHA-256；
2. ZIP 安全解压，拒绝绝对路径、双点路径、符号链接和重复覆盖；
3. 建 inventory：相对路径、字节数、SHA-256、mtime 不作身份依据；
4. 逐站流式读取；断言列集合、dtype、站 ID、52,128 候选槽、严格时间单调和唯一；
5. 不删除原行；与完整 5 分钟 grid 左连接，生成 observed_mask；
6. 对有效行断言非负整数、busy+idle=capacity、fast/slow 分解；异常只入 quarantine；
7. 精确选择 minute=55；不插值、不前后填充、不把 NA 变 0；
8. 在任何目标统计前按共址 group 冻结 site split；
9. 按固定 episode 日历构造 context/query，所有 fit 型转换只用 source train；
10. 输出 Parquet、manifest、schema、exclusion log、coverage report、split file 和 leakage report。

验收阈值不是为了追求“零错误”，而是触发明确动作：

| 检查 | PASS | FAIL 动作 |
|---|---|---|
| 哈希 | 与冻结值完全相同 | 停止；新版本重做 gate |
| schema | 必需列全在、类型可无损转换 | Data Not Ready |
| target support | 所有进入模型的有效 HH:55 均为整数且 $0\le y\le C_s$ | 隔离异常；若系统性则撤销方法 |
| 时间 | 无重复，5 分钟 grid 可解释 | quarantine 站并报告 |
| missing | NA 与 zero 分开，mask 可追踪 | 停止建模 |
| split | group 唯一属于一个集合 | 停止；重建 split |
| episode | context/query 不重叠、不跨时间边界 | 停止 evaluator |

追踪产物建议：

~~~text
raw_manifest.json
official_record_snapshot.md
schema.json
station_integrity.parquet
missingness_report.parquet
hourly_hh55.parquet
site_groups.parquet
split_v1.json
episodes_v1.parquet
feature_legality.yaml
leakage_audit.md
model_registry.json
evaluation_manifest.json
~~~

### 2.5 Time/Site Split, Exogenous Variables, Leakage Protection, and Evaluation Units

split 在任何 target-derived 标准化、聚类或异常阈值之前形成。用 dataset-version-derived 固定 salt 对 coordinate groups 作 SHA-256 排序，再按 group count 切成 60/20/20；只生成这一版 split。容量与 coverage band 仅用于**事后描述**，不因分布难看而重抽 seed；若某集合无法支持合同所需 episode，数据/任务 gate 失败并缩小 claim，而不是 split shopping。坐标相同组一起分；TAZ 不用作主特征。

最小泄漏审计：

| 项目 | 桌面判定 | 实施条件 |
|---|---|---|
| 同站跨 train/test | 设计 Pass | group split 单元测试 |
| 共址多 ID 跨集合 | 设计 Pass | exact-coordinate group 一致 |
| scaler/imputer/cluster 看 test | 设计 Pass | 主方案无 target scaler/cluster；所有 fit 仅 source |
| 上下文跨 query | 设计 Pass | episode 边界断言 |
| zero-shot 看目标历史 | 设计 Pass | context tensor 长度 0；独立 dataloader |
| few-shot 看冻结窗外历史 | 设计 Pass | episode loader 只暴露 168 h |
| 未来天气/价格 | 设计 Pass | 主 feature allowlist 拒绝 |
| POI 时间穿越 | 设计 Pass | 主 feature allowlist 拒绝 |
| graph 由 test target 建 | 设计 Pass | 主方案无图 |
| 早停/校准看盲测 | 设计 Pass | validation-only registry；blind token 一次解封 |
| query missing 筛选站 | 设计 Pass | 只 mask 评分项，不改变站/episode 入选 |

基本评价单元是 site × episode × horizon。先在每站内平均可评分 episode/horizon，再做站点 macro；另报 capacity-weighted micro，不能替代 macro。多次 episode 高度相关，因此置信区间以站为第一层重采样、站内以 episode block 为第二层；不能对数千个重叠样本做 IID t 检验。

## 3. Literature, Baseline, and Problem Evidence

### 3.1 Retrieval Method, Budget, C/F/E Sets, and Access Limits

检索起始日期按 b.md 建议设为 2020-01-01，终止日期为 2026-08-30；早期的 DeepAR、CNP 等只在机制不可替代时纳入。共冻结 12 个决策相关 query batches，覆盖至少三种措辞：

1. probabilistic / quantile / distributional EV charging forecast；
2. unseen station / cold-start / few-shot / global time-series EV charging；
3. bounded count / beta-binomial / neural process / conformal under shift。

独立入口包括 arXiv、SpringerLink、IEEE Xplore、PMLR/NeurIPS、AAAI、作者机构仓库与期刊官网。Google Scholar、Semantic Scholar 搜索界面和 OpenReview 论坛页在本环境触发安全/人机验证，不能声称完成；Crossref 只用于可打开的单 DOI 元数据，不把搜索摘要当全文。检索界面大多不暴露稳定总命中数，附录 A 因而记录“首屏可见并实际筛读数”，不虚构数据库总量。

按 b.md 的嵌套定义采用以下去重集合，而不是把 E 误写成 excluded：

- **C（candidate papers，21 篇）**：达到题名/摘要相关性并有稳定 identifier 的去重候选；含最终参考文献中的 21 篇学术工作，不计 GluonTS docs、dataset record 或 GitHub skill；
- **F（core full texts，12 篇，$F\subseteq C$）**：UrbanEV、DeepAR、Ostermann–Haug、DiffPLF、global N-HiTS、EV site archetypes、CNP、ECNP、beta-binomial bounded counts、ACI、CityEVCP 与 INLA EV；
- **E（experimentally auditable，8 篇，$E\subseteq F$）**：UrbanEV、DeepAR、Ostermann–Haug、DiffPLF、global N-HiTS、EV site archetypes、CNP 与 ECNP。它们进入 Appendix A.5 的 paper × organization / external-method / experimental-dimension 审计。

未进入 C 的 screening leads 包括替代主数据集、只做站点选址/调度而无预测分布、没有站点预测对象、或无法确认题名/作者/来源的结果；它们只进入 batch exclusion reason，不作为“领域没有工作”的分母。C 中但不在 F 的 access-limited/间接论文也不用于推断方法细节。

2026-08-25 才提交的 behavior-guided 论文和 2025 NeurIPS workshop 的 EV archetype 论文均明确标作 **[Author Preprint/Workshop, Single-Source]**；它们是重要反证，但不是与同行评审期刊同等级的定论。文献报告数字因数据、站点、目标、horizon 和 metric 不同，均不与 UrbanEV 待做实验直接比较。

按 b.md 要求检索了 GitHub research skill。选中 [ByteDance/deer-flow systematic-literature-review](https://github.com/bytedance/deer-flow/blob/main/skills/public/systematic-literature-review/SKILL.md)，owner 为 ByteDance，2026-08-30 的 HEAD 为 <code>8eda71fd978b4dc4b26843fe995407f09f4aa87e</code>，MIT 许可。只采用其“短核心关键词、相关性排序、结构化抽取研究问题/方法/发现/限制、跨论文合成而非罗列”的做法。未安装或运行仓库脚本：该 skill 仅搜 arXiv且强依赖其自身 subagent runtime，不满足 b.md 的多入口与本报告 **[Single-Agent Constrained Review]** 合同；执行第三方代码也不是科学证据。这一限制记录在附录 A。

### 3.2 Recommended Baseline: Mechanism, Reproduction Cost, and Selection Basis

推荐主基线为 **DeepAR-NB**。DeepAR 以共享 RNN 在多条相关时间序列上学习，自回归输出每个未来步的分布参数；原论文明确为 count data 提供负二项选择。[DeepAR 论文](https://doi.org/10.1016/j.ijforecast.2019.07.001) 是全球概率预测基线，[GluonTS 官方 PyTorch 文档](https://ts.gluon.ai/stable/api/gluonts/gluonts.torch.model.deepar.module.html) 暴露 past target、observed mask、static/dynamic feature、scaling 和 distribution output 接口。

复现冻结建议：

| 项目 | 冻结值/规则 |
|---|---|
| package | [GluonTS 0.17.0](https://pypi.org/project/gluonts/0.17.0/)，2026-07-31 发布；安装包 hash 写入 lockfile |
| likelihood | NegativeBinomialOutput；同一原始 count measure 上算 NLL |
| context / prediction | 168 / 6 小时，取 1/3/6；12 h 单独 estimator |
| lags | 1、24、168；若官方自动 lag 不同，配置必须落盘 |
| inputs | capacity、calendar、mask；无 station ID、坐标、天气、POI、未来价格 |
| scaling | 主配置关闭 target mean scaling；另做仅由 context 估计的合法 scaling 控制 |
| samples | 推断样本数预冻结；可直接读 NB 参数时用解析 PMF |
| station sampling | 先均匀采站，再采 source episode，避免大站/高覆盖站垄断 |
| status | 未实现、未 smoke-tested、未 reproduced |

机制限制：

1. NB 支持为 $\{0,1,\ldots\}$，会给 $Y>C_s$ 正概率；简单 clip 样本不能产生正确 NLL；
2. DeepAR 的 site scale 和 lag history 可能把少样本可用性与模型能力混在一起；
3. 它没有“更多合法 context 只能增加证据”的结构；
4. RNN 依赖连续 history，因此主评价使用“168 h context 后立即预测”的 deployment episode，避免让基线跨很长无标签间隔自回归。

为防止 strawman，以下同样是正式比较对象：

| 基线 | 输出 | 作用 |
|---|---|---|
| Seasonal empirical PMF | hour-of-week 经验离散分布 | 最低透明概率基线 |
| EB-SBB | source hour-of-week Beta prior + target context 共轭分数更新 | 直接检验神经机制是否胜过简单容量先验 |
| Truncated DeepAR-NB | 把 NB 在 0..C 上严格重归一 | 区分“支持修复”与“上下文机制” |
| Direct NB-MLP | 相同 calendar/capacity、无 RNN | 检查 backbone 与 episode 接口 |
| Point N-HiTS/TFT + independent conformal | 点模型后独立区间 | 排除收益只来自后处理校准 |

透明 controls 的实现也预先定义，避免名称相同但实现可任意：

- **Seasonal empirical PMF**：仅用 source-train 的 hour-of-week observations；把每个 source ratio $r=y/C$ 映射到目标容量的 $rC_s$，按 floor/ceil 线性分配概率质量，再给每个合法 count cell 加 $10^{-3}$ Dirichlet smoothing 并归一。它不读取 target context，是 zero/few-shot 共用的 source-only floor；
- **EB-SBB**：按 hour-of-week 在 source sites 上以 site-balanced Beta-Binomial marginal likelihood 拟合 $(\alpha_g,\beta_g)$；log-space L-BFGS-B 从 $(1,1)$ 单一起点运行，参数 bounds $[10^{-4},10^4]$、gradient tolerance $10^{-8}$、最多 1,000 iterations，失败则回退到全 source 共用 prior 并记失败。few-shot 时每个已观测小时贡献一单位 fractional evidence，
  $a=\alpha_g+\sum_i m_i y_i/C_s$、$b=\beta_g+\sum_i m_i(1-y_i/C_s)$；zero-shot 不加和。optimizer/tolerance 固定，结果确定性执行一次；
- **Truncated DeepAR-NB**：对 $k=0,\ldots,C_s$ 使用 $p_{\mathrm{NB}}(k)/F_{\mathrm{NB}}(C_s)$ 的严格重归一；主 control 以此 likelihood 训练，另可把同一 E3 checkpoint 只在 inference 重归一作为诊断，二者不能混名；
- **Direct NB-MLP**：相同 168 h legal inputs 经 parameter-matched causal pooling，直接输出各 horizon NB 参数；用来隔离 autoregression，而不是弱化输入；
- **ordinary BB-CNP**：与 CAMP 相同 encoder/query/BB PMF，但 pooled context 可通过 unrestricted MLP 改变 $a,b$，不保证 chronological append 的 evidence monotonicity。

EB-SBB 把一个小时快照视为一单位证据而非 $C_s$ 个独立连接器证据，以减轻同站连接器相关性和大站支配；这个选择同样须在 validation 前冻结。所有 smoothing、floor/ceil tie rule 与 numerical tolerance 写入 config。

外部 reproduction 与内部 CAMP-NP 配置分开计账；不同论文的 reported numbers 只作机制背景。

### 3.3 Literature Report–Mechanism Limitation–Falsifiable Failure–Competing Root Cause–Diagnostic Table

| 文献证据 | 已解决/作者报告 | 对本任务的机制限制 | UrbanEV 上可证伪失败 | 合理竞争根因 | 决策诊断 |
|---|---|---|---|---|---|
| [UrbanEV data paper](https://doi.org/10.1038/s41597-025-04874-4) | [Literature Report] 区域小时点预测、时间 CV、RMSE/MAPE/RAE/MAE | 不是严格留站；原处理含填充/过滤；非概率 | 复现式处理可能改善点误差却损害校准 | 原 raw missing 而非模型分布 | raw mask vs 官方 processed 对照 |
| [DeepAR](https://doi.org/10.1016/j.ijforecast.2019.07.001) | 全局 RNN 与 NB count likelihood | NB 无 capacity 上界 | 大容量/高占用站越界质量增加且 rPIT 偏斜 | backbone/context 不足，而非 likelihood | NB、截断 NB、BB 同 backbone |
| [Large-Scale Forecasting of EV Demand](https://pure.uva.nl/ws/files/171150045/VEHITS2024_Large_Scale_Forecasting_Emobility.pdf) | [Literature Report] global N-HiTS 在多站/外部站点点预测有竞争力 | 每日 kWh、长达 690 天、线性插值、点 metric | 简单 global 模型可能已足够，复杂 meta 机制无增益 | 数据量/日尺度比机制重要 | 相同 168 h、同预算 N-HiTS/MLP 控制 |
| [EV Site Archetypes Few-Shot](https://arxiv.org/abs/2510.26910v2) | [Literature Report] 8,000 个美国快充站，28 天 history，hard cluster expert 胜 global；workshop/preprint | 每日能量、点预测、proprietary 主数据；已否定“few-shot/原型本身新颖” | 单分布 CAMP-NP 可能被多原型混合击败 | 站点异质性是多峰而非 evidence 量 | C1 vs CBSAM，参数/数据匹配 |
| [CNP](https://proceedings.mlr.press/v80/garnelo18a.html) 与 [ECNP](https://ojs.aaai.org/index.php/AAAI/article/view/26125) | context set 到条件分布；ECNP 分解 uncertainty | 主要 Gaussian/Student-t；无站点容量、missing/legal-origin 协议 | CAMP-NP 只是换 likelihood，单调更新无独立作用 | 通用 CNP 已捕获全部 few-shot 价值 | 普通 BB-CNP、CAMP-NP、同参数量控制 |
| [Beta-binomial bounded-count model](https://doi.org/10.1007/s00184-023-00894-5) | 有协变量、有界、extra-binomial count dynamics | 单/少量序列统计模型，不是未见站 amortized context | 简单 EB-SBB 足以匹配 CAMP-NP | 神经权重只增加方差/过拟合 | EB-SBB 与 learned weights |
| [Ostermann & Haug 2024](https://doi.org/10.1186/s42162-024-00319-1) | [Literature Report] 多聚合层级量化 EV load quantiles，并评 pinball/interval | 连续功率/能量、seen groups、无容量 count support | UrbanEV 的概率难点可能主要是 aggregation level | 站级噪声不可约 | 站点 vs TAZ 仅作 protocol sensitivity |
| [DiffPLF](https://arxiv.org/abs/2402.13548) | [Literature Report] 条件扩散生成 EV load scenarios | Palo Alto 聚合连续 load、历史/未来变量、推断重 | 小数据/弱外生量下成本高且无稳定 gain | 联合依赖而非边际支持才重要 | 只有决策明确需 scenarios 时重启 |
| [Behavior-Guided Online PF](https://arxiv.org/abs/2608.24441) | [Author Preprint] 长期行为 + 最近 7 日 + delayed online quantile adaptation | 10 个 seen stations、需约 60 日初始 history、持续收到标签；非固定留站 | 冻结 few-shot 不如合法 online 更新 | temporal drift 主导 | 另设 online regime，禁止与主结果合并 |
| [Adaptive Conformal Inference](https://proceedings.neurips.cc/paper/2021/hash/0d441de75945e5acbc865406fc9a2559-Abstract.html) | 分布漂移下在线更新 coverage 控制 | 需要逐步真值；主要是 marginal coverage，不修正 generative PMF | conformal 改善 coverage 但区间变宽、CRPS/NLL不改善 | 原模型校准而非结构错误 | 独立校准控制 + sharpness |
| [CityEVCP](https://arxiv.org/abs/2410.18766) | [Literature Report] POI/hypergraph/动态变量改善 5 分钟区域点预测 | 1 个月、同区域时间切分、MSE；random node dropping 不是严格 target-history-free 留站 | 图复杂度可能仅利用同城相关 | 空间 spillover 真有剩余信号 | CRS 修复后 source-only 残差 Moran/graph 控制 |
| [INLA EV spatiotemporal model](https://arxiv.org/abs/2604.19841) | [Author Preprint] 96 CPID、日 session Poisson、空间/时间潜变量及后验 | 时间 80/20、相同站、Poisson 无容量、点 metric 为主 | 统计空间模型可用更低成本达到校准 | 深度模型不必要 | 无坐标主合同；后续只在 CRS 通过后比较 |

主根因排序是：

1. **[Falsifiable Hypothesis] 容量支持与上下文证据接口错配**；
2. **竞争根因：站点原型多模态**；
3. **竞争根因：非随机缺失/采集漂移，而不是预测分布**。

只有第 1 项在验证集通过 NB 越界、BB 头增益和 monotone-update attribution 三道诊断后，CAMP-NP 才保留首位。

### 3.4 Nearest Work, Key Counterevidence, and Research Gap

首选候选的最近工作不是按题名相似挑选，而按 task regime、信息集、概率表示和数学操作比较：

| 最近工作 | 相同点 | 关键不同 | 对 novelty 的约束 |
|---|---|---|---|
| DeepAR | global probabilistic、count | NB 无容量；RNN history，不是单调 context update | “全局概率模型”不新 |
| van Etten et al. 2024 | 未见站/global time series | daily kWh、point N-HiTS、长 history | “跨站泛化”不新 |
| Nikhal et al. 2025 | unseen-site few-shot、site heterogeneity | 28 天、hard archetype、daily energy、point forecast | “few-shot/专家”不新；直接支持 C2 竞争 |
| CNP / ECNP | context-set meta-learning、distribution/evidence | Gaussian/NIG，通用任务；无 capacity-count 与 forecast-origin legality | “神经过程/证据”不新 |
| Chen–Li–Zhu 2023 | beta-binomial、bounded count、covariates | INGARCH/seen series；无 held-out-site context posterior | “Beta-Binomial”不新 |
| OpenReview 2026 contextual TS neural process | 题名直接覆盖 contextual TS NP | 论坛/全文受人机验证阻塞 | **[Full Text Unverified]**，创新判定必须 conditional |

对 C1 的五项 preferred-nearest work（DeepAR、CNP、ECNP、Chen–Li–Zhu bounded count、Nikhal）检查了可访问的 reference lists、official related/citing pages 与 identifier 链；Google Scholar/Semantic Scholar forward-citation 界面不可访问，故 forward coverage 不是完整 citation index。可核链条包括：Nikhal 明确引用 van Etten 的 global EV 模型；ECNP 明确建立在 CNP 上；2026 OpenReview 题名显示 neural process 已向 contextual time-series forecasting 延伸；2026 behavior-guided preprint 把 EV 概率预测推进到持续在线适配。[UrbanEV data paper](https://doi.org/10.1038/s41597-025-04874-4) 还列出 HSTGCN 和 ChatEV 等点预测工作，CityEVCP 与同一作者体系/任务相邻。由此必须删除“首次跨站”“首次 few-shot”“首次 neural process”“首次容量分布”等宽泛措辞。

在当前可核材料中，尚未看到一项工作**同时**满足以下接口：站级 occupied-connector count；已知且变化的 $C_s$；完整离散 PMF；训练/测试站完全隔离；目标站只有固定 168 h 窗；每个 context 点只作非负查询相关伪计数更新；zero-shot 为空集退化；missing mask 不插值；proper score + calibration + sharpness 的 site-macro 评价。这是一个**检索可见、待全文和实验确认的窄 gap**，不是全球首次证明。

关键反证优先级：

1. OpenReview contextual TS NP 全文若已有同构的非负 pseudo-count Beta-Binomial 更新，CAMP-NP novelty 失败；
2. IEEE cold-start / Applied Energy 受限全文若已联合做未见站、有界 count PMF 和固定 context，必须重构；
3. EB-SBB 若与 CAMP-NP 等效，神经机制不必要；
4. 普通 BB-CNP 若同等好，单调伪计数不是独立贡献；
5. missingness strata 若主导全部 gain，论文应转向数据/观测机制而非 forecasting architecture。

## 4. Falsifiable Candidates and Selection

### 4.1 Evidence, Mechanism, Probabilistic Semantics, and Novelty Checks for 2–4 Candidates

候选生成前冻结的 problem evidence ledger：

| Problem evidence | Corresponding baseline failure | Representative existing solution | Solved part | Limitation/inapplicable condition here | Gap evidence | Sufficient to generate a candidate? |
|---|---|---|---|---|---|---|
| $Y$ 是 $0..C_s$ 的整数 [User Local Audit] | NB 给 $Y>C_s$ 正质量 | truncated NB；beta-binomial INGARCH | 有界支持/过离散 | 未解决未见站固定 context | 任务属性直接、可消融 | 是，C1 |
| few-shot/global EV 已有，但主要 point/daily | 全局模型不会显式解释 context evidence | CNP/ECNP；EV archetype experts | 少样本/不确定性或原型 | 通用连续输出；无 capacity/mask/legal-origin 联合接口 | 最近工作逐项缺一，但全文有访问缺口 | 是，C1/C2，novelty conditional |
| 站点异质性强是多篇文献共同动机 | 单一 PMF 可能平均不同原型 | hard clustering + TFT | point few-shot 原型 | proprietary daily kWh；hard assignment | mixture 是否必要可由 residual 诊断 | 是，C2 |
| 分布漂移会破坏 coverage | 固定模型可能 miscalibrated | ACI/online quantile adaptation | 在线 marginal coverage | 需持续 target labels；不修正完整 PMF | 只适合作控制/另制度 | 不作为主候选，C3 降级 |
| 城市站点存在空间关系文献 | 无图模型可能漏 spillover | HSTGCN/CityEVCP/INLA/DiffPLF | seen-site/区域时空依赖 | CRS 冲突、共址 ID、未来外生量非法、成本高 | 当前数据门禁未过 | 否，C4 拒绝 |

候选比较：

| Candidate | Source problem/gap | Falsifiable failure and evidence state | Competing root causes | Core intervention and formula | Probabilistic semantics | Cross-site regime | Nearest work | Pre/post novelty-check change | Substantive difference | Key control | Resources | Risk | Conclusion |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **C1 CAMP-NP** | capacity support + fixed few-shot evidence | [Falsifiable Hypothesis] NB 越界/尾部失配且普通 context 编码不足 | 多原型；missing drift | $a=a^0+\sum w_i y_i/C,\ b=b^0+\sum w_i(1-y_i/C)$，$w_i\ge0$，Beta-Binomial($C,a,b$) | 完整离散边际 PMF；精确 $0..C$ | few-shot 主、zero-shot 次 | ECNP、CNP、BB-INGARCH、DeepAR | 初稿“evidential BB-CNP”因 ECNP 重合，重构为 query-dependent 非负 pseudo-count 单调接口，并撤销 epistemic/aleatoric 首创主张 | 容量锚定 + context 增量不可为负 + 同一空/非空接口 + deployment legality | 普通 BB-CNP、EB-SBB、NB/truncated NB、同参数量 | 中 | 通用 CNP 已近似等效；exchangeability 可能错 | **首选，Conditional** |
| **C2 CBSAM**（Capacity-Bounded Soft Archetype Mixture） | 站点行为可能多峰 | [Falsifiable Hypothesis] 单一 BB 对某些站系统性双峰/重尾 | 只是模型容量增加 | $\sum_{j=1}^J\pi_j(\mathcal C) \operatorname{BB}(C,a_j,b_j)$，soft gate 仅看合法 context | 有界 mixture PMF；边际 | few-shot；zero-shot gate 用 source prior | Nikhal et al. hard archetype + mixture-of-experts | 初稿 hard cluster 被直接最近工作否定，改为 end-to-end soft probabilistic mixture，并降为第二 | 不把测试站全期 profile 聚类；gate 与 PMF 联合、soft uncertainty | 单 BB、hard EB cluster、参数匹配宽单模型 | 中高 | mixture 旧、可辨识性/塌缩、参数多 | **第二，Conditional** |
| **C3 target-adaptive conformal** | coverage drift | [Falsifiable Hypothesis] PMF ranking 好但 coverage 偏 | likelihood 错而非校准 | validation/online nonconformity 更新区间 | 区间，不是完整生成分布 | 只能 validation-site 或另列 online | ACI、dynamic conformal | 原作为 companion；因需持续标签且可宽化区间，降为独立强控制 | 无足够中心创新 | point+conformal、宽度/interval score | 低 | 保证范围在 shift 下有限 | **降级为控制，不计贡献** |
| **C4 graph diffusion** | 空间 spillover/联合场景 | 尚无 UrbanEV source-only residual 证据 | 坐标错误、未来天气泄漏 | 图 encoder + conditional diffusion | 联合样本候选 | seen/跨站混合不清 | CityEVCP、DiffPLF、INLA | 最近工作和数据 gate 后不再重构 | 当前无可辩护差异 | 无图、简单距离图、合法外生量 | 很高 | CRS/共址/成本/外生量 | **拒绝；满足触发条件才重开** |

候选级 nearest-work coverage（操作相似，不是题名计数）：

| Candidate | 至少三项 nearest work | 覆盖的比较轴 |
|---|---|---|
| C1 | DeepAR；CNP；ECNP；Chen–Li–Zhu BB-INGARCH；van Etten/Nikhal | global distribution、context map、evidence、bounded count、unseen/few-shot |
| C2 | Nikhal；van Etten；CNP；ECNP | hard archetype、global unseen site、soft context representation、uncertainty head |
| C3 | Gibbs–Candès ACI；Xu–Xie dynamic time-series conformal；Barber et al. beyond exchangeability | online update、temporal intervals、nonexchangeability |
| C4 | CityEVCP；DiffPLF；Bouaachra et al. INLA | graph/POI、scenario diffusion、probabilistic spatial field |

C2 的四项没有一篇同时实现合法 168 h soft gate + bounded mixture PMF；这只是 conditional gap。通用 mixture-of-experts 本身显然不新，因此 C2 的 novelty 低于 C1，必须靠 task-specific information boundary 和 validation multimodality 才能保留。

完整 research-to-innovation derivation：

| 固定链步骤 | 本报告的冻结决定 | 禁止的跳跃/控制 |
|---|---|---|
| 核对 UrbanEV 版本 | Dryad 2026-02-04 + DOI/论文/GitHub 交叉；用户哈希 | 不以名称相似或镜像替代 |
| 官方/最低适用门禁 | CONDITIONAL PASS；删除冲突输入 | 不因有文件就自动 PASS |
| 任务审计 | busy HH:55、有界整数、mask、六个月深圳 | 不叫 kW/kWh/arrival |
| 处理与协议 | raw mask-preserving、共址 group split、fixed episodes | 不用作者填充结果反推规则 |
| 推荐基线 | DeepAR-NB + empirical PMF + EB-SBB | 不只选弱基线 |
| 失败假设 | NB 越界/尾部与 context evidence 失配 | 不写成已观察事实 |
| 主/竞争根因 | 主：support/evidence；竞争：archetype/missing drift | 不把误差现象等同根因 |
| 区分诊断 | NB vs truncated NB vs BB-CNP vs CAMP；coverage/missing strata | 不在盲测后改诊断 |
| 干预点 | predictive distribution 与 context-to-parameter interface | 不先选 attention/graph |
| 2–4 候选 | C1/C2 保留，C3 降控制，C4 拒绝 | 不用数量保护弱想法 |
| 最近工作 | ECNP/CNP/BB、global EV、archetype、ACI、DiffPLF | 不以名字检索代替操作比较 |
| 最小可归因方法 | C1 只有 prior head + monotone pseudo-count + BB head | 不加 graph/calibrator 装饰 |
| 强替代控制 | 普通 BB-CNP、EB-SBB、参数/数据/校准匹配 | 不把额外参数算机制收益 |
| 条件完整首选 | CAMP-NP + substantive deployment protocol；后者单独判门禁 | 不预先声称两项贡献成立 |
| 预训练成功/否决 | macro-CRPS/NLL + calibration/sharpness + capacity/strata 联合规则 | 不看结果换 metric/claim |

### 4.2 Novelty Boundary, Alternative Explanations, Resources, and Falsifiability

按 b.md 的六问压缩 C1：

| Step | 回答 |
|---|---|
| 1. Failure hypothesis | [Falsifiable Hypothesis] DeepAR-NB 在高 utilization、小容量和少上下文 episode 可能出现越界质量、尾部 rPIT 偏斜和 macro-CRPS 劣化；证据来自 support mismatch 和文献机制，不是 UrbanEV 结果。 |
| 2. Root-cause hypothesis | 主因是 distribution support + context evidence interface；竞争因是多原型、missing drift、backbone capacity。 |
| 3. Intervention | 改 context 到 $(a,b)$ 的信息流：每个观测只能提供非负、query-related fractional successes/failures；不是添加通用 attention。 |
| 4. Novelty | 与 ECNP 的 NIG/continuous evidence、CNP 任意聚合、BB-INGARCH seen-series、EV archetype hard experts 均有明确操作/接口差；但 OpenReview 全文缺失，故 conditional。 |
| 5. Attribution | 最小 C1 对比 ordinary BB-CNP、EB-SBB、parameter-matched MLP、NB/truncated NB、更多 context 和 post-hoc calibration。 |
| 6. Falsification | 若 C1 对普通 BB-CNP 的 macro-CRPS 改善置信区间跨 0，或校准改善只靠更宽区间，或 EB-SBB 等效，或主要问题是 missing strata，则删除/改写主张。 |

资源复杂度以 batch $B$、context $K=168$、horizon 数 $H$、隐藏维 $D$、容量最大值 $C_{\max}$ 表示。C1 的 context-query 计算为 $O(BHKD)$，精确 PMF/evaluator 为 $O(BHC_{\max})$；参数增加主要是两个小 MLP/投影头，精确数在实现后由 parameter registry 给出。C2 再乘 mixture 数 $J$，因此预算和塌缩风险更高。C3 计算低但科学范围窄；C4 明显超预算。

### 4.3 Conditional Preferred Candidate, Second Candidate, and Ranking-Reversal Conditions

冻结的完整首选句：

> Under **[unseen-site few-shot with a frozen 168-hour adaptation window]**, based on **[UrbanEV official/user audit support, EV probabilistic-forecast literature, and DeepAR/CNP/beta-binomial mechanism analysis]**, assume that the baseline may exhibit **[capacity-violating tail mass and misuse of scarce target-site context, visible in macro-CRPS/NLL and discrete calibration]**, with competing root causes **[multimodal site archetypes and non-random telemetry missingness]**. We therefore propose the falsifiable method **[CAMP-NP]**, which changes **[probabilistic modeling and source-to-target information flow]** through **[a capacity-indexed Beta-Binomial predictive law whose query-dependent context contributions are nonnegative fractional success/failure pseudo-counts]**, with an expected improvement in **[site-macro proper scores, high-occupancy Brier score, and calibration at matched sharpness]**. Its falsifiable difference from the nearest work **[ECNP/CNP, beta-binomial bounded-count models, and EV few-shot archetype experts]** is **[the exact bounded count support plus monotone, deployment-legal pseudo-count update under a common zero/few-shot interface]**. If data audit does not support the problem, diagnostics support another root cause, or **[ordinary BB-CNP/EB-SBB is equivalent, gains vanish under matched capacity and budget, or intervals widen without proper-score gain]** occurs, **[switch to CBSAM for verified multimodality, demote to calibration/protocol work, or report the method claim as falsified]**.

排序反转采用验证集预规则：

| 观测（只允许 validation） | 决定 |
|---|---|
| NB 越界质量高；truncated NB 修复大部分 gain | 把贡献缩成 support-aware distribution；CAMP 需额外胜 BB-CNP 才保留 |
| ordinary BB-CNP ≈ CAMP，EB-SBB 也近似 | C1 核心失败；不以复杂度微小优势写创新 |
| 单 BB posterior predictive 检查呈稳定多峰，CBSAM 在参数匹配下同时改善 CRPS/校准 | C2 升首选 |
| point ranking 好、仅 marginal coverage 偏，conformal 不显著扩宽 | C3 成必要 companion，但不冒充生成模型 |
| error 与 missing coverage/月份几乎完全共变 | 转向观测机制/协议论文或 Not Ready |
| source-only residual 空间相关强、CRS 通过、合法图有独立增益 | 才重开 C4；不使用 blind test 触发 |

### 4.4 Innovation Organization, Substantive-Contribution Count, and Admission Gate

| Organization option | Central claim | Substantive contributions/count | Component relation | Increment over nearest work | Attribution evidence | Cost | Q3-level applied gate |
|---|---|---|---|---|---|---|---|
| Keep one deep mechanism | CAMP monotone bounded posterior | 1 个候选中心机制 | 一体：pseudo-count 与 BB support | 任务特定组合/接口 | 强，可 4–5 个控制 | 中 | **Conditional Pass** |
| **Central + necessary companion（选择）** | CAMP + deployment-valid benchmark protocol | 1 中心 + 1 条件性 protocol | protocol 使 few/zero-shot claim 可合法验证 | 相对 UrbanEV 现有 zone/time-CV 基准实质不同 | 方法消融 + processed/author protocol 对照 | 中 | **Conditional Pass** |
| Unified inseparable framework | 把 protocol 写进方法 | 名义 1 | 会模糊数据与模型归因 | 增量不清 | 弱 | 中 | Fail |
| Several orthogonal contributions | 再加图、conformal、天气 | 3–4 名义 bullets | 彼此不必要 | 多为已有模块 | 预算不足 | 高 | Fail |

“两项贡献”目前不是既成事实：

| 类型 | 候选贡献 | 是否计为 substantive | 门禁 |
|---|---|---:|---|
| central method | CAMP-NP 的 capacity-indexed monotone pseudo-count interface | **条件计 1** | 必须胜 ordinary BB-CNP/EB-SBB 且最近全文无同构 |
| necessary companion/protocol | raw snapshot、mask-preserving、grouped held-out-site、fixed deployment episodes、legal information/evaluation contract | **条件计 1** | 必须相对 UrbanEV 官方基准有独立分析价值和 protocol 对照 |
| routine data work | 下载、CSV 转 Parquet、常规清洗 | 0 | 基础设施 |
| calibration | rPIT/coverage/interval width | 0 | 正确评价，不是创新 |
| contribution bullets | 图、天气、conformal、更多 horizon | 0 | 已删除 |

内部 0–2 admission rubric：problem value 2、central novelty 1、task specificity 2、unification 2、necessity 2、attribution 2、falsifiability 2、evidence sufficiency 1、applicability boundary 1、integrity 2，总计 **17/20，Conditional Pass**。这是设计门禁，不是录用概率。未指定目标期刊，故 JCR Q3 与中国科学院三区的年份、学科分类和版本均为 **[To Verify]**；两者不得混用。

### 4.5 Multi-Agent Candidate Lifecycle, Referee Verdicts, and Unresolved Dissent

执行模式只有一个：**[Single-Agent Constrained Review]**。没有独立 agent、隐藏提案、投票或统计独立性。按三 checkpoint 执行：

| Checkpoint / role pass | Frozen input | Output | Verdict/reconstruction | Unresolved dissent |
|---|---|---|---|---|
| Evidence freeze | 官方版本、dataset.md、15F/6C、主合同、候选预算 4 | problem ledger，禁止图/未来变量 | 通过 | missing 机制仍未知 |
| Candidate generation | 同一 evidence pack | C1–C4（相关的同模型尝试） | C1/C2 保留；C3/C4 降级/拒绝 | C1 可能只是组合创新 |
| Novelty adversary | 冻结 C1 初稿 | ECNP、Nikhal、van Etten、OpenReview 反证 | 把“evidential BB-CNP”重构为 CAMP；删除 first/few-shot novelty | OpenReview 全文未核 |
| Falsification adversary | 冻结重构 C1 | ordinary BB-CNP、EB-SBB、missing/archetype 替代解释 | 加入 ranking reversal 和联合否决 | 一次数据集无法外部验证 |
| Referee/self-review | 冻结候选和 objections | Q3 gate、claim boundary、成本审查 | **Conditional Pass，非独立评审** | protocol 是否能成为第二贡献须实验后判断 |

候选 lifecycle：

| ID | 初始想法 | 最近工作/反方结论 | 状态 | Referee verdict | Ranking-reversal trigger |
|---|---|---|---|---|---|
| C1 | evidential BB neural process | ECNP 已覆盖 evidential CNP；原名过宽 | 实质重构为 CAMP-NP | 首选 conditional | BB-CNP/EB-SBB 等效则淘汰 |
| C2 | hard site archetype experts | Nikhal 已直接做 hard experts | 改为 soft bounded mixture，第二 | 可行但 novelty 较弱 | 验证到多峰且胜 C1 则升位 |
| C3 | adaptive conformal companion | ACI 已成熟且需 target feedback | 降为控制/另制度 | 不计创新 | 只有 calibration root cause 时纳入 |
| C4 | graph conditional diffusion | CityEVCP/DiffPLF/INLA 最近；数据 gate 失败 | 拒绝 | 当前不应实施 | CRS + residual spatial evidence + budget 全过 |

检索在 b.md 的 12-batch 上限停止，而不是声称完全饱和：R11–R12 仍强化了 conformal/online 边界，但未改变 C1/C2 排序。coverage gap 是 OpenReview 与两篇受限全文，以及未完成严格“两轮无新增”停止条件；投稿前必须做一次定时更新检索。当前继续无上限扩张仍不应替代 raw audit 与 baseline diagnostics。

## 5. Conditional Preferred Method: 容量锚定单调伪计数神经过程 / Capacity-Anchored Monotone Pseudo-count Neural Process

### 5.1 Problem, Notation, Evidence State, and Falsifiable Core Claim

简称 **CAMP-NP**。名称只用于实现和比较，题名检索未见同名并不能证明原创。

设：

- $s\in\mathcal S_{\mathrm{src}}\cup\mathcal S_{\mathrm{val}}\cup\mathcal S_{\mathrm{test}}$ 为站，三集合互斥；
- $C_s\in\mathbb N^+$ 为该 episode 可用的连接器容量；
- $e$ 为模拟部署 episode，context 长度 $K_e\in\{0,24,72,168\}$；
- $m_i\in\{0,1\}$ 为第 $i$ 个 context 小时是否观测；
- $y_i/C_s\in[0,1]$ 为已观测占用比例，但最终目标仍是原始 count；
- $x_i$ 含 calendar、相对 query 时间、mask/time-since-observed 和 $\log(1+C_s)$；
- $x_q$ 为 horizon $h\in\{1,3,6\}$ 的已知 query 特征；
- $\mathcal C_{s,e}=\{(x_i,y_i,m_i)\}_{i=1}^{K_e}$ 是**唯一合法**目标站信息。

目标是学习：

$$
p_\theta(Y_q=k\mid x_q,C_s,\mathcal C_{s,e}),\quad
k=0,\ldots,C_s .
$$

**证据状态**：target support 与 missing/zero 分离来自 [User Local Audit]；全局/少样本/有界分布的合理性来自文献；CAMP-NP 在 UrbanEV 上更优、单调伪计数更必要均为 **[Falsifiable Hypothesis]**。它不是已验证 Bayesian posterior，也不宣称把 epistemic 与 aleatoric uncertainty 唯一分解。

冻结的实践成功标准：

1. 相对验证后最强的 fair control（可能是 EB-SBB 或 ordinary BB-CNP），blind-test site-macro CRPS 相对改善至少 2%，且 paired site-block 95% CI 的 CAMP-minus-control 上界小于 0；
2. 至少 2/3 主 horizons 满足第 1 条，且任何主 horizon 不允许 CRPS 显著恶化；
3. site-macro NLL 同向或统计非劣；高占用阈值 $Y/C\ge0.8$ 的 Brier score 不恶化；
4. 80% interval 的绝对 coverage error 不比最强 control 差超过 1 percentage point，normalized width 不增加超过 5%；
5. CAMP 输出的越界概率严格为 0、PMF 求和误差低于 evaluator 的数值容差；
6. 对 ordinary BB-CNP 的增益必须在 parameter-matched、相同 seed/batch/episode 下仍存在。

2%、1 pp、5% 是 **[Planning Decision]** 的 practical-equivalence margins；可在任何正式 run 前依据应用方成本函数一次性改写，但改写及理由必须进入 preregistration，不能看 validation/model ranking 后调整。

### 5.2 Core Mechanism, Probabilistic Support, Tensor Flow, and Complexity

先由 source-trained prior head 产生正参数：

$$
(a_q^0,b_q^0)=\epsilon+
\operatorname{softplus}\!\left(g_\theta(x_q,\log(1+C_s))\right),
\qquad \epsilon=10^{-4}.
$$

context 用因果编码器产生 $e_i=f_\theta(e_{i-1},x_i,m_i y_i/C_s)$。query 对每个已观测点给一个有上界的非负伪计数强度：

$$
u_{iq}
=m_i\,u_{\max}\,
\sigma\!\left(r_\theta(e_i,x_q,\Delta t_{iq})\right),
\qquad 0\le u_{iq}\le u_{\max}.
$$

最终 Beta 参数为：

$$
a_q=a_q^0+\sum_{i=1}^{K_e}u_{iq}\frac{y_i}{C_s},
\qquad
b_q=b_q^0+\sum_{i=1}^{K_e}u_{iq}
\left(1-\frac{y_i}{C_s}\right).
$$

对固定 query、capacity 与已存在 context，**在时间尾端追加**一个合法观测时，旧的 causal states/weights 不变，新点只会给 $a_q,b_q$ 添加非负量；这就是结构性“证据单调”。它不适用于回填早先 missing 值、改写既有 context、改变 query 或从另一窗口重新编码。它也不保证 predictive variance 对每个新点必然下降，因为新增观测可能改变均值并揭示冲突；只保证按时间追加时总 pseudo-count $a_q+b_q$ 的 context 增量不减少。$u_{\max}$ 只用 validation 冻结，防止单点制造无限确信。

层次预测写为：

$$
p_q\sim\operatorname{Beta}(a_q,b_q),\qquad
Y_q\mid p_q,C_s\sim\operatorname{Binomial}(C_s,p_q),
$$

因此：

$$
P(Y_q=k)=
\binom{C_s}{k}
\frac{B(k+a_q,C_s-k+b_q)}{B(a_q,b_q)},
\quad k=0,\ldots,C_s.
$$

$$
\mathbb E[Y_q]=C_s\frac{a_q}{a_q+b_q},\qquad
\operatorname{Var}(Y_q)=
\frac{C_s a_q b_q(a_q+b_q+C_s)}
{(a_q+b_q)^2(a_q+b_q+1)}.
$$

Beta-Binomial 的 conditional exchangeability 是模型假设：连接器受共同站点行为驱动时可以产生 extra-binomial variation，但快/慢充、维护和排队会违反同质性。该假设必须与 categorical PMF、zero-inflated BB 和 C2 mixture 做 posterior predictive checks，不能由公式自动成立。

数值实现用 <code>lgamma</code> 计算 log-combination 与 log-Beta；$a,b$ 下限为 $10^{-4}$，float64 evaluator 复核 PMF normalization。零值由 $k=0$ 的合法概率处理，不加 log1p、不 zero-impute。每站不同 $C_s$ 时，以 mask padding 到 batch 的 $C_{\max}$；padding 概率在 normalization 前为 $-\infty$。

张量流：

| 张量 | shape | 内容 |
|---|---|---|
| context_x | $[B,K,F]$ | calendar、相对时间、capacity、mask |
| context_y | $[B,K,1]$ | $y/C$，missing 处数值占位但 mask=0 |
| context_h | $[B,K,D]$ | causal encoder state |
| query_x | $[B,H,F_q]$ | 1/3/6 h future-known features |
| pseudo_weight | $[B,H,K]$ | $u_{iq}\ge0$ |
| alpha_beta | $[B,H,2]$ | $a_q,b_q$ |
| log_pmf | $[B,H,C_{\max}+1]$ | capacity mask 后的离散 PMF |

时间复杂度约为 $O(BKD^2+BHKD+BHC_{\max})$，显存为 $O(BKD+BHK+BHC_{\max})$。精确参数数、峰值显存、吞吐和 $T_{\mathrm{core}}$ 均须 smoke test 记录；不能从公式推断运行时间。

训练 objective 只有 site-balanced query NLL：

$$
\mathcal L_{\mathrm{NLL}}
=\frac{1}{|\mathcal B_S|}
\sum_{s\in\mathcal B_S}
\frac{1}{|\mathcal Q_s|}
\sum_{q\in\mathcal Q_s}
-\log P_\theta(Y_q=y_q).
$$

weight decay 是优化正则而不是第二创新 loss。主模型不加 coverage penalty、contrastive loss、graph loss 或 post-hoc calibration loss，防止核心归因被掩盖。

### 5.3 seen/zero-shot/few-shot Behavior and Baseline Interface

| 模式 | context | 参数行为 | 可支持的 claim |
|---|---|---|---|
| seen-site diagnostic | 同站合法过去窗 | 与 few-shot 相同 forward | 只诊断，不算跨站 |
| zero-shot | $K=0$ | $a=a^0,b=b^0$ | 容量+日历源站先验 |
| few-shot | 固定 $K\le168$ | 加非负 fractional pseudo-count；$\theta$ 冻结 | 目标站短历史适配 |
| online | 持续加入新标签 | 本报告主方法禁止 | 若另建 protocol 必须另报 |

任何 target-site gradient step、用 query truth 重算 scaler、在盲测站选 $u_{\max}$ 或 cluster 都使 few-shot 失效。

与 DeepAR 的 fair interface：

- 共用同一 episode loader、calendar encoding、capacity/mask、hidden width候选、source-site sampler、optimizer budget 和 checkpoint rules；
- DeepAR 用 causal RNN + NB autoregressive head；CAMP 保留 causal context encoder，但用 query-to-context 非负 pseudo-count head和 direct Beta-Binomial decoder；
- immediate 1/3/6 h query 保证 DeepAR 无需跨长时间无标签递推；
- ordinary BB-CNP 采用同 encoder/decoder 宽度，但允许 unrestricted pooled representation 直接输出 $a,b$，用来隔离“有界 likelihood”与“单调信息流”；
- parameter-matched control 通过调节隐藏宽度使参数量落在 CAMP 的 ±2% 内；若做不到，报告绝对差并以训练 FLOPs/steps 再匹配。

### 5.4 Training, Calibration, Inference, and Adaptation Pseudocode

以下全部是 **[Research Plan to Validate; Not Executed]**。日期清单表示 168 h context 的开始；forecast origin 是该窗口末端之后，不能把二者混为同一 timestamp。

完整 key data-to-experiment path：

~~~text
INPUT OFFICIAL RECORDS:
    Dryad DOI/version-23 file URLs and expected bytes/SHA-256/license
    official README, data-paper DOI, repository commit
INPUT RAW RECORD:
    station_id, timestamp_naive, busy, idle, fast_busy, fast_idle,
    slow_busy, slow_idle, prices, duration, volume
INPUT STATION RECORD:
    station_id, longitude, latitude, fast_count, slow_count,
    charge_count, TAZ
OUTPUT EVENT RECORD:
    event_id, coordinate_group, split, site_id, context_start,
    forecast_origin, query_time, horizon, capacity, target,
    target_observed, context_coverage, legal_feature_vector

ACQUIRE:
    download the frozen official payload into immutable raw/version-23
    verify DOI, version, license, byte size, SHA-256 before extraction
    inventory extracted files and repository commit; save raw_manifest.json

LOAD_AND_AUDIT:
    stream every station CSV under an explicit schema
    parse timestamps without silently assigning an offset
    assert sorted unique 5-minute grid; log duplicates/conflicts, never average silently
    distinguish six-status-missing from observed busy=0
    verify integral nonnegative statuses and busy+idle=charge_count
    verify fast/slow reconciliation and 0<=busy<=capacity
    stop before modeling on any unexplained support or identity violation
    save schema.json, station_manifest.csv, missingness_report.parquet,
         support_violations.parquet, exclusion_log.csv

CONSTRUCT_TARGET:
    adopt Asia/Shanghai only after the frozen timezone check
    select exact HH:55 rows without fill, interpolation, or carry-forward
    set target=busy when observed; otherwise target_observed=false
    retain every clock slot and missing mask
    save hourly_hh55.parquet with
         [site_id, timestamp, capacity, target, observed_mask]

SPLIT_BEFORE_TRANSFORMS:
    group exact-coordinate station IDs
    assign groups by frozen hash to source/validation/blind 60/20/20
    fit every scaler/bin/imputer on source-train only
    primary target needs no target scaler
    save split_manifest.json and transform_registry.json

BUILD_EPISODES:
    source train timestamps=2022-09-01..2022-11-30
    validation episodes use December without touching blind stations
    after config freeze, optionally refit source sites through 2022-12-31
    blind context_start dates=[2023-01-01, 01-15, 02-01, 02-15]
    expose exactly 168 clock hours for few-shot and zero hours for zero-shot
    set forecast_origin=end of the 168-hour window
    create query timestamps at origin+{1,3,6} hours
    do not select episodes/sites using future query availability or truth
    save episodes_v1.parquet and event_manifest.json

BUILD_TENSORS:
    context_x:[B,168,F], context_y:[B,168,1], context_mask:[B,168,1]
    query_x:[B,3,Fq], capacity:[B,1], target:[B,3], score_mask:[B,3]
    allow only origin-known calendar, capacity, past target/mask
    reject realized future weather, future prices, station ID, full-site profiles

TRAIN_AND_SELECT:
    run seasonal/EB controls and train DeepAR-NB, truncated-NB,
        ordinary BB-CNP, CAMP-NP, CBSAM under the frozen budget
    use source loss and validation-site early stopping/config gates only
    save config, environment, code/checkpoint hashes, learning curves
    never inspect blind labels or evaluator outputs during selection

CALIBRATE:
    primary predictions remain uncalibrated
    optional scalar discrete temperature is fitted by horizon on validation sites only
    save calibrator parameters/hash; never fit on blind context query labels

INFER:
    assert target site/group absent from source and validation
    zero-shot passes an empty context; few-shot passes only the frozen 168-hour tensor
    freeze all parameters; output log_pmf:[B,3,Cmax+1] plus capacity mask
    save long predictions with checkpoint/input/config hashes

EVALUATE_AFTER_ONE-TIME_UNSEAL:
    assert target support and PMF normalization
    compute discrete CRPS, NLL, randomized PIT, 50/80/95 coverage,
        normalized width, interval score, threshold Brier, mean/median MAE
    aggregate event -> episode -> site, then site-macro
    stratify only by frozen horizon/capacity/coverage/utilization/month/missingness
    run paired site-first, episode-block-second bootstrap shared across models
    save evaluation_manifest.json, metric_long.parquet, bootstrap.parquet,
         tables, plots, failure_registry.json
CHECKPOINT:
    every artifact carries study_id, source hash, split hash, config hash, code commit
~~~

训练：

~~~text
INPUT: frozen source sites, source time range, episode calendar,
       context lengths {0,24,72,168}, horizons {1,3,6}
FREEZE: site sampler, seeds, feature allowlist, capacity field

for epoch:
    sample sites uniformly
    for each site, sample a source-only deployment episode
    sample K without looking at query targets
    build K clock-aligned context points with observed_mask
    encode context causally
    for each horizon:
        compute source prior a0,b0 from legal query features
        compute nonnegative bounded u_iq for observed context points
        add fractional occupied/free pseudo-counts
        compute exact beta-binomial log probability of query count
    average NLL first within site, then across sites
    update theta; gradient clip using frozen threshold
    evaluate only on validation sites/episodes
    checkpoint by validation site-macro NLL
stop by pre-frozen patience; retain every formal run, including negative runs
~~~

few-shot/zero-shot inference：

~~~text
load frozen checkpoint and evaluation manifest
for each blind-test station and episode:
    verify station never appeared in source/validation
    if regime == zero-shot:
        context = empty
    if regime == few-shot:
        expose exactly the frozen 168-hour context and mask
        reject any tensor from before/after that window
    run one forward pass; never update theta
    return exact PMF for 1h, 3h, 6h (12h sensitivity)
    store PMF hash, checkpoint hash, input episode id
~~~

公平校准：

1. 主表首先报告 uncalibrated PMF；
2. 作为独立控制，可为每种方法、每个 horizon 在 validation sites 上拟合一个 scalar discrete temperature $T_h>0$：

$$
\tilde p_{qk}=
\frac{p_{qk}^{1/T_h}}{\sum_{j=0}^{C_s}p_{qj}^{1/T_h}}.
$$

3. calibration method、目标（validation NLL）、bounds、seed 在 blind test 前冻结；
4. 报告 base 与 +cal 两行；不能把 +cal 的 gain 全归给 CAMP；
5. target-site context 可用于 CAMP forward，但不能再用 query labels 拟合 $T_h$。持续 ACI 属另一个 online regime。

evaluation：

~~~text
for each scored target:
    assert 0 <= y <= capacity and PMF sums to one
    NLL = -log PMF[y]
    discrete_CRPS = sum_k (CDF[k] - I[y <= k])^2
    randomized_PIT = CDF[y-1] + fixed_uniform(event_id) * PMF[y]
    form 50/80/95% randomized equal-tail intervals
    compute coverage, normalized width, interval score
    compute Brier for utilization >= 0.5 and >= 0.8
    compute MAE of predictive median and mean
aggregate within site, then macro across sites
bootstrap sites first and episode blocks second
report few-shot and zero-shot separately
~~~

randomized PIT 中的 uniform 由 evaluation event id 与公开 seed 的确定性 hash 生成，避免反复随机化挑好 histogram。离散 CRPS 在 $k=0,\ldots,C_s-1$ 精确求和。NLL 只比较同一 count measure 且 normalization 正确的模型。

### 5.5 Differences from Nearest Work, Applicability Boundary, and Elimination Conditions

| 最近工作 | CAMP-NP 的差异 | 不允许的夸大 |
|---|---|---|
| DeepAR | exact bounded PMF；direct query；monotone context pseudo-count | 不称首次 global probability forecast |
| CNP | 每个 context 点的 success/failure 增量显式非负，不是任意 pooled embedding 后输出连续 Gaussian | 不称首次 context-set forecast |
| ECNP | Beta-Binomial capacity count；不声称 NIG 式 uncertainty decomposition；deployment time-series protocol | 不称首次 evidential neural process |
| BB-INGARCH | source-learned query prior + held-out-site amortized context；统一 zero/few-shot | 不称首次 beta-binomial time series |
| van Etten | hourly occupancy PMF、168 h episode、proper calibration | 不称首次 unseen EV station |
| Nikhal | soft evidence/完整 PMF，不用全期 profile/hard cluster；C2 才是 mixture | 不称首次 few-shot EV |
| behavior-guided online | 固定窗、无后续标签、无 LLM/online update | 不把 offline few-shot 结果外推在线 drift |

适用边界：

- 仅对相同数据采集系统内、容量已知的新站 occupied-connector snapshot；
- capacity 在 episode 内稳定；若连接器增减，必须使用时间变 capacity 并重新定义支持；
- calendar/shared source pattern 对目标站仍有信息；
- missing context 可显式识别，但 missing 原因无需被解释为需求；
- 输出为边际 PMF，不用于需要跨时联合轨迹的储能/电网优化；
- 当前证据只支持深圳 collection，不支持跨城市、运营商或人口泛化。

直接淘汰/重构条件：

1. raw support 审计失败或 capacity 无可靠映射：淘汰 CAMP；
2. ordinary BB-CNP 或 EB-SBB 达到 practical equivalence：单调机制淘汰；
3. CAMP 只改善 MAE、不改善 proper score：概率主张淘汰；
4. coverage 靠区间显著变宽才改善：校准主张淘汰；
5. C2 在预冻结多峰诊断和参数匹配下稳定胜出：切换首选；
6. 低 coverage/月度缺失完全解释方法排名：转成 missingness/protocol 研究；
7. blind test 被用于任何调参或 feature 修改：该盲测永久作废，不能“重新封存”。

## 6. Post-Setup Validation and Engineering Route

### 6.1 Download and First Raw-Data Audit

本报告不把本地审计摘要替代成当前环境里的原始数据。首个工程动作是从冻结的 Dryad v23 下载，校验官方 ZIP、README、代码归档和 manifest 的 hash，然后重新生成审计结果；只有与 [User Local Audit] 一致或差异有可解释的版本原因，才允许建模。推荐命令与产物的等价逻辑如下，具体 CLI 可按环境替换，但不得跳过任何断言：

~~~text
freeze release DOI, version id, access date, license, source URLs
download into immutable raw/version-23/
verify expected sha256 for README, ZIP, and repository archive
unzip read-only; inventory every file with byte size and sha256
parse station filename, station id, coordinates, timestamp, capacity, six statuses
assert one station id maps to one declared coordinate/capacity history
reconcile status sum against capacity at every nonmissing timestamp
mark, never silently impute, all-six-status-missing rows
aggregate 5-minute snapshots to HH:55 by a frozen timestamp rule
write Parquet plus schema.json, station_manifest.csv, audit.json, and exclusion_log.csv
compare counts, time span, missingness, and station coverage with the frozen audit
stop on unexplained differences before constructing any split
~~~

第一轮 raw audit 的硬性检查：

| 检查 | 通过条件 | 失败动作 |
|---|---|---|
| release identity | DOI、version、字节数、hash、许可证均与 manifest 一致 | 停止；不自动改用“最新”版本 |
| station inventory | station id 唯一、文件可解析、station 数可复现 | 列出差异；先解决版本或损坏 |
| time grid | timezone 明示；5 min grid 与 HH:55 规则无重复/倒序 | 停止并修正时间语义 |
| capacity/support | $C_{s,t}$ 为非负整数且 $0\le Y_{s,t}\le C_{s,t}$ | CAMP 与所有 count likelihood 均停止 |
| status reconciliation | 非缺失行的六状态和与连接器总数关系可解释 | 输出冲突矩阵；不能把冲突当需求 |
| missingness | 区分全状态缺失、局部状态缺失、真零、站点未上线 | 建立 missing reason；禁止零填充 |
| location grouping | 精确重复坐标与 station id 关系有报告 | split 前建立 group id |
| auxiliary metadata | CRS、POI 日期、天气时间语义和 join rate 可复现 | 对有疑义字段先禁用 |

验收产物必须至少包含：

- <code>raw_manifest.json</code>：URL、access time、version、license、byte size、SHA-256；
- <code>schema.json</code>：字段、dtype、unit、timezone、合法取值、missing code；
- <code>station_manifest.csv</code>：station、coordinate group、capacity history、首末时间、coverage；
- <code>aggregation_audit.json</code>：5 min 与 hourly 行数、规则、丢弃原因和 support violations；
- <code>split_manifest.json</code>：group hash、source/validation/test station 列表、episode origins；
- <code>feature_manifest.json</code>：每个 feature 的来源时间、availability time、transformation fit set；
- <code>environment.lock</code>、checkpoint hash、prediction manifest 和 evaluation manifest。

任何人工修补均须进入 append-only <code>exclusion_log.csv</code>，包含 station、timestamp、原值、动作、原因和操作者；不得覆盖 raw 文件。当前 [User Local Audit] 的 1,682 个站、87,679,296 个 5-min 行、69,123,930 个有效行、18,555,366 个全六状态缺失行以及 HH:55 审计值只是重跑时的预期对照，不是无需重跑的验收证明。

### 6.2 Baseline Smoke Test and Failure-Hypothesis Decision Gate

smoke test 只回答“pipeline 与假设是否值得进入正式实验”，不产生论文结果。它只使用 source-train 与 validation sites，不触碰 blind-test labels：

1. 从 source-train 抽取 10 个站、validation 抽取 5 个站，覆盖低/中/高 capacity 与 missingness；
2. 运行 seasonal empirical、direct NB-MLP、DeepAR-NB、truncated DeepAR-NB、EB-SBB、ordinary BB-CNP、CAMP-NP 各 2 个 epoch；
3. 对每个 batch 单元测试 PMF normalization、support、mask、context/query 时间顺序与 station disjointness；
4. 记录 NB 在真实 capacity 之外的概率质量
   $$
   M_{\mathrm{out}}=\Pr_{\mathrm{NB}}(Y>C_s)=1-F_{\mathrm{NB}}(C_s);
   $$
5. 绘制/保存但不挑选 randomized-PIT、reliability、residual–capacity、residual–missingness 与 context-length 曲线；
6. 以同一 evaluation code 计算 discrete CRPS、NLL、interval score 和 MAE。

进入正式实验的 decision gate：

| 观察 | 对根因的含义 | 决定 |
|---|---|---|
| raw 中出现 $Y>C_s$ 或 capacity 不可追踪 | 目标/支持定义错误 | **Stop**；先修数据合同 |
| DeepAR-NB 的 $M_{\mathrm{out}}$ 实质性且 truncated NB/BB 改善 proper score | bounded-support 根因得到支持 | 保留 C1 与 support controls |
| truncated DeepAR-NB 已与 CAMP practical equivalent | support 而非 context 机制足够 | 缩小或淘汰 CAMP 主张 |
| ordinary BB-CNP 与 CAMP practical equivalent | 单调伪计数核心无增益 | 淘汰 C1；不靠增加层数挽救 |
| PIT 呈多峰/站点亚型，C2 的 validation CRPS 稳定更好 | 单一 Beta-Binomial 不足 | 切换 CBSAM 为首选 |
| 差异只存在于低 coverage 或一个月份 | missingness/temporal shift 是竞争根因 | 先改协议；方法 gate 暂停 |
| 只改善 MAE、NLL/CRPS 不改善 | 不是概率预测贡献 | 淘汰 probabilistic claim |
| 任一方法依赖 query future labels、全期 profile 或 test-site scaling | 泄漏 | 结果无效并重建 split |

“实质性/practical equivalence”不在看完结果后决定。默认采用 Section 5.1 的 2% relative CRPS margin；只有在查看任何 model-comparison validation 结果前，才能依据外部应用成本函数一次性改写并冻结 $\delta_{\mathrm{CRPS}}$。同时报告绝对差与相对 DeepAR-NB 的差；若没有可辩护的业务阈值，就只作统计不确定性与效果量解释，不把 $p>0.05$ 等同“等效”。

### 6.3 Minimal Method, Alternative Controls, and Final-Method Experiments

**配置矩阵。** 最小可审计正式矩阵为下表；每个 learned method 默认 3 个公开 seeds，完全相同的 split、episode、legal features 与 evaluator。非学习 seasonal empirical 不重复 seed。

| ID | 配置 | 隔离的机制 | 正式地位 |
|---|---|---|---|
| E0 | seasonal empirical PMF | 非神经、周期信息 | sanity floor |
| E1 | EB-SBB | bounded support + shrinkage，无 amortized context | 强简单 baseline |
| E2 | direct NB-MLP | 同信息、无 autoregressive recurrence | 结构 control |
| E3 | DeepAR-NB | 公开可复现的主 baseline | primary baseline |
| E4 | truncated DeepAR-NB | 仅修复 support | support control |
| E5 | ordinary BB-CNP | bounded PMF + context set，无单调伪计数约束 | parameter/info-flow control |
| E6 | CAMP-NP | C1 全机制 | preferred candidate |
| E7 | CAMP-NP, context weights constant | 去除 query-specific relevance | relevance ablation |
| E8 | CAMP-NP, capacity prior removed/replaced by shared prior | 去除 capacity anchoring | prior ablation |
| E9 | CBSAM | C2 mixture alternative | predeclared switch candidate |
| E10 | point N-HiTS + independent validation conformal | point + post-hoc uncertainty | alternative paradigm |

E5 与 E6 的 encoder depth、hidden width 和参数量须尽量匹配；若无法精确匹配，报告参数量、FLOPs 与 wall time，并加一个 width-adjusted control。E4 必须使用正确 renormalized PMF，而不是把 $Y>C_s$ clip 到 $C_s$。E10 的区间不能被伪装成完整 PMF；因此只比较共同定义的 interval/point metrics，不能比较 NLL。

**实验阶段与一次性盲测：**

1. **Unit/Smoke**：只检验实现；
2. **Pilot**：source-train 训练、source-validation/validation sites 做单因素选择与 gate；保存所有尝试；
3. **Freeze**：签署 config、代码 commit、container digest、split/evaluation manifest；
4. **Formal blind test**：一次执行所有冻结 E0–E10；失败 job 可按预写 retry 规则重启，但不能改 config；
5. **Analysis**：按预写 strata、bootstrap 与 plots 自动生成；
6. **External robustness**：只有另一数据集通过独立 contract 后才执行，且不回流调节 UrbanEV。

推荐核心预算：E2–E10 的 9 个随机训练配置 × 3 seeds = 27 个训练，确定性的 E0/E1 各执行一次，合计 29 个核心 runs；另加最多 6 个预注册 sensitivity runs（12 h、24/72 h context、低 coverage、first-episode-only），不是无限 ablation。若预算只能支持最低证据集，优先 E0/E1/E3/E4/E5/E6/E9：5 个随机模型 × 3 seeds，加 2 个确定性配置，共 17 个 runs。这足以检验 support、ordinary context、单调机制与 mixture switch，但不能支持完整 ablation 叙述。

每个 run 输出长格式 predictions：event id、site、episode、origin、horizon、capacity、truth、PMF 或 interval、context coverage、seed、checkpoint hash。所有 tables 由 prediction files 生成，禁止手工抄数。

### 6.4 Baseline-Recipe Fair Start, Conditional Final Recipe, and Resource-Reduction Order

**公平起点。** DeepAR-NB 先按 GluonTS 公开默认结构的可复现近似起点执行：2 层、40 hidden units、dropout 0.1；所有自研神经方法从相同 hidden scale、Adam、learning-rate candidate、gradient clipping、batch sampler、最大 epoch 与 early-stop rule 起步。所有方法均直接预测 count，不对目标用 target-site 全期 mean/variance 标准化；capacity 只能作为当时已知输入或 likelihood support。

规划初始 recipe（带注释的值均只可在 validation 冻结）：

~~~yaml
data:
  frequency: 1h
  context_hours: 168
  horizons: [1, 3, 6]
  sensitivity_horizon: 12
  target: occupied_connector_count
  missing_policy: masked_no_zero_fill
split:
  site_group: exact_coordinate_group
  assignment: deterministic_hash_60_20_20
  source_train: 2022-09-01_to_2022-11-30
  source_validation: 2022-12-01_to_2022-12-31
  blind_context_starts: [2023-01-01, 2023-01-15, 2023-02-01, 2023-02-15]
model:
  hidden_size: 40
  layers: 2
  dropout: 0.1
  distribution: beta_binomial
  u_max: validation_frozen
optimizer:
  name: Adam
  learning_rate: 0.001
  gradient_clip_norm: 1.0
  max_epochs: 100
  early_stop_patience: 10
sampling:
  batch_episodes: 64
  sites_uniform_first: true
  query_points_per_episode: all_frozen_horizons
calibration:
  primary: none
  optional: scalar_discrete_temperature_by_horizon_on_validation
reproducibility:
  seeds: [202601, 202602, 202603]
  deterministic_evaluation: true
~~~

若 batch 64 OOM，只可按 64→32→16 顺序减小并保持 gradient accumulation 后的有效 batch；这属于资源适配，不得看 blind score 决定。学习率只允许在 validation 上按 $10^{-3}$ 后 $3\times10^{-4}$ 的顺序触发式尝试：第一值若训练发散或 validation NLL 连续非有限才用第二值，不能事后选较好者。$u_{\max}$ 先设 1；若 validation 中单点 effective sample size 爆炸或 calibration 明显恶化，才尝试 0.25 并冻结。CBSAM 先 $J=2$，只有预注册多峰诊断成立且两分量不能解释时才在 validation 尝试 $J=4$。

**机制变更—风险—补偿—诊断：**

| 从 baseline 的变更 | 新风险 | 公平补偿 | 必做诊断 |
|---|---|---|---|
| NB→Beta-Binomial | gain 可能只来自合法 support | E4 truncated NB、E1 EB-SBB | $M_{\mathrm{out}}$、tail CRPS |
| recurrence→context/query | 参数与信息流不同 | E2/E5、参数匹配、同 legal window | context shuffle、K=0 |
| monotone pseudo-count | 过度确信、冲突 context 无法撤回 | $u_{\max}$、constant-weight ablation | ESS vs K、conflict cases |
| capacity prior | capacity proxy 可能主导 | E8、capacity strata | prior/posterior shift |
| optional temperature | gain 来自后处理 | base/+cal 分列 | NLL、coverage、width 同报 |
| mixture C2 | collapse/label switching | 最小 $J=2$、同预算 | component mass、entropy、seed stability |

**资源削减顺序。** 若计算/工期不足，按以下次序缩减，不能先删除反证 controls：

1. 删除 12 h sensitivity；
2. 删除 weather-on 辅助实验，保留 weather-off 主表；
3. 删除 E7/E8 中次要 ablation，但至少留 ordinary BB-CNP；
4. 将非首选方法 pilot 限于 1 seed，但正式最低证据集仍保留 E3/E4/E5/E6/E9 的 3 seeds；
5. 缩短 source pilot，不缩小 blind sites；
6. 若仍超预算，停止并将工作降级为 desk-study/protocol paper，不用单 seed 小样本声称胜出。

### 6.5 Metrics, Calibration, Statistical Uncertainty, Freeze, and Fallback Rules

**主次指标。**

- 主指标：site-macro discrete CRPS；先在同一 site 内按 episode/horizon 聚合，再对 site 等权；
- 共同 proper secondary：NLL；对数概率采用数值稳定实现并报告非有限事件；
- calibration：randomized-PIT histogram/KS 仅作诊断，50/80/95% empirical coverage、normalized width、interval score；
- decision events：利用率 $Y/C\ge0.5$ 与 $\ge0.8$ 的 Brier score 与 reliability；
- point secondary：predictive median/mean 的 MAE；不使用遇零不稳定的 MAPE；
- scale reporting：同时给 count scale 与按 capacity 归一化的 score；主结论以预冻结版本为准；
- E10 等无完整 PMF 的方法只进入 coverage/width/interval score/MAE 表。

正式主比较为 CAMP-NP vs DeepAR-NB 的 paired site-macro CRPS；关键机制比较为 CAMP-NP vs truncated DeepAR-NB、ordinary BB-CNP、EB-SBB；ranking-reversal 比较为 CAMP-NP vs CBSAM。每项同时给绝对差、相对差、每站 paired difference、95% uncertainty interval，不只给 win/loss。

**统计不确定性。** 训练随机性用 3 seeds；抽样不确定性用 paired hierarchical bootstrap：先以 station 为 block 有放回抽样，再在站内以 168 h episode 为 block 抽样，所有模型共享同一 bootstrap draw，建议 10,000 次。区间是数据/episode sampling uncertainty 的描述，不替代多 seed；seed 结果分列并给 seed-mean。若站数太少或 bootstrap 分布异常，追加 station-level permutation/randomization sensitivity，但不以不同检验挑显著性。预注册 strata 仅包括：

- horizon 1/3/6 h（12 h sensitivity）；
- source-only capacity quartiles 固定的 capacity bands；
- context coverage $[0.8,0.9),[0.9,0.99),[0.99,1]$；
- source-only utilization bands；
- January/February；
- source-defined high/low missingness。

任何事后发现的 subgroup 明确标为 exploratory。多项 secondary comparisons 用 Benjamini–Hochberg 仅作辅助；主比较不因校正而重新定义。统计显著但小于预冻结 $\delta_{\mathrm{CRPS}}$ 的结果不得称 practical improvement。

**blind-test 前冻结清单：**

1. dataset version、raw hashes、parser commit、timezone、hourly aggregation；
2. target/status mapping、capacity rule、missing rule、合法 exogenous set；
3. coordinate grouping、site split、episode origins、coverage strata；
4. 所有模型结构、参数预算、optimizer、stop/retry、seed；
5. PMF support、calibration policy、primary/secondary metrics；
6. practical-equivalence threshold、bootstrap、plots、rounding；
7. C1 淘汰条件、C2 switch 条件与失败 fallback；
8. container digest、code commit、config hashes 与签署时间。

**fallback rules：**

| 触发 | fallback | 可保留的论文边界 |
|---|---|---|
| CAMP 不胜 DeepAR-NB | 报负结果与 bounded baseline；不改盲测 | protocol/support benchmark |
| E4≈CAMP 且胜 E3 | 选 truncated DeepAR-NB | bounded support 是结论 |
| E5≈CAMP | 选 ordinary BB-CNP | context adaptation 可写，monotone mechanism 不可写 |
| E9 稳定胜 E6 且多峰诊断成立 | 预注册切换 CBSAM | mixture archetype conditional claim |
| 所有复杂法≈E1 | 选 EB-SBB | 简单 shrinkage result |
| zero-shot 有效、few-shot 无增益 | 取消 adaptation claim | global cross-site prior result |
| few-shot 只在低 quality context 变差 | 增加 abstention/quality flag，不改主模型 | deployment-quality finding |
| data audit 或 leakage 失败 | 全部模型结论撤回 | 仅保留 desk research 和修复后的新协议 |

## 7. Risks, Paper Boundary, and Readiness

### 7.1 Claim–Evidence–Experiment–Cost–Falsification Mapping

| 拟议 claim | 当前证据 | 必需实验/对照 | 最低成本 | falsification / 降级 |
|---|---|---|---|---|
| UrbanEV 的 occupied count 是 capacity-bounded、zero-heavy/overdispersed 的跨站概率任务 | 官方字段说明 + [User Local Audit]；overdispersion 尚未在当前环境重算 | raw support、mean–variance、zero rate、capacity strata audit | 数据 audit，无训练 | support 不成立则弃用 BB family |
| DeepAR-NB 会分配 capacity 外质量并伤害 proper score | likelihood 机制推导；尚无本数据实验 | E3 vs E4，报告 $M_{\mathrm{out}}$ | 6 runs（2×3 seeds） | $M_{\mathrm{out}}$ 微小或 E4 无增益则 support claim 缩小 |
| fixed 168 h context 能改善真正 unseen sites | CNP/EV few-shot literature 给可行性，不是本任务证据 | K=0 vs K=168；E1/E3/E5/E6 | 最低证据矩阵 | few-shot≈zero-shot 则取消 adaptation claim |
| CAMP 的单调伪计数机制优于普通 bounded CNP | 结构可证；性能未知 | E5 vs E6 + E7/E8 + ESS/PIT | 9–15 runs | practical equivalent 或 calibration 变差则淘汰核心 |
| CAMP 输出更好的 calibrated PMF，而非只改点误差 | exact PMF；无 empirical result | CRPS/NLL/PIT/coverage-width/base+cal | 与 E6 同 runs | proper score 不改善或靠显著变宽则 claim 失败 |
| 多亚型需要 CBSAM mixture | EV archetype/多峰可能性；当前仅竞争解释 | 预冻结 multimodality 诊断、E6 vs E9、collapse check | 6 runs | 组件 collapse 或无稳定 gain 则维持 CAMP |
| 方法可迁移到另一数据集 | 目前无可比外部数据合同 | 完整 external contract 与 untouched-site replication | **超出核心预算** | 未完成前不得声称跨城市泛化 |
| 方法帮助 Niger 儿童或改善真实生活结果 | 无因果链、部署或 field evidence | 独立的人道项目设计、伦理/安全/效果评估 | **不属于本研究** | 本报告不得提出该 claim |

成本数字是 run 数规划，不是 GPU 小时；真实 wall time、显存与能源消耗须在 smoke test 后测量并标为 [Experiment Pending]。严肃程度不改变证据阈值：本研究的责任是给出可执行且可推翻的路线，不能把善意目标转写成未经验证的科学结果。

### 7.2 Risk, Trigger, Mitigation, and Candidate-Switch Path

| 风险 | 监测 trigger | 预防/缓解 | switch path |
|---|---|---|---|
| dataset version drift | hash/行数与 v23 manifest 不符 | immutable raw、显式 version，不跟随 latest | 停止；为新版本开新 study id |
| capacity/status 语义错误 | $Y>C$、状态和冲突、capacity jumps | data dictionary + row reconciliation | 重定义 target；所有旧 predictions 作废 |
| missing≠zero | 缺失段与低负荷混淆、排名集中于低 coverage | mask、coverage strata、no zero fill | 转 missingness-aware protocol |
| site leakage | 重复坐标跨 split、target profile 用于 grouping | coordinate group hash、source-only transforms | split 作废并重新冻结 |
| realized-weather leakage | future realized weather 进入 query | weather-off primary；只用 forecast-available archive | 移除 weather；不以缺字段补未来真值 |
| temporal leakage | context 越过 origin、rolling features center 对齐 | event-level max-source-time assertion | 作废受影响 run |
| baseline under-tuning | DeepAR 不收敛或预算不对等 | baseline-first recipe、同 early stop、公开 logs | 修复后重跑全部模型；不能只重跑 baseline |
| CAMP overconfidence | ESS 随 K 爆炸、coverage 降、PIT U-shape | $u_{\max}$、base/+cal、conflict diagnostic | E5 或 E1 |
| Beta-Binomial unimodality不足 | residual modes、mixture validation gain | C2 预注册、J=2 起步 | CBSAM |
| mixture collapse | 单组件质量趋近 1、seed 不稳定 | 最小 J、component audit | CAMP |
| test-set overfitting | 看 test 后改 feature/config | one-pass manifest、sealed prediction store | 不允许 switch；新数据才可复验 |
| external validity过度外推 | 只在深圳/同采集系统有效 | 明写 domain boundary | 外部数据独立复验 |
| compute failure | OOM/timeout | 预写 batch reduction/retry | 最低证据集或停止 |
| paper novelty被最近工作覆盖 | 新近工作实现同一机制/协议 | submission 前更新检索与 claim table | 重定位为 benchmark/negative result |

candidate path 是预先有向的，而非结果后搜索：

~~~text
raw/data contract pass
  -> bounded-support failure present?
       no  -> CAMP support rationale weak; prefer simple/ordinary models
       yes -> E4 solves it?
                yes -> truncated DeepAR-NB
                no  -> E5 vs E6
                         E6 wins with calibration -> CAMP
                         equivalent -> ordinary BB-CNP
                         multimodal residual + E9 wins -> CBSAM
                         all equivalent to E1 -> EB-SBB
~~~

这条路径只在 validation 阶段用于冻结首选；blind test 之后只能报告预注册比较，不能再“切换并重新宣称主方法”。

### 7.3 Contributions That Can Be Written Honestly and Claims Not Yet Allowed

**当前 desk-research 阶段可写的贡献：**

1. 一个针对 UrbanEVDataset 的、site-disjoint、时间合法、同时覆盖 zero-shot 与固定窗 few-shot 的概率预测合同；
2. 将 capacity-bounded count support、unseen-site context adaptation 与 calibration 分解为可隔离的机制和 controls；
3. CAMP-NP 的完整候选规范：合法 PMF、单调 pseudo-count 结构、空 context 退化、复杂度和淘汰条件；
4. CBSAM 的预声明 second-candidate switch path，防止失败后任意发明替代法；
5. 从 raw hashes 到 prediction manifest、paired hierarchical bootstrap 的可执行工程路线；
6. 一份明确记录 full-text/access 限制、检索入口、rejected candidates 与 evidence ceiling 的审计报告。

**只有通过实验后才可写的贡献：**

- CAMP 在 blind unseen sites 上显著且实质性改善 CRPS/NLL；
- few-shot context 比 zero-shot 有增益；
- 单调机制而非 support、参数量、calibration 或 metadata 解释增益；
- CBSAM 揭示可重复的潜在站点亚型；
- 任意跨数据集、跨城市、在线 drift 或 downstream operational benefit。

**当前明确禁止的表述：**

- “state of the art”“首次”“证明更准确/更校准”“可泛化到所有城市”；
- 把点预测结果说成完整概率预测；
- 把 post-hoc calibration 的增益归入 base architecture；
- 把 validation/test 的 target-site full history、hard cluster 或 realized future weather称为 few-shot；
- 把 [User Local Audit] 写成当前环境已复现；
- 把 [Full Text Unverified] 论文的细节当作已核实；
- 把尚未核实的 JCR/CAS 分区、期刊格式或投稿窗口写成事实；
- 把本研究与儿童生存、援助成效或任何人道结果建立未经评估的因果声称。

推荐论文边界是“在单一公开深圳数据集上的、capacity-bounded unseen-site probabilistic occupancy forecasting”；不是能源调度、充电到达事件预测、跨城市迁移、在线学习或人道干预评估。

### 7.4 Desk-Research, Data, Experiment, and Submission Readiness

| 层级 | verdict | 已具备 | 缺口 / 下一 gate |
|---|---|---|---|
| Desk research | **Ready for protocol freeze** | 数据版本、任务、baseline、2 个候选、反证 controls、检索/证据 ledger | 投稿前仍须一次 time-bounded update search |
| Data | **Conditionally Ready** | 官方 v23 可访问；[User Local Audit] 有 hashes、规模与 missing 摘要 | 当前环境未持有/重跑 raw；CRS、status/capacity reconciliation、weather legality 待验 |
| Engineering | **Specified, Not Executed** | manifests、pseudocode、config、retry/reduction | parser、unit tests、container 与 timing 尚未执行 |
| Experiment | **Not Ready to Claim Results** | 31-run 推荐矩阵与 19-run 最低矩阵 | 所有 scores、plots、effect sizes 均 [Experiment Pending] |
| Submission | **Not Ready** | honest paper boundary 与 falsification plan | empirical evidence、external robustness（若声称）、target venue style/JCR/CAS [To Verify] |

**Admission gate verdict：Conditional Pass。** CAMP-NP 值得进入 source/validation pilot，因为问题定义、支持机制、普通 BB-CNP control、第二候选和淘汰条件均已明确；它尚未获得“最终方法”或“可投稿贡献”的地位。最关键的第一个实验不是大规模 tuning，而是 raw support audit 加 E3/E4/E5/E6 的小型 validation gate。最小证据与推荐预算之间的差距必须在开跑前选择：若无法完成至少 17 个核心 runs，就应缩小论文 claim，而不是减少 seeds、controls 或 blind sites。

**Referee-style desk verdict：major evidence required, method pilot justified.** 最有价值的可证伪点是 exact capacity support 与 context pseudo-count；最危险的替代解释是 truncated likelihood、ordinary bounded CNP、missingness 与 station archetypes。只有在这些 controls 下仍取得 site-macro proper-score 与 calibration gain，CAMP 才能被保留。

## Appendix A: Batch Retrieval Log and Evidence Ledger

### A.1 Batch-Level Retrieval Log

检索日期均为 2026-08-30。检索工具未返回数据库总命中量，故“总命中”统一标为 **not exposed**；表中“可见/全文”是本次界面实际查看的最小数量，不伪造 PRISMA 总数。去重键优先 DOI，其次 arXiv id/title。检索遵守 b.md 的 C/F/E 逻辑：C 为候选池，F 为全文或可验证正文，E 为最终证据集；被拒绝工作仍保留入口与原因。

| Batch | 数据库/入口 | 实际 query 或入口 | 总命中 | 可见筛选 | 全文/正文可核 | 去重新增 C/F/E | 主要排除原因 |
|---|---|---|---|---:|---:|---:|---|
| R01 | Dryad / Nature / GitHub | UrbanEVDataset DOI、paper title、official repository | not exposed | 4 official pages | 4 | 1/1/1 + official records | version pages 合并；非论文记录不计 C |
| R02 | Crossref / publisher | DeepAR probabilistic forecasting autoregressive recurrent networks | not exposed | 5 | 4 | 1/1/1 | 非原始/二手说明 |
| R03 | SpringerLink | electric vehicle probabilistic charging demand forecast | not exposed | 4 | 2 | 1/1/1 | 与占用快照/跨站不匹配 |
| R04 | arXiv | electric vehicle charging demand probabilistic forecast diffusion | not exposed | 5 | 4 | 3/3/1 | 点预测或非 station-level |
| R05 | IEEE Xplore / Crossref | cold-start electric vehicle charging station forecasting | not exposed | 3 | 0 publisher full text | 2/0/0 | 正文不可访问 |
| R06 | arXiv / ACM | unseen charging station few-shot site archetype forecast | not exposed | 4 | 3 | 2/1/1 | 日 kWh 点预测而非 hourly PMF |
| R07 | arXiv / institutional PDF | global EV charging demand forecasting unseen locations N-HiTS | not exposed | 3 | 3 | 1/1/1 | 方法证据纳入，任务非相同 |
| R08 | PMLR / AAAI | conditional neural process evidential neural process regression | not exposed | 5 | 4 | 2/2/2 | 与本任务无直接数据证据 |
| R09 | Springer / Crossref | beta-binomial bounded count time series overdispersion | not exposed | 4 | 3 | 1/1/0 | 通用统计机制，不是 EV |
| R10 | OpenReview | meta-learning contextual time series forecasting neural processes | not exposed | 2 | 0 | 1/0/0 | robots/access 阻断 |
| R11 | NeurIPS / PMLR / Project Euclid | adaptive conformal inference time series exchangeability | not exposed | 5 | 5 | 3/1/0 | 只保留 calibration/online 边界 |
| R12 | arXiv, recent-date check | 2025 2026 EV charging few-shot online probabilistic forecast | not exposed | 6 | 4 | 3/0/0 | point/online regime 或非核心全文 |

停止记录：达到预先允许的 12-batch 上限。R11–R12 仍改变了 calibration/online 竞争边界，因此严格的“两轮无新增”饱和条件**没有**满足；只是没有改变 C1/C2 排序，也没有发现同一“capacity-anchored monotone pseudo-count + fixed unseen-site PMF”组合。故 novelty coverage 保持 conditional，投稿前须定时更新检索。未访问付费正文不以摘要补造细节。

### A.2 Evidence Ledger

| Evidence id | 来源 | 支持的陈述 | Evidence distance | Source type | Verification state | 限制 |
|---|---|---|---|---|---|---|
| D1 | Dryad v23 record/files | release、license、version history、official files | [Direct] | [Official Material] | [Cross-Checked] | 当前环境未重下 raw |
| D2 | UrbanEV Scientific Data paper | dataset construction、field meanings、spatiotemporal scope | [Direct] | [Peer-Reviewed Paper] | [Cross-Checked] | paper 不验证本报告 split |
| D3 | UrbanEV official GitHub | processing/code availability、license | [Direct] | [Official Material] | [Cross-Checked] | reproduction 尚未执行 |
| U1 | dataset.md 固定审计 | hashes、1,682 stations、行数、missingness、capacity/TAZ 摘要 | [Direct] | [User Local Audit] | [Single-Source] | 仅参考；非当前重跑 |
| L1 | DeepAR paper + GluonTS docs | autoregressive probabilistic baseline、NB implementation route | [Indirect] | [Peer-Reviewed Paper]/[Official Material] | [Cross-Checked] | NB support failure是推导，效果待试 |
| L2 | Ostermann & Haug | EV charging probabilistic forecasting precedent | [Indirect] | [Peer-Reviewed Paper] | [Single-Source] | 任务/数据不同 |
| L3 | DiffPLF | diffusion-based EV charging-load probabilistic forecast | [Indirect] | [Peer-Reviewed Paper]/[Author Preprint] | [Cross-Checked] | load，不是 unseen occupancy count |
| L4 | van Etten et al. | global/unseen-location EV forecasting precedent | [Indirect] | [Author Preprint/Repository] | [Single-Source] | point-focused、任务不同 |
| L5 | Nikhal et al. | few-shot EV site archetype precedent | [Indirect] | [Author Preprint/Repository] | [Single-Source] | daily kWh、hard profile information |
| L6 | Garnelo et al. | CNP context/query framework | [Indirect] | [Peer-Reviewed Paper] | [Cross-Checked] | continuous regression、无 capacity support |
| L7 | Pandey et al. | evidential NP precedent | [Indirect] | [Peer-Reviewed Paper] | [Cross-Checked] | uncertainty family与本方法不同 |
| L8 | Chen et al. | beta-binomial bounded count time-series mechanism | [Indirect] | [Peer-Reviewed Paper] | [Cross-Checked] | 非 EV、非 meta-learning |
| L9 | Gibbs–Candès / Xu–Xie / Barber et al. | conformal under shift/time-series 边界 | [Indirect] | [Peer-Reviewed Paper] | [Cross-Checked] | 不自动给 full PMF |
| L10 | recent EV preprints | online/queueing/contextual竞争边界 | [Indirect] | [Author Preprint/Repository] | [Single-Source] | 可能更新，部分尚未同行评审 |
| A1 | OpenReview contextual TS NP | 潜在最近方法 | [Indirect] | [Secondary Lead] | [To Verify] | 页面阻断，不能核实细节 |
| A2 | IEEE cold-start EV paper | cold-start 先例 | [Indirect] | [Peer-Reviewed Paper] | [To Verify] | 仅 metadata/abstract 可见 |
| G1 | ByteDance deer-flow review skill | 检索短 query、结构抽取、综合原则 | [Indirect] | [Official Material] | [Single-Source] | 未安装/执行，见 A.3；非科学证据 |
| P1 | CAMP/CBSAM/实验结果 | 所有效果与 calibration claim | [Research Hypothesis] | [Project Analysis] | [To Verify] | 当前无任何 measured result |

### A.3 GitHub Skill Retrieval and Use Record

- repository：ByteDance/deer-flow；
- skill：<code>systematic-literature-review/SKILL.md</code>；
- inspected HEAD：<code>8eda71fd978b4dc4b26843fe995407f09f4aa87e</code>；
- repository license：MIT；
- access date：2026-08-30；
- 采用的原则：短关键词分批检索、相关性筛选、结构化抽取、跨来源综合与显式局限；
- 未采用/未执行：该 skill 的 arXiv-focused/subagent workflow。原因是本任务必须核验 Dryad、publisher、PMLR、OpenReview、GitHub 等多类来源，且当前执行由单一主 agent 完成；安装或执行它不会增加原始证据；
- 安全说明：只检查公开文本，没有运行第三方脚本、没有授予凭据、没有修改环境。

### A.4 Traceability and Count Interpretation

本检索不是系统综述的穷尽性 PRISMA review。因为多个入口不公开稳定总命中数，报告只承诺“批次、query、可见筛选和全文可核数量可追踪”。同一论文的 arXiv、DOI、publisher 与 repository 只算一项 intellectual work；dataset paper、dataset record 和 code repository 在证据 ledger 中分别保留，因为它们证明的事情不同。任何后续更新必须追加 batch，而不能改写上述计数。

12 个 batches 的去重新增恰为 $|C|=21$、$|F|=12$、$|E|=8$；C/F/E 是嵌套 paper sets，不含 official dataset pages、software docs 或 skill。界面未暴露的 initial count 仍为 not exposed，不能由可见数反推。

### A.5 Experimentally Auditable Paper Matrices

以下只对 $E$ 的 8 篇做结构编码，样本是 purposive nearest-work set，不代表领域频率。central/companion count 是本报告按数学操作与必要性编码，不是作者自称“创新点”数量；作者 contribution-bullet count 若不能稳定抽取则记 NR。

**Paper × innovation-organization dimensions**

| Paper | Unified problem/core claim [Literature Report] | Organization | Central method (count) | Necessary companion (count) | Author bullets | Attribution/control pattern | Common risk/applicability here |
|---|---|---|---:|---:|---:|---|---|
| UrbanEV | open urban EV demand data + benchmark | problem-reformulation/data protocol | none (0) | processing/benchmark (1) | NR | processed/raw and benchmark comparisons | split/missing handling may not support unseen-site PMF |
| DeepAR | one global autoregressive probabilistic model across related series | single mechanism | RNN distributional decoder (1) | count likelihood/scaling interface (1) | NR | multi-dataset baseline comparisons | NB support and episode fairness need controls |
| Ostermann–Haug | probabilistic EV load forecasts across aggregation levels | central mechanism + experimental analysis | quantile/distributional forecast (1) | aggregation study (1) | NR | pinball/interval and level comparison | continuous load/seen aggregates differ |
| DiffPLF | conditional diffusion generates EV load scenarios | unified framework | conditional diffusion (1) | conditioning interface (1) | NR | deterministic/probabilistic alternatives and ablations | high compute; continuous aggregate load |
| van Etten et al. | global model forecasts demand at many/unseen locations | application/problem reformulation | global N-HiTS use (1) | global-vs-local protocol (1) | NR | local/global point baselines | point daily kWh and long history |
| Nikhal et al. | site archetypes improve few-shot new-site forecasts | central + necessary companion | archetype expert routing (1) | hard clustering/profile construction (1) | NR | global vs archetype experts | hard profile may exceed legal 168 h; proprietary core data |
| CNP | context set conditions a predictive process | single mechanism | permutation-invariant context/query map (1) | none (0) | NR | GP/NP-style task comparisons and context-size studies | continuous generic regression |
| ECNP | evidential CNP represents predictive uncertainty | central mechanism | evidential distribution head (1) | CNP backbone (1) | NR | CNP/evidential controls and uncertainty analysis | not bounded-count or deployment-specific |

**Descriptive organization summary**

| Organization form | Sample n/N | Central-method-count distribution | Necessary-companion-count distribution | Contribution-bullet distribution | Common attribution | Common reviewer risk | Applicability here |
|---|---:|---|---|---|---|---|---|
| single mechanism | 2/8 | both 1 | 0: one; 1: one | NR for both | backbone/head or context-size controls | generic mechanism may not be task-specific | supports one deep CAMP core |
| central + necessary companion | 3/8 | all 1 | all 1 | NR | component ablation + matched alternative | companion may carry the gain | CAMP + legal protocol only if independently necessary |
| unified framework | 1/8 | 1 | 1 | NR | end-to-end alternatives/ablation | compute and module attribution | argues against graph+diffusion stacking here |
| problem/data reformulation | 2/8 | 0: one; 1: one | both 1 | NR | protocol/global-vs-local comparisons | protocol novelty can be overstated | unseen-site protocol is conditional second contribution |

Because author bullet counts were not extracted consistently, no frequency is fabricated. The table supports only a qualitative design choice: one attributable central mechanism plus one necessary protocol companion is defensible; it does not establish an acceptance formula or required innovation count.

**Paper × external-method matrix**

| Paper | External-method/comparator role | Fairness evidence visible in current extraction | Directly comparable to this project? | Control imported here |
|---|---|---|---|---|
| UrbanEV | author benchmark models on processed zone/station tasks | same dataset family, author-defined processing | no: target/split/output differ | author-protocol sensitivity |
| DeepAR | classical and neural probabilistic forecasting baselines | original paper multi-dataset evaluation; official implementation | no: no UrbanEV split | faithful GluonTS DeepAR-NB |
| Ostermann–Haug | EV probabilistic/quantile alternatives across aggregation | common task within paper | no: continuous load and seen groups | pinball/interval paradigm, not scores |
| DiffPLF | deterministic and probabilistic load-forecast alternatives | within-paper retraining/ablation reported | no: aggregate continuous load | only scenario control if joint paths needed |
| van Etten et al. | local versus global point models | shared evaluation within author study | no: daily kWh/history/data differ | global N-HiTS point control |
| Nikhal et al. | global model versus hard archetype experts | shared author protocol | no: proprietary daily data/point target | source-only archetype alternative |
| CNP | GP/NP-family context-regression comparators | common benchmark tasks | no: generic continuous regression | ordinary BB-CNP |
| ECNP | CNP and evidential/uncertainty comparators | common benchmark tasks | no: output family/task differ | evidential-head novelty adversary |

No reported score in this matrix is numerically pooled or compared with planned UrbanEV scores.

**Paper × experimental-dimension matrix**

| Paper | Object/frequency/horizon/context | Cross-site regime and split | Probabilistic representation | Proper/calibration/sharpness/point evidence | Seeds/resources/reproduction |
|---|---|---|---|---|---|
| UrbanEV | regional/station EV demand; hourly author benchmark | time CV; not the frozen strict held-out-site task | point outputs in reported benchmark | RMSE/MAPE/RAE/MAE [Literature Report] | official data/code; this split not reproduced |
| DeepAR | generic related time series; dataset-specific windows | global train/evaluation series protocols | parametric predictive distributions, NB for counts | likelihood/probabilistic metrics by original tasks | official GluonTS route; exact UrbanEV cost TBD |
| Ostermann–Haug | EV load at multiple aggregation levels | seen aggregation groups/time evaluation | quantiles/intervals | pinball and interval evaluation [Literature Report] | full-text method visible; UrbanEV reproduction absent |
| DiffPLF | aggregate EV charging load scenarios | time split on source dataset; not unseen occupancy sites | conditional diffusion samples | probabilistic/scenario and point evaluation within paper | author preprint + journal identifier; high relative inference cost |
| van Etten et al. | daily kWh; long histories; unseen locations | global/unseen-location evaluation | point forecast | point errors only for present purpose | author PDF; code/resource parity TBD |
| Nikhal et al. | daily energy; 28 d context; 7 d horizon | new-site few-shot with hard archetypes | point forecast | point metrics; no full PMF evidence | workshop/preprint; proprietary main data |
| CNP | synthetic/standard regression tasks; variable context | task/context holdout, not EV sites | typically Gaussian conditional marginal | likelihood/uncertainty and prediction diagnostics | official PMLR full text; general method |
| ECNP | benchmark regression with context sets | context/task evaluation, not EV sites | evidential predictive distribution | uncertainty/calibration-style analysis in paper | AAAI full text; general method |

Seed counts, hardware and run times are not normalized across these papers because current protocols and outputs are incomparable. “NR/TBD” is retained rather than converting absent reporting into zero. This matrix justifies experimental dimensions—proper score, calibration plus width, point secondary, held-out sites, context legality and matched controls—without treating their frequency as population evidence.

## Appendix B: References, Official Dataset Links, and Access Status

### B.1 Official UrbanEV Dataset Family

| Item | Official link | Frozen identity/status | 本报告使用方式 |
|---|---|---|---|
| Dryad dataset | [doi:10.5061/dryad.np5hqc04z](https://doi.org/10.5061/dryad.np5hqc04z) | v23/internal 23；published 2026-02-04；CC0；[Official Record] | primary raw source |
| Dryad README | 由 Dryad v23 files 页面下载 | 14,914 bytes；SHA-256 <code>ec2aba0a893d2b310b12ddfd006679ae92a85149aa7a767e1734e82d175ac177</code> [User Local Audit] | schema/official note |
| Dryad ZIP | 由 Dryad v23 files 页面下载 | 320,234,247 bytes；SHA-256 <code>a041322ed75eab8c49095fe5d0501b05f8d56dae16209586f8e4d7333ad1051f</code> [User Local Audit] | frozen payload |
| Data paper | [Scientific Data, doi:10.1038/s41597-025-04874-4](https://doi.org/10.1038/s41597-025-04874-4) | peer reviewed；open article [Full Text Accessed] | construction/schema/context |
| Official code/data supplement | [IntelligentSystemsLab/UrbanEV](https://github.com/IntelligentSystemsLab/UrbanEV) | CC0；frozen commit <code>44f2aa0c8d89f192bce00bafb0def74a21b39c68</code> [Official Repository] | auxiliary artifacts only |

已核对的 Dryad version history：

| Publication date | version/page size | 解释 |
|---|---:|---|
| 2025-03-17 | 101.83 MB | 初始公开版本 |
| 2025-04-25 | 320.23 MB | 数据扩充/修订版本 |
| 2025-09-24 | 350.97 MB | 后续修订 |
| 2026-02-04 | 320.25 MB | 本报告冻结的 v23 |

version history 只用于说明为何必须冻结 payload；不得混合四版文件。官方网页显示的十进制 MB 与本地 byte count 的小差异不是 hash 差异。access date 均为 2026-08-30；本轮没有重新下载 320 MB payload，故完整性仍以 [User Local Audit] 标示。

### B.2 Core References

以下按本报告的证据用途排列，而非按结果好坏排序。链接直接指向 DOI、官方 proceedings、author preprint 或 official docs；受限条目明确标记。

1. Li et al. (2025). *UrbanEV: an open benchmark dataset for urban electric vehicle charging demand prediction*. Scientific Data. [doi:10.1038/s41597-025-04874-4](https://doi.org/10.1038/s41597-025-04874-4). **[Peer Reviewed; Full Text Accessed]**
2. Salinas, Flunkert, Gasthaus, and Januschowski (2020). *DeepAR: Probabilistic forecasting with autoregressive recurrent networks*. International Journal of Forecasting, 36(3), 1181–1191. [doi:10.1016/j.ijforecast.2019.07.001](https://doi.org/10.1016/j.ijforecast.2019.07.001). **[Peer Reviewed; Full Text Accessed]**
3. GluonTS maintainers (2026). *DeepAR PyTorch module documentation*. [Official documentation](https://ts.gluon.ai/stable/api/gluonts/gluonts.torch.model.deepar.module.html); [GluonTS 0.17.0 release](https://pypi.org/project/gluonts/0.17.0/). **[Official Docs]**
4. Ostermann, A., and Haug, T. (2024). *Probabilistic forecast of electric vehicle charging demand: analysis of different aggregation levels and energy procurement*. Energy Informatics, 7, Article 13. [doi:10.1186/s42162-024-00319-1](https://doi.org/10.1186/s42162-024-00319-1). **[Peer Reviewed; Full Text Accessed]**
5. Li, S., Xiong, H., and Chen, Y. (2024). *DiffPLF: A conditional diffusion model for probabilistic forecasting of EV charging load*. Electric Power Systems Research, 235, 110723. [arXiv:2402.13548](https://arxiv.org/abs/2402.13548); [doi:10.1016/j.epsr.2024.110723](https://doi.org/10.1016/j.epsr.2024.110723). **[Peer Reviewed/Author Preprint; Full Text Accessed]**
6. Matrone, S., Heydarian Ardakani, A., Ogliari, E., Shirazi, E., and Leva, S. (2025). *Probabilistic Forecast of EV Charging Demand using Quantile Regression and LSTM with Attention Mechanism*. Proceedings of ACM e-Energy 2025, 1005–1007. [doi:10.1145/3679240.3734687](https://doi.org/10.1145/3679240.3734687). **[Peer Reviewed; Full Text Accessed]**
7. van Etten, T., Degeler, V., and Luo, D. (2024). *Large-Scale Forecasting of Electric Vehicle Charging Demand Using Global Time Series Modeling*. VEHITS 2024 author manuscript. [University of Amsterdam full text](https://pure.uva.nl/ws/files/171150045/VEHITS2024_Large_Scale_Forecasting_Emobility.pdf). **[Author Full Text Accessed]**
8. Nikhal, K., Ackerknecht, L., Riggan, B. S., and Stahlfeld, P. (2025). *Discovering EV Charging Site Archetypes Through Few Shot Forecasting: The First U.S.-Wide Study*. Tackling Climate Change with Machine Learning Workshop at NeurIPS 2025. [arXiv:2510.26910v2](https://arxiv.org/abs/2510.26910v2). **[Author Preprint/Workshop; Full Text Accessed; Single-Source]**
9. Garnelo et al. (2018). *Conditional Neural Processes*. Proceedings of ICML, PMLR 80. [Official PMLR page](https://proceedings.mlr.press/v80/garnelo18a.html). **[Peer Reviewed; Full Text Accessed]**
10. Pandey, D. S., and Yu, Q. (2023). *Evidential Conditional Neural Processes*. Proceedings of the AAAI Conference on Artificial Intelligence, 37(8), 9389–9397. [doi:10.1609/AAAI.V37I8.26125](https://doi.org/10.1609/AAAI.V37I8.26125). **[Peer Reviewed; Full Text Accessed]**
11. Chen, H., Li, Q., and Zhu, F. (2023). *A covariate-driven beta-binomial integer-valued GARCH model for bounded counts with an application*. Metrika, 86, 805–826. [doi:10.1007/s00184-023-00894-5](https://doi.org/10.1007/s00184-023-00894-5). **[Peer Reviewed; Article Text Accessed]**
12. Gibbs and Candès (2021). *Adaptive Conformal Inference Under Distribution Shift*. NeurIPS 34. [Official proceedings page](https://proceedings.neurips.cc/paper/2021/hash/0d441de75945e5acbc865406fc9a2559-Abstract.html). **[Peer Reviewed; Full Text Accessed]**
13. Xu and Xie (2021). *Conformal prediction interval for dynamic time-series*. Proceedings of ICML, PMLR 139. [Official PMLR page](https://proceedings.mlr.press/v139/xu21h.html). **[Peer Reviewed; Full Text Accessed]**
14. Barber, Candès, Ramdas, and Tibshirani (2023). *Conformal prediction beyond exchangeability*. Annals of Statistics. [doi:10.1214/23-AOS2276](https://doi.org/10.1214/23-AOS2276). **[Peer Reviewed; Full Text Accessed]**
15. Oreshkin, Carpov, Chapados, and Bengio (2020). *Meta-learning framework with applications to zero-shot time-series forecasting*. [arXiv:2002.02887](https://arxiv.org/abs/2002.02887). **[Author Preprint; Full Text Accessed]**
16. Kuang, H., Deng, K., You, L., and Li, J. (2024). *Citywide Electric Vehicle Charging Demand Prediction Approach Considering Urban Region and Dynamic Influences*. arXiv. [arXiv:2410.18766](https://arxiv.org/abs/2410.18766). **[Author Preprint; Full Text Accessed]**
17. Bouaachra, K., Amara-Ouali, Y., Goude, Y., and Lachieze-Rey, R. (2026). *Spatio-temporal modelling of electric vehicle charging demand*. arXiv. [arXiv:2604.19841](https://arxiv.org/abs/2604.19841). **[Author Preprint; Full Text Accessed]**
18. Li, C., Liu, Q., Xu, Y., and Liang, Y. (2026). *A Behavior-Guided Online Probabilistic Forecasting Method for Electric vehicle Charging Loads*. arXiv; submitted to IEEE. [arXiv:2608.24441](https://arxiv.org/abs/2608.24441). **[Author Preprint; submitted 2026-08-25; Full Text Accessed; Single-Source]**
19. Zhang, X., Chan, K. W., Li, H., Wang, H., Qiu, J., and Wang, G. (2021). *Deep-Learning-Based Probabilistic Forecasting of Electric Vehicle Charging Load With a Novel Queuing Model*. IEEE Transactions on Cybernetics, 51(6), 3157–3170. [doi:10.1109/TCYB.2020.2975134](https://doi.org/10.1109/TCYB.2020.2975134); [PolyU accepted manuscript](https://ira.lib.polyu.edu.hk/handle/10397/93379). **[Peer Reviewed; Author Accepted Manuscript Accessed]**
20. Cao, T., Xu, Y., Liu, G., Tao, S., Tang, W., and Sun, H. (2024). *Feature-enhanced deep learning method for electric vehicle charging demand probabilistic forecasting of charging station*. Applied Energy, Article 123751. [doi:10.1016/j.apenergy.2024.123751](https://doi.org/10.1016/j.apenergy.2024.123751). **[Peer Reviewed; Full Text Unverified; metadata/abstract only]**
21. Forootani, A., Rastegar, M., and Zareipour, H. (2024). *Transfer Learning-Based Framework Enhanced by Deep Generative Model for Cold-Start Forecasting of Residential EV Charging Behavior*. IEEE Transactions on Intelligent Vehicles, 9(1), 190–198. [IEEE document 10301576](https://ieeexplore.ieee.org/document/10301576). **[Peer Reviewed; Full Text Unverified; metadata/abstract only]**
22. Authors and venue **[To Verify]** (2026). *Meta-Learning Contextual Time Series Forecasting with Neural Processes*. [OpenReview forum jDQyU5j8pn](https://openreview.net/forum?id=jDQyU5j8pn). **[Full Text Unverified; forum blocked by verification]**
23. ByteDance (2026 snapshot). *systematic-literature-review skill*, deer-flow. [GitHub source](https://github.com/bytedance/deer-flow/blob/main/skills/public/systematic-literature-review/SKILL.md). **[Public Repository Text Inspected; not scientific evidence]**

### B.3 Access and Citation Rules

- **[Full Text Accessed]** 表示本轮或固定材料中可以核对方法/限制，不表示复现实验；
- **[Full Text Unverified]** 只允许支持“该工作可能相关/需补查”，不能支持细节、优越性或 novelty 排除；
- arXiv/author manuscript 与 peer-reviewed version 并存时，以 DOI 为 bibliographic record、以 author copy 补正文访问；
- 2026 preprints 可能在投稿前更新，必须按 title/arXiv version 再核；
- target journal 未指定；JCR quartile、CAS 分区、APC、页数和模板均为 **[To Verify]**，本报告不猜测。

## Appendix C: Detailed Data-Processing, Split, and Leakage Checklist

### C.1 Acquisition and Raw Integrity

- [ ] 冻结 Dryad DOI、internal version 23、publication date 与 CC0；
- [ ] 校验 README SHA-256；
- [ ] 校验 ZIP SHA-256 与 byte size；
- [ ] 校验 repository commit/archive SHA-256；
- [ ] 把 raw 设为 immutable，只写 derived 目录；
- [ ] 对 ZIP 内每个文件记录 path、size、hash；
- [ ] 确认 1,682 个 station CSV 与 station_information id 一一对应；
- [ ] 记录 parser/environment/container hashes；
- [ ] 与 [User Local Audit] 对比，但不把“接近”替代为一致性解释；
- [ ] 所有差异写入 append-only audit log。

### C.2 Schema, Time, Target, Capacity, and Missingness

- [ ] timezone 明示为 Asia/Shanghai 的假设并保留 naive source value；
- [ ] 第一/末时间、5-min cadence、duplicate、parse failure 重算；
- [ ] busy、idle、fast/slow components 均检查 nonnegative integer；
- [ ] 对每个完整行验证 busy=fast_busy+slow_busy；
- [ ] 对每个完整行验证 idle=fast_idle+slow_idle；
- [ ] 对每个完整行验证 busy+idle=charge_count；
- [ ] 将 all-six-status-missing 与 busy=0 分开；
- [ ] 禁止 forward/backward/last-value/zero fill；
- [ ] capacity 以 station_information.charge_count 为主，不以 incomplete pile table 替代；
- [ ] capacity 若随时间变化则显式建 $C_{s,t}$，否则验证 static；
- [ ] HH:55 是 frozen snapshot 规则，不是 hour mean/sum；
- [ ] 任何 unavailable HH:55 只 mask，不因 outcome 删除 station/episode；
- [ ] duration/volume 标作 derived，不作为独立 sensor ground truth；
- [ ] price 缺 announcement/effective-time，不进入主 legal feature set。

### C.3 Station Identity, Coordinates, and Split

- [ ] 对精确相同坐标建立 coordinate_group_id；
- [ ] 同 group 的所有 station ids 只能在一个 split；
- [ ] 不用 target profile、全期 missingness、未来 utilization 做 grouping；
- [ ] deterministic hash function、salt 与排序写入 manifest；
- [ ] 60/20/20 group split 在任何 feature fit 前执行；
- [ ] source、validation、blind station 列表落盘并 hash；
- [ ] 报每 split 的 station/group/capacity/coverage分布；
- [ ] 如果共址但 operator/physical identity 不明，做 group-level sensitivity；
- [ ] CRS 冲突未解决前禁用距离图、TAZ spatial join 与 POI distance；
- [ ] station id 不作为可泛化 embedding 输入。

### C.4 Episode Construction

- [ ] primary few-shot context 恰为 origin 前 168 hourly slots；
- [ ] zero-shot 用同一 station/origin/horizon 且 context 为空；
- [ ] query horizons 仅为 +1/+3/+6 h；+12 h 属 sensitivity；
- [ ] context coverage 主层要求 $\ge0.8$，低于阈值另列，不静默删除；
- [ ] query missing 只使该 event unscored，不以 truth availability 重抽 episode；
- [ ] blind context starts 固定为 2023-01-01、01-15、02-01、02-15，forecast origin 为各 168 h 窗口结束后；
- [ ] 起点若与 dataset boundary/context不足冲突，规则在 test 前解决；
- [ ] 重叠 episode 保留 episode block id，不作 IID 样本；
- [ ] context 结束必须严格早于每个 query timestamp；
- [ ] 每个 event id 由 split/site/origin/horizon 共同 hash。

### C.5 Feature Availability and Transform Fitting

- [ ] calendar 由 forecast timestamp 确定，可合法使用；
- [ ] capacity 必须在 forecast origin 已知；
- [ ] mask 仅指示过去 context availability；
- [ ] rolling/lag feature 的 max source timestamp < origin；
- [ ] source-only normalization/imputation/bin edges 在 source-train fit；
- [ ] validation 只选 config，不 fit target distribution；
- [ ] blind target station 全期 mean/variance/profile 不可见；
- [ ] realized future weather 不进入 query；只有 forecast archive 才可用；
- [ ] POI snapshot 只在其 observation date 与 station join 合法时作辅助；
- [ ] 任何 metadata 缺失指示在 test 前冻结；
- [ ] feature manifest 记录 source file、event time、availability time、fit split；
- [ ] weather-off 是 primary，避免 auxiliary-data availability 决定主结论。

### C.6 Training, Calibration, and Test Isolation

- [ ] batch sampler 先均匀采 site，再采 episode；
- [ ] DeepAR 与自研法获得同 context 和 legal query features；
- [ ] early stopping、learning-rate trigger、OOM retry 只看 source/validation；
- [ ] seeds、max epochs、batch reduction 在 blind 前冻结；
- [ ] calibration 只在 validation sites 拟合并按 horizon 冻结；
- [ ] target few-shot context 只用于 forward inference，不做 gradient update；
- [ ] query labels 不用于任何 adaptation/calibration；
- [ ] blind evaluator 与 model training package 分离；
- [ ] blind token 只解封一次；
- [ ] test 后 code/config 变化自动使 study id 失效；
- [ ] failed run 只按预注册技术 retry，不改模型；
- [ ] checkpoint、input、prediction、evaluator hashes 可串联。

### C.7 PMF and Metric Assertions

- [ ] 每个 PMF 定义于整数 $0,\ldots,C_s$；
- [ ] 概率非负、finite、和在 tolerance 内为 1；
- [ ] truth 在 support 内；
- [ ] truncated NB 使用解析/稳定 log normalization；
- [ ] discrete CRPS 与 NLL 在同一 count measure；
- [ ] randomized PIT 的 uniform 由 event id + frozen seed 确定；
- [ ] 50/80/95% interval 构造规则一致；
- [ ] coverage 与 width/interval score 同时报告；
- [ ] site 内聚合后 site-macro；micro 只能 secondary；
- [ ] paired bootstrap 对所有模型共享 draw；
- [ ] seed 与 bootstrap uncertainty 分开；
- [ ] 预注册 strata 和 exploratory strata 分开；
- [ ] 无完整 PMF 的方法不进入 NLL/CRPS 表；
- [ ] tables/figures 从 long prediction artifacts 自动生成。

### C.8 Final Acceptance Record

只有 C.1–C.7 的适用项全部签署，data gate 才由 Conditional Pass 升为 Pass。任何 waiver 必须写明责任人、理由、影响的 claim 与 compensating control；“模型能跑”不是 waiver。最终 acceptance record 至少包含 data steward、model lead、evaluation lead 三个角色签名；在本报告的单 agent 阶段，这些是未来职责，不是假装已有多人签署。

## Appendix D: Second Candidate and Ranking-Reversal Evidence

### D.1 Candidate Definition: Capacity-Bounded Soft Archetype Mixture

CBSAM 是失败假设明确时的第二候选，不是为保险而附加的 ensemble。其唯一触发根因是：合法 168 h context 暗示一个新站属于多个潜在行为亚型，而单一 Beta-Binomial 对同一 query 的条件分布无法表达分离的内部模式或亚型相关尾部。

令 context encoder 的 mask-aware summary 为

$$
h_{\mathcal C}=\rho_\theta\!\left(
\left[
\sum_{i\in\mathcal C}m_i\phi_\theta(x_i,y_i/C_s)
+\phi_{\emptyset}\mathbf 1[|\mathcal C|=0],
\ \log(1+K_{\mathrm{obs}}),
\ \log(1+C_s)
\right]\right).
$$

where $m_i$ is the context-observation mask. Then $\pi_{qj}=\operatorname{softmax}_j(g_\theta(h_{\mathcal C},x_q,\log(1+C_s)))$. Each component $j=1,\ldots,J$ returns positive $a_{qj},b_{qj}$, and the exact predictive PMF is

$$
p(Y_q=k\mid\mathcal C,x_q,C_s)
=\sum_{j=1}^{J}\pi_{qj}
\,\binom{C_s}{k}
\,\frac{B(k+a_{qj},C_s-k+b_{qj})}{B(a_{qj},b_{qj})},
\quad k=0,\ldots,C_s.
$$

The mixture weights are query-dependent but can use only the legal context, calendar/capacity inputs known at origin, and source-learned parameters. The primary CBSAM form uses $J=2$; $J=4$ is a validation-only triggered sensitivity. Training minimizes the exact mixture NLL with log-sum-exp. Zero-shot uses $\phi_{\emptyset}$ and the source-learned archetype prior; few-shot is a frozen forward pass, not target-site gradient fitting.

The displayed summary uses a sum aggregator. A mean aggregator is not interchangeable because context count itself is evidence; if mean is tested, it must receive an explicit $K_{\mathrm{obs}}$ feature and be recorded as a separate variant. Components are exchangeable labels; interpretation never relies on “component 1=commuter” without post hoc external validation.

### D.2 Evidence for Keeping CBSAM Second

| Evidence | Supports | Does not establish |
|---|---|---|
| Nikhal et al. report gains from hard site archetype experts | unseen EV sites can be heterogeneous | their daily kWh clusters apply to UrbanEV hourly occupancy |
| UrbanEV has many stations/capacities/TAZs [User Local Audit] | heterogeneity is plausible | conditional multimodality |
| single Beta-Binomial has a restricted shape family | separated interior modes may be misspecified | such modes occur after legal conditioning |
| C2 exact mixture remains on $0..C_s$ | support is correct | better calibration or novelty |
| soft gate avoids all-period hard target profiles | information set can be legal | gate cannot overfit/scatter components |

CBSAM stays second because the current evidence is plausibility rather than an observed failure, mixture-of-experts is mature prior art, parameter/identifiability cost is higher, and a 168 h context may already identify a unimodal conditional distribution. It rises only on validation evidence, never because CAMP loses on blind test.

### D.3 Predeclared Multimodality and Collapse Diagnostics

CBSAM may rise above CAMP only if all following are true on validation sites:

1. capacity-normalized residual or conditional empirical histograms show separated modes within predeclared hour-of-week/capacity strata, not merely a zero spike plus overdispersion;
2. a posterior-predictive check rejects the single-BB shape in the same strata across at least two seeds;
3. CBSAM improves site-macro CRPS beyond frozen $\delta_{\mathrm{CRPS}}$, does not worsen NLL materially, and improves or preserves coverage at matched width;
4. gain remains after matching CAMP parameter count/compute and after the ordinary wide-single-BB control;
5. both components carry nontrivial mass across sites and seeds; the result is not one active component plus a dead component;
6. ranking is not explained entirely by February coverage or a small number of coordinate groups.

Collapse diagnostics saved per query/site:

- effective component count $\exp\{-\sum_j\pi_j\log\pi_j\}$;
- fraction with $\max_j\pi_j>0.99$;
- component-specific predictive means/dispersion and pairwise separation;
- component utilization by site/capacity/hour-of-week;
- seed-aligned distributional summaries without assigning semantic labels;
- log-likelihood decomposition and calibration by dominant component.

The primary run uses no entropy or load-balancing regularizer, because such a term can manufacture component use. If all seeds collapse, C2 is rejected rather than “fixed” on test data. A regularized mixture would require a new validation study id and a new external blind set.

### D.4 Controls, Cost, and Reversal Decision

| Control | Question |
|---|---|
| single CAMP / ordinary BB-CNP | is mixture necessary at all? |
| parameter-matched wider single model | is gain only parameter count? |
| hard source-only EB archetypes | does end-to-end soft gating add value? |
| shuffled legal context | is gate using target evidence or calendar priors only? |
| K=0 vs K=168 | do mixture weights adapt with context? |
| J=2 vs triggered J=4 | does extra modality help reproducibly? |
| base vs temperature-calibrated | is mixture gain just recalibration? |

CBSAM complexity is $O(BHJKD+B H J C_{\max})$ for context/query and exact PMF evaluation, with roughly $J$ output heads plus a gate. Recommended allocation is 3 seeds for J=2, with no J=4 runs unless the validation trigger fires. The ranking-reversal record must include trigger values, diagnostic plots, config hash and timestamp before blind-test access.

Final reversal rule:

$$
\text{Choose CBSAM on validation}
\iff
\text{multimodality trigger}
\land
\Delta\mathrm{CRPS}<-\delta_{\mathrm{CRPS}}
\land
\text{calibration noninferior}
\land
\text{no collapse}
\land
\text{matched-control gain}.
$$

If any conjunct fails, CAMP remains the candidate taken to the blind comparison; if CAMP itself fails its gate, choose the simplest surviving baseline rather than automatically promoting CBSAM.

## Appendix E: Agent/Role Task Ledger, Rejected Candidates, and Decision Log

### E.1 Execution and Role Ledger

This report is a **[Single-Agent Constrained Review]**. The roles below are sequential adversarial passes by one agent, not independent researchers, votes, replications, or blinded reviews. This disclosure matters because agreement between rows is not independent evidence.

| Role pass | Inputs inspected | Concrete output | Independence/status |
|---|---|---|---|
| data auditor | b.md, dataset.md, Dryad/Nature/GitHub records | version freeze, task-fit/data gate, raw checklist | same agent; desk-only |
| literature mapper | 12 retrieval batches, full/limited sources | C/F/E sets, nearest-work and access ledger | same agent; no bibliographic database export |
| baseline engineer | DeepAR paper/docs, task contract | DeepAR-NB recipe, strong baseline family | same agent; not executed |
| mechanism proposer | frozen problem ledger | C1–C4 candidate set | same agent |
| novelty adversary | ECNP/CNP, BB model, global/few-shot EV work | reconstructed C1, demoted novelty claims | same agent |
| falsification adversary | competing root causes and controls | E0–E10 matrix, elimination/switch rules | same agent |
| statistical reviewer | episode dependence, discrete PMF semantics | macro metrics, hierarchical bootstrap, freeze | same agent; no results |
| engineering reviewer | manifests, failures, compute limits | pseudocode, resource-reduction and fallback | same agent; not implemented |
| referee pass | all frozen sections | 17/20 internal rubric, Conditional Pass | self-review, not external peer review |

No sub-agent was spawned, no hidden candidate was omitted from a vote, and no claim of consensus is made. The future signed data/model/evaluation roles in Appendix C should be assigned to separate humans where feasible.

### E.2 Rejected or Demoted Candidate Ledger

| ID | Candidate/idea | Why initially plausible | Evidence/adversary finding | Final status | Reopen condition |
|---|---|---|---|---|---|
| C1-v0 | “evidential BB-CNP” | context uncertainty + bounded count | ECNP already claims evidential CNP; name/claim too broad | reconstructed as CAMP-NP | none; old claim remains retired |
| C2-v0 | hard archetype experts | station heterogeneity | Nikhal et al. directly cover hard few-shot experts | reconstructed as soft bounded mixture | legal multimodality trigger |
| C3 | adaptive conformal as contribution | drift harms coverage | ACI/conformal mature; needs labels; intervals not full PMF | demoted to control/online regime | calibration root cause with separate protocol |
| C4 | graph conditional diffusion | spatial spillover and scenarios | CRS conflict, leakage/cost, CityEVCP/DiffPLF/INLA proximity | rejected | CRS + source residual spatial signal + joint-scenario need |
| R1 | graph CAMP | could add nearby-station signal | violates minimal attribution; target-neighbor availability unclear | rejected | after C1 replicated, separate study |
| R2 | weather-conditioned CAMP | weather affects charging | only realized weather locally documented; forecast archive absent | removed from primary | legal forecast archive with availability timestamps |
| R3 | price-aware demand model | price may drive utilization | no announcement/effective-time semantics; potential simultaneity | rejected | causal timing/provenance established |
| R4 | author-processed imputed target | easier/high coverage | confounds telemetry missingness and outcome | rejected for primary | transparent sensitivity only |
| R5 | station-id embedding | boosts fit on source | unusable for unseen IDs and leaks identity shortcuts | rejected | never for primary unseen-site claim |
| R6 | test-site full-profile clustering | strong archetypes | violates fixed 168 h information set | rejected | never under current contract |
| R7 | online fine-tuning | adapts to drift | changes fixed few-shot regime and needs target labels | separate study | predeclared online deployment question |
| R8 | alternative dataset search | may simplify external validity | b.md fixed UrbanEVDataset; current data family is usable conditionally | not pursued | only after core result, for external robustness |

### E.3 Decision Log

| Decision id | Frozen decision | Basis | What would change it |
|---|---|---|---|
| D00 | exact research object is UrbanEV occupied-connector count PMF | official schema + b.md | raw schema contradiction |
| D01 | hourly reference is exact HH:55, no imputation | [User Local Audit] and deployment clarity | preregistered alternative aggregation study |
| D02 | primary is unseen-site few-shot; zero-shot secondary | contribution value + dataset length | context audit makes 168 h infeasible |
| D03 | 168 h context; 1/3/6 h horizons; 12 h sensitivity | fixed budget/task contract | only validation before blind freeze |
| D04 | exact-coordinate groups stay together in 60/20/20 split | 264 IDs in 125 duplicate-coordinate groups [User Local Audit] | improved physical-site identifier, applied before split |
| D05 | primary legal features are capacity/calendar/mask | availability and leakage audit | documented forecast-time auxiliary source |
| D06 | DeepAR-NB is primary baseline, not sole baseline | recognized probabilistic global model | reproduction failure after faithful effort |
| D07 | root-cause priority is bounded support/context evidence | target mechanics + literature gap | E3/E4/E5 smoke gate contradicts |
| D08 | retain C1/C2; demote C3; reject C4 | candidate/adversary ledger | predeclared validation triggers |
| D09 | reconstruct C1 as CAMP after ECNP counterevidence | novelty operation comparison | same mechanism found in accessible prior work |
| D10 | CAMP has exact BB PMF and nonnegative query weights | minimum attributable intervention | ordinary BB-CNP equivalence |
| D11 | CBSAM is second, J=2 primary | archetype evidence but weak task-specific proof | Appendix D conjunction |
| D12 | primary metric is site-macro discrete CRPS | complete PMF and site-level deployment | only before blind test, with written rationale |
| D13 | one blind pass and hierarchical site/episode bootstrap | dependence/leakage control | new independent test set |
| D14 | data gate Conditional; experiment/submission Not Ready | raw unavailable here; no runs | completed signed audits and formal results |

### E.4 Unresolved Dissent and Required Future Decisions

1. **Novelty dissent:** OpenReview contextual time-series NP full text is unavailable; CAMP may be a narrow recombination rather than a new method. Resolve before submission.
2. **Data dissent:** station IDs at exact coordinates may be separate operators rather than duplicate physical sites; grouping is conservative but may reduce independence/size. Resolve through metadata or retain group split.
3. **Target dissent:** HH:55 is transparent but may be noisier than hourly mean/maximum and answers a snapshot question. Keep primary; compare an aggregation sensitivity only if raw semantics support it.
4. **Missingness dissent:** February fragmentation may dominate results and is not missing at random. Main claims require coverage/month strata and no causal interpretation of missingness.
5. **Distribution dissent:** one-step marginal PMFs do not supply coherent joint paths. Do not use for multi-hour operational optimization without a separate joint model.
6. **Protocol-contribution dissent:** a careful split is necessary but may not count as a second scholarly contribution. Decide only after showing that conventional processing/splits materially change conclusions.
7. **Venue dissent:** no target journal is named; Q3/CAS labels and formatting remain [To Verify].
8. **Human-impact dissent:** no evidence links forecasting scores to an intervention in Niger. Any humanitarian project needs its own local needs assessment, partner governance, safety, logistics, ethics and outcome evaluation.

### E.5 Completion State

The desk deliverable has completed problem definition, fixed-dataset audit interpretation, literature mapping, candidate derivation, method specification, controls, engineering route, risks, references and decision traceability. It has deliberately **not** fabricated raw-data reproduction, model execution, performance values, venue eligibility, external peer consensus or humanitarian outcomes. The next irreversible action is not submission: it is the signed raw audit followed by the source/validation smoke gate in Section 6.2.
