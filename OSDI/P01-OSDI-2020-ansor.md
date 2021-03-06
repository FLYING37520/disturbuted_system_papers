# Title

<!-- 此部分是论文标题-->
Ansor: Generating high-performance tensor programs for deep learning

## Website
<!-- 网址，有DOI的建议用DOI地址-->
https://www.usenix.org/conference/osdi20/presentation/zheng

## Citing

@inproceedings{zheng2020ansor,
  title={Ansor: Generating high-performance tensor programs for deep learning},
  author={Zheng, Lianmin and Jia, Chengfan and Sun, Minmin and Wu, Zhao and Yu, Cody Hao and Haj-Ali, Ameer and Wang, Yida and Yang, Jun and Zhuo, Danyang and Sen, Koushik and others},
  booktitle={14th $\{$USENIX$\}$ Symposium on Operating Systems Design and Implementation ($\{$OSDI$\}$ 20)},
  pages={863--879},
  year={2020}
}

## Brief introduction

<!-- 通过三五句话描述这篇文章，包括 1. 论文的应用场景；2. 论文克服已有方法的局限性；3. 论文主要的技术手段； 4. 论文的预期结果 -->
无需人工指定模板，自动生成高性能张量程序。先前基于模板的搜索方法需要大量人工，而基于决策序列的方法不够准确。ansor将张量程序的高层结构与底层优化参数进行分离表示，从而覆盖了极大的搜索空间。通过在该空间随机采样，再结合代价模型和进化搜索进行细化调优，最终得到高性能的、针对目标平台的张量程序。

## Key Methology

<!-- 分点写，论述论文中主要技术手段的实施过程 -->
自顶向下，分三大模块：
1. 任务调度器。
   
   位于架构顶端，将计算图切分为子图，并在宏观上控制子图的优化方案。
   保证对热点子图的重点优化，采取梯度下降的方法，选择最需要被优化的子图。
2. 程序采样器。
   
   每次针对一个子图执行采样任务，生成一批待调优的程序。
   对于一个子图对应的张量程序的搜索空间，而Ansor首先将张量程序按照高层的循环结构与底层的优化参数进行了分离解耦表示：高层的结构类别很少，而底层的参数很多。Ansor通过递归地应用一组推导规则来自动地扩展搜索空间（推导规则的设计较复杂），构建出庞大的搜索空间。随后通过随机采样，从而覆盖整个搜索空间。对采样出的高层程序结构随机地安上底层的优化参数，得到一批待调优的程序。
3. 性能微调器
   
   从程序采样器得到一批程序。微调器的过程是迭代的，每次迭代使用进化搜索生成一批程序，并根据学习到的代价模型找到一批更好的程序，随后在实机运行测量，并根据运行结果再优化该代价模型。进化搜索的变异针对tile size, parallel等优化参数，不改变实际运行效果。进化搜索的交叉互换是计算图中的节点级别进行的，大概率避免了依赖问题。代价模型采取XGBoost等梯度提升决策树实现。


## Data sets & Experimental Design

<!-- 撰写实验环境的设置，实验的对象，实验的比较方面，以及实验的结果（不要列举数据，要概括谈） -->
模型：
ResNet-50
Mobilenet-V2
3D-ResNet
DCGAN
BERT

1. 针对生成的张量程序的性能：
    分三个层次：
    - 单个算子
    - 子图
    - 整个神经网络
    均优于AutoTVM。
2. 针对生成张量程序时所耗的搜索时间：
    比AutoTVM快一点点，仍需花费数个小时。
3. 针对性能微调器中使用的可学习的代价模型的预测质量


## Conclusion And Future Work

<!-- 作者或者阅读者对本文工作的总结，以及未来可能的改进方向 -->
1. 无法支持图中具有动态参数shape的优化。
2. 生成搜索空间时使用的推导规则仅能有效支持密集计算型的算子。对于稀疏的算子，希望能够重用大部分ansor的工作，并设计更优秀的搜索空间。
3. Ansor仅能支持高层的优化，依赖于其他的底层代码生成器。