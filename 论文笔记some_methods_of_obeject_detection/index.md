# Some methods of Object Recognition

本文旨在总结诸如Selective Search、R-CNN等对象识别方法的主要思想。

## Selctive Search

该方法的目的是对象识别，也就是说在一张图片里找出对象所在的区域，并识别他。其主要思想：先利用已知的、计算复杂度较低的一种算法将原图片划分为很多个小型区域，之后计算一个区与其领域的相关性，若相关性满足初始设置好的阈值，则将二者合并，遍历的进行这一操作，直到无法合并为止。并在此过程中，对每一个区域进行对象识别，若成功找到对象则保留结果。

<!--more-->

优势：提供了一种Object detection的思路。

不足：主要有二。其一，由于区域合并之后会对新的区域再次进行识别（这一过程涉及计算），所以会不可避免的出现重复计算的情况。其二，有可能出现下一次标记的区域识别出来的对象正是上一次已经识别出的对象，且缺乏有效手段将之分离。


## R-CNN

此方法在Selective Search的基础上，将特征提取层变为了卷积层，同时对于大小不等的Object Proposals采取了warpped的方法，使其以固定尺寸输入进卷积层并提取特征。

优势：卷积层提取特征的能力更加强大

不足：使用wrapped处理Object Proposals不可避免的会队原始图像进行诸如压缩、拉伸等操作，很大可能会出现信息失真。


## Spatial Pryamid Pooling (SSP)

此方法相较于R-CNN最大的区别在于，不在输入图像时采取wrapped或者cropped的方法，而是直接将图像输入进卷积层，并在卷积层最后接上一个SPP层，SPP层会对最后一个卷积操作输出的feature maps进行处理，同样是对其进行卷积操作，至于其卷积核的尺寸和步长则需要根据相应的feature maps的尺寸作出调整。假设feature maps传入的是一个mxn的矩阵，则相应的，卷积核的尺寸为($⌈m/a⌉$, $⌈n/a$⌉)，步长为($⌊a/n⌋$)。($⌈⌉$指向上取整，$⌊⌋$指向下取整)

优势：可以把任意尺寸的图像作为输入。

不足：直接把运用Selective Search获得的Object Proposals输入进卷积层，会对许多特征进行重复提取，使得计算效率变差。


## Fast R-CNN

结合了SSP和R-CNN，首先还是对原始图像进行Selective Serach得到Object Proposals，然后把原始图像输入进卷积层提取得到feature maps，从feature maps中选择与Object Proposals相对应的区域传入RoI(Region of Interest,其实质就是SPP层)。

优势：由于是直接利用原始图片进行特征提取，所以避免了出现重复提取特征的情况，极大的加快了计算效率。

不足：基本还是在Selective Search的基础上进行扩展，然而Selective Search本身需要花费大量计算时间。

