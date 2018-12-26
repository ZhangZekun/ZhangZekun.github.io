---
layout: post
date: 2018-11-21
title: opengl渲染管线流程简介和重要知识点记录
description: 简单介绍opengl渲染管线，对一些重要点会略有提及
categories: [cg]
---


* Do not remove this line (it will not be displayed)
{:toc}
# opengl渲染管线

* 顶点处理阶段（Vertex Processing）
  * 顶点着色器阶段（Vertex Shader）
  * 细分着色器阶段（Tessellation）（可选）
  * 几何着色器阶段（Geometry Shader）（可选）
* 顶点后处理（Vertex Post-Processing）
* 图元装配（Primitive Assembly）
* 光栅化（Rasterization）
* 片段着色器（Fragment Shader）
* 逐样本处理（Per-Sample Processing）



# Vertex Shader

顶点着色器主要负责解析保存到显存中的顶点数据，并进行计算，将结果传递给下一个渲染流程。

这个是最最基础的着色器，这里不多介绍。



# Tessellation

细分阶段是一个可选的顶点处理阶段，主要用于将传入细分着色器的每一个patch（例如由4个顶点组成的patch）转变为一个个更小的图元（如三角形）。

细分阶段主要可以分为三个步骤，即细分控制着色器阶段（[Tessellation Control Shader](https://www.khronos.org/opengl/wiki/Tessellation_Control_Shader) ），图元生成阶段（[tessellation primitive generator](https://www.khronos.org/opengl/wiki/Tessellation#Tessellation_primitive_generator) ），细分计算着色器阶段（[Tessellation Evaluation Shader](https://www.khronos.org/opengl/wiki/Tessellation_Evaluation_Shader) ）。



## 细分控制着色器（TCS）

TCS以patch为单位接收从顶点着色器传入的数据，可以在

~~~
layout(vertices = out_patch_size) out;
~~~

定义输出patch的大小，即可以修改输出到下一阶段的patch的大小。

TCS的作用主要有两个：

1. 输出数据给细分计算着色器
2. 控制如何将patch转变为一个个小图元。

**输出数据**

输出数据有两种方式，一种是以顶点为单位，一种是以patch为单位。

以顶点为单位：

```C++
//在TCS中定义
out vec2 vertexTexCoord[];
//进行赋值
vertexTexCoord[gl_InvocationID] = xxxx
```

以patch为单位：

~~~~C++
patch out vec4 data;
~~~~

这两种方式输出的数据，都可以直接在细分计算着色器中通过以下方式获取：

~~~c++
//方式1
in vec2 vertexTexCoord[]
//方式2
patch in vec4 data
~~~

**控制patch的划分方式**

主要是通过以下变量控制patch的划分方式，在图元生成阶段发生作用

```
patch out float gl_TessLevelOuter[4];
patch out float gl_TessLevelInner[2];
```

这两个变量会影响patch的划分方式，但不是唯一因素，在下面我们再会详细解释。



## 图元生成阶段

图元生成阶段的主要工作就是，根据TCS和TES内的参数，对TCS传入的patch（如4个顶点构成的正方形），进行内部划分，得到多个内部顶点，最终在这个抽象正方形内生成多个三角形等。

TCS内定义的gl_TessLevelOuter，gl_TessLevelInner变量，以及TES定义的Abstract patch type都会影响对patch的划分结果。Abstract patch type的值有：

- isolines: patch是直线，输出是一系列直线。
- triangles: patch是三角形，输出是一系列三角形。
- quads: patch是四边形，输出是一系列三角形

图元生成阶段，会在一个抽象图形上进行划分，如参数设置为triangles时，会在一个三角形（三角形的大小，形状都任意，不需要考虑TCS中传入的关于patch的任何顶点数据，所以称为抽象图形）上根据gl_TessLevelOuter，gl_TessLevelInner变量生成内部顶点，最终生成多个三角形。

具体的生成过程，文档里面写得很详细了，不赘述。这里只说一句，所有生成过程，都是先考虑gl_TessLevelInner，再考虑gl_TessLevelOuter。

[图元生成](https://www.khronos.org/opengl/wiki/Tessellation#Tessellation_evaluation_shader)



## 细分计算着色器

经过图元生成阶段，一个抽象图形内部会新增很多顶点，这些顶点构成了更小的图元，如一个个三角形。

细分计算着色器以patch为单位，接收从TCS传来的数据。但一个细分计算着色器只针对一个顶点进行计算，有点类型顶点着色器。

**以由三角形patch生成一系列三角形图元为例：**

每个细分计算着色器针对是一个生成的三角形图元的一个顶点，这个顶点会接收图元阶段传来的vec3 gl_TessCoord变量。该变量可以用于插值计算，通过三角形patch的三个顶点的位置，计算出这个生成三角形图元顶点的位置。方法如下：

```C++
vec3 accum = vec3(0.0f)
accum += gl_TessCoord[0] * position[0]
accum += gl_TessCoord[1] * position[1]
accum += gl_TessCoord[2] * position[2]
```

所以说，图元生成阶段是作用在抽象图形上，因为它不需要知道三角形patch的三个顶点就可以进行。



# Geometry Shader

顶点着色器或者细分计算着色器会对每个顶点进行计算，将数据传输到几何着色器上。

几何着色器的作用主要就是，以一个图元为输入，输出零个或多个图元。

通过以下参数定义输入的图元类型：

~~~C++
layout(input_primitive) in;
~~~

定义输出的图元类型：

```c++
layout(output_primitive, max_vertices = vert_count) out;
//可选值有points，line_strip，triangle_strip
```

**注意几何着色器的传入单位是图元，也就是说，如果传入类型为线段，那么我们就可以在几何着色器中通过gl_in[0], gl_in[1]获得线段两个顶点的数据。**

几何着色器可以实现，诸如输入一个线段，生成一个过线段的四边形，即输出两个三角形。

几何着色器还有多种功能，如实例渲染[instanced rendering](https://www.khronos.org/opengl/wiki/Instancing)，Layered rendering等，这些暂时没用到，这里也不提了。



# Vertex Post-Processing

在这阶段之前，都称为顶点处理阶段，包括顶点着色器，细分着色器，几何着色器等阶段。

在顶点处理阶段中，有以下[interface block](https://www.khronos.org/opengl/wiki/Interface_Block_(GLSL))

~~~c++
gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
};
~~~

这个是每个顶点自带的属性，需要注意的是这里gl_Position要保存的应该是**裁切空间clipping space**下的坐标，就是经过projection, view, model矩阵作用后的坐标。

这里提一句，如果是透视变换，裁切空间下的坐标[x,y,z,w]，$w_{clp} = z_{view}$， 即裁减空间下的w值与该点在**摄像机坐标**下的z值相反，$z_{clip} != z_{view}$

顶点后处理阶段主要有以下工作：

* 裁减顶点

  $$x_{clp},y_{clp},z_{clp}都应该在[-w_{clp}, w_{clip}]范围内 $$，超过范围的顶点会被裁减掉。

* Depth clamping

  这个不太重要，大致就是用户设置特殊的裁减条件。

* 透视除法（Perspective divide）

  就是将x,y,z都除以w，得到标准设备坐标下的顶点表示。

* 视口变换（Viewport transform）

  用户通过：

  ~~~C++
  void glViewport(GLint x, GLint y, GLsizei width, GLsizei height);
  ~~~

  定义了窗口的大小。

  该步骤将该顶点的标准设备坐标，转化为屏幕空间坐标。

# Primitive_Assembly

图元装配阶段就是将传到此阶段的一系列顶点，转换成一系列图元（如三角形）。同时会在此阶段对图元进行**面剔除，排序，阻止图元光栅化等操作**。

如果程序中只使用了顶点着色器和片段着色器，那么图元装配阶段就是将顶点着色器传来的顶点流，转化为图元（如三角形）。

如果程序中使用了几何着色器，因为几何着色器输入的是图元，输出的也是图元，所以图元转换也会发生在这个阶段，这个称为Early primitive assembly。



# Rasterization

在顶点后处理阶段，我们已经将顶点转化到屏幕坐标上了。

在图元装配阶段，我们又将一系列顶点转化为一个个图元（如三角形）。

以三角形为例，光栅化阶段就是根据三角形三个顶点的屏幕坐标，确定显示窗口内的哪一些像素会被三角形覆盖。这些像素的信息，如深度、颜色，会在片段着色器内进行计算。

光栅化算法就是用来计算得到这些特定的像素。



# Fragment Shader

A **Fragment** is a collection of values produced by the [Rasterizer](https://www.khronos.org/opengl/wiki/Rasterizer). Each fragment represents a sample-sized segment of a rasterized [Primitive](https://www.khronos.org/opengl/wiki/Primitive). The size covered by a fragment is related to the pixel area, but rasterization can produce multiple fragments from the same triangle per-pixel, depending on various multisampling parameters and OpenGL state. There will be at least one fragment produced for every pixel area covered by the primitive being rasterized.

因为我还不太清楚multisampling的原理和作用，这里就不误导大家了。但是在简单的应用中，我们可以理解为一个像素就是一个Fragment，片段着色器就是对每个像素进行计算，虽然这应该是不准确的。

以三角形图元为例，片段着色器可以获得顶点处理阶段的最后一个操作的out变量，具体的值就是由三个顶点的对应变量插值得到。

在片段着色器上，我们可以计算像素的颜色等信息，将数据通过如下方式保存到特定Buffer里面：

```c++
layout(location = 1) out int materialID;
layout(location = 4) out vec3 normal;
layout(location = 0) out vec4 diffuseColor;
layout(location = 3) out vec3 position;
layout(location = 2) out vec4 specularColor;
```

如location = 0 对应的buffer保存漫反射颜色等，这样我们可以一次性生成多个buffer。只要将屏幕的颜色buffer绑定到location0，就可以在屏幕上显示出颜色了。



# Per-Sample Processing

这是渲染管线的最后一个阶段，用于处理片段着色器输出的每个片段数据。

这里阶段会执行以下操作：

- [The pixel ownership test](https://www.khronos.org/opengl/wiki/Per-Sample_Processing#Pixel_ownership_test)
- [The scissor test](https://www.khronos.org/opengl/wiki/Per-Sample_Processing#Scissor_test)
- [The stencil test](https://www.khronos.org/opengl/wiki/Per-Sample_Processing#Stencil_test)
- [The depth test](https://www.khronos.org/opengl/wiki/Per-Sample_Processing#Depth_test)
- [Occlusion query updating](https://www.khronos.org/opengl/wiki/Per-Sample_Processing#Occlusion_query_updating)

这里不展开，直接看文档就OK了。



