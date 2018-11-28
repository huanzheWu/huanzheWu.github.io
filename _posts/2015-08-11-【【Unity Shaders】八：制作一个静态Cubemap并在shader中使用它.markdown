---
layout: post
title: 【Unity Shaders】八：制作一个静态Cubemap，并在shader中使用它
category: 技术
tags: Unity Shaders
description: 
---

>16:05  2015/08/11 于工学一号馆312

##环境映射概述

环境映射技术模拟一个物体反射它周围的环境，它假设一个物体的环境是离物体无限远的，因此环境能够被编码到一个称为环境贴图的全方位图像里，立方贴图正是一种全方位图像。所有近来的图形处理都支持立方贴图纹理，立方贴图不是由一副纹理图像构成，而是由6副。熟悉天空盒制作的同学应该对如何由6副图片形成无缝连接的环境有很好的理解。下面这幅图展示了由6副纹理贴图构成的环境的立方贴图：


*<center><img src="/public/img/181.png" style="width:50%"></center>*

##Unity表明着色器对立方贴图的存取

我们知道一个2D的纹理可以通过一个2D纹理坐标集来在纹理中查询颜色值，在之前的文章中我们也对2D纹理的进行纹理存取：

>float4 col = tex2D(_MainTex,In.uv_MainTex);

而对于立方贴图，我们采用的是一个表示3D方向向量的三元纹理坐标集来存取纹理。这个向量可以看成是从立方体中心射出的光线，当光线向外的时候它会与立方体贴图的6个表面之一相交。立方体贴图纹理存取的结果是在与这6个面相交的点的过滤颜色。在Unity表面着色器中，我们使用texCUBE来完成立方体贴图的纹理存取：

>float colCube = texCube(_CubeMap,In.worldRefl)

其中_CubeMap是立方体纹理贴图，在下面我们会介绍在Unity中如何产生静态立方纹理贴图。反而是第二个参数，worldRef1是什么？这是Input提供给我们的世界空间中的反射向量，环境贴图通常是基于世界空间来确定方向的，因此我们需要在世界空间中计算反射向量。在CG中，我们必须把顶点的数据以及法向量变换到世界空间中，然后再进行反射光线的计算。

> float3 positionW = mul(modelToWorld,position).xyz;
>
>float3 N = mul((float3x3)modelToWorld,normal);

我们看一个高度反射物体的时候看到的不是物体本身，而是物体反射它周围的环境。反射视线是基于初始的视线到达表面上某点以及改点的法向量的。当你使用一个立方贴图来编码环境从各个方向上看上去的样子的时候，渲染反射表面上一点大概只需要为表面上的那个点计算反射的视线方向，然后我们就可以基于**反射的视线方向**来存取立方贴图，从而为表面上的这个点决定环境的颜色。

下面这幅图显示的是一个物体以及一张立方贴图。因为我们是从2D来看的，所以物体只是是梯形，而立方贴图用正方形来表示。入射光线从眼睛出发指向物体表面某点，根据该点的表面法向量计算反射光线，由反射光线的方向来对立方贴图进行纹理存取。

*<center><img src="/public/img/182.png" style="width:50%"></center>*

下面这张图显示这种情况的几何排列：

*<center><img src="/public/img/183.png" style="width:50%"></center>*

在CG中提供了函数来进行反射光线的计算：

>reflect(I ,N )
>
>这个函数为入射光线I和表面法向量N返回反射向量。

然而在Unity的表面着色器中，我们使用简单这一句就完成了纹理存取的一系列的事情。

>float colCube = texCube(_CubeMap,In.worldRefl)

##立方体生成脚本的编写

我们必须学会自己制作静态的立方体贴图，因为立方体贴图（Cubemaps）来源我们的游戏场景，网上已有的Cubemaps并不适用在我们的场景中。下面我将提供一个C#脚本，使用这个脚本能够方便快捷地创建一个Cubemap。最终使用了我们制作的Cubemap完成的Shader是下面这种效果：

*<center><img src="/public/img/184.png" style="width:50%"></center>*

我们开始制作：

1. 新创建一个C#脚本，命名为GenerateStaticCubemap。然后在Project面板中创建一个名为Editor文件夹，把GenerateStaticCubemap脚本放在该文件夹下。
2. 在编辑器中打开该脚本，添加如下using指令：

        using UnityEngine;  
        using UnityEditor;  
        using System.Collections  
        
3. 我们的脚本会在Unity编辑器中创建编辑窗口，所以我们的GenerateStaticCubemap类要继承于**ScriptableWizard**类。这使我们可以用一些底层函数来完成目标。
        
        public class GenerateStaticCubemap : ScriptableWizard { 
        
4. 我们需要一个Cumemap类型变量以及一个位置变量来产生最后的立方体贴图：

        	public Transform renderPosition;
        	public Cubemap cubemap;
        	
5. 我们写一个函数：OnWizardUpdate()，这个函数在它在向导（wizard）第一次弹出或者当GUI被用户改变时（如拖进去某些对象，输入某些字符等）时被调用，我们可以在这里检查用户已经向向导中填入我们需要的所有的资源。在这里，如果Cubemap或者它的位置（一个transform）没有被填充，那么就设置内置变量isValid为false，直到拿到所有资源。
        
                
        	void OnWizardUpdate() {
        		helpString = "Select transform to render" +
        			" from and cubemap to render into";
        		if (renderPosition != null && cubemap != null) {
        			isValid = true;
        		}
        		else {
        			isValid = false;
        		}
        	}
        	
6. 当isValid变量为true时，向导将调用OnWizardCreate()函数。我们在这函数里来得到最终的Cubemap：
            
               void OnWizardCreate() {  
               GameObject go = new GameObject("CubemapCamera");
               go.AddComponent<Camera>();
            
                go.transform.position = renderPosition.position;  
                go.transform.rotation = Quaternion.identity;
            
                go.GetComponent<Camera>().RenderToCubemap(cubemap);
              
                DestroyImmediate(go);  
            }  
            
7. 我们需要从Unity编辑器打开这个向导，所以这里需要MenuItem关键词：

        	[MenuItem("CookBook/Render Cubemap")]
        	static void RenderCubemap() {
        		ScriptableWizard.DisplayWizard("Render CubeMap", typeof(GenerateStaticCubemap), "Render!");
        	}
        	
好啦，这样cubemap生成器就大功告成，完整代码如下：

            using UnityEngine;
            using UnityEditor;
            using System.Collections;
            
            public class GenerateStaticCubemap : ScriptableWizard {
                public Transform renderPosition;
                public Cubemap cubemap;
            
                void OnWizardUpdate() 
                {  
                     helpString = "Select transform to render" +  
                    " from and cubemap to render into";  
                if (renderPosition != null && cubemap != null) {  
                    isValid = true;  
                }  
                else {  
                    isValid = false;  
                    }  
                }  
            void OnWizardCreate() {  
               GameObject go = new GameObject("CubemapCamera");
               go.AddComponent<Camera>();
            
                go.transform.position = renderPosition.position;  
                go.transform.rotation = Quaternion.identity;
            
                go.GetComponent<Camera>().RenderToCubemap(cubemap);
              
                DestroyImmediate(go);  
            }  
            
                [MenuItem("CookBook/Render Cubemap")]  
                static void RenderCubemap() {  
                ScriptableWizard.DisplayWizard("Render CubeMap", typeof(GenerateStaticCubemap), "Render!");  
            }  
                        	
            }
            

##生成立方体纹理

回到我们的Unity编辑器，在编辑器上方可以看到：

*<center><img src="/public/img/185.png" style="width:50%"></center>*

我们先创建一个Cubemap文件，命名为MyCubemape，然后再创建一个球体Sphere：
*<center><img src="/public/img/186.png" style="width:50%"></center>*

打开我们写的Render CubeMap,把MyCubemap以及Sphere赋予它，这里Sphere主要是来用提供位置的（也即是上面C#脚本中的Transform变量，最后是用来设置摄像机位置的）:
*<center><img src="/public/img/187.png" style="width:50%"></center>*

点击Render！，我们可以得到一个立方纹理贴图：

*<center><img src="/public/img/188.png" style="width:50%"></center>*

##在表面着色器中使用立方体贴图

有了上面的介绍，我们的shader代码就好理解多了，如果你从前面的文章一直看下来的，对表面着色器三要素有了解的，下面这段代码基本不用解释了。这里贴上代码：

        Shader "MyShader/Using CubeMap"
        {
        	Properties
        	{
        		_MainTex("Main Texture",2D)="white"{}
        		_MainColor("Diffuse Tint",Color)=(1,1,1,1)
        		_CubeMap("CubeMap",CUBE)=""{}
        		_ReflAmount("Reflection Amount",Range(0.01,1))=0.5
        	}
        	SubShader
        	{
        	CGPROGRAM
        	#pragma surface surf Lambert
        	sampler2D _Maintex;
        	samplerCUBE _CubeMap;
        	float4 _MainTint;
        	float _ReflAmount;
        
        	//输入结构
        	struct Input
        	{
        		float2 uv_MainTex;
        		float3 worldRefl;
        	};
        
        	//表面函数
        	void surf (Input IN ,inout SurfaceOutput o)
        	{
        		half4 c =tex2D(_Maintex,IN.uv_MainTex)*_MainTint;
        		
        		//这句话是重点
        		o.Emission = texCUBE(_CubeMap,IN.worldRefl).rgb*_ReflAmount;
        		
        		o.Albedo = c.rgb;
        		o.Alpha = c.a;
        	}
        
        	ENDCG
        	}
        	 FallBack "Diffuse"  
        }
        
保存，回到Unity编辑器。创建一个材质并绑定上面这段shader代码，这里的CubeMap选择我们上面刚刚制作的那个，并为材质赋予一张基本贴图：
*<center><img src="/public/img/189.png" style="width:50%"></center>*

然后把这材质球赋予我们的Sphere，可以看到我们前面的效果已经出来了：

*<center><img src="/public/img/190.png" style="width:50%"></center>*


##延展
前面的讨论中我们提及，环境映射假设离物体无限远，这是因为我们的立方体纹理存取只取决于世界坐标下的反射向量，反射向量只决定了方向，而没有决定距离，即反射向量方向相同时，位置上的变化不影响表面反射外观，如果环境中所有东东都离表面足够远，那么这种说法就成立了。

另外，也因为如此，环境映射在平面上表现很差，例如镜子，在镜面上反射需要依赖于位置。相反的，环境映射在曲面上表现得很好，例如我们的球。

好了，这篇文章就到这里为止，下一篇文章见。