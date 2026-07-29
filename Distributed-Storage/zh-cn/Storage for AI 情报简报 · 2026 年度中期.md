# Storage for AI 情报简报 · 2026 年度中期（YTD）

---

## 1. 检索范围与存储工程师结论

**报告窗口**：2026-01-01 00:00 — 2026-07-29 23:59（Asia/Shanghai, UTC+8）。
**周期状态**：**年度报告不完整（year-to-date）**。窗口覆盖 2026 年前 7 个月，SC'26（11 月）、FMS 2026（8 月）、SNIA SDC 2026（9 月 28–30）、MLPerf Storage 下一轮（预计 8 月）均在窗口外。

**已覆盖并入选的渠道**：FAST '26（2/24–26, Santa Clara，含 Spring 双截止期）、NSDI '26（5/4–6, Renton）、EuroSys '26（4/27–30, Edinburgh，含 Fall/Spring 两批）、OSDI '26（7/13–15, Seattle）、ISC High Performance 2026（6/22–26, Hamburg，含 IO500 ISC26 榜与 Lustre/DAOS BOF）、NVIDIA GTC 2026（3/16–19）、Google Cloud Next '26（4 月）、AWS 存储发布流、arXiv 2026 H1 预印本、FalconFS/Xerxes 等开源仓库、Glenn Lockwood 与 backend.ai 等专家长文。

**已搜索但未产出入选项的渠道**（见 §9）：MLCommons MLPerf Storage（本窗口内无新一轮）、Meta/OpenAI/Anthropic 的存储侧工程博客、CERN/ECMWF/EuroHPC 的 2026 年 AI I/O 论文、SNIA 正式规范发布。

### 给分布式存储工程师的 5 条结论（以云存储→AI 存储的变化为框架）

1. **"KV cache 是引擎内部的数据结构"这个前提在 2026 年上半年正式终结。** NVIDIA 在 GTC 2026 用一个参考架构（BlueField-4 STX）+ 一个产品层（CMX）把 KV cache 定义成独立的"AI-native 数据类"，并给出 G1（HBM）/G2（DRAM）/G3（Pod 内 NVMe）/G3.5（DPU 终结的上下文层）/G4（外部存储）的分层模型，配套 DOCA Memos 作为存储厂商接入 G3.5 的开放接口。对存储工程师的含义：你的对象存储/文件系统正在被要求承担一个**新的一等工作负载**——极高写入吞吐、极短生命周期、**可重算因此可放弃持久性**、但读路径对尾延迟极度敏感。这不是"再加一个用例"，这是一个与传统企业数据在耐久性/一致性/成本模型上完全不同的数据类。

2. **传输顺序必须等于计算顺序，"总带宽"作为验收指标已经不够了。** 2026 年上半年三个独立工作（Strata/OSDI '26、ObjectCache/arXiv、Tutti/arXiv）收敛到同一机制：把 KV cache 的检索改成 **layer-major、按层交付**，让每一层的传输窗口对齐该层的 GPU 计算时间。ObjectCache 甚至把带宽分配从"按字节比例"改成"按每请求的 per-layer stall target"。这是一个**从局部性猜测（预取）转向确定性截止时间（deadline scheduling）**的范式转变。传统存储系统里"QoS = IOPS/带宽上下限"的抽象在这里不够用，需要的是可表达"第 N 层必须在 T 毫秒内到位"的接口。

3. **"客户端元数据缓存加速元数据操作"这条分布式文件系统的老共识，在 AI 数据集上被生产环境数据推翻。** FalconFS（NSDI '26，华为 + 上交，10,000 NPU 生产运行一年，已开源）的核心论断是：在深度学习流水线里客户端缓存**既无效又浪费宝贵内存**，因此采用 **stateless client**，把路径解析完全放回服务端（混合元数据索引 + 惰性命名空间复制 + 并发请求合并）。它能这么做的前提是一个 AI 特有的负载性质：**数据集目录极大**（目录条目数远超元数据节点数），使 filename hashing 天然负载均衡。这是本窗口内最值得直接搬进现有 DFS 的设计。

4. **Checkpoint 的主战场从"写得更快"转到"少写"和"复用已有副本"。** AdaCheck 用张量冗余把 checkpoint 缩小 6–896×；Checkmate 直接复用网络里已经为梯度同步复制的数据，把 checkpoint 挪出正常路径；GCR 用控制/数据分离 + CPU 影子执行做指令级增量识别；Google Cloud 的 ingest-on-write 让写入时即入缓存以加速恢复。**没有一个赢家是靠更快的介质拿到的。** 如果你在评估"上更快的 NVMe 池来解 checkpoint"，本窗口的证据方向与之相反。

5. **存储侧算力（storage-tier CPU / DPU）从"卸载压缩加密"扩展到"承担应用语义"，但生产界最有价值的经验是"最小侵入"。** OSDI '26 最佳论文（字节跳动 Seed + 中科大）把多模态数据的 transformation 卸载到存储层 CPU，并明确论证这些优化**不是系统特定的**、可同时用于遗留和现代存储系统——这句话是整个窗口里对存储工程师最重要的一句：真正落地的收益来自识别架构级错配（跨 DC 评估流量、启动期 checkpoint 加载争用、多模态转换的 CPU 瓶颈），而不是重写存储栈。

---

## 2. 工作负载变化地图（2026 H1）

| 路径 | 本窗口内确认的变化 | 关键证据 |
|---|---|---|
| **训练输入** | 瓶颈从"聚合带宽不足"转向三处：①**跨 DC 流量**（用远端 checkpoint 做在训评估）②**启动期 I/O 争用**（checkpoint 加载与数据加载互相踩）③**多模态转换的 CPU 饱和**（不是 I/O 饱和）。元数据侧确认小文件+巨目录并发是主约束，且客户端缓存是负资产。 | OSDI '26 ByteDance/USTC；NSDI '26 FalconFS；EuroSys '26 MegaScale-Data / MinatoLoader；FAST '26 Seneca |
| **Checkpointing** | 从"周期性大写"变为"可压缩的冗余状态管理 + 可复用的网络副本"。同时 GPU 级系统 C/R（不只是模型状态）成为独立方向，用于弹性伸缩与任务切换。 | FAST '26 AdaCheck / GCR；NSDI '26 Checkmate、Checkpoint Lite；Google Cloud Next '26 |
| **模型分发/冷启动** | 两条并行：①**存储侧去重/压缩**（模型族内张量级去重 + XOR delta）②**加载路径重构**（可编程 page cache、镜像预载、容器启动 FS）。冷启动被当作 serverless SLO 问题而非容量问题。 | NSDI '26 ZipLLM / HydraServe；FAST '26 MAIO(PPC) / CoFS / ThinkAhead |
| **推理状态（KV cache）** | 最剧烈的变化区。①硬件参考架构化（STX/CMX + BlueField-4）②对象存储成为合法容量层（S3 协议被扩展）③layer-major 按层交付成为共识机制④跨模型/跨位置/跨区域的 KV 复用成为独立研究线。 | GTC 2026；OSDI '26 Strata / DirectKV / ECHO；arXiv ObjectCache / Tutti；FAST '26 Bidaw / CacheSlide / SolidAttention；NSDI '26 SYMPHONY / DroidSpeak |
| **Agent / RAG 数据路径** | 出现"跨区域语义知识缓存"这一新层次（不是精确匹配缓存），以及 agent 对并行文件系统的**破坏性负载**这一运维现象。 | NSDI '26 Cortex；ISC'26 Lustre BOF 轶事（Lockwood 记录） |
| **通用云存储（可迁移机制）** | 超大规模生产系统披露密集：Apple 百 EB 级对象存储、阿里云本地盘演进、字节 discard-based GC（TCO -20%）、CMU+Google+Microsoft 的 exascale TCO 供给模型。 | FAST '26 ACOS / "Here, There and Everywhere" / DisCoGC；EuroSys '26 TCO-driven Storage Provisioning |

---

## 3. 深度必读（按技术价值排序）

### 3.1 Teaching the Old Dog New Tricks: Building Efficient Data Pipelines for Large-Scale LLM Pre-Training

- **作者/单位**：Luofan Chen、Chenhan Wang（中国科大 + 字节跳动 Seed），Weidong Zhang、Jinxin Chi、Cheng Chen 等（ByteDance Seed），Wencong Xiao（ByteDance），Kang Chen（清华），Cheng Li（中国科大 + 合肥综合性国家科学中心人工智能研究院）等 20 人
- **时间/会场**：OSDI '26（2026-07-13～15，Seattle），**Operational Systems track，Jay Lepreau Best Paper Award**（136 篇录用中 3 篇）
- **类型/证据级别**：同行评审 + 第一方生产结果（peer-reviewed + first-party production）
- **链接**：https://www.usenix.org/conference/osdi26/presentation/chen-luofan ｜ 获奖说明（中文）：http://news.ustc.edu.cn/info/1055/95731.htm
- **一句话判断**：本窗口最重要的一篇——它不是发明新存储系统，而是用千卡级生产 trace 证明了三个被低估的架构级错配，并给出可移植到现有存储系统的三个修复。**评分 24/25**（架构5/证据5/存储相关性5/AI特异性5/生产价值3/新颖性1）
- **问题与负载特征**：千 GPU 级预训练作业的数据流水线。生产 trace 的量化分析暴露三个此前**未被充分报道**的瓶颈：
  1. **跨数据中心流量**：在训模型评估需要读取远端 checkpoint，成为主要延迟来源；
  2. **启动期 checkpoint 加载 I/O 争用**，延迟作业初始化；
  3. **加载期数据转换**对多模态模型是显著的、**CPU 密集型**（而非 I/O 密集型）瓶颈。
- **架构与端到端数据路径**：三个优化——①**基于全局命名空间的预测式 checkpoint 复制**（把评估要读的 checkpoint 提前复制到评估侧 DC，消除跨 DC 读）②**主动热文件复制**（缓解启动期争用）③**把数据转换卸载到存储层 CPU 资源**（把转换从计算节点 CPU 移到存储节点 CPU，缩短热路径并释放 dataloader worker）。
- **语义与运维**：论文明确主张这些优化**不是系统特定的**，可同时应用于遗留与现代存储系统，是"最小工程侵入的高回报升级路径"。一致性/故障语义在摘要层未展开（**待读正文核实**：预测式复制的失效回退、热文件副本的一致性窗口）。
- **实验与结果**：每次评估浪费的 GPU 小时 **16,800 → 4,000**；每次训练启动的 checkpoint 加载时间 **-40.8%**；由 dataloading 导致的训练停顿 **-63.2%**。
- **相对传统云存储真正新的是什么**：**不是**复制、**不是**热点副本、**不是**近数据计算——这三样都是老技术。真正新的是**触发条件与放置目标**：复制的对象是"下一次评估将要读的 checkpoint"（由训练调度器的语义决定，不是由访问历史决定），这是把**应用调度信息作为存储放置输入**，传统 cloud storage 的 tiering/prefetch 完全拿不到这个信号。
- **可复用设计与未解问题**：①在你的 DFS/对象存储里开一个"调度器提示"通道（作业将读哪些 key、何时读），比任何访问预测模型都准；②统计你的存储节点 CPU 空闲率——本文的转换卸载成立的前提是存储层有可用 CPU，如果你的存储节点已被 EC/压缩打满则不成立；③未解：跨 DC 预测复制在多租户下如何做配额与公平性；转换卸载后的故障域扩大（存储节点崩溃现在会杀掉数据转换）如何隔离。

---

### 3.2 FalconFS: Distributed File System for Large-Scale Deep Learning Pipeline

- **作者/单位**：Jingwei Xu（上海交大 + 华为），Junbin Kang（华为），Mingkai Dong（上海交大），Mingyu Liu / Lu Zhang / Shaohong Guo / Ziyan Qiu / Anqi Yu / Tianhong Ding / Xinwei Hu（华为），Mingzhen You / Ziyi Tian（上海交大），Haibo Chen（上海交大 + 华为）
- **时间/会场**：NSDI '26（2026-05-04～06，Renton；Spring cycle 录用于 2025-07），预印本 arXiv 2507.10367（v4 更新于 2026-01-08）
- **类型/证据级别**：同行评审 + 第一方生产结果 + **已开源**
- **链接**：论文 https://www.usenix.org/conference/nsdi26/presentation/xu ｜ PDF https://www.usenix.org/system/files/nsdi26-xu.pdf ｜ arXiv https://arxiv.org/abs/2507.10367 ｜ 代码 https://github.com/falcon-infra/falconfs
- **一句话判断**：唯一一篇在本窗口内**明确推翻一条分布式文件系统长期共识**并有生产背书的论文，且代码可读。**评分 23/25**（架构5/证据5/存储相关性5/AI特异性4/生产价值3/新颖性1）
- **问题与负载特征**：自动驾驶深度学习流水线。海量小文件（评测用 112 KiB × 1000 万文件 / 100 万目录）、极高元数据操作率、客户端内存被训练进程争抢。
- **架构与数据路径**：**stateless client** ——放弃客户端路径解析与元数据缓存，改为服务端解析：
  - **混合元数据索引**：默认 filename hashing（无客户端状态、rename 高效），依据"DL 数据集目录规模远大于元数据节点数"的实测分布来规避 hash 负载不均；
  - **惰性命名空间复制**（lazy namespace replication）降低路径解析的跨节点往返；
  - **并发请求合并**（concurrent request merging）提升服务端并发；
  - **VFS shortcut** 降低部署侵入（配合优化后的 FUSE 模块，该部分代码称将后续开源——**这是一个可复现性缺口**）。
- **语义与运维**：元数据服务端化把一致性推理集中化（比客户端缓存失效协议更容易论证），但把元数据服务器做成了新的扩展/故障焦点。评测中**未开启元数据复制**，因此论文数字不包含元数据高可用开销——评估时必须自行加回。
- **实验与结果**：13 台双路机器集群，与 CephFS / Lustre 对比：小文件读写吞吐 **最高 5.72×**；DL 模型训练吞吐 **最高 12.81×**。MLPerf Storage 模拟 ResNet-50 训练、90% 加速器利用率阈值下，FalconFS 支撑 **80 个加速器**，Lustre 仅 **32 个**。>256 KiB 的文件性能被聚合 SSD 带宽限制（即已打满介质）。生产：华为自动驾驶系统 **10,000 NPU，运行一年**。
- **相对传统云存储真正新的是什么**：**真新**。"客户端缓存是好东西"是 NFS/AFS 以来的默认假设，本文用负载特性（巨目录 + 客户端内存稀缺）论证在该场景下应当反向设计。这不是重命名，是前提改变导致的结论反转。
- **可复用设计与未解问题**：①如果你的 DFS 客户端 dentry/inode 缓存在 AI 训练下命中率低，先测量**缓存占用的客户端内存对 dataloader worker 数量的挤压**，这个成本传统上从不被计入；②filename hashing 的适用性完全依赖"目录规模 ≫ MDS 数量"，混合负载（含小目录、深层级科研数据）会退化——论文的混合索引就是为此留的后门，需读正文确认切换策略；③未解：元数据复制开启后的性能、rename 密集负载、跨 NPU/GPU 厂商的 VFS shortcut 可移植性。

---

### 3.3 NVIDIA BlueField-4 STX 参考架构 + NVIDIA CMX 上下文记忆存储平台

- **来源/单位**：NVIDIA（GTC 2026，2026-03-16 发布）
- **类型/证据级别**：**产品/参考架构声明（vendor claim）——无公开基线、拓扑、缓存状态与负载定义，设备 H2 2026 出货，本窗口内无第三方复现**
- **链接**：官方新闻 https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption ｜ CMX 产品页 https://www.nvidia.com/en-us/data-center/ai-storage/cmx/ ｜ 独立分析 https://nand-research.com/nvidia-stx-cmx-infrastructure-for-agentic-ai-context-storage/ ｜ 分层演进梳理 https://www.blocksandfiles.com/ai-ml/2026/03/30/nvidia-and-its-partners-kv-cache-extenders/5209284
- **一句话判断**：本窗口对**存储行业结构**影响最大的事件，但技术证据等级最低——必须按"架构信号"而非"性能数据"来读。**评分 17/25**（架构4/证据1/存储相关性5/AI特异性5/生产价值1/新颖性1）
- **架构与数据路径**：BlueField-4 = Vera CPU + ConnectX-9 SuperNIC 的存储向 DPU，跑 DOCA，网络侧 Spectrum-X Ethernet。CMX 是 STX 的首个机架级实现，作为**G3.5 上下文层**插在 Pod 内 NVMe（G3）与外部存储（G4）之间。数据路径：Dynamo 的 KV Block Manager 决定分层放置 → NIXL 执行跨层传输（NVLink / RDMA / GPUDirect Storage / NVMe-oF / 对象）→ BlueField-4 在 KV I/O plane 上终结 NVMe-oF 与 object/RDMA 协议 → 加解密与 CRC 完整性校验由 DPU 内的加速引擎线速完成，不消耗 host CPU → Grove 提供带 KV 局部性感知的拓扑放置，使工作负载跨节点迁移后仍可复用上下文。**DOCA Memos** 是留给存储厂商接入 G3.5 的开放接口。
- **厂商声明数字**：相较传统 CPU 型存储，**最高 5× tokens/s、4× 能效、2× 数据摄取速度**；CMX 声明"最高 5× tokens per second"。**这些数字均无基线定义、无拓扑、无缓存命中率、无模型/上下文长度说明——不可用于任何选型判断。**
- **生态与时间线**：存储与制造伙伴 AIC、Cloudian、DDN、Dell、Everpure（原 Pure Storage）、Hitachi Vantara、HPE、IBM、MinIO、NetApp、Nutanix、Supermicro、QCT、VAST Data、WEKA；上下文存储早期采用方 CoreWeave、Crusoe、IREN、Lambda、Mistral AI、Nebius、Oracle Cloud Infrastructure、Vultr。Supermicro 于 2026-03-17 公布首批 CMX 存储服务器原型。出货预期 **2026 H2**。前身是 2026 年 1 月 CES 上的 Inference Context Memory Storage Platform（ICMSP）。
- **相对传统云存储真正新的是什么**：**部分新**。DPU 卸载协议终结、线速加密/CRC 都不是新技术；真正新的是 **NVIDIA 显式把 KV cache 定义为独立数据类并为其定义了一个标准化的层级位置（G3.5）与厂商接入接口（DOCA Memos）**。这在治理层面的含义大于技术层面：它把"KV cache 该放哪"从各家自研变成了一个有参考架构的市场。
- **对现有系统的含义与风险**：①如果你运营对象存储或并行文件系统，G4 的位置正在被重新定义为"CMX 之后的持久层"，这会改变你的读写混合比与对象大小分布（更少的小随机读，更多的批量层级化写入）；②DOCA Memos 是 NVIDIA 定义的接口 → 存在生态锁定风险，值得对比 SNIA SDC 2026 上"Object-over-RDMA to generic JBOF without proprietary transport lock-in"这一议题的走向；③KV cache 可重算，因此该层**不需要传统耐久性**——你的 EC/复制策略在这一层是纯成本浪费，这是必须重新做设计判断的地方。

---

### 3.4 Strata: Hierarchical Context Caching for Long Context Language Model Serving

- **作者/单位**：Zhiqiang Xie 等（Stanford、上海交大、CU Boulder、CMU、NVIDIA、UMich）
- **时间/会场**：OSDI '26（2026-07-13～15）；预印本 arXiv 2508.18572（2025-08）
- **类型/证据级别**：同行评审 + 生产部署（**已作为 SGLang 的一部分实现并在生产中部署**）
- **链接**：https://www.usenix.org/conference/osdi26/technical-sessions ｜ arXiv https://arxiv.org/abs/2508.18572
- **一句话判断**：把"分层 KV 缓存为什么慢"这个问题定位到**分页布局导致的碎片化 I/O + 调度器无视加载延迟**，并给出 GPU-assisted I/O 这一可直接借用的机制。**评分 22/25**
- **架构与数据路径**：核心洞察是"系统是 loading-bound 而非 compute-bound"。两个机制：
  1. **GPU-assisted I/O**：不再反复调用 `cudaMemcpyAsync` 传小块，而是**启动一个 CUDA kernel**，数千线程各自把一小块数据从源（GPU global memory 或 CPU registered pinned memory）读入寄存器再流向目的。优势有三：并发度从 CPU 的数十提升到数千；**有效粒度低至 128 字节**，无需为了 I/O 效率而膨胀 page 大小；**布局变换在 I/O kernel 里近乎免费**，从而解耦 GPU 与 CPU 侧内存布局。
  2. **cache-aware 请求调度**：缓解 delay hit，平衡 batch 以隐藏加载延迟，并机会性地用互补任务覆盖不可避免的 stall。
- **语义与运维**：已知挑战是 GPU-assisted I/O kernel 与其他 kernel 共跑时的**运行期干扰**（论文明确承认并引用先前工作）。这是引入该机制到生产的第一风险点。
- **实验与结果**：吞吐相比 **vLLM-LMCache 最高 5×**、相比 **NVIDIA TensorRT-LLM 3.75×**，且不损害短上下文性能。
- **相对传统云存储真正新的是什么**：**真新且可迁移**。"用加速器做数据搬运以获得细粒度高并发"这一思路，对任何需要在 GPU 与主机/远端之间搬运碎片化数据的存储客户端都适用——包括训练数据加载与 checkpoint 恢复，不限于 KV cache。存储侧值得注意的是"**128 字节有效粒度**"这个数字：它意味着不必再为了满足 DMA 效率而把 KV page 放大，从而不必在缓存命中率与 I/O 效率之间做妥协。
- **可复用设计与未解问题**：①在你的 GDS 集成里，评估把 host-side gather/scatter 换成 GPU kernel 实现；②未解：与计算 kernel 的 SM 竞争如何在多租户下限制；GPU-assisted I/O 对 NVMe/远端（而非 CPU pinned memory）源端的适用边界。

---

### 3.5 ObjectCache: Layerwise Object-Storage Retrieval for KV Cache Reuse

- **作者/单位**：Yu Zhu 等 5 人（**待核实机构归属：arXiv 摘要页与 HTML 未在检索片段中给出完整单位，不做推断**）
- **时间/会场**：arXiv 2605.22850，2026-05-16 提交（v1）
- **类型/证据级别**：**预印本（preprint），无同行评审、无 artifact 声明**
- **链接**：https://arxiv.org/abs/2605.22850
- **一句话判断**：本窗口内**唯一一篇把 S3 语义本身扩展来适配 KV cache 的工作**，对做对象存储的人价值最高。**评分 20/25**（架构5/证据3/存储相关性5/AI特异性5/生产价值0/新颖性2）
- **问题与负载特征**：长且可复用的输入上下文（system prompt、RAG、代码仓库、长对话、agent 历史）导致 KV cache 容量成为主约束。目标：把 KV cache 放进 S3 兼容对象存储使容量不再是限制，同时把对 TTFT 的影响压到可接受。
- **架构与数据路径（这是本文的核心贡献）**：
  - **协议侧**：KV cache 以**细粒度、哈希寻址（hash-addressed）的 chunk** 存储，同时**扩展 S3 兼容请求，附加一个紧凑 descriptor**，其中声明：命中的 chunk 集合、模型布局、**交付顺序**、以及 RDMA 目标。存储服务端据此**一次性 gather 多个 chunk range，按层组装出 layer-major 的载荷，逐层通过 RDMA 直送推理节点**。
  - **调度侧**：利用逐层传输可与逐层 GPU 计算重叠这一性质，**按每请求的 per-layer stall target 分配带宽**，而不是均分或按字节比例——避免把带宽给到"再快也不能进一步降低 TTFT"的请求。
- **系统语义**：哈希寻址 chunk 天然内容寻址、可去重；但论文**未披露**一致性模型、失效/驱逐协议、多租户隔离与故障恢复行为——这是主要缺口。作者自己在讨论章节界定了定位：ObjectCache **不替代热层**（GPU/DRAM KV 池服务最热前缀），而是提供"持久、共享、S3 兼容的大前缀池容量层"，其价值在于放置灵活性与低存储成本，并使请求可以落在任意空闲 GPU 节点而不必绑定到缓存所在节点。
- **实验与结果**：100 Gbps RoCE 集群，栈为 **NIXL + Ceph RGW + DAOS**，模型 **Llama 3.1 8B**。64K 上下文：相比本地 DRAM **仅增加 5.6% 延迟**；4K 上下文（计算量不足以掩盖传输）：相比最优的本地 layerwise 基线**增加 56–75 ms**。共享带宽上限下，其调度器相比等分带宽把 added TTFT 降低 **1.2–1.8×**。作者明确给出适用边界：重算代价高、部分 prefill 提供足够的 per-layer 重叠窗口、且存储层能提供接近所需重叠带宽时才有用。
- **相对传统云存储真正新的是什么**：**真新**。"对象存储做缓存层"是老技术；"**把消费顺序写进请求描述符，让服务端按消费顺序 gather 并 push**"是对 S3 GET 语义的实质扩展——它把存储服务端从被动响应者变成了知道客户端计算 pipeline 的协作者。这一点比 KV cache 本身更值得存储工程师注意：**同类扩展可以用在任何有确定性消费顺序的负载上**（列式扫描、模型权重加载、EC 修复）。
- **可复用设计与未解问题**：①在你的 S3 网关上试做一个"顺序化多范围 gather + RDMA push"扩展 API，用模型权重加载做首个验证负载（比 KV cache 更容易复现）；②4K 上下文下 56–75 ms 的绝对开销说明**短上下文场景不适用**，这与 backend.ai 的独立分析一致（见 §7 反证）；③未解：一致性/驱逐/多租户/故障全部未评测；单模型 8B、单一 100 Gbps 拓扑，规模外推不成立。

---

### 3.6 AdaCheck: An Adaptive Checkpointing System for Efficient LLM Training with Redundancy Utilization

- **作者/单位**：Weijie Liu、Shengwei Li、Zhiquan Lai、Keshi Ge、Dongsheng Li、Kai Lu（国防科技大学），Qiaoling Chen（南洋理工），Peng Sun（上海人工智能实验室）
- **时间/会场**：FAST '26（2026-02-24～26，Santa Clara）
- **类型/证据级别**：同行评审（peer-reviewed），有 slides 与 video
- **链接**：https://www.usenix.org/conference/fast26/presentation/liu-weijie
- **一句话判断**：本窗口内 checkpoint 方向最"存储友好"的一篇——它把问题重构为**状态冗余识别**，而冗余识别本质上是去重，是存储工程师的本行。**评分 21/25**
- **问题与负载特征**：现有 checkpoint 系统几乎都是离线方案且绑定特定并行策略或模型架构；缺乏对自动并行规划器产出的**不规则并行**的适应性；并且**没有意识到大部分模型状态可以从 checkpoint 中排除**。
- **架构与机制**：用 **tensor redundancy** 抽象对"由并行策略与模型架构引起的状态冗余"建模；
  - **离线冗余利用**：用更少的状态集合构造 checkpoint；
  - **冗余检测器**：基于 hash 的数据一致性检查 + **ring-based 通信算法**（即用一次环形通信在全体 worker 间确认哪些张量是彼此的副本）；
  - **在线冗余利用**：进一步利用**跨训练迭代**的状态冗余压缩 checkpoint 体积。
- **实验与结果**：适应多种并行策略（含自动规划器产生的不规则并行）与稠密/稀疏架构。相比 SOTA：checkpoint 体积缩小 **6.00–896×**，checkpointing 频率提升 **1.46–111×**，对训练吞吐**几乎无开销**。
- **相对传统云存储真正新的是什么**：**部分新**。基于 hash 的去重是经典技术；新的是**冗余的来源是可从并行策略静态推导的**（不必靠内容扫描发现）——这是应用语义驱动的去重，去重率因此可以远高于内容无关的通用去重（896× 这个上界只可能来自结构性冗余，不是数据相似性）。
- **可复用设计与未解问题**：①如果你的存储系统为 AI 客户提供去重，**并行策略元数据是比 chunk 指纹更强的信号**，值得设计一个客户端 hint 通道；②ring-based 检测意味着检测本身要占用训练网络，需核实其与集合通信的干扰；③未解：跨迭代在线冗余利用对恢复正确性的影响（增量链长度、失效时的重建代价）未在摘要层给出。

---

### 3.7 GPU Checkpoint/Restore Made Fast and Lightweight（GCR）

- **作者/单位**：Shaoxun Zeng、Tingxu Ren、Jiwu Shu、Youyou Lu（清华大学）
- **时间/会场**：FAST '26；**Distinguished Artifact Award Winner**
- **类型/证据级别**：同行评审 + **artifact evaluated**
- **链接**：https://www.usenix.org/conference/fast26/presentation/zeng
- **一句话判断**：系统级 GPU C/R 从"要么快要么低开销"变成两者兼得，且增量 checkpoint 首次做到指令级脏数据识别。**评分 21/25**
- **机制**：**控制/数据分离的混合 C/R 方案**同时拿到低 C/R 延迟与对正常 GPU 执行的近零开销；为高效支持增量 checkpoint，引入 **CPU 侧 shadow execution**，用 **dirty templates** 在**指令级粒度**做脏缓冲识别，从而避免传统脏页跟踪的开销。
- **结果**：checkpoint 延迟相比 **cuda-ckpt（NVIDIA 官方方案）-72.1%**、相比 **PhOS（此前 SOTA）-63.6%**；恢复延迟 **-54.2% / -87.1%**；正常执行开销 **<1%**；增量 checkpoint 使体积 **-86.6%**、延迟 **-43.8%**。
- **相对传统做法真正新的是什么**：**真新**。用 CPU 影子执行来替代硬件脏页跟踪以获得更细粒度，这是一个可以推广到"任何需要增量快照但硬件粒度太粗"的场景的技巧——包括持久内存快照与块存储 CDP。
- **对存储的含义**：GPU 级 C/R 一旦变便宜，**弹性伸缩与任务切换会成为常态操作**，这会把 checkpoint 的访问模式从"低频大写"变成"高频中等写 + 频繁读"。如果你在为 checkpoint 设计存储池，容量规划模型需要相应改变。

---

### 3.8 Fast Cloud Storage for AI Jobs via Grouped I/O API with Transparent Read/Write Optimizations（AITURBO）

- **作者/单位**：Yingyi Hao、Xingda Wei、Dingyan Zhang、Tianle Sun、Rong Chen（上海交大），Ting Yao、Yiwen Zhang、Zhiyong Fu、Huatao Wu（华为云）
- **时间/会场**：FAST '26
- **类型/证据级别**：同行评审 + 第一方生产结果（已部署于华为生产云的训练作业）
- **链接**：https://www.usenix.org/conference/fast26/presentation/hao
- **一句话判断**：本窗口内唯一一篇**用一个新 I/O 接口同时覆盖 checkpoint 读写与 KV cache 读**的存储系统论文，接口设计思路值得直接借鉴。**评分 20/25**
- **机制**：①利用**加速器之间的高带宽计算 fabric** 来满足 AI 应用的带宽需求，从而不增加存储成本（即：把 GPU 互联当作存储数据路径的一部分）；②提出 **grouped I/O API**——应用把一组相关 I/O 声明为一个 group，存储层据此**自动推导优化的读/写计划**。作者论证这些计划可以**优于或持平应用层手工优化**，因为它们捕捉了 AI 负载的通用 I/O 模式且拥有存储层的全局视角。
- **结果**：在 checkpoint 读、checkpoint 写、KV-cache 读三类负载上，相比 SOTA（含 Megatron、Gemini、Mooncake，无论其是否启用应用层优化）取得持平或更好性能，且**应用侧代码改动极小**。
- **相对传统做法真正新的是什么**：**真新的是接口层次**。这是对"存储应该暴露什么抽象给 AI 应用"的直接回答：不是更快的 POSIX，不是新的对象 API，而是**让应用声明 I/O 之间的关系（group），把计划生成留给存储层**。这与 ObjectCache 的 descriptor 是同一思想的两种实现——**2026 年上半年最重要的接口趋势就是"让应用声明意图，存储层生成计划"**。
- **可复用设计**：这是我建议优先做原型验证的接口（见 §8 实验一）。

---

### 3.9 ZipLLM: Efficient LLM Storage via Model-Aware Synergistic Data Deduplication and Compression

- **作者/单位**：Zirui Wang 等（University of Virginia、Harvard）
- **时间/会场**：NSDI '26
- **类型/证据级别**：同行评审 + 全量公开数据集特征刻画 + 有项目主页
- **链接**：https://www.usenix.org/conference/nsdi26/presentation/wang-zirui ｜ arXiv https://arxiv.org/abs/2505.06252 ｜ 主页 https://storageai.github.io/ZLLM/
- **一句话判断**：模型仓库这个"新的冷数据大户"第一次得到系统性刻画，结论直接可用于任何模型托管/分发平台。**评分 19/25**
- **机制与发现**：刻画**全部公开可得的 Hugging Face LLM 仓库**，识别出三件事：模型族内存在**结构化稀疏 delta**；可用**位级相似度**做模型族聚类；**张量级**是正确的去重粒度（不是文件级，也不是内容定义分块）。据此设计 **BitX**——对微调模型与基座模型之间的 XOR 差分做**无损** delta 压缩；ZipLLM 把张量级去重与 BitX 统一。
- **结果**：模型存储消耗 **-54%**，比此前的去重与压缩方案好 **>20%**。
- **相对传统做法真正新的是什么**：**部分新**。delta 压缩与去重是老技术；新的是**粒度选择的依据**（张量级，来自模型结构）与 **XOR 差分作为微调模型的天然 delta 形式**。对存储工程师：这是又一个"应用语义比内容指纹更强"的例证，与 AdaCheck 同构。
- **注意**：-54% 是相对未去重的存储消耗；与通用去重相比的增量是 >20%。这两个数字不要混用。

---

### 3.10 Sugon ParaStor F9000 登顶 IO500 ISC26 双榜（产业事件 + 专家分析）

- **来源/单位**：中科曙光（Sugon）；IO500 ISC26 榜（2026-06-24 于 ISC High Performance 2026, Hamburg 发布）
- **类型/证据级别**：**厂商基准提交（vendor benchmark submission，配置已公开）+ 独立专家分析（非独立复现）**
- **链接**：IO500 提交配置 https://io500.org/submissions/configuration/803 ｜ IO500 ISC26 榜 https://io500.org/list/isc26/io500 ｜ 官方稿 https://www.prnewswire.com/news-releases/sugon-showcases-in-europe-tops-the-io500-302809317.html ｜ **专家分析（必读）** https://blog.glennklockwood.com/2026/07/isc26-recap.html
- **一句话判断**：本窗口内 HPC 存储领域最大的单一事件；技术上不新，但**它披露的架构细节让人得以判断"生产级 all-flash 并行文件系统在 2026 年长什么样"**。**评分 18/25**（架构4/证据3/存储相关性5/AI特异性2/生产价值3/新颖性1）
- **披露的架构（据 IO500 提交与 Lockwood 分析）**：shared-nothing 并行文件系统；Lustre 风格 **2U24 双控制器 HA 机箱**，每控制器 64 核 CPU、12× 15.36 TB NVMe、**4× 400G 国产 InfiniBand NIC**；数据用标准 **14+2 Reed-Solomon EC**；元数据**三副本**以避免小 I/O 的同步校验开销；自定义内核客户端；支持 RDMA 传输；带宽/IOPS 的 **min/max QoS 策略**；**XDS**（其 GPUDirect Storage 等价物）支持完全绕过主机；客户端可用本地 SSD 与 RAM 做预取与缓存以改善小文件性能；支持闪存到 HDD 的分层（IO500 提交仅测全闪）。
- **规模与数字**：扩展到 **221 机箱 / 442 服务器 / 5,304 块 15.36 TB NVMe**，裸容量 **72.3 PiB**，格式化后 **63 PiB**（几乎恰好等于 14+2 的 EC 开销）。ior-easy 折算 **每服务器读 80 GB/s、写 73 GB/s**，即**每 SSD 读写均超过 6 GB/s**。每机箱 160 GB/s 读 / 146 GB/s 写，与当周展示的最新 HPE Cray Lustre 一体机（190 / 140 GB/s）**处于同一水平**。它以**更少的服务器（442 vs 642）与更少 SSD（约 5K vs 10K）** 超过了此前登顶的 Argonne DAOS 系统，带宽与元数据性能均为其两倍以上。
- **必须一并记录的专家质疑（Lockwood）**：
  - 14+2 EC 与 12 盘服务器**不整除**，推断 EC 跨服务器做 → 有延迟影响，且使双控 HA 机箱变得冗余；机箱内 + 机箱间双层 HA 既昂贵又复杂，暗示其 EC 可能不如宣称的好；
  - 声明的元数据性能"只有在使用高级索引结构 + redirect-on-write 时才可能达到"，但**元数据处理机制完全未披露**；格式化容量几乎不留空间给元数据与内部数据结构 → 怀疑其在可靠性与持久性上有取舍；
  - **性能最小值型 QoS** 在工程上极难实现，短期内做对的可能性低；
  - 结论性提醒："**在空系统上跑得快是最容易的部分；日复一日保持可预测和可靠才是难的。**"
  - 另一个关键对比点：**ParaStor 是真正的 POSIX(ish) 文件系统**，因此必须在遵守 POSIX 的同时拿到高元数据性能；**DAOS 并未做到这一点**，其 IO500 成绩使用的是非标准的类文件 API（DFS）。
- **对 AI 负载的相关性**：**中等偏低**。IO500 本质上是元数据性能排行榜（GiB/s 与 kIOPS 的等价换算方式有其人为性），与 AI 训练/推理的端到端指标（GPU 利用率、JCT、TTFT）没有直接映射。不要把 IO500 排名当作 AI 存储选型依据。

---

### 3.11 其他值得读但不单独展开的条目（按路径分组）

**训练输入**
- **Seneca**（FAST '26；Syracuse、Huaibei Normal、Samsung Semiconductor、FIU）：为数据存储与摄取（DSI）流水线做**缓存分区 + 数据采样**优化。两个机制：用数据流水线性能模型对 **encoded / decoded / augmented 三种数据形态**做最优缓存分区；随机 batch 采样时**机会性地优先服务已缓存数据**，使并发作业互相受益。改造 PyTorch 实现。makespan 相比 PyTorch **-45.23%**，数据处理吞吐相比次优 dataloader **最高 3.45×**。→ 值得注意的是"按数据形态分区缓存"这个想法：同一份样本在流水线不同阶段有不同大小与复用率，传统缓存把它们当同一类对象是错的。https://www.usenix.org/conference/fast26/presentation/desai
- **MegaScale-Data: Scaling DataLoader for Multisource Large Foundation Model Training**（EuroSys '26；HKU + ByteDance）：多来源数据的 dataloader；现代 LFM 训练框架以数据并行方式使用 dataloader、每个 loader 处理不相交子集，当训练数据来自多个不同来源时该模型失效。https://dl.acm.org/doi/10.1145/3767295.3803568
- **MinatoLoader**（EuroSys '26；McGill 加拿大 + INESC TEC / University of Minho 葡萄牙）：数据预处理加速。**本窗口内欧洲机构在训练输入路径上的主要产出。** https://dl.acm.org/doi/10.1145/3767295.3769376
- **MesaFS: An I/O-Efficient Metadata Service for Distributed File Systems**（EuroSys '26；清华 Hao Guo / Jiwu Shu / Youyou Lu）https://dl.acm.org/doi/10.1145/3767295.3803573
- **SwitchFS: Asynchronous Metadata Updates for Distributed Filesystems with In-Network Coordination**（EuroSys '26；上海交大）——与 SIGMOD '26 的 Switch∆ 是同一思路线（在网络内做元数据可见性协调）。https://dl.acm.org/doi/10.1145/3767295.3769349

**Checkpointing**
- **Checkmate: Zero Performance Overhead Model Checkpointing via Network Gradient Replication**（NSDI '26；Tufts + MIT）：复用为梯度同步而在网络中已复制的数据来构造 checkpoint，避免正常 checkpoint 路径上的额外网络传输与磁盘 I/O；用**动态可重配的网内 checkpoint 副本放置**与容错加速器流水线在故障下保持韧性，同时把 checkpoint 创建与梯度计算/传播重叠。32 节点 GPU 集群上相比 GPU 优化的 checkpointing **吞吐提升近 100%**，相比异步 checkpointing 最高 **13.7%**。https://arxiv.org/abs/2507.13522
- **Checkpoint Lite, Recover Right: Efficient Fault Tolerant Training of MoE Models Using Sparse Checkpoints**（NSDI '26；Stanford + NVIDIA）https://www.usenix.org/conference/nsdi26/presentation/gandhi

**模型分发 / 冷启动**
- **Accelerating Model Loading in LLM Inference by Programmable Page Cache**（FAST '26；华为）：**PPC** 可编程 page cache 框架 + **MAIO** 策略；用 I/O 模板机制充分利用 SSD 带宽、XPU 亲和性与数据局部性来优化预取与驱逐。模型加载延迟相比现有优化 **-79%**；真实弹性部署场景下推理吞吐 **+36%**。作者强调**兼容性**是能否广泛落地的决定因素——这是一个把优化做在内核文件系统缓存策略层而非旁路的选择，运维上比自研用户态栈友好得多。https://www.usenix.org/conference/fast26/presentation/liu-yubo
- **HydraServe**（NSDI '26；PKU + 阿里云）：serverless LLM 冷启动；主动跨服务器分发模型以快速拉取、worker 内重叠冷启动各阶段、避开 GPU-网络争用的 worker 放置、合并流水线以降低冷启动资源占用。冷启动延迟 **-1.7× ~ -4.7×**，SLO 达成率 **+1.43× ~ +1.74×**。代码 https://github.com/LLMServe/hydraserve
- **ThinkAhead**（FAST '26；上海交大 + 阿里云 + CUHK）：分析阿里云 EBS 约 **16 万次真实镜像加载事件**，发现首次块访问期间的慢 I/O 占全部慢 I/O 的 **40%**，是主要瓶颈。数据驱动的块预载序列预测。数据块命中率 **最高 7.27×**，尾部等待时间 **-98.7%**。→ 对做块存储的人：这是本窗口内最好的一份"冷启动慢 I/O 归因"生产数据。https://www.usenix.org/conference/fast26/presentation/chen
- **CoFS**（FAST '26；麒麟软件）：容器启动文件系统，镜像构建期构造**最小完美哈希函数（MPHF）**，把文件元数据存入按 hash 值索引的稠密数组；多数情况下**在内核态用不到一次 I/O 完成 lookup**，绕开 FUSE 的用户态查找；再用第二个 MPHF 支持基于全路径哈希的并行查找；数据访问用宿主内核文件系统的稀疏文件实现细粒度缓存。相比 fuse-loopback 查找性能 **+86%**。https://www.usenix.org/conference/fast26/presentation/wang-li

**推理状态**
- **No Buffer, No Bottleneck: Efficient Zero-Copy KV Cache Offloading for Long-Context LLMs**（OSDI '26；UVA）：**DirectKV**，让 GPU kernel 通过高带宽 CPU-GPU 互联**直接访问驻留在 CPU 的 KV block**，取消中间缓冲。https://www.usenix.org/conference/osdi26/presentation/luo
- **Tutti: Making SSD-Backed KV Cache Practical for Long-Context LLM Serving**（arXiv 2605.03375，2026-05）：GPU 中心的对象存储设计 + layerwise GPU 计算-I/O 流水线，使 SSD 后端 KV 缓存达到接近 DRAM 的效率。文中给出的分层观测很有价值：**DRAM 层是高效的**——从 CPU DRAM 加载 KV 相比 HBM 只有适度开销，因为低延迟细粒度访问加上 LMCache 的 GPU-assisted copy 把大量顺序 `cudaMemcpyAsync` 合并成少量 GPU kernel，且 DRAM 的低延迟与强随机访问使 layer-wise 流水线能有效把数据搬运隐藏在 attention 计算之后。https://arxiv.org/html/2605.03375
- **Bidaw**（FAST '26；清华 + 中国电信 + 中国地质大学）：两层（host memory + SSD）KV 存储的**双向计算-存储感知**。量化了问题规模：在其交互式对话负载上，现有方案从两层存储加载 KV 使服务延迟**最高增加 3.8×**、吞吐**下降 2.0×**（相比理想大内存）。两个机制：计算引擎按 **KV 加载延迟感知**调度请求（分离 KV 位于不同层的请求，并按 KV 大小重排以减少阻塞）；存储系统**利用 LLM 生成的响应来预测用户访问模式**以指导驱逐。延迟 **-3.58×**，吞吐 **+1.83×**。https://www.usenix.org/conference/fast26/presentation/hu-shipeng
- **CacheSlide**（FAST '26；上海交大 + 浪潮 + 华为云 + PKU）：识别 agent 工作流中的 **Relative-Position-Dependent Caching (RPDC)** 模式——可复用片段在绝对位置漂移下保持一致的相对顺序。扩展 vLLM 的 KV 管理，引入 Chunked Contextual Position Encoding 与 Weighted Correction Attention，并做 layer-wise 与 spill-aware 优化。延迟 **-3.11~4.3×**，吞吐 **+3.5~5.8×**。https://www.usenix.org/conference/fast26/presentation/liu-yang
- **SolidAttention**（FAST '26；上海交大）：面向内存受限 PC 的 SSD 服务；动态注意力稀疏算法与 SSD 存储管理协同设计，把多个 KV pair 合并成粗粒度 block 以打满 SSD 带宽，并用推测性预取利用稀疏注意力的时间局部性。128k 上下文下推理速度 **最高 3.1×**，KV 内存占用 **-98%**，精度无损。https://www.usenix.org/conference/fast26/presentation/zheng
- **SYMPHONY**（NSDI '26；UT Austin + UW-Madison）：利用多轮对话的 hint 把 KV cache **迁出服务关键路径**（而非重算或把服务钉在特定机器上），支撑请求数 **>8×** 于 SOTA 基线且延迟画像相近。https://arxiv.org/abs/2412.16434
- **DroidSpeak**（NSDI '26；UChicago + Microsoft）：首个在**同架构不同模型**（含跨分布式节点）之间复用前缀 KV cache 的系统；选择性重算一小部分层、复用其余层，并把重算与复用缓存的加载流水化。吞吐 **最高 4×**，prefill 延迟 **约 3.1×**，质量损失可忽略。https://arxiv.org/abs/2411.02820
- **Cortex**（NSDI '26；NUS + USTC + UofT + Sea AI Lab）：面向 LLM agent 的**跨区域语义知识缓存**，目标是语义复用而非精确匹配复用；基于 Semantic Element 与 Semantic Retrieval Index 抽象，加上语义感知的命中/驱逐/预取与同置的轻量 LLM judger。搜索类负载吞吐 **最高 3.6×**，精度接近无缓存基线。https://arxiv.org/abs/2509.17360

**云产品发布（第一方声明，非独立验证）**
- **Google Cloud Next '26（2026-04-22）—— Cloud Storage Rapid 家族 + Managed Lustre**：**Rapid Bucket**（高性能 zonal 对象存储，声明 >15 TB/s 聚合读吞吐、最高 **20M ops/s**、亚毫秒延迟）；**Rapid Cache**（原 Anywhere Cache，对既有 bucket 加速读，无需改代码，声明 2.5 TB/s 聚合读吞吐）及其 **ingest-on-write**（写入 bucket 时同步入缓存，使首次读即命中，声明 checkpoint 恢复快 **2.2×**）。多模态训练中声明 GPU 阻塞时间 **-50%**、数据加载 **最高 2.5×**；相比传统对象存储 checkpoint 恢复 **最高 5×**、写入 **最高 3.2×**。**Managed Lustre**（基于 DDN EXAScaler）声明最高 **10 TB/s** 吞吐（同比 10×），由 C4NX VM + Hyperdisk Exapools 支撑，checkpoint 写入与恢复相比其他 Google Cloud 存储方案快 **2.6×**；新增 **Dynamic 层**（$0.06/GB-月，从持久盘而非对象缓存供数以消除性能悬崖）；支持 **TPUDirect 与 RDMA** 使数据绕过 host 直达加速器；新 **Z4M** 实例供 ISV 集成第三方并行文件系统（点名 VAST Data、Sycomp）。https://cloud.google.com/blog/products/storage-data-transfer/next26-storage-announcements ｜ https://cloud.google.com/blog/products/compute/ai-infrastructure-at-next26
  - 值得注意的**第一方用户证言**（Managed Lustre 产品页）：某数学推理模型团队称把 Managed Lustre 作为"热 checkpoint"的区域缓存后，训练实验中断减少至少 **50%**，可跑两倍实验数；训练作业写 checkpoint、后续推理或新训练作业在离线系统消费，数据获取速度提升 **最高 15×**、启动时间 **-50%+**。
- **AWS（2026 H1 存储发布流）**：**S3 Vectors 正式 GA**（单 bucket 支持最高 10 亿向量）；**S3 Tables** 新增复制与 Intelligent-Tiering；**S3 Storage Lens** 增加性能指标、支持数十亿 prefix；**S3 general purpose bucket 的账户区域命名空间**；**FSx for Lustre Intelligent-Tiering** 扩展至 13 个新区域（2026-06-16）；**S3 Access Points 支持 FSx for NetApp ONTAP**（企业文件数据以 S3 API 直接供 Bedrock/SageMaker/Athena 消费）。https://aws.amazon.com/blogs/aws/category/storage

---

## 4. 横向对比：KV cache 存储层（同一路径的 5 个方案）

| | **NVIDIA CMX (G3.5)** | **Strata** | **ObjectCache** | **Tutti** | **Bidaw** |
|---|---|---|---|---|---|
| 接口/协议 | DOCA Memos 开放接口；NVMe-oF + object/RDMA 由 DPU 终结 | SGLang 内部（GPU-assisted I/O kernel） | **扩展的 S3 兼容 GET**（附带交付顺序 descriptor） | GPU 中心对象存储设计 | 两层存储（host mem + SSD）内部接口 |
| 放置/分层 | G1 HBM → G2 DRAM → G3 Pod 内 NVMe → **G3.5 DPU 上下文层** → G4 外部存储 | GPU ↔ host（布局解耦） | 热层在 GPU/DRAM；ObjectCache 为**共享持久容量层** | HBM ↔ DRAM ↔ SSD | host memory ↔ SSD |
| 热路径机制 | Dynamo KVBM 预置 + NIXL 传输 + Grove 拓扑放置 | CUDA kernel 数千线程搬运，128B 有效粒度 | 服务端 gather 多 range → **layer-major** → RDMA 直送 | layerwise GPU 计算-I/O 流水线 | 延迟感知请求调度 + 响应驱动的驱逐预测 |
| 元数据/控制面 | Dynamo + Grove（KV 局部性感知） | cache-aware 调度器 | 哈希寻址 chunk + descriptor | 未充分披露 | 计算引擎与存储双向感知 |
| 语义披露 | 未披露一致性/故障语义 | 承认 kernel 共跑干扰 | **一致性/驱逐/多租户/故障均未评测** | 未充分披露 | 驱逐策略披露；一致性未展开 |
| 硬件假设 | BlueField-4 + ConnectX-9 + Spectrum-X（强绑定） | 需 pinned memory + CUDA | 100 Gbps RoCE + NIXL + Ceph RGW/DAOS | 2× SSD（读 29 GB/s / 写 12 GB/s 峰值） | 商用 host memory + SSD |
| 结果指标 | 5× tokens/s、4× 能效、2× 摄取（**均无基线**） | 吞吐 5×(vs vLLM-LMCache)、3.75×(vs TRT-LLM) | 64K 上下文仅 +5.6% vs 本地 DRAM；调度 1.2–1.8× | 相比 SOTA GDS SSD 方案改善（数字待核） | 延迟 -3.58×、吞吐 +1.83× |
| 证据等级 | **厂商声明** | 同行评审 + 生产部署 | **预印本** | **预印本** | 同行评审 |
| 开源 | 否（Dynamo/NIXL 开源，CMX 不是） | 是（SGLang 内） | 未声明 artifact | 未声明 | 未声明 |

**读表要点**：五个方案的机制层高度收敛（分层 + 按层交付 + 感知调度），差异主要在**接口位置**（DPU / 推理引擎 / 存储协议 / 存储客户端 / 双向）与**证据强度**。对存储工程师最可操作的是 ObjectCache 的协议扩展与 AITURBO 的 grouped I/O API——这两个都在**存储侧**留下了可实现的接口，其余方案的机制在推理引擎侧。

---

## 5. 工程与开源雷达

| 项目 | 成熟度 | 许可（已核实?） | 集成面 | 预估评估成本 |
|---|---|---|---|---|
| **FalconFS** — github.com/falcon-infra/falconfs | **生产级**（华为自动驾驶 10,000 NPU 一年） | Copyright (c) 2025 Huawei Technologies；**具体开源许可未在检索片段中确认，需查仓库 LICENSE** | POSIX(ish) + VFS shortcut + 优化 FUSE 模块（**该 FUSE 部分代码尚未开源**） | 中高：需搭 5–13 节点集群；元数据复制默认关闭需自行开启后重测 |
| **NVIDIA NIXL** | 生产（GTC 2025 开源，2026 广泛集成） | 开源（Apache 系，需核实版本） | 后端插件：UCX/RDMA、**GDS**、NVMe、S3；被 vLLM/SGLang/LMCache/Dynamo/Mooncake/llm-d 使用；自带 `nixlbench` 基准工具 | **低**：`nixlbench` 可直接对你的存储后端做 GDS 路径验证，是本窗口内最低成本的入口 |
| **LMCache** | 生产 | 开源 | 2026-03 与 Dynamo 1.0 完成集成；存储插件接口接入 Dynamo 分层策略；全面支持 NIXL（含 NVLink / RDMA NIC / GDS）；VAST Data 与 WEKA 已做生产规模验证 | 低中：适合作为"你的存储做 KV 容量层"的第一个真实客户端 |
| **NVIDIA Dynamo (1.0)** | 生产 | 开源 | KVBM 分层管理 + KV 感知路由；GTC 2026 宣称 7× 吞吐提升（**厂商声明**） | 中 |
| **Xerxes**（CXL 仿真框架，FAST '26，PKU 等）— github.com/ChaseLab-PKU/Xerxes | 研究原型，**开源** | 未核实 | 建模最新 CXL 协议新特性：port-based routing、device-managed coherence、PCIe 6.0；专用 interconnect 层支持多种拓扑 | 中：在无 CXL 硬件时评估 G2/G3 层设计的可行工具 |
| **Cylon**（CXL-SSD 全系统模拟器，FAST '26，Virginia Tech，基于 FEMU） | 研究原型 | 未核实 | 复现亚微秒级 cache 命中与数十微秒级 miss（落 NAND）；可配置缓存策略；应用级接口支持软硬件协同设计；已对真实 CXL-SSD 原型做校准 | 中 |
| **WARP**（FDP SSD 仿真器与研究平台，FAST '26，Virginia Tech + Samsung + Western Digital） | 研究原型，声明为**首个开放 FDP 仿真器** | 未核实 | 跨设备跨负载刻画显示：**当 RUH 隔离与对象生命周期对齐时 FDP 能维持接近 1 的 WAF，但在误分类、RUH 干扰或对抗性失效模式下失败** | 中：如果你在考虑用 FDP 降低 checkpoint/KV 写放大，这是先做仿真再买硬件的正确顺序 |
| **Alibaba 生产 AI 集群 trace** — github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2026（OSDI '26 *Heterogeneity at Hyperscale*） | 数据集 | 未核实 | 6 个月生产 AI 集群 trace，含 stranded capacity、局部性约束、异构 GPU | **低**：可直接用于容量与放置策略的离线评估 |
| **HydraServe** — github.com/LLMServe/hydraserve | 研究原型 | 未核实 | serverless LLM 冷启动 | 中 |

**标准与规范动态（窗口内）**：
- SNIA 在 2026 年推出 **StorageAI** 议题线（2026-02-03 起公开讨论"存储、计算与 AI 协调"），并与 NVM Express、OCP、Ultra Ethernet Consortium、Linux Foundation 对齐。SDC 2026（9/28–30, Santa Clara）已公布的议题中有一条与本报告直接相关且值得跟踪：**"Democratizing AI Storage: Can We Achieve Object-over-RDMA to Generic JBOF Without Proprietary Transport Lock-In?"** —— 这是对 NVIDIA DOCA Memos 路线的开放替代方向。https://www.snia.org/sniadeveloper/sessions-2026
- **CXL 4.0 规范**（2025-11-18 发布，**窗口外但为本窗口所有 CXL 相关工作的前提**）：128 GT/s（PCIe 7.0），bundled ports 聚合多个物理连接为单一逻辑连接，声明可达 1.5 TB/s。
- **Advancing Data Integrity in Linux**（FAST '26；Samsung Semiconductor、Christoph Hellwig、EPFL）：给 Linux 主线补齐 E2E 数据保护——新增 **io_uring 接口让应用与数据一起传递完整性元数据**；在 block-integrity 中支持**灵活的 PI 放置**（使此前被拒的设备配置可用）；提出 **FS-PI**（文件系统自己生成与校验完整性元数据并利用支持 PI 的硬件），在 XFS 中首次引入原生数据校验和，在 BTRFS 中用轻量路径替代 checksum tree。BTRFS 性能 **+26%**、host CPU 利用率 **-58%**、设备写入 **-52%**（SSD 寿命 +23%）。→ 与 AI 无直接关系，但对任何以 NVMe 为底的 AI 存储池是本窗口最实用的一篇。https://www.usenix.org/conference/fast26/presentation/gupta

---

## 6. 视频精选

本窗口内的会议录像多为注册者可见或未公开时间戳，因此只列**已确认有公开 video 且技术信息密度高**的项，且**不编造时间戳**。

1. **FAST '26 Keynote — "Everything You Always Wanted to Know about Storage Analysis (But Were Afraid to Ask)"**，Erez Zadok（Stony Brook University，FSL 主任、ACM TOS 主编）。2026-02-25，约 60 分钟（含 Q&A）。USENIX 页面标注有 slides 与 video。**为什么值得专家的时间**：这是一位做了 30 年存储性能刻画的人对"基准测试为什么骗人"的系统性总结（其贯穿全场的答案是 "it depends"）。在本窗口充斥厂商声明数字（5×、4×、10×）的背景下，这是校准你自己判断标准的最佳材料。https://www.usenix.org/conference/fast26/presentation/zadok
2. **FAST '26 技术会议录像**：以下与本报告直接相关的论文在 USENIX 页面标注有 **video**：GCR（GPU C/R）、AITURBO（Grouped I/O）、AdaCheck、Bidaw、CacheSlide、SolidAttention、MAIO/PPC、ThinkAhead、CoFS、Cylon、WARP、Advancing Data Integrity in Linux、DisCoGC。入口：https://www.usenix.org/conference/fast26/technical-sessions
3. **未验证、不推荐直接引用**：GTC 2026 关于 STX/CMX 的 session 录像、SiliconANGLE theCUBE 的 VAST Data 访谈（含 "单 GPU 服务器推理能力 10× 提升"的说法，属**赞助段落的厂商口头声明**，无方法学披露）。若需引用 CMX 架构，请用 §3.3 的官方与独立分析链接而非视频。

---

## 7. 趋势判断与反证

### 趋势 1：KV cache 成为一等存储数据类，且其"可重算"属性正在重写耐久性设计
**支持**：NVIDIA 显式定义"AI-native 数据类"并给出 G3.5 层与 DOCA Memos 接口（GTC 2026）；ObjectCache 把 S3 定位为 KV 的持久共享容量层；15 家主流存储厂商同时接入 STX 生态；FAST '26 出现两个独立的 "AI and LLMs" session，其中 KV 存储占多数。
**反证与限制**：
- backend.ai 的独立分析（2026-04）指出 **KV cache offload 不是普适收益**：团队级 RAG 场景看似完美契合（长 system prompt + 检索产生重复前缀），但实践中**该前缀的 KV block 会常驻 GPU 而不被驱逐**，此时叠加 offload 只带来哈希与搬运开销、反而更慢——正确做法是去掉外部 offload 层、依赖 vLLM 内建 prefix caching。这直接否定了"给所有推理场景加 KV 存储层"的销售叙事。
- ObjectCache 自己给出的边界条件（重算代价高 + 有足够 per-layer 重叠窗口 + 存储层能提供所需重叠带宽）意味着**三个条件同时满足才有用**；其 4K 上下文下 +56–75 ms 的开销是明确的负结果。
- CMX 的全部性能数字为厂商声明，设备 2026 H2 才出货，本窗口内**零第三方复现**。

### 趋势 2：接口层的变化比介质层更重要——"应用声明意图，存储层生成计划"
**支持**：AITURBO 的 grouped I/O API（声明 I/O 之间的关系，存储层推导读写计划，效果持平或优于应用层手工优化）；ObjectCache 的 S3 descriptor（声明命中 chunk、布局、**交付顺序**、RDMA 目标）；OSDI '26 字节论文的预测式复制（用训练调度器语义而非访问历史决定放置）；AdaCheck 与 ZipLLM 的去重都依赖应用语义（并行策略、模型族结构）而非内容指纹。
**反证与限制**：这四例都在**单一组织内**做了应用与存储的协同设计（华为云 + 上交、字节 + 中科大等）。作为通用存储产品，你无法要求所有客户改接口。字节论文的价值恰在于反向论证：**它挑选的三个优化都是"最小工程侵入"且不依赖新接口的**——这暗示需要新接口的方案在跨组织落地上会更难。

### 趋势 3：元数据路径的传统共识被 AI 负载特性推翻，且元数据被推向网络与服务端
**支持**：FalconFS 的 stateless client（客户端缓存"既无效又浪费内存"，生产 10,000 NPU 验证）；SwitchFS 的网内元数据协调（EuroSys '26）；MesaFS 的 I/O 高效元数据服务（EuroSys '26）；Lockify 对 Linux 内核 DLM 的观测（共享盘文件系统中，即使低竞争，文件/目录创建的锁获取开销**随客户端数增长**，FAST '26）；ParaStor 用元数据三副本以规避小 I/O 的同步校验开销。
**反证与限制**：
- FalconFS 的结论**依赖"DL 数据集目录规模 ≫ 元数据节点数"**这一负载性质。混合负载（科研数据的深层级小目录）下 filename hashing 会退化，论文自己保留了混合索引作为后门。
- FalconFS 评测**未开启元数据复制**，因此其数字不含元数据 HA 开销。
- Lockwood 的 ISC'26 观察提供了一个重要的相反视角：**"只有糟糕的 AI 才需要大量 IOPS"**——他主动询问多位从业者"到底是哪个 AI 负载需要这些 IOPS"，答案几乎都是"我不知道"或"你得问我的用户"；唯一能背书该论断的人给出的理由是"今天在超算上跑 AI 的人大多不知道自己在做什么，其 I/O 模式未针对并行存储优化"。他的结论是：为 AI 支持高 IOPS 更多是在**迁就一批缺乏经验的新用户**，而不是支持一种根本性的新负载；HPC 界似乎愿意花硬件的钱（IOPS）去解决本该用软件解决的问题（糟糕的用户代码）。**这条反证必须与趋势 3 一起读**：元数据压力的一部分可能是可以在应用侧消除的。

### 趋势 4：Checkpoint 从带宽问题变成冗余与副本复用问题
**支持**：AdaCheck（结构性张量冗余，6–896× 缩小）；Checkmate（复用网络里已复制的梯度，正常路径零额外传输）；GCR（控制/数据分离 + CPU 影子执行做指令级增量，体积 -86.6%）；ZipLLM（模型族内张量级去重 + XOR delta，-54%）；Google Cloud ingest-on-write（写时入缓存以加速恢复，2.2×）。**五个独立工作，无一以更快介质为主要手段。**
**反证与限制**：
- 云厂商这一侧的叙事恰恰相反：Google 的主要卖点是 Managed Lustre **10 TB/s**、Rapid Bucket **20M ops/s** 与亚毫秒延迟——即"用更快的存储解决 checkpoint"。两条路线并不矛盾（一个降低数据量，一个提升带宽），但**它们竞争同一笔预算**，而本窗口内**没有任何公开数据比较"投资去重/复用"与"投资带宽"的边际收益**。这是一个真实的决策空白。
- 所有去重/冗余方案都增加了**恢复路径的复杂度**（增量链、跨迭代依赖、网内副本的失效处理），而这些方案的评测普遍不包含故障注入下的恢复正确性与恢复时间。

### 趋势 5：存储侧算力正式进入 AI 数据路径，但收益与故障域扩张同时发生
**支持**：字节把多模态数据转换卸载到**存储层 CPU**；BlueField-4 在 DPU 上终结 NVMe-oF 与 object/RDMA 协议并线速做加密与 CRC；ParaStor 的 XDS 完全绕过主机；PolarStore 的双层压缩（PolarCSD 硬件内压缩 + 软件轻量压缩，压缩比 3.55、存储成本 -60%，部署于数千台存储服务器、管理 100+ PB）；ASIC 压缩加速器在存储系统中的设计、放置与 profiling（EuroSys '26，DapuStor）。
**反证与限制**：卸载把计算节点的故障隔离性交换成了存储节点的故障域扩大（存储节点崩溃现在会杀掉数据转换）。字节论文强调"最小侵入"，说明他们自己也把可移植性置于极致优化之上。此外**存储节点 CPU 空闲是前提**——如果你的存储节点已被 EC、压缩、校验和打满（见 FS-PI 论文中 BTRFS 的 CPU 数据），卸载空间不存在。

### 趋势 6：地缘政治与主权因素成为存储技术路线的一阶变量
**支持**：ParaStor F9000 以**更少的服务器与 SSD** 登顶 IO500 双榜（Lockwood 评价：中国"已不再是填补出口管制留下的空白，而是在为自身需求构建自己的一流 HPC 硬件与软件栈"），其 400G IB NIC 为国产；OSDI 自 1994 年以来**首次由中国大陆机构作为第一完成单位获最佳论文**且全部作者来自国内机构；ISC'26 上主权 AI 成为主导议题（20 个自我标注为"AI 主权"的 session 中约 60% 为厂商推介）；本窗口内的一个直接触发事件是 **2026-06-12 美国政府对 Anthropic Fable 5 / Mythos 5 的外国国民使用限制**（该限制于 6 月 30 日解除、7 月 1 日恢复访问；官方说明 https://www.anthropic.com/news/fable-mythos-access）。
**反证与限制**：这是政策与市场观察而非技术判断，本报告不对其做价值评价。技术上需要注意的是：**IO500 排名与 AI 端到端指标没有映射关系**，把它当作技术能力的单一证据是错误的；ParaStor 的元数据机制未披露、无独立复现，Lockwood 的质疑（EC 跨服务器的延迟含义、元数据持久性取舍、性能最小值型 QoS 的可实现性）在有独立测试前均未被排除。

---

## 8. 建议实验与关注名单

### 实验一（优先做）：在现有对象存储上实现"顺序化多范围 gather + RDMA push"
- **动机**：ObjectCache 与 AITURBO 收敛到同一个接口结论，而这个接口可以在**你已有的 S3 网关**上做原型，不需要改推理引擎。
- **负载**：不要用 KV cache 做第一个负载（依赖太多）。用**模型权重加载**：单一大模型、确定的 layer 消费顺序、可复现。
- **基线**：①标准 S3 GET（多并发 range request，客户端 gather）②GDS 直读（若你已支持）。
- **指标**：模型加载端到端时间（对齐 MAIO 论文的 -79% 量级作为参照）、每层就绪时刻相对 GPU 计算需求时刻的偏差分布（这是新指标，比平均带宽有信息量）、存储侧 CPU 与 NIC 利用率、字节移动量。
- **故障用例**：descriptor 中声明的 chunk 部分缺失（副本不可用）时的降级行为；RDMA 目标不可达时的回退路径；gather 中途存储节点故障。
- **决策价值**：如果每层就绪偏差能压到 GPU 每层计算时间以内，你的对象存储就具备了作为 G4/持久上下文层的技术资格；如果不能，说明瓶颈在服务端 gather 而非网络，那么再投带宽是浪费。
- **成本估计**：2–4 人周做原型 + 1 人周做测量。用 `nixlbench` 做 GDS 路径的前置验证可先花 2 天排除环境问题。

### 实验二：量化客户端元数据缓存对 dataloader 的内存挤压（验证 FalconFS 的前提）
- **动机**：FalconFS 的核心论断（客户端缓存"既无效又浪费内存"）是可以在你自己的系统上**廉价验证**的，而结论如果成立，收益是架构级的。
- **负载**：MLPerf Storage 的 ResNet-50 配置（10M 文件 / 1M 目录 / 112 KiB）作为可比基线；再加一组你自己的真实数据集分布。
- **基线**：现有客户端（dentry/inode 缓存开启）vs 缓存关闭 vs 缓存限额分档。
- **指标**：客户端缓存命中率、缓存占用的客户端内存、**在固定客户端内存预算下可运行的 dataloader worker 数**（关键指标，传统上从不测量）、90% 加速器利用率阈值下可支撑的加速器数（对齐 FalconFS 的 80 vs 32）。
- **故障用例**：元数据服务端化后，MDS 成为新焦点——必须测 MDS 故障切换期间的作业行为，并**开启元数据复制后重测性能**（FalconFS 论文未做）。
- **决策价值**：直接决定"是否值得投入做 stateless client 改造"。

### 实验三：比较"去重/冗余利用"与"提升带宽"在 checkpoint 上的边际收益
- **动机**：§7 趋势 4 的反证指出这是本窗口最明确的公开数据空白，而它直接决定预算分配。
- **设计**：固定训练作业与故障率模型，三组配置：①基线存储 + 标准 checkpoint ②基线存储 + 冗余利用（可先实现 AdaCheck 的离线部分：按并行策略识别副本张量）③升级带宽（或加 Tier-0 本地闪存）+ 标准 checkpoint。
- **指标**：checkpoint 暂停时间、恢复时间、**每次评估浪费的 GPU 小时**（对齐字节论文的 16,800 → 4,000 口径）、每 TB 有效 checkpoint 的成本、故障注入下的恢复成功率。
- **决策价值**：给出你自己环境下的等效交换比（多少 GB/s 带宽 ≈ 多少倍去重率），这是任何论文都不会替你回答的问题。

### 实验四（若你在 KV cache 方向）：先做"是否需要 offload"的判定测试
- **动机**：backend.ai 的反证表明在部分场景（前缀常驻 GPU）叠加 offload 净负收益。**先证明需要，再选方案。**
- **方法**：在目标负载上采集 `kv_cache_usage_percent`、`prefix_cache_hit_rate`、驱逐率、有效缓存吞吐（llm-d 已给出这套指标口径，生产暖缓存命中率参考值约 87%）。**若驱逐率接近零，停止——不要引入外部 offload 层。**
- **决策价值**：避免为一个不存在的瓶颈采购一层存储。

### 关注名单

**团队/机构**：
- 上海交大 IPADS（Haibo Chen、Mingkai Dong、Xingda Wei、Rong Chen、Erci Xu）——本窗口内在 FAST/NSDI/OSDI/EuroSys 四会同时高产，且 FalconFS / SwitchFS / AITURBO / KUNSERVE 全部落在本报告范围内。https://ipads.se.sjtu.edu.cn/pub/publication
- 清华存储组（Youyou Lu、Jiwu Shu、Guangyan Zhang）——GCR、MesaFS、Bidaw、DRBoost、OdinANN。https://storage.cs.tsinghua.edu.cn/pub/
- 字节跳动 Seed 基础设施——OSDI '26 最佳论文、MegaScale-Data / MegaScale-MoE / MegaScale-Omni、DisCoGC、SDCHunter、AEGIS；本窗口内**披露密度最高的生产团队**。
- 华为（云 + 存储 + 2012 实验室）——FalconFS、AITURBO、MAIO/PPC、TapeOBS、RDMA connection sharing、形式化方法在华为云可靠性中的实践。
- 中科大 李诚组（Cheng Li）+ Youhui Bai。
- Virginia Tech（Huaicheng Li、Sam H. Noh）——Cylon、WARP，两个开放仿真平台。
- 阿里云存储（Jiesheng Wu、Jiaji Zhu、Zhongyu Wang）+ 上海交大 Erci Xu / Guangtao Xue——本地盘演进、ThinkAhead、Omar、RASK。
- NVIDIA 存储/DPU 线（STX / CMX / DOCA Memos / Dynamo / NIXL / Grove）——本窗口内定义行业结构的一方。
- Glenn K. Lockwood（blog.glennklockwood.com）——本窗口内质量最高的独立存储评论来源，且明确标注自己的立场与雇主关系。
- Oana Balmau（McGill，MLPerf Storage 工作组联席主席）——MinatoLoader 作者兼基准制定者，值得同时跟踪其研究与基准演进。

**项目**：FalconFS、NIXL（含 `nixlbench`）、LMCache、Dynamo KVBM、Xerxes、Cylon、WARP、alibaba/clusterdata GPU trace v2026、SGLang（Strata 已并入）。

**下一个窗口必须覆盖的事件**：MLPerf Storage 下一轮（预计 8 月，将是本报告最大缺口的填补）、FMS 2026（8 月）、SNIA SDC 2026（9/28–30，特别是 object-over-RDMA 去锁定议题）、SC'26（11 月，含 IO500 SC26 榜与 14 篇 LineShine 相关 Gordon Bell 提交）、CMX/STX 设备 2026 H2 出货后的第一批独立测试。

---

## 9. 覆盖与证据缺口

### 时间完整性
本报告为 **year-to-date（2026-01-01 至 2026-07-29）**，非完整年度。四个对本主题最重要的年度事件全部在窗口之后：MLPerf Storage 新一轮、FMS、SNIA SDC、SC'26。**下次做 2026 完整年报时必须重做排序**，本报告的 §3 排名不应被当作年度定论。

### 无法验证或未验证的内容
1. **NVIDIA STX/CMX 的全部性能数字**（5× tokens/s、4× 能效、2× 摄取、CMX 5×）：无基线定义、无拓扑、无缓存状态、无模型与上下文长度、无负载定义。设备 2026 H2 出货，窗口内零第三方复现。**不可用于选型。**
2. **ParaStor F9000**：配置已公开（IO500 submission #803），但元数据处理机制、持久性取舍、EC 布局的实际实现均未披露；无独立测试。Lockwood 提出的三点质疑（EC 跨服务器的延迟含义、格式化容量几乎不留元数据空间、性能最小值型 QoS 的可实现性）在有独立复现前均未被排除。
3. **ObjectCache / Tutti**：预印本，无同行评审、无 artifact 声明。ObjectCache 的作者机构归属未在可检索片段中完整给出，本报告**不做推断**。两者均未评测一致性、驱逐、多租户隔离与故障恢复。
4. **云厂商发布（Google Cloud Next '26、AWS）**：全部为第一方声明。Google 的用户证言（15× 数据获取、-50% 启动时间、中断减少 50%）为客户口述，无方法学披露。
5. **FAST '26 / OSDI '26 / NSDI '26 部分论文**：本报告的技术细节主要来自 USENIX 官方摘要与部分公开 PDF。**§3 中标注"待读正文核实"的点尚未从正文验证**，包括：字节论文预测式复制的失效回退与一致性窗口、FalconFS 混合索引的切换策略、AdaCheck 在线冗余对恢复正确性的影响。OSDI '26 多数论文 PDF 在检索时仍需注册（页面显示 locked 图标）。
6. **视频与时间戳**：未提供任何未经验证的时间戳。多数会议录像为注册者可见。theCUBE 的 VAST Data 访谈为**赞助段落**，其"单 GPU 服务器 10×"说法无方法学披露，不予采信。
7. **开源许可**：表格中大部分项目的具体许可证**未从仓库 LICENSE 文件核实**，仅记录了可确认的版权归属。使用前需自行核实。

### 缺乏证据的主题（不是没找到，是公开证据本身不足）
- **KV cache 层的一致性、多租户隔离、故障恢复与经济学**：本窗口内几乎所有 KV 存储工作都只评测性能。这是整个方向最大的系统性缺口。
- **"投资去重"与"投资带宽"的边际收益比较**（见 §8 实验三）：无任何公开数据。
- **CMX/G3.5 层的耐久性设计**：KV cache 可重算这一属性对 EC/复制策略意味着什么，没有任何公开的设计论证。
- **agent 负载对并行文件系统的破坏性**：ISC'26 上有多个 BOF 提到"AI agent 很擅长把 Lustre 搞挂"（agent 像糟糕的用户那样创建海量文件、反复遍历命名空间，但比人类更擅长并行且无法被用户服务打电话劝停），但这**只是轶事，无任何量化刻画或论文**。这是一个明显的研究空白，也是本报告认为下个窗口最值得关注的新问题。

### 地域与来源类型分布（长窗口审计）

判定依据为**作者当前的机构归属**，不依据姓名、族裔或国籍；跨区域合作单独标注。

| 区域 | 入选/候选项数量与代表 | 主要披露渠道 |
|---|---|---|
| **中国 / 东亚** | 数量与技术深度上占本窗口主导。中国大陆/香港：FalconFS（华为+上交）、OSDI 最佳论文（中科大+字节）、GCR / MesaFS / Bidaw（清华）、AITURBO / SwitchFS / CacheSlide / SolidAttention（上交系）、AdaCheck（国防科大+上海AI Lab）、MegaScale 系列（字节+港大/北大）、DisCoGC（字节+清华）、ThinkAhead / Omar / 本地盘演进（阿里云+上交）、TapeOBS / MAIO（华为）、Xerxes（北大等）、ParaStor F9000（曙光）、CETOFS（中科院计算所）、CoFS（麒麟软件）、Heterogeneity at Hyperscale（港科大+阿里+复旦）。韩国：ZUFS 跨层优化（SK hynix + Google + 首尔大）、ScaleSwap（中央大）、Lockify（成均馆）、DOGI（POSTECH 等）、BASK / MTTM / CofferOS（KAIST）。 | 顶会论文 + 企业实验室 + 开源仓库 |
| **北美** | Apple ACOS（百 EB 级对象存储生产论文）、MOST / HARE（UW-Madison，含 Google）、TCO-driven Storage Provisioning（CMU + Google + Microsoft）、Checkmate（Tufts + MIT）、Strata（Stanford 牵头，多机构）、ZipLLM / DirectKV（UVA + Harvard）、SYMPHONY（UT Austin + UW-Madison）、DroidSpeak（UChicago + Microsoft）、Cylon / WARP（Virginia Tech）、NVIDIA STX/CMX、Google Cloud Next '26、AWS 发布流、PASS / Prediction-Informed Power Management（UW）、A Logically Disaggregated Cache（UIUC）、2DIO（Northeastern）。 | **超大规模厂商的 AI 存储细节主要通过云产品发布而非工程论文披露**；学术侧集中在机制层 |
| **欧洲** | 在**AI 核心数据路径**上产出较少，集中在内核 I/O、缓存、可靠性、能耗与 HPC I/O：uCache / Proteus（TU Munich）、PaCaR（RWTH Aachen + LIP6/Inria）、FUR / Accelerating Transactional Execution via PIM（INESC-ID, Lisboa）、MinatoLoader（INESC TEC & Minho + McGill）、REPS / Zeppelin / RoPeerTo（ETH Zürich，部分与 Microsoft/AMD/Politecnico di Milano 合作）、Untangling GPU Power Consumption（Lyon/Inria/OVHcloud + ETS）、VM live migration（Grenoble/INRIA/Rennes）、Wayfinder（Lancaster/Manchester/UBC/NEC Labs Europe）、Neuro-C（Uppsala/RISE/Politecnico di Milano）、Advancing Data Integrity in Linux（Samsung Semiconductor + EPFL + Hellwig）、ISC 2026（Hamburg）、EuroSys '26（Edinburgh，程序主席之一 André Brinkmann 来自 Mainz，FAST '26 程序共同主席同为 Brinkmann）。 | 系统/HPC 会场 + 内核社区 + 研究实验室 |
| **跨区域合作** | Strata（美+中+多校）、Prism（10+ 机构横跨美/欧/中）、UEP / UCCL-Tran（UC Berkeley + 清华 + AMD/AWS/Broadcom + UPB 罗马尼亚）、OpenTela（ETH + Cambridge + EPFL + MIT + 港科大）、FlexLLM（CMU + Purdue + Anthropic + Mistral AI + Stanford）、WARP（Virginia Tech + Samsung + Western Digital）、Advancing Data Integrity（韩企 + 瑞士 + 独立开发者）。 | — |
| **其他** | MBZUAI（阿联酋；Chun Jason Xue、Qiao Li — UnICom、ColdCode、Xerxes 合作）、NUS / HKUST / CUHK / Technion、KAUST（Marco Canini；同时是 OSDI '26 开放获取赞助方）。 | — |

**关于欧洲覆盖的说明**：这不是检索遗漏。我针对欧洲专门检索了 EuroSys '26 全量录用列表、CHEOPS workshop（EuroSys '26 附属，其议题明确包含"ML/AI 应用的存储需求：LLM、向量嵌入、KV cache 数据、模型训练 checkpoint"）、ISC 2026 全周报道、DAOS / Lustre / JUPITER / EuroHPC / CERN / ECMWF 相关渠道。结论是：**欧洲机构在本窗口内于 AI 存储核心数据路径（KV cache、checkpoint 机制、AI 元数据）上的第一作者产出确实少于中国与北美**，其强项集中在内核 I/O 栈、缓存、持久内存、能效与传统 HPC I/O。**这是真实的证据分布差异，本报告不通过降低技术门槛来凑区域配额。** 需要注意的是欧洲以另一种方式深度参与：会议治理（FAST '26 与 EuroSys '26 的程序主席）、内核上游（Hellwig 的 Linux E2EDP 工作）、以及 Mistral AI 作为 STX 早期采用方与 FlexLLM 合作方。

**来源类型分布**：同行评审论文约 60%（FAST/NSDI/OSDI/EuroSys）；第一方生产结果约 15%（FalconFS、字节数据流水线、AITURBO、Apple ACOS、DisCoGC、PolarStore、TapeOBS、Strata）；厂商声明与产品发布约 15%（NVIDIA、Google Cloud、AWS、Sugon）；预印本约 5%（ObjectCache、Tutti）；独立专家评论约 5%（Lockwood、backend.ai）。**厂商声明占比偏高是本窗口的固有特征**（GTC 与 Cloud Next 均落在窗口内，而 MLPerf 这一唯一的跨厂商可比基准没有新一轮），这是解读本报告时必须持续意识到的偏置。
