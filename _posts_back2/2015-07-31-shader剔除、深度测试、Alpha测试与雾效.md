---
layout: post
title: 【Unity Shaders】三：固定编程shader：剔除、深度测试、Alpha测试以及基本雾效
category: 技术
tags: Unity Shaders
description: 
---

#剔除、深度测试、Alpha测试以及基本雾效
##剔除的概念
这里所说的剔除指的是背面剔除。当我们规定了三角形面片的绕序顺序后，就能够区分一个面对我们来说是正面还是背面了，例如，我们规定三角形顶点绕序为顺时针为正面（DX默认），那么三角形顶点绕序为逆时针的就是背面。根据我们的日常生活经验，一个不透明的物体，我们总是只能够看见它的正面而看不到背面，在追求效率的游戏编程中，我们应该把看不到的背面剔除被，不用去渲染它，这就是剔除的原理。

从这角度我们只能看见骰子的3个面：
*<center><img src="/public/img/138.png" style="width:50%"></center>*
##剔除语法
下面的语句用于控制几何体的哪一面会被剔除（不被绘制）：
>Cull Back 不绘制背离观察者的几何面
>
>Cull Front 不绘制面向观察者的几何面，用于由内自外的旋转对象
>
>Cull Off 关闭剔除，即显示所有的面，用于特殊效果

##例程：
    Shader "巫焕哲的Shader/裁剪/正面"
    {
      
    	SubShader
		{
    		Pass
    		{
    			 Material   
    			{  
    			  Emission(0.3,0.3,0.3,0.3)  
   				  Diffuse (1,1,1,1)  
    			} 
    			//开启正面裁剪（不绘制正面）
    			Cull Front
    			Lighting On
    		}
    	}
    }

材质球：可以发现材质球比原来默认的材质颜色变暗，这是因为我们绘制的是物体的背面：

*<center><img src="/public/img/139.png" style="width:50%"></center>*

我们建立一个Cube：

*<center><img src="/public/img/140.png" style="width:50%"></center>*
 
把材质赋予Cube后可以看到Cube的正面已经被裁剪掉了:

*<center><img src="/public/img/141.png" style="width:50%"></center>*

##深度测试的概念
在一个游戏场景中通常有许多的物体需要绘制，我们从一个观测点出发，沿着观察方向，总是会有物体发生遮挡：离观测点近的不透明物体遮挡了离观测点源的物体，使得被遮挡的物体不可见或者部分可见。深度测试可以简化复杂场景的绘制，确保只有场景内的对象最靠近观测点的表面参与绘制。
##深度测试语法
>ZWite On/Off （开/关）

此语句用于控制是否把对象的像素写入深度缓冲，默认是开启的。如果需要绘制纯色物体便将此项打开，如果要绘制半透明效果，则关闭深度缓冲。

>ZTest Less |Greater | LEqual | GEqual | Equal | NotEqual | Always


此语句用于控制深度测试如何执行。缺省值的LEqual，其余选项含义对应如下：


    Greater
    只渲染大于AlphaValue值的像素
    GEqual
    只渲染大于等于AlphaValue值的像素
    Less
    只渲染小于AlphaValue值的像素
    LEqual
    只渲染小于等于AlphaValue值的像素
    Equal
    只渲染等于AlphaValue值的像素
    NotEqual
    只渲染不等于AlphaValue值的像素
    Always
    渲染所有像素，等于关闭透明度测试。等于用AlphaTest Off
    Never
    不渲染任何像素

>Offset Factor ,Units

此语句用两个参数（Facto和Units）来定义深度偏移。


- Factor参数表示 Z缩放的最大斜率的值。


- Units参数表示可分辨的最小深度缓冲区的值。

于是，我们就可以强制使位于同一位置上的两个集合体中的一个几何体绘制在另一个的上层。例如偏移量Offset 设为0, -1（即Factor=0, Units=-1）的值使得靠近摄像机的几何体忽略几何体的斜率，而偏移量为-1,-1（即Factor =-1, Units=-1）时，则会让几何体偏移一个微小的角度，让观察使看起来更近些。
##Alpha测试的概念
Alpha Test ,中文就是透明度测试。简而言之就是V&F shader中最后fragment函数输出的该点颜色值
的alpha值与固定值进行比较。 AlphaTest语句通常位于Pass{}中的起始位置。
##Alpha测试语法
>AlphaTest Off

此语句用于渲染所有像素（缺省）
>AplhaTest comparison AlphaValue

使用这个语句来控制渲染中某一个确定范围内的透明度值的像素，其中 comparison的取值可以为：
    
    Greater
    Only render pixels whose alpha is greater than AlphaValue. 大于
    GEqual
    Only render pixels whose alpha is greater than or equal to AlphaValue. 大于等于
    Less
    Only render pixels whose alpha value is less than AlphaValue. 小于
    LEqual
    Only render pixels whose alpha value is less than or equal to from AlphaValue. 小于等于
    Equal
    Only render pixels whose alpha value equals AlphaValue. 等于
    NotEqual
    Only render pixels whose alpha value differs from AlphaValue. 不等于
    Always
    Render all pixels. This is functionally equivalent to AlphaTest Off. 
    渲染所有像素，等于关闭透明度测试AlphaTest Off
    Never
    Don't render any pixels. 不渲染任何像素



AlphaValue为一个范围在0~1之间的浮点值，也可以是一个指向浮点属性或是一个范围属性，在后一种情况下需要使用标准的方括号写法标注索引名字，如([变量名]).

##雾效

在计算机图形学中，雾化通过混合以生成的像素颜色和基于到镜头的距离来确定最终的颜色来完成。雾化**不会改变**已经混合的像素的**透明度值**，**只改变RGB值**。
##雾效语法
>Fog{FogCommands}

此语句用于设定雾命令的内容，具体的内容见下面语句:
>Mode Off | Global | Linear | Exp | Exp2

此语句用于定义雾模式。缺省是全局的，依据雾在渲染设定中是否打开确定可从无变化到平方值

>Color ColorValue

此语句用于设定雾的颜色值

>Density FloatValue


此语句以指数的方式设定雾的密度。
>语句之五：Range FloatValue , FloatValue

此语句用于为linear类型的雾设定远近距离值。

[这里的例程有待理解](http://blog.csdn.net/poem_qianmo/article/details/41923661)
