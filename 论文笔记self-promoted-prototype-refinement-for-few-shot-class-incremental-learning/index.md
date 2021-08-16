# 论文笔记——Self-Promoted Prototype Refinement for Few-Shot Class-Incremental Learning


# 一、简介

本文发表于 [CVPR 2021](https://openaccess.thecvf.com/content/CVPR2021/papers/Zhu_Self-Promoted_Prototype_Refinement_for_Few-Shot_Class-Incremental_Learning_CVPR_2021_paper.pdf)，暂时没有开源代码。

本文研究的问题是 FSCIL，即 few-shot incremental learning。与传统的增量学习问题相比，FSCIL 还面临着增量样本少的挑战。FSCIL 在某些情况更接近真实环境，例如，在工业视觉检测任务中，随着生产的不断进步，由于设备磨损等原因，经常会出现有缺陷新类别。值得一提的是，另外一种比较真实的 CIL 场景是新任务不仅包含大量的新类样本，同时也包含少量的旧类样本，比如由于换季导致了流行商品的变化，这杯称为 Blurry-CIL。

回到本文，本文首先是指出传统的增量学习算法对于 FSCIL 是无力的，因为 FSCIL 中新类样本少，缺乏足够多的监督指标 。如图 1 所示，典型的增量学习方法 [iCaRL, EEIL] 扩展的表示空间代表性不足，因此与旧类相比，新类通常表现出不足的聚合。 另外，由于新的类样本不足，用于分类的原型在增量学习后容易与其他类混淆，这大大恶化了后续任务。

![image-20210815123007085](https://i.loli.net/2021/08/16/VbRalneN465WLuS.png)

为了缓解上述问题，首先，本文采用随机情节选择策略（RESS, random episode selection strategy）通过强制特征适应各种随机模拟的增量过程来增强特征表示的可扩展性；其次，引入了一种自我提升的原型细化机制（SPPR, self-promoted prototype refinement mechanism），通过利用新类样本和旧类原型的表示之间的关系矩阵来更新现有原型。 这增强了新类的表达能力，同时保留了旧类之间的关系特征。 特别地，提出了一种称为动态关系投影的新模块，将新类样本的表示和旧类的原型映射到相同的嵌入空间中，并通过使用两个嵌入的距离度量计算它们之间的**投影矩阵**空间。 我们将矩阵作为原型细化的权重来引导原型向保持现有知识和增强新类的可辨别性的动态变化。



# 二、方法

![image-20210815123854725](https://i.loli.net/2021/08/16/icUfDj9ZImVeG23.png)

**算法流程**	base task 正常训练；IL task 中，每次迭代新数据直接输⼊进模型。⽽对于旧数据，每次从其中挑选 N-way K-shot 样本计算它们在当前CNN参数下的原型特征，然后采⽤⽂中提到的 dynamic Relation  Projection ⽅法把被本次迭代新计算得到的 $C$​ 个类的原型特征和原来未选择到的 N-C 个类的特征结合起来，构建相关矩阵，转换之后得到⼀个 $class$ x $dimension$ 的矩阵，和新类提取到的特征⼀起经过 softmax 层算损失。

> 简⽽⾔之就是对旧类⽤原型特征当作全连接层的输⼊，只是这个原型的构成是 N-way K-shot 算得的 $K$​ 个类别的新原型和 $N-K$ 个类别的旧原型，并将新旧原型通过数学构建起联系。⽂中说这种⽅式显示的考虑新旧类的关系，确实，$C$ 原型是引⼊新类后迭代后的参数计算的，所以这⾥包含了新类的信息，⽽ $N-K$​​ 原型则保 证了就特征提取层的参数。随机的选择则保证了参数的 extensibility。
>
> > 多说⼀句，这⾥有⼀个不容易察觉的点是由于新类的样本量太少，所以在设计损失的时候旧类是⽤原型来作为 softmax 输⼊的。这在物理上平衡了新旧类的样本数量，应该是⼀个能够保证准确率的基本前提



# 三、实验

**实验设置**	数据方面，base task 拥有充足的数据来初始化模型，后续增量以 N-way K-shot 的方式。对于 cifar-100，base task 有60个类，后续增量有40个类。

为了减少增量样本随机抽样带来的误差，本文选取不同的样本进行5次测试，然后对每个模型取平均值。

Table 1 准确率低时因为本文仅仅只训练了 70 epochs 就中止了。（不知道为啥只训练 70 个epochs，即便后续小样本增量要降低训练次数，但第一次样本充足，为什么这么快就结束了呢？70 epochs 意味着base task 尚未训练充分，不知道这对于后续增量是否有影响。吐个槽，我想不通这点作者是怎么给编辑解释的，可能更完整的实验在补充材料里吧？）

Fig 5 中星号代表将相应的 CIL 方法应用于 FSCIL 任务的结果，TOPIC 和 TOPIC-MML 则本来就是设计来应对 FSCIL 问题的方法。（粉色折线展示了方法用于常规增量的实验结果）

![image-20210815124545356](https://i.loli.net/2021/08/16/zvYWBHQs42nfudr.png)

![image-20210815124918765](https://i.loli.net/2021/08/16/Ed6pO89XBPs4oZt.png)

![image-20210815125008273](https://i.loli.net/2021/08/16/7reFWwNObyZi3aH.png)




