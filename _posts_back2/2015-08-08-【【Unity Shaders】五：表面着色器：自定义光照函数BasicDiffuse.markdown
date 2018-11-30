---
layout: post
title: 【Unity Shaders】五：表面着色器：自定义光照函数BasicDiffuse
category: 技术
tags: Unity Shaders
description: 
---
      23：39  2015年8月8日  于工学一号馆312
在前面的一篇文章中[【Unity Shaders】四：表面着色器三要素概述](http://qg-kkk.github.io/2015/08/07/%E3%80%90CG%E8%AF%AD%E8%A8%80%E3%80%91%E8%A1%A8%E9%9D%A2%E7%9D%80%E8%89%B2%E5%99%A8%E6%A6%82%E8%BF%B0.html "【Unity Shaders】四：表面着色器三要素概述")，我们介绍了表面着色器的特性以及它的三要素，也就是

- 编译指令
- 自定义输入结构
- 输出结构

编译指令：
>＃pragma surface surfaceFunction lightModel [opeionalparams]

我们说surfaceFunction一般是命名为surf，也可以换成其他的函数名，只要和编译指令中指定的表面函数名对应上就好。对于lightModel（光照模型）,Unity内置的光照模型是Lambert（漫反射）和BlinnPhong（镜面放射）。但是有时候我们想使用自己的光照函数来实现特殊的效果，那么我们需要提供自己的光照函数。具体应该怎么做呢？

##使用内置光照模型函数
我们先来看看使用默认光照模型（Lambert）的表面着色器。代码如下：

      Shader "MyShader/Biild_in LightingModle:Lambert"
    {
    	Properties
    	{
    	    //定义一些变量，可在监视面板中看到
    	    
    		_EmissiveColor("Emissive Color",Color)= (1,1,1,1)
    		_AmbientColor("Ambient Color",Color) = (1,1,1,1)
    		_Slider("Slider",Range(1,10))=5
    	}
    	SubShader
    	{
    		CGPROGRAM
    		//这里使用了内置光照模型Lambert
    		#pragma surface surf Lambert
            
            //Properties中声明的变量在这里要重新声明，以便下面的代码使用
    		float4 _EmissiveColor;
    		float4 _AmbientColor;
    		float _Slider;
    
    		//输入结构
    		struct Input
    		{
    		    //包含了uv信息 ，注意变量必须以 uv开头
    			float2 uv_MainTex;
    		};
    		
    		//表面函数
    		void surf(Input IN,inout SurfaceOutput o)
    		{
    		    //填充SrufaceOutput结构
    			float4 c;
    			c = pow((_EmissiveColor+_AmbientColor),_Slider);
    			o.Albedo = c.rgb;
    			o.Alpha = c.a;
    		}
    		ENDCG 
    	}
    	FallBack "Diffuse"
    }
    
相应的注释都写在代码注释中了，如果看了上一篇博客的话，这段代码应该不难理解。这段代码使用了Unity内置的光照模型Lambert，定义了自发光与环境光属性，并设置一个滑动条以改变物体颜色。在Unity中查看该段shader效果：

*<center><img src="/public/img/163.png" style="width:50%"></center>*

##使用自定义光照函数：准备工作
下面，我们将使用自己写的光照函数来替换掉Unity内置的光照模型Lambert。假设我们的光照函数为BasicDiffuse,则在编译指令中声明光照函数名称：

> ＃pragma surface surf BaseDiffuse

而在定义光照函数时，我们需要在函数名前面加上Lighting,也即是：
> Lighting<光照函数名称>

所以我们的BaseDiffuse函数在定义时这样写：

>inline float4 LightingBasicDiffuse(...)


有三种可供选择的光照模型函数：
>half4 LightingName (SurfaceOutput s, half3 lightDir, half atten){}

这个函数被用于forward rendering（正向渲染），但是不需要考虑view direction（观察角度）时。
>half4 LightingName (SurfaceOutput s, half3 lightDir, half3 viewDir, half atten){}

这个函数被用于forward rendering（正向渲染），并且需要考虑view direction（观察角度）时。

>half4 LightingName_PrePass (SurfaceOutput s, half4 light){}


这个函数被用于需要使用defferred rendering（延迟渲染）时。
##使用自定义光照函数：正式动手
为了将上面这段代码改为使用我们自己的光照模型函数的表面着色器，我们需要做的是:
1. 将编译指令改为：

>＃pragma surface surf BasicDiffuse
 
2.加入光照函数，写在CGPROGRAM...ENDCG块中：

        inline float4 LightingBasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
        {
            float difLight = max (dot(s.Normal , lightDir), 0 );
            flaot 4 col;
            col.rgb = s.Albedo *_LightColor0.rgb *(difLight * atten * 2);
            col.a = s.Alpha ;
            return col;
        }
        
3.保存，进入unity看编译结果。完整代码如下：

      Shader "MyShader/BasicDiffuse"
    {
    	Properties
    	{
    	    //定义一些变量，可在监视面板中看到
    	    
    		_EmissiveColor("Emissive Color",Color)= (1,1,1,1)
    		_AmbientColor("Ambient Color",Color) = (1,1,1,1)
    		_Slider("Slider",Range(1,10))=5
    	}
    	SubShader
    	{
    		CGPROGRAM
    		//这里使用了内置光照模型Lambert
    		#pragma surface surf BasicDiffuse
            
            inline float4 LightingBasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
            {
                float difLight = max (dot(s.Normal , lightDir), 0 );
                float4 col;
                col.rgb = s.Albedo *_LightColor0.rgb *(difLight * atten * 2);
                col.a = s.Alpha ;
                return col;
            }            

            //Properties中声明的变量在这里要重新声明，以便下面的代码使用
    		float4 _EmissiveColor;
    		float4 _AmbientColor;
    		float _Slider;
    
    		//输入结构
    		struct Input
    		{
    		    //包含了uv信息 ，注意变量必须以 uv开头
    			float2 uv_MainTex;
    		};
    		
    		//表面函数
    		void surf(Input IN,inout SurfaceOutput o)
    		{
    		    //填充SrufaceOutput结构
    			float4 c;
    			c = pow((_EmissiveColor+_AmbientColor),_Slider);
    			o.Albedo = c.rgb;
    			o.Alpha = c.a;
    		}
    		ENDCG 
    	}
    	FallBack "Diffuse"
    }
    

Unity编译成功后，我们可以看看使用默认Lambert光照模型（上图）和自定义光照模型BasicDiffuse（下图）的效果图：

*<center><img src="/public/img/164.png" style="width:50%"></center>*

*<center><img src="/public/img/165.png" style="width:50%"></center>*

##自定义光照模型解析
大家对光照函数这段代码可能还不怎么理解：

       inline float4 LightingBasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
            {
                float difLight = max (dot(s.Normal , lightDir), 0 );
                flaot 4 col;
                col.rgb = s.Albedo *_LightColor0.rgb *(difLight * atten * 2);
                col.a = s.Alpha ;
                return col;
            }
            
- 由函数参数我们可以看出这是三种类型光照函数的一种：

>half4 LightingName (SurfaceOutput s, half3 lightDir, half atten){}

这个函数被用于forward rendering（正向渲染），但是不需要考虑view direction（观察角度），对于漫反射来说，从哪个角度看到的光照效果都是相同的。

- 首先是参数s。s是surf函数的输出。由代码可以看出，surf函数对参数o进行赋值，也即是填充SurfaceOutput结构o。surf函数填充了o的Alpha（反射率，也即是颜色）和Alpha（透明度）。LightingBasicDiffuse函数输出的是表面上某点的颜色值和透明度值。参数lightDir表示光源的方向，而atten表示光源的衰减率。
- 光照函数的第一行 

> float difLight = max (dot(s.Normal , lightDir), 0 );

使用了max与dot函数，其中dot函数是向量的点乘函数，向量a点乘向量b为：
>a * b  = |a|  |b|  cos< a, b >

由于dot函数的两个参数都是单位向量，我们可以认为dot(s.Normal,lightDir)的结果是灯光方向向量与平面某点法向量夹角的余弦值。由于余弦值可能是负的，故使用max函数来保证最后得到的值>=0，避免出现了非预期的效果。
- 接下来是计算颜色值col。col的rgb部分由三个部分计算得到，第一个是surface本身反射率，反射率越大，进入人眼光线就越多，颜色就越鲜亮。第二个是_LightColor0,这个是Unity的内置变量，我们可以使用它来得到场景中的灯光颜色。这里顺便附上[Unity内置变量查询](http://docs.unity3d.com/Manual/SL-UnityShaderVariables.html "Unity内置变量查询")。最后乘以第一步得到的光照值与衰减系数。这里的乘以一个倍数2，只是加强一下最后效果。



好了，今天的文章就到这里。在下一篇文章里，我们将继续学习表面着色器的编程，编写更加高级的自定义光照函数。


        




 


    
    





