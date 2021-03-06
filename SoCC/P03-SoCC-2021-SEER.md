# Title

<!-- 此部分是论文标题-->
Elastic Hyperparameter Tuning on the Cloud

## Website
<!-- 网址，有DOI的建议用DOI地址-->
https://doi.org/10.1145/3472883.3486989 

## Citing
@inproceedings{dunlap2021elastic,
  title={Elastic Hyperparameter Tuning on the Cloud},
  author={Dunlap, Lisa and Kandasamy, Kirthevasan and Misra, Ujval and Liaw, Richard and Jordan, Michael and Stoica, Ion and Gonzalez, Joseph E},
  booktitle={Proceedings of the ACM Symposium on Cloud Computing},
  pages={33--46},
  year={2021}
}
<!-- 引用格式，建议使用latex格式-->

## Brief Introduction

<!-- 通过三五句话描述这篇文章，包括 1. 论文的应用场景；2. 论文克服已有方法的局限性；3. 论文主要的技术手段； 4. 论文的预期结果 -->
本文针对在使用同构的弹性云资源时的深度学习模型超参数自动调优方法。先前的超参数调优工作通常固定了集群资源大小。但是当今的云服务可以提供弹性的资源数量，以GPU时间作为支付单位，本文分析了调优任务的特征，指出了并行训练与找到更优的超参数之间存在trade-off的关系，探索了一种在给定时间和预算的限制下，有效利用了云服务弹性资源数量的特点，制定最合理的调优计划的方法，更容易找到最精确的模型。

## Key Methodology

<!-- 分点写，论述论文中主要技术手段的实施过程 -->
超参数调优通常使用的方法是SH，即Successive Halving。这个方法的大意是：按阶段，每个阶段中训练一批不同超参数的任务，阶段结束后，终止其中一批表现较差的任务，将多出来的资源给更好的任务，再重复这个过程。

1. 先前相关工作

    先前的超参数调优任务受限于一个假设：集群的资源数量是固定的。这就导致资源在空间上是受限的，会导致多个训练任务存在严重的竞争关系。但是云资源通常按照GPU时间进行计费，获得更多的GPU资源是相对容易的。所以本文的调优任务是受限于时间和预算的。

    先前的HyperSched会倾向于为性能更好的训练任务分配更多的并行资源，但是由于大多数DL任务的并行效率一般，分配更多资源并不能带来很好的提升，但是又占用大量资源，导致其他任务不能得到充分训练，可能会导致一些潜在的更优超参数被掩盖埋没。

2. 设计思路

    通常，为了寻找超参数时能够节省资源，一个快速收敛的模型应该给予更多的trails数量，并给予较小规模的GPU资源。而一个收敛较慢的模型，应该给予较少的trails，但给予更多的GPU资源，以便于让它通过并行训练加速，从而能够加速收敛。但由于本系统并没有这些先验知识。所以本系统采取了bracket的方式解决以上问题。

    首先，定义某一个超参数配置的训练为trail。本文用bracket将集群资源进行分组。每个bracket中，可承载一定数量的trail，其中每个trail的占用资源数量是固定的，称该trail占用资源量为该bracket的资源单位。

    如：集群可以分为2个bracket，第一个bracket中所有trail都是占用2个GPU资源，第二个都是占用1个GPU资源。但是每个bracket中的GPU数量并不是有限的，是可以通过云服务进行弹性获取的。

    从高层看，可以将每个bracket看做一次SH任务，所有的bracket中的trails启动并经过同样的一段时间后，认为这个stage到期，随后对所有bracket中的所有trail进行评估，删除一部分训练效果较差的trails。进入下一轮前，将依据trails的loss进行排序。排序后，将那些表现更好的trails放入到资源单位更高的bracket当中，那些表现相对较差的进入到资源单位更低的bracket当中。这样就在每个stage之后，能够使得性能更好的trail获得更多的资源，加速收敛。同时，资源单位较低的bracket通常含有数量更多的trails，基数较大，也容易发现一些收敛较慢但是效果较好的trails。

    给定以上思路，则后面要解决的就是如何在给定时间和预算成本限制的情况下，划分bracket的个数，选择每个bracket的资源单位大小以及可承载的trails数量，选择每个stage的时间长度。在选择每个bracket的资源单位时，本文按照从小到大的顺序，这是因为DL模型在扩容时的性能无法做到线性提升，资源单位越低，则资源利用效率越高。本文通过复杂的数学分析得出了一种最优的指定这些参数的方案，并给出了严密的理论证明。

## Data sets & Experimental Design

<!-- 撰写实验环境的设置，实验的对象，实验的比较方面，以及实验的结果（不要列举数据，要概括谈） -->
- 实验设置
  - AWS p3.8xlarge
  - 每个节点4个V100

- 比较对象
  - 4个固定集群大小的方案：Random，ASHA，HyperSched，BOHB
  - 为公平性，使用2个经过修改后变为弹性集群大小的方案：E-Hyperband，E-Grid Search。

- 比较方案
  - 在三个深度学习应用领域进行比较
  - 比较在给定数据集下，指定同样的截止时间和GPU时间（预算相同），比较最终选择的模型的精度和偏差。
  - 分别固定预算和截止时间的情况下，计算每个方法最终挑选超参数的模型精度。

## Conclusion And Future Work

<!-- 作者或者阅读者对本文工作的总结，以及未来可能的改进方向 -->
将在时间和预算限制下的弹性资源超参数调优问题进行形式化描述，并将之实现于一个资源调度方案之中。它利用了云资源的弹性以便拓展足够大的配置空间，同时能够识别不同级别并行之间的权衡利弊，避免了盲目低效地水平扩展单个trail，而让其他trail失去后来居上的机会。