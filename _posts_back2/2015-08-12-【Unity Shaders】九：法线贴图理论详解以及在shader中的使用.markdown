---
layout: post
title: 【Unity Shaders】九：法线贴图理论详解以及在shader中的使用
category: 技术
tags: Unity Shaders
description: 
---

>00:09     2015/08/12 于工学一号馆312


在这一篇文章里，我们将介绍如何使用法线贴图来在一个平面上做出凹凸的效果。写这系列文章不仅仅只是展示shader代码的编写，我更希望把涉及到的，我学习到的知识都与大家分享。那好，在正式讲解Shader代码之前，我们先来看看凹凸映射效果以及法向量贴图的知识。

##凹凸映射
凹凸映射把由一个纹理提供的物体表面法向量的扰动与每个片段的光照相结合，来模拟光照与凹凸表面的相互作用，使得本来需要几何镶嵌才呈现得出的凹凸效果在一个平面上也能显示出来。

使用凹凸映射的原因：

1. 用足够的几何细节来记录表面的凹凸不平的性质的方法来表示模型对交互渲染来说是非常巨大和麻烦的。
2. 表面的特征也许比一个像素还小，这意味着光栅器不能精确地渲染所有的几何细节。

凹凸映射的好处包括了：

1. 在场景中提供了一个级别更高的视觉复杂度，而没有增加更多的几何形状。
2. 简化了内容创作，因为你可以用纹理来对表面细节进行编码，而不需要美工人员设计高度详细的3D模型。
3. 应用不同的凹凸贴图到同一个模型的不同实例的能力，给了每个实例一种不同的表面外观。例如，一个建筑物模型能够被用一个砖凹凸贴图渲染一次，而第二次使用泥灰凹凸贴图。

##法向量的存储

我们传统的纹理通常包含RGB或RGBA颜色值，对于RGB纹理，每一个像素都由三个分量组成，分别代表了红色、绿色、蓝色，通常这些分量都为一个无符号字节。

法向量贴图是凹凸贴图的一种形式，对于法向量贴图来说，存储在纹理元素中的不是颜色值，而是法向量。每个法向量是一个从表面向外指的方向向量。传统的RGB纹理格式用来存储法向量贴图。与颜色值不同的是，颜色是无符号的，而方向向量需要有符号值，除了无符号外，纹理中的颜色值通常被限制在[0，1]的范围内，而方向向量的取值范围是[-1,1]，为了能使针对无符号颜色的纹理过滤硬件能正常操作，我们必须要[-1,1]通过缩放与偏移，将其压缩至[0,1]内：

>colorComponent = 0.5 normalComponent + 0.5

而在过滤硬件处理好后，要把法向量拓展回它们本身的范围，可以这样：

> normalComponent = 2 * (colorComponent - 0.5) 

总结上面这几段话：通过使用一个RGB纹理的红色、绿色、蓝色分量来存储一个法向量的x、y和z分量，并对有符号的值进行范围压缩到[0,1]无符号范围，然后法向量可以被存储在一个RGB纹理中。


##从高度图生成法向量贴图

高度图纹理对每个像素的高度进行编码，而不是对向量进行编码，因此，高度图在每个纹理元素存储了一个单独的无符号分量，而不是使用3个分量来存储一个向量。高度图由黑色,白色和之间的254种渐变灰度所生成，较暗的部分高度较低，教亮部分高度较高。下面显示是一张高度图：

*<center><img src="/public/img/191.png" style="width:50%"></center>*



我们的法线贴图可以从高度贴图中生成，生成规则是：

计算高度图一个纹理元素对应的法向量，需要对给定的纹理元素、它正上方和右方的纹理元素的高度进行采样，采样得到了三个高度值：给定纹理元素的高度Hg，给定纹理元素正上方纹理元素的高度Ha，给定纹理元素右方纹理元素的高度值Hr。说起来还挺绕口的。得到这三个值之后，就可以来构成对应法向量了。由Hg，Ha，Hr可以得到两个差分向量：

>flaot3 d1 = (1, 0, Ha - Hg )
>
>flaot3 d2 = (0, 1, Hr - Hg )

我们的法向量可以由向量d1 与 d2 做外乘，然后规范化（单位化）得到。 即：

> float3 Normal = normalize ( nod1 X d2 )

把一个高度图转换为一个法向量贴图是一个完全自动的过程，并且它通常与范围压缩在预处理阶段进行。

从高度图到法线贴图的转换，z分量总是正的并且通常或一定为1。z分量通常被存储在蓝色分量重，而范围压缩把z值转化到[0.5,1]范围，因此，存储一个RGB纹理中经过范围压缩的法向量贴图最主要的颜色的蓝色：

*<center><img src="/public/img/192.png" style="width:50%"></center>*

##使用Unity3D 的 Shader来实现法线贴图与反射

文章剩下的内容来讲诉如何在surface shader中使用法线贴图制造凹凸效果。


1. 我们需要一个Cubemap来产生反射的效果，就使用[上一篇文章【制作一个静态Cubemap，并在shader中使用它】](http://qg-kkk.github.io/2015/08/11/%E3%80%90%E3%80%90Unity%20Shaders%E3%80%91%E5%85%AB%EF%BC%9A%E5%88%B6%E4%BD%9C%E4%B8%80%E4%B8%AA%E9%9D%99%E6%80%81Cubemap%E5%B9%B6%E5%9C%A8shader%E4%B8%AD%E4%BD%BF%E7%94%A8%E5%AE%83.html "上一篇文章【制作一个静态Cubemap，并在shader中使用它】")中的Cubemap就行了。

*<center><img src="/public/img/193.png" style="width:50%"></center>*

2. 我们找一张法线贴图：


*<center><img src="/public/img/194.png" style="width:50%"></center>*

3. 写我们的shader代码：
  
        Shader "MyShader/Normal Texture"
        {
        	Properties
        	{
        		_MainTex("Main Texture",2D)="white"{}
        		_NormalTex("Normal Texture",2D)=""{}
        		_CubeMap ("Cubemap",CUBE)=""{}
        		_Slider("Slider",Range(0.1,1))=0.5
        	}
        	SubShader
        	{
        		CGPROGRAM
        		#pragma surface surf Lambert
        		sampler2D _MainTex;
        		sampler2D _NormalTex;
        		samplerCUBE _CubeMap;
        		float _Slider;
        
        		struct Input
        		{
        			float2 uv_MainTex;
        			float2 uv_NormalTex;
        			float3 worldRefl; 
        		    //关键1
        			INTERNAL_DATA 
        		};
        
        		inline void surf (Input IN,inout SurfaceOutput o)
        		{
        			half4 MainTexCol = tex2D(_MainTex,IN.uv_MainTex);
        
        			//关键2
        			float3 normals = UnpackNormal(tex2D(_NormalTex,IN.uv_NormalTex)).rgb;
        
        			o.Normal = normals;
        
        			//关键3
        			o.Emission = texCUBE(_CubeMap,WorldReflectionVector(IN,o.Normal)).rgb;
        
        			o.Albedo = MainTexCol.rgb*_Slider;
        
        			o.Alpha = MainTexCol.a; 
        		}
        
        		ENDCG
        	}
        	   FallBack "Diffuse"  
        }
        

保存shader代码，回到Unity编辑器，将各种贴图以及制作好的Cubemap赋予材质，可以得到下面的效果：
*<center><img src="/public/img/195.png" style="width:50%"></center>*

对比于没有使用凹凸贴图的材质球：
*<center><img src="/public/img/196.png" style="width:50%"></center>*

最后，我们将两种材质赋予两个sphere，在scene中进行比较：

*<center><img src="/public/img/197.png" style="width:50%"></center>*

##解释



这段代码有三个比较关键的地方：

- 在Input结构中添加下面变量，这是Unity内置变量。

>INTERNAL_DATA 

- UnpackNormal函数，看了上面的法向量的存储应该知道这函数是干什么的，它使法向量从RGB范围恢复到[-1,1]范围内。

- 我们通过声明INTERNAL_DATA来访问修改后的法线信息，然后使用WorldReflectionVector (IN, o.Normal)去查找Cubemap中对应的反射信息。所以在surf函数中查询立方体贴图的时候有别于上一篇文章的代码：

>texCUBE(_CubeMap,WorldReflectionVector(IN,o.Normal))



这篇文章就写到这里，下篇文章见！










