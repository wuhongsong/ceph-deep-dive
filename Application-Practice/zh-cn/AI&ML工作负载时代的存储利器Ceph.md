

**摘要**
AI/ML 训练、推断等环节对存储提出了前所未有的高性能要求。本文结合 SNIA《Ceph Storage in a World of AI/ML Workloads》演示文稿内容，分析了 AI 存储的挑战、Ceph 的优势，以及在实际部署中提升效率的关键方法。

---

## AI/ML 工作负载生命周期

一个典型的 AI/ML 生命周期通常包括：

> 原始数据 → 训练数据 → 模型 → 结果 → 再训练

在训练过程中，网络带宽、数据预处理能力、模型规模都会影响整体性能。
在实际案例中，推荐的存储吞吐量为 **5 GB/s**，高性能参考系统可达 **20 GB/s**。

---

## 检查点（Checkpointing）挑战

检查点保存是 AI 模型训练的关键步骤，其数据量会随着模型规模迅速增加：

* Granite 13b：170 GiB，约 5 秒
* Llama3 70b：913 GiB，约 28 秒
* GPT-3 175b：2.28 TiB，约 70 秒
* Llama3 405b：5.28 TiB，约 162 秒

存储系统性能直接决定了保存检查点的速度，从而影响整体训练效率和成本。

---

## 推断（Inference）中的存储需求

在推荐系统或事件驱动的推断场景中（如 Facebook 数据中心）：

* 深度推荐模型占推断计算使用 **80%+**
* 占训练计算使用 **50%+**

这类场景需要存储系统具备**高并发 I/O**和**低延迟响应**的能力。

---

## 为什么选择 Ceph？

Ceph 在 AI/ML 存储场景中有明显优势：

* **多协议支持**：同时提供 Block、File、Object（如 S3、NFS、SMB）
* **硬件中立**：无厂商锁定，可自由搭配 CPU、内存、网络、存储介质
* **可扩展性强**：从小规模部署扩展到数百节点，读吞吐可从 20 GB/s 提升至 160 GB/s
* **开源生态完善**：生产环境验证成熟，社区活跃，厂商支持广泛

---

## 提升 Ceph 存储效率的策略

* **压缩与硬件加速**
  Ceph RGW 与 Bluestore 支持数据压缩。
  例如 S3 对象压缩，可使写入吞吐提升 **250%+**，读取提升 **180%+**。
* **合理架构设计**
  在系统初期规划好压缩策略、硬件加速方式，可显著降低 TCO（总体拥有成本）。

---

## 实际部署案例

SNIA 提供的参考部署案例：

* **4 节点 Ceph 集群**

  * 每节点：2×32 核 CPU、512 GB 内存、2×100 GbE 网络
  * 存储：24 块 TLC NVMe SSD
  * 加速：4× GPU
* **性能表现**：读取 30 GB/s，写入 4.66 GB/s

该结果展示了 Ceph 在高性能硬件条件下，完全可以胜任 AI/ML 训练与推断的存储需求。

---

## 社区与活动

* **Ceph Days**：2025 年在班加罗尔、圣何塞、伦敦等地举办
* **Cephalocon**：2024 年在 CERN 举办，2025 年活动正在筹备中
* **SNIA 教育库**：提供大量 Ceph 与 AI/ML 相关技术资料

---

## 核心要点总结

1. **高效 GPU 利用率** 需要存储系统匹配足够带宽与吞吐能力
2. **网络规划** 是扩展的基础，确保 AI/ML 吞吐需求得到满足
3. **开源灵活性** 让 Ceph 能快速整合加速与优化功能

---

## 参考文献

* SNIA：《Ceph Storage in a World of AI/ML Workloads》
  [https://snia.org/sites/default/files/CSI/Ceph%20Storage%20in%20a%20World%20of%20AI\_ML%20Workloads.pdf](https://snia.org/sites/default/files/CSI/Ceph%20Storage%20in%20a%20World%20of%20AI_ML%20Workloads.pdf)
* Ceph 官方文档：[https://docs.ceph.com](https://docs.ceph.com)
* SNIA 教育库：[https://snia.org/education](https://snia.org/education)
* Facebook AI Research：Deep Learning Recommendation Models (DLRM) 研究
* Cephalocon 大会官网：[https://cephalocon.org](https://cephalocon.org)

---
