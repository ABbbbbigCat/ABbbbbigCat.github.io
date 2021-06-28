# 论文笔记——Large Scale Incremental Learning


# 简介

本文发表于[CVPR 2019](https://arxiv.org/abs/1905.13260)，代码[见此](https://github.com/wuyuebupt/LargeScaleIncrementalLearning)。

本文在回溯策略上同 iCaRL 一样遵循了 herding 策略，主要致力于解决 rehearsal strategy 中新旧类不平衡问题，提出在全连接层后面增加偏置纠正（Bias Correction，简称**BIC**）。其公式如下：

$$q_k=\begin{cases} o_k & 1 \leqslant k \leqslant n \\ \alpha o_k+\beta & n+1 \leqslant k \leqslant n + m \end{cases}$$

式中，$o_k$代表全连接层输出的logits，$\alpha$和$\beta$是可学习的偏置参数。当优化偏置参数时，本文冻结了特征提取层和全连接层使用一个很小validation set（从所有可获得的样本中分割出来的部分，**保证新旧类样本平衡**）来调整偏置参数，其损失函数如下：

$$L_b=-\sum_{k=1}^{n+m} \delta_{y=k}log[softmax(q_k)]$$

> 按照原文描述，之所以选择筛选出validation set用以偏执纠正层的学习，是为了将validation set从特征表示中排除，这**或**能使其更好的代表新旧类之间在特征空间上的无偏分布。当然，为了保证纠正层训练的无偏性，validation set中新旧类的数据量需要保持平衡。
>
> > 原本在 Fig 5 中使用了 <font color='red'>may better</font>，这样的用词颇为鸡贼，这说明作者无法确定（或无法证明）validation set是否可以保证无偏。另一方面，validation set保证无偏对于最终的结果似乎并非好事，因为在此前的表示学习中由于新旧数据不平衡的问题，模型已经偏向了新类。因此，纠正的时候与其强调无偏，更应该强调将表示学习中的偏置给纠正回来。
> >
> > 这或许也说明了BIC为何在CIFAR100上最终效果几无提升。

# 模型结构

![image-20210628164802414](https://i.loli.net/2021/06/28/nFOE6HZyGiRqDzr.png)

# 实验结果

![image-20210628172505880](https://i.loli.net/2021/06/28/cmyMU5r817lL3hp.png)

![image-20210628172212935](https://i.loli.net/2021/06/28/mJrICi2jYZo5tu9.png)

(备注：ImageNet准确率高是因为取的是**Top-5准确率**)

> 从上述实验结果不难看出，BIC在小数据集如CIFAR上的提升几乎没有，但在ImageNet这样大数据集上却有比较明显的提升。作者们因此认为BIC更适合于Large Scale数据集。
>
> 但实际上，在ImageNet上所采用的是Top-5准确率（有些离谱的是top-5并没有在原文中出现，笔者是在对应代码的描述中确定的），这样的评价标准虽然可以表明BIC使预测更接近真实的结果，却并不能代表实际的准确率。




