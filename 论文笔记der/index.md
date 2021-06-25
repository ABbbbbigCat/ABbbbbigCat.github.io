# 论文笔记——DER: Dynamically Expandable Representation for Class Incremental Learning


# 简介

本文发表于[CVPR 2021](https://arxiv.org/abs/2103.16788)，随文代码在[这里](https://github.com/Rhyssiyan/DER-ClassIL.pytorch)。

本文主要基于 architectural strategy，设计了一种动态扩充网络模型的方法。同时，该方法也采用了 rehearsal strategy 保存了旧数据的样例集 $M_t$ 用于未来训练（基于 herding 方法）。在思想上，本文遵循了 architecture 和 rehearsal 策略，完全固定提取到的特征并用留存数据和新数据对其进行训练来获得分类器。在性能上，DER能够获取十分不错的准确率，但不断扩充的模型和 **super feature** 会让模型的容量不断增大。



# 模型结构

![image-20210625110552198](https://i.loli.net/2021/06/25/etGldzXEMRAcr64.png)



# 训练过程

具体而言，DER在每次增量时**固定**之前学习到的特征提取器，并且学习一个新的、基于新数据 $D_t$ 和留存数据 $M_{1:t-1}$ 训练的特征提取器来扩充模型。对于新特征提取器，DER使用了一个辅助损失函数 $L_{H_{t}^a}$ 来学习 $D_t$ 和 $M_{1:t-1}$ 的差异性，并利用一个 Mask layer 对特征提取器的channel进行减枝以学习<font color=red>compact feature</font>。最终，DER将新学习到的特征提取器所提取到的特征<font color="red">embed</font>到之前所学习到的特征中构成 super feature，<font color="red">重新初始化和训练模型分类器</font>。

所以，该模型的最终的损失函数为：

$$ L_{DER}=L_{H_t}+\lambda_aL_{H_t^a}+\lambda_sL_S$$

$$ L_s = \frac{\sum_{l=1}^LK_l||m_{l-1}||_1||m_l||_1}{\sum_{l=1}^LK_lc_{l-1}c_l}$$

其中，$L->总层数, K_l->l卷积层的核大小，c_l->l层的通道数$。$m_l=\sigma(se_l)$，$e_l$是可学习掩码参数，$s$是一个比例因子（scaling factor）用于控制公式的锐度（sharpness），有：$s=\frac{1}{s_{max}}+(s_{max}-\frac{1}{s_{max}})\frac{b-1}{B-1}$，$s_{max}>>1$是一个超参数，$b$是批量索引（batch index），$B$是一次epoch中的批量总数。显然，$b$的变化引起了$s$的变化。（注意到对$\sigma(se_l)$求导时，得到的结果为$g_{el}=s\sigma(se_l)[1-\sigma(se_l)]$会受$s$的变化影响而不稳定，故本文提出了利用$g_{el}^{'}=\frac{\sigma(e_l)[1-\sigma(e_l)]}{s\sigma(se_l)[1-\sigma(se_l)]}g_{e_l}$对其进行补偿）。

> $L_s$是一个Sparsity Loss，其大小受 $m_l$的影响，$m_l$越稀疏，$L_s$ 越小。稀疏的 $m_l$ 可以实现对特征提取层的剪枝。就个人当前的知识面而言，$L_s$ 的设计很是巧妙



# 实验结果

模型：对于 CIFAR-100 和 ImageNet 都采用 18-layer ResNet 作为基本模型，细节上遵循  [RPSNet](https://proceedings.neurips.cc/paper/2019/hash/83da7c539e1ab4e759623c38d8737e9e-Abstract.html) 

任务设置：本文对两种主流的任务设置方式都进行了实现：其一，每次增量 N 个类（文中以 B0 表示）；其二，初始训练数据集一半的类，之后每次增量 N 个类（文中用 B50 表示）

数据留存：固定大小 $M = 2000$

![image-20210625113011174](https://i.loli.net/2021/06/25/9JGq6EFhTKAomVR.png)

![image-20210625113029833](https://i.loli.net/2021/06/25/GbiPAS7WM3CVkXs.png)

![image-20210625113108524](https://i.loli.net/2021/06/25/EIPc6mbwYdy2itQ.png)


