# 增量学习论文笔记——iCaRL: Incremental Classifier and Representation Learning


# 简介

本文是增量学习的经典论文，发表于[CVPR 2017](https://arxiv.org/abs/1611.07725)，基于pytorch的代码在[此处](https://github.com/donlee90/icarl)。

本文是rehearsal策略的经典算法，其核心点主要是：第一，分类采取了NME（nearest-mean-of-exemplars）；第二，基于herding的优先级筛选样例（保留距离均值更近的样例）；第三，使用知识蒸馏和原型预演进行表征学习。

> 知识蒸馏由Hinton团队提出，最初用于模型间的知识迁移。由于另外一篇经典的增量学习论文[LwF](https://link.springer.com/chapter/10.1007/978-3-319-46493-0_37)，知识蒸馏被用到增量学习中。
>
> NME相较于nearest-class-mean(NCM)的优势在于它不需要保存所有的原始样本，而是通过计算样本的平均特征向量后。
>
> herding 策略是 rehearsal 中很常见的策略，它主张选择最靠近均值的数据以确保样例集能够代表数据的核心知识。

值得一提的是，本文采用了**固定样例集大小**的策略来节省样例集存储空间。具体而言，对于CIFAR-100，iCaRL固定了样例集大小 $m = 2000 $，每当有新类加入，样例集中保存的各种类的数目都会按比例缩小。其筛选的标准还是基于与**均值的距离**。

> 固定样例集大小的规则目前已经被广泛采纳，对于回溯而言，许多研究者们默认了采用$m=2000$的标准，例如[Mnemonics Training](https://arxiv.org/pdf/2002.10211.pdf)

# 训练过程

![image-20210619022437424](https://i.loli.net/2021/06/19/D28mMg9qtONGlre.png)

**Algorithm 1** 给出了iCaRL的增量训练过程，**Algorithm 3** 给出了iCaRL如何进行表示学习

**模型**：32-layer resnet (For CIFAR-100); 在特征提取部分使用CNN网络，然后是单个分类层，其 sigmoid 输出节点与迄今为止观察到的类一样多。对于任意的类 $y\in{1,...,t}$，网络的输出结果为（sigmoid层用于模型的损失函数构造，让模型参数得以训练，实际分类则采用NME）：

$$g_y(x)=\frac{1}{1+exp(-\alpha(x))} \quad with \quad \alpha_y(x)=w_{y}^t\varphi(x)$$



![image-20210619024612911](https://i.loli.net/2021/06/19/BpoujyxNHQIgSTf.png)

# 实验结果

![image-20210619025100005](https://i.loli.net/2021/06/19/vfIwTay9RFLJ3Nj.png)

> [Kuzborskij](https://ieeexplore.ieee.org/document/6619275)等人的研究表明，在向现有的线性多类分类器中添加新类时，只要能够从所有类的**少量数据**中重新训练分类器，就可以避免精度损失。这说明了Rehearsal在缓解灾难性遗忘方面的效果。
>
> 基于Rehearsal的研究主要有两各分关注点，其一是如何让少量数据更好的发挥作用，其二是如何筛选少量数据。


