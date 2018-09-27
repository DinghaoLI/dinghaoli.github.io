---
layout: post
title:  "《分布式系统》读书笔记 - 第一章 分布式系统的特征"
date:   2018-09-25 03:00:00 +0800
categories: DistributedSystems
tags: DistributedSystems

---


## 第一章 分布式系统的特征

### 1.分布式系统特征：

- 并发性
- 缺乏全局时钟
- 故障独立性

### 2.共享的模式不同

- 一种是“无知型”分布式系统：如Google Web搜索引擎给所有用户提供统一的界面，用户不关心系统架构。

- 另一种是”计算支持协同工作“型（Computer Supported Cooperated Working）：合作用户在封闭小组内协同工作，这需要可靠和一致的访问和更新。如共享文档，共享操作接口等。如Google Doc文件的协作。

### 3.分布式系统的挑战

- 异构性（网络，硬件，系统，编程语言等）   

	可利用中间件去消除异构，中间件是指一个软件层，它提供了一个编程抽象，同时屏蔽了底层网络、硬件、操作系统、编程语言的异构性。大多数中间件在在互联网协议上实现，由这些协议层屏蔽了底层网络的差异。
	或用虚拟机的方式解决，如JAVA。
	
- 开放性（拓展方式，开放接口等）

- 安全性（机密、完整、可用）

- 可伸缩性（如果资源数量激增和用户数量激增，系统仍能保持其有效性）

	- 控制物理资源的开销：希望达到O(n)的伸缩性（现实中很难），一个服务器支持20个用户，那么希望2台机器能支持40个用户。
	
	- 控制性能损失：比如系统查询amazon的DNS，层次结构（树）的伸缩性比线性结构算法好。能达到O(log n)。但是即使使用层次结构，数量的增加也会导致性能损失。
	
	- 防止软件资源用尽：如Ipv4的设计就容易用尽。

	- 避免性能瓶颈：算法应该是分散型的，以避免性能瓶颈。如域名系统的分区，Web服务的缓存复制。

- 故障处理
	- 掩盖故障：考虑最坏的情况
	- 容错：利用冗余
	- 故障恢复：回滚数据
- 并发性   

	支持同时访问，使对象在并发环境中保持正确性。
	
- 透明性：
	
	透明性被定义成用户和应用程序员屏蔽分布式系统的组件的分离性，使系统被认为是一个整体，而不是独立组建的集合。
	- 访问透明性：相同的操作访问本地/远程资源 ✨
	- 位置透明性：无需知道资源的物理/网络位置 ✨
	- 并发透明性：几个进程能并发地使用共享资源进行操作且互不干扰。
	- 复制透明性：使用资源的多个实例提升可靠性和性能，用户和程序员无需知道相关信息。
	- 故障透明性：屏蔽错误
	- 移动透明性：资源和客户能够在系统内移动（怎么算移动？）
	- 性能透明性：负载变化，系统可以重新配置以提高性能
	- 伸缩透明性：系统和应用可以拓展但是不改变系统结构和应用算法

- 服务质量
	对服务访问质量的保障






















