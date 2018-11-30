---
layout: post
title: 【Unity Shaders】七：让Texture动起来：UV动画与sprite sheet
category: 技术
tags: Unity Shaders
description: 
---

>22:46 2015/08/10 于工学一号馆312

这一篇文章中，我们将讲解如何使用表面着色器来修改纹理Uv坐标以滚动贴图，然后再介绍sprite sheet实现2D动画。
##简单的UV移动效果


首先来看看，为了实现纹理的uv动画，我们需要做什么：

1. 首先，我们要在ProPerties模块中加入两个控制UV坐标变换速度的变量：

    	Properties
    	{
    	    //主纹理贴图
    		_MainTexture("Main Texture",2D)="white"{}
    		
    		//两个控制速度的变量
    		_xRcrollingSpeed("xRcrollingSpeed",float)=1
    		_yRcrollingSpeed("yRcrollingSpeed",float)=1
    	}
    	
2. 不要忘记，上面这些变量在CGPROGRAM...ENDCG模块中要再声明一遍，因为我们后面要访问它们。

    
        
    		CGPROGRAM
    		
    		#pragma surface surf Lambert
    		
    		sampler2D _MainTexture;
    		float _xRcrollingSpeed;
    		float _yRcrollingSpeed;
    	
            ...

3. 要记得表面着色器的要素之一：输入结构的定义

    	struct Input
		{
			float2 uv_MainTexture;
		};
		
4. 重点来了，在表面函数中我们进行坐标的变化：
    

        void surf(Input IN,inout SurfaceOutput o)
    		{
    			
    			float2 sourceUv = IN.uv_MainTexture;
    			
                //关注重点在这里
    			float xRcrollingSpeed = _xRcrollingSpeed*_Time.y;
    			float yRcrollingSpeed = _yRcrollingSpeed*_Time.y;
    				
    			sourceUv += float2(xRcrollingSpeed,yRcrollingSpeed);
    
    			float4 c = tex2D(_MainTexture,sourceUv);
    
    			o.Albedo = c.rgb;
    			o.Alpha = c.a;
    
    		}
    		
完整的代码：

        Shader "MyShader/ScrollingUV"
        {
        	Properties
        	{
        		_MainTexture("Main Texture",2D)="white"{}
        		_xRcrollingSpeed("xRcrollingSpeed",float)=1
        		_yRcrollingSpeed("yRcrollingSpeed",float)=1
        	}
        	SubShader
        	{
        		CGPROGRAM
        		#pragma surface surf Lambert
        		struct Input
        		{
        			float2 uv_MainTexture;
        		};
        
        		sampler2D _MainTexture;
        		float _xRcrollingSpeed;
        		float _yRcrollingSpeed;
        
        		void surf(Input IN,inout SurfaceOutput o)
        		{
        			
        			float2 sourceUv = IN.uv_MainTexture;
        			
        			float xRcrollingSpeed = _xRcrollingSpeed*_Time.y;
        			float yRcrollingSpeed = _yRcrollingSpeed*_Time.y;
        				
        			sourceUv += float2(xRcrollingSpeed,yRcrollingSpeed);
        
        			float4 c = tex2D(_MainTexture,sourceUv);
        
        
        			o.Albedo = c.rgb;
        			o.Alpha = c.a;
        
        		}
        
        		ENDCG
        	}
        	 FallBack "Diffuse" 
        }
        

把这段代码保存好，回到Unity的Inspector面板，把下面这张图赋予材质球，点击运行就可以看到动态的纹理效果啦。

*<center><img src="/public/img/176.png" style="width:50%"></center>*

本来使用录像工具鲁了一段视频，再转化为gif，结果图片不清晰，还是不贴出来了。大家可以在自己的Unity中试验。


##序列帧动画效果
有时候，我们得到的图片中含有某对象的一系列动作帧，把这些动作按顺序逐步播放就能得到连贯的动画：

*<center><img src="/public/img/177.png" style="width:50%"></center>*

那么这里讲的就是如何把上面这一张图制作成2D动画。实际上在Unity已经有许多插件来完成这些工作，但是为了更好地了解2D动画的原理，熟悉shader如何改变UV坐标达到动画效果，我们还是亲手来制作一下。完了完成目标，我们需要做什么？

1. 新建一个shader，在编辑器中打开，在Properties添加三个新属性：

        	Properties
        	{
        		_MainTexture("Main Texture",2D)="white"{}
        
        		//添加这三个控制属性
        		_TexWidth("Sheet Width",float)=0.0
        		_CellAmount("Cell Amount",float)=0.0
        		_SwitchSpeed("Switch Speed",Range(1,10))=5
        	}
2.国际惯例，在CGPROGRAM...与ENDCG中添加上面属性的声明
    
        	SubShader
    	{
    		CGPROGRAM
    		#pragma surface surf Lambert
    		
    		sampler2D _MainTex;
    		float _TexWidth;
    		float _CellCount;
    		float _SwitchSpeed;
    
    		ENDCG
    	}
    	
3.输入结构的就不再提了
    	
4.然后开始写我们的表面函数surf了：
    void surf(Input IN,inout SurfaceOutput o)
    		{
    			//将uv坐标值保存在变量中
    			float2 spriteUV= IN.uv_MainTex;
    
    			//计算每个动作占据整张图的百分比
    			float cellUVPercentage = 1.0/_CellAmount;
    
    			//通过系统时间计算偏移量来得到不同的小图
    			float timeVal = fmod (_Time.y*_SwitchSpeed,_CellAmount);
    			timeVal = ceil(timeVal);
    
    			//改变x方向上的偏移量
    			float xValue = spriteUV.x;
    			xValue += timeVal;
    			xValue *=cellUVPercentage;
    
    			spriteUV = float2(xValue, spriteUV.y);
    			
    			float4 c = tex2D (_MainTex, spriteUV);
    			o.Albedo = c.rgb;
    			o.Alpha = c.a;
    		}
    		

最后贴上我们shader的完整代码：

        Shader "MyShader/sprite sheet"
        {
        	Properties
        	{
        		_MainTex("Base (RGB)",2D)="white"{}
        
        		//添加这三个控制属性
        		_TexWidth("Sheet Width",float)=0.0
        		_CellAmount("Cell Amount",float)=0.0
        		_SwitchSpeed("Switch Speed",Range(1,30))=12
        	}
        	SubShader
        	{
        		Tags { "RenderType"="Opaque" }  
                LOD 200  
        
        		CGPROGRAM
        		#pragma surface surf Lambert
        
        		sampler2D _MainTex;
        
        		float _TexWidth;
        		float _CellAmount;
        		float _SwitchSpeed;
        
        		//输入结构
        		struct 	Input
        		{
        			float2 uv_MainTex;
        		};
        
        		//表面函数
        		void surf(Input IN,inout SurfaceOutput o)
        		{
        			//将uv坐标值保存在变量中
        			float2 spriteUV= IN.uv_MainTex;
        
        			//计算每个动作占据整张图的百分比
        			float cellUVPercentage = 1.0/_CellAmount;
        
        			//通过系统时间计算偏移量来得到不同的小图
        			float timeVal = fmod (_Time.y*_SwitchSpeed,_CellAmount);
        			timeVal = ceil(timeVal);
        
        			//改变x方向上的偏移量
        			float xValue = spriteUV.x;
        			xValue += timeVal;
        			xValue *=cellUVPercentage;
        
        			spriteUV = float2(xValue, spriteUV.y);
        			
        			float4 c = tex2D (_MainTex, spriteUV);
        			o.Albedo = c.rgb;
        			o.Alpha = c.a;
        		}
        		ENDCG
        	}
        	 FallBack "Diffuse"  
        }
##代码解析
>float cellUVPercentage = 1.0/_CellAmount;

为了每个时刻只显示一个小图，我们需要对整张图片进行缩放，这里计算的就是缩放比例。实例中一张texture共有8个小图，所以 cellUVPercentage = 1.0/_CellAmount = 1.0/8.0 = 0.125;

>float timeVal = fmod (_Time.y*_SwitchSpeed,_CellAmount);

首先来看_Time是什么。它是内置的shader变量，可以在[内置变量](http://docs.unity3d.com/Manual/SL-UnityShaderVariables.html "内置变量")进行查询
*<center><img src="/public/img/178.PNG" style="width:50%"></center>*
可以看到_Time记录了从场景开始运行时的时间计数，它有三个参数，代表了不同的时间倍数。如_Time.y就代表三倍时间计数。


*<center><img src="/public/img/179.PNG" style="width:50%"></center>*

fmod函数是对浮点数的求余计算，它返回x/y的余数。实例中返回的范围是在0-8间的小数。为了得到整数，使用函数ceil函数向上求整：

>timeVal = ceil(timeVal); 

下面这部分代码是最难理解的：

>float xValue = spriteUV.x;  
>
>02.xValue += timeVal;  
>
>03.xValue *= cellUVPercentage;  

第一行首先声明一个新的变量xValue，用于存储用于图片采样的x坐标。它首先被初始为surf函数的输入参数In的横坐标。类型为Input的输入参数In代表输入的texture的UV坐标，范围为0到1。第二行向原值加上小图的整数偏移量，最后为了只显示一张小图，我们还需将x值乘以小图所占百分比cellUVPercentage。


保存代码之后，点击Play就可以看到动画效果啦，当然图片还是静态的...
*<center><img src="/public/img/180.PNG" style="width:50%"></center>*







关于这些图片，在google搜索[sprite sheet](https://www.google.com/search?q=sprite+sheet&biw=1371&bih=688&tbm=isch&tbo=u&source=univ&sa=X&ved=0CBwQsARqFQoTCKvE0cDYnscCFcyZiAodR48Iug&dpr=1.4 "sprite sheet")就能找到很多。
	

    




