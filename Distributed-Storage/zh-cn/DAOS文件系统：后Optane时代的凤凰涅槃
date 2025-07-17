## 前言：IO500上的屠榜

因为看到DAOS项目在IO500上的屠榜，所以一直对这个项目保持着关注(未深入)。

![image.png|IO500](../../image/io500.png)

## 背景

在之前的文章中，简要介绍了DAOS分布式存储项目。

然而，随着Intel在2022年终止Optane业务，很多童鞋都开始关心：**DAOS在失去"核心硬件支撑"后，是否还能继续？它的未来在哪里？**

## 短期冲击，但没有终结

Optane的停产确实对DAOS产生了不小影响，尤其是在元数据加速和持久性方面。但DAOS并没有被困于这场"硬件地震"，而是通过架构上的调整实现了"再生"：

- **引入写前日志机制（WAL）**：通过将日志写入DRAM并同步到NVMe SSD，以替代原本依赖PMem的低延迟持久化路径
- **持续优化NVMe访问路径**：使在没有Optane的环境下也能维持高IOPS和低延迟

![image.png|IO500](../../image/daos01.png)

## 开源社区的力量：从Intel主导到基金会治理

2023年11月，**DAOS基金会**正式成立，由Argonne国家实验室、Enakta Labs、Google、HPE和Intel等机构联合支持。该基金会设有技术指导委员会，由HPE的高级技术专家Johann Lombardi担任主席，他是DAOS的长期开发者和倡导者。

这一转变意味着：

- DAOS从一个由Intel主导的项目，过渡为中立的开源生态
- 吸引了更广泛的用户和贡献者，推动开发路线更加公开透明
- 增强了对未来持续演进的信心

### 开源指标：DAOS vs. Ceph

![image.png|IO500](../../image/star.png)


![image.png|IO500](../../image/company.png)


## 面向未来：2.8版与3.0路线图

**DAOS v2.6**在2024年7月Release，是最后一个英特尔发行版。

根据最新发布信息：

- **DAOS 2.8版本**（2025年底）：这是从Intel主导转为社区主导的第一个Release版本，持续增强NVMe适配、元数据引擎与K8s云原生支持
- **DAOS 3.0版本**（预计2026年）：将带来更彻底的模块化设计，强化与AI和容器生态的集成

![image.png|IO500](../../image/roadmap.png)

## DAOS典型用户案例

### 1. Aurora 超级计算机（美国能源部 Argonne 国家实验室）

- **部署规模**：Aurora 是美国能源部支持的百亿亿次级（exascale）超级计算机，DAOS 被用作其主要存储系统之一
- **应用场景**：支持大规模 AI 模型训练、科学模拟和数据分析任务
- **技术亮点**：DAOS 提供高 IOPS 和低延迟性能，满足 Aurora 对高性能存储的严格要求

### 2. SuperMUC-NG 超级计算机（德国 Leibniz 超算中心）

- **部署规模**：SuperMUC-NG 是德国领先的超级计算平台之一，DAOS 被集成用于高性能计算任务
- **应用场景**：支持科学研究和工程模拟等高性能计算需求
- **技术亮点**：DAOS 的高并发访问能力和可扩展性使其成为 SuperMUC-NG 的理想存储解决方案

### 3. Google Cloud

- **应用场景**：Parallelstore 适用于多种高性能计算和 AI 训练场景，底层采用的DAOS
- **技术亮点**（在约100TB的部署规模下）：
  - **吞吐量**：高达 ~115 GiB/s
  - **IOPS**：读取约 300 万次/秒，写入约 100 万次/秒
  - **延迟**：约 0.3 毫秒

## DAOS当前的困境与不足

DAOS仍在多个顶尖超级计算平台上运行，如Argonne的Aurora系统、德国Leibniz超算中心的SuperMUC-NG等。但DAOS也面临新的挑战：

### 不支持NVIDIA GPUDirect

在AI培训和推理领域，以NVIDIA GPU服务器为主导，大多数存储硬件和软件的供应商都支持GPUDirect协议，但是DAOS目前并不支持，这使得其在AI训练/推理场景中与WEKA等竞争对手存在差距；但是，DAOS支持了RDMA。

### 现代部署模式支持不足

针对容器、Kubernetes等现代部署模式的支持仍需加强。

### 用户量相对较少

与Ceph、Lustre等知名开源软件相比，DAOS当前的用户主要还是科研院所以及HPC用户，用户规模还不够多，用户所属行业也不丰富。

## DAOS资料汇总

- **[DAOS WIKI](https://daos-stack.github.io/)**：内容非常丰富
- **[DAOS Slack](https://daos-stack.slack.com/)**：社区交流平台
- **[DAOS 官网](https://daos.io/)**：官方网站
- **[DAOS Github](https://github.com/daos-stack/daos)**：源代码仓库

## 总结

DAOS并没有因Optane的终结而衰落，反而因"断奶"而加速独立。通过技术演进与社区支持，它正逐步摆脱对特定硬件的依赖，转向更普适、更强健的发展路径。

不得不说DAOS还是一个非常年轻的项目，还有许多不足，比如用户规模还不够，功能还不够丰富等等。但是相比Ceph等系统，DAOS的架构和性能优势还是很明显的。

DAOS的未来发展会怎么样呢？我们且骑驴看唱本……

## 参考文献

1. [DAOS WIKI](https://daos-stack.github.io/)
2. [DAOS Slack](https://daos-stack.slack.com/)
3. [DAOS 官网](https://daos.io/)
4. [DAOS Github](https://github.com/daos-stack/daos)
5. [DAOS file system lives on after Optane](https://zhuanlan.zhihu.com/p/1910394685379282773)
