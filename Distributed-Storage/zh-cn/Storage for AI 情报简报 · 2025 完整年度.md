# Storage for AI 情报简报 · 2025 完整年度

> **说明**：本报告为**回溯性年报**，撰写时间为 2026-07-29，晚于报告窗口约七个月。这带来一个方法学上的优势和一个偏置，两者都必须明说：
> **优势**——2025 年提出的技术已经可以用 2026 年的后续发展来检验，哪些成为了行业默认、哪些没有下文，本报告会明确标注。
> **偏置**——同期视角丢失。2025 年当时被认为重要但后来无声的工作，可能在本报告中被系统性低估；且部分 2025 年的一手链接已被更新版本覆盖。

---

## 1. 检索范围与存储工程师结论

**报告窗口**：2025-01-01 00:00 — 2025-12-31 23:59（Asia/Shanghai, UTC+8）。**完整年度**，无 YTD 标记。

**已覆盖并入选的渠道**：FAST '25（2/25–27, Santa Clara）、EuroSys '25 及其 CHEOPS '25 workshop（3/30–4/3, Rotterdam）、NSDI '25（4/28–30, Philadelphia）、ATC '25（7/7–9, Boston）、SOSP '25（10/13–16, Seoul）、SC'25 及 PDSW（11 月, St. Louis）、HPCA 2025、NVIDIA GTC 2025（3 月）、Google Cloud Next 2025（4 月）、AWS re:Invent 2025（12 月）及全年 S3 发布流、MLCommons MLPerf Storage v2.0（8 月）、DeepSeek Open Source Week（2 月末）、Mooncake / ByteCheckpoint / Dynamo / NIXL / LMCache 开源仓库时间线、UALink / Ultra Ethernet / CXL 标准发布、Glenn Lockwood 与 VAST 的技术长文。

**已搜索但产出有限的渠道**（见 §9）：Meta / OpenAI / Anthropic 的存储侧工程披露、CERN / ECMWF / EuroHPC 的 2025 年 AI I/O 论文、IO500 的 2025 两轮榜单细节。

### 给分布式存储工程师的 5 条结论（云存储 → AI 存储）

1. **2025 年是"KV cache 从引擎内部数据结构变成存储问题"被学术界正式承认的一年，但还没有专用硬件。** FAST '25 最佳论文给了 Mooncake——一个把 GPU 集群里"未被充分利用的 CPU、DRAM、SSD 和 NIC"聚合成分离式 KVCache 池的架构，标题直接叫 *Trading More Storage for Less Computation*。同一个 session 里 IMPRESS 讨论多层前缀 KV 存储。NVIDIA 在 GTC 2025 开源 Dynamo 与 NIXL，NIXL 一上来就带五个后端（RDMA/IB、RoCE via UCX、TCP、NVMe-oF、S3 兼容对象存储）。**但 2025 年全年没有任何为 KV cache 定制的硬件参考架构**——那要等到 2026 年 1 月 CES 的 ICMSP 和 3 月 GTC 的 CMX/STX。对存储工程师的含义：2025 年是**接口与生态定型的一年**，NIXL 的五后端选择实际上定义了此后所有存储厂商的接入面。

2. **"AI 训练需要并行文件系统"这个假设在 2025 年被系统性拆解，而且拆解它的是数据而非观点。** Lockwood 在 2 月发的生命周期分析给出了整个窗口内最有杀伤力的一组数字：Llama-3 405B 训练用了 15.6 万亿 token，按每 token 3–5 字节算，**全部输入数据只有 60 TB**；分摊到 16,000 张 GPU、54 天的训练，**每张 GPU 全程只读 3.75 GB**；而 60 TB 能塞进**三台** DGX H100 的本地 SSD（每台 30 TB）。同时 FAST '25 的 Cloudscape 用近 400 个真实 AWS 架构给出实证：**S3 出现在 68% 的架构里，文件系统服务只有 4%**。DeepSeek 的 3FS 更进一步——它**主动放弃读缓存**，理由是训练对每个样本只读一次，而重复按同序读取同一批数据反而有害。三条独立证据指向同一结论：训练输入路径的瓶颈不是聚合带宽，是**性能方差**，而消除方差最便宜的办法是本地 NVMe，不是更快的共享存储。

3. **Checkpoint 在 2025 年最重要的贡献是一个抽象，不是一个性能数字：与并行策略解耦。** ByteCheckpoint（NSDI '25）的核心是 **parallelism-agnostic checkpoint representation**，使得加载时可以做 checkpoint resharding——从一种并行配置切换到另一种，或者从预训练交给评估/后训练任务。它的性能数字（checkpoint stall 平均降低 54.20×，最高 161.50×）固然亮眼，但真正改变工程实践的是"checkpoint 不再绑定写它时的并行拓扑"。配合 Lockwood 指出的一个容易被忽略的事实——**checkpoint 大小不随作业规模变化**（405B 模型在 3 节点和 16,000 节点上的 checkpoint 一样大，因为每步之后的全局同步使所有数据并行副本相同）——你会得到一个和 HPC 完全不同的容量规划模型。

4. **2025 年出现了三份独立的定量证据，一致表明厂商推荐的 I/O 配置远高于实际需求。** ①Lockwood 的分级 checkpoint 推算：500 GB checkpoint 每 10 分钟排一次到共享存储，**共享存储只需要约 1 GB/s 总带宽**；②他与 VAST 在 SC'25 PDSW 上报告的实测——分析 **40 个生产 LLM 训练作业的 8.5 万次 checkpoint**，结论是即使万亿参数模型也只需几百 GB/s 的全局 checkpoint 带宽，通常远低于 1 TB/s，并据此给出需求侧（demand-side）容量模型；③CHEOPS '25 的块层 trace 显示**模型 offload 根本打不满 NVMe SSD**，且 KV cache offload 的读写带宽极不对称（读 2.0 GiB/s vs 写 11.0 MiB/s）。**如果你在 2025 年因为"AI 需要极高 IOPS/带宽"而做过采购，这三份证据值得回头核对。**

5. **开源在 2025 年第一次让外部工程师能读到前沿 AI 存储的真实实现。** 3FS（MIT 许可，2/28）、Mooncake Store（3/7）、ByteCheckpoint（4 月）、Dynamo + NIXL（GTC 3 月）——四个都是被生产验证过的系统，且都在 2025 年内开源。这在此前是没有的：AI 存储的最佳实践一直锁在几家公司内部。对存储工程师的实际价值：**3FS 的 CRAQ 复制实现、Mooncake 的 KVCache 池调度器、NIXL 的后端抽象，都是可以直接读代码而不是读论文的**。

---

## 2. 工作负载变化地图（2025 全年）

| 路径 | 本窗口内确认的变化 | 关键证据 |
|---|---|---|
| **训练输入** | 共识从"需要超高带宽共享存储"转向"**本地 NVMe + 一次性 staging**"。数据准备被明确移出 GPU 集群，放到独立 CPU 集群（价格比 13–22×）。读缓存被论证为在单 epoch 训练下无价值甚至有害。 | Lockwood 生命周期分析；DeepSeek 3FS 设计说明；FAST '25 Cloudscape |
| **Checkpointing** | 从"单层大写"变为**分级异步**（GPU→CPU DRAM→邻居 SSD via RDMA→共享存储），并首次出现与并行策略解耦的表示。云厂商开始把分级 checkpoint 产品化。 | NSDI '25 ByteCheckpoint；Lockwood 分级方案；AWS SageMaker HyperPod managed tiered checkpointing；MLPerf Storage v2.0 首次加入 checkpoint 基准 |
| **模型分发/冷启动** | 容器镜像与模型权重加载被识别为独立的存储问题；出现"运行时镜像"这类新抽象，以及把模型权重原始张量块缓存在共享 host memory 的做法。 | FAST '25 FlacIO；SOSP '25 Aegaeon |
| **推理状态（KV cache）** | 从零到形成完整栈：架构（Mooncake）、多层存储（IMPRESS）、传输库（NIXL 五后端）、缓存层（LMCache）、编排（Dynamo / llm-d）。**但全年无专用硬件**。 | FAST '25 Mooncake / IMPRESS；GTC 2025 Dynamo+NIXL；Red Hat Summit 2025 llm-d；SOSP '25 DiffKV / Jenga |
| **GPU 直连存储** | GPU 中心存储从"绕过 CPU 访问 NVMe"进化到"要有文件抽象与隔离"；同时 NVIDIA 提出 GPU 侧 IOPS 屏障问题（SCADA）。 | FAST '25 GeminiFS；Lockwood GTC 2025 recap |
| **向量 / RAG** | 对象存储原生支持向量成为云厂商动作；ANNS 系统开始以 SSD+单卡 GPU 为成本目标。 | AWS S3 Vectors（7 月预览 → 12 月 GA）；FAST '25 FusionANNS |
| **网络与内存标准** | 三大标准在 2025 年集中落地（UALink 1.0 四月、Ultra Ethernet 1.0 六月、CXL 4.0 十一月），**全部在网络/内存侧，存储协议本身未变** —— 这正是 2026 年出现协议级扩展的前提。 | UALink / UEC / CXL 官方发布 |

---

## 3. 深度必读（按技术价值排序）

### 3.1 LLM training without a parallel file system（技术长文）

- **作者/单位**：Glenn K. Lockwood（时任 Microsoft AI Infrastructure Architect）
- **时间**：2025-02-01
- **类型/证据级别**：**独立专家分析 + 第一方运维经验**（作者明确声明博客非权威来源、且披露雇主关系）；核心论证基于公开数据与算术，可自行复核
- **链接**：https://blog.glennklockwood.com/2025/02/llm-training-without-parallel-file.html ｜ 二次报道 https://www.blocksandfiles.com/ai-ml/2025/02/04/very-large-ai-model-training-uses-object-storage/1602990
- **一句话判断**：2025 年对分布式存储工程师**最高信息密度**的单篇材料，且它的价值在 2026 年被 ByteDance/OSDI 最佳论文与 IO500 争论进一步验证。**评分 23/25**（架构5/证据4/存储相关性5/AI特异性5/生产价值3/新颖性1）
- **它的论证结构**：把 LLM 生命周期拆成数据摄取、数据准备、模型训练、部署推理四段，逐段论证并行文件系统提供的能力（目录层级、细粒度一致性、文件锁、read-modify-write）在哪一段都不是必需的。
  - **数据摄取**：爬取的是数亿计的小网页，但会立刻被打包进大的 *data container*，元数据单独存进分布式 KV store。write-once、不原地修改 → 天然契合对象不可变性。用文件系统存单个网页会导致全量扫描要遍历数千亿小文件。
  - **数据准备**：Spark 式流水线，每任务读几百 MB → 内存处理 → 整块写回。**去重步骤需要全局同步**（每条数据与每条数据比较），因此常放在紧邻原始数据对象存储的集中位置，有时配 RDMA fabric。关键：**这一步不在 GPU 节点上做**。给出的 Azure 归一化价格是这个决策的依据：$1.00 = 96 核通用 VM（384 GB RAM）；$1.65 = 176 核 HPC VM（NDR IB，768 GB）；$22.55 = 96 核 + 8×H100 VM。GPU 在数据处理上并不能带来 13–22× 加速，所以在 GPU 上做内联数据处理没有经济性。
  - **训练读**：两个理由使"每步从共享存储随机重读"的想象不成立——①输入不是小文件，早已被打包成大对象；②tokenized 数据非常密。量化：Llama-3 405B 用 15.6 万亿 token，token 大小 3–5 字节 → **全部输入 60 TB**；16,000 GPU、54 天 → **每 GPU 全程 3.75 GB**。60 TB 可以放进 3 台 DGX H100（各 30 TB 本地 SSD）；在 2,000 节点的集群上等于**整个数据集有数百份副本**。因此真正的挑战不是带宽而是**性能方差**，最便宜的消除方式是节点本地 NVMe（GPU 与数据之间只隔一段 PCIe）。附带三个好处：GPU 永不等共享存储；节点故障时输入数据可从存活节点经后端 IB 恢复，训练开始后**再也不需要读共享存储**；扩容加 GPU 时 I/O 性能线性扩展。前沿模型恰好只训练一个 epoch（每 token 处理一次以取得最优模型质量），所以训练循环内两张 GPU 永远不需要读同一个 token，无需节点间搬运。
  - **训练写（checkpoint）**：**关键洞察——checkpoint 大小不随作业规模变化**。405B 模型在 16,000 节点上的 checkpoint 与在 3 节点上一样大，因为每步后的全局同步使各数据并行副本相同，只需保存一份权重（当前 SOTA LLM 不到 100 TB 量级）。实践采用三级异步：①每步后 GPU→本节点 CPU DRAM（受 PCIe/NVLink 限制，500 GB 约 1 秒，GPU 可立刻解阻塞，但 DRAM 副本不受保护）；②异步经 RDMA 从 CPU DRAM→**邻居节点本地 SSD**（500 GB 约 10 秒，可每 10 次 DRAM checkpoint 做一次）；③异步从本地 SSD→共享存储（500 GB 约 1–2 分钟，可每 10 分钟一次）。**由此推出共享存储的要求极低：若 500 GB 每 10 分钟排一次，共享存储只需 1 GB/s 总带宽**；且写模式是完整 checkpoint 文件的简单拷贝——不再有"奇形张量边序列化边落盘"，而是不透明字节流以任意最优传输大小和并发写向远端对象。这**完美契合对象存储**。引用 VAST 的量化：万亿参数模型在 3,072 GPU 上用一个 273 GB/s 的文件系统即可达到 **99.7% forward progress**（仅 0.3% 时间花在 checkpoint）；并指出 HDD 型 Azure Blob 用 IOR 压测写入超过 1 TB/s。
  - **推理**：模型权重加载是整对象只读批量拷贝，对象 API 天然适配；向量库与 KV cache 并不受益于并行文件系统。
- **一个通常被忽略的运维论点**：没有并行文件系统意味着**没有持久的、脆弱的客户端-服务端状态**。作者在一个新 H200 训练集群的验收现场意识到：IB 拥塞与路由问题期间**不存在文件系统 eviction 问题**，重大 fabric 事件后无需清理挂载点；I/O 可能失败但会在 fabric 恢复后自行重试恢复；身份也不重要，所有测试可以 root 跑，因为客户端内核与远端存储之间没有隐式信任。**去掉计算节点、LDAP 与健康挂载之间的依赖，消除了快速上线新集群的大量麻烦。** 这是一条纯运维价值、在任何论文里都不会出现的论点。
- **作者自己给出的反面**：不主张已有 Slurm/PFS 工作流的场所丢掉并行文件系统（用户体验成本高）；WEKA / VAST / Qumulo 都已把 S3 作为一等接口，多协议访问提供了平滑迁移路径；WEKA WARRP、VAST InsightEngine 这类"AI data platform"正在把并行文件系统的架构优势延伸到推理与向量查询侧。
- **2026 年的后验**：这篇文章的核心论断在 2026 年得到两方面强化——ByteDance/中科大的 OSDI '26 最佳论文用生产 trace 证明真正的瓶颈是跨 DC 流量、启动期争用与 CPU 密集的数据转换（都不是共享存储带宽）；Lockwood 本人在 ISC'26 上进一步提出"只有糟糕的 AI 才需要大量 IOPS"。另一方面，Google Cloud 在 2026 年把 Managed Lustre 推到 10 TB/s、把 Rapid Bucket 推到 20M ops/s，说明云厂商的商业判断与这篇文章相反。**两条路线在 2026 年仍未收敛。**

---

### 3.2 Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot

- **作者/单位**：Ruoyu Qin（Moonshot AI + 清华），Zheming Li、Weiran He、Jialei Cui、Xinran Xu（Moonshot AI），Feng Ren、Mingxing Zhang、Yongwei Wu、Weimin Zheng（清华）
- **时间/会场**：FAST '25（2025-02-25～27，Santa Clara），**Best Paper Award**（2025-02-25 宣布），pp. 155–170
- **类型/证据级别**：同行评审 + 第一方生产结果 + **开源（含真实 trace）**
- **链接**：论文 https://www.usenix.org/conference/fast25/presentation/qin ｜ 代码与 trace https://github.com/kvcache-ai/Mooncake ｜ 文档 https://kvcache-ai.github.io/Mooncake/
- **一句话判断**：定义了 2025 年整个 KV cache 存储方向的论文，且它的开源节奏比论文本身影响更大。**评分 23/25**（架构5/证据5/存储相关性5/AI特异性5/生产价值3/新颖性0——分离式缓存池本身不新，新的是把它作为 LLM 服务的一等架构）
- **架构与数据路径**：KVCache 中心的分离式架构。①分离 prefill 与 decode 集群；②**把 GPU 集群里未被充分利用的 CPU、DRAM、SSD 和 NIC 资源组织成一个分离式 KVCache 池**；③核心是一个 KVCache 中心的全局缓存 + 调度器，在最大化吞吐与满足延迟相关 SLO 之间平衡。
- **实验与结果**：真实 trace 下**有效请求容量提升 59%~498%**（同时满足 SLO）；长上下文输入场景优势最明显。生产：运行在**数千节点**上，每日处理**超过 1000 亿 token**；实际部署中使 Kimi 在 NVIDIA A800 与 H800 集群上分别多处理 **115% 与 107%** 的请求。
- **相对传统云存储真正新的是什么**：机制层面"用闲置资源做分离式缓存池"是老思想（分离式内存、客户端缓存）。**真正新的是它把"多存储换少计算"确立为一个显式的架构取舍，并给出了 SLO 感知的调度器作为这个取舍的执行者。** 对存储工程师最有价值的一句话在标题里：这是一个**用存储容量换 GPU 计算**的经济学问题，而不是一个性能问题。
- **2025 年内的开源节奏（比论文更重要）**：2024-12 vLLM 官方支持 Mooncake Transfer Engine（分离式 prefill 与 KV 传输）→ 2025-02-21 发布 FAST'25 论文所用的更新 trace → **2025-03-07 开源 Mooncake Store**（基于 Transfer Engine 的分布式 KVCache）→ 2025-04-10 SGLang 官方支持 Transfer Engine → 2025-04-22 LMCache 官方支持 Mooncake Store 作为远程 connector → 2025-05-05 Mooncake 团队支持 SGLang 发布在 96 张 H100 上部署 DeepSeek PD 分离的指南 → 2025-05-08 Mooncake × LMCache 合作 → 2025-12-19 Transfer Engine 直接集成进 vLLM v1 作为 PD 分离下的 KV Connector，同日集成进 TensorRT-LLM → 2025-12-23 SGLang 引入以 Mooncake 为传输后端的 **Encode-Prefill-Decode (EPD) 分离**（把计算密集的多模态 encoder 与语言模型节点解耦，用 Mooncake 的 RDMA 引擎做大型多模态 embedding 的零拷贝传输）。**2026-01-28 更新**：腾讯与 NVIDIA 联合社区的 FlexKV 也开始支持通过 Mooncake Transfer Engine 做分布式 KVCache 复用。
- **可复用设计与未解问题**：①"闲置资源盘点"是任何 GPU 集群运营者都可以先做的免费工作——统计你的 GPU 节点上 CPU、DRAM、SSD、NIC 的实际利用率，这决定了你有多少可用的 KVCache 池容量；②Transfer Engine 已经被 vLLM / SGLang / TensorRT-LLM 三大引擎接受，**这是 2025 年 KV 传输事实标准竞争的结果之一**（另一个是 NIXL），值得读它的后端抽象；③未解：论文层面对一致性、驱逐语义、多租户隔离与故障恢复的披露有限——这个缺口一直延续到 2026 年（见 2026 年报 §9）。

---

### 3.3 DeepSeek 3FS（Fire-Flyer File System）开源

- **来源/单位**：DeepSeek AI；作为 "Open Source Week" 第五天发布（2025-02-28），**MIT 许可**
- **类型/证据级别**：**开源实现 + 第一方生产/压测数字**（无同行评审；相关的 Fire-Flyer 2 论文为 2024-08，在窗口外）
- **链接**：https://github.com/deepseek-ai/3FS ｜ 第三方架构梳理 https://juicefs.com/en/blog/engineering/deepseek-3fs-vs-juicefs-architecture-feature
- **一句话判断**：2025 年对存储工程师**最可读**的一份前沿 AI 文件系统实现，且它的一个设计选择（放弃读缓存）在概念上比它的性能数字更重要。**评分 21/25**（架构5/证据3/存储相关性5/AI特异性5/生产价值3/新颖性0）
- **架构与数据路径**：分离式架构，把数千块 SSD 的吞吐与数百个存储节点的网络带宽聚合起来，使应用以**位置无关（locality-oblivious）**的方式访问 PB 级存储。组件：存储层（数据块）、**无状态元数据服务**（不保存状态信息，从而提升扩展性与可靠性；集群配置存在 ZooKeeper/etcd 等可靠服务里）、客户端（FUSE 客户端提供通用 POSIX 接口易用性；native 客户端性能更高但需改应用）、以及实现 **CRAQ 协议**的复制系统以提供强一致性。所有组件之间用 RDMA 通信。元数据与存储服务向集群管理器发心跳，管理器处理成员变更并分发配置；多个管理器部署、其中一个被选为主。
- **最值得注意的设计选择**：**几乎单一地优先随机读性能，并近乎完全忽略读缓存。** 理由是训练时计算单元不断随机访问训练数据、且每条数据只读一次，读缓存因此近乎无用；更进一步，反复按同一顺序重复读取同一批数据可能导致无关数据被训练进同一批次，对模型开发有害。**这是一个由 AI 负载语义直接推导出的、与所有传统文件系统相反的设计决策**，也是本条目在概念上的核心价值。
- **DeepSeek 声明的数字（第一方，压测）**：180 个存储节点（每节点 16 块 14–16 TB NVMe SSD + 2 张 200 Gbps IB NIC），服务 10,000 张 PCIe A100 GPU。并发客户端请求下**聚合读吞吐约 6.6 TiB/s**，且这是在**后台还跑着训练任务（额外约 1.4 TB/s 读吞吐）**的情况下取得的。GraySort 基准：25 个存储节点 + 50 个计算节点，把分布在 8,192 个分区上的 **110.5 TiB 数据在 30 分 14 秒内排序完**，平均吞吐 3.66 TiB/分钟（用其开源的 smallpond）。**KV cache 操作峰值读吞吐 40 GiB/s**。
- **对比的可信度问题（必须注意）**：常被引用的对照是"Ceph 在 2024 年初首次达到 1.1 TB/s 读吞吐（68 节点、每节点 10 块 16TB SSD、2×100 Gbps）"。**这是不同硬件配置、不同时间点的跨系统比较，不构成受控实验。** 节点数（180 vs 68）、每节点 SSD 数（16 vs 10）、网络（200G vs 100G）全部不同。用它论证"3FS 比 Ceph 快 6 倍"是错误的。
- **AI 用例（DeepSeek 自述）**：数据准备阶段的大数据集管理；训练 dataloader 的直接随机访问（**从而可能不再需要复杂的预取**）；高吞吐并行模型 checkpointing；以及从低成本大容量 SSD 提供推理 KVCache。
- **可复用设计与未解问题**：①"你的 AI 负载是否真的受益于读缓存"是可以在自己系统上廉价验证的——统计单 epoch 训练下的缓存命中率，如果接近零，那么缓存占用的内存是纯损失（这条与 2026 年 FalconFS 关于客户端元数据缓存的结论同构，两者相隔一年、路径不同、结论一致）；②CRAQ 是一个值得研究的选择：它用链式复制换取了强一致下的读扩展性，与多数 AI 存储系统的最终一致取向不同；③未解：无第三方复现；元数据服务无状态化后的扩展上限未公开；FUSE 与 native 客户端的性能差距未在开源材料中系统量化。

---

### 3.4 ByteCheckpoint: A Unified Checkpointing System for Large Foundation Model Development

- **作者/单位**：Borui Wan（香港大学），Mingji Han、Yiyao Sheng、Yanghua Peng、Haibin Lin、Mofan Zhang、Zhichao Lai、Menghan Yu、Junda Zhang、Zuquan Song、Xin Liu（字节跳动），Chuan Wu（香港大学）
- **时间/会场**：NSDI '25（2025-04-28～30，Philadelphia），pp. 559–578；2024-12 录用，**2025-04 正式开源**
- **类型/证据级别**：同行评审 + 第一方生产结果 + **开源（PyPI 可装）**
- **链接**：论文 https://www.usenix.org/conference/nsdi25/presentation/wan-borui ｜ PDF https://www.usenix.org/system/files/nsdi25-wan-borui.pdf ｜ arXiv https://arxiv.org/abs/2407.20143 ｜ 代码 https://github.com/ByteDance-Seed/ByteCheckpoint ｜ 视频（13 分钟）https://www.youtube.com/watch?v=dPsejzDZv7o
- **一句话判断**：2025 年 checkpoint 方向最重要的**抽象**贡献，性能数字只是副产品。**评分 22/25**
- **问题定位**：LFM 开发全生命周期都需要 checkpoint 在不同并行配置之间 **reshard**——故障恢复、GPU 资源变化、并行策略变化、把 checkpoint 派发给评估任务、从预训练转到后训练。生产环境里不同模型用不同框架和不同存储后端（取决于模型规模与训练规模）。
- **架构**：①**parallelism-agnostic checkpoint representation**，使加载时高效 resharding 成为可能（这是全文核心）；②通用的保存/加载工作流，适配多种训练框架与多种存储后端——为每个框架定制一个 **Planner**，各 worker 先用 Planner 生成本地计划，再协作生成全局计划；**Engine** 在每个训练 worker 上执行计划，分析给定 checkpoint 路径以确定合适的存储后端，然后与 Storage I/O 层交互执行；③全栈 I/O 优化保证效率与扩展性；④一套监控工具用于大规模性能分析与瓶颈定位。checkpoint 在磁盘上分为三部分目录（model / optimizer / 其余），各含一个 `.metadata` 文件与多个 `.distcp` 张量数据文件。
- **实验与结果**：相比现有开源 checkpoint 系统，运行时 checkpoint stall **平均降低 54.20×**；保存与加载时间分别**最高提升 9.96× 与 8.80×**。与 Megatron Distributed Checkpoint (MCP) 等基线相比，stall 降低幅度 **12.13×–161.50×**，端到端保存与加载分别平均快 **6.05× 与 3.88×**。
- **相对传统做法真正新的是什么**：**真新**。传统存储系统里"数据布局与写它的进程拓扑解耦"是通过文件系统抽象天然获得的；但分布式 checkpoint 因为张量分片，事实上把并行拓扑烧进了数据布局。ByteCheckpoint 把这层耦合显式打开。对存储工程师的可迁移含义：**如果你的系统为 AI 客户存 checkpoint，客户能否 reshard 决定了你的数据是否可迁移**，这是一个存储侧应该关心而通常不关心的属性。
- **可复用设计与未解问题**：①它已经是可 `pip install` 的库，评估成本极低，是本报告中**最便宜的验证入口**；②2025-08 起支持 MUSA 平台，说明它的后端抽象经受了非 CUDA 加速器的检验；③未解：resharding 的正确性验证方法（论文提供了 metadata/tensor 文件检查工具，但跨并行配置的数值等价性如何保证未在摘要层展开）。

---

### 3.5 MLPerf Storage v2.0 —— 首次引入 checkpoint 基准

- **来源/单位**：MLCommons MLPerf Storage 工作组（联席主席 Curtis Anderson、Oana Balmau；MLPerf 负责人 David Kanter）
- **时间**：2025-08-04 公布结果
- **类型/证据级别**：**跨厂商可比的独立基准**（架构中立、可复现；Apache 许可、GitHub 可得）——本报告全年**唯一**的此类证据
- **链接**：官方 https://mlcommons.org/2025/08/mlperf-storage-v2-0-results/ ｜ 结果表 https://mlcommons.org/benchmarks/storage/ ｜ 基准套件 github.com/mlcommons/storage
- **一句话判断**：2025 年唯一能用来做跨厂商比较的数据源，而它最重要的变化是**第一次把 checkpoint 纳入基准**。**评分 20/25**
- **本轮的关键变化**：v2.0 **新增复现真实 AI 训练 checkpointing 的测试**，重点是 scale-out 系统上的 LLM。设立动机在官方稿里说得很清楚：AI 训练界已经有数学模型可以通过权衡"定期 checkpoint 的开销"与"故障恢复的预期频率与代价（回滚、恢复最近 checkpoint、重启训练、重做丢失的工作）"来优化集群性能与利用率，但**这些模型需要关于存储系统规模与性能的准确数据**——而这正是此前缺失的。
- **参与规模与构成**：**26 个组织、超过 200 条性能结果**，来自 7 个国家，创 MLPerf 基准的组织数与提交数记录。受测系统服务的加速器数约为 v1.0 轮的 **2 倍**。技术路线多样性：6 个本地存储方案、2 个使用存储内加速器的系统、13 个软件定义方案、12 个块系统、16 个本地部署共享存储方案、2 个对象存储。
- **值得记住的具体结果**：
  - **Argonne Aurora**：**DAOS** 用 1,024 个 DAOS 服务器中的 **128 个**子集达到接近 **1 TB/s 写、600 GB/s 读**，使 **LLaMA3-405B 的完整 checkpoint 在 10 秒内完成**；Lustre（驱动 ALCF 的 100 PB 系统）同时提交。基准使用 DLIO 模拟训练期 I/O。https://www.alcf.anl.gov/news/auroras-daos-and-lustre-excel-mlperf-storage-benchmark-large-scale-ai
  - **Alluxio**：分布式缓存在多个通常受 I/O 瓶颈困扰的负载上达到**最高 99.57% GPU 利用率**；ResNet-50 24.14 GiB/s 支撑 128 个加速器。
  - **Western Digital OpenFlex Data24 4200**：3D-UNet 用 3 个客户端向 36 个模拟 H100 交付 **101.6 GB/s**（公开报告值 106.5 GB/s，NVMe over RoCE 下接近介质极限）；ResNet-50 在 3 节点上支撑 **186 vGPU** 且维持 >33 GB/s。
- **基准本身的方法学争议（必须一并记录）**：Western Digital 自己的白皮书指出，MLPerf Storage 对 **"clients" 与 "vGPU 数" 作为归一化指标的严重依赖可能产生误导且难以解读**——基准客户端既非标准化、也不与真实应用服务器一一对应。同一份材料给出的例子很有说服力：某提交以绝对值取得最高 3D-UNet 吞吐（>120 GB/s），但那是通过大量横向扩展（15 个客户端）达成的；一旦把系统规模、功耗与物理占用一并考虑，架构效率的排序会完全不同。**结论：读 MLPerf Storage 结果必须同时看客户端数、节点数、功耗与机架单元，只看总吞吐会得到错误结论。**
- **2026 年的后验**：**MLPerf Storage 在 2026 年上半年没有新一轮**（v2.0 是 2025-08，下一轮预计 2026-08）。这意味着截至本报告撰写时，**跨厂商可比数据仍然是这一轮的**——这既提高了本条目的长期价值，也构成 2026 年报最大的证据缺口。

---

### 3.6 NVIDIA Dynamo + NIXL（GTC 2025 开源）

- **来源/单位**：NVIDIA（GTC 2025，2025-03）
- **类型/证据级别**：**开源实现 + 厂商声明**（架构可读代码，性能数字为厂商声明）
- **链接**：官方博客 https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/ ｜ llm-d 集成说明 https://developer.nvidia.com/blog/nvidia-dynamo-accelerates-llm-d-community-initiatives-for-advancing-large-scale-distributed-inference ｜ 报道 https://www.blocksandfiles.com/ai-ml/2025/07/07/nvidia-extends-llm-memory-with-tiered-kv-caching-and-dynamo-engine/101056
- **一句话判断**：2025 年真正改变存储厂商接入面的事件——**NIXL 的五个后端选择定义了此后所有 AI 存储产品需要支持什么**。**评分 19/25**（架构4/证据2/存储相关性5/AI特异性5/生产价值2/新颖性1）
- **架构（四个组件）**：**Disaggregated Serving**（分离 prefill 与 decode，NVIDIA 称参与推理的 GPU 越多、效率增益越大）；**Smart Router**（把请求路由到缓存命中率最高的 worker，即 KV-cache-aware routing）；**Distributed KV Cache Manager**（维护全局 radix tree registry，在多层内存系统里管理 KV cache——GPU HBM → 系统内存 → 直连 SSD → 网络外部存储；目标是降低 TTFT、提升吞吐、支持更长上下文）；**NIXL**（NVIDIA Inference Transfer Library，为推理负载优化的数据传输，降低同步开销、智能批处理、支持异构内存访问、动态选择最优传输后端）。工程上用 Rust 写性能敏感模块、Python 提供灵活性。
- **对存储工程师最关键的一点——NIXL 的五个后端**：RDMA/InfiniBand、经 UCX 的 RoCE、TCP 兜底、**NVMe-oF**、**S3 兼容对象存储**。这份清单实际上是 NVIDIA 对"KV cache 会落在哪些存储介质上"的下注。**Dynamo 1.0 只支持卸载到系统 CPU 内存，SSD 与网络对象存储在后续版本中扩展。** vLLM 通过 `--kv-transfer-config` 把 NIXL 作为分离式 prefill 的主要 connector 选项之一（与 LMCacheConnector、MooncakeConnector 并列）。
- **2025 年内的生态扩散**：Red Hat Summit 2025（5 月）发布 **llm-d** 社区——基于 vLLM 与 Inference Gateway、Kubernetes 原生的大规模推理架构，**依赖 NIXL** 做 PD 分离下的 KV cache 传输；Dynamo KV Cache Manager 把较少访问的 KV cache 卸载到 CPU host memory、SSD 或网络存储，并通过 NIXL 对接不同存储提供方，为 llm-d 实现无缝 KV 分层。2025-09 **Dynamo 集成 LMCache 作为其 KV 缓存层方案**（LMCache PR #1223），带来跨应用重启的持久缓存与 GPU+CPU+SSD 混合缓存策略。
- **相对传统做法真正新的是什么**：**部分新**。分层缓存、内容寻址、全局 registry 都不新。真正新的是**一个统一的数据搬运 API，用同一套语义在不同内存与存储层级之间移动数据**，且明确支持非阻塞与非连续传输——后者是 KV cache 分页布局的直接需求。这个 API 的存在，使得存储厂商第一次有了一个明确的、非私有的接入点。
- **风险与后验**：2026 年 NVIDIA 用 CMX/STX + DOCA Memos 把这个接入点进一步硬件化和体系化，同时也进一步收紧了对生态的控制。2025 年 NIXL 的开源性质与 2026 年 DOCA Memos 的定位差异，值得任何做长期技术选型的人对比阅读。

---

### 3.7 IMPRESS: An Importance-Informed Multi-Tier Prefix KV Storage System for Large Language Model Inference

- **作者/单位**：Weijian Chen、Shuibing He、Haoyang Qu、Ruidong Zhang、Siling Yang、Ping Chen、Gang Chen（浙江大学），Yi Zheng、Baoxing Huai（华为云）
- **时间/会场**：FAST '25
- **类型/证据级别**：同行评审
- **链接**：https://www.usenix.org/conference/fast25/presentation/chen-weijian
- **一句话判断**：2025 年第一篇明确指出"**把前缀 KV 放到磁盘并复用它不一定能降低 TTFT**"并给出选择性加载方案的存储论文。**评分 19/25**
- **问题定位**：现代 LLM 应用常在用户 query 前置长上下文以提升输出质量，这些上下文在多次 query 间部分或完全重复。现有系统存储并复用这些上下文的 KV（前缀 KV）以减少冗余计算与 TTFT。**但当 CPU 内存不足、前缀 KV 必须落盘时，复用它们并不总能降低 TTFT，因为磁盘 I/O 延迟很高。**
- **机制**：利用一个洞察——**跨注意力头之间，重要 token 的索引集合有显著相似性**——设计 I/O 高效的重要 KV 识别算法；再通过重要性感知的 KV 管理来优化前缀 KV 的存储与缓存，**只加载重要的前缀 KV**。
- **结果**：相比 SOTA 系统 **TTFT 最高降低 2.8×**，同时保持可比的推理精度。
- **相对传统做法真正新的是什么**：**真新**。传统存储的缓存/分层决策是基于访问频率与近期性（frequency/recency）——这是内容无关的。IMPRESS 的决策依据是**数据在模型计算中的重要性**，这是一个内容与语义相关的信号，传统缓存框架无法表达。它与同年的 DiffKV（SOSP '25，"token 重要性各异、注意力头间动态稀疏模式不同"）指向同一方向。
- **对存储工程师的含义**：**"缓存哪一部分"这个决策正在从存储层移交给应用层。** 如果你的系统提供 KV 存储服务，纯 LRU/LFU 驱逐会持续输给能利用重要性信号的方案——除非你提供一个让应用传递重要性提示的接口。这与 2026 年"应用声明意图、存储层生成计划"的趋势是同一条线的早期形态。

---

### 3.8 An I/O Characterizing Study of Offloading LLM Models and KV Caches to NVMe SSD

- **作者/单位**：Zebin Ren、Krijn Doekemeijer、Tiziano De Matteis、Animesh Trivedi（Vrije Universiteit Amsterdam），Christian Pinto、Radu Stoica（IBM Research）
- **时间/会场**：**CHEOPS '25**（第 5 届 Challenges and Opportunities of Efficient and Performant Storage Systems workshop，与 EuroSys '25 同址，2025-03-30～04-03，Rotterdam），pp. 23–33
- **类型/证据级别**：同行评审（workshop）+ **开源脚本与 I/O trace**
- **链接**：PDF https://atlarge-research.com/pdfs/2025-cheops-llm.pdf ｜ ACM https://dl.acm.org/doi/10.1145/3719330.3721230 ｜ 代码与 trace https://github.com/stonet-research/cheops25-IO-characterization-of-LLM-model-kv-cache-offloading-nvme
- **一句话判断**：2025 年**唯一**一份公开的、可复现的 LLM offload 块层 I/O 刻画，也是欧洲在本方向的主要产出。**评分 18/25**（架构2/证据4/存储相关性5/AI特异性5/生产价值1/新颖性1）
- **方法**：从两个支持模型与 KV cache 落盘 offload 的推理框架（**DeepSpeed** 与 **FlexGen**）采集并刻画**块层 I/O trace**。配置：DeepSpeed 用 64 GiB 主存做 NVMeVirt SSD、FlexGen 用 96 GiB；模型 offload 设定输入 256 token、输出 32 token（已验证 token 数不影响 decode 性能）；KV cache offload 时关闭模型 offload，用 OPT-6.7B（能完整放进 24 GiB GPU 内存的最大模型），输入输出各 256 token 以体现更长上下文。
- **四条结论（每一条都对存储设计有直接含义）**：
  1. **基于 libaio 的张量 offload 在读写方向上都比 POSIX 提供更高 I/O 带宽**；
  2. 模型 offload 的 I/O 负载在块层**以 128 KiB 读为主**，DeepSpeed 与 FlexGen 均如此；
  3. **模型 offload 打不满 NVMe SSD**；
  4. KV cache offload 的 I/O 负载**同时包含读与写、同样以 128 KiB 请求为主**，但**读的平均带宽远高于写：2.0 GiB/s vs 11.0 MiB/s**。
- **为什么这四条重要**：结论 2 与 4 直接否证了"AI 存储需要极高 IOPS 应对小随机 I/O"这一常见说法——**128 KiB 是一个对任何现代存储系统都友好的请求大小**。结论 3 说明在这类负载下瓶颈在别处（框架、PCIe、同步），加更快的 SSD 不会有收益。结论 4 揭示的读写带宽三个数量级的不对称，意味着**为 KV offload 设计的存储层应该是读优化的，写路径可以牺牲**——这与传统"读写均衡"的设计取向不同。
- **必须承认的外推限制**：单机规模、OPT-6.7B（远小于前沿模型）、用 NVMeVirt **仿真** SSD 而非真实设备、只覆盖两个框架且都不是 2025 年主流的生产推理引擎（vLLM/SGLang 未覆盖）。**这些数字不能外推到集群规模或前沿模型。** 但它开源了 trace，任何人都可以在自己的环境上重跑并对比——这是本报告中**可复现性最高**的条目。

---

### 3.9 GeminiFS: A Companion File System for GPUs

- **作者/单位**：Shi Qiu、Weinan Liu、Yifan Hu、Jianqin Yan、Zhirong Shen（厦门大学 NICE Lab），Xin Yao、Renhai Chen、Gong Zhang（华为理论实验室），Yiming Zhang（厦门大学 NICE Lab + 上海交大）
- **时间/会场**：FAST '25
- **类型/证据级别**：同行评审
- **链接**：https://www.usenix.org/conference/fast25/presentation/qiu
- **一句话判断**：GPU 中心存储从"能绕过 CPU"进化到"要有文件系统语义"的转折点。**评分 19/25**
- **问题定位**：GPU 中心存储方案让 GPU 经 NVMe 队列直接访问存储设备、完全绕过 CPU，解决了 CPU 中心方案的高 CPU-GPU 同步开销、I/O 流量放大与高 CPU 处理延迟。**但 SOTA 的 GPU 中心方案没有文件抽象，也没有传统宿主文件系统的管理能力（如细粒度隔离与访问控制）**，无法满足 GNN 与 LLM 这类需要快速文件访问与数据共享的 GPU 加速 ML 应用，因此在实际 ML 场景中低效且不便。
- **架构与数据路径**：向 GPU 程序提供文件系统接口，实现对 NVMe 存储的**直接基于文件的访问**，而该存储由宿主文件系统管理。三个关键机制：①**把元数据直接嵌入文件本身**来实现宿主与 GPU 文件系统之间的元数据同步；②**扩展现有 NVMe 驱动，使 CPU 与 GPU 可以并行为存储设备建立各自的控制平面**；③提供 GPU 友好的**软件定义 page cache** 以充分利用 GPU 内部带宽。另提供 libGemini 库为 GPU 程序员抽象底层复杂性。
- **相对传统做法真正新的是什么**：**真新的是"companion file system"这个定位**——不取代宿主文件系统，而是与之共存并共享同一份数据，通过把元数据内联到文件里解决双方的元数据一致性。这个技巧对任何需要让两个不共享内核的执行域访问同一存储的场景都适用。
- **2026 年的后验**：这条线在 2026 年由 RosenBridge（FAST '26，把 GDS 这类快速 I/O 路径打通到虚拟化边界之外）和 Strata 的 GPU-assisted I/O（用 CUDA kernel 做搬运）继续推进。**"GPU 作为存储客户端的一等公民"是一个从 2025 延续到 2026 的稳定方向。**

---

### 3.10 其他值得读但不单独展开的条目（按路径分组）

**训练输入 / 通用存储机制（FAST '25）**
- **Cloudscape: A Study of Storage Services in Modern Cloud Architectures**（UW-Madison + NetApp）：近 **400 个部署在 AWS 上的云架构**数据集，深入分析存储服务使用情况。核心发现：**S3 是最普遍的存储服务（68%），文件系统服务很罕见（4%）**；存储层异构是常态；存储服务主要与 Lambda 和 EC2 交互，同时也作为更专门的 ML 与分析服务的底座。→ **这是 §3.1 论断的独立实证支撑**，也是全年最好用的一份"云上到底怎么用存储"的定量材料。https://www.usenix.org/conference/fast25/presentation/satija
- **FlacIO: Flat and Collective I/O for Container Image Service**（华为）：分析容器镜像服务的 I/O 瓶颈，指出现有方案存在高 I/O 放大与过量网络流量，根因在于**面向存储、面向全局的容器镜像抽象**。提出**面向内存、面向服务**的抽象——**runtime image**，表示容器服务根文件系统的内存状态，从而实现高效网络传输与快速根文件系统构建；配合宿主节点上的 runtime page cache。结果：容器冷启动延迟相比完整镜像方案降低**最高 23×**、相比 lazy loading 降低 **4.6×**；真实应用中在对象存储与 ML 训练场景分别获得 **2.25× 与 1.7×** 加速。https://www.usenix.org/conference/fast25/presentation/liu-yubo
- **FusionANNS**（HUST + 华为）：用 SSD 和**仅一张入门级 GPU** 做十亿级 ANNS。三个设计：多层索引避免 CPU-GPU 间数据交换；启发式 re-ranking 在保证高精度下消除不必要的 I/O 与计算；冗余感知的 I/O 去重。相比 SPANN **QPS 9.4–13.1×、成本效率 5.7–8.8×**；相比 RUMMY **QPS 2–4.9×、成本效率 2.3–6.8×**。→ 对做向量存储的人：这是"用存储换 GPU"在 ANNS 上的对应版本。
- **LeapGNN**（浙大 + WSU Vancouver）：反转分布式 GNN 训练的范式——现有框架是**模型中心**的，需要把海量图顶点特征搬到模型侧，造成通信瓶颈；由于模型尺寸通常远小于特征尺寸，LeapGNN 改为**特征中心**，把模型搬到特征所在处。配合基于 micrograph 的训练策略增强局部性、特征预聚集合并多次 fetch、以及 micrograph 合并法减少 kernel 切换与同步开销。相比 SOTA（P3）**最高 4.2×**。→ "搬小的那一个"这个原则可直接推广。
- **HiDPU**（山东大学 + 天津大学 + 华为 + CUHK）：面向 DPU 的分离式存储混合索引。观察到分离式存储的数据访问中**地址翻译带来显著 CPU 计算开销与高系统延迟**，且大规模下索引结构本身消耗大量内存。方案：多级索引结构以缓解 DPU 内存、算力与 DPU-host 交互开销的限制；映射条目按地址连续性分为 accurate / PTHash / LPTHash 三类 segment，跨 segment 构建分层 learned index；小的上层索引与高频元数据留在 DPU 上，把交互限制在单次；两阶段异步索引更新策略保证 DPU 与 host 内存间的索引一致性。在**华为 Hi1823 DPU** 上：内存节省**最高 92%**，查询性能**最高 6.3×**。
- **3L-Cache**（北京工业大学 + 微软亚洲研究院）：低开销的学习型驱逐策略。用 4,855 条 trace 评测；相比 HALP 平均 CPU 开销 **-60.9%**、相比 LRB **-94.9%**；小缓存下仅为 LRU 平均开销的 6.4×、大缓存下 3.4×，同时在十二种 SOTA 策略中取得最佳 byte miss ratio 或 object miss ratio。→ 学习型缓存的**开销**问题是它无法进生产的主因，这篇是 2025 年在这个方向上最实用的一篇。
- **PolyStore**（Rutgers + EPFL + Samsung）：针对新兴存储介质的"非层级化"趋势，提出**横向结构**的存储架构，在介质优化的文件系统之上加一个跨用户态与 OS 的 meta 层，使应用可并发访问多个存储设备并做透明细粒度数据放置。微基准 1.11–9.38×、真实应用 1.52–2.02×。

**Checkpoint / GPU 状态（SOSP '25，2025-10-13～16，Seoul）**
- **PhoenixOS (PhOS): Concurrent OS-level GPU Checkpoint and Restore with Validated Speculation**（上海交大 IPADS + NUS）：OS 级 GPU C/R。论文本身给出了一个重要的工程动机：领先的 LLM 训练框架 Megatron 最近才整合了三年前提出的并发 checkpoint 优化，**而 checkpoint 代码现已占其代码库的四分之一**；同时许多新兴训练框架（如 RL 用的）仍缺少这些特性。OS 级 C/R 不给开发者施加实现负担。→ **2026 年的后验**：清华的 GCR（FAST '26）以 PhOS 为 SOTA 基线，报告 checkpoint 延迟再降 63.6%、恢复降 87.1%。这是一条一年内被显著推进的线。https://ipads.se.sjtu.edu.cn/_media/publications/phos-sosp25.pdf
- **DiffKV**（上海交大 + CUHK）：差异化内存管理 + 并行 KV compaction。依据三个观察：key 与 value 对注意力计算的影响不同；token 重要性各异；注意力头间的动态稀疏模式不同。GPU 上的内存管理器并行地把碎片化空闲链表压实成连续区域。
- **Jenga: Effective Memory Management for Serving LLM with Heterogeneity**（UC Berkeley + 清华 + 多机构，含 vLLM 核心作者）
- **cache_ext: Customizing the Page Cache with eBPF**（Columbia + IBM Research）→ 对存储工程师直接有用：把 page cache 策略变成可编程的，与 2026 年华为的 PPC（可编程 page cache）是同一思路的两个独立实现。
- **Aegaeon: Effective GPU Pooling for Concurrent LLM Serving on the Market**（北大 + 阿里）：在共享 host memory 区域里缓存来自模型 checkpoint 的**原始张量块**（称为 Model Cache），每张 GPU 关联一个专用的 page-locked Stage Buffer 用于设备与主机之间的内存拷贝暂存；当被扩容的模型已在 host memory 中缓存时，以多线程、分块、流水线方式直接从 Model Cache 拷到 GPU，**加载时间达到 SOTA 水平（低于一秒）**。→ 模型分发路径上"缓存原始张量而非序列化文件"是一个可直接借用的技巧。
- **Weaver: Efficient Multi-LLM Serving with Attention Offloading**（ATC '25，2025-07-07～09，Boston；清华 Shiwei Gao、Qing Wang、Shaoxun Zeng、Youyou Lu、Jiwu Shu），pp. 587–595。

**云产品（第一方声明，非独立验证）**
- **Google Cloud Next 2025（4 月）—— Rapid Storage**：新的 Cloud Storage **zonal bucket**，为高频访问数据与延迟敏感应用提供个位数毫秒级一致访问；声明**亚 1 ms 随机读写延迟、20× 更快的数据访问、6 TB/s 吞吐**。建立在 Colossus 之上，**与 AI 加速器同置**以最小化延迟；可经 Cloud Storage FUSE 与 TensorFlow/PyTorch 集成；声明随机读写延迟比其他厂商低 5×。Google 的 Sameet Agarwal 表述了动机："要以峰值效率训练、checkpoint 与服务 AI 模型，你需要让 GPU 或 TPU 一直有数据可吃，以最小化浪费的计算。"→ **2026 年的后验**：Rapid Storage 在 2026 年 4 月更名并 GA 为 **Rapid Bucket**，指标提升到 >15 TB/s、20M ops/s。
- **Google Cloud Managed Lustre GA（2025-07-08）**：基于 **DDN EXAScaler**，四个性能层（每 TiB 容量 125 / 250 / 500 / 1000 MB/s），可扩展到 8 PB。POSIX 兼容并行文件系统。https://cloud.google.com/blog/products/storage-data-transfer/google-cloud-managed-lustre-for-ai-hpc
- **AWS Amazon S3 Vectors**：**2025-07 预览**——首个原生支持存储与查询向量的云对象存储，声明把上传、存储、查询向量的成本降低**最高 90%**，与 Bedrock Knowledge Bases、SageMaker、OpenSearch 集成，亚秒级查询。**2025-12 re:Invent GA**——每索引 **20 亿向量**（预览期的 40×）、每 bucket 最高 **20 万亿向量**、高频查询性能 **2–3×** 更快、成本相比替代方案降低最高 90%。同期 S3 其他更新：单对象最大 **50 TB**、批量作业 **10×** 更快、S3 Tables 分层、FSx for NetApp ONTAP 变为 S3 可访问、S3 Storage Lens 新增性能指标。AWS 称 S3 已存储**超过 500 万亿个对象、数百 EB 容量**。https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-s3-vectors-preview-native-support-storing-querying-vectors/ ｜ https://blocksandfiles.com/2025/12/03/aws-s3/
- **AWS SageMaker HyperPod managed tiered checkpointing**（2025 年下半年）：用 **CPU 内存**做高性能 checkpoint 存储，并**自动向相邻计算节点复制**以提升可靠性；周期性拷贝到 S3 做持久化；可为内存层与持久层分别配置频率与保留策略。论证与 Lockwood 的分级方案一致：仅依赖远端持久存储的传统 checkpoint 会在创建时产生计算开销，因为写 TB 级参数可能限流、消耗昂贵网络带宽并需要复杂的分布式编排；且**单个 writer 经网络无法用满网络吞吐**。→ 这是分级 checkpoint 在 2025 年被**产品化**的标志。https://aws.amazon.com/blogs/machine-learning/accelerate-your-model-training-with-managed-tiered-checkpointing-on-amazon-sagemaker-hyperpod/
- **Amazon FSx for Lustre Intelligent-Tiering**（2025）：起价低于 **$0.005/GB-月**，只对实际存储的数据计费，在 Frequent Access / Infrequent Access / Archive 之间自动分层；唯一完全弹性的 Lustre 文件系统；可选 SSD 读缓存，提供多 TB/s 吞吐与亚毫秒延迟访问热数据。

**标准（2025 年集中落地）**
- **UALink 200G 1.0**（2025-04）：加速器与交换机之间的低延迟高带宽互联，单 fabric 可扩展到 **1,024 个加速器**（NVLink 上限 576），每 lane 200 GT/s；每加速器分配一个端口，用 10 位唯一标识做精确路由。NVLink 5.0 单连接带宽为 UALink 1.0 的 3× 以上（2,538 vs 800 GB/s），但 UALink 支持近 2× 的最大集群规模且跨厂商。
- **Ultra Ethernet 1.0**（2025-06-11，UEC）：**560 余页**的完整以太网通信栈。标准物理层支持 QSFP-DD 与 OSFP 光模块；数据链路层改进含**原生 RDMA 支持**；新传输协议 **Ultra Ethernet Transport (UET)** 做智能流控；提供可观测性与自动化的开放 API。目标不只是速度，还包括可预测性能、扩展性与高效资源利用，同时保持硬件中立与免许可。https://ultraethernet.org/ultra-ethernet-consortium-uec-launches-specification-1-0-transforming-ethernet-for-ai-and-hpc-at-scale/
- **CXL 4.0**（2025-11-18，于 SC25 发布）：从 PCIe 6.x（64 GT/s）转向 **PCIe 7.0（128 GT/s）**，带宽翻倍，同时保持 CXL 3.x 引入的 256 字节 FLIT 格式；引入 **bundled ports**，把多个物理端口聚合为单一逻辑连接，提供 **1.5 TB/s** 带宽；目标是**多机架内存池化**——100+ TB 共享内存并跨机架保持缓存一致性（CXL Consortium 明确把生产部署目标定在 2026 年末–2027）。四天前（11-12）韩国 Panmnesia 宣布其 PCIe 6.0 / CXL 3.2 Fabric Switch 样片可得，是首个实现 CXL fabric **端口路由（PBR）** 的硅片。同期华为宣布将开源 **UB-Mesh** 协议（意在统一替代 PCIe、CXL、NVLink 与 TCP/IP）。https://www.hpcwire.com/off-the-wire/cxl-consortium-releases-the-compute-express-link-4-0-specification-increasing-speed-and-bandwidth/
- **HPE Discovery @ ORNL 公布（2025-10）**：Frontier 的后继系统，用 HPE GX5000 Cray 架构，下一代 AMD EPYC "Venice" + AMD Instinct MI430X。存储侧同时配置 **K3000（史上第一个工厂预制的 DAOS 存储系统）** 与 Lustre 型 **E2000**。HPE 指出 DAOS 系统在 IO500 上排名第一（Argonne Aurora）与第二（LRZ SuperMUC），两者合计的基准分数是后 30 个存储系统之和的四倍。DAOS 2.8 在 SC25 邻近的 DAOS User Group（11-16）上讨论。

---

## 4. 横向对比：2025 年的 KV cache 存储栈（同一路径的 5 个方案）

| | **Mooncake** | **NVIDIA Dynamo + NIXL** | **LMCache** | **IMPRESS** | **DiffKV** |
|---|---|---|---|---|---|
| 层次定位 | 完整服务架构（PD 分离 + 全局 KVCache 池 + 调度器） | 编排框架 + 传输库 | KV 缓存层（可插入 vLLM / Dynamo） | 多层前缀 KV 存储系统 | GPU 内存管理器 |
| 存储介质 | GPU 集群内闲置 CPU / DRAM / SSD / NIC | HBM → DRAM → 直连 SSD → 网络存储（1.0 仅到 CPU 内存） | GPU + CPU + SSD 混合 | CPU 内存 + 磁盘多层 | GPU 内存（含并行 compaction） |
| 传输 | Mooncake Transfer Engine（RDMA） | NIXL：RDMA/IB、RoCE(UCX)、TCP、**NVMe-oF**、**S3** | 经 NIXL | 未强调 | 不涉及 |
| 决策依据 | SLO 感知的全局调度 | KV-cache-aware routing + 全局 radix tree | 复用/驱逐/检索逻辑 | **token 重要性**（跨注意力头索引集合相似性） | key/value 差异 + token 重要性 + 头间稀疏模式 |
| 关键结果 | 有效请求容量 +59%~498%；生产 >1000 亿 token/日 | 厂商声明，无独立基线 | 生态位（被 Dynamo 采纳为 KV 缓存层） | TTFT 最高 -2.8× | 见论文 |
| 证据等级 | 同行评审 + 生产 + 开源 trace | 开源 + 厂商声明 | 开源 | 同行评审 | 同行评审 |
| 语义披露 | 一致性/驱逐/多租户披露有限 | 未披露 | 部分 | 部分 | GPU 内，不涉分布式一致性 |

**读表要点**：2025 年五个方案的**层次是互补的而非竞争的**——Mooncake 定架构、NIXL 定传输、LMCache 占缓存层、IMPRESS/DiffKV 提供"缓存哪一部分"的判据。到 2025 年底，事实上的组合是 **Mooncake 或 NIXL 做传输 + LMCache 做缓存层 + vLLM/SGLang 做引擎**。对存储工程师最关键的一列是"传输"：**NIXL 把 NVMe-oF 与 S3 列为一等后端，这是 KV cache 能落到你的存储系统上的技术前提。**

---

## 5. 工程与开源雷达（2025 年开源，2026 年状态已核对）

| 项目 | 2025 年状态 | 许可（已核实?） | 集成面 | 预估评估成本 | 2026 年后验 |
|---|---|---|---|---|---|
| **DeepSeek 3FS** — github.com/deepseek-ai/3FS | 2025-02-28 开源；生产使用自 2019 年起 | **MIT**（已核实） | FUSE 客户端 + native 客户端；CRAQ 强一致复制；无状态元数据服务（ZooKeeper/etcd） | 中高：需 RDMA 网络 + 多节点 NVMe | 仍是被广泛引用的参照实现；成为其他系统（如 RDMA-first 对象存储研究）的对比基线 |
| **Mooncake**（Transfer Engine + Store）— github.com/kvcache-ai/Mooncake | Store 于 2025-03-07 开源；Transfer Engine 自 2024-12 起被 vLLM 支持 | 未从仓库 LICENSE 核实 | vLLM v1 KV Connector、SGLang、TensorRT-LLM 均已集成；含 FAST'25 论文所用真实 trace | **低中**：trace 可直接用于离线评估，无需部署 | 2026-01 腾讯+NVIDIA 的 FlexKV 也接入其 Transfer Engine；生态持续扩大 |
| **ByteCheckpoint** — github.com/ByteDance-Seed/ByteCheckpoint | 2025-04 正式开源；2025-08 支持 MUSA 平台 | 未从仓库 LICENSE 核实 | `pip install bytecheckpoint`；多训练框架 Planner；多存储后端；含 checkpoint 合并/转换/修改与 metadata/tensor 文件检查工具 | **最低**：可 pip 安装，本报告中最便宜的验证入口 | 仍是 checkpoint resharding 的主要参考实现 |
| **NVIDIA NIXL / Dynamo** | GTC 2025（3 月）开源 | 开源（具体版本未核实） | 五后端：RDMA/IB、RoCE(UCX)、TCP、NVMe-oF、S3；`nixlbench` 基准工具；vLLM `--kv-transfer-config` | **低**：`nixlbench` 可直接对你的存储后端验证 GDS/S3 路径 | 2026 年成为 CMX/STX 体系的传输层；LMCache 与 Dynamo 1.0 深度集成 |
| **LMCache** | 2025-04 支持 Mooncake Store；2025-09 被 Dynamo 采纳为 KV 缓存层 | 开源 | 存储插件接口；支持 NIXL（含 GDS） | 低中：作为"你的存储做 KV 容量层"的第一个真实客户端 | 2026-03 与 Dynamo 1.0 完成集成；VAST 与 WEKA 已做生产规模验证 |
| **CHEOPS '25 I/O trace 与脚本** — github.com/stonet-research/cheops25-IO-characterization-of-LLM-model-kv-cache-offloading-nvme | 2025-04 随论文开源 | 未核实 | DeepSpeed 与 FlexGen 的块层 trace + 采集脚本 | **最低**：trace 直接可分析 | 未见后续更新；仍是唯一公开的此类 trace |
| **MLPerf Storage 基准套件** — github.com/mlcommons/storage | v2.0（2025-08），首次含 checkpoint 测试 | Apache | DLIO 驱动；ResNet-50 / 3D-UNet / CosmoFlow + checkpoint | 中：自测需搭客户端集群 | **2026 上半年无新一轮**；v2.0 数据仍是唯一跨厂商可比源 |

---

## 6. 视频精选

2025 年的会议录像多数已公开，但本报告不提供未经验证的时间戳。

1. **NSDI '25 — ByteCheckpoint: A Unified Checkpointing System for Large Foundation Model Development**，Borui Wan（香港大学）。**13 分钟**。**为什么值得专家的时间**：13 分钟讲清一个抽象（parallelism-agnostic 表示 + 加载时 resharding）和一组数字（stall 平均 -54.20×、保存最高 9.96×），信息密度极高，是本年度性价比最高的一段录像。https://www.youtube.com/watch?v=dPsejzDZv7o
2. **FAST '25 Keynote — Insights Gained from Delivering Two Generations of AI Supercomputers and Storage Solutions in IBM Cloud**，Dr. Seetharami Seelam（IBM Research，Distinguished Engineer）。2025-02-25，60 分钟。**为什么值得**：讲 IBM Cloud 中两代 **Vela** 云原生 AI 系统（IBM AI 事业的底座，支撑 Watsonx 与 RHEL AI 服务）在开发与运营中遭遇的扩展、性能与高可用挑战，涵盖计算、网络、存储，并分享用云原生平台管理这些系统**两年以上**的经验教训。这是全年**唯一**一份来自超大规模厂商的、系统性的 AI 超算存储运维经验公开分享。https://www.usenix.org/conference/fast25/presentation/seelam
3. **FAST '25 Machine Learning and Storage session 录像**：Mooncake、FusionANNS、IMPRESS、GPHash、GeminiFS 五篇均有公开录像。入口：https://www.usenix.org/conference/fast25/technical-sessions
4. **不推荐直接引用**：GTC 2025 的 Dynamo 发布 session 与各厂商在 SC25/OCP 2025 上的 CXL KV cache 演示——均为厂商声明，无方法学披露。

---

## 7. 趋势判断与反证

### 趋势 1：AI 存储的最佳实践在 2025 年第一次开源
**支持**：3FS（MIT，2/28）、Mooncake Store（3/7）、ByteCheckpoint（4 月）、Dynamo + NIXL（GTC 3 月）、Mooncake 的真实 trace（2/21）、CHEOPS 的块层 trace（4 月）。**六份都是被生产验证或可复现的材料，全部在 2025 年内公开。**
**反证与限制**：开源的是**实现**，不是**运营知识**。3FS 与 Mooncake 的生产数字全部是第一方、无独立复现；Mooncake 论文对一致性、驱逐语义、多租户隔离与故障恢复的披露有限；3FS 的元数据服务无状态化后的扩展上限未公开。**更重要的是，开源集中在中国厂商**——美国超大规模厂商（Meta、OpenAI、Anthropic）在 2025 年几乎没有存储侧披露，Microsoft 的披露走的是 Lockwood 的个人博客而非公司渠道。所谓"能读到前沿实现"其实只覆盖了前沿的一部分。

### 趋势 2："AI 训练需要并行文件系统"被系统性质疑，但商业投入方向相反
**支持**：Lockwood 的生命周期分析（Llama-3 405B 全部输入 60 TB、每 GPU 全程 3.75 GB、checkpoint 大小不随作业规模变化、共享存储只需约 1 GB/s）；Cloudscape 的实证（云上 S3 68% vs 文件系统服务 4%）；3FS 主动放弃读缓存；CHEOPS 的 trace 显示模型 offload 打不满 NVMe 且请求以 128 KiB 为主（不是小随机 I/O）。
**反证与限制**：
- **MLPerf Storage v2.0 的构成本身是最强反证**：26 个组织的 200+ 提交里有 **16 个本地部署共享存储方案、13 个软件定义方案、12 个块系统**，只有 **2 个对象存储**。即使承认对象存储在架构上"足够"，市场在 2025 年买的仍然主要是共享文件/块系统。
- **云厂商的判断相反**：Google 在 2025 年 4 月推出 Rapid Storage（6 TB/s、亚毫秒）、7 月把 Managed Lustre（DDN EXAScaler）推向 GA 并提供四个性能层，AWS 持续投入 FSx for Lustre 并新增 Intelligent-Tiering。**如果并行文件系统对 AI 训练不必要，这些投资很难解释。**
- **Argonne Aurora 的 DAOS 成绩**（128 个服务器达到近 1 TB/s 写、405B checkpoint <10 秒）说明高性能共享存储在 checkpoint 上确实有效，问题只是"是否必要"而非"是否有效"。
- Lockwood 自己给出的限制也必须记住：他**不主张**已有 Slurm/PFS 工作流的场所丢掉并行文件系统；他描述的架构对"习惯在容器里开发并提交 pod"的用户友好，对"习惯 vim + Slurm"的用户不友好。**这是一个用户体验与迁移成本问题，不是纯技术问题。**

### 趋势 3：Checkpoint 从"更快的写"转向"分级 + 与并行策略解耦"，并在同年被产品化
**支持**：ByteCheckpoint 的 parallelism-agnostic 表示 + 加载时 resharding（抽象层）；Lockwood 的三级异步方案（GPU→DRAM→邻居 SSD→共享存储，附带每级的时间预算与频率）；AWS SageMaker HyperPod managed tiered checkpointing（产品化，CPU 内存 + 自动向相邻节点复制 + 可配置 S3 持久化频率与保留策略）；MLPerf Storage v2.0 首次把 checkpoint 纳入基准（说明这已是标准实践）。
**反证与限制**：分级方案把一个存储问题变成了一个**分布式协议问题**——邻居节点的选择、副本失效的检测、catastrophic failure 下最多丢失多少训练进度（Lockwood 的例子里是最多 10 分钟），这些都是新的正确性风险，而 2025 年的公开材料**几乎没有对分级 checkpoint 做故障注入评测**。ByteCheckpoint 提供了文件检查工具，但跨并行配置 resharding 后的数值等价性如何验证，摘要层未展开。

### 趋势 4：缓存与分层的决策依据从"访问模式"转向"模型语义"
**支持**：IMPRESS 用**跨注意力头的重要 token 索引集合相似性**来决定加载哪些前缀 KV（TTFT 最高 -2.8×）；DiffKV 依据 key/value 的不同影响、token 重要性差异、头间动态稀疏模式做差异化管理；Dynamo 的 KV-cache-aware routing 用缓存命中率而非负载做路由。
**反证与限制**：这些方案都要求**存储层能接收应用侧的语义信号**，而 2025 年没有任何标准接口承载这种信号——每一个都是应用与存储的私有协同设计。这是 2026 年出现 grouped I/O API（AITURBO）与 S3 descriptor 扩展（ObjectCache）的直接动因。**换句话说：2025 年证明了语义信号有价值，2026 年才开始解决怎么传递它。**

### 趋势 5：标准层在 2025 年集中落地，但存储协议本身没变
**支持**：UALink 200G 1.0（4 月，1,024 加速器）、Ultra Ethernet 1.0（6 月，560+ 页，原生 RDMA + UET）、CXL 4.0（11 月于 SC25，128 GT/s、bundled ports 1.5 TB/s、多机架内存池化）。加上 Panmnesia 首个 CXL fabric PBR 交换机样片（11 月）、华为宣布开源 UB-Mesh。
**反证与限制**：**三个标准全部在网络与内存侧，没有一个改变存储访问协议。** NVMe、S3、POSIX 在 2025 年都没有为 AI 负载增加任何原语。这直接解释了为什么 2026 年的创新集中在协议扩展（ObjectCache 在 S3 GET 上加交付顺序 descriptor、AITURBO 的 grouped I/O API）——**2025 年把网络与内存的带宽问题解决到一定程度后，接口表达能力成了新瓶颈**。此外 CXL Consortium 自己把多机架内存池化的生产部署目标定在 2026 年末–2027，意味着 CXL 4.0 在 2025 年是**规范事件而非部署事件**。

### 趋势 6：定量证据开始出现，且一致指向"实际需求低于厂商推荐"
**支持**：三份独立材料——①Lockwood 的分级 checkpoint 推算（共享存储约 1 GB/s）；②Lockwood/VAST 在 SC'25 PDSW 报告的 **40 个生产 LLM 训练作业、8.5 万次 checkpoint** 实测（即使万亿参数模型也只需几百 GB/s 全局 checkpoint 带宽、通常远低于 1 TB/s；并给出需求侧容量模型以避免超配 I/O、把更多资源（功耗、冷却）留给计算）；③CHEOPS 的块层 trace（128 KiB 主导、模型 offload 打不满 NVMe、KV offload 读写带宽差三个数量级）。加上 VAST 早前的量化（万亿参数模型在 3,072 GPU 上用 273 GB/s 文件系统达到 99.7% forward progress）。
**反证与限制**：
- ①与②的作者是同一人，且②的发布渠道是 VAST（存储厂商）——**"存储需求比你想的低"这个结论来自一家存储厂商，需要意识到其商业动机可能是"卖架构效率而非卖峰值带宽"**，这不否证数据，但影响解读。
- ③的规模限制严重（单机、OPT-6.7B、仿真 SSD、未覆盖 vLLM/SGLang）。
- MLPerf Storage v2.0 的方法学争议（Western Digital 白皮书指出 client/vGPU 归一化可能误导）说明**即使有跨厂商基准，也不能直接读排名**。
- **最关键的空白**：没有任何 2025 年的公开材料比较"投资去重/分级"与"投资带宽"的边际收益。这个空白在 2026 年仍然存在。

---

## 8. 建议实验与关注名单

### 实验一（优先做，成本最低）：用 ByteCheckpoint 与 Mooncake trace 做零部署评估
- **动机**：这两个是本报告中**唯一可以不搭集群就开始评估**的材料——ByteCheckpoint 可 `pip install`，Mooncake 发布了 FAST'25 论文所用的真实 trace。
- **做法**：①用 ByteCheckpoint 在你现有的存储后端上跑保存/加载，测 checkpoint stall（对齐其报告的相对 MCP 12.13–161.50× 的口径）；②用 Mooncake trace 在离线模拟里评估你的存储系统作为 KVCache 池的命中率与容量需求。
- **指标**：checkpoint stall 时长分布、保存/加载端到端时间、**跨并行配置 resharding 是否成功且数值等价**（这一条论文没充分展开，值得自己验证）；KVCache 侧的命中率、驱逐率、容量占用。
- **成本**：1–2 人周。

### 实验二：用 `nixlbench` 验证你的存储在 GDS 与 S3 路径上的实际能力
- **动机**：NIXL 的五后端（RDMA/IB、RoCE、TCP、NVMe-oF、S3）定义了 KV cache 能落到你的存储上的技术前提，而 `nixlbench` 是官方自带的验证工具。
- **做法**：先跑 GDS 后端确认 GPU 内存与你的存储之间的原始 I/O 路径正确且性能符合预期；再跑 S3 后端。
- **指标**：不同 block size 下的带宽（**重点看 128 KiB 附近**，因为 CHEOPS 的 trace 显示这是实际主导的请求大小）、单 GPU 能否被打满、读写不对称程度（对比 CHEOPS 的 2.0 GiB/s 读 vs 11.0 MiB/s 写）。
- **故障用例**：RDMA 目标不可达时的回退；S3 后端在部分对象缺失时的行为。
- **决策价值**：这是"你的存储能不能进 KV cache 栈"的**准入测试**，成本低于任何架构改造。

### 实验三：验证"读缓存对你的 AI 负载是否有价值"（3FS 的核心假设）
- **动机**：3FS 主动放弃读缓存，理由是单 epoch 训练下每条数据只读一次；FalconFS（2026）用同样的逻辑放弃了客户端元数据缓存。**这个假设在你自己的负载上是否成立，是可以廉价验证的。**
- **做法**：在真实训练作业上统计客户端与服务端读缓存的命中率，并分别测量缓存占用的内存。
- **指标**：读缓存命中率；缓存占用的客户端内存；**在固定客户端内存预算下可运行的 dataloader worker 数**（传统上从不测量的指标）。
- **决策价值**：如果命中率接近零，那么缓存占用的内存是纯损失，且可以直接换成更多的 dataloader worker 或更大的本地 staging 空间。

### 实验四：复核你在 2025 年的 I/O 配置决策
- **动机**：§7 趋势 6 的三份定量证据一致表明实际需求低于厂商推荐。如果你在 2025 年做过 AI 存储采购，这是一次低成本的复盘机会。
- **做法**：用 Lockwood/VAST 的需求侧模型（LLM 规模 + checkpoint 间隔 → 所需全局带宽）反算你的实际需求，与你实际配置的带宽对比；同时用你自己的 checkpoint 日志统计实际使用的峰值带宽与持续带宽。
- **指标**：实际峰值/持续 checkpoint 带宽 vs 配置带宽；forward progress 百分比（对齐 VAST 的 99.7% 口径）；空闲带宽对应的资本与功耗成本。
- **决策价值**：如果差距显著，下一轮采购可以把预算从存储带宽转向容量或计算。**注意**：做这个复盘时要意识到需求侧模型来自存储厂商，结论要用自己的日志验证而非直接采信。

### 关注名单（2025 年识别，2026 年状态已核对）

**团队/机构**：
- **Moonshot AI + 清华（Mingxing Zhang、Yongwei Wu、Weimin Zheng）**——Mooncake 及其开源生态。2026 年后验：Transfer Engine 已被三大推理引擎接受。
- **DeepSeek**——3FS。2026 年后验：无重大后续开源，但 3FS 仍是被广泛引用的参照实现。
- **字节跳动（Seed + 基础架构）+ 香港大学 Chuan Wu**——ByteCheckpoint、MegaScale 系列。2026 年后验：**这是本报告最应该重点跟踪的团队**——2026 年他们拿下 OSDI 最佳论文，并持续输出 MegaScale-Data / MegaScale-MoE / MegaScale-Omni、DisCoGC、SDCHunter、AEGIS。
- **上海交大 IPADS（Haibo Chen、Xingda Wei、Rong Chen、Mingkai Dong）**——PhOS、DiffKV、Liquid-State Drive。2026 年后验：FalconFS、AITURBO、SwitchFS、KUNSERVE，四会同时高产。
- **浙江大学 Shuibing He 组**——IMPRESS、LeapGNN。
- **厦门大学 NICE Lab（Yiming Zhang）+ 华为理论实验室**——GeminiFS、AtomicDisk。2026 年后验：RosenBridge、MlsDisk、SkySync/ParaSync。
- **华为（云 + 存储 + 理论实验室）**——FlacIO、IMPRESS、GeminiFS、FusionANNS、HiDPU。2026 年后验：FalconFS、AITURBO、MAIO/PPC、TapeOBS。
- **VU Amsterdam（Animesh Trivedi）+ IBM Research Zurich（Radu Stoica、Christian Pinto）**——CHEOPS '25 I/O 刻画，欧洲在本方向的主要产出，且开源 trace。
- **Glenn K. Lockwood**——2025 年质量最高的独立分析来源（时任 Microsoft，2026 年已在 VAST）。**注意其雇主变化对解读的影响**：2025 年 2 月的文章立场是"对象存储足够"（Microsoft 视角），2025 年 11 月与 VAST 联署的 checkpoint 带宽研究立场是"架构效率优于峰值带宽"（存储厂商视角）。两者不矛盾，但需要意识到。
- **MLCommons MLPerf Storage 工作组（Curtis Anderson、Oana Balmau）**——唯一的跨厂商可比基准的制定者。Balmau 同时是研究者（2026 年 MinatoLoader 作者），值得同时跟踪其研究与基准演进。
- **UW-Madison（Andrea & Remzi Arpaci-Dusseau）**——Cloudscape、Ananke。2026 年后验：MOST、HARE。

**项目**：3FS、Mooncake（Transfer Engine + Store + traces）、ByteCheckpoint、NIXL（含 `nixlbench`）、Dynamo、LMCache、llm-d、MLPerf Storage 套件、CHEOPS '25 traces。

**2025 年埋下、2026 年兑现的线（供交叉阅读）**：
| 2025 年的种子 | 2026 年的兑现 |
|---|---|
| PhOS（SOSP '25，OS 级 GPU C/R） | GCR（FAST '26，清华）以 PhOS 为 SOTA 基线，checkpoint 延迟再降 63.6% |
| Google Rapid Storage（6 TB/s） | Rapid Bucket GA（>15 TB/s、20M ops/s） |
| Dynamo/NIXL 开源（五后端） | CMX/STX + DOCA Memos（把 KV cache 层硬件化与体系化） |
| 3FS 放弃读缓存 | FalconFS 放弃客户端元数据缓存（同一逻辑、不同路径） |
| IMPRESS/DiffKV 的语义信号 | AITURBO grouped I/O API、ObjectCache 的 S3 descriptor（解决怎么传递信号） |
| 标准只动网络与内存 | 存储协议开始被扩展 |
| MLPerf Storage v2.0 首次含 checkpoint | **2026 上半年无新一轮**——v2.0 仍是唯一跨厂商可比源 |

---

## 9. 覆盖与证据缺口

### 报告性质带来的偏置（最重要的一条）
本报告是**回溯撰写的**（2026-07-29，晚于窗口约七个月）。这意味着：
- **入选偏向"后来被验证重要"的工作**。2025 年当时受关注但没有下文的方向（例如若干 CXL 内存池化的 KV cache 演示、部分 PIM/存内计算方案）在本报告中被系统性低估。
- **部分 2025 年的一手链接已被更新版本覆盖**（如 Google Rapid Storage 的官方页面已并入 2026 年的 Cloud Storage Rapid 叙事）。
- 若你需要严格的同期视角，本报告的 §3 排序不适用。

### 无法验证或未验证的内容
1. **3FS 的 6.6 TiB/s**：第一方压测数字。与 Ceph 1.1 TB/s 的对比是**不同硬件、不同时间点的跨系统比较**（180 vs 68 节点、16 vs 10 块 SSD/节点、200G vs 100G 网络），**不构成受控实验**，不可用于"快 6 倍"这类论断。
2. **Mooncake 的生产数字**（数千节点、>1000 亿 token/日、+59%~498% 有效请求容量、Kimi +115%/+107%）：均为第一方，无独立复现。一致性、驱逐语义、多租户隔离、故障恢复未充分披露。
3. **NVIDIA Dynamo 的性能声明**：无独立基线。Dynamo 1.0 实际只支持卸载到 CPU 内存，SSD 与网络对象存储是后续版本——2025 年公开材料中常被混淆表述为"已支持四层"。
4. **MLPerf Storage v2.0**：虽是跨厂商可比基准，但其 client/vGPU 归一化指标存在**公开的方法学争议**（Western Digital 白皮书指出可能误导且难以解读）。读结果必须同时看客户端数、节点数、功耗与机架单元。
5. **Google / AWS 的云产品数字**：全部第一方声明。Google Rapid Storage 的"20× 更快数据访问""比其他厂商低 5× 随机读写延迟"未给出对照系统与测试方法。
6. **Lockwood/VAST 的 8.5 万 checkpoint 研究**：数据规模与代表性优秀（40 个生产作业），但发布渠道是存储厂商博客与 SC'25 PDSW **WIP**（work-in-progress，非完整同行评审论文），且作者已在该厂商任职。数据可信，解读需注意商业动机。
7. **CHEOPS '25 的四条结论**：方法透明、trace 开源，但规模限制严重（单机、OPT-6.7B、**NVMeVirt 仿真 SSD**、只覆盖 DeepSpeed 与 FlexGen 而非 2025 年主流的 vLLM/SGLang）。**不可外推到集群规模或前沿模型。**
8. **开源许可**：仅 3FS 的 MIT 许可从公开报道中确认；Mooncake、ByteCheckpoint、NIXL、Xerxes 等的具体许可证**未从仓库 LICENSE 文件核实**，使用前需自行确认。
9. **IO500 的 2025 两轮榜单**：本次检索未取得 ISC'25 与 SC'25 榜的完整细节（仅通过 HPE 的二手陈述得知 Aurora DAOS 与 SuperMUC DAOS 分列一二）。**这是本报告的一个明确检索缺口。**

### 缺乏证据的主题（不是没找到，是公开证据本身不足）
- **KV cache 层的一致性、多租户隔离、故障恢复与经济学**：2025 年几乎所有 KV 存储工作只评测性能。**这个缺口一年后（2026）依然存在**，说明它是结构性的，不是暂时的。
- **"投资去重/分级"vs"投资带宽"的边际收益比较**：跨 2025 与 2026 两年均无任何公开数据。
- **分级 checkpoint 的故障注入评测**：Lockwood 的方案与 AWS HyperPod 的产品都描述了正常路径，但邻居副本失效检测、catastrophic failure 下的实际进度损失、resharding 后的数值等价性，都没有公开的评测。
- **美国超大规模厂商的存储侧工程披露**：Meta、OpenAI、Anthropic 在 2025 年全年基本沉默。唯一的系统性分享来自 IBM（FAST '25 keynote，Vela 两代云原生 AI 系统的运维经验）与 Microsoft（经个人博客）。
- **欧洲在 AI 存储核心数据路径上的产出**：CHEOPS '25 是主要且几乎唯一的产出。**这与 2026 年的观察一致**，是一个跨两年稳定的结构性分布差异，不是单年检索遗漏。

### 地域与来源类型分布（长窗口审计）

判定依据为**作者当前的机构归属**，不依据姓名、族裔或国籍；跨区域合作单独标注。

| 区域 | 入选/候选项与代表 | 主要披露渠道 |
|---|---|---|
| **中国 / 东亚** | 在数量与技术深度上主导，且**四份重要开源全部来自中国厂商**。中国大陆/香港：3FS（DeepSeek）、Mooncake（Moonshot AI + 清华，FAST '25 最佳论文）、ByteCheckpoint（字节 + 港大）、IMPRESS（浙大 + 华为云）、GeminiFS（厦大 + 华为）、FusionANNS（HUST + 华为）、FlacIO（华为）、HiDPU（山大 + 天大 + 华为 + CUHK）、LeapGNN（浙大）、3L-Cache（北工大 + MSRA）、GogetaFS（哈工大深圳 + 阿里）、NCBlob（HUST）、Liquid-State Drive（上交）、AtomicDisk（蚂蚁 + 厦大 NICE Lab + 上交）、Maat（电子科大 + 港理工）、Archer（华师大 + 中科大）、MedFS（南理工 + OPPO）、Oasis（CUHK + UT Dallas）、GPHash（HUST）、PIMLex（南开）、AegonKV（HUST）；SOSP '25：PhOS / DiffKV（上交）、Aegaeon（北大 + 阿里）；ATC '25：Weaver（清华）。韩国：DJFS / D2FS / OPIMQ（KAIST + Samsung + UW-Madison）、ScaleLFS（首尔大 + 中央大）、AWUPF（成均馆 + Virginia Tech）。 | 顶会论文 + **开源仓库（本年度最大变化）** |
| **北美** | Lockwood 的生命周期分析（Microsoft）、Cloudscape（UW-Madison + NetApp）、Ananke（MSR + UW-Madison）、On Scalable Integrity Checking（UW-Madison）、Jenga（UC Berkeley + 多机构）、cache_ext（Columbia + IBM Research）、VectorCDC（Waterloo）、Selective On-Device Execution（UNIST，亚洲）、HaSiS（CUHK + 山大 + Indiana + RPI/ScaleFlux）、PolyStore（Rutgers + EPFL + Samsung）、NVIDIA Dynamo/NIXL/SCADA、Google Rapid Storage + Managed Lustre GA、AWS S3 Vectors + HyperPod tiered checkpointing + FSx Lustre Intelligent-Tiering、Argonne Aurora DAOS（MLPerf）、MLCommons MLPerf Storage v2.0、IBM FAST '25 keynote（Vela）、HPE Discovery/K3000 DAOS。 | **云产品发布 + 基准组织 + 少量工程论文**；超大规模厂商的 AI 存储细节主要通过产品发布披露 |
| **欧洲** | **产出显著少于中美，且集中在刻画、内核与标准而非核心数据路径**：CHEOPS '25 I/O 刻画（VU Amsterdam + IBM Research Zurich，**含开源 trace**，本年度欧洲最重要产出）、EuroSys '25 主办（Rotterdam）、DNA storage motif 生成（Imperial College London）、Silhouette（FSU + Toronto，非欧）、PolyStore 中的 EPFL、Ultra Ethernet 1.0（Eviden/Atos 为创始成员之一）、CXL 4.0（多方，含欧洲成员）、EuroHPC JUPITER / LUMI / Leonardo / MareNostrum5 / Deucalion 的技术指南更新、Technion（Gala Yadgar 任 FAST '25 两个 session 主席）。 | 系统会场 + 内核/标准社区 + 研究实验室 |
| **跨区域合作** | Mooncake（Moonshot AI + 清华）、ByteCheckpoint（字节 + 港大）、IMPRESS（浙大 + 华为云）、GeminiFS（厦大 + 华为）、CHEOPS（荷兰 + 瑞士 IBM）、PolyStore（美 + 瑞士 + 韩企）、AWUPF（韩 + 美）、Jenga（美 + 中）、HaSiS（港 + 陆 + 美）。 | — |
| **其他** | MBZUAI（阿联酋）、NUS、HKUST、Nanyang Technological University（Gelei Deng）。 | — |

**关于欧洲覆盖的说明**：本次针对欧洲专门检索了 EuroSys '25 及其 CHEOPS workshop、ISC 2025、DAOS / Lustre / JUPITER / LUMI / EuroHPC / CERN / ECMWF 相关渠道。结论与 2026 年报一致：**欧洲机构在 AI 存储核心数据路径上的第一作者产出确实少于中国与北美**，其强项在于 I/O 刻画、内核栈、持久内存、可靠性与传统 HPC I/O，以及**会议治理与标准制定**（EuroSys 主办、Ultra Ethernet 创始成员、Linux 内核上游）。这是一个跨两年稳定的分布差异，不通过降低技术门槛来凑配额。

**来源类型分布**：同行评审论文约 50%（FAST '25 / NSDI '25 / SOSP '25 / ATC '25 / CHEOPS '25）；开源实现 + 第一方生产结果约 20%（3FS、Mooncake、ByteCheckpoint、Dynamo/NIXL）；云产品与厂商声明约 15%（NVIDIA、Google、AWS）；独立基准约 8%（MLPerf Storage v2.0）；独立专家分析约 7%（Lockwood）。

**与 2026 年报的对比**：2025 年的**开源实现占比显著更高**（20% vs 2026 年几乎为零的同类新增），而 2026 年的**厂商声明占比更高**（GTC 的 CMX/STX + Cloud Next 均在窗口内，而 MLPerf 无新一轮）。**如果你只能读一年的材料，2025 年的可读代码更多，2026 年的架构信号更强。**
