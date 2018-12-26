---
layout: post
date: 2018-11-24
title: learn-opengl上的light章节总结和重要点记录
description: learn-opengl上的light章节总结和重要点记录
categories: [cg]
---


* Do not remove this line (it will not be displayed)
{:toc}
# 三种光源类型介绍

## Directional Light

平行光的典型就是太阳光。

在模拟平行光时，**我们用一个向量表示光的方向，不需要光的位置**。我们也可以理解为，一个无穷远的平面将空间一分为二，该平面与平行光垂直，不断发射出光线。

~~~c++
struct Light {
    // vec3 position; // No longer necessery when using directional lights.
    vec3 direction;
    vec3 intensity;
};
...
void main()
{
  vec3 lightDir = normalize(-light.direction);
  ...
}

~~~

这里默认的是光不会衰减。

## Point lights

用一个点表示光源，所以我们只需要光的位置和光强。

~~~C++
struct Light {
    vec3 position; 
    vec3 intensity;
};
...

~~~

为了让点光源更加真实，一般我们会加入光的衰减机制。即保证光源在一定短距离内，衰减比较快速；超过一定距离之后，会衰减得比较慢，这样和现实中点光源的情况更为接近。

我们使用以下公式来表示光的衰减：

$$F_{att} = \frac {1.0}{K_c + K_l * d + K_q * d^2} $$

$F_{att}$ 是一个小数，取值范围为$(0, 1]$ 。$K_c, K_l, K_q$ 分别是常数系数、线性系数、平方系数，一般$K_c$ 为1， $K_l 和K_q$ 根据点光源的照亮半径适当调整。 

适当选取系数，$F_att$ 随着距离的变化如下：

![点光源衰减](/../../blogImages/cg4_1.png)



场景使用单个点光源时，我们可以在片段着色器中用该系数乘以最终计算出来的片段颜色，得到衰减后的光照效果。

## Spotlight

![点光源衰减](/../../blogImages/cg4_2.png)

聚光灯具有位置，方向，照亮范围这三个数据，即：

~~~c++
struct Light {
    vec3  position;
    vec3  direction;
    float cutOff;
    ...
};    
~~~

我们在片段着色器判断片段是否在聚光灯作用范围内时，可以用片段与聚光灯位置形成的方向向量，去与聚光灯方向向量进行叉乘得到$\theta$， 再判断其与cutOff的大小关系来确定。

注意cutOff不是角度，而是$cutOff = cos(angle)$

但使用聚光灯会出现照亮区域边缘过度不平滑的问题，从而造成不真实感，如下：

![点光源衰减](/../../blogImages/cg4_3.png)

为了解决这个问题，引入了内外聚光区域的概念。

原本的聚光灯只有一个内聚光区域，在区域内的物体都会被照亮。现在聚光灯多了一个外聚光区域，该区域面积会覆盖住内聚光区域。落在内聚光的物体会被完全照亮，落在外聚光区域外的物体会完全变暗，落在外聚光区域但不在内聚光区域的物体接收的光强会从1减少到0。

我们使用以下公式来进行光强的计算：

$$ I =  \frac {cost(\theta) - cos(\gamma)}{cos(\phi) - cos(\gamma)}$$

其中$\theta$ 是光照方向和聚光灯方向的夹角角度， $\phi$ 是内聚光区域角度， $\gamma$ 是外聚光区域角度， 很容易可以看出，$\theta$ 在$[\phi, \gamma]$ 变化时，$I$ 会从1减到0。

使用这种方法，得到的效果如下：

![点光源衰减，引入聚光灯内外区域](/../../blogImages/cg4_4.png)



# 多光源（不考虑阴影）

假设场景中三种类型的光源，只需要分别计算再叠加即可。因为没有考虑阴影，非常的简单。



# Phong Light Model

冯氏光照模型使用环境光照、漫反射光照、镜面反射光照的叠加来模拟物体表面反射的光。

## Ambient lighting

就是直接给物体表面一个常数颜色，用来简单地模拟场景其他物体反射、折射的光对物体的影响

## diffuse diffuse lighting

![漫反射](/../../blogImages/cg4_5.png)

利用光源到片段位置形成的向量，使用该向量与法线向量的夹角的余弦值作为系数，该系数乘以光强，就是漫反射光照的强度了。

## Specular Lighting

![漫反射](/../../blogImages/cg4_6.png)

镜面反射光照，就是利用视线和反射光线的夹角$\theta$，使用$cos^{a}(\theta)$ 作为系数，其中a是一个常数系数，一般会大于1。用系数乘以光强，就是镜面反射光照的强度了。注意在冯氏光照模型中，$cos(\theta)$会被约束到[0, 1]范围内，当$\theta > 90^\circ$， 最终系数会变为0。

a的大小会影响高亮区域面积的大小，如下：

![镜面反射参数的影响](/../../blogImages/cg4_7.png)

# Blinn-Phong

## 为什么需要用Blinn-Phong

冯氏光照模型的镜面反射有一个缺点，Blinn-Phong就是对冯氏光照的镜面反射做了改进。

![镜面反射参数的影响](/../../blogImages/cg4_8.png)



左图时理想情况下镜面反射光照计算的示意图。

右图是一个特殊情况，就是视线和反射光线的夹角大于90度，按冯氏光照模型的做法，这个夹角的余弦值会被约束成0，即该点的镜面放射光照强度为0。

在该光照点的左侧，会有一个光照点使得夹角刚好是90度，此时光照强度为0。再往左移动一点，夹角会逐渐变小，如89°，此时$cos(\theta) = 0.01745 $ 。当a很大时，如a = 32时，$cos^{a}(\theta)$会无限接近于0。再往左移动一点，夹角会逐渐变小，如60°，当a很大时，如a = 32时，$cos^{a}(\theta)$ 无限接近于0。再往左移动一点，夹角会逐渐变小，如30°，当a很大时，如a = 32时，$cos^{a}(\theta) = 0.0100$  。也就是说，**当a很大时**，在90度点左侧，已经有一个平缓的从高光到无光的过度，不会出现说在90度点的右侧突然镜面高光消失的情况。

但当a很小时，如a=1时，还是以上面三个数字为例子。在90度点往左移动一点，夹角会逐渐变小，如89°，此时$cos^{a}(\theta)= 0.01745 $ 。再左移，夹角为60°，$cos^{a}(\theta) = 0.5 $。 再左移，夹角为30°，$cos^{a}(\theta) = 0.8660 $ 。可以发现，当夹角为89°时，镜面高光还很强烈，但到了90°时镜面高光直接消失，所以会出现下面这种情况：

![冯氏光照的缺点](/../../blogImages/cg4_9.png)

## 怎么使用Blinn-Phong

![Blinn-Phong示意图](/../../blogImages/cg4_10.png)



Blinn-Phong就是在计算镜面反射光照时，用法线向量和中线向量的夹角，代替之前冯氏光照计算镜面反射光照所使用的夹角。

中线向量就是单位化后光向量和视线向量相加的结果。

主要GLSL代码如下：

~~~c++
vec3 lightDir   = normalize(lightPos - FragPos);
vec3 viewDir    = normalize(viewPos - FragPos);
vec3 halfwayDir = normalize(lightDir + viewDir);
float spec = pow(max(dot(normal, halfwayDir), 0.0), shininess);
vec3 specular = lightColor * spec;
~~~

使用这个新的夹角，能够保证夹角时在0到90°之间，可以有效防止镜面高光突然截断的问题。

效果如下：

![Blinn-Phong示意图](/../../blogImages/cg4_11.png)



# Gamma correction

## 什么是伽马校正

![两种颜色空间](/../../blogImages/cg4_12.png)

我们知道，我们一般用8位来表示一个灰度像素值。这个位于[0, 255]之间的数值，我们可以将其归一化为[0, 1]之间的一个浮点数，即用0到1的浮点数来表示一个灰度像素值。

在上图中，physical brightness的每一个颜色块之间的亮度（即电脑屏幕发射出的光强度）间隔是保持一致的，即$Δ_{linear}=I_n−I_{n−1}$保持不变。但奇怪的是，我们人眼会觉得，0.0和0.1之间的亮度差距明显比0.9和1.0之间的更大。

而上图的Perceived brightness，人眼看起来会觉得亮度变化是比较均匀的，特别是在较暗的方块之间。但实际上，每一个方块和相邻方块亮度的差距，是不一致的。

**我们之前说的用[0, 1]表示像素值，我们默认的单位都是屏幕亮度，0表示没有亮度，0.5表示0.5个亮度，1表示1个亮度，即白光。这一种表示，我们可以称之为physically linear space**

**现在我们依然用[0, 1]表示像素值，但这次我们默认的单位是人眼亮度，0表示人眼没有感受到亮度，1表示人眼感受到白光，0.5表示人眼感受到黑白光强的一半。这一种表示，我们可以称之为perceptually linear space**

现代的电子设备，如打印机，显示器等，希望获得的输入像素值，都是位于perceptually linear space内，然后设备内部会对该值进行处理，将其变回到physically linear space，再在屏幕上发出对应亮度的光。

**读入一张physically linear space下的图片，将其所有像素值转到perceptually linear space下，就是Gamma correction，也叫Gamma encoding。转化公式为：** $V_{Physical Value} = V_{Perceived Value} ^ {2.2}，V_{Perceived Value} = V_{Physical Value} ^ {\frac 1 {2.2}}$

## 为什么需要伽马矫正

learn opengl和相关博客都有提到一些原因，但没有说得很清楚，我这里说一下自己的理解。

我觉得主要就是为了方便。以下图为例：

![两种颜色空间](/../../blogImages/cg4_12.png)

如果有一个艺术家，想在电脑上用软件画出像Perceived brightness这样人眼亮度间隔均匀的11个方格。如果我们是在physically linear space上进行绘画，那么他需要在调色板上人工地选11个数值来表示0到1.0之间均匀变化的亮度，（伽马矫正的计算方法$V_{Physical Value} = V_{Perceived Value} ^ {2.2}$，我计算了一下，应该选取的像素值为0，0.0063，0.0289，0.0707，0.1332，.....，1.0）。

很明显，在physically linear space很难进行操作，但如果在perceptually linear space进行绘图，我们只需要选取（0.0, 0.1, 0.2, 0.3, ....,1.0）就可以表示人眼亮度间隔均匀的11个方格。

**也就是说，伽马矫正提供了一种方法，即我们可以通过将数值翻倍，从而达到人眼亮度翻倍的效果。**

## 图像处理算法应在physically linear space下进行

值得注意的是，常见的图像格式，如jpg, png，里面保存的像素数据，可能是在physically linear space下，也可能是在perceptually linear space下，如果是在perceptually linear space下该图就是sRGB类型。

插件的图像处理算法，如像素点之间插值，颜色混合，透明混合，图像放缩，防走样等算法，都要求图像是在physically linear space下。这样处理出来的效果会与现实情况比较相符。

其实也很好理解，因为physically linear space下的数值表示的是物理世界中的光强，而perceptually linear space表示的是人眼感受到的光强，要进行各类计算，当然不能以人的感受为标准，要符合物理世界规律才会更为准确。

## opengl编程中需要注意伽马矫正的点

上面提到，图像处理算法应该在physically linear space上进行。所以，当有涉及颜色的操作时，我们就需要特别注意了。

最常见的，贴图映射时，贴图有两种类型，一种是sRGB(即像素数值都是在perceptually linear space下)，一种是在physically linear space下。当我们进行采样时，如果图像数据是physically linear space下的，那么就可以直接取值计算。如果图像是sRGB类型，则需要先转化到physically linear space下再进行计算。如下：

~~~C++
float gamma = 2.2;
vec3 diffuseColor = pow(texture(diffuse, texCoords).rgb, vec3(gamma));
~~~

在片段着色器中，我们会得到片段的最终颜色。因为计算过程都是在physically linear space下进行的，我们需要对计算结果进行伽马矫正，得到perceptually linear space下的表示。可以使用下面的方法：

~~~c++
void main()
{
    // do super fancy lighting 
    [...]
    // apply gamma correction
    float gamma = 2.2;
    FragColor.rgb = pow(fragColor.rgb, vec3(1.0/gamma));
}
~~~

这样输入到缓存里面的就是perceptually linear space下的表示，显示器最终才能输出正确的结果。

## 有用的链接

 [很详细介绍伽马矫正的博客](http://blog.johnnovak.net/2016/09/21/what-every-coder-should-know-about-gamma/)

[Learn Opengl相关文章](https://learnopengl.com/Advanced-Lighting/Gamma-Correction)

# Shadow Mapping

