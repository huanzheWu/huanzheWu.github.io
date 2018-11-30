---
layout: post
title: 【Unity Shaders】一：固定编程shader：Unity中的Shader及其基本框架
category: 技术
tags: Unity Shaders
description: 
---

#shader和Material的基本关系
Shader（着色器）实际上就是一小段程序，它负责将输入的Mesh（网格）以指定的方式和输入的贴图或者颜色等组合作用，然后输出。绘图单元可以依据这个输出来将图像绘制到屏幕上。输入的贴图或者颜色等，加上对应的Shader，以及对Shader的特定的参数设置，将这些内容（Shader及输入参数）打包存储在一起，得到的就是一个Material（材质）。之后，我们便可以将材质赋予合适的renderer（渲染器）来进行渲染（输出）了。
所以说Shader并没有什么特别神奇的，它只是一段**规定好输入（颜色，贴图等）和输出（渲染器能够读懂的点和颜色的对应关系）的程序**。而Shader开发者要做的就是根据输入，进行计算变换，产生输出而已。
#Unity中Shader的三种基本类型
按照渲染管线的分类，可以把Sharder分成3个类别：
##固定功能着色器（Fixed Function Shader）
固定功能着色器为**固定功能渲染管线**的具体表现。
##表面着色器
存在于Unity3D中由U3D发扬光大的一门技术。Untiy3D为我们把Shader的复杂性包装起来，降低shader的书写门槛。
##顶点着色器和片段着色器
GPU上含有两个组件：可编程顶点处理器和可编程片段处理器，顶点和片段处理器被分离成可编程单元，可编程顶点处理器是一个硬件单元，可以运行顶点程序，而可编程片段处理器则是一个可以运行片段程序的单元。
###顶点着色器
顶点着色程序从GPU前端（寄存器）中提取图元信息(顶点位置、法向量、纹理坐标)，并完成顶点坐标空间变换、法向量空间转换、光照计算等操作，最后将计算数据传送到指定寄存器中。
###片段着色器
片段程序从上述寄存器中获取需要的数据：纹理坐标与光照信息等，并根据这些信息以及从应用程序传递的纹理信息进行每个片段的颜色计算（纹理查询），最后将处理后的数据传送光栅操作模块。

## 三种着色器的共同点：
- 都必须从唯一一个根Shader开始
- Prooerties参数部分，作用以及语法完全相同。
- 具体功能都在SubShader里。
- 都可以打标签
- 都可以Fallback
- 都可以处理基本的功能，例如光照漫反射以及镜面反射。但如uv计算效果等高级功能，固定功能着色器无法完成。

## 三种着色器的不同点
- 表面着色器没有通道pass{}，加了会报错，该着色器已经把具体内容打包在光照模型中了。
- 固定渲染管线每句代码之后都没有**“;”**
- 核心结构不同：


######固定渲染管线的核心是：
>Material{}以及SetTexture[_MainTex]{}

######顶点与片段着色器的核心是：

>CGPROGRAM
>
>＃pragma vertex vert
>
>＃pragma fragment frag 
>
>＃include "UnityCG.cginc"
>
>ENDCG

######表面着色器的核心是：
>1.表面着色器使用Unity3D自带光照模型Lambert，也不做顶点处理，只需要一个表面处理函数surf即可。
>CGPROGRAM
>
>＃Pragma surface surf Lambert
>
>ENDCF
>
>2.这套表示使用的是自己写的光照模型lsyLightModel，并且使用了顶点处理函数vert
>
>CGPROGRAM
>
>＃pragma  surface    surf    lsyLightModel    vertex:vert
>
>ENDCG 
    
  

##在Unity中如何区分以上三种着色器
- 没有嵌套CG语言，即代码中没有CGPROGARAM和ENDCG关键字的，就是固定功能着色器。
- 嵌套CG语言，代码中有surf函数的为表面着色器
- 嵌套了CG语言，代码中有#pragma vertex name和  #pragma fragment frag声明的，就是顶点着色器&片段着色器。

#Unity中Shader的基本框架
Unity中Shader整体的框架写法可以用如下的形式来概括：
>Shader "name" { [Properties] SubShaders[Fallback] }

Unity中所有着色器都由关键字shader开始，随后的字符表示着色器的名字，这个名字会显示在Inspector检视面板中，所有的代码都应该放在{}里面。
####name
该名字不需要和shader文件名同名，它应该是简单的描述性词语，在name后面加上/能够中Inspector面板中创造出二级菜单（多个/创建多级菜单）。

#shader整体框架
如上面的整体框架，我们可以画出下面这图：
*<center><img src="/public/img/130.png" style="width:50%"></center>*
从这幅图可以看到，Unity中的shader可以分为以下三个模块：
##属性Properties
Properties一般定义中着色器的**起始部分**。属性模块可以定义一些属性，用来指定那个这段代码有哪些输入。在使用shader的时候可以**直接中材质面板里编辑这些属性**。Properties块内的语法都是单行的。属性的定义格式如下所示：

*<center><img src="/public/img/131.png" style="width:50%"></center>*

属性可以划分为以下几类：

###数字与滑动条

>name ("display name", Range (min, max)) = number //滑动条调节
>
>name ("display name", Float) = number//数字调节
>
>name ("display name", Int) = number//数字调节
###颜色与向量(注意是四元数)
>name ("display name", Color) = (number,number,number,number)
>
>name ("display name", Vector) = (number,number,number,number)
###贴图
>name ("display name", 2D) = "defaulttexture" {}
>
>name ("display name", Cube) = "defaulttexture" {}//立方体贴图
>
>name ("display name", 3D) = "defaulttexture" {}

###细节

1. Range与float的属性只能是单精度值。
2. 在后面的着色器程序中，属性值通过[name]来访问。而display name将显示在材质检视器中。
3. 可以使用在属性定义加上等号为每个属性提供缺省值。
4. 对于纹理(2D, Rect, Cube) 缺省值既可以是一个空字符串也可以是某个内置的缺省纹理："white", "black", "gray" or"bump"

使用示例
<pre><code>
// properties for a water shader
Properties
{
    _WaveScale ("Wave scale", Range (0.02,0.15)) = 0.07 // sliders
    _ReflDistort ("Reflection distort", Range (0,1.5)) = 0.5
    _RefrDistort ("Refraction distort", Range (0,1.5)) = 0.4
    _RefrColor ("Refraction color", Color) = (.34, .85, .92, 1) // color
    _ReflectionTex ("Environment Reflection", 2D) = "" {} // textures
    _RefractionTex ("Environment Refraction", 2D) = "" {}
    _Fresnel ("Fresnel (A) ", 2D) = "" {}
    _BumpMap ("Bumpmap (RGB) ", 2D) = "" {}
}
</code></pre>
##子着色器SubShader
可以有一个或者有多个子着色器SubShader(至少有一个)，子着色器SubShader中含有一个或者多个通道Pass（也是至少要有一个）。
###在Pass中一般可以写以下的代码
>Color Color 设定对象的纯颜色，可以是括号中的四个值，也可以是被方框包围的颜色属性名称

>Material{Material Block} 材质被用于定义对象的材质属性，关于材质块的内容可以看下面的介绍

>Lighting On/Off 定义上述材质块的定义是否有效，On时材质块效果有效，Off时颜色通过Color命令直接给出

>SeparateSpecular On/Off 开启独立镜面反射，这个命令会添加高光光照到着色器通道的末尾，因此贴图对高光没有影响。只在光照开启时有效。
>
>ColorMaterial AmbientAndDiffuse | Emission
使用每顶点的颜色替代材质中的颜色集。AmbientAndDiffuse 替代材质的阴影光和漫反射值;Emission 替代 材质中的光发射值。

###Pash中材质块Material{}代码写法

上面已经说了，在Pass中可以书写材质块代码用于定义对象的材质属性，如下的代码可以写在材质块中：
>Diffuse Color(R,G,B,A);对象基本颜色，漫反射

>Ambient Color(R,G,B,A);环境光，当对象被RenderSettings.中设定的环境色所照射时对象所表现的颜色。

>Specular Color(R,G,B,A);对象反射高光的颜色

>Emission Color 对象自发光

>Shininess Number 取值在0-1之间表示加亮时的光泽度


对象完整光照的最终颜色是：
FinalColor=Ambient * RenderSettings ambientsetting + (Light Color * Diffuse + Light Color *Specular) + Emission

即：
最终颜色=环境光反射颜色* 渲染设置环境设置 *（灯光颜色*漫反射颜色+灯光颜色*镜面反射颜色）+自发光

示例代码：

<pre><code>
Shader "我的Shader"  
{  
	Properties
	{
		_MainColor ("主颜色",Color)=(0,0,0,0)
		_AmbientColor("环境光颜色",Color)=(0.8,0.2,0.2,0)
		_SpecularColor("放射高光颜色",Color)=(0.8,0.2,0.2,0)
		_ShininessNum("光泽度",Range(0,1))= 0.5 //滑动条
		  
	}
    //---------------------------------【子着色器1】----------------------------------  
    SubShader  
    {     
		
        //----------------通道---------------  
        Pass  
        {  
			Color(1,0,0,0) //当光照开启时无效。

            //----------材质块------------  
            Material  
            {  
                //将漫反射和环境光反射颜色设为相同  
                Diffuse[_MainColor]
                Ambient[_AmbientColor] 
				Specular[_specularColor]
				Shininess[_ShinnessNum]
            }  
				 
            //开启光照 //光照影响了颜色 
            Lighting On
			SeparateSpecular On

        }  
    }  
}  
</code></pre>
效果：

*<center><img src="/public/img/132.png" style="width:50%"></center>*

##回滚FallBack
Shader基本框架的最后是指定一个回滚函数Fallback，用来处理所有的子着色器都不能运行时的情况（当目标设备太老时，所有的设备都有其不支持的特性时使用了Fallback），可以认为是一种defult。
>Fallback" Diffuse "  


关于纹理的载入在下一节讲解。