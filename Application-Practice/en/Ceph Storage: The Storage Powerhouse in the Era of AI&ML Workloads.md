# Ceph Storage: The Storage Powerhouse in the Era of AI/ML Workloads

**Abstract**  
AI/ML training, inference, and related processes place unprecedented demands on storage performance.  
This article, based on the SNIA presentation *"Ceph Storage in a World of AI/ML Workloads"*, analyzes the challenges of AI storage, the advantages of Ceph, and key methods to improve efficiency in real deployments.

---

## AI/ML Workload Lifecycle

A typical AI/ML lifecycle includes:

> Raw Data → Training Data → Model → Results → Retraining

During training, network bandwidth, data preprocessing capability, and model size all affect overall performance.  
In practice, the recommended storage throughput is **5 GB/s**, with high-performance reference systems reaching **20 GB/s**.

---

## Checkpointing Challenges

Checkpoint saving is a critical step in AI model training, and its data size grows rapidly with model scale:

* **Granite 13b**: 170 GiB, ~5 seconds  
* **Llama3 70b**: 913 GiB, ~28 seconds  
* **GPT-3 175b**: 2.28 TiB, ~70 seconds  
* **Llama3 405b**: 5.28 TiB, ~162 seconds  

Storage system performance directly determines checkpoint save speed, impacting overall training efficiency and cost.

---

## Storage Requirements for Inference

In recommendation systems or event-driven inference scenarios (e.g., Facebook data centers):

* Deep recommendation models consume **80%+** of inference compute
* Consume **50%+** of training compute

These scenarios require storage systems with **high concurrent I/O** and **low-latency response**.

---

## Why Choose Ceph?

Ceph offers significant advantages in AI/ML storage scenarios:

* **Multi-protocol support**: Block, File, and Object (e.g., S3, NFS, SMB)
* **Hardware agnostic**: No vendor lock-in, flexible CPU, memory, network, and media choices
* **High scalability**: From small deployments to hundreds of nodes, read throughput can grow from 20 GB/s to 160 GB/s
* **Mature open-source ecosystem**: Proven in production, active community, broad vendor support

---

## Strategies to Improve Ceph Storage Efficiency

* **Compression & hardware acceleration**  
  Ceph RGW and Bluestore support data compression.  
  For example, S3 object compression can boost write throughput by **250%+** and read throughput by **180%+**.

* **Thoughtful architecture design**  
  Planning compression strategies and hardware acceleration early can significantly reduce TCO (Total Cost of Ownership).

---

## Real-World Deployment Example

SNIA reference deployment:

* **4-node Ceph cluster**
  * Each node: 2×32-core CPUs, 512 GB RAM, 2×100 GbE network
  * Storage: 24× TLC NVMe SSDs
  * Acceleration: 4× GPUs
* **Performance**: 30 GB/s read, 4.66 GB/s write

This shows Ceph can fully meet AI/ML training and inference storage demands under high-performance hardware conditions.

---

## Community & Events

* **Ceph Days**: 2025 in Bangalore, San Jose, London, and more
* **Cephalocon**: 2024 at CERN, 2025 in planning
* **SNIA Education Library**: Rich resources on Ceph and AI/ML technologies

---

## Key Takeaways

1. **Maximize GPU utilization** with storage systems that match required bandwidth and throughput
2. **Network planning** is the foundation for scaling to meet AI/ML throughput needs
3. **Open-source flexibility** enables Ceph to quickly integrate acceleration and optimization features

---

## References

* SNIA: *Ceph Storage in a World of AI/ML Workloads*  
  [https://snia.org/sites/default/files/CSI/Ceph%20Storage%20in%20a%20World%20of%20AI_ML%20Workloads.pdf](https://snia.org/sites/default/files/CSI/Ceph%20Storage%20in%20a%20World%20of%20AI_ML%20Workloads.pdf)
* Ceph Official Documentation: [https://docs.ceph.com](https://docs.ceph.com)
* SNIA Education Library: [https://snia.org/education](https://snia.org/education)
* Facebook AI Research: Deep Learning Recommendation Models (DLRM)
* Cephalocon: [https://cephalocon.org](https://cephalocon.org)
