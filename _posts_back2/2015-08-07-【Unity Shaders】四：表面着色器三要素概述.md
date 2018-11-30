---
layout: post
title: 【Unity Shaders】四：表面着色器三要素概述
category: 技术
tags: Unity Shaders
description: 
---

##表面着色器概述
表面着色器只存在于Unity中，算是Unity微创新自创的一套着色器标准。它使得shader的书写门槛降低，使shader技术更容易使用。表面着色器的一些特性如下：

- SurfaceShader可以看成是一个光照VS/FS的生成器，它减少了开发者重复编写代码的工作。


- SurfacebShader的语句编写在**CGPROGRAM...ENDCG**块内，而且SurfacebShader不允许有pass，它自己会编译成多个Pass。


- SurfacebShader使用一个编译指令来声明它是一个表面着色器。



表面着色器的三要素是：编译指令、输入结构、输出结构

##SrufaceShader的编译指令

>＃pragma surface surfaceFunction lightModel [optionalparams]

这个编译指令的参数可分为以下两类：

- 必需的参数
    -   surfaceFunction：表示Cg函数中有表面着色器(surface shader)代码。这个函数的格式应该是这样：void surf (Input IN,inout SurfaceOutput o)， Input是你自己定义的结构。Input结构中应该包含所有纹理坐标(texture coordinates)和表面函数(surfaceFunction)所需要的额外的必需变量。


    - lightModel ：光照模型，内置的光照模型有Lambert与BlinnPhong。我们也可以定义自己的光照模型在这里作为指令的参数（在后面进行解释）。关于Lambert与BlinnPhong光照模型可以参考这篇文章：[常见光照模型解析](http://qg-kkk.github.io/2015/08/06/%E3%80%90CG%E8%AF%AD%E8%A8%80%E3%80%91%E5%B8%B8%E8%A7%81%E5%85%89%E7%85%A7%E6%A8%A1%E5%9E%8B%E8%A7%A3%E6%9E%90.html "常见光照模型解析")


-   可选参数[optionalparams]：这些参数的内容可以参考官方文档 ：    [官方文档](http://docs.unity3d.com/Manual/SL-SurfaceShaders.html   "官方文档")

###SurfaceShader的输入结构
当SurfaceShader编译指令指定了表面函数surf与一个Lambert漫反射光照模型，这时编译指令是这样的：

>  #pragma surface surf Lambert    

这个surf就是表面函数了，表面函数的声明如下所示：

> void surf (Input IN, inout SurfaceOutput o) 

可以看到，surf函数含有两个参数，第一个是Input类型的IN，Input是什么类型？实际上，Input是你自己写定义的输入结构，这个结构通常拥有着色器需要的所有纹理坐标信息，这个纹理坐标必须被命名为**“uv”后接纹理名**，或者是uv2开始，即使用第二纹理坐标集，除了纹理的UV信息，你也可以在结构中输入其他着色函数需要的数据，这些数据包括：

- float3 viewDir - 视图方向( view direction)值。为了计算视差效果(Parallax effects)，边缘光照(rim lighting)等，需要包含视图方向( view direction)值。
- float4 with COLOR semantic -每个顶点(per-vertex)颜色的插值。
- float4 screenPos - 屏幕空间中的位置。 为了反射效果，需要包含屏幕空间中的位置信息。比如在Dark Unity中所使用的 WetStreet着色器。
- float3 worldPos - 世界空间中的位置。
- float3 worldRefl - 世界空间中的反射向量。如果表面着色器(surface shader)不写入法线(o.Normal)参数，将包含这个参数。 请参考这个例子：Reflect-Diffuse 着色器。
- float3 worldNormal - 世界空间中的法线向量(normal vector)。如果表面着色器(surface shader)不写入法线(o.Normal)参数，将包含这个参数。
- float3 worldRefl; INTERNAL_DATA - 世界空间中的反射向量。如果表面着色器(surface shader)不写入法线(o.Normal)参数，将包含这个参数。为了获得基于每个顶点法线贴图( per-pixel normal map)的反射向量(reflection vector)需要使用世界反射向量(WorldReflectionVector (IN, o.Normal))。请参考这个例子： Reflect-Bumped着色器。
- float3 worldNormal; INTERNAL_DATA -世界空间中的法线向量(normal vector)。如果表面着色器(surface shader)不写入法线(o.Normal)参数，将包含这个参数。为了获得基于每个顶点法线贴图( per-pixel normal map)的法线向量(normal vector)需要使用世界法线向量(WorldNormalVector (IN, o.Normal))。



例如下面是一个我们自己定义的结构：

        //输入结构    
       struct Input     
       {    
            float2 uv_MainTex;//纹理贴图    
            float2 uv_BumpMap;//法线贴图    
            float3 viewDir;//观察方向    
        }; 
           
        

###表面着色器的标准输出结构
要书写Surface Shader，了解表面着色器的标准输出结构必不可少，定义一个表面函数（上面的surf），需要用自定义的输入结构来输入相关的UV或数据信息，并在表面函数体内填充输出结构SrufaceOutput.surfOutput描述的是表面的特性：反射率、法向量、自发光、镜面反射度、光泽度、透明度。这部分代码是使用CG或者是HLSL来编写的。

顶点着色器计算了需要填充输入什么，输出什么相关的信息，并产生真实的顶点/像素着色器，以及把渲染路径传递到正向或延时渲染路径。

那么，这个标准的输出结构是这样的：

        struct SurfaceOutput   
        {  
            half3 Albedo;            //反射率，也就是纹理颜色值（r,g,b)   
            half3 Normal;            //法线，法向量(x, y, z)   
            half3 Emission;          //自发光颜色值(r, g,b)   
            half Specular;           //镜面反射度   
            half Gloss;              //光泽度  
            half Alpha;              //透明度  
        };  
        

而这个结构体的成员，会在sruf函数中进行赋值。比如这样：


        //表面着色函数的编写  
        void surf (Input IN, inout SurfaceOutput o)  
        {  
            //反射率，也就是纹理颜色值赋为(0.6, 0.6, 0.6)  
               o.Albedo= 0.6;  
            //  材质表面光泽度0.8
               o.Gloss = 0.8; 
        }
        
##表面着色器简单程序示例

下面建立一个简单的表面着色器，包含了上面所说的表面着色器三要素，可以看着代码结合上面的解说进行理解。
        
         Shader "Example/Diffuse Simple" {
            P
            SubShader {
            
              Tags { "RenderType" = "Opaque" }
              
              //表面着色器代码写在CGPROGRAM...ENDCG块中
              CGPROGRAM
              
              //要素一：编译指令
              #pragma surface surf Lambert
              
              //要素二：自定义输入结构
              struct Input {
                  float4 color : COLOR;
              };
              
              //要素三：标准输入结构SurfaceOutput
              
              void surf (Input IN, inout SurfaceOutput o) {
                  o.Albedo = 1; //把颜色调为（1，1，1）即白色
              }
              
              ENDCG
            }
            
            Fallback "Diffuse"
          }
          
运行结果是：


*<center><img src="/public/img/162.png" style="width:50%"></center>*
         
   

