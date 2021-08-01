# 论文笔记——Prototype Augmentation and Self-Supervision for Incremental Learning


# 一、简介

本文发表于 [CVPR 2021](https://openaccess.thecvf.com/content/CVPR2021/papers/Zhu_Prototype_Augmentation_and_Self-Supervision_for_Incremental_Learning_CVPR_2021_paper.pdf)，随文代码[见此](https://github.com/Impression2805/CVPR21_PASS)。

本文指出了增量学习过程中 **task-level overfitting phenomenon**。直观上，这是说模型在训练当前任务的时候，只会专注于捕获对当前分类任务有用的信息，而可能忽略那些在当前对于区分度贡献度较小但却会影响未来训练的信息。由于增量学习通常会使用之前模型来初始化当前模型，因此之前任务的 task-level overfitting 会影响后续模型训练。[1] 的研究发现，使用存储的样本从零开始训练的模型可以出人意料地优于许多最近提出的算法。这项研究揭示了之前的模型（主要 携带 task-specific features）可能是一对于当前任务糟糕的初始化选择，如 Fig 1 (b) 所示。因此，该模型需要更多的更新才能很好地执行当前任务，这在另一方面增加了遗忘问题。



![image-20210727142045409](https://i.loli.net/2021/07/27/cDVWO5rH2ujo9k1.png)

基于上述分析，本文提出通过维护决策边界和减少 task-level overfitting phenomenon 来提高 CIL 的性能，如 Fig 1 (a) 所示。本文所提出的 **PASS** 主要包括：1）Prototype Augmentation; 2) Self-Supervision。

一方面，ProtoAug 记忆了一个 class-representative prototype (通常指类在深层特征空间中的平均值)，并且在学习新类时通过高斯噪声增强记忆原型。然后将增量原型和新数据的深度特征一起联合训练以维护新旧类的区分和平衡。（这么做受到最近一项长尾识别工作 [2] 的启发，该工作通过增加带有特定扰动的尾类来扩展尾类的分布）

另一方面，受到 Self-supervised learning (SSL) 对缓解任务水平过度拟合现象的启示，本文建议利用 SSL 的优势来学习 task-agnostic and transferable representation。（直观地说，使用SSL，不同的任务在参数空间中会更接近，并且在当前任务上训练的模型会更好地初始化学习下一个任务）

> 值得一提的是，本文仅仅保留了 class prototype，没有使用 memory size。在增量训练中，通过 将 Prototype Augmentation 反馈给分类器用于监督学习。



# 二、方法论

## 1. Prototype Augmentation

**Compute prototype**	$\mu_{t,k}=\frac{1}{N_{t,k}}\sum_{n=1}^{N_t,k}F(X_{t,k};\theta_t)$​



**ProtoAug**	$F_{t_{old},k_{old}}=\mu_{t_{old},k_old}+e*r$

其中，$e\sim N(0,1)$​ 派生自高斯噪声，其维度和 prototype 一致。$r$​ 是控制 augmented prototypes 不确定性的尺度。特别地，尺度 $r$​​​ 可以预定义，或者计算为 class representations 的平均方差：

$$r_t^2=\frac{1}{K_{old}+K_{new}}(K_{old}*r_{t-1}^2+\sum_{k=1}^{k_{new}}\frac{Tr(\sum_{t,k})}{D})$$

其中，$K_{old}$​ 和 $K_{new}$​ 分别代表旧类数目和新类数目。$D$​ 是深度特征空间的维度。$\sum_{t,k}$​ 是 $t$​ 阶段 $k$​ 类特征的协方差矩阵，$Tr$​ 运算计算矩阵的迹。作者观察到 $r_t$​​​ 在 CIL 实验过程中的不同阶段略有变化。 因此，只能计算和使用第一个任务中特征的平均方差：$r^2=r_1^2=\frac{1}{K_1*D}\sum_{k=1}^{K_1}Tr(\sum_{1,k})$

之后，将新类特征和增广原型反馈给统一分类器：

$$\left\{\theta_t,\phi_t\right\}=\arg min_{\theta_t,\phi_t}\left\{L_{t,ce}+L_{t,protoAug}\right\}=\arg min_{\theta_t,\phi_t}\left\{L_t(G(F(X_t;\theta_t);\phi_t),Y_t)+\sum_{i=1}^{t-1}L(G(F_i;\phi_t),Y_i) \right\}$$

其中，$F_i$代表旧类集 $C_i$增加的特征



## 2. SSL based Label Augmentaion

受 [3] 的启发，本文简单地通过基于 SSL 扩充当前类来学习统一模型。 具体来说，对于每个类，将其训练数据旋转 90、180 和 270 度以生成 3 个新类（预测图片旋转的角度），将原始 K 类问题扩展为新的 4K 类问题：

$$X_t^{'}=rotate(X_t,\theta),\quad \theta \in \left\{90,180,27\right\}$$

上述方法在同时学习原始任务和自监督任务的过程中放松了一定的不变约束，有利于学习更丰富的特征。实验结果表明，这种简单的方法可以提高CIL的性能。



## 3. Integrated Objective of PASS

**Knowledge distillation**	$L_{t,kd}=||F_t(X_t^{'};\theta_t)-F_{t-1}(X_t^{'};\theta_{t-1})||$



**PASS**	$L_{t,total}=L_{t,ce}+\lambda*L_{t,protAug}+\gamma*L_{t,kd}$



![image-20210802012219909](https://i.loli.net/2021/08/02/O2qaoLjVh638sXf.png)



# 三、初步实验

为了验证 ProtoAug 和 SSL 的有效性，本文分别在 MNIST 和 CIFAR 10 / CIFAR 100 上进行了实验。Fig 3 表明 ProtoAug 有助于模型更好的划分边界；Table 1 证明了 SSL 的有效性



![image-20210727171733210](https://i.loli.net/2021/07/27/ZvMrOlHwjbBaR1c.png)

![image-20210727172312004](https://i.loli.net/2021/07/27/bgMIvzS13HVp5P9.png)

![image-20210727173018908](https://i.loli.net/2021/07/27/MO8ktG6CEquzSQ7.png)

如 Fig. 4 所示，对于未见过的类，SSL 导致新类的内部距离更小，这意味着使用 SSL 学习的模型在新类上的泛化能力优于 baseline。 而对于训练类，基线具有更紧凑的特征分布。 这表明过度的特征压缩可能会损害新类泛化的表示学习。Fig 4 (d) 表明 SSL 导致更高的特征空间密度 $π_{ratio}$，泛化的改进与 [4] 中的观察一致。



> 这部分的设计很有启发，从方法本身的角度来看，本文提出的 ProtoAug 和 SSL 并不新颖，其中也没有很具原创性的数学表达。人工智能领域下的理论论证向来不容易，于是本文通过实验章节前的“初步实验”系统的从可视化的角度论证了方法的有效性。（尽管实际上最直观的还是边界清晰与否，样本紧凑程度）



# 四、实验

![image-20210727174947795](https://i.loli.net/2021/07/27/x7CZpnEi1chQG6D.png)

![image-20210727174854920](https://i.loli.net/2021/07/27/YbU1a295hFVDu7Z.png)







# 参考

1. Ameya Prabhu, Philip HS Torr, and Puneet K Dokania. Gdumb: A simple approach that questions our progress in continual learning. In *ECCV*, pages 524–540, 2020.
2. Jialun Liu, Yifan Sun, Chuchu Han, Zhaopeng Dou, and Wenhui Li. Deep representation learning on long-tailed data: A learnable embedding augmentation perspective. In *CVPR*, pages 2967–2976, 2020.
3. Hankook Lee, Sung Ju Hwang, and Jinwoo Shin. Self-supervised label augmentation via input transformations. In *ICML*, 2020.
4. Revisiting Training Strategies and Generalization Performance in Deep Metric Learning





# 附录

特征空间密度：$π_{ratio}(F) = π_{intra}(F)/π_{inter}(F)$，[Roth et al. [4]](http://proceedings.mlr.press/v119/roth20a.html) 提出并发现训练和测试分布之间发生相当大的变化时，增加的特征空间密度 $π_{ratio}$ 与更强的泛化有关。


