# 论文笔记——Learning a Unified Classifier Incrementally via Rebalancing


# 简介

本文发表于[CVPR 2019]()，复现代码[见此](https://github.com/hshustc/CVPR19_Incremental_Learning)。

LUCIR同样是致力于缓解rehearsal中不平衡问题，如 Fig.1 所示，本文指出不平衡的数据主要带来了三种问题：(1) 量级不平衡：新类权重向量的量级明显高于旧类； (2) 偏差：之前的知识，即旧类的特征和权重向量之间的关系没有很好地保留； (3)歧义：新类的权重向量与旧类的权重向量接近，往往会导致歧义。

![image-20210719220650265](https://i.loli.net/2021/07/19/YofNtjpOh6vZ4u3.png)

# 方法

![image-20210720023629014](https://i.loli.net/2021/07/20/5CwmeM7U8KrgxRY.png)



## Cosine Normalization

![image-20210719221355075](https://i.loli.net/2021/07/19/72cyeLwBpXlNSRr.png)

如 Fig.3 所示，由于类别不平衡，新类别的embedding和bias明显高于旧类别。LUCIR提出在最后一层使用 **cosine normalization**：

$$p_i(x)=\frac{exp(\eta<\overline{\theta}_i,\overline{f}(x)>)}{\sum_jexp(\eta<\overline{\theta}_j,\overline{f}(x)>)}$$

其中，$f$是特征提取器，$\theta$是权重，$\overline{v}=v/||v||_2$代表$l_2-normalized$向量，$<\overline{v}_1,\overline{v}_2>=\overline{v}_1^T\overline{v}_2$衡量了两个标准向量的余弦相似度。引入可学习标量$η$来控制$softmax$分布的峰值，因为$<\overline{v}_1,\overline{v}_2>$的范围被限制在[−1, 1]。它可以有效地消除因幅度显着差异而引起的偏差。

对于蒸馏损失，由于原始模型中的标量 $η$与当前网络中的标量 $η$ 不同，因此模拟 $softmax$ 之前的分数而不是 $softmax$ 之后的概率是合理的。 还值得注意的是，由于余弦归一化，$softmax$ 之前的分数都在相同的范围内（即 [-1, 1]），因此具有可比性。 形式上，蒸馏损失更新为：

$$L_{dis}^C(x)=-\sum_{i=1}^{|C_0|}||<\overline{\theta}_i,\overline{f}(x)>, <\overline{\theta}_i^{*},\overline{f}^{*}(x)||$$



## Less-Forget Constraint

![image-20210719224038625](https://i.loli.net/2021/07/19/fNjc29xYGh5iFI8.png)

如 Fig.4 所示，$L^C_{dis}$主要考虑局部几何结构，即归一化特征与旧类嵌入之间的夹角，此约束无法阻止嵌入和特征完全旋转。为了加强对已有知识的约束，LUCIR建议修复旧的类嵌入，并计算一个新的特征提取损失，如下所示：

$$L^G_{dis}(x)=1-<\overline{f}^*(x),\overline{f}(x)>$$

其中 $\overline{f}^*(x)$ 和$ \overline{f}(x)$ 分别是原始模型和当前模型提取的归一化特征。



## Inter-Class Separation

为了避免新类和旧类之间的模糊性，LUCIR引入了一个边缘排序损失（margin ranking loss），以确保它们能够很好地分离：

$$L_{mr}=\sum_{k=1}^Kmax(m-<\overline{\theta}(x),\overline{f}(x)>+<\overline{\theta}^k,\overline{f}(x)>,0)$$

其中$m$是边缘阈值，$\overline{θ}(x)$是$x$的ground-truth class embedding，$θ^k$是被选为$x$的hard negatives的top-k个新类嵌入之一。

## Integrated Objective

$$L=\frac{1}{N}\sum_{x\in N}(L_{ce}(x)+\lambda L_{dis}^G(x))+\frac{1}{N_0}\sum{x \in N_0}L_{mr}(x)$$

其中，N是 $\chi$ 中抽取的训练批次，$N_o \subset N$是样例集，$\lambda=\lambda_{base}\sqrt{|C_n|/|C_o|}$ 是损失权重（$\lambda_{base}$是每个数据集的固定常量，$|C|$是类别数目）



# 实验

![image-20210719230737084](https://i.loli.net/2021/07/19/mse5tROdSA64hWn.png)

![image-20210719230852795](https://i.loli.net/2021/07/19/BV7X4JrqWGmTOML.png)



> 本文对于平衡性的分析可以说是很全面了，即考虑全连接层的偏置，又考虑特征偏置，同时还指出了新旧类之间存在模糊。针对这三个问题，LUCIR分别提出了三种损失函数作为约束。放缩性研究部分证明了各个损失项的有效性。不过提升比较有限，这部分的研究值得继续深挖。（当然，在样例集有限的前提下，这样的深挖也是有极限的）


