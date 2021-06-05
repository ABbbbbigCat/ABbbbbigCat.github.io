# OpenGL学习笔记——光照


# 1. 颜色
图形学中物体所呈现的颜色可以理解为光照射到该物体上后，该物体所反射出来的颜色，即**物体从一个光源反射各个颜色分量的大小**。

```
// 光的颜色*物体颜色 = 物体反射处的颜色
glm::vec3 lightColor(0.33f, 0.42f, 0.18f);
glm::vec3 toyColor(1.0f, 0.5f, 0.31f);
glm::vec3 result = lightColor * toyColor; // = (0.33f, 0.21f, 0.06f);
```

# 2. 基础光照
最经典的（基础的）光照模型是Bllin-Phong光照模型，该模型定义了环境光(ambient), 漫反射(diffuse)和高光(specular)，它们共同作用于物体来为物体着色。

## 2.1 环境光
环境光由我们自己定义
```c
// 环境光
float ambientStrength = 0.1;
vec3 ambient = ambientStrength * lightColor;
```
## 2.2 漫反射
漫反射的计算需要知道物体表面的**法向量**（垂直于片段表面的一个向量，我们只需要定义三角形顶点的法向量，任一片段表面的法向量可以由插值计算得出），以及定义的**光线**。为了得到余弦值$cos\theta$，需要保证光线和法线都是单位向量，故需要注意对向量进行标准化。
$$L_d =k_d(I/r^2)max(0,n·I)$$
上式将光视作强度（实际上这种说法并不符合物理学定义，因为很难给光的强度赋予实际的物理意义。由物理意义的光源需要借助辐射度量学的知识），其中$k_d$代表漫反射系数，即物体材质颜色。$I/r^2$表示光的强度随距离而衰减。然而，若$I$是类似于太阳的存在，则可以近似的忽略光线强度衰减。我们通常将平行于场景的**平行光**定义为类似于太阳的不会发生衰减的光，而对于**点光源**则认为它会随着距离逐渐衰减。（后续代码中点光源没有考虑衰减只是为了便于学习）
### 2.2.1 法线转换为世界坐标系
考虑到片段着色器中的计算都是在世界坐标系中进行的，相应的，法线也应该转换为世界坐标系。不过这不能简单的乘以转换矩阵。
首先，法向量只是一个方向向量，不能表达空间中的特定位置。因此，如果我们打算把法向量乘以一个模型矩阵，我们就要从矩阵中移除位移部分，只选用模型矩阵左上角3×3的矩阵（可以把法向量的$w$分量设置为0，再乘以4×4矩阵）。对于法向量，我们只希望对它实施缩放和旋转变换。然而不等比缩放会导致法线不再垂直于片元表面，可以通过**法线矩阵**来修正这个错误，其定义为「模型矩阵左上角的逆矩阵的转置矩阵」。
```
Normal = mat3(transpose(inverse(model))) * aNormal;
```
*（注意：对于着色器而言，逆矩阵是一种开销较大的计算。这里选择在着色器中计算是出于学习原因。在实际的工程中，在绘制之前最好用CPU计算出法线矩阵，然后通过uniform把值传递给着色器（像模型矩阵一样））*

## 2.3 镜面反射
镜面反射的结果受观察者（摄像机）影响，通过计算光线经过法线的反射向量与观察者向量之间的夹角，可以得出镜面反射的强度。考虑到这个强度通常较小区分不太明显，因此需要对夹角添加一个$p$次方（加大差异性）。$p$被称作反光度（shininess），其值越大，反光能力越强，散射越小，高光点就会越小。
OpenGL内置的反射函数可以帮助我们轻松获得反射向量（冯模型，一个明显的问题是视角和反射光线的夹角可能大于90°导致高光为零）。我们也可以不计算反射向量，利用$normalize(I + v)$求半程向量$h$，通过$h·n$近似获得反射向量与观察者向量间的夹角（Blinn-Phong模型，避免了之前所说的高光为零的情况。对于反光度很小的物体，Blinn-Phong着色的渲染效果更加真实）。
$$L_s=k_s(I/r^2)max(cos\alpha, 0)=k_s(I/r^2)max(h·n,0)$$


## 2.4 基于BIIion-Phong模型的片元着色器
```c
#version 330 core
out vec4 FragColor;

uniform vec3 lightPos;
uniform vec3 objectColor;
uniform vec3 lightColor;
uniform vec3 viewPos;

in vec3 FragPos;
in vec3 Normal;

void main()
{
    // 环境光
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * lightColor;

    // 漫反射
    vec3 norm = normalize(Normal);
    vec3 lightDir = normalize(lightPos - FragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = diff * lightColor;

    // 镜面光照
    float specularStrength = 0.8;
    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 reflectDir = reflect(-lightDir, norm);                 // reflect()要求向量的方向从片元指向光源，这与lightDir正相反
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);   // 32是反光度，反光度越高，反射光的能力越强，散射得越少，高光点就会越小
    vec3 specular = specularStrength * spec * lightColor;

    vec3 result = (ambient + diffuse + specular) * objectColor;
    FragColor = vec4(result, 1.0);
}
```
以上着色器（冯氏着色器）是在片元上进行着色，基础的光照模型也可以考虑在顶点进行着色（Gouraud着色）。其优势在于需要处理的点更少，效率高。然而，顶点着色器中的最终颜色值是仅仅只是那个顶点的颜色值，片段的颜色值是由插值光照颜色所得来的。结果就是这种光照看起来不会非常真实（甚至有些奇怪），除非使用了大量顶点。

# 3. 材质
根据基础光照模型可知，物体最终的着色情况主要取决于光线和物体材质。通过将光线和物体材质进行封装，可以更加便捷的管理光照和材质。（然而，实际的物体材质很少会完全一致，这里只是最简单的模型）

```c
// 封装了光与材质的片元着色器
#version 330 core

// 定义物体材质
struct Material {
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
    float shininess;
}; 

// 定义光照情况
struct Light {
    vec3 position;

    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};

out vec4 FragColor;

uniform Material material;
uniform Light light;
uniform vec3 viewPos;

in vec3 FragPos;
in vec3 Normal;

void main()
{
    // 环境光
    vec3 ambient = material.ambient * light.ambient;

    // 漫反射
    vec3 norm = normalize(Normal);
    vec3 lightDir = normalize(light.position - FragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = material.diffuse * diff * light.diffuse;

    // 镜面光照
    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 reflectDir = reflect(-lightDir, norm);                 // reflect()要求向量的方向从片元指向光源，这与lightDir正相反
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);   
    vec3 specular = material.specular * spec * light.ambient;

    vec3 result = ambient + diffuse + specular;
    FragColor = vec4(result, 1.0);
}
```
# 4. 光照贴图
上一章定义了一个最简单的材质模型，本章通过引入漫反射贴图和镜面光贴图以期让材质更加真实。漫反射贴图的引入可以让我们省略环境光材质的定义，因为环境光颜色在几乎所有情况下都等于漫反射颜色。镜面光贴图的引入是为了让物体的高光呈现更加真实的效果。
下面的代码除了引入反射贴图和镜面光贴图外，还引入了发射光贴图（发射光贴图是指物体本身会发光的情况。冯氏模型中，我们虽然可以模仿物体本身发光，但却难以反映出这些光对于周围物体的着色影响。）
```c
#version 330 core

// 定义物体材质
struct Material {
    // 移除环境光，因为环境光颜色在几乎所有情况下都等于漫反射颜色
    sampler2D diffuse;
    // 根据材质判断物体表面是否应该形成高光
    sampler2D specular;
    // 物体自身发光贴图
    sampler2D emission;
    float shininess;
}; 

// 定义光照情况
struct Light {
    vec3 position;

    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};

out vec4 FragColor;

uniform Material material;
uniform Light light;
uniform vec3 viewPos;

in vec2 TexCoords;          // 获取材质坐标
in vec3 FragPos;
in vec3 Normal;


void main()
{
    // 环境光
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));

    // 漫反射
    vec3 norm = normalize(Normal);
    vec3 lightDir = normalize(light.position - FragPos);
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = light.diffuse *  diff * vec3(texture(material.diffuse, TexCoords));

    // 镜面光照
    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 reflectDir = reflect(-lightDir, norm);                 // reflect()要求向量的方向从片元指向光源，这与lightDir正相反
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);   
    vec3 specular = light.ambient * spec * vec3(texture(material.specular, TexCoords));;

    // 发光
    vec3 emission = texture(material.emission, TexCoords).rgb;

    vec3 result = ambient + diffuse + specular + emission;
    FragColor = vec4(result, 1.0);
}
```
# 5. 透光物
本部分主要讨论不同的光源类型对于着色的影响。
## 5.1 平行光
平行光模拟的是无限远处光源对于着色器的影响（类似于太阳）。因此再着色器中定义光源时，我们不再需要知道光源的位置，而是**定义光照向量**。
```
struct Light {
    // vec3 position; // 使用定向光就不再需要了
    vec3 direction;

    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};
...
void main()
{
  vec3 lightDir = normalize(-light.direction);
  ...
}
```
注意到上面求光照方向时，对方向取反，这是因为定义时通常习惯定义光到片元的向量。而在实际使用时，我们是用片元到光的向量来计算漫反射和镜面反射的。
```
// 设置光照方向（我们总是定义从光到场景的方向，很容易看出这是一个向下照射的光）
lightingShader.setVec3("light.direction", -0.2f, -1.0f, -0.3f);
```
## 5.2 点光源
### 5.2.1 衰减
与平行光不同，*点光源*(Point Light)需要考虑能量的衰减（Attenuation）。能量衰减与光源到着色点间的距离成反比，衰减公式的定义如下：
$$F_{at}=1.0/(K_c + k_l*d+k_q*d^2)$$
其中$k_c$是常数项衰减因子，始终定义为1.0，保证分母始终大于1.0。$k_l$和$k_q$分别为一次项和二次项。从定义可以看出，由于二次项的存在，光线会在大部分时候以线性的方式衰退，直到距离变得足够大，让二次项超过一次项，光的强度会以更快的速度下降。

![光的衰减情况](https://i.loli.net/2021/06/06/gStWmAXEwIF786N.png)

### 5.2.2 选值
衰减因子的选择是一个问题。正确地设定它们的值取决于很多因素：环境、希望光覆盖的距离、光的类型等。在大多数情况下，这都是经验的问题，以及适量的调整。下面这个表格显示了模拟一个（大概）真实的，覆盖特定半径（距离）的光源时，这些项可能取的一些值。第一列指定的是在给定的三项时光所能覆盖的距离。这些值是大多数光源很好的起始点，它们由[Ogre3D](http://wiki.ogre3d.org/tiki-index.php?page=-Point+Light+Attenuation)的Wiki所提供：
|范围| 常数项 |一次项|二次项|
|--|--|--|--|
|7|1.0|0.7|1.8|
|13|	1.0|	0.35	|0.44|
|20|	1.0|	0.22 |0.20|
|32|	1.0|	0.14|	0.07|
|50|	1.0|	0.09|	0.032|
|65|	1.0|	0.07|	0.017|
|100|	1.0|	0.045|	0.0075|
|160|	1.0|	0.027|	0.0028|
|200|	1.0|	0.022|	0.0019|
|325|	1.0|	0.014|	0.0007|
|600|	1.0|	0.007|	0.0002|
|3250|	1.0|	0.0014|	0.000007|
### 5.2.3 实现衰减
为了实现衰减，需要在着色器中光的定义里添加常数项，一次项和二次项。并根据衰减定义，将器附加到光照上。（对于环境光，我们可以将环境光分量保持不变，让环境光照不会随着距离减少，但是如果我们使用多于一个的光源，所有的环境光分量将会开始叠加，所以在这种情况下我们也希望衰减环境光照）

```
struct Light 
{
	...

    float constant;
    float linear;
    float quadratic;
};

...
void main
{
	...
	float distance    = length(light.position - FragPos);
	float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));
	...
	ambient  *= attenuation; 
	diffuse  *= attenuation;
	specular *= attenuation;
}
```
## 5.4 聚光
### 5.4.1 聚光的定义
*聚光*（Spotlight）是位于环境中某个位置的光源，它只朝一个特定方向而不是所有方向照射光线。这样的结果就是只有在聚光方向的特定半径内的物体才会被照亮，其它的物体都会保持黑暗。聚光很好的例子就是路灯或手电筒。
OpenGL中定义的聚光用一个世界空间位置、一个方向和一个*切光角*(Cutoff Angle)来表示的，切光角指定了聚光的半径（圆锥的半径）。对于每个片段，我们会计算片段是否位于聚光的切光方向之间（也就是在锥形内），如果是的话，我们就会相应地照亮片段。下面这张图会让你明白聚光是如何工作的：

![在这里插入图片描述](https://i.loli.net/2021/06/06/3q6lWgNvcAOdHB9.png)

 - LightDir：从片段指向光源的向量。
 - SpotDir：聚光所指向的方向。
 - $ϕ$：指定了聚光半径的切光角。落在这个角度之外的物体都不会被这个聚光所照亮。
 - $θ$：LightDir向量和SpotDir向量之间的夹角。在聚光内部的话θ值应该比ϕ值小。
通过公式$LightDir·SpotDir$可以获取$θ$的余弦值，并将它与与切光角$ϕ$对比。（注意，这里用的是余弦值！）
### 5.2 手电筒
手电筒（Flashlight）是一种典型的聚光光源，其位置和方向会随着玩家的位置和方向不断更新。（考虑到聚光的特性，其对环境光贡献通常较小，甚至没有）

```
// 定义手电筒
struct Light {
    vec3  position;
    vec3  direction;
    float cutOff;
    ...
};
...
void main() {
	...
	float theta = dot(lightDir, normalize(-light.direction));
	if(theta > light.cutOff) 		// //请记住，这里用的是角的余弦而不是度，所以这里用的是>
	{       
	  // 执行光照计算
	}
	else  // 否则，使用环境光，让场景在聚光之外时不至于完全黑暗
  		color = vec4(light.ambient * vec3(texture(material.diffuse, TexCoords)), 1.0);
}
```
### 5.3 平滑/软化边缘
按照上述做法，最后得到的效果并不如意，这是因为光照的边缘处的差异过于极端。如下图所示，注意到光边缘处内外明暗十分显著，缺乏平滑过渡。为了让聚光效果显得更加真实，我们需要对边缘做平滑/软化处理。

![在这里插入图片描述](https://i.loli.net/2021/06/06/Uhmk5MFZrcBdNQb.png)

为了创建一种看起来边缘平滑的聚光，我们需要模拟聚光有一个*内圆锥*(Inner Cone)和一个*外圆锥*(Outer Cone)。内圆锥即上面的光照部分，而外圆锥则主要起到让光从内圆锥逐渐变暗直到外圆锥边界的效果。
为了创建一个外圆锥，我们只需要再定义一个余弦值来代表聚光方向向量和外圆锥向量（等于它的半径）的夹角。然后，如果一个片段处于内外圆锥之间，将会给它计算出一个0.0到1.0之间的强度值。如果片段在内圆锥之内，它的强度就是1.0，如果在外圆锥之外强度值就是0.0。公式如下（所有角度都代表余弦值）：
$$I=({\theta}-{\gamma})/{\epsilon}$$

其中，${\epsilon}={\phi}-{\gamma}$是内($\phi$)外($\gamma$)圆锥的余弦差值。最终的$I$值就是在当前片段聚光的强度。[learnOpenGL](https://learnopengl-cn.github.io/02%20Lighting/05%20Light%20casters/#_8)中给出了一些设置情况：
| $\theta$ |$\theta$(角度)  |$\phi$|$\phi$(角度)|$\gamma$|$\gamma$(角度)|$\epsilon$|$I$|
|--|--|--|--|--|--|--|--|
|0.87|	30|	0.91|	25|	0.82|	35|	0.91 - 0.82 = 0.09|	0.87 - 0.82 / 0.09 = 0.56|
|0.9|	26|	0.91|	25|	0.82|	35|	0.91 - 0.82 = 0.09|	0.9 - 0.82 / 0.09 = 0.89|
|0.97|	14|	0.91|	25|	0.82|	35|	0.91 - 0.82 = 0.09|	0.97 - 0.82 / 0.09 = 1.67|
|0.83|	34|	0.91|	25|	0.82|	35|	0.91 - 0.82 = 0.09|	0.83 - 0.82 / 0.09 = 0.11|
|0.64|	50|	0.91|	25|	0.82|	35|	0.91 - 0.82 = 0.09|	0.64 - 0.82 / 0.09 = -2.0|
|0.966|	15|	0.9978|	12.5|	0.953|	17.5|	0.966 - 0.953 = 0.0448|	0.966 - 0.953 / 0.0448 = 0.29|


根据以上定义，我们可以重新定义我们的片段着色器：

```
...
struct Light {
	...
    float outerCutOff;		// 
    ...
};
...
void main		// 内外边缘的引入让我们不再需要判断着色片段是否在范围内，因为intensity的计算完成了这个工作
{
	...
	float theta     = dot(lightDir, normalize(-light.direction));
	float epsilon   = light.cutOff - light.outerCutOff;
	float intensity = clamp((theta - light.outerCutOff) / epsilon, 0.0, 1.0);    // 把第一个参数约束在了0.0到1.0之间
	...
	// 将不对环境光做出影响，让它总是能有一点光
	diffuse  *= intensity;
	specular *= intensity;
	...
}
```
边缘软化后的效果：

![软化后的结果](https://i.loli.net/2021/06/06/EvjuryPz6OFKt71.png)

# 6. 多光源
多光源的本质就是多种类型的光共同作用于物体后产生的着色效果。我们可以将各种光源的着色过程封装为相应的函数，并累加其计算结果得到最后的着色效果。

```c
#version 330 core
out vec4 FragColor;

// 定义材质
struct Material {
    sampler2D diffuse;
    sampler2D specular;
    float shininess;
}; 

// 定义平行光
struct DirLight {
    vec3 direction;
	
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};

// 定义点光源
struct PointLight {
    vec3 position;
    
    float constant;
    float linear;
    float quadratic;
	
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
};

// 定义聚光
struct SpotLight {
    vec3 position;
    vec3 direction;
    float cutOff;
    float outerCutOff;
  
    float constant;
    float linear;
    float quadratic;
  
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;       
};

#define NR_POINT_LIGHTS 4

in vec3 FragPos;
in vec3 Normal;
in vec2 TexCoords;

uniform vec3 viewPos;
uniform DirLight dirLight;
uniform PointLight pointLights[NR_POINT_LIGHTS];
uniform SpotLight spotLight;
uniform Material material;

// 函数定义
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir);
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir);
vec3 CalcSpotLight(SpotLight light, vec3 normal, vec3 fragPos, vec3 viewDir);

void main()
{    
    // 属性
    vec3 norm = normalize(Normal);
    vec3 viewDir = normalize(viewPos - FragPos);
    
    // 计算平行光
    vec3 result = CalcDirLight(dirLight, norm, viewDir);
    // 计算点光源（由于有四个点光源，故计算四次）
    for(int i = 0; i < NR_POINT_LIGHTS; i++)
        result += CalcPointLight(pointLights[i], norm, FragPos, viewDir);    
    // 计算聚光
    result += CalcSpotLight(spotLight, norm, FragPos, viewDir);    
    
    FragColor = vec4(result, 1.0);
}

// 计算平行光着色情况
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir)
{
    vec3 lightDir = normalize(-light.direction);
    // 漫反射
    float diff = max(dot(normal, lightDir), 0.0);
    // 镜面反射
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    // 混合后结果
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    return (ambient + diffuse + specular);
}

// 计算点光源着色情况
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir)
{
    vec3 lightDir = normalize(light.position - fragPos);
    // 漫反射
    float diff = max(dot(normal, lightDir), 0.0);
    // 镜面反射
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    // 衰减因子
    float distance = length(light.position - fragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));    
    // 混合后结果
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    ambient *= attenuation;
    diffuse *= attenuation;
    specular *= attenuation;
    return (ambient + diffuse + specular);
}

// 计算聚光着色情况
vec3 CalcSpotLight(SpotLight light, vec3 normal, vec3 fragPos, vec3 viewDir)
{
    vec3 lightDir = normalize(light.position - fragPos);
    // 漫反射
    float diff = max(dot(normal, lightDir), 0.0);
    // 镜面反射
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    // 衰减因子
    float distance = length(light.position - fragPos);
    float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));    
    // 聚光强度
    float theta = dot(lightDir, normalize(-light.direction)); 
    float epsilon = light.cutOff - light.outerCutOff;
    float intensity = clamp((theta - light.outerCutOff) / epsilon, 0.0, 1.0);
    // 衰减因子
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    ambient *= attenuation * intensity;
    diffuse *= attenuation * intensity;
    specular *= attenuation * intensity;
    return (ambient + diffuse + specular);
}
```

