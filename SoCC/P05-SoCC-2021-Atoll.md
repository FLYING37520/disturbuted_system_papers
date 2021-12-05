# Title

Atoll: A Scalable Low-Latency Serverless Platform

## Website
https://doi.org/10.1145/3472883.3486981

## Citing

@inproceedings{10.1145/3472883.3486981,
author = {Singhvi, Arjun and Balasubramanian, Arjun and Houck, Kevin and Shaikh, Mohammed Danish and Venkataraman, Shivaram and Akella, Aditya},
title = {Atoll: A Scalable Low-Latency Serverless Platform},
year = {2021},
pages = {138–152},
numpages = {15},
keywords = {Scalable, Serverless Computing, Low-Latency},
series = {SoCC '21}
}

## Brief Introduction

在无服务器计算中，现有平台的控制层与数据层难以处理拥有不可预知的到达模式的短生命周期函数，该论文提出了无服务器平台 Atoll，目的是满足应用层面的延迟需求。Atoll 将集群按照“半全局调度器 - worker pool”组合为单位进行切分、使用对 DDL 敏感的调度策略以及沙箱主动分配、对沙箱敏感的路由添加一个负载均衡层、自动对每个应用进行半全局调度器的规模调整。经过实验，现有的无服务器平台 GFR 的期限错过率最高是 Atoll 的66倍，尾延迟约为 Atoll 的3倍。

## Key Methodology

1. 调度机制

	1. 半全局调度器（SGS）
	
	  Atoll 将集群划分为多个 worker pool，每个 worker pool 由若干物理机以及一个 SGS 组成。每个 SGS 负责一组应用，SGS 与应用的对应关系可以在负载均衡服务的管理下粗时间粒度地进行调整。文中将一台机架划分为一个 worker pool。
	
	2. DDL 敏感的调度策略
	
	  使用剩余缓冲时间（请求可以在队列中等待而不超出 DDL 的剩余时间）最小优先算法（SRSF）。如果存在并列的情况，就选择剩余工作最少的请求，以增加调度机会，使超出 DDL 的请求数尽可能少。将含有多个函数的应用组织为 DAG，对于每个 DAG，在根节点函数完成执行后，将关键路径的执行时间作为应用所需的执行时间，以此使用 SRSF 进行调度。
	
	3. 沙箱主动分配
	
		1. 沙箱需求估计
		
		   使用指数加权移动平均（EWMA）对请求的到达速率进行预测，并建立泊松分布模型。再通过逆分布函数求出时间间隔内可能到达的最大请求数。由于存在执行时间大于时间间隔的函数，因此对计算得到的最大值适当按比例增大。
		
		2. 沙箱放置策略
		
		   将一个函数所需的沙箱平均分配到不同的 worker 上，这与现有的方法不同（尽可能将同一个函数的沙箱分配到同一个 worker 上）。具体方法：对于给定的函数，寻找放置其沙箱最少的 worker 并放置其沙箱。这种方式提高了统计复用性，减少了放置沙箱时出现冷启动的可能。
		
		3. 沙箱的驱逐机制
		
		   - 软驱逐：当 1 中对沙箱需求的估计值减少后，对函数中多余的沙箱进行软驱逐。与放置策略正相反，软驱逐策略选择拥有当前函数沙箱最多的 worker，对其沙箱进行软驱逐（只进行标记）。
		
		   - 硬驱逐：当新的沙箱要被放置在内存池饱和的 worker 上时，根据其占用空间的估计，对当前占用空间相近的沙箱进行硬驱逐。

2、负载均衡

1. 功能

   根据 SGS 对集群的划分，平衡不同 SGS 之间的负载。执行沙箱敏感的路由策略：对请求进行路由，使从沙箱主动分配中受益的请求数最大化。

2. 对每个应用的 SGS 数量进行调整

   1. 判断调整时机
   
      每个 SGS 使用 EWMA 来测量每个应用在一个时间段内的延迟，当 SGS 对 LBS 做出响应时，一并发送该测量值。LBS 通过这些接收的信息对 SGS 的数量进行调整。
   
    2. 调整机制
   
       - SGS 的扩张（数量增加）
       
         LBS 收集来自不同 SGS 的关于一个应用的请求的延迟信息，根据其 DDL 进行标准化得到一个调整指数（scaling metric），如果该值大于扩张阈值，则 LBS 将使用一致性哈希来将下一个 SGS 与当前应用相连接。
       
       - SGS 的缩减（数量减小）
       
         当延迟低于缩减阈值时，LBS 将根据后入先出的原则移除 SGS。
       
       - 调整指数的计算
       
         根据每个 SGS 上主动分配的沙箱数作为权重，对 SGS 的排队时延进行加权求和，然后根据应用的剩余缓冲时间进行标准化。
       
       - 调整过程的透明化（逐渐调整，而不是立即调整）
       
         在扩张时，使用彩票调度策略。为新的 SGS 分配小票数，使请求逐渐到达新的 SGS。并令新的 SGS 主动分配现有 SGS 的平均沙箱数。
       
         在缩减时，对一个应用使用两个 SGS 列表：活跃列表与移除列表。缩减时，将 SGS 从活跃列表移动到移除列表中，而后进行扩张时，为移除列表中的 SGS 根据折现系数分配一个较小的票数。

## Data sets & Experimental Design

实验数据：trace-driven workload from Microsoft（开源）、两个合成的 workload

实验环境：74结点的集群，每个节点为 256GB & 10Gbps NIC，分为8个 SGS

实验对象：GFR、GDR、GDPI、Atoll

比较方面：DDL 错过率、端到端时延

宏基准测试：对于不同类型的 workload，Atoll都拥有最低的DDL 错过率与最低的尾延迟，且沙箱主动分配与软驱逐对性能提升的贡献最大。

微基准测试（缩小集群规模）：随着工作负载的增加，沙箱的平均传播带来的DDL 错过率改善更明显。硬驱逐的尾延迟更低。逐渐调整 SGS 的尾延迟更低。LBS 对拥有较小 slack 的应用有着更为激进的调度（SGS 的调整更明显）。LBS 对应用间的争用敏感，在争用激烈时为被争用的低速率应用扩张更多的 SGS。

 对沙箱分配开支的影响：在不同的开支下，都有最低的 DDL 错过率与尾延迟。

内存压力下的主动分配沙箱池：模拟负载峰值时，主动分配的内存池完全不足以为所有函数分配足够沙箱的情况。减少内存池大小，会使 DDL 错过率与尾延迟上升，但是始终低于基线水平。

敏感度分析：对于扩张阈值，存在一个 DDL 错过率与尾延迟的 tradeoff。细粒度的 SGS 划分会提高 DDL 错过率与尾延迟。但是，拥有过多 worker 的 SGS 会因为过高的调度开销而提高排队时延，并且导致 LBS 不必要地扩张 SGS。


## Conclusion And Future Work

该论文提出了一种注重低时延、低 DDL 错过率的无服务器平台。在对于多函数应用调度的部分（即用到 DAG）的部分的执行时间计算比较模糊，也没有考虑到节点之间的依赖关系。