---
title: 从Built-in到URP
date: 2021-05-22 9:00:00
tags: URP
description: HLSL的语法。
cover: https://cdn.jsdelivr.net/gh/jerryc127/CDN@latest/cover/default_bg.png
---

## HLSL语法

### 变量

-   **bool** – true or false.
-   **float**- 32位浮点数。通常用于世界空间位置，纹理坐标或涉及复杂函数（例如三角函数或幂/幂）的标量计算。
-   **half** – 16位浮点数。通常用于短向量，方向，对象空间位置，颜色。
-   **double** – 64位浮点数。不能用作输入/输出
-   **fixed** – 仅在内置着色器中使用，在URP中不支持，请改用**half** 。
-   **real** – 一般手机平台相当于half，pc平台相当于float。
-   **int** – 32位有符号整数
-   **uint** – 32位无符号整数（GLES2除外，不支持此整数，而是将其定义为int）。
-   
### 向量

-  float4 – 包含4个浮点的向量
-   float3 – 包含3个浮点的向量
-   float2 – 包含2个浮点的向量
-   
### 矩阵

-   float4x4 – 4行，4列
-   float4x3 – 4行，3列
-   float2x1 – 2行，1列
-   float1x4 – 1行，4列
```glsl
float3x3 matrix = {0,1,2,
                   3,4,5,
                   6,7,8};
float3 row0 = matrix[0]; // (0, 1, 2)
float3 row1 = matrix[1]; // (3, 4, 5)
float3 row2 = matrix[2]; // (6, 7, 8)
float row1column2 = matrix[1][2]; // 5
// 注意我们也可以这样做
float row1column2 = matrix[1].2;
```
矩阵通常用于不同坐标空间之间的转换。为此，我们需要进行矩阵乘法，可以使用**mul**函数来完成，传统的*/不在适用。

### 数组

可以在着色器中指定数组，尽管Shaderlab属性或材质检查器不支持它们，并且必须从C＃脚本中进行设置。必须在着色器中指定数组的大小，并且数组大小应保持恒定以防止出现问题。如果我们不知道数组的大小，则需要设置最大值并以0s传入数组填充。我们可以指定另一个float来作为需要遍历数组的长度，例如此处的示例。

```glsl
float _Array[10]; // Float array
float4 _Array[10]; // Vector array
float4x4 _Array[10]; // Matrix array
```

设置浮点数组时，请使用**material.SetFloatArray**或**Shader.SetGlobalFloatArray**。还有**SetVectorArray**和**SetMatrixArray**及其全局版本

### 其他种类

HLSL还包括其他类型，例如“纹理”和“采样器”，可以使用URP中的以下宏进行定义。
```glsl
TEXTURE2D(textureName);
SAMPLER(sampler_textureName);
```
还有缓冲区，尽管我从未真正使用过它们，所以对它们的用法并不熟悉。它们是使用**material.SetBuffer**或**Shader.SetGlobalBuffer**从C＃设置的。

```glsl
#ifdef SHADER_API_D3D11
StructuredBuffer<float3> buffer;
#endif
// I think this is only supported in Direct3D 11?
// and also require #pragma target 4.5 or higher?
// see https://docs.unity3d.com/Manual/SL-ShaderCompileTargets.html
```

你可能还希望研究HLSL的其他部分，例如[流控制](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-flow-control) （if，for，while等），但是如果我们熟悉语法，则其语法基本上与C＃相同。我们还可以[在此处](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-operators)找到HLSL支持的所有运算符的列表。

### 函数

HLSL中的函数声明与C＃非常相似。这是一个例子：

```glsl
float3 example(float3 a, float3 b){
    return a + b;
}
```

其中float3是返回类型，示例是函数名称，括号内是传递给函数的参数。在没有返回类型的情况下，将使用void。您还可以在参数类型之前使用“ out”来指定输出参数，如果希望它成为可编辑并回传的输入，则可以使用“ inout”来指定输出参数。

### 宏

宏在编译着色器之前进行处理，并且在使用宏时将替换为带有替换参数的定义。例如

```glsl
#define EXAMPLE(x, y) ((x) * (y))
```

```glsl
float f = EXAMPLE(3, 5);
float3 a = float3(1,1,1);
float3 f2 = EXAMPLE(a, float3(0,1,0));
 
// becomes :
float f = ((3) * (5));
float a = float(1,1,1);
float3 f2 = ((a) * (float3(0,1,0)));
// then the shader is compiled.
 
// Note that the macro has () around x and y.
// This is because we could do :
float b = EXAMPLE(1+2, 3+4);
// becomes :
float b = ((1+2) * (3+4)); // 3 * 7, so 21
// If those () wasn't included, it would instead be :
float b = (1+2*3+4)
// which equals 11 due to * taking precedence over +
```

他们还可以做一些功能无法做到的事情。例如 ：

```glsl
#define TRANSFORM_TEX(tex,name) (tex.xy * name##_ST.xy + name##_ST.zw)
 
// Usage :
OUT.uv = TRANSFORM_TEX(IN.uv, _MainTex)
 
// becomes :
OUT.uv = (IN.uv.xy * _MainTex_ST.xy + _MainTex_ST.zw);
```

“##”运算符是一种特殊情况，其中宏可能很有用。它使我们可以将名称和\_ST部分连接起来，从而为此用法输入生成\*\*\_MainTex\_ST\*\*。如果省略##部分，它将仅生成“\*\*name\_ST\*\*”，从而导致错误，因为尚未定义。（当然，仍然需要定义\*\*\_MainTex\_ST\*\*，但这是预期的行为，因为在纹理名称后附加\*\*\_ST\*\*是Unity处理\*\*纹理的平铺和偏移值\*\*的方式）。
## Tags

URP LIGHTMODE TAGS :

-   **UniversalForward** – 用于前向渲染
-   **ShadowCaster** – 用于投射阴影
-   **DepthOnly** – 似乎在为场景视图渲染深度纹理时使用，而不是在运行中使用吗？不过，某些渲染器功能可能会使用它。
-   **Meta** – 仅在光照贴图烘焙期间使用
-   **Universal2D** – 在启用 2D 渲染器时使用，而不是前向渲染器。
-   **UniversalGBuffer** – 与延迟渲染有关。我认为这是测试功能。

```
Tags { "LightMode" = "UniversalForward" }
```

> 可以在子着色器中定义多个Pass块，但是每个都应该用一个特定的LightMode标记(见下面)。URP使用了单通道前向渲染器，所以只有第一个“通用前向”通道(GPU支持的)将用于渲染对象——你不能同时渲染多个对象。虽然我们可以让其他传递没有标记，但要注意**它们将中断SRP批处理程序的批处理**。相反，我们建议使用单独的着色器/材质，无论是在单独的MeshRenderers上，还是使用Forward Renderer上的Render Objects特性，用一个overrideMaterial在一个特定的图层上重新渲染对象。

## 属性

在Shaderlab示例中，我们有一个**HLSLINCLUDE**，它会自动将代码包含在Subshader内部的每个Pass中。

我们可以使用**UnityPerMaterial CBUFFER**来确保着色器兼容SRP批处理。这个**CBUFFER**需要包括所有公开的属性(与Shaderlab属性块中的相同)。但它不能包括其他未公开的变量，纹理也不需要被包括。

```glsl
HLSLINCLUDE
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
 
    CBUFFER_START(UnityPerMaterial)
    float4 _BaseMap_ST;
    float4 _BaseMap_TexelSize;
    float4 _BaseColor;
    //float4 _ExampleDir;
    //float _ExampleFloat;
    CBUFFER_END
ENDHLSL
```

> 需要注意的是\*\*\_BaseMap\_ST\*\*与\*\*\_BaseMap\_TexelSize\*\*是两个东西，前者是纹理的缩放与偏移，而后者代表纹理的大小。

## 结构体

在定义顶点或片段着色器功能之前，我们需要定义一些用于将数据传入和传出的结构。在内置函数中，它们通常被命名为“**appdata**”和“**v2f**”（顶点到片段的缩写），而URP着色器则倾向于使用“ **Attributes**”和“ **Varyings** ”。这些只是名称，可能不太重要。

```glsl
struct Attributes {
    float4 positionOS   : POSITION;
    float2 uv           : TEXCOORD0;
    float4 color        : COLOR;
};
```

该属性结构将输入到**顶点着色器**。它允许我们使用大写字母中被称为语义的部分从网格中获取每个顶点的数据。其中包括：顶点位置（**POSITION**），顶点颜色（**COLOR**）和**UV**（又称为纹理坐标）。网格具有8个不同的UV通道，可以通过**TEXCOORD0**到**TEXCOORD7**进行访问。

我们还可以通过**NORMAL**访问顶点法线，并通过**TANGENT**访问切线。

在这些结构之后，您通常还会看到已定义了**纹理**和**采样器**（虽然纹理位于着色器属性中，但尚未在hlsl中定义。其他属性包括在**CBUFFER**中）。在URP中，我们使用以下内容：

```glsl
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);
```

## 顶点着色器

我们的顶点着色器需要做的主要事情是将网格从对象空间位置转换为剪辑空间位置。为了在目标屏幕位置正确渲染片元/像素。

在内置着色器中，您可以使用**UnityObjectToClipPos**函数执行此操作，但是URP已将其重命名为**TransformObjectToHClip**（可以在函数库SpaceTransforms.hlsl中找到）。也就是说，还有另一种方法来处理URP中的转换，如下所示。

```glsl
Varyings vert(Attributes IN) {
    Varyings OUT;
 
    VertexPositionInputs positionInputs = GetVertexPositionInputs(IN.positionOS.xyz);
    OUT.positionCS = positionInputs.positionCS;
    // Or this :
    //OUT.positionCS = TransformObjectToHClip(IN.positionOS.xyz);
 
    OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
    OUT.color = IN.color;
    return OUT;
}
```

1.  我们从**Attributes**中输入对象空间的位置，并获得一个**VertexPositionInputs**结构，其中包含：
    
    -   positionWS，在世界空间中的位置
    -   positionVS，视图空间中的位置
    -   positionCS，裁剪空间中的位置
    -   positionNDC，标准化设备坐标中的位置
2.  顶点着色器还负责将数据传递到片段。对于顶点颜色，这只是一个简单的`OUT.color = IN.color;`。
    
3.  如果我们希望能够对纹理进行采样，则还需要传递模型的UV（纹理坐标）。虽然我们可以做`OUT.uv = IN.uv;`（假设两者均为float2），通常会使用**TRANSFORM\_TEX**宏，该宏采用uv和texture属性名称，并应用材质检查器的偏移和平铺进行矫正（存储在“ \_BaseMap” +“ \_ ST”中，S用于比例尺和T））。此宏位于内置和URP中（在core / ShaderLibrary / Macros.hlsl内部，应自动包含在Core.hlsl中）。
    
    实际上，这只是`IN.uv.xy * _BaseMap_ST.xy + _BaseMap_ST.z`的简写，因此您也可以这样写（将\_BaseMap换成预期的纹理属性。`（texture）_ST`float4变量还必须添加到**UnityPerMaterial CBUFFER**（已在属性部分中讨论过）。
    

```glsl
VertexNormalInputs normalInputs = GetVertexNormalInputs(IN.normalOS, IN.tangentOS);
```

**GetVertexNormalInputs**可用于将对象空间的法线和切线转换为世界空间。它包含:

-   normalWS，在世界空间中的法线向量
-   tangentWS，在世界空间中的切线向量
-   bitangentWS，在世界空间中的副切线向量

还有一个仅将法线作为输入的版本，将tangentWS保留为（1,0,0），bitangentWS保留为（0,1,0），或者您也可以改用`TransformObjectToWorldNormal(IN.normalOS)`。

## 片元着色器

```glsl
half4 frag(Varyings IN) : SV_Target {
    half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
 
    return baseMap * _BaseColor * IN.color;
}
```

这将生成一个着色器，该着色器基于\_BaseMap纹理输出一个Half4颜色，该着色器还由\_BaseColor和顶点颜色（**IN.color**）进行着色。

**SV\_Target**部分是与**half4**输出一起使用的语义，它告诉着色器它是颜色输出。

还有一个**SV\_Depth**输出，它是一个浮点数，用于覆盖每个像素的Z缓冲区值。（可以将它们放入一个结构中以同时输出**SV\_Target**和**SV\_Depth**）。在大多数情况下，不需要覆盖它，对于许多GPU，它都会关闭某些基于深度缓冲区的优化，因此除非您知道自己在做什么和需要做什么，否则不要覆盖它。

我们的片段着色器使用URP ShaderLibrary提供的**SAMPLE\_TEXTURE2D**宏对\*\*\_BaseMap\*\*纹理进行采样，该宏将纹理，采样器和UV作为输入。

我们可能还想做的是，如果像素的alpha值低于某个阈值，则将其丢弃，以使整个网格都不可见。

例如，对于四边形上的草/叶纹理。既可以在不透明着色器中也可以在透明着色器中完成此操作，通常将其称为Alpha裁剪。如果您熟悉shadergraph，可以使用主节点上的“Alpha Clip Threshold”输入来处理它。

解决此问题的常用方法是提供\*\*\_Cutoff属性\*\*以控制阈值，然后执行以下操作。（此属性必须添加到我们的Shaderlab属性以及UnityPerMaterial CBUFFER中以实现SRP Batcher兼容性）。

```glsl
if (_BaseMap.a < _Cutoff){
    discard;
}
// OR
clip(_BaseMap.a - _Cutoff);
// inside the fragment function, before returning
```

环境光在URP下用\_GlossyEnvironmentColor获取，但得到的效果可能与Builit-in下的结果相差较大，这时候可以考虑用球谐函数获取

```glsl
//URP使用的环境光
half3 ambient = _GlossyEnvironmentColor
//使用球谐函数获取
half3 ambient = SampleSH(worldNormal);

//--Builit-in
half3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
```

## 关键字和着色器变体

### 着色器变体

在着色器中，我们可以指定更多的\*\*#pragma\*\*指令，其中一些指令包括\*\*multi\_compile\*\*和\*\*shader\_feature\*\*。这些可用于指定用于将“着色器”代码的某些部分“打开”或“关闭”的关键字。着色器实际上被编译为多个版本的着色器，称为\*\*着色器变体\*\*。

`MULTI_COMPILE`

```glsl
#pragma multi_compile _A _B _C (...etc)
```

在此示例中，我们将生成着色器的三个变体，其中\_A，\_B和\_C是关键字。

在着色器代码中，我们可以使用以下内容：

```glsl
#ifdef _A
// 如果A启用，编译此代码
#endif
 
#ifndef _B
// 当B被禁用时编译此代码，也就是只在A和C中。
// 注意#ifndef中额外的“n”表示“如果没有定义”
#else
// 如果B启用，编译此代码
#endif
 
#if defined(_A) || defined(_C)
// 用A或c (aka与上面的相同，假设没有其他关键字)编译此代码
// 如果需要多个条件，则必须使用长形式的"#if defined()"
// 其中|| = or， && = and
// 注意，因为关键字是在一个multi_compile语句中定义的
// 实际上不可能同时启用两者，所以&&在这里没有意义。
#endif
 
// 还有#elif，用于else if语句。
```

`SHADER_FEATURE`

```glsl
#pragma shader_feature _A _B
```

这与**multi\_compile**完全相同，但是未使用的变体将不包括在最终版本中。因此，在运行时启用/禁用这些关键字是不好的，因为它所需的着色器可能未包含在构建中！如果需要在运行时处理关键字，请改用**multi\_compile**。

这些指令还有“顶点”和“片元”版本，可用于仅针对顶点或片段程序编译着色器变体，从而减少了变体的总数。例如 ：

```glsl
#pragma multi_compile_vertex _ _A
#pragma multi_compile_fragment _ _B
// also shader_feature_vertex and shader_feature_fragment
```

在此示例中，\_A关键字仅用于顶点程序，\_B仅用于片元。不能同时启用\_A和\_B的变体。Unity告诉我们，这会产生2个着色器变体，尽管当您查看实际的编译代码时，它更像是一个禁用两个着色器的着色器变体和两个“half”的变体。

#### 着色器变体的增长

每增加一个multi\_compile和shader\_feature，它就会为启用/禁用关键字的每种可能组合生成越来越多的着色器变体。以以下为例：

```glsl
#pragma multi_compile _A _B _C
#pragma multi_compile _D _E
#pragma shader_feature _F _G
```

在这里，第一行将生成3个着色器变体。但是第二行需要为已启用\_D或\_E的那些变体生成2个着色器变体。

因此，A＆D，A＆E，B＆D，B＆E，C＆D和C＆E。现在有6个变体。

第三行，是这6个中的每一个的另外2个变体，因此我们现在总共有12个着色器变体。由于该行是shader\_feature，因此某些变体可能不会包含在构建中。

每个添加了2个关键字的**multi\_compile**都会使产生的变体数量加倍，因此包含10个变体的着色器将产生1024个着色器变体！它需要编译最终构建中需要包含的每个着色器变体，因此将增加构建时间以及构建大小。

#### 如何查看着色器的变体个数

如果要查看一个着色器产生多少个着色器变体，请单击该着色器，然后在检查器中有一个“Compile and Show Code”按钮，旁边是一个小的下拉箭头，其中列出了所包含的变体数。如果单击“skip unused shader\_features”，则可以切换以查看变体的总数。

![image-20201210114824886](file:///E:/%5cGit%5cSiKi%5c%e6%96%87%e6%a1%a3%5cURP%5cBuilt-in%e5%88%b0URP.assets%5cimage-20201210114824886.png)

### 关键字

每个项目最多还有256个关键字，因此最好遵循其他着色器的命名约定。

您还会注意到，对于许多**multi\_compile**和**shader\_features**而言，第一个关键字通常仅保留为“ \_”。实际上，这实际上不会产生关键字，因此会为256个最大值的其他关键字留出更多空间。

```glsl
#pragma multi_compile _ _KEYWORD
 
#pragma shader_feature _KEYWORD
// 仅是shader_features的简写
#pragma shader_feature _ _KEYWORD
 
// 如果您需要知道该关键字是否已禁用
// 然后我们可以这样做：
#ifndef _KEYWORD
// 或#if！defined（_KEYWORD）
// 或#ifdef _KEYWORD #else
// code
#endif
```

我们还可以通过使用multi\_compile和shader\_feature的本地版本来避免耗尽最大的关键字数。这些生成的关键字对于该着色器来说是本地的，但是每个着色器最多也有64个本地关键字。

```glsl
#pragma multi_compile_local _ _KEYWORD
#pragma shader_feature_local _KEYWORD
 
// 还有local_fragment/vertex !
#pragma multi_compile_local_fragment _ _KEYWORD
#pragma shader_feature_local_vertex _KEYWORD
```

## 光照

Universal RP不支持表面着色器，但是ShaderLibrary确实提供了帮助我们处理大量光照计算的功能。这些包含在**Lighting.hlsl**中

在**Lighting.hlsl**中，有一个**GetMainLight**函数，如果您熟悉着色器图中的自定义照明，您可能已经知道。为了使用此功能，我们首先在**HLSLPROGRAM**的顶部引用Lighting.hlsl文件，我还将添加一些**multi\_compile**指令，这些指令提供了接收阴影所需的关键字。

```glsl
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE
#pragma multi_compile _ _SHADOWS_SOFT
 
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
```

接下来，我们将需要顶点法线来处理阴影/光照，因此我们将它们添加到**Attributes**和**Varyings**结构中，并更新顶点着色器。在这里，我仅显示基于上一节中制作的Unlit着色器添加的代码。

```glsl
struct Attributes {
    ...
    float4 normalOS     : NORMAL;
};
 
struct Varyings {
    ...
    float3 normalWS     : NORMAL;
    float3 positionWS   : TEXCOORD2;
};
...
Varyings vert(Attributes IN) {
    Varyings OUT;
    VertexPositionInputs positionInputs = GetVertexPositionInputs(IN.positionOS.xyz);
    ...
    OUT.positionWS = positionInputs.positionWS;
 
    VertexNormalInputs normalInputs = GetVertexNormalInputs(IN.normalOS.xyz);
    OUT.normalWS = normalInputs.normalWS;
 
    return OUT;
}
```

在片元着色器中，我们现在可以采用世界空间法线，并使用世界空间位置来计算阴影坐标。

```glsl
half4 frag(Varyings IN) : SV_Target {
    half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
    half4 color = baseMap * _BaseColor * IN.color;
 
    float4 shadowCoord = TransformWorldToShadowCoord(IN.positionWS.xyz);
    Light light = GetMainLight(shadowCoord);
 
    half3 diffuse = LightingLambert(light.color, light.direction, IN.normalWS);
 
    return half4(color.rgb * diffuse * light.shadowAttenuation, color.a);
}
```

虽然我们的着色器将从其他着色器接收阴影，但是请注意，它没有**ShadowCaster**传递，因此不会将阴影投射到自身或其他对象上。请参见**ShadowCaster**部分。

如果我们需要阴影，但对象上没有漫反射阴影，则也可以删除漫反射阴影计算，而只需使用**light.shadowAttenuation**。

如果要进一步扩展以包括环境/烘焙GI和其他光源，请以**Lighting.hlsl**中的**UniversalFragmentBlinnPhong**方法为例，或者让它为您处理照明。它使用**InputData**结构，下一部分讨论的PBR示例也将使用该结构。

## PBR光照

基于物理的渲染（PBR）是Unity的“Standard”着色器使用的着色/照明模型，以及UPR的“ Lit”着色器和ShaderGraph中的PBR主节点。

如前一节所述，内置管道中的阴影/照明通常由Surface Shaders处理，其中“Standard”选用是PBR模型。它们使用了一个曲面函数，该函数输出了**反照率，法线，发射，平滑度，遮挡，Alpha和Metallic**（如果使用“ StandardSpecular”工作流程，则为Specular）。Unity将采用这些并在幕后生成一个顶点和片段着色器，为您处理某些计算，例如PBR阴影/照明和阴影。

Universal RP不支持表面着色器，但是ShaderLibrary确实提供了帮助我们处理大量光照计算的功能。这些包含在**Lighting.hlsl**中。在本节中，我们将重点介绍**UniversalFragmentPBR**：

```glsl
half4 UniversalFragmentPBR(InputData inputData, half3 albedo, half metallic, half3 specular, half smoothness, half occlusion,  half3 emission, half alpha)
 
// 在v10.xx中添加了带有SurfaceData结构的版本
// 对于之前的版本，需要改用以上版本。
//（但是您仍然可以使用SurfaceData结构来组织/保存数据）
half4 UniversalFragmentPBR(InputData inputData, SurfaceData surfaceData)
 
// 还有:
half4 UniversalFragmentBlinnPhong(InputData inputData, half3 diffuse, half4 specularGloss, half smoothness, half3 emission, half alpha)
//复制Unity v4之前的“旧”表面着色器，
//并由URP的“ SimpleLit”着色器使用
//使用Lambert（漫反射）和BlinnPhong（镜面反射）照明模型
```

首先，我们应该添加PBR照明模型使用的一些属性。我省去了金属/高光贴图和遮挡贴图，主要是因为它们没有很好的功能来为您处理采样（除非您从LitInput.hlsl中复制它们，这是URP提供的Lit shader的一部分） ，而不是实际的ShaderLibrary），并且此部分已经相当长且足够复杂。实际上我几乎无法解释，因为它主要是知道在哪里使用哪个函数。您以后总是可以使用LitInput作为示例来添加它们。

```glsl
Properties {
    _BaseMap ("Base Texture", 2D) = "white" {}
    _BaseColor ("Example Colour", Color) = (0, 0.66, 0.73, 1)
    _Smoothness ("Smoothness", Float) = 0.5
 
    [Toggle(_ALPHATEST_ON)] _EnableAlphaTest("Enable Alpha Cutoff", Float) = 0.0
    _Cutoff ("Alpha Cutoff", Float) = 0.5
 
    [Toggle(_NORMALMAP)] _EnableBumpMap("Enable Normal/Bump Map", Float) = 0.0
    _BumpMap ("Normal/Bump Texture", 2D) = "bump" {}
    _BumpScale ("Bump Scale", Float) = 1
 
    [Toggle(_EMISSION)] _EnableEmission("Enable Emission", Float) = 0.0
    _EmissionMap ("Emission Texture", 2D) = "white" {}
    _EmissionColor ("Emission Colour", Color) = (0, 0, 0, 0)
    }
...
// And need to adjust the CBUFFER to include these too
CBUFFER_START(UnityPerMaterial)
    float4 _BaseMap_ST; // Texture tiling & offset inspector values
    float4 _BaseColor;
    float _BumpScale;
    float4 _EmissionColor;
    float _Smoothness;
    float _Cutoff;
CBUFFER_END
```

我们还需要对Unlit着色器代码进行大量更改，包括添加一些**multi\_compile**和**shader\_features**以及对**Attributes**和**Varyings**结构进行调整，因为我们需要来自网格的法线和切线数据并将其发送到片元中以便使用它们用于照明计算。

“属性”块中的这些**TOGGLE**特性使我们能够从材质检查器启用/禁用shader\_feature关键字。（或者，我们可以为着色器编写自定义编辑器/检查器GUI或使用调试检查器）。

如果要支持烘焙的光照贴图，我们还需要在**TEXCOORD1**通道中传递的**光照贴图UV**。

我还使用了来自ShaderLibrary的**SurfaceInput.hlsl**来帮助完成某些事情，它可以帮助**SurfaceData**结构保存PBR所需的数据以及一些用于采样的反射率，法线和发射贴图的函数（请注意，该结构似乎已经移动了到URP v10中的**SurfaceData.hlsl**，但SurfaceInput.hlsl会自动包含它）

```glsl
// Material Keywords
#pragma shader_feature _NORMALMAP
#pragma shader_feature _ALPHATEST_ON
#pragma shader_feature _ALPHAPREMULTIPLY_ON
#pragma shader_feature _EMISSION
//#pragma shader_feature _METALLICSPECGLOSSMAP
//#pragma shader_feature _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A
//#pragma shader_feature _OCCLUSIONMAP
 
//#pragma shader_feature _SPECULARHIGHLIGHTS_OFF
//#pragma shader_feature _ENVIRONMENTREFLECTIONS_OFF
//#pragma shader_feature _SPECULAR_SETUP
#pragma shader_feature _RECEIVE_SHADOWS_OFF
 
// URP Keywords
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE
#pragma multi_compile _ _ADDITIONAL_LIGHTS_VERTEX _ADDITIONAL_LIGHTS
#pragma multi_compile _ _ADDITIONAL_LIGHT_SHADOWS
#pragma multi_compile _ _SHADOWS_SOFT
#pragma multi_compile _ _MIXED_LIGHTING_SUBTRACTIVE
 
// Unity defined keywords
#pragma multi_compile _ DIRLIGHTMAP_COMBINED
#pragma multi_compile _ LIGHTMAP_ON
#pragma multi_compile_fog
 
// Some added includes, required to use the Lighting functions
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"
// And this one for the SurfaceData struct and albedo/normal/emission sampling functions.
// Note : It also defines the _BaseMap, _BumpMap and _EmissionMap textures for us, so we should use these as Shaderlab Properties too.
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"
 
struct Attributes {
    float4 positionOS   : POSITION;
    float3 normalOS     : NORMAL;
    float4 tangentOS    : TANGENT;
    float4 color        : COLOR;
    float2 uv           : TEXCOORD0;
    float2 lightmapUV   : TEXCOORD1;
};
 
struct Varyings {
    float4 positionCS               : SV_POSITION;
    float4 color                    : COLOR;
    float2 uv                       : TEXCOORD0;
    DECLARE_LIGHTMAP_OR_SH(lightmapUV, vertexSH, 1);
    // Note this macro is using TEXCOORD1
#ifdef REQUIRES_WORLD_SPACE_POS_INTERPOLATOR
    float3 positionWS               : TEXCOORD2;
#endif
    float3 normalWS                 : TEXCOORD3;
#ifdef _NORMALMAP
    float4 tangentWS                : TEXCOORD4;
#endif
    float3 viewDirWS                : TEXCOORD5;
    half4 fogFactorAndVertexLight   : TEXCOORD6;
    // x: fogFactor, yzw: vertex light
#ifdef REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR
    float4 shadowCoord              : TEXCOORD7;
#endif
};
 
//TEXTURE2D(_BaseMap);
//SAMPLER(sampler_BaseMap);
// Removed, since SurfaceInput.hlsl now defines the _BaseMap for us
```

我们的“变量”现在还包含正在使用的光照贴图UV，法线和切线，但是我们还添加了“视图方向”，这对于照明计算，雾，顶点照明支持和接收阴影的阴影坐标是必不可少的。

现在我们需要更新顶点着色器以处理所有这些更改，这主要是仅知道要使用的功能：

```glsl
#if SHADER_LIBRARY_VERSION_MAJOR < 9
    // This function was added in URP v9.x.x versions
    // If we want to support URP versions before, we need to handle it instead.
    // Computes the world space view direction (pointing towards the viewer).
    float3 GetWorldSpaceViewDir(float3 positionWS) {
        if (unity_OrthoParams.w == 0) {
            // Perspective
            return _WorldSpaceCameraPos - positionWS;
        } else {
            // Orthographic
            float4x4 viewMat = GetWorldToViewMatrix();
            return viewMat[2].xyz;
        }
    }
#endif
 
Varyings vert(Attributes IN) {
    Varyings OUT;
 
    // Vertex Position
    VertexPositionInputs positionInputs = GetVertexPositionInputs(IN.positionOS.xyz);
    OUT.positionCS = positionInputs.positionCS;
#ifdef REQUIRES_WORLD_SPACE_POS_INTERPOLATOR
    OUT.positionWS = positionInputs.positionWS;
#endif
    // UVs & Vertex Colour
    OUT.uv = TRANSFORM_TEX(IN.uv, _BaseMap);
    OUT.color = IN.color;
 
    // View Direction
    OUT.viewDirWS = GetWorldSpaceViewDir(positionInputs.positionWS);
 
    // Normals & Tangents
    VertexNormalInputs normalInputs = GetVertexNormalInputs(IN.normalOS, IN.tangentOS);
    OUT.normalWS =  normalInputs.normalWS;
#ifdef _NORMALMAP
    real sign = IN.tangentOS.w * GetOddNegativeScale();
    OUT.tangentWS = half4(normalInputs.tangentWS.xyz, sign);
#endif
 
    // Vertex Lighting & Fog
    half3 vertexLight = VertexLighting(positionInputs.positionWS, normalInputs.normalWS);
    half fogFactor = ComputeFogFactor(positionInputs.positionCS.z);
    OUT.fogFactorAndVertexLight = half4(fogFactor, vertexLight);
 
    // Baked Lighting & SH (used for Ambient if there is no baked)
    OUTPUT_LIGHTMAP_UV(IN.lightmapUV, unity_LightmapST, OUT.lightmapUV);
    OUTPUT_SH(OUT.normalWS.xyz, OUT.vertexSH);
 
    // Shadow Coord
#ifdef REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR
    OUT.shadowCoord = GetShadowCoord(positionInputs);
#endif
    return OUT;
}
```

现在，我们还可以更新该片元着色器以实际使用**UniversalFragmentPBR**函数。由于它需要**InputData**结构输入，因此我们需要创建和设置它。代替在片元着色器中执行此操作，我们将创建另一个函数来帮助组织事物。

类似地，要处理所有反照率，金属，镜面，平滑度，遮挡，发射和Alpha输入，我们将使用**SurfaceData**结构（由我们之前包含的SurfaceInput.hlsl提供），并创建另一个函数来处理它。

```glsl
InputData InitializeInputData(Varyings IN, half3 normalTS){
    InputData inputData = (InputData)0;
 
#if defined(REQUIRES_WORLD_SPACE_POS_INTERPOLATOR)
    inputData.positionWS = IN.positionWS;
#endif
                 
    half3 viewDirWS = SafeNormalize(IN.viewDirWS);
#ifdef _NORMALMAP
    float sgn = IN.tangentWS.w; // should be either +1 or -1
    float3 bitangent = sgn * cross(IN.normalWS.xyz, IN.tangentWS.xyz);
    inputData.normalWS = TransformTangentToWorld(normalTS, half3x3(IN.tangentWS.xyz, bitangent.xyz, IN.normalWS.xyz));
#else
    inputData.normalWS = IN.normalWS;
#endif
 
    inputData.normalWS = NormalizeNormalPerPixel(inputData.normalWS);
    inputData.viewDirectionWS = viewDirWS;
 
#if defined(REQUIRES_VERTEX_SHADOW_COORD_INTERPOLATOR)
    inputData.shadowCoord = IN.shadowCoord;
#elif defined(MAIN_LIGHT_CALCULATE_SHADOWS)
    inputData.shadowCoord = TransformWorldToShadowCoord(inputData.positionWS);
#else
    inputData.shadowCoord = float4(0, 0, 0, 0);
#endif
 
    inputData.fogCoord = IN.fogFactorAndVertexLight.x;
    inputData.vertexLighting = IN.fogFactorAndVertexLight.yzw;
    inputData.bakedGI = SAMPLE_GI(IN.lightmapUV, IN.vertexSH, inputData.normalWS);
    return inputData;
}
 
SurfaceData InitializeSurfaceData(Varyings IN){
    SurfaceData surfaceData = (SurfaceData)0;
    // Note, we can just use SurfaceData surfaceData; here and not set it.
    // However we then need to ensure all values in the struct are set before returning.
    // By casting 0 to SurfaceData, we automatically set all the contents to 0.
         
    half4 albedoAlpha = SampleAlbedoAlpha(IN.uv, TEXTURE2D_ARGS(_BaseMap, sampler_BaseMap));
    surfaceData.alpha = Alpha(albedoAlpha.a, _BaseColor, _Cutoff);
    surfaceData.albedo = albedoAlpha.rgb * _BaseColor.rgb * IN.color.rgb;
 
    // Not supporting the metallic/specular map or occlusion map
    // for an example of that see : https://github.com/Unity-Technologies/Graphics/blob/master/com.unity.render-pipelines.universal/Shaders/LitInput.hlsl
 
    surfaceData.smoothness = _Smoothness;
    surfaceData.normalTS = SampleNormal(IN.uv, TEXTURE2D_ARGS(_BumpMap, sampler_BumpMap), _BumpScale);
    surfaceData.emission = SampleEmission(IN.uv, _EmissionColor.rgb, TEXTURE2D_ARGS(_EmissionMap, sampler_EmissionMap));
    surfaceData.occlusion = 1;
    return surfaceData;
}
 
half4 frag(Varyings IN) : SV_Target {
    SurfaceData surfaceData = InitializeSurfaceData(IN);
    InputData inputData = InitializeInputData(IN, surfaceData.normalTS);
                 
    // In URP v10+ versions we could use this :
    // half4 color = UniversalFragmentPBR(inputData, surfaceData);
 
    // But for other versions, we need to use this instead.
    // We could also avoid using the SurfaceData struct completely, but it helps to organise things.
    half4 color = UniversalFragmentPBR(inputData, surfaceData.albedo, surfaceData.metallic, 
      surfaceData.specular, surfaceData.smoothness, surfaceData.occlusion, 
      surfaceData.emission, surfaceData.alpha);
                 
    color.rgb = MixFog(color.rgb, inputData.fogCoord);
 
    // color.a = OutputAlpha(color.a);
    // Not sure if this is important really. It's implemented as :
    // saturate(outputAlpha + _DrawObjectPassData.a);
    // Where _DrawObjectPassData.a is 1 for opaque objects and 0 for alpha blended.
    // But it was added in URP v8, and versions before just didn't have it.
    // And I'm writing thing for v7.3.1 currently
    // We could still saturate the alpha to ensure it doesn't go outside the 0-1 range though :
    color.a = saturate(color.a);
 
    return color;
}
```

当前，虽然我们的着色器可以接收阴影，但它不包含ShadowCaster传递，因此不会投射任何阴影。这将在下一部分中处理。

## ShadowCaster & DepthOnly Passes

### SHADOWCASTER

如果我们希望着色器投射阴影，则需要通过标签“ **LightMode**” =“ **ShadowCaster**”的传递。可以在“Unlit”和“Lit”着色器上进行此操作，但要注意，尽管它们会投射阴影，但如果您不在**UniversalForward**Pass中处理阴影，它们将不会接收阴影。

除了使用**UsePass**时（在Shaderlab部分中已讨论过）。尽管我们可以使用其他着色器中的阴影投射器，例如UsePass“Universal Render Pipeline/Lit/ShadowCaster”，但由于该着色器中使用的CBUFFER可能不同，因此SRP Batcher兼容性可能会丢失。

相反，您应该自己定义这些Pass，有一个取巧的解决方法，我们可以执行以下操作：

```glsl
Pass {
    Name "ShadowCaster"
    Tags { "LightMode"="ShadowCaster" }
 
    ZWrite On
    ZTest LEqual
 
    HLSLPROGRAM
    // Required to compile gles 2.0 with standard srp library
    #pragma prefer_hlslcc gles
    #pragma exclude_renderers d3d11_9x gles
    //#pragma target 4.5
 
    // Material Keywords
    #pragma shader_feature _ALPHATEST_ON
    #pragma shader_feature _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A
 
    // GPU Instancing
    #pragma multi_compile_instancing
    #pragma multi_compile _ DOTS_INSTANCING_ON
             
    #pragma vertex ShadowPassVertex
    #pragma fragment ShadowPassFragment
     
    #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/CommonMaterial.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/Shaders/ShadowCasterPass.hlsl"
 
    ENDHLSL
}
```

我们使用了**ShadowCasterPass.hlsl**中的函数，意味着定义该Pass较为容易，但是它需要使用\*\*\_BaseMap\*\*，\*\*\_BaseColor\*\*和\*\*\_Cutoff\*\*属性，我们也需要将它们添加到\*\*UnityPerMaterial CBUFFER\*\*中。阴影投射器中的fragment函数仅在需要阴影的位置返回0，并丢弃不应有阴影的像素（请注意，仅在启用了\*\*\_ALPHATEST\_ON\*\*关键字的情况下才会发生裁剪）

如果我们的常规着色器通道也进行顶点位移，则也需要将其添加到ShadowCaster通道中，以便正确投射位移的阴影。为了解决这个问题，我们要么将ShadowCasterPass的内容复制到我们的过程中，要么只是定义一个新的顶点函数并交换#pragma顶点ShadowPassVertex。例如 ：

```glsl
#pragma vertex vert
 
...
 
// function copied from ShadowCasterPass and edited slightly.
Varyings vert(Attributes input) {
    Varyings output;
    UNITY_SETUP_INSTANCE_ID(input);
 
    // Example Displacement
    input.positionOS += float4(0, _SinTime.y, 0, 0);
 
    output.uv = TRANSFORM_TEX(input.texcoord, _BaseMap);
    output.positionCS = GetShadowPositionHClip(input);
    return output;
}
```

### DEPTHONLY

着色器还应包含标记为“ **LightMode**” =“ **DepthOnly**”的过程。此过程与**ShadowCaster**非常相似，但没有阴影偏差偏移。我不完全确定URP中使用**DepthOnly**传递的用途。场景视图似乎在渲染深度纹理时使用了它（由ShaderGraph中的“Scene Depth”节点使用），而“游戏视图”深度纹理在没有此传递的情况下似乎可以正常工作。但是，可能还有其他一些东西，例如自定义渲染功能（用于前向渲染器）依赖于DepthOnly传递。

我们可以以类似的方式处理DepthOnly传递，但有一些细微差异：

```glsl
Pass {
    Name "DepthOnly"
    Tags { "LightMode"="DepthOnly" }
 
    ZWrite On
    ColorMask 0
 
    HLSLPROGRAM
    // Required to compile gles 2.0 with standard srp library
    #pragma prefer_hlslcc gles
    #pragma exclude_renderers d3d11_9x gles
    //#pragma target 4.5
 
    // Material Keywords
    #pragma shader_feature _ALPHATEST_ON
    #pragma shader_feature _SMOOTHNESS_TEXTURE_ALBEDO_CHANNEL_A
 
    // GPU Instancing
    #pragma multi_compile_instancing
    #pragma multi_compile _ DOTS_INSTANCING_ON
             
    #pragma vertex DepthOnlyVertex
    #pragma fragment DepthOnlyFragment
             
    #include "Packages/com.unity.render-pipelines.core/ShaderLibrary/CommonMaterial.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/SurfaceInput.hlsl"
    #include "Packages/com.unity.render-pipelines.universal/Shaders/DepthOnlyPass.hlsl"
 
    // Again, using this means we also need _BaseMap, _BaseColor and _Cutoff shader properties
    // Also including them in cbuffer, except _BaseMap as it's a texture.
 
    ENDHLSL
}
```

这次使用Unity的URP着色器提供的**DepthOnlyPass** 。同样，如果需要顶点位移，我们应该将**DepthOnlyVertex**函数复制到我们的代码中，将其重命名为vert，然后像上面的**ShadowCaster**示例中一样添加位移代码。

## 引用
- https://fungusfox.gitee.io/p/%E4%BB%8Ebuilt-in%E5%88%B0urp/