### 第一个Shader

```c++
Shader "Custom/BasicDiffuse" {
	Properties {
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
	}
	SubShader {
		Tags { "RenderType"="Opaque" }
		LOD 200
		
		CGPROGRAM
		#pragma surface surf Lambert
 
		sampler2D _MainTex;
 
		struct Input {
			float2 uv_MainTex;
		};
 
		void surf (Input IN, inout SurfaceOutput o) {
			fixed4 c = tex2D (_MainTex, IN.uv_MainTex);
			o.Albedo = c.rgb;
			o.Alpha = c.a;
		}
		ENDCG
	}
	FallBack "Diffuse"
}

```

#### **Properties**

里面包含shader的属性

```c++
_MainTex ("Albedo (RGB)", 2D) = "white" {}
//MainTex表示变量名
//Albedo(RGB)是在编辑器里显示的名称
//2D是它的类型，表示它是一个纹理
//white是默认值。
```

- Range(min,max)，创建float属性，滑条形式

- Color，颜色属性

- 2D，纹理属性

- Rect，创建一个非2次方的纹理属性 作为2D GUI元素

- Cude，在Inspector面板上创建立方贴图属性，允许用户直接拖曳立方贴图作为着色器属性

- Float，在 Inspector 面板上创建一个非滑动条的 float 属性

- Vector，创建一个拥有 4个float值的属性，可以用于标记方向或颜色 


#### SubShader

​	子着色器，一个着色器可以有多个SubShader。子着色器是代码的主体，计算着色的时候，平台会按顺序选择一个可以使用的子着色器进行执行，如果所有的子着色器都无法使用，则会执行最后**FallBack**里指定的着色器。

##### Tag（标签） 标记着色器的一些特性

常用Tag：

- RenderType，渲染类型，常用为Opaque（不透明）和Transparent（透明）
- IgnoreProjector，是否忽略投影器
- ForceNoShadowCasting，是否强制无阴影
- Queue，渲染队列，内置值Background=1000，Geometry=2000，AlphaTest=2450，Transparent=3000，Overlay=4000，但是并不限于这些值，可以填写自己的值

##### LOD

Level of Details的缩写，表示着色器的细节层级，高于Unity的最大LOD（Quality Settings里设置）的shader将不可用。在调低画质时，可以根据这个值舍弃一部分的shader。

##### CGPROGRAM与ENDCG，表示在二者范围内有一段Cg（C for graphics）代码

#pragma surface surf Lambert

这一行表明我们使用的是一个表明着色器，方法名称是surf，光照模型是Lambert。

然后是

sampler2D  _MainTex;

sampler2D对应于Properties里面的2D，是2D贴图的数据结构。而_MainTex也对应于Properties里面的_MainTex，保存了编辑器（或者代码）里设置的贴图。二者必须是同名，才能将贴图数据链接起来。简而言之，下面的_MainTex是上面的_MainTex在Cg代码里的代理。

然后是一个结构（struct）定义：

```c++
struct Input {
	float2 uv_MainTex;
};
```

这个结构是为surf方法定义了输入参数的数据结构。

float2表示这是一个二维的浮点型坐标。

其他的内置类型还包括：

half：半精度浮点型，范围[-60000,60000]

fixed：低精度定点型，范围[-2,2]

int：整型

bool：布尔型

sample*d：纹理类型

uv_MainTex表示_MainTex的纹理坐标（参考百度百科[UV坐标](http://baike.baidu.com/link?url=Ud19scY012C7WZnd-LDdi7etKQ8CbX9fAWGW6D4xOYle6lGyU5dgEzYPO_K5vyAOXeW22-EC68wd6JtCS6fzDa)），这是一种命名约定。

最后是surf方法

```
void surf (Input IN, inout SurfaceOutput o) {
    fixed4 c = tex2D (_MainTex, IN.uv_MainTex);
    o.Albedo = c.rgb;
    o.Alpha = c.a;
}
```

Input结构是我们在上面定义的。

SurfaceOutput的数据结构：

```c++
struct SurfaceOutput {
    half3 Albedo;     //像素的颜色
    half3 Normal;     //像素的法向值
    half3 Emission;   //像素的发散颜色
    half Specular;    //像素的镜面高光
    half Gloss;       //像素的发光强度
    half Alpha;       //像素的透明度
};
```

  surf方法主体第一行：

```c++
fixed4 c = tex2D (_MainTex, IN.uv_MainTex);
```

使用tex2D方法从_MainTex里面取出指定纹理坐标（IN.uv_MainTex）的色彩值。

取出来色彩值之后，很简单，将c的rgb分量赋值给输出o的Albedo，把c的a分量赋值给输出o的  Alpha。

写一个颜色的

```
Shader "Custom/TestColor" {
	Properties {
		_Color ("Color", Color) = (1,1,1,1)
	}
	SubShader {
		Tags { "RenderType"="Opaque" }
		LOD 200
		
		CGPROGRAM
		#pragma surface surf Lambert
 
		fixed4 _Color;
 
		struct Input {
			float2 uv_MainTex;
		};
 
		void surf (Input IN, inout SurfaceOutput o) {
			o.Albedo = _Color.rgb;
			o.Alpha = _Color.a;
		}
		ENDCG
	}
	FallBack "Diffuse"
}

```

  