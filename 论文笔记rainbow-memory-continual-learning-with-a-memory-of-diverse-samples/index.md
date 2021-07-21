# 论文笔记——Rainbow Memory: Continual Learning with a Memory of Diverse Samples


# 简介

本文发表于 [CVPR 2021](https://arxiv.org/pdf/2103.17230.pdf)，代码[见此](https://github.com/clovaai/rainbow-memory)。

本文聚焦于 rehearsal 策略中样本筛选问题，作者认为，被选择存储在内存中的样本不仅应该代表它们对应的类别，而且应该区别于其他类别。 为了选择这样的样本，作者认为靠近分类边界的样本最具辨别力，靠近分布中心的样本最具代表性。 

为了满足这两个特征，本文提出对特征空间中不同的样本进行采样，提出一种新的多样性感知采样方法，通过利用分类不确定性有效管理容量有限的内存。另外，为了进一步提升样本多样性，本文建议采用 data augmentation。

另外，如 Fig 2 所示，本文针对现实中更可能的增量环境 Blurry-CIL (每次新任务中也会包含少量之前看过的类别，任务与任务之间并非完全隔绝的) 做了研究，探讨了该环境下的增量学习问题。 



![image-20210721094358900](https://i.loli.net/2021/07/21/lP3W4K5f8YqUv7G.png)





# 方法

## Diversity-Aware Memory Update

![image-20210721020321281](https://i.loli.net/2021/07/21/ZESHutlRgJIL2pA.png)

为了确保多样性，需要估计每个样本在类别判别特征空间中的相对位置。 但是计算特征的相对位置在计算上是昂贵的，因为它需要计算样本到样本的距离 $(O(N^2))$。 相反，本文建议通过分类模型估计的样本的不确定性来估计相对位置，**即假设模型的更确定样本将位于更靠近类分布中心的位置，反之亦然**。

![image-20210721021132678](https://i.loli.net/2021/07/21/HyhQXvuPOoxkJLZ.png)

![image-20210721021156121](https://i.loli.net/2021/07/21/rOCSZeHTV2Uk1cP.png)



Algorithm 1 给出了样例更新的流程，其中公式 4 如上所示。在公式4中，$u(x)$ 表示样本 $x$ 的不确定性，$S_c$ 是类别 $c$ 是预测的 top-1 类别的次数。$1_c$ 表示二进制索引向量。 值较低的 $u(x)$ 对应于扰动上更一致的 top-1 类，表明 $x$ 位于模型非常自信的区域。

公式 4 的思想很直观，如 Fig 3 所示，对输入施加扰动，然后计算扰动样本 $\tilde{x}$ 被推断为正确类的次数 $S_c$，最后计算得到 $u(x)$。显然，$Sc$越大，$u(x)$越小，从而不确定性越小。



## Diversity Enhancement by Augmentation

### Mixed-Label Data Augmentation

随着任务迭代的进行，新任务中的样本可能遵循与情景记忆中的样本（即之前的任务）不同的分布。RW采用混合标记的 DA (data augmentation) 来“混合”新任务类中的图像和内存中旧类的样本。 这种混合标签 DA 减轻了由任务上的类分布变化引起的副作用并提高了性能。

作为代表性的混合标记 DA 方法之一，CutMix [1] 生成混合样本和平滑标签，给定一组监督样本 $(x1, y1)$ 和 $(x2, y2)$，如下所示：

![image-20210721091936649](https://i.loli.net/2021/07/21/L2PTVtFGdBbRK1k.png)

其中，集合 $m$ 表示根据从beta分布中提取的超参数$β$为图像x1随机选择的像素区域。如（5）所示，混合标签DA生成的人工样本很难被**视为源图像的变体**，这与传统数据增强不同，传统数据增强在不破坏类边界的情况下通过翻转、旋转和/或对比操作原始图像。



**Automated Data Augmentation.** 除了上述混合标记的 DAs 之外，RW进一步使用 AutoDA 通过在 CIL 下将多个 DA 合成到模型性能上来丰富增强效果。 特别是，文章采用 AutoAugment [2]，提供用于确定增强次数及其幅度的参数。（[2] 就是一种自动化增强数据的方法）



# 实验

![image-20210721093917952](https://i.loli.net/2021/07/21/SH1pDmVQ2I5uEwo.png)

![image-20210721094022338](https://i.loli.net/2021/07/21/jCwuA2PG3c1LnV4.png)

![image-20210721094158125](https://i.loli.net/2021/07/21/o3stUaXjSqgpV98.png)



# 参考

1. [Sangdoo Yun, Dongyoon Han, Seong Joon Oh, Sanghyuk Chun, Junsuk Choe, and Youngjoon Yoo. Cutmix: Regularization strategy to train strong classifiers with localizable features. In ICCV, pages 6023–6032, 2019.](https://arxiv.org/abs/1905.04899)
2. [Ekin D. Cubuk, Barret Zoph, Dandelion Mane, Vijay Vasudevan, and Quoc V. Le. AutoAugment: Learning augmentation strategies from data. In CVPR, June 2019. ](https://openaccess.thecvf.com/content_CVPR_2019/html/Cubuk_AutoAugment_Learning_Augmentation_Strategies_From_Data_CVPR_2019_paper.html)


