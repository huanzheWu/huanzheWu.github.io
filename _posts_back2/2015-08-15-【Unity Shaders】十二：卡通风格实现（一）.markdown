---
layout: post
title: 【Unity Shaders】十二：卡通风格实现（一）
category: 技术
tags: Unity Shaders
description: 
---


> 17:47  2015/08/15 于工学一号馆312

在这篇文章里，我们使用表面着色器来做卡通效果。卡通效果有许多种表现方法，这可以写成一个系列。不过目前我只学习了一种(￣﹏￣）就是今天要讲的就是这一种。要表现这种卡通效果要抓住三个point：

- 简化颜色
- 使用渐变图(ramp Texture)来控制diffuse shading
- 模型边缘描黑边

我们先来看一下效果如何：

*<center><img src="/public/img/207.png" style="width:100%"></center>*

其中左边的机器人为卡通风格，而右边机器人为原来的模型。

下面我们分点来进行卡通风格制作的介绍。

##简化颜色

简化颜色的意思即简化了模型上使用的颜色。我们先在**Properties**添加新属性：

        _SimFac("Simplify Factor",Range(0.1,20))=0.5
       
相应的，为了CGPROGRAM ...ENDCG模块中可以使用上面这属性，我们要添加它对应的引用：

        float _SimFac;
       
然后在我们的表面函数**surf**中添加如下语句来对像素的颜色做简化：

        o.Albedo = floor (o.Albedo*_SimFac)/_SimFac;
        
###解释

我们定义了 **_SimFac**来控制颜色简化的程度。那么如何来控制呢？关键就在第三行代码上。**floor**函数对操作数进行向下取整，我们将像素的颜色乘以简化因子，取整之后再除以简化因子来达到收缩颜色的效果。

举个例子：我们假设颜色为(0.75,0.75,0.75),简化因子我们取2，那么执行了代码之后颜色缩为：（0.5，0.5，0.5）

(0.5,0.5,0.5)= floor((0.75,0.75,0.75)*2)/2

对于0.51~0.99范围内的颜色，在简化因子为2的情况下，都会被缩为0.5，从而达到了简化颜色的目的。

*<center><img src="/public/img/208.png" style="width:50%"></center>*

我们也不难推导出，简化有因子越大，颜色简化越弱，例如在简化因子为8时，机器人颜色明显丰富多了：

*<center><img src="/public/img/209.png" style="width:50%"></center>*

###使用渐变图(ramp Texture)来控制diffuse shading

在[Diffuse Shading——漫反射光照改善技巧](http://qg-kkk.github.io/2015/08/09/%E3%80%90%E3%80%90Unity%20Shaders%E3%80%91%E5%85%AD%EF%BC%9A%E8%A1%A8%E9%9D%A2%E7%9D%80%E8%89%B2%E5%99%A8%EF%BC%9ADiffuse%20Shading%E2%80%94%E2%80%94%E6%BC%AB%E5%8F%8D%E5%B0%84%E5%85%89%E7%85%A7%E6%94%B9%E5%96%84%E6%8A%80%E5%B7%A7%20.html "Diffuse Shading——漫反射光照改善技巧")这篇文章中有介绍到，我们使用渐变图来控制漫反射光照的颜色，允许你着重强调surface的颜色，而减弱反射光线的影响。这种技术在《军团要塞2》中流行起来，用于渲染非写实画面如卡通风格游戏。在这里我们就要把这种技术应用上啦，不了解的同学请看完上面这篇文章。

好，首先我们需要一张ramp Texture，渐变图：

*<center><img src="/public/img/210.png" style="width:50%"></center>*

这张渐变图有个特定，按就是边界明显，不像我们以前用过的渐变图那样缓慢变化。这是因为卡通风格里经常有分界明显的明暗变化。

为了使用这张渐变图，我们在**Properties**添加新属性：

        _RampTex("Ramp Texture",2D)=""{}
        
同样的，在CGPROGRAM ...ENDCG模块中引用它：
        sampler2D _RampTex;
        
然后，在我们自定义的光照函数Cartoon中添加如下代码（啥？不知道如何自定义光照函数？请看[自定义光照函数BasicDiffuse](http://qg-kkk.github.io/2015/08/08/%E3%80%90%E3%80%90Unity%20Shaders%E3%80%91%E4%BA%94%EF%BC%9A%E8%A1%A8%E9%9D%A2%E7%9D%80%E8%89%B2%E5%99%A8%EF%BC%9A%E8%87%AA%E5%AE%9A%E4%B9%89%E5%85%89%E7%85%A7%E5%87%BD%E6%95%B0BasicDiffuse.html "自定义光照函数BasicDiffuse")）：

        float NdotL = max(0,dot(s.Normal,lightDir));
		float hNdotL = NdotL *0.5+0.5;

		float NdotV = max(0,dot(s.Normal,viewDir));
		float hNdotV = NdotV*0.5+0.5;

		float3 ram= tex2D(_RampTex,float2(hNdotL,hNdotV)).rgb;
		
###解释

看了[Diffuse Shading——漫反射光照改善技巧](http://qg-kkk.github.io/2015/08/09/%E3%80%90%E3%80%90Unity%20Shaders%E3%80%91%E5%85%AD%EF%BC%9A%E8%A1%A8%E9%9D%A2%E7%9D%80%E8%89%B2%E5%99%A8%EF%BC%9ADiffuse%20Shading%E2%80%94%E2%80%94%E6%BC%AB%E5%8F%8D%E5%B0%84%E5%85%89%E7%85%A7%E6%94%B9%E5%96%84%E6%8A%80%E5%B7%A7%20.html "Diffuse Shading——漫反射光照改善技巧")相信能够理解上面这段代码，也就是更具几个向量来计算渐变图取样坐标。

我们来看看效果：

没有使用渐变图：

*<center><img src="/public/img/211.png" style="width:50%"></center>*

使用了渐变图：

*<center><img src="/public/img/212.png" style="width:50%"></center>*

##给模型边缘描边

首先我们要先解决一个问题，怎么判断一个像素点位于模型的边缘（轮廓）？没错，就是使用向量的点乘来判断，对于一个球体来说，球体边缘的法向量与从正面看的观察向量成90度角，这两个向量的点乘结果为0.我们可以使用一个阈值来控制边缘的大小。具体代码请看：

首先我们还是定义一个边缘阈值：

		_OutLine("OutLine",Range(0,1))=0.1
		
同样的，在下面的模块中对它进行引用：

    		float _OutLine;
    		
然后重点代码就来了，在表面函数surf中，我们添加如下代码：

            //对观察向量与表面法向量进行点乘
            float OutLine = max(0,dot(normalize( o.Normal) ,normalize( IN.viewDir)));
			
			//C语言的语法。根据阈值进行描边。
			OutLine = OutLine< _OutLine ? OutLine / 4 : 1;
			
			o.Albedo = MainCol.rgb * _Emissive.rgb * OutLine;


###解释

主要来看第二行代码，当两个法向量的点乘小于阈值时，我们把表面像素点的最终颜色降为原来的1/4，形成黑色的效果，否则则保持原来的颜色（乘以1）。

来看看效果吧，我们来看机器人的脑壳可以看到明显的黑边：


*<center><img src="/public/img/213.png" style="width:100%"></center>*


把这三个点get到之后，我们就可以做出我们的卡通效果啦！~

##完整shader代码

            Shader "MyShader/Cartoon1"
            {
            	Properties
            	{
            
            		_RampTex("Ramp Texture",2D)=""{}
            		_SimFac("Simplify Factor",Range(0.1,20))=0.5
            		_OutLine("OutLine",Range(0,1))=0.1
            		_MainTex("Main Texture",2D)=""{}
            		_Emissive("Emissive",Color)=(1,1,1,1)
            		_BumpTex("Bump Texture",2D)=""{}
            
            	}
            	SubShader
            	{
            		CGPROGRAM
            		#pragma surface surf Cartoon
            
            		sampler2D _RampTex;
            		sampler2D _MainTex;
            		sampler2D _BumpTex;
            
            		float _SimFac;
            		float _OutLine;
            		float4 _Emissive;
            
            		struct Input 
            		{
            			float2 uv_MainTex;
            			float2 uv_BumpTex;
            			float3 viewDir;
            		};
            
            		inline void surf(Input IN,inout SurfaceOutput o)
            		{
            			float4 MainCol = tex2D(_MainTex,IN.uv_MainTex);
            
            			o.Normal = UnpackNormal( tex2D(_BumpTex, IN.uv_BumpTex));  
            
            			float OutLine = max(0,dot(normalize( o.Normal) ,normalize( IN.viewDir)));
            			
            			//对周围黑边进行处理
            			OutLine = OutLine< _OutLine ? OutLine / 4 : 1;
            
            			o.Albedo = MainCol.rgb * _Emissive.rgb * OutLine;
            			
            			//对颜色进行简化
            			o.Albedo = floor (o.Albedo*_SimFac)/_SimFac;
            			
            			o.Alpha = _Emissive.a;
            
            		}
            		inline float4 LightingCartoon(SurfaceOutput s,float3 viewDir,float3 lightDir,float atten)
            		{
            			float NdotL = max(0,dot(s.Normal,lightDir));
            			float hNdotL = NdotL *0.5+0.5;
            
            			float NdotV = max(0,dot(s.Normal,viewDir));
            			float hNdotV = NdotV*0.5+0.5;
            
            			float3 ram= tex2D(_RampTex,float2(hNdotL,hNdotV)).rgb;
            
            			float4 col;
            
            			col.rgb = s.Albedo* ram*_LightColor0.rgb ;
            
            			col.a = s.Alpha;
            
            			return col;
            
            		}
            		ENDCG
            	}
            }

##描边缺点

这种卡通描边方式是有缺陷的，在较平的平面上会出现突变产生整片的黑色区域：


*<center><img src="/public/img/214.png" style="width:50%"></center>*


这是因为我们采用了顶点法向量来判断边界的，那么对于正方体这种法线固定单一的情况，判断出来的边界要么基本不存在要么就大的离谱！对于这样的对象，一个更好的方法是用Pixel&Fragment Shader、经过两个Pass渲染描边：第一个Pass，我们只渲染背面的网格，在它们的周围进行描边；第二个Pass中，再正常渲染正面的网格。其实，这是符合我们对于边界的认知的，我们看见的物体也都是看到了它们的正面而已。


不过，问题总是会有解决办法的！在以后的卡通shader中我们会解决这个问题的o(^▽^)o

##项目工程下载

[cartoon Shader](http://pan.baidu.com/s/1sjoRXxj "cartoon Shader")








