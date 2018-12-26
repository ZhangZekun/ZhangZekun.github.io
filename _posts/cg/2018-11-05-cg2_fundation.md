---
layout: post
date: 2018-11-05
title: 计图渲染基础
description: 记录学习过程中，一些比较重要的点，可能知识点有点零碎，只适合我自己一个人看。
categories: [cg]
---


* Do not remove this line (it will not be displayed)
{:toc}
# 渲染场景图像的两个步骤

1. Visibility

即决定场景中哪些面或点是可见的，有光线追踪和光栅化两种方法可以解决。

**其中光栅化在做隐藏面剔除时有两种算法：z-buffer和???painter's algorithms，Newell's algorithm**

注意，考虑到有些物体可能会有半透明性质，所以z-buffer的实现需要保存所有映射到某像素上的点的信息，以便于后期计算像素颜色时进行进行混色。

2. Shading



# 商用渲染器

REYES算法：皮克斯工作室出品的渲染器RenderMan所使用的基于光栅化技术的算法。

RIS 算法：皮克斯工作室最新所使用的基于光线追踪技术的算法。



# 全局光照算法

1. 也可以使用光栅化渲染来模拟间接光,实现全局光照算法，常见的方法有???point cloud based, photon maps, virtual point lights, shadow maps等。但是如果用光栅化处理visibility problem，就需要用另一种方法来处理全局光照，而光线追踪则只需要在一种逻辑内同时做着两件事。
2. ???**Radiosity**（辐射度算法)也可以用来模拟全局光照
3. ???Photon maps is a good example of a technique designed to efficiently simulate caustics (a mirror reflecting light onto a diffuse suface for example — which is a form of indirect specular reflection) which are very hard or computational expensive to simulate with ray tracing.



# 光线追踪算法的部分缺点

1. 计算代价高，这个在上一篇博客有提到过。

2. ???会引入噪声。

   ![光线经过玻璃球后聚焦到P点](/../../blogImages/cg2_1.png)

3. 使用unidirectional path tracing（单向追踪算法）如后向追踪的话，难以模拟一些光的效果，如caustics（焦散线），即光线经过镜头，聚焦到某一点上的效果,如上图的黄色光线。或者模拟LS+DE型光线传播路径（光线经过多次镜面反射，再经过一次漫反射到达眼睛）的话也比较难。

   两者的原因都是一样的。在到达眼睛之前，光线经过漫反射物体表面一点P，我们在实际模拟中会以P点法向量为中心的半球体内随机发射一定数量的光线，这些光线很可能不会经过如上图的玻璃球，所以无法模拟出焦散线效果。这些光线也可能会在经过多次镜面反射之后，无法到达光源处，所以无法得到颜色信息，相当于这些光线对LS+DE型光线传播路径起不到模拟效果。

   所以单向追踪有其局限性。这个问题可以用**Photon mapping算法** 来解决。该算法需要在渲染前进行预计算，然后在渲染时使用预计算的结果。

   后来为了解决photon mapping算法需要预计算，算法和光线追踪技术的框架不一致等问题，提出了其他算法，如 bi-directional path tracing等。



# 待了解术语和技术

1. **Monte Carlo ray tracing**

2. **unidirectional path tracing**

3. **Photon mapping**

4. bi-directional path tracing.

5. unidirectional path tracing, bi-directional path tracing, metropolis light transport, instant radiosity, photon mapping, radiosity caching

   ​

# 重要概念或知识点

1. When the algorithm（ray trace） was first developed, Appel and Whitted only considered the case of mirror surfaces and transparent objects.​