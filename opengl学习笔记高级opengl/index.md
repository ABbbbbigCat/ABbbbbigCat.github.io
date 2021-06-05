# OpenGL学习笔记——高级OpenGL


# 1. 深度测试
深度测试通过衡量物体的*深度缓冲*（Depth Buffer, 物体与视口的差距）决定了最后渲染的图像。OpenGL支持[八种深度测试函数](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/01%20Depth%20testing/)，其中最长使用的是<font color="#00dd00#">GL_LESS</font>，效果是“在片段深度值小于缓冲的深度值时通过测试”。OpenGL默认情况下，深度测试是关闭的，需要通过<font color="#00dd00">glEnable(GL_DEPTH_TEST)</font>开启。
某些情况下我们会需要对所有片段都执行深度测试并丢弃相应的片段，但**不希望更新深度缓冲**。这个时候我们可以将<font color="#00dd00#">glDepthMask()</font>设置为<font color="#00dd00#">GL_FALSE</font>。此时的深度缓冲是**只读**的，它可以帮助我们进行深度测试，但不会对深度图进行更新。
深度测试实际采取的并非线性计算$F_{depth}=(z-near)/(far-near)$（在透视矩阵应用之前在观察空间中是线性的），而是非线性计算，其公式如下所示:
$$F_{depth}=(1/z-1/near)/(1/far-1/near)$$
与线性相比，非线性计算中深度值很大一部分是由很小的z值所决定的，这给了近处的物体很大的*深度精度*。这个（从观察者的视角）变换z值的方程是嵌入在投影矩阵中的，所以当我们想将一个顶点坐标从观察空间至裁剪空间的时候这个非线性方程就被应用了。
## 1.2 提前深度测试
现在大部分的GPU都提供一个叫做**提前深度测试**(Early Depth Testing)的硬件特性。提前深度测试允许深度测试在片段着色器之前运行。只要我们清楚一个片段永远不会是可见的（它在其他物体之后），我们就能提前丢弃这个片段。
片段着色器通常开销都是很大的，所以我们应该尽可能避免运行它们。当使用提前深度测试时，片段着色器的一个限制是你不能写入片段的深度值（通过修改<font color="#000066">GL_FragDepeth</font>的值）。如果一个片段着色器对它的深度值进行了写入，提前深度测试是不可能的。OpenGL不能提前知道深度值。
## 1.1 深度冲突
一个很常见的视觉错误会在两个平面或者三角形非常紧密地平行排列在一起时会发生，深度缓冲没有足够的精度来决定两个形状哪个在前面。结果就是这两个形状不断地在切换前后顺序，这会导致很奇怪的花纹。这个现象叫做*深度冲突*(Z-fighting)，因为它看起来像是这两个形状在争夺(Fight)谁该处于顶端。
缓解深度冲突一些简单且易于实现的技巧有：

 - **永远不要把多个物体摆得太靠近，以至于它们的一些三角形会重叠**。
 - **尽可能将近平面设置远一些**。深度缓冲公式中精度在靠近近平面时是非常高的，所以如果我们将**近**平面远离观察者，我们将会对整个平截头体有着更大的精度。然而，将**近**平面设置太远将会导致近处的物体被裁剪掉，所以这通常需要实验和微调来决定最适合你的场景的**近**平面距离。
 - 另外一个很好的技巧是牺牲一些性能，**使用更高精度的深度缓冲**。大部分深度缓冲的精度都是24位的，但现在大部分的显卡都支持32位的深度缓冲，这将会极大地提高精度。

# 2. 模板测试
**模板测试**(Stencil Test)允许我们根据一些条件丢弃特定片段。当片段着色器处理完一个片段之后，模板测试会开始执行，和深度测试一样，它也可能会丢弃片段。接下来，被保留的片段会进入深度测试，它可能会丢弃更多的片段。模板测试是根据又一个缓冲来进行的，它叫做模板缓冲(Stencil Buffer，通常每个模板值是8位)，我们可以在渲染的时候更新它来获得一些很有意思的效果。（比如战旗类游戏的角色边框等）
模板测试提供了和深度测试中<font color="#00dd00#">glDepthMask()</font>一样效果的函数：

```
glStencilMask(0xFF); // 每一位写入模板缓冲时都保持原样
glStencilMask(0x00); // 每一位在写入模板缓冲时都会变成0（禁用写入，保证模板图是只读的）
```
模板测试主要通过<font color="#00dd00#">glStencilFunc(GLenum func, GLint ref, GLuint mask)</font>和<font color="#00dd00#">glStencilOp(GLenum sfail, GLenum dpfail, GLenum dppass)</font>这两个函数，其参数具体定义可以在[这里](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/02%20Stencil%20testing/)得到。前者描述了描述了**OpenGL应该对模板缓冲内容做什么**，后者则说明应该**如何更新模板缓冲**。
## 2.1 物体轮廓
模板缓冲一个直接的效果就是帮助我们绘制物体边框。其基本思路是借助模板缓冲在同义词渲染循环中对同一物体进行两次渲染。第一次渲染开启模板测试并更新模板缓冲；第二次渲染将物体的尺寸增大一些，以第一次渲染的模板缓冲为判断基准，关闭模板缓冲更新（正常渲染保存的模板缓冲就是我们所需要的模板缓冲）和深度测试（第二次渲染主要是为了描边，这与深度无关）后渲染物体。

```
glEnable(GL_STENCIL_TEST);			// 在循环之外要先开启深度测试
glStencilFunc(GL_KEEP, GL_KEEP, GL_REPLACE);	// 告诉OpenGL在不同情况对模板缓冲做什么
...
// 渲染循环
while			
{
	...
	glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);		// 清理颜色缓冲值/深度缓冲值/模板缓冲值
	...
	glStencilFunc(GL_ALWAYS, 1, 0xFF); 	// 总是能够通过模板测试，这样得到的缓冲图能够描述物体的形态
	glStenciMask(0xFF);		// 第一次渲染要更新模板缓冲
	normalShader.use();
	draw();
	...
	glStenciFunc(GL_NOTEQUAL, 1, 0xFF);	// 测试值与模板缓冲值不同时，能够通过模板测试(即尺寸放大后多出来的部分)
	glStenciMask(0x00);	//	禁止更新模板缓冲
	glDIsable(GL_DEPTH_TEST);	// 关闭深度测试，因为这一次渲染不需要考虑深度，值需要描边
	shaderSingleColor.use(); 
	draw();
	glStencilMask(0xFF);		// 保证模板缓冲可以被glClear()清除
	glEnable(GL_DEPTH_TEST);
	...
}
```
# 3. 混合
OpenGL中，**混合**(Blending)通常是实现**物体透明度**(Transparency)的一种技术。透明就是说一个物体（或者其中的一部分）不是纯色(Solid Color)的，它的颜色是物体本身的颜色和它背后其它物体的颜色的不同强度结合。物体的透明度由**alpha**值决定，alpha颜色值是颜色向量的第四个分量。混合的开启仍然需要调用<font color="#00dd00#">glEnable(GL_BLEND)</font>。
## 3.1 丢弃片段
对于部分全透明但部分不透明的物体（如草），可以选择直接丢弃透明片段而不是混合，这样可以避免深度测试和混合一起时产生一些的麻烦（当写入深度缓冲时，深度缓冲不会检查片段是否是透明的，所以**透明的部分会和其它值一样写入到深度缓冲中**。如果我们先绘制透明物体再绘制不透明物体，那么无法通过深度测试的不透明物体将会被**直接丢弃**）。
## 3.2 混合
OpenGL中的混合通过下面这个方程实现：
 $$\bar{C}_{result}=\bar{C}_{source}∗F_{source}+\bar{C}_{destination}∗F_{destination}$$
 - $\bar{C}_{source}$ ：源颜色向量。这是源自纹理的颜色向量。
 - $\bar{C}_{destination}$ ：目标颜色向量。这是当前储存在颜色缓冲中的颜色向量。
 - $F_{source}$：源因子值。指定了alpha值对源颜色的影响。
 - $F_{destination}$：目标因子值。指定了alpha值对目标颜色的影响。

OpenGL中的<font color="#00dd00#">glBlendFunc(GLenum sfactor, GLenum dfactor)</font>函数接受两个参数，来设置**源**和**目标**因子。并进一步提供了<font color="#00dd00#">glBlendFuncSeparate(GLenum sfactorRGB, GLenum dfactorRGB, GLenum sfactorAlpha, GLenum dfactorAplha)</font>为RGB和alpha通道分别设置不同的选项。
OpenGL的混合具有很强的灵活性，<font color="#00dd00#">glBlendEquation(GLenum mode)</font>允许我们设置运算符。默认情况下<font color="#00dd00#">mode</font>被设置为<font color="#00dd00#">GL_FUNC_ADD</font>，表示将两向量相加（这足以令我们应付绝大多数场景渲染）。如果我们想让最终的结果为两向量相减，可以用<font color="#00dd00#">GL_FUNC_SUBTRACT, GL_FUNC_REVERSE_SUBTRACT</font>，前者是顺序相加（源-目标），后者为逆序。

## 3.3 不要打乱顺序
为了应对深度缓冲和混合混用时产生的问题，要求我们在渲染不能随意决定渲染顺序（更细节部分可以参考[LearnOpenGL](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/03%20Blending/)）。
要想让混合在多个物体上工作，我们需要最先绘制最远的物体，最后绘制最近的物体。这条标准时针对透明物体的，对于非透明物体，只需要保证它们在绘制透明物体之前绘制即可。大体的原则如下：

 1. 先绘制所有不透明的物体。
 2. 对所有透明的物体排序。
 3. 按顺序绘制所有透明的物体。


虽然按照距离排序物体这种方法对我们这个场景能够正常工作，但它并没有考虑旋转、缩放或者其它的变换，奇怪形状的物体需要一个不同的计量，而不是仅仅一个位置向量。完整渲染一个包含不透明和透明物体的场景并不是那么容易。更高级的技术还有**[次序无关透明度](https://zhuanlan.zhihu.com/p/92337395)**(Order Independent Transparency, OIT)

# 4. 面剔除
对于一个3D**立方体**而言，我们最多能同时看到的面是**3**个。这提醒我们在绘制它时可以省略无法看到的面。这能帮助我们提高至少50%的效率（多数情况下我们只能看到2个甚至1个面）。
通过设置三角形顶点的环绕顺序，OpenGL支持我们自由的决定是否剔除面，以及剔除哪些面。面剔除默认是关闭的，开启需要通过<font color="#00dd00#">glEnable(GL_CULL_FACE)</font>。默认情况下，OpenGL认为从镜头看过去**逆时针排布的是正面**并且会**剔除背面**（这种情况下在立方体内部向外看会发现立方体没有被渲染，因为从这个位置向外看时所有面都是顺时针排布）。我们可以通过<font color="#00dd00#">glCullFace(GLenum mode)</font>决定剔除的是正面，背面或者通通剔除；通过<font color="#00dd00#">glFrontFace(GLenum mode)</font>决定是以逆时针还是顺时针定义正面。

# 5. 帧缓冲
到目前为止，我们已经使用了很多屏幕缓冲了：用于写入颜色值的颜色缓冲、用于写入深度信息的深度缓冲和允许我们根据一些条件丢弃特定片段的模板缓冲。这些缓冲结合起来叫做**帧缓冲**(Framebuffer)，它被储存在内存中。OpenGL允许我们定义我们自己的帧缓冲，也就是说我们能够定义我们自己的颜色缓冲，甚至是深度缓冲和模板缓冲。一个完整的帧缓冲需要满足以下的条件：
- 附加至少一个缓冲（颜色、深度或模板缓冲）。
- 至少有一个颜色附件(Attachment)。
- 所有的附件都必须是完整的（保留内存）。
- 每个缓冲都应该有相同的样本数（sample）。

当完成所有条件后，我们可以使用下面这条命令判断我们的帧缓冲是否完整

```
if(glCheckFramebufferStatus(GL_FRAMEBUFFER) == GL_FRAMEBUFFER_COMPLETE)
  // 执行
```
## 5.1 纹理(Texture)附件和渲染缓冲对象(Renderbuffer object)附件
在完整性检查执行之前，我们需要给帧缓冲附加一个附件。附件是一个内存位置，它能够作为帧缓冲的一个缓冲，可以将它想象为一个图像。当创建一个附件的时候我们有两个选项：纹理或渲染缓冲对象。
### 5.1.1 纹理附件
将纹理附加到帧缓冲区时，所有渲染命令都将写入纹理，就好像它是普通的颜色/深度或模板缓冲区一样。 使用纹理的优势在于，所有渲染操作的结果将会被**储存在一个纹理图像中**，我们之后可以在着色器中很**方便**地使用它。
为帧缓冲区创建纹理与创建普通纹理大致相同：

```cpp
unsigned int texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
  
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 800, 600, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR); 
```
注意到我们将尺寸（width和height）设置为与屏幕一致，且向data参数传递了NULL。同时，我们不关心环绕方式（warping method）和多级渐远纹理（mipmap），因为多数情况我们用不到它们。对于这个纹理，我们仅仅分配了内存而没有填充它。填充这个纹理将会在我们渲染到帧缓冲之后来进行。
### 5.1.2 渲染缓冲对象附件
渲染缓冲对象(Renderbuffer Object)是在纹理之后引入到OpenGL中，作为一个可用的帧缓冲附件类型。就像纹理图像一样，渲染缓冲对象是一个真正的缓冲，即一系列的字节、整数、像素等。然而，渲染缓冲对象**不能被直接读取**（它是**只写**的）。这给它带来了一个额外的优势，那就是OpenGL可以进行一些内存优化，它会将数据储存为OpenGL**原生的渲染格式**，从而使其在离屏渲染（Off-screen Rendering）到帧缓冲（framebuffer）上的性能优于纹理附件。
渲染缓冲对象直接将所有的渲染数据储存到它的缓冲中，不会做任何针对纹理格式的转换，让它变为一个更快的可写储存介质。因为它的数据已经是原生的格式了，当写入或者复制它的数据到其它缓冲中时是非常快的。所以，交换缓冲这样的操作在使用渲染缓冲对象时会非常快。我们在每个渲染迭代最后使用的<font color="#00dd00#">glfwSwapBuffers</font>，也可以通过渲染缓冲对象实现：只需要写入一个渲染缓冲图像，并在最后交换到另外一个渲染缓冲就可以了。渲染缓冲对象对这种操作非常完美。
### 5.1.3 对比
渲染缓冲对象能为你的帧缓冲对象提供一些优化，但知道什么时候使用渲染缓冲对象，什么时候使用纹理是很重要的。通常的规则是，如果你**不需要从一个缓冲中采样数据**，那么对这个缓冲使用渲染缓冲对象会是明智的选择。如果你**需要从缓冲中采样颜色或深度值等数据**，那么你应该选择纹理附件。性能方面它不会产生非常大的影响的。
[Tips: [LearnOpenGL](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/05%20Framebuffers/#_2)中**渲染到纹理**章节的代码，实际上做的事是将通常渲染后的画面绘制到自定义的帧缓冲中（不会渲染到屏幕上），并将该画面作为纹理图片保存到纹理附件中。随后再绑定回默认帧缓冲(会渲染到屏幕上)，以之前保存的纹理附件作为纹理渲染画面，**使得整个场景都被渲染到了一个纹理上**]

## 5.2 后期处理
既然整个场景都被渲染到了一个纹理上，我们可以简单地通过修改纹理数据创建出一些非常有意思的效果。[LearnOpenGL](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/05%20Framebuffers/#_2)提到的后期处理包括反相（对所有颜色取1-color的操作），灰度（移除场景中除了黑白灰以外所有的颜色），以及核效果（在当前纹理坐标的周围取一小块区域，对当前纹理值周围的多个纹理值进行采样，创建一些意思的效果，比如模糊、边缘检测）等。
## 5.3 代码样例

```cpp
	// 创建帧缓冲
	// ------------
	unsigned int framebuffer;
	glGenFramebuffers(1, &framebuffer);
	glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
	// 生成纹理
	unsigned int texColorBuffer;
	glGenTextures(1, &texColorBuffer);
	glBindTexture(GL_TEXTURE_2D, texColorBuffer);
	glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
	glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texColorBuffer, 0);	// 将它附加到当前绑定的帧缓冲对象
	// 创建渲染缓冲对象
	unsigned int rbo;
	glGenRenderbuffers(1, &rbo);
	glBindRenderbuffer(GL_RENDERBUFFER, rbo);
	glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, SCR_WIDTH, SCR_HEIGHT);
	glBindRenderbuffer(GL_RENDERBUFFER, 0);
	glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);
	// 检查帧缓冲是否完整
	if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
		std::cout << "ERROR::FRAMEBUFFER:: Framebuffer is not complete!" << std::endl;
	glBindFramebuffer(GL_FRAMEBUFFER, 0);
	while(!glfwWindowShouldClose(window)){
		
		/*渲染前处理*/
		
		// 开始渲染
		// ------
		// 绑定到帧缓冲区并绘制场景，就像通常情况下为纹理上色一样（此时绘制的场景不会显示到屏幕上）
		glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
		glEnable(GL_DEPTH_TEST); // enable depth testing (is disabled for rendering screen-space quad)

		/*将场景渲染到帧缓冲中*/
		
		// 现在绑定回默认帧缓冲区，并使用附加的帧缓冲区颜色纹理绘制一个四边形平面
		glBindFramebuffer(GL_FRAMEBUFFER, 0);
		glDisable(GL_DEPTH_TEST); // 关闭深度测试确保screen-space不会因为深度测试被舍弃			
		glClearColor(1.0f, 1.0f, 1.0f, 1.0f); // 清理屏幕
		glClear(GL_COLOR_BUFFER_BIT);
		screenShader.use();
		glBindVertexArray(quadVAO);		// quadVAO通常是一个屏幕大小的四边形，包含三个点
		glBindTexture(GL_TEXTURE_2D, texColorBuffer);	// 使用颜色附加纹理作为纹理的四面
		glDrawArrays(GL_TRIANGLES, 0, 6);
		/*检查并调用事件，交换缓冲*/
	}
	
```

# 6. 立方体贴图
简而言之，立方体贴图就是一个包含了6个2D纹理的纹理，每个2D纹理都组成了立方体的一个面：一个有纹理的立方体。使用立方体贴图的原因是它有一个非常有用的特性，[**它可以通过一个方向向量来进行索引/采样**](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/06%20Cubemaps/#_7)。假设我们有一个1x1x1的单位立方体，方向向量的原点位于它的中心。使用一个橘黄色的方向向量来从立方体贴图上采样一个纹理值会像是这样：

![在这里插入图片描述](https://i.loli.net/2021/06/06/PX38J7WtYdzvO5K.png)

## 6.1 天空盒
立方体贴图一个比较常见的用途是用来渲染场景四周的天空盒，它的纹理加载与普通2D的纹理加载需要注意对R轴的配置：

```cpp
// 立方体贴图要考虑三维情况
glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
```
对于天空盒的渲染，不需要使用**model**矩阵，同时对于view矩阵也需要调整（view矩阵的旋转、缩放和位移都会改变天空盒的所有位置），我们可以用下面的代码让天空盒保持不变：

```cpp
// 取4x4矩阵左上角的3x3矩阵来移除变换矩阵的位移部分
glm::mat4 view = glm::mat4(glm::mat3(camera.GetViewMatrix()));
```
为了让代码更加高效，我们选择**最后渲染天空盒**，因为天空盒默认总是在场景中最远处的位置，最后对它进行渲染可以减少我们调用着色器进行渲染的次数。然而天空盒只是一个1x1x1的立方体，这意味着它很难通过大部分深度测试。我们可以通过以下代码让天空盒的深度值z总是为1.0：
```c
void main()
{
    TexCoords = aPos;
    vec4 pos = projection * view * vec4(aPos, 1.0);
    // 透视除法会在顶点着色器运行之后执行，将gl_Position的xyz坐标除以w分量。
    // 通过将z值设位w，保证了天空盒的深度值总为1.0
    gl_Position = pos.xyww;
}
```

## 6.2 环境映射
我们现在将整个环境映射到了一个纹理对象上了，能利用这个信息的不仅仅只有天空盒。通过使用环境的立方体贴图，我们可以给物体反射和折射的属性。这样使用环境立方体贴图的技术叫做环境映射(Environment Mapping)，其中最流行的两个是反射(Reflection)和折射(Refraction)。
- 其中**反射**的原理与**光照**中的高光十分相似，通过着色位置到摄像机的向量和着色处的法线，我们可以利用GLSL内建的<font color="#00dd00#">reflect</font>函数获取反射向量。然后利用立方体贴图的特性，获取该着色片段的纹理。

```c
void main()
{             
    vec3 I = normalize(Position - cameraPos);	// 入射向量
    vec3 R = reflect(I, normalize(Normal));		// 反射向量
    FragColor = vec4(texture(skybox, R).rgb, 1.0);
}
```

![在这里插入图片描述](https://i.loli.net/2021/06/06/KdUCW38l1OVBumR.png)

- 折射的思想与反射类似，都是通过入射视角和法线关系获取折射向量。与反射一样，利用GLSL内建的<font color="#00dd00#">refract</font>函数，以及[折射率](https://baike.baidu.com/item/%E6%8A%98%E5%B0%84%E7%8E%87#4)可以很容易获取折射向量。（教程里涉及的是单面折射，多面折射需要更加精确的物理分析）

```c
void main()
{             
    float ratio = 1.00 / 1.52;						// 从空气进入玻璃的折射率
    vec3 I = normalize(Position - cameraPos);		// 入射向量
    vec3 R = refract(I, normalize(Normal), ratio);	// 折射向量
    FragColor = vec4(texture(skybox, R).rgb, 1.0);
}
```
![在这里插入图片描述](https://i.loli.net/2021/06/06/EPXpsSmeUvQyNLx.png)
## 6.3 动态环境贴图
上面提到的环境映射是静态环境映射，这存在问题：例如，对于一面可以反射的镜子而言，它能反射的只有四周的环境。这当然不合理，因为它应该优先反射离自己最近的物体。
我们可以利用之前的**帧缓冲为物体的6个不同角度创建出场景的纹理**，并在每个渲染迭代中将它们储存到一个立方体贴图中。之后我们就可以使用这个（动态生成的）立方体贴图来创建出更真实的，包含其它物体的，反射和折射表面了。这就叫做<font color="#00dd00#">动态环境映射</font>(Dynamic Environment Mapping)
动态环境贴图的主要麻烦是：**非常大的性能开销**，因为我们需要为使用环境贴图的物体渲染场景**6**次！现代程序通常会尽可能使用天空盒，并在可能的时候使用预编译的立方体贴图，只要它们能产生一点动态环境贴图的效果。虽然动态环境贴图是一个很棒的技术，但是要想在不降低性能的情况下让它工作还是需要非常多的技巧（研究点）。

# 7. 高级数据
目前为止，我们一直使用glBufferData函数来填充缓冲对象所管理的内存，这个函数会分配一块内存，并将数据添加到这块内存中。如果我们将它的**data**参数设置为**NULL**，那么这个函数将只分配内存但不进行填充。在我们需要**预留**(Reserve)特定大小的内存，之后回到这个缓冲一点一点填充的时候会很有用。
除了使用一次函数调用填充整个缓冲之外，我们可以使用<font color="#00dd00#">glBufferSubData</font>，填充缓冲的特定区域。

```cpp
// 注意在此之前先用glBufferData预留足够大的空间
glBufferSubData(GL_ARRAY_BUFFER, 24, sizeof(data), &data); // 范围： [24, 24 + sizeof(data)]
```
将数据导入缓冲的另外一种方法是，请求缓冲内存的指针，直接将数据复制到缓冲当中：
```cpp
float data[] = {...};
glBindBuffer(GL_ARRAY_BUFFER, buffer);
// 通过调用glMapBuffer函数，OpenGL会返回当前绑定缓冲的内存指针，供我们操作：
void *ptr = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
// 复制数据到内存
memcpy(ptr, data, sizeof(data));
// 记得告诉OpenGL我们不再需要这个指针了
glUnmapBuffer(GL_ARRAY_BUFFER);
```
如果要直接映射数据到缓冲，而不事先将其存储到临时内存中，glMapBuffer这个函数会很有用。比如说，你可以从文件中读取数据，并直接将它们复制到缓冲内存中。

## 7.2 分批顶点属性

之前我们对顶点缓冲采取的都是**交错**(Interleave)处理，即将每一个顶点的位置、发现和/或纹理坐标紧密放置在一起。利用<font color="#00dd00#">glBufferSubData</font>，我们可以采用**分批**(Batched)的方式：

```cpp
float positions[] = { ... };
float normals[] = { ... };
float tex[] = { ... };

/*生成并绑定缓冲对象*/

// 填充缓冲，以下代码省略了glEnableVertexAttribArray()
glBufferData(GL_ARRAY_BUFFER, sizeof(positions) + sizeof(normals) + sizeof(tex), nullptr, GL_STATIC_DRAW);	// 别忘了先为缓冲分配足够内存
glBufferSubData(GL_ARRAY_BUFFER, 0, sizeof(positions), &positions);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions), sizeof(normals), &normals);
glBufferSubData(GL_ARRAY_BUFFER, sizeof(positions) + sizeof(normals), sizeof(tex), &tex);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), 0);  
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)(sizeof(positions)));  
glVertexAttribPointer(
  2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)(sizeof(positions) + sizeof(normals)));
```
## 7.2 复制缓冲
<font color="#00dd00#">glCopyBufferSubData</font>能够让我们从一个缓冲中复制数据到另一个缓冲中

```cpp
void glCopyBufferSubData(GLenum readtarget, GLenum writetarget, GLintptr readoffset, GLintptr writeoffset, GLsizeiptr size);
// readtarget和writetarget参数需要填入复制源和复制目标的缓冲目标
// glCopyBufferSubData会从readtarget中读取size大小的数据，并将其写入writetarget缓冲的writeoffset偏移量处
```
我们可以将<font color="#000066">GL_ARRAY_BUFFER</font>缓冲复制到<font color="#000066">GL_ELEMENT_ARRAY_BUFFER</font>缓冲，但当源和目标都是顶点数组缓冲时，OpenGL提供了额外的两个缓冲目标，叫做<font color="#000066">GL_COPY_READ_BUFFER</font>和<font color="#000066">GL_COPY_WRITE_BUFFER</font>。

```cpp
float vertexData[] = { ... };
glBindBuffer(GL_COPY_READ_BUFFER, vbo1);	// 或者	glBindBuffer(GL_ARRAY_BUFFER, vbo1);
glBindBuffer(GL_COPY_WRITE_BUFFER, vbo2);
glCopyBufferSubData(GL_COPY_READ_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));	// 或者 glCopyBufferSubData(GL_ARRAY_BUFFER, GL_COPY_WRITE_BUFFER, 0, 0, sizeof(vertexData));
```

# 8. 高级GLSL
本章主要介绍有一些用的**内建变量**(Built-in Variable)，管理着色器输入和输出的新方式以及一个叫做**Uniform缓冲对象**(Uniform Buffer Object)的有用工具。了解这些可以让编程更加轻松。
## 8.1 GLSL的内建变量
### 8.1.2 顶点着色器变量
#### gl_PointSize
GLSL定义了一个叫做<font color="#000066">gl_PointSize</font>的输出变量，它是一个float变量，我们可以使用它来设置点的宽高（像素）。在顶点着色器中修改点的大小的话，你就能对每个顶点设置不同的值了。
在顶点着色器中修改点大小的功能默认是禁用的，如果需要启用它的话，需要启用OpenGL的

```cpp
glEnable(GL_PROGRAM_POINT_SIZE);
```
下面的例子重新设置了顶点的像素，使得点的大小会随着观察者距顶点距离变远而增大。

```c
void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);    
    gl_PointSize = gl_Position.z;    
}
```
对每个顶点使用不同的点大小，会在**粒子生成**之类的技术中很有意思

#### gl_VertexID
GLSL还定义了一个有趣的输入变量，我们只能对它进行读取，叫做<font color="#000066">gl_VertexID</font>。它是一个整型变量，储存了正在绘制顶点的当前ID。当（使用glDrawElements）进行索引渲染的时候，这个变量会存储正在绘制顶点的当前索引。当（使用glDrawArrays）不使用索引进行绘制的时候，这个变量会储存从渲染调用开始的已处理顶点数量。（通常情况我们不会用到它，但知道该信息是可读的总是好的）

### 8.1.2 片段着色器
#### gl_FragCoord
<font color="#000066">gl_FragCoord</font>是一个vec4的**只读**的输入变量，其四个分量分别对应x, y, z和1/w。它的x和y分量是片段的窗口空间(Window-space)坐标，其原点为窗口的左下角。x, y是浮点数，且小数部分恒为0.5。x - 0.5和y - 0.5分别位于[0, windowWidth - 1]和[0, windowHeight - 1]内

```cpp
// 我们首先glViewport()设定了一个800x600的窗口
// 以下代码实现了通过x分割屏幕渲染颜色的效果
void main()
{             
    if(gl_FragCoord.x < 400)
        FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    else
        FragColor = vec4(0.0, 1.0, 0.0, 1.0);        
}
```
<font color="#000066">gl_FragCoord</font>的一个常见用处是用于**对比不同片段计算的视觉输出效果**，这在技术演示中可以经常看到。
#### gl_FrontFacing
<font color="#000066">gl_FrontFacing</font>是一个bool型的输入变量。如果我们不（启用<font color="#000066">GL_FACE_CULL</font>来）使用面剔除，它会告诉我们当前片段是属于正向面的一部分还是背向面的一部分。（面剔除章节所提到的根据顶点定义的顺/逆时针来决定是正面还是反面）

```cpp
// frontTexture, backTexture都是预定定义的uniform的纹理变量
void main(
{             
    if(gl_FrontFacing)	// 若为True，则为正面
        FragColor = texture(frontTexture, TexCoords);
    else
        FragColor = texture(backTexture, TexCoords);
}
```
#### gl_FragDepth
GLSL提供给我们一个叫做gl_FragDepth的输出变量，我们可以使用它来在着色器内设置片段的深度值。

```c
// 想要写入深度值，直接将写入一个[0.0, 1.0]的数即可
gl_FragDepth = 0.0;
```
不过修改深度值意味着我们不能进行**提前深度测试**(Early Depth Testing)，因为深度值可变意味着OpenGL无法在着色器运行**前**确定片段的深度值。
不过，从OpenGL **4.2**起，我们仍可以对两者进行一定的调和，在片段着色器的顶部使用深度条件(Depth Condition)重新声明<font color="#000066">gl_FragDepth</font>变量：

```c
layout (depth_condition) out float gl_FragDepth;
// condition可以为下面的值：
// any		  默认值。提前深度测试是禁用的，你会损失很多性能
// greater	  你只能让深度值比gl_FragCoord.z更大
// less		  你只能让深度值比gl_FragCoord.z更小
// unchanged  如果你要写入gl_FragDepth，你将只能写入gl_FragCoord.z的值
```
这样，当深度值比片段的深度值要小的时候，OpenGL仍是能够进行提前深度测试的。

### 8.1.3 接口块
GLSL为我们提供了**接口块**(Interface Block)，便于我们以数组或结构体的形式从顶点着色器向片段着色器之间传递变量。
接口块的声明和<font color="#00dd00#">struct</font>的声明有点相像，不同的是，现在根据它是一个输入还是输出块(Block)，使用<font color="#000066">in</font>或<font color="#000066">out</font>关键字来定义的。

```cpp
// 定义于顶点着色器
out VS_OUT
{
    vec2 TexCoords;
} vs_out;	// vs_out是该结构体的一个实体
```

```c
// 定义于片段着色器
in VS_OUT	// 块名要与顶点着色器保持一致
{
    vec2 TexCoords;
} fs_in;	// fs_in是该结构体的一个实体，它的命名没有规定，但应避免使用误导性名称
```
## 8.2 Uniform缓冲对象
此前在设置uniform变量时，我们面临的一个问题是：当使用多于一个的着色器时，尽管大部分的uniform变量都是相同的，我们还是需要不断地设置它们。
OpenGL为我们提供了一个叫做**Uniform缓冲对象**(Uniform Buffer Object)的工具，它允许我们定义一系列在多个着色器中相同的全局Uniform变量。当使用Uniform缓冲对象时，我们只需要设置相关的**uniform**一次。
Uniform缓冲对象是一个缓冲对象，故我们可以使用<font color="#00dd00#">glGenBuffers</font>来创建它，将它绑定到<font color="#000066">GL_UNIFORM_BUFFER</font>缓冲目标，并将所有相关的uniform数据存入缓冲。
### 8.2.1 Uniform块布局
在Uniform缓冲对象中储存数据是有一些规则的。默认情况下，GLSL会使用一个叫做**共享**(Shared)布局的Uniform内存布局，共享是因为一旦硬件定义了偏移量，它们在多个程序中是共享并一致的（OpenGL没有声明内存块中变量间的间距(Spacing)，这允许硬件能够在它认为合适的位置放置变量）。使用共享布局，GLSL是可以为了**优化**而对uniform变量的位置进行变动的，只要变量的顺序保持不变。这带来的问题是，我们无法清楚的知道每个uniform变量的偏移量，以至于我们不知道如何准确填充Uniform缓冲（尽管OpenGL提供了<font color="#00dd00#">glGetUniformIndices</font>查询这个信息，但这并不直观）。
为了能够准确且更加容易地填充Uniform缓冲对象，我们将Uniform的块布局方式显示的设置为<font color="#000066">std140</font>。
<font color="#000066">std140</font>是一种内存布局方式，在这种内存布局方式下，Uniform块中的每个变量都有一个**基准对齐量**(Base Alignment)，它等于一个变量在Uniform块中所占据的空间（包括填充量(Padding)），这个基准对齐量是使用[std140布局的规则](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_uniform_buffer_object.txt)计算出来的。接下来，对每个变量，我们再计算它的**对齐偏移量**(Aligned Offset)，它是一个变量从块起始位置的字节偏移量。一个变量的对齐字节偏移量必须等于基准对齐量的倍数。

```c
/*
 			常见的对其规则
类型					布局规则
标量，比如int和bool		每个标量的基准对齐量为N。
向量					2N或者4N。这意味着vec3的基准对齐量为4N。
标量或向量的数组			每个元素的基准对齐量与vec4的相同。
矩阵					储存为列向量的数组，每个向量的基准对齐量与vec4的相同。
结构体					等于所有元素根据规则计算后的大小，但会填充到vec4大小的倍数。
*/

// Uniform内存块计算示例（单位是byte）
layout (std140) uniform ExampleBlock
{
                     // 基准对齐量       // 对齐偏移量
    float value;     // 4               // 0 
    vec3 vector;     // 16              // 16  (必须是16的倍数，所以 4->16)
    mat4 matrix;     // 16              // 32  (列 0)
                     // 16              // 48  (列 1)
                     // 16              // 64  (列 2)
                     // 16              // 80  (列 3)
    float values[3]; // 16              // 96  (values[0])
                     // 16              // 112 (values[1])
                     // 16              // 128 (values[2])
    bool boolean;    // 4               // 144
    int integer;     // 4               // 148
}; 
```
虽然std140布局不是最高效的布局，但它保证了内存布局在每个声明了这个Uniform块的程序中是一致的。
除了<font color="#000066">shader</font>和<font color="#000066">std140</font>布局外，还有有一种名为<font color="#000066">packed</font>的布局。它不能保证这个布局在每个程序中保持不变，因为它允许编译器去将uniform变量从Uniform块中优化掉，这在每个着色器中都可能是不同的。
### 8.2.2 使用Uniform缓冲
为了知道Uniform缓冲和Uniform块的对应关系，在OpenGL上下文中，定义了一些绑定点(Binding Point)，我们可以将一个Uniform缓冲链接至它。在创建Uniform缓冲之后，我们将它绑定到其中一个绑定点上，并将着色器中的Uniform块绑定到相同的绑定点，把它们连接到一起。过程如下图所示：

![在这里插入图片描述](https://i.loli.net/2021/06/06/qQTtiskEl82V1BX.png)

我们可以调用<font color="#00dd00#">glGetUniformBlockIndex</font>和<font color="#00dd00#">glUniformBlockBinding</font>实现绑定，例如对于上图中的Light Uniform模块，我们可以采用下面的方式对其绑定：

```c
unsigned int lights_index = glGetUniformBlockIndex(shaderA.ID, "Lights");   
glUniformBlockBinding(shaderA.ID, lights_index, 2);
```
从OpenGL **4.2**版本起，我们也可以添加一个布局标识符，显式地将Uniform块的绑定点储存在着色器中，这样就不用再调用<font color="#00dd00#">glGetUniformBlockIndex</font>和<font color="#00dd00#">glUniformBlockBinding</font>了。下面的代码显式地设置了Lights Uniform块的绑定点。
```c
layout(std140, binding = 2) uniform Lights { ... };
```

### 8.3.3 设置Uniform缓冲
现在，我们已经知道了Uniform缓冲的原理，是时候总结如何设置Uniform缓冲了。
对于着色器，我们以结构体的形式创建Uniform块：
```c
layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
};
```
我们在程序中创建Uniform缓冲，并将其绑定到正确的位置：

```cpp
// 配置uniform缓冲对象
// ---------------------------------
// 首先，获取相关的区块索引
unsigned int uniformBlockIndexRed = glGetUniformBlockIndex(shaderRed.ID, "Matrices");
unsigned int uniformBlockIndexGreen = glGetUniformBlockIndex(shaderGreen.ID, "Matrices");
// 然后我们将每个着色器的统一块链接到这个统一绑定点（将Matrices Uniform块链接到绑定点0）
glUniformBlockBinding(shaderRed.ID, uniformBlockIndexRed, 0);	
glUniformBlockBinding(shaderGreen.ID, uniformBlockIndexGreen, 0);
// 现在，创建缓冲区
unsigned int uboMatrices;
glGenBuffers(1, &uboMatrices);
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferData(GL_UNIFORM_BUFFER, 2 * sizeof(glm::mat4), NULL, GL_STATIC_DRAW);
glBindBuffer(GL_UNIFORM_BUFFER, 0);
// 定义链接到统一绑定点的缓冲区的范围
glBindBufferRange(GL_UNIFORM_BUFFER, 0, uboMatrices, 0, 2 * sizeof(glm::mat4));

// 存储投影矩阵（仅执行一次）（注意：这里不再通过更改FoV来使用缩放）
glm::mat4 projection = glm::perspective(45.0f, (float)SCR_WIDTH / (float)SCR_HEIGHT, 0.1f, 100.0f);
glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
glBufferSubData(GL_UNIFORM_BUFFER, 0, sizeof(glm::mat4), glm::value_ptr(projection));
glBindBuffer(GL_UNIFORM_BUFFER, 0);

// 渲染循环
// -------
while{
	...
	// 渲染
    // ----
    // 在统一块中设置视图矩阵（我们每个循环迭代只需执行一次）
    glm::mat4 view = camera.GetViewMatrix();
    glBindBuffer(GL_UNIFORM_BUFFER, uboMatrices);
    glBufferSubData(GL_UNIFORM_BUFFER, sizeof(glm::mat4), sizeof(glm::mat4), glm::value_ptr(view));
    glBindBuffer(GL_UNIFORM_BUFFER, 0);
	
	/*绘图*/

	...
}
```
# 9. 几何着色器
在顶点和片段着色器之间有一个可选的**几何着色器**(Geometry Shader)，它的输入是一个图元（如点或三角形）的一组顶点。几何着色器可以在顶点发送到下一着色器阶段之前对它们随意变换。然而，几何着色器最有趣的地方在于，它能够将（这一组）顶点变换为完全不同的图元，并且还能生成比原来更多的顶点。
下面例子中，我们首先声明了几何着色输入/输出图元类型，图元值的具体类型可以参照[LearnOpenGL](https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/09%20Geometry%20Shader/#_4)。
```c
#version 330 core
layout (points) in;		// 声明从顶点着色器输入的图元类型
layout (line_strip, max_vertices = 2) out;	// 前者指定几何着色器输出的图元类型。后者指定了它最大能够输出的顶点数量（超过该数量的点不会被绘制）
// 这个例子中，我们发射了两个顶点，它们从原始顶点位置平移了一段距离，之后调用了EndPrimitive，将这两个顶点合成为一个包含两个顶点的线条。
void main() {    
    gl_Position = gl_in[0].gl_Position + vec4(-0.1, 0.0, 0.0, 0.0); 
    EmitVertex();	// 使gl_Position中的向量被添加到图元中

    gl_Position = gl_in[0].gl_Position + vec4( 0.1, 0.0, 0.0, 0.0);
    EmitVertex();	// 使所有发射出的(Emitted)顶点都会合成为指定的输出渲染图元

    EndPrimitive();
}
```
GLSL提供给我们一个内建(Built-in)变量gl_in[]帮助我们生成更有意义的结果：

```c
in gl_Vertex	// 它被声明为一个接口块
{
    vec4  gl_Position;
    float gl_PointSize;
    float gl_ClipDistance[];
} gl_in[];		// 要注意的是，它被声明为一个数组，因为大多数的渲染图元包含多于1个的顶点
```
## 9.1 爆破物体
几何着色器可以支持我们实现爆破效果，所谓的爆破，指的是**将每个三角形沿着法向量的方向移动一小段时间**。其效果是，整个物体看起来像是沿着每个三角形的法线向量爆炸一样。这样的几何着色器效果的一个好处就是，无论物体有多复杂，它都能够应用上去。

## 9.2 法向量可视化
几何着色器的一个很大的作用是：显示任意物体的法向量。当编写光照着色器时，你可能会最终会得到一些奇怪的视觉输出，但又很难确定导致问题的原因。**光照错误很常见的原因就是法向量错误**，这可能是由于不正确加载顶点数据、错误地将它们定义为顶点属性或在着色器中不正确地管理所导致的。我们想要的是使用某种方式来检测提供的法向量是正确的。检**测法向量是否正确的一个很好的方式就是对它们进行可视化**，几何着色器正是实现这一目的非常有用的工具。
其思路是：首先在没有几何体着色器的情况下正常绘制场景，然后再次绘制场景，但是这次仅显示通过几何体着色器生成的法向矢量。几何着色器将三角形图元作为输入，并从它们的法线方向生成3条线——每个顶点一个法线向量。
```c
// 法向量可视化
shader.use();
DrawScene();
normalDisplayShader.use();
DrawScene();
```
# 10. 实例化（Instancing）
想像一下我们有成千上万个相同的、简单的模型，它们的顶点排布一样，贴图纹理一样，区别只是在场景的位置和自身的伸缩、旋转情况。若按照我们此前一直在用的方式进行渲染，那我们很快就会遭遇性能瓶颈，因为这个过程调用了太多次的draw。与渲染实际的顶点相比，告诉GPU渲染你的顶点数据使用像<font color="#00dd00#">gldrawarray</font>或<font color="#00dd00#">glDrawElements</font>这样的函数消耗了相当多的性能，因为OpenGL必须在绘制你的顶点数据之前做必要的准备(比如告诉GPU从哪个缓冲区读取数据，在哪里找到顶点属性以及所有这些相对较慢的CPU到GPU总线)。
这种情况下，我们需要用到实例化（instancing）技术，它可以通过一个渲染调用在一次绘制许多（相等网格数据）对象，从而在每次需要渲染对象时为我们节省了所有CPU-> GPU通信。要使用实例化进行渲染，需要用到<font color="#00dd00#">glDrawArraysInstanced</font>或者<font color="#00dd00#">glDrawElementsInstanced</font>函数。它们带有一个额外的参数—— instance count，来设置我们要渲染的实例数。不过仅此而已还不够，因为绘制出的画面会重叠，为了实现实例化，我们还需要定义每个待渲染物体的
为此，我们需要用到一个GLSL的内建变量<font color="#000066">gl_InstanceID</font>，它代表着每个待渲染物体的ID，下标从0开始。我们只需要在顶点着色器中（以数组形式）定义多个uniform变量，为每个实例做不同的转换，就能渲染出非重叠的图像了。

```c
// 定义顶点着色器
#version 330 core
layout (location = 0) in vec2 aPos;
...
uniform vec2 offsets[100];
void main()
{
    gl_Position = vec4(aPos + offsets[gl_InstanceID], 0.0, 1.0);
	...
}  
```

```cpp
// 渲染100个相同的实体，它们在空间中位置不同
shader.use();
for(unsigned int i = 0; i < 100; i++)
{
    shader.setVec2(("offsets[" + std::to_string(i) + "]")), translations[i]);	// tanslations是一个vector<vec2>的数组
}  
```

## 10.1 实例数组（Instanced arrays）
尽管先前的实现在此特定用例下效果很好，但是每当渲染**100**个以上的实例（这是很常见的）时，我们最终都会达到可发送到着色器的统一数据量的[限制](https://www.khronos.org/opengl/wiki/Uniform_(GLSL)#Implementation_limits)（每个着色器阶段都有可用uniform的数量限制）。 一种替代选择是**实例数组**。 实例数被组定义为一个**顶点属性**（允许我们存储更多数据），它是按实例而不是按顶点更新的。

```cpp
// 将实例数组定义为顶点属性
unsigned int instanceVBO;
glGenBuffers(1, &instanceVBO);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(glm::vec2) * 100, &translations[0], GL_STATIC_DRAW);
glBindBuffer(GL_ARRAY_BUFFER, 0); 
glEnableVertexAttribArray(2);
glBindBuffer(GL_ARRAY_BUFFER, instanceVBO);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 2 * sizeof(float), (void*)0);
glBindBuffer(GL_ARRAY_BUFFER, 0);	
glVertexAttribDivisor(2, 1);  	// 通过将此属性设置为1，我们告诉OpenGL我们想在开始渲染新实例时更新顶点属性的内容。
// glVertexAttribDivisor函数告诉OpenGL何时将顶点属性的内容更新到下一个元素。它的第一个参数是所讨论的顶点属性，第二个参数是属性除数。
```
```c
// 顶点着色器
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aOffset;

out vec3 fColor;

void main()
{
    gl_Position = vec4(aPos + aOffset, 0.0, 1.0);
    fColor = aColor;
}  
```

# 11. 反走样（Anti Aliasing）
渲染画面的物体边缘处有时会出现许多锯齿，尤其是在放大物体时，锯齿会更加明显。这种现象被称之为走样，它产生的本质是因为**场景定义在三维空间中式连续的，而最终显示的却是一个二维离散的数组。所以判断一个点到底没有被某个像素覆盖的时候单纯是一个“有”或“没有"问题，丢失了连续性信息，从而产生锯齿。**（之所以物体边缘容易产生锯齿是因为物体边缘处通常伴随着**频域**的极速变化）
显然，走样的产生与分辨率有关，所以一个很自然的反走样手段是**超采样抗锯齿**(Super Sample Anti-aliasing, SSAA)，它会使用比正常分辨率更高的分辨率（即超采样）来渲染场景，当图像输出在帧缓冲中更新时，分辨率会被下采样(Downsample)至正常的分辨率。不过这种方式的开销太大，如今已鲜有人问津。
## 11.1 多重采样
**多重采样抗锯齿**(Multisample Anti-aliasing, MSAA)是目前一种更聪明的技术，因为它只在光栅化阶段（光栅化：光栅器接受三维空间中顶点作为输入，经过光栅化后将其转换为二维屏幕上的一个片段）判断像素是否被三角形覆盖。多重采样实际做的就是在一个像素中放置多个采样点（自定义采样点的排布），通过判断有多少个采样点落在三角形内部，我们可以对最后的着色做平滑处理。

![在这里插入图片描述](https://i.loli.net/2021/06/06/rTCZ3zkgYeh6oP2.png)

上图展示了放置单一采样点和多个采样点的区别，对于前者我们对像素的处理只有“是”或“不是”；而对于后者我们增加了一些平滑处理，实际的着色情况可以表示为$color=textrue*0.5$。

```cpp
// GLFW负责创建多重采样缓冲
glfwWindowHint(GLFW_SAMPLES, 4);	// 设置采样点
...
glEnable(GL_MULTISAMPLE);			// 开启多重采样
```
## 11.2 离屏MSAA
除了上述所说的由GLFW创建多重采样缓冲，我们也可以使用自己定义的帧缓冲来实现离屏渲染。
有两种方式可以创建多重采样缓冲，将其作为帧缓冲的附件：纹理附件和渲染缓冲附件。这和在帧缓冲教程中所讨论的普通附件很相似。这里有一个完整[示例](https://learnopengl.com/code_viewer_gh.php?code=src/4.advanced_opengl/11.2.anti_aliasing_offscreen/anti_aliasing_offscreen.cpp)。与普通的离屏渲染相比，离屏MSAA需要声明**多重采样**：

```cpp
// 注意区分glTexImage2D
// 它的第二个参数设置的是纹理所拥有的样本个数。
// 最后一个参数设置为GL_TRUE，表明图像将会对每个纹素使用相同的样本位置以及相同数量的子采样点个数。
glTexImage2DMultisample(GL_TEXTURE_2D_MULTISAMPLE, 4, GL_RGB, width, height, GL_TRUE);	
```

```cpp
// 注意到第三个参数被设置为了GL_TEXTURE_2D_MULTISAMPLE，表示多重采样
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D_MULTISAMPLE, tex, 0);
```

```cpp
// 注意区分glRenderbufferStorage
glRenderbufferStorageMultisample(GL_RENDERBUFFER, 4, GL_DEPTH24_STENCIL8, width, height);
```
需要注意的是多重采样缓冲有一点特别，我们不能直接将它们的缓冲图像用于其他运算，比如在着色器中对它们进行采样。一个多重采样的图像包含比普通图像更多的信息，我们所要做的是缩小或者还原(Resolve)图像。调用<font color="#00dd00#">glBlitFramebuffer</font>可以将一个帧缓冲中的某个区域复制到另一个帧缓冲中，并且将多重采样缓冲还原。多重采样离屏渲染流程：
1. 多重采样帧缓冲绘制图形
2. 拷贝多重采样纹理到普通帧缓冲中的纹理对象里
3. 绘制普通帧缓冲中的纹理对象到默认帧（当前屏幕）缓冲

如果我们想对多重采样缓冲渲染的图像做**后期处理**，那我们需要先将其复制到一个**没有使用**多重采样纹理附件的**中介缓冲对象**中，然后用这个普通的颜色附件来做后期处理。因为我们不能直接在片段着色器中使用多重采样纹理。（不过这意味着可能会重新出现锯齿，因为屏幕纹理又变回了一个只有单一采样点的普通纹理。我们可以进行模糊处理或者创造自己的抗锯齿算法）
```cpp
glBindFramebuffer(GL_READ_FRAMEBUFFER, multisampledFBO);	// 将多重采样绑定到只读缓冲中
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, 0);					// 将默认缓冲绑定到只写缓冲中，意味着渲染的画面会出现在屏幕上
// glBindFramebuffer(GL_DRAW_FRAMEBUFFER, intermediateFBO);	// 若要做后期处理，则把一个普通的缓冲绑定到只写缓冲中
glBlitFramebuffer(0, 0, width, height, 0, 0, width, height, GL_COLOR_BUFFER_BIT, GL_NEAREST);	
```
## 11.3 自定义抗锯齿算法
将一个多重采样的纹理图像不进行还原直接传入着色器也是可行的。GLSL为我们提供了对纹理图像的每个子样本采样的选项，因此我们可以创建自己的抗锯齿算法（在大型的图形应用中通常都会这么做）。

```c
// 为了获取每个子样本的颜色值，需要将纹理uniform采样器设置为sampler2DMS
uniform sampler2DMS screenTextureMS;
```
```c
// 使用texelFetch函数能够获取每个子样本的颜色值
vec4 colorSample = texelFetch(screenTextureMS, TexCoords, 3); 
```

