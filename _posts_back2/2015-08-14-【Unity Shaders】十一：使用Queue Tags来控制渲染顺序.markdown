---
layout: post
title: 【Unity Shaders】十一：使用Queue Tags来控制渲染顺序
category: 技术
tags: Unity Shaders
description: 
---


> 10:53  2015/08/14 于工学一号馆312


我们可以使用Tags告诉渲染引擎场景中的对象应该什么时候绘制以及如何来渲染。本篇文章主要来介绍在SubShader中使用的渲染队列标签Queue Tags。
Queue Tags 可以决定一个物体什么时候被绘制，觉得场景中不同标签的物体的绘制顺序，具体使用方法与细节请继续往下看。

##语法
>Tags{"TagName1"="Value1" "TagName2"="Value2"}

##细节

- Tags的数量没有限制，我们可以定义任意多的Tag。

- Tags是标准的键值对，也就是可以根据一个键值（key）来获取实值（value）。

- SubShader中的Tags被用来决定渲染顺序，当然也有其他作用的标签，具体可以看[ShaderLab: SubShader Tags](http://docs.unity3d.com/Manual/SL-SubShaderTags.html "ShaderLab: SubShader Tags")。

- 注意Queue Tags必须写在subshader中，而不是在pass中。

- 除了unity提供的预定义tags外，我们也可以定义自己的队列标签。



##Queue tag--决定渲染顺序

使用Queue tag 能够决定我们的对象以什么顺序被渲染。着色器决定对象属于哪一个渲染队列，通过这种方法，透明的物体能够被保证在所有不透明物体绘制完后再绘制。


有四种预定义好的Queue tag。如下所示：

>**Background**   渲染队列值：**1000**

这个标签为背景标签。这个标签将在所有其他标签之前被渲染，可以被用来标记作为背景的对象。

>**Geometry（默认）**   渲染队列值：**2000**

这个标签被最多的对象所使用，不透明的几何体使用这标签。

>**AlphaTest**        渲染队列值：**2450**

需要Alpha测试的几何体使用这标签。它和Geometry队列不同，对于在所有立体物体绘制后渲染的通道检查的对象，它更有效。
>**Transparent**        渲染队列值：**3000**

这个标签将在Geometry与AlphaTest之后进行渲染。任何通道混合的（也就是说，那些不写入深度缓存的Shaders）对象使用该队列，例如玻璃和粒子效果。

>**Overlay**        渲染队列值：**4000**

最后需要渲染的对象选择这个标签，例如覆盖物效果、镜头光晕等。

##自定义中间队列

在同一个标签内，我们也可以觉得物体的绘制顺序，比如
>Tags{"Queue"="Geometry-1"} 与 Tags{"Queue"="Geometry"}

前者比后者优先渲染。被渲染队列值越小的标签标记的对象越优先渲染。


##试验

我们举个例子来说明Queue Tags的使用。首先我们创建一个场景，在场景中摆上两个球，命名为ball1与ball2，ball1在ball2前面(离摄像机比较近)。它们的位置关系是这样的：

*<center><img src="/public/img/203.png" style="width:70%"></center>*

接下来我们写两个shader，分别应用在两个球体上。ball1我们使用这个shader：

        Shader "MyShader/QueueTags"
        {
        	Properties
        	{
        		_Emissive("Emissive",Color)=(1,1,1,1)
        		_MainTex("Main Texure",2D)=""{}
        	}
        	SubShader
        	{
        		Tags{"Queue"= "Geometry-1"}
        		ZWrite Off
        		CGPROGRAM
        		#pragma surface surf Lambert
        
        		sampler2D _MainTex;
        		float4 _Emissive;
        
        		struct Input
        		{
        			float2 uv_MainTex;
        		};
        
        		inline void surf(Input IN,inout SurfaceOutput o)
        		{
        			float4 col = tex2D(_MainTex,IN.uv_MainTex);
        			o.Albedo = col.rgb+_Emissive;
        			o.Alpha = col.a;
        		}
        
        		ENDCG
        	}
        	FallBack "Diffuse"
        }
        
而ball2使用这个shader：
            
            Shader "MyShader/QueueTags"
            {
            	Properties
            	{
            		_Emissive("Emissive",Color)=(1,1,1,1)
            		_MainTex("Main Texure",2D)=""{}
            	}
            	SubShader
            	{
            		Tags{"Queue"= "Geometry"}
            		ZWrite Off
            		CGPROGRAM
            		#pragma surface surf Lambert
            
            		sampler2D _MainTex;
            		float4 _Emissive;
            
            		struct Input
            		{
            			float2 uv_MainTex;
            		};
            
            		inline void surf(Input IN,inout SurfaceOutput o)
            		{
            			float4 col = tex2D(_MainTex,IN.uv_MainTex);
            			o.Albedo = col.rgb+_Emissive;
            			o.Alpha = col.a;
            		}
            
            		ENDCG
            	}
            	FallBack "Diffuse"
            }
            

注意到它们唯一的区别就在于**Tags{"Queue"= "Geometry"}**与**Tags{"Queue"= "Geometry+1"}**上。

分别将这两个shader赋给俩Materal，再将Material拖拽给两个球体。顺便给俩球体调好对比颜色。我们可以来看看效果：

*<center><img src="/public/img/204.png" style="width:70%"></center>*

可以看到，本来在ball2（黄色）前面的ball1（红色）已经跑到ball2的后面去了。

##Zbuffer off

细心的同学已经发现了，在ball1的shader中有一个语句：

>ZWrite Off

这句话的意思是关闭写入深度缓存，即不把ball1的深入值写入深度缓存中，深入缓存的写入以及深度测试才是决定物体是否遮挡的决定性因素。也即是说，就算是ball1先被画出来，但在进行深度写入以及深入测试时，ball1还是离摄像机比较近，所以黄球被红球遮挡住的部分就不绘制了，地面被红球挡住的部分也不绘制了。下面的效果展示了进行深入缓存写入的效果（即去掉了ZWrite off）：
*<center><img src="/public/img/205.png" style="width:70%"></center>*

项目工程可以[在这里下载](http://pan.baidu.com/s/1c0pYq1i "在这里下载")。
今天的文章就讲到这里啦，下篇文章见~











