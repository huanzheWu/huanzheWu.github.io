---
layout: post
title: 【Unity Shaders】十三：模块化：Unity内置CgInclude文件与自定义CgInclude文件
category: 技术
tags: Unity Shaders
description: 
---


##前言
编程的时候我们很强调封装的思想，封装能减少代码的冗杂度，使代码能被重用。在C语言中我们使用struct对数据进行封装，使用函数对功能进行封装，C++中则多了个类，来抽象出一类对象的属性及行为进行封装。在shaderLab中我们同样可以进行代码的封装操作。在自定义我们的封装之前，我们来看看unity为我们做好的封装.


在之前的文章中我们说过，surface shader 是Unity的骄傲，Unity在cg语言的基础上包装了一层皮，使得surface shader的使用简单方便。从这段时间学习的例子中我们也可以看到，在surface shader中我们通过四个函数就可以写出很不错的效果，实际上是unity在背后为我们写了很多东西。当然，unity内置选项为了适应多数shader而具有一般性，没有针对你要渲染的物体进行优化，这会导致一定的效率降低，不过这不在本文的讨论范围内。


对于Unity为什么要选择Cg语言作为shaderLab的基础，我的理解是：cg的跨平台性。我们知道，着色器语言有几种：工作在OpenGL接口上的GLSL，工作在Directx接口上的HLSL。CG语言是微软与英伟达合作开发的shader语言，它同时工作在上述两种接口上，而不需要考虑平台问题。Unity允许在shader中嵌套Cg的代码片段，然后再编译成OpenGL、DirectX，Flash等使得shader可以工作在不同平台上。

##内置的CgInclude文件
好，现在就来看看Unity内置的cg代码片段。这些封装好的文件我们称为**CgInclude文件**。在Unity的安装目录下：

>Unity\Editor\Data\CGIncludes

可以找到Unity内置的CgInclude文件，这些文件后缀**.cginc**。在[Built-in shader include files](http://docs.unity3d.com/Manual/SL-BuiltinIncludes.html "Built-in shader include files")中对这些内置文件有介绍。

##使用内置的CgInclude文件
实际上，在我们写过的surface shader中已经有用到了这些内置cg代码，不信？看下面这句代码：

>＃pragma surface surf Lambert

光照函数**Lambert**即是cgIncludes。在CGInclude文件夹下找到另一个文件——Lighting.cginc。这个文件里面包含了所有我们在类似#pragma surface surf Lambert这样的声明里使用的光照模型。

好，到目前为止，我们知道了在surface shader中Unity为我们提供了很多内置的cg代码，这些代码都可以在Unity\Editor\Data\CGIncludes中找到，我们也可以在shader中使用它们。


然后，我们来学习如何定义我们自己的cgInculdes文件来存储光照模型或者辅助函数，使得我们的shader代码模块化。实际上做法类似于C语言的头文件。下面就来开始咯。我们的目标是：把[漫反射光照改善技巧](http://qg-kkk.github.io/2015/08/09/%E3%80%90%E3%80%90Unity%20Shaders%E3%80%91%E5%85%AD%EF%BC%9A%E8%A1%A8%E9%9D%A2%E7%9D%80%E8%89%B2%E5%99%A8%EF%BC%9ADiffuse%20Shading%E2%80%94%E2%80%94%E6%BC%AB%E5%8F%8D%E5%B0%84%E5%85%89%E7%85%A7%E6%94%B9%E5%96%84%E6%8A%80%E5%B7%A7%20.html "漫反射光照改善技巧")中提及的半Lambert光照函数的代码存储在cgIncludes文件内，这段代码复制粘贴过来：

     inline float4 LightingBasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
            {
                float difLight = max (dot(s.Normal , lightDir), 0 );
                float hLambert = difLight *0.5+0.5;

                float4 col;
                col.rgb = s.Albedo *_LightColor0.rgb *(hLambert * atten * 2);
                col.a = s.Alpha ;
                return col;
            }    
            

下面开始动手。

##创建自定义CgInclude文件

- 创建一个txt文件，将它命名为BasicDiffuse.txt


- 将文件的后缀由.txt强制改为.cginc

- 将BasicDiffuse.cginc导入unity中，我们在unity项目新创建一个文件夹，名为CgIncludes。并将我们的BasicDiffuse.cginc放在这个文件夹下。像这样：


*<center><img src="/public/img/215.png" style="width:50%"></center>*

我们就创建好了我们的文件，接下来可以创建自定义CgInclude代码了，双击CgInclude文件，在编译器中将我们的LightingBasicDiffuse代码写进文件。

##写入自定义CgInclude文件

打开CgInclude文件后，我们开始写入代码咯：

- 首先写入预编译指令。预编译指令能保证我们文件的代码在同一个shader文件中只被包含一次，具体的原因可以去看C语言或C++的预编译指令的作用。

        #ifndef MY_CG_INCLUDE
        #define MY_CG_INCLUDE
        //自定义代码
        #endif  
        
- 接下来我们来写自定义代码




         //内置函数
         inline float4 BasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
            {
                float difLight = max (dot(s.Normal , lightDir), 0 );
                float hLambert = difLight *0.5+0.5;

                float4 col;
                col.rgb = s.Albedo *_LightColor0.rgb *(hLambert * atten * 2);
                col.a = s.Alpha ;
                return col;
            }    
        
- 完整的BasicDiffuse.Cginc文件代码如下：

        #ifndef MY_CG_INCLUDE
        #define MY_CG_INCLUDE

         //内置函数
         inline float4 BasicDiffuse (SurfaceOutput s,fixed3 lightDir , fixed atten)
            {
                float difLight = max (dot(s.Normal , lightDir), 0 );
                float hLambert = difLight *0.5+0.5;

                float4 col;
                col.rgb = s.Albedo *_LightColor0.rgb *(hLambert * atten * 2);
                col.a = s.Alpha ;
                return col;
            } 
           #endif  
 
##使用自定义CgInclude文件
“头文件”写好了，我们应该在shader文件中使用它了。

- 我们需要在块中包含我们自己的CgInclude文件，就像C++中需要在开头添加头文件引用一样。同时，之前我们的Shader使用内置的Lambert光照模型，但现在我们想要使用自定义的BasicDiffuse光照模型。因为我们已经包含了该CgInclude文件，我们可以直接在#pragma指令中指明这一模型：


		CGPROGRAM
		#include "../CgInclude/BasicDiffuse.cginc"
		#pragma surface surf BasicDiffuse
		
注意到include后带了一个路径，它是.cginc文件与Shader文件的相对路径，也即如果它与Shader文件放在同一个文件夹下的话，那么直接这样写名称即可（像C语言头文件）
 >＃include  "BasicDiffuse.cginc"
 
然而上面我们是把它放在了CgInclude文件夹内，所以写上相对路径。

然后[漫反射光照改善技巧](http://qg-kkk.github.io/2015/08/09/%E3%80%90%E3%80%90Unity%20Shaders%E3%80%91%E5%85%AD%EF%BC%9A%E8%A1%A8%E9%9D%A2%E7%9D%80%E8%89%B2%E5%99%A8%EF%BC%9ADiffuse%20Shading%E2%80%94%E2%80%94%E6%BC%AB%E5%8F%8D%E5%B0%84%E5%85%89%E7%85%A7%E6%94%B9%E5%96%84%E6%8A%80%E5%B7%A7%20.html "漫反射光照改善技巧")中的代码我们就改写为下面这样啦：

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
				#include "../CgInclude/BasicDiffuse.cginc"
                #pragma surface surf BasicDiffuse


                sampler2D _MainTex;
                sampler2D _RampTex;
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


把shader赋给材质后，可以发现它的效果与以前是一样的。这样，我们通过自己写的CgInclude文件就实现了函数的模块化。

##项目工程下载
[CgInlucde File](http://pan.baidu.com/s/1dDfhlbN "CgInlucde File")