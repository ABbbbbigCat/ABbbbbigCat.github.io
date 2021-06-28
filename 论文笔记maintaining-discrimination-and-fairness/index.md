# 论文笔记——Maintaining Discrimination and Fairness in Class Incremental Learning


# 简介

本文发表于 [CVPR 2020](https://arxiv.org/pdf/1911.07053.pdf)，参考代码[见此](https://github.com/hugoycj/Incremental-Learning-with-Weight-Aligning)。

本文主要做了两方面工作：其一，分析了知识蒸馏（Knowing Distillation）在增量学习中实际发挥的效果，提出KD的效果在于强调了旧类的**差异性**；其二，探究了新旧类不平衡的原因，提出了Weight Aligning维护**公平性**。

![image-20210628151518957](https://i.loli.net/2021/06/28/MstOaYWTKm3eVLn.png)

具体而言，table 1 表明施加KD增加了旧类被错分为新类的概率，降低了旧类被误分为其他旧类的概率。本文对此给出的解释是——KD维护了旧类的**差异性**。

> 传统观点中，KD的主要作用在于维护旧知识——以旧模型的输出作为标签来约束模型训练，从而在宏观中维护旧知识。本文在此基础上进一步从细节上探究了KD发挥的效果，最后通过旧类别被误分为其他旧类和新类的情况推得KD维护了新旧类的差异性。因为KD的施加降低了旧类被误分为其他旧类的概率（**强调了旧类间的差异，维护了模型的旧知识**），增加了旧类被误分为新类的概率（被误分为其他旧类的损失相较于被误分为新类的损失更大，且由于新旧类不平衡问题导致了对新类的响应高于对旧类响应）
>
> 不过这样的分析似乎并不全面，观察 Table 1 不难发现，KD损失的引入增大了新类的error，降低了旧类的error，由此可见KD同时维护了新旧类的公平性。



![image-20210628152040926](https://i.loli.net/2021/06/28/f7eLYGvKuU2Bodi.png)

上图是在进行增量训练时，分类层**权重范**，观察到新类的权重范数明显大于旧类，由此推知训练过程中发生了不平衡，分类结果朝着新类倾斜。对此，本文提出了一个放缩因子来缩小新类权重的值，从而维护**公平性**。其公式如下：

$$\hat{W}_{new}=\gamma W_{new}$$

$$\gamma=\frac{Mean(Norm_{old})}{Mean(Norm_{new})}$$



# 模型结构

![image-20210628151202803](https://i.loli.net/2021/06/28/zpoj9QihUkWxlMm.png)

模型：For CIFAR-100, use 32-layer ResNet; For ImageNet, use 18-layer ResNet

WA 模型的特点是使用线性纠正对$W_n$进行了缩放，以此来解决新旧类之间不平衡问题。

# 实验结果

![image-20210628154845583](https://i.loli.net/2021/06/28/6tmW34M8FqaflGz.png)

![image-20210628155118497](https://i.loli.net/2021/06/28/IMsiefZjdhvLDGg.png)

