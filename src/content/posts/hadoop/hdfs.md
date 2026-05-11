---
title: HDFS深入理解
published: 2026-05-11
description: HDFS的架构设计和核心概念
tags: [HDFS, Hadoop, BigData]
category: BigData
draft: true
---

# HDFS深入理解

## 学习重点

|知识点|学习重点|
|---|---|
|NameNode|理解它是如何管理”元数据”（文件名、目录结构、块位置）的，以及它如果挂了怎么办。|
|DataNode|理解它如何定时向 NameNode 打报告（心跳机制）。|
|数据流|重点看 “写数据” 的流程：客户端是怎么跟 NameNode 通信，然后直接把数据流推给 DataNode 的。|

主要目标：搞懂Block
BlockID：文件的唯一块编号
Block Pool ID：属于哪个命名空间
Nodes： 放在哪个容器里

创建一个目录：
hdfs dfs -mkdir -p /user/zhangjia/test

上传一个文件：
hdfs dfs -put /etc/hosts /user/zhangjia/test/my_first_file.txt

读取文件内容：
hdfs dfs -cat /user/zhangjia/test/my_first_file.txt


# HDFS 深度解析：从 Docker 实战到分布式存储内核

> **整理人：** Gemini (协作伙伴：Zhang Jia)
> **日期：** 2026-04-15
> **核心主题：** 揭秘分布式文件系统的“容错”与“管理”哲学

---

## 一、 HDFS 的物理基石：Block（数据块）

在 HDFS 中，无论文件多大，都会被切成固定大小的 **Block**（默认 **128MB**）。

* **为什么要切块？** 突破单机硬盘容量限制；方便备份（副本机制）；极大地提高并行计算能力（MapReduce 可以并行读取块）。
* **为什么是 128MB？** 这是一个权衡点。太小会导致元数据过多撑爆 NameNode 内存；太大会导致计算任务并行度不够。
* **物理真相：** 在 DataNode 的磁盘上，Block 就是一个普通的 Linux 二进制文件。



---

## 二、 架构角色：谁在管理这个仓库？

HDFS 采用 **主从架构 (Master/Slave)**，分工明确：

1.  **NameNode (NN - 大管家/大脑)**：
    * 存储**元数据**（文件路径、文件权限、块 ID 列表）。
    * **不存实际数据**。它是集群的唯一入口，其内存大小决定了集群能支持的文件总量。
2.  **DataNode (DN - 仓库管理员/肌肉)**：
    * 负责存储实际的 Block。
    * 负责执行客户端的读写请求。
3.  **Secondary NameNode (SNN - 秘书)**：
    * **误区纠正**：它不是 NN 的热备。它只负责定期合并日志（Checkpoint），防止 NN 重启时因加载日志过长而瘫痪。
4.  **Standby NameNode (HA 模式下的备 NN)**：
    * 在启用高可用（HA）时存在。它是“热备”，时刻同步主 NN 状态，秒级接班。

---

## 三、 硬核原理：数据是如何读写的？

### 1. 写入流水线 (Write Pipeline)
当你执行 `hdfs dfs -put` 时，数据采用**“接力赛”**模式传输：
* **Pipeline 建立**：客户端向 NN 请求，得到一组 DN 名单（如 DN1, DN2, DN3）。
* **串联传输**：客户端只发给 DN1，DN1 传给 DN2，DN2 再传给 DN3。
* **ACK 回传**：只有当三个节点都写成功并逐级回传确认信号后，客户端才认为写入成功。
* **容错处理**：若 DN2 挂了，客户端会联系 NN 剔除它，剩下的节点继续传。副本数由 NN 在事后异步补齐。



### 2. 寻址读取 (Read)
* NN 会返回文件所有 Block 的物理位置列表。
* **就近原则**：为了效率，客户端会优先连接距离自己网络拓扑最近的 DN 进行读取。

---

## 四、 核心机制：脑与手的通信

### 1. 元数据的秘密 (FsImage & Edits)
NameNode 的账本由两部分组成：
* **FsImage**：整个文件系统的“静态快照”。
* **Edits Log**：记录所有增量操作的“流水账”。
* **原理**：平时只写 Edits（快），重启或 Checkpoint 时合并到 FsImage（准）。

### 2. 肌肉的回报 (Heartbeat & Block Report)
* **心跳 (Heartbeat)**：DN 每 3 秒打卡。NN 在回执中下达指令（如：删除某块、复制某块）。
* **块汇报 (Block Report)**：DN 定期报告手里所有的块。**NN 内存中“块与节点的映射关系”完全靠 DN 的汇报动态构建，不持久化到磁盘。**

---

## 五、 灾难预防：高可用 (HA) 机制

现代 HDFS 通过 **Active/Standby** 模式消除单点故障：

* **ZooKeeper (ZKFC)**：利用临时节点锁实现自动选主，监控 NN 健康。
* **JournalNodes (共享日志)**：作为主备 NN 之间的“中间账本”，确保数据实时同步。
* **隔离 (Fencing)**：防止“脑裂”（两个大脑同时工作）。备 NN 上位时会强杀（Kill）或切断电源（STONITH）来隔离旧主 NN。



---

## 六、 总结：HDFS 的设计哲学

当你通过 `fsck` 看到 **Under-replicated** 时，不要惊慌。HDFS 的核心逻辑是：
1.  **期望状态 (Desired)**：通过配置设定的目标副本数。
2.  **当前状态 (Actual)**：DataNode 实时汇报的物理副本数。
3.  **最终一致性**：只要还有 1 份数据活着，系统就是健康的。NN 会在后台自动调度，直到 Actual = Desired。

---