---
layout: post
title: 【Unity Shaders】六：表面着色器：Diffuse Shading——漫反射光照改善技巧 
category: 技术
tags: Unity Shaders
description: 
---

        15:25 2015/08/09 于工学一号馆312

在上一篇文章中[【Unity Shaders】五：表面着色器：自定义光照函数BasicDiffuse](http://qg-kkk.github.io/2015/08/08/%E3%80%90%E3%80%90Unity%20Shaders%E3%80%91%E4%BA%94%EF%BC%9A%E8%A1%A8%E9%9D%A2%E7%9D%80%E8%89%B2%E5%99%A8%EF%BC%9A%E8%87%AA%E5%AE%9A%E4%B9%89%E5%85%89%E7%85%A7%E5%87%BD%E6%95%B0BasicDiffuse.html "【Unity Shaders】五：表面着色器：自定义光照函数BasicDiffuse")，我们在表面着色器中定义了自己的光照函数BasicDiffuse，在本篇文章中，我们将对这个基本的diffuse进行改造，改造成一种在游戏《半条命2》中首次使用的光照模型--半Lambert光照，最后我们将学习使用渐变图来渲染漫反射。首先贴出我们上篇文章中写下来的代码：

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
    
##半Lambert光照

###准备工作
如果你看过之前的文章，应该不会对Lmabert光照模型感到陌生，它就是Unity内置的光照模型。Lambert定律认为在平面某点的漫反射光的光强与该反射点的法向量和入射光角度的余弦值成正比。Half Lambert 最初是由Value提出来的，用于《半条命2》的画面渲染，它是为了防止某个物体背光面丢失而显得太过平面化。这个光照模型是没有基于任何物理原理的，它的提出仅仅是一种感性的视觉增强。

我们先在上面这段代码中加上几句代码及删除一些代码，使得物体可以使用贴图进行渲染：

    
        Shader "MyShader/BasicDiffuse"
        {
            Properties
            {
                //定义一些变量，可在监视面板中看到
                //新增
    			_MainTex("Main Texture",2D) = "white "{}
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
                
                //新增：注意在这里进行声明
    			sampler2D _MainTex;
        
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
                    //新改动：同时要修改这里
                    c = tex2D(_MainTex,IN.uv_MainTex);
                    o.Albedo = c.rgb;
                    o.Alpha = c.a;
                }
                ENDCG 
            }
            FallBack "Diffuse"
        }
        

我们把场景中方向光调整至使得模型正面处于背光状态，来看看模型此时的效果：

*<center><img src="/public/img/168.png" style="width:50%"></center>*

###正式动手
说了这么多，我们还是要来演示一下Half Lambert的效果，代码改动上非常简单，我们只要稍微修改上面的LightingBasicDiffuse函数：
    
            inline float4 LightingBasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
                {
                    float difLight = max (dot(s.Normal , lightDir), 0 );
    				float hLambert = difLight *0.5+0.5;
    
                    float4 col;
                    col.rgb = s.Albedo *_LightColor0.rgb *(hLambert * atten * 2);
                    col.a = s.Alpha ;
                    return col;
                }    




由代码可以看出，我们定义了一个新的变量hLambert来替换difLight用于计算某点的颜色值。difLight的范围是0.0-1.0，而通过hLambert，我们将结果由0.0-1.0映射到了0.5-1.0，从而达到了增加亮度的目的。

保存代码，Unity编译好后再看模型，发现模型比刚才亮度增加了很多：

*<center><img src="/public/img/169.png" style="width:50%"></center>*

下面这张图也同样展示了Lambert光照与Half Lambert光照的区别：

*<center><img src="/public/img/170.png" style="width:50%"></center>*

##使用渐变图（ramp Texture）来控制diffuse shading

使用渐变图来控制漫反射光照的颜色，允许你着重强调surface的颜色，而减弱漫反射光线或其他光线的影响，这种技术在《军团要塞2》中流行起来：
*<center><img src="/public/img/171.png" style="width:50%"></center>*
这种技术也是由Value提出来的，用来渲染他们的游戏角色，常用于非写实画面的，比如在很多卡通风格的游戏中可以看到这种技术。

渐变图可以使用PS来制作。制作过程不再多说。我们先使用下面这个渐变图来：

*<center><img src="/public/img/172.jpg" style="width:50%"></center>*

我们需要新增一张贴图，方式与_MainTex相同。然后我们着重改动光照函数的代码：

           inline float4 LightingBasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
                {
                    float difLight = max (dot(s.Normal , lightDir), 0 );
    				float hLambert = difLight *0.5+0.5;
        
                    //新增加
    				float3 ramp = tex2D (_RampTex,float2(hLambert,hLambert)).rgb;
    
                    float4 col;
                    //改动
                    col.rgb = s.Albedo *_LightColor0.rgb *ramp*(atten*2);
                    col.a = s.Alpha ;
                    return col;
                }            
                
这里重点代码是：

>float3 ramp = tex2D (_RampTex,float2(hLambert,hLambert)).rgb;


这行代码返回一个rgb值。tex2D函数接受两个参数：第一个参数是操作的texture，第二个参数是需要采样的UV坐标。这里，我们使用一个漫反射浮点值（即hLambert）来映射到渐变图上的某一个颜色值。最后得到的结果便是，我们将会根据计算得到的Half Lambert光照值来决定光线照射到一个物体表面的颜色变化。

这里贴上半Lambert+渐变图渲染的最终代码：


            Shader "MyShader/HalfLambert_RampTexture"
            {
                Properties
                {
                    //定义一些变量，可在监视面板中看到
            
        			_MainTex("Main Texture",2D) = "white "{}
        			_RampTex("RampTexture",2D)="white"{}
        			
                }
                SubShader
                {
        		 Tags { "RenderType" = "Opaque" }  
                    CGPROGRAM
                    //这里使用了内置光照模型Lambert
                    #pragma surface surf BasicDiffuse
               //Properties中声明的变量在这里要重新声明，以便下面的代码使用
        
        			sampler2D _MainTex;
        			sampler2D _RampTex;
                    inline float4 LightingBasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
                    {
                        float difLight = max (dot(s.Normal , lightDir), 0 );
        				float hLambert = difLight *0.5+0.5;
        
        				float3 ramp = tex2D (_RampTex,float2(hLambert,hLambert)).rgb;
        
                        float4 col;
                        col.rgb = s.Albedo *_LightColor0.rgb *ramp*(atten*2);
                        col.a = s.Alpha ;
                        return col;
                    }            
            
                 
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
                        c = tex2D(_MainTex,IN.uv_MainTex);
                        o.Albedo = c.rgb;
                        o.Alpha = c.a;
                    }
                    ENDCG 
                }
                FallBack "Diffuse"
            }

在监视面板拉上渐变图：
*<center><img src="/public/img/173.png" style="width:50%"></center>*

这时可以看到

*<center><img src="/public/img/174.png" style="width:50%"></center>*


今天的文章就先写到这里，下篇文章见。
        

