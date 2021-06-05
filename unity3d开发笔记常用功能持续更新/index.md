# Unity3d开发笔记——常用功能（持续更新…）


# Unity3d 使用Spine动画

Spine 是一款针对 **2D 骨骼动画**编辑工具。与传统的逐帧动画相比，Spine 动画由于只保存骨骼的动画数据，它所**占用的空间非常小**。对于程序而言，可以通过代码控制骨骼，实现程序动画。*SkeletonAnimation* 是 Spine 引入的重要组件。

基本使用方法可参考CSDN上的一篇[博客](https://blog.csdn.net/linshuhe1/article/details/79792432)，更细致的使用可参看官方文档中的"[怎么使用SkeletonAnimation](http://zh.esotericsoftware.com/spine-unity-zh-old#怎么使用SkeletonAnimation)"。

教程：[Spine 示例项目](http://zh.esotericsoftware.com/spine-examples)

# Unity3d 进度条

设置 unity3d 的进度条，最常见的一种手段是将图片属性栏的 "Image Type" 属性设置为 "Filled"，通过对图片进行填充来模拟进度条。多数情况下，这是个不错的选择。但当图片素材本身在拉伸时会变形时，这个方法就不太好用了。

这个时候，我们可以通过**改变图片 "Rect Transform" 中的 "Pivot" 后通过改变图片的 "Scale"** 来实现进度条效果。当然在这之前我们需要把图片属性设置为 "Sliced"，并利用 "Sprite Editor" 改变图片的九宫格属性，以确保其拉伸时不会发生形变。

 ![image-20210521110859326](https://i.loli.net/2021/06/06/iqVb8thQz5omCpI.png)

![image-20210602131220553](https://i.loli.net/2021/06/06/uHSWtLxfI9igkoR.png)

如果存在进度条特效，只需要关注进度条特效的位置，使其变化与进度条保持一致即可

# Unity3d 文件引用拷贝

在开发过程中，我们有时会碰到接手长期活动的新一期开发工作。这个时候我们会用到上一期活动的文件。直接拷贝并不可取，因为这会使拷贝内容仍旧依赖于上一期文件。这个时候就需要利用到Unity3d的中引用拷贝了。对需要拷贝的文件右击，并按照如下方式拷贝，就能实现引用拷贝了。

![image-20210523181331984](https://i.loli.net/2021/06/06/FNLqSEzmlPdkorT.png)


