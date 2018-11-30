---
layout: post
title: 【Unity Shaders】十：菲涅尔效果及布料Shader
category: 技术
tags: Unity Shaders
description: 
---


> 23:00  2015/08/13 于工学一号馆312

本文主要介绍布料（Cloth）shader的实现。布料在游戏中非常常见，主角身上的衣服，房间里的窗帘等等都是布料构成。布料shader的重点在于如何让布料的纤维适当分散在整个表面的光照，使它看起来有真实布料的质感，如何让布料上细小的纤维能够产生边缘关照效果。为了实现布料效果，我们需要先来介绍一点物理学的知识。
##菲涅尔效果
大家都知道反射与折射，把反射与折射结合在一起就可以创造涅菲尔效果。一般而言，当光到达两种材质的接触面的时候，一部分光发生反射被表面反射出去，另一部分光发生折射穿过接触面，这个现象就称为涅菲尔现象。水面上就可以发生涅菲尔效果：当你垂直向水面时才可以看到水里的鱼，当远眺水面（视线与水面夹角很小）往往看到的是水面的反光。

涅菲尔效果为图像增加的真实性，它允许你创建物体时展示反射与折射的混合，使得物体看起来更像真实世界的物体。

涅菲尔公式描述了多少光背放射和多少光背折射。然而，量化了涅菲尔效果的涅菲尔公式是非常复杂的。所以我们往往使用经验公式---而非真正的涅菲尔公式，来模拟涅菲尔效果。实际上在游戏引擎中很少使用真正的物理公式精确模拟底层物理，一些技巧往往可以通过很少的计算来实现很不错的效果。

既然如此，我们就给出一个菲涅尔公式的近拟：

>refletionCoefficient
>
> = max ( 0, min (1, bias + scale * (1+ dot(I,N ) )^ power ) )

公式中I表示入射向量，N表示表面法向量。当I与N几乎重合的时候（垂直看水面），反射系数几乎为0，表面大部分光被折射。当I与N分开的时候（夹角逐步变小），反射系数应该逐渐增加并最终增加到1.也即反射系数的范围被限制在[0,1]之间。

这个公式得出的结果**refletionCoefficient**是一个系数，我们使用它来对反射与折射做权重分配：

>Col = refletionCoefficient*反射颜色+ (1-refletionCoefficient)*折射颜色

##细节贴图
这个例子中会用到细节法线贴图与细节贴图，我们将这两种法线融合在一起，可以得到更高层次的表现。在这里我们要学习的技术就是如何把两个法线贴图的效果融合。这种技术用来模拟细节层次上的凹凸感，分散整个表面的高光反射。

本节用到的几张贴图如下：

*<center><img src="/public/img/198.png" style="width:50%"></center>*

注意两张法线贴图导入Unity后，需要将它们的类型从**Texture**转为**Normal map**。这背后发生了什么，我以后再说。

##开始写shader

- 属性块
属性块包含的属性如下，

        	Properties
        	{
        	    //布料主要颜色
        		_MainTint ("Global Tint", Color) = (1,1,1,1)
        		
        		//布料法线贴图
        		_BumpMap ("Normal Map", 2D) = "bump" {}
        		
        		//细节法线贴图
        		_DetailBump ("Detail Normal Map", 2D) = "bump" {}
        		
        		//细节贴图
        		_DetailTex ("Fabric Weave", 2D) = "white" {}
        
        		//发生涅菲尔效果时的颜色（远眺水面看到的水面颜色）
        		_FresnelColor ("Fresnel Color", Color) = (1,1,1,1)
        		
        		//发生菲涅尔效果的强度
        		_FresnelPower ("Fresnel Power", Range(0, 12)) = 3
        		
        		_RimPower ("Rim FallOff", Range(0, 12)) = 3
        
        		//镜面高光强度
        		_SpecIntesity ("Specular Intensiity", Range(0, 1)) = 0.2
        
        		_SpecWidth ("Specular Width", Range(0, 1)) = 0.2	
        	}
        	

- 在CGPROGRAM...ENDCG中说明上面属性：

        SubShader 
        	{
        		Tags { "RenderType"="Opaque" }
        		LOD 200
        		
        		CGPROGRAM
        		//这里要指定我们自己的光照函数Velvet
        		#pragma surface surf Velvet
        		#pragma target 3.0
        
                //声明属性，以便下面的代码使用这些属性
        		sampler2D _BumpMap;
        		sampler2D _DetailBump;
        		sampler2D _DetailTex;
        		float4 _MainTint;
        		float4 _FresnelColor;
        		float _FresnelPower;
        		float _RimPower;
        		float _SpecIntesity;
        		float _SpecWidth;
        		...
        		ENDCG
        	}
        	
- 定义输入结构Input，我们有三张贴图，所以以UV开头定义三个坐标成员：

        	struct Input 
        		{
        			float2 uv_BumpMap;
        			float2 uv_DetailBump;
        			float2 uv_DetailTex;
        		};
        		


        		
- 接下来是重点了，我们来写自己的光照函数。记得光照函数要在光照函数名前面加上固定字段**Lighting**：

    		inline fixed4 LightingVelvet (SurfaceOutput s, fixed3 lightDir, half3 viewDir, fixed atten)
    		{
    			//对各种向量都进行规范化
    			viewDir = normalize(viewDir);
    			lightDir = normalize(lightDir);
    			half3 halfVec = normalize (lightDir + viewDir);
    			fixed NdotL = max (0, dot (s.Normal, lightDir));
    			
    			//创建镜面反射系数
    			float NdotH = max (0, dot (s.Normal, halfVec));
    			float spec = pow (NdotH, s.Specular*128.0) * s.Gloss;
    			
    			//创建菲涅尔效果
    			//不要被这两句话吓到
    			//它们也只是量化菲涅尔公式的一种近拟，类似我们上面介绍的公式。
    			float HdotV = pow(1-max(0, dot(halfVec, viewDir)), _FresnelPower);
    			float NdotE = pow(1-max(0, dot(s.Normal, viewDir)), _RimPower);
    
    			//也可以使用我们上面的公式来创建，效果是一样的
    			//float HdotV = max(0, min(1,pow((1+dot(-viewDir,s.Normal)),_FresnelPower))); 
    
    			float finalSpecMask =  HdotV+NdotE;
    			
    			//输出最终的颜色
    			fixed4 c;
    			c.rgb = (s.Albedo * NdotL * _LightColor0.rgb)
    					 + (spec * (finalSpecMask * _FresnelColor)) * (atten * 2);
    			c.a = 1.0;
    			return c;
    		}
    		
在这里我们使用的并不是上面介绍的菲涅尔公式，而是新的一条公式，然而它们的原理都是相同的：都准守菲涅尔效果。我们可以画图模拟这条公式，结果就会很明了。

- 表面函数。需要的说明也在注释中说清楚了。



	    void surf (Input IN, inout SurfaceOutput o) 
		{
			//对三张贴图的取样
			half4 c = tex2D (_DetailTex, IN.uv_DetailTex);

			//UnpackNormal函数是把压缩的法向量还原到[-1,1]范围，具体细节以后会说到。
			fixed3 normals = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap)).rgb;
			fixed3 detailNormals = UnpackNormal(tex2D(_DetailBump, IN.uv_DetailBump)).rgb;

			//这里对两种法向量进行混合，不过操作也很简单，即对应元素相加而已。
			fixed3 finalNormals = float3(normals.x + detailNormals.x, 
										normals.y + detailNormals.y, 
										normals.z + detailNormals.z);
			
			o.Normal = normalize(finalNormals);
			//对高光强度赋值，范围0-1
			o.Specular = _SpecWidth;
			o.Gloss = _SpecIntesity;
			//颜色
			o.Albedo = c.rgb * _MainTint;
			//alpha
			o.Alpha = c.a;
		}
	

##完整代码

            Shader "Custom/ClothShader" {
            	Properties
            	{
            		_MainTint ("Global Tint", Color) = (1,1,1,1)
            		_BumpMap ("Normal Map", 2D) = "bump" {}
            		_DetailBump ("Detail Normal Map", 2D) = "bump" {}
            		_DetailTex ("Fabric Weave", 2D) = "white" {}
            
            		//发生涅菲尔效果时的颜色（远眺水面看到的水面颜色）
            		_FresnelColor ("Fresnel Color", Color) = (1,1,1,1)
            		//发生菲涅尔效果的强度
            		_FresnelPower ("Fresnel Power", Range(0, 12)) = 3
            		
            		_RimPower ("Rim FallOff", Range(0, 12)) = 3
            
            		//镜面高光强度
            		_SpecIntesity ("Specular Intensiity", Range(0, 1)) = 0.2
            
            		_SpecWidth ("Specular Width", Range(0, 1)) = 0.2	
            	}
            	
            	SubShader 
            	{
            		Tags { "RenderType"="Opaque" }
            		LOD 200
            		
            		CGPROGRAM
            		#pragma surface surf Velvet
            		#pragma target 3.0
            
            		sampler2D _BumpMap;
            		sampler2D _DetailBump;
            		sampler2D _DetailTex;
            		float4 _MainTint;
            		float4 _FresnelColor;
            		float _FresnelPower;
            		float _RimPower;
            		float _SpecIntesity;
            		float _SpecWidth;
            
            		struct Input 
            		{
            			float2 uv_BumpMap;
            			float2 uv_DetailBump;
            			float2 uv_DetailTex;
            		};
            		
            		inline fixed4 LightingVelvet (SurfaceOutput s, fixed3 lightDir, half3 viewDir, fixed atten)
            		{
            			//对各种向量都进行规范化
            			viewDir = normalize(viewDir);
            			lightDir = normalize(lightDir);
            			half3 halfVec = normalize (lightDir + viewDir);
            			fixed NdotL = max (0, dot (s.Normal, lightDir));
            			
            			//创建镜面反射系数
            			float NdotH = max (0, dot (s.Normal, halfVec));
            			float spec = pow (NdotH, s.Specular*128.0) * s.Gloss;
            			
            			//创建菲涅尔效果
            			//不要被这两句话吓到
            			//它们也只是量化菲涅尔公式的一种近拟，类似我们上面介绍的公式。
            			float HdotV = pow(1-max(0, dot(halfVec, viewDir)), _FresnelPower);
            			float NdotE = pow(1-max(0, dot(s.Normal, viewDir)), _RimPower);
            
            			//也可以使用我们上面的公式来创建，效果是一样的
            			//float HdotV = max(0, min(1,pow((1+dot(-viewDir,s.Normal)),_FresnelPower))); 
            
            			float finalSpecMask =  HdotV+NdotE;
            			
            			//输出最终的颜色
            			fixed4 c;
            			c.rgb = (s.Albedo * NdotL * _LightColor0.rgb)
            					 + (spec * (finalSpecMask * _FresnelColor)) * (atten * 2);
            			c.a = 1.0;
            			return c;
            		}
            
            		void surf (Input IN, inout SurfaceOutput o) 
            		{
            			//对三张贴图的取样
            			half4 c = tex2D (_DetailTex, IN.uv_DetailTex);
            
            			//UnpackNormal函数是把压缩的法向量还原到[-1,1]范围，具体细节以后会说到。
            			fixed3 normals = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap)).rgb;
            			fixed3 detailNormals = UnpackNormal(tex2D(_DetailBump, IN.uv_DetailBump)).rgb;
            
            			//这里对两种法向量进行混合，不过操作也很简单，即对应元素相加而已。
            			fixed3 finalNormals = float3(normals.x + detailNormals.x, 
            										normals.y + detailNormals.y, 
            										normals.z + detailNormals.z);
            			
            			o.Normal = normalize(finalNormals);
            			//对高光强度赋值，范围0-1
            			o.Specular = _SpecWidth;
            			o.Gloss = _SpecIntesity;
            			//颜色
            			o.Albedo = c.rgb * _MainTint;
            			//alpha
            			o.Alpha = c.a;
            
            		}
            		ENDCG
            	} 
            	FallBack "Diffuse"
            }



##效果

对3张贴图正确拖拽到材质球上，可以看到：

*<center><img src="/public/img/200.png" style="width:50%"></center>*

将材质球赋给衣服模型：

*<center><img src="/public/img/201.png" style="width:50%"></center>*

从一个和衣服表面很小的夹角看，观察菲涅尔效果：

*<center><img src="/public/img/202.png" style="width:100%"></center>*

其他的效果就自己试试啦。

该Unity工程可以从[这里下载](http://pan.baidu.com/s/1jG0BzBg "这里下载")


今天就讲这么多，下篇文章见~