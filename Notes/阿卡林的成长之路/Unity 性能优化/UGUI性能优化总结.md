---
title: UGUI性能优化总结
date: 2021-06-15 11:00:00
tags: UGUI,性能优化
description: UGUI性能优化总结。转载于无境阿花的博客，末尾有链接。
cover: https://cdn.jsdelivr.net/gh/jerryc127/CDN@latest/cover/default_bg.png
---
# UGUI性能优化总结
## UGUI基础
### Canvas
Canvas是一个Native层实现的Unity组件，被Unity渲染系统用于渲染分层几何体（layered geometry）在游戏世界空间中。 Canvas负责把它们包含的Mesh合批，生成合适的渲染命令发送给Unity图形系统。以上行为都是在Native C++代码中完成，我们称之为**Rebatch**或者**Batch Build**，当一个Canvas中包含的几何体需要Rebacth时，这个Canvas就会被标记为Dirty状态。

### Canvas Render
几何体数据是通过Canvas Renderer组件被提交到Canvas中。

### 子Canvas
  Canvas组件可以嵌套在另一个Canvas组件下，我们称为子Canvas，子Canvas可以把它的子物体与父Canvas分离，使得当子Canvas被标记为Dirty时，并不会强制让父Canvas也强制Rebuild，反之亦然。但在某些特殊情况下，使用子Canvas进行分离的方法可能会失效，例如当对父Canvas的更改导致子Canvas的大小发生变化时。
  
  ### Graphic
  Canvas组件可以嵌套在另一个Canvas组件下，我们称为子Canvas，子Canvas可以把它的子物体与父Canvas分离，使得当子Canvas被标记为Dirty时，并不会强制让父Canvas也强制Rebuild，反之亦然。但在某些特殊情况下，使用子Canvas进行分离的方法可能会失效，例如当对父Canvas的更改导致子Canvas的大小发生变化时。
    
  ### Layout 
  Layout控制着RectTransform的大小和位置，通常用于创建复杂的布局，这些布局需要对其内容进行相对大小调整或相对位置调整。Layout仅依赖于RectTransforms，并且仅影响其关联RectTransforms的属性。这些Layout类不依赖于Graphic类，可以独立于UGUI的Graphic类之外使用。 
  
   ### CanvasUpdateRegistry
   CanvasUpdateRegistry,Graphic和Layout都依赖于CanvasUpdateRegistry类，它没有暴露在Unity编辑器的界面上。这个类追踪那些必须更新的Layout类和Graphic类，并在其关联的Canvas调用willRenderCanvases事件时根据需要触发更新。
   
   ### Rebuild 
   Layout和Graphic的更新称为Rebuild。本文档后面将进一步详细讨论Rebuild过程。 
   
   ### 渲染细节 
   Canvas中所有几何体的绘制都在一个透明队列中，这就意味着由UGUI制作的几何体始终开启Alpha Blend，所以从多边形栅格化的每个像素都将被采样，即使它是完全不透明的。在移动设备上，过高的Overdraw会急剧升高填充率（Fill Rate）以超过GPU的可承受能力。 
   
   ### Batch Build过程（Canvases）
   -  Batch Build（Rebatch）：Canvas把表示它UI元素的网格合并起来，并生成合适的渲染命令发送到Unity的图形管线中。这个过程的结果会被缓存起来复用，直到这个Canvas被标记为Dirty，当Canvas中任何一个网格发生变化时，就会被标记成Dirty状态。 
   - Canvas的网格是从从那些Canvas下的CanvasRenderer组件中获取的，但不包括子Canvas。 
   - 批处理计算需要对网格进行深度排序，并检查网格是否重叠，材质是否共享等。这种操作是多线程的，因此它的性能在不同的CPU架构中通常会有很大的不同，尤其是在移动soc（通常很少有CPU核心）和现代桌面CPU（通常有4个或更多核心）之间。  
   
   ### Rebuild过程（Graphics） 
   >- Rebuild：指重新计算Graphic的布局和网格的过程，这个过程在CanvasUpdateRegistry中执行（这是一个C#类，我们可以在Unity的Bitbucket上找到源码）。 
  >- 在CanvasUpdateRegistry中，最重要的方法是PerformUpdate。每当Canvas组件调用WillRenderCanvases事件时，就会调用此方法。此事件每帧调用一次。 
   
   #### PerformUpdate执行流程 
   - 被标记为Dirty状态的Layout通过ICanvasElement.Rebuild方法请求重建布局。
   - 所有裁剪组件（例如Mask），对需要被裁剪的组件进行剔除。这在ClippingRegistry.Cull中执行。 
   - 被标记了Dirty状态的Graphic请求重建他们的图形元素。 
    
   对于Layout和Graphic的重建，过程分为多个部分。布局重建分为三个部分（预布局、布局和后期布局），而图形重建分为两个部分（预渲染和后期预渲染）。
   
   #### Layout的Rebuild 
   要重新计算包含一个或多个Layout的适当位置和大小，必须以适当的层级顺序进行布局计算，因为在GameObject在Hierarchy中靠近根的Layout，很可能会影响嵌套其中的子物体的Layout的位置和大小，所以必须优先计算。
   
   为了做到这一点，UGUI把被标记为Dirty状态的Layout按照它们在Hierarchy中的层级对它们进行排序。Hierarchy中较高的项（即父变换较少的项）将移动到列表的前面。 
    
   然后这些排序后的Layout列表会请求重建它们的布局，这过程是那些UI元素被Layout真正改变位置和大小的过程。 
    
   #### Graphic的Rebuild 
   当Graphic进行Rebuild时，UGUI将控制权转交给ICanvasElement接口的Rebuild方法。Graphic类实现了这个接口，并在Rebuild过程的预渲染阶段执行两个不同的重建步骤。
   - 如果顶点数据已标记为Dirty状态（如组件的矩形变换更改大小时），则重建网格
   - 如果材质数据已标记为Dirty状态（如组件的材质或纹理发生改变时），则将更新附着的画布渲染器的材质。  
   
   Graphic的Rebuild不需要按特定顺序遍历Graphic组件列表，也不需要任何排序操作。
## Rebatch和Rebuild的触发条件
### 触发Rebatch的条件
  当Canvas下有Mesh发生改变时，如： 
  
  >1.  SetActive 
  >
  >2.  Transform属性变化
  >
  >3.  Graphic的Color属性变化（改Mesh顶点色） 
  >
  >4.  Text文本内容变化 
  >
  >5.  Depth发生变化

### 触发Rebuild的条件
  >1.  Layout修改RectTransform部分影响布局的属性
  >
  >2.  Graphic的Mesh或Material发生变化
  >
  >3.  Mask裁剪内容变化 

**Rebuild通常引起Rebatch**

## 主要关注的性能指标
  >1.  Canvas.BuildBatch函数耗时
  >
  >2.  Canvas.SendWillRenderCanvases函数耗时
  >
  >3.  Draw Call
  >
  >4.  Overdraw
  >
  >5.  堆内存

## 动静分离（Canvas.BuildBatch函数耗时优化） 
基于Rebatch是以Canvas为单位，当Canvas下UI元素发生变化时，会引起整个Canvas的重构，其中会包括网格合并，网格重叠检测，层级排序等操作。对于同一个界面，我们可以再细分Canvas，把相对静态的、不会变动的UI放在一个Canvas里，而相对变化比较频繁的UI就放在另一个Canvas里，使得频繁变化的Canvas里只对自己的Canvas下的元素进行Rebatch，而节省掉另一个Canvas中不需要变化的元素的Rebatch计算。 

只有同一个Canvas下的UI元素才有可能合批，在中间新增Canvas会打断合批，动静分离优化本质是DrawCall换重构耗时的权衡。 

Rebatch是在Canvas.BuildBatch函数中进行，而在Unity 5.2版本后，已经对Canvas.BuildBatch做了优化，优化后使用子线程进行计算，已经很大程度缓解了主线程的压力，目前来说动静分离并没有那么需要关注了。

## 合批规则(Draw Call)
  UGUI的合批除了要求相同贴图和材质外，还对Depth和Hierarchy的层级等有要求。规则比较复杂繁琐，且源码不放开，实际情况很难做到完全把控。 
  
  可以借助FrameDebugger实时调试合批情况。
  
  ### 合批范围
  处于同一个Canvas下的（不包括子Canvas）UI元素才能进行合批，同时排除掉active=false，CanvasGroup.alpha=0的元素不参与。
  
  ### Depth计算
  - 染的UI元素Depth为-1，UI下没有和其他UI相交时，该UI的Depth为0。 
  - 当前UI下面有一个UI与其相交，若两者贴图和材质相同时，它们Depth相同，否则上面的UI的Depth是下面UI的Depth+1。 
  - 前UI下面与多个UI相交，假设处于下面的多个UI中，最高Depth是MaxDepth，如果下面这些UI的Depth全是MaxDepth，则此UI不能与下面的UI合批，如果只有一个是MaxDepth，且贴图材质相同，则可以合批。

这里的相交是用Mesh计算相交，不是指RectTransform大小相交。

### 合批过程  
>1.  Depth算完后，依次根据Depth、material ID、texture ID、RendererOrder（即UI层级队列顺序，HierarchyOrder）排序（条件的优先级依次递减），剔除depth == -1的UI元素，得到Batch前的UI元素队列VisiableList。 
>
>2. 对VisiableList中相邻且可以Batch（相同material和texture）的UI元素合并批次，然后再生成相应mesh数据进行绘制。

### 注意
> - 节点Position.z不为0时，视为3D UI，其子物体全部不参与合批，且打断前后合批。



## 图集
不同的图集/材质会产生DrawCall，通过图集把多个图片合并到一张贴图里，让这些图片可以进行合批，以此减少DrawCall。

打图集规则：
  >1.  同个界面、同个功能的图在同一个图集
  >
  >2.  不要把重复的图打到不同的图集中
  >
  >3.  通用的，使用频率高的图，单独放在一个图集
  >
  >4.  尺寸过大的图尽量不打进图集
  
对于类似背包界面这种包含多个系统功能道具图标这类界面，可以考虑使用动态图集

## Overdraw
Overdraw是指一帧当中，同一个像素被重复绘制的次数。Fill Rate(填充率)是指显卡每帧每秒能够渲染的像素数。在每帧绘制中，如果一个像素被反复绘制的次数越多，那么它占用的资源也必然更多。Overdraw与Fill Rate成正比，目前在移动设备上，FillRate的压力主要来自半透明物体。因为多数情况下，半透明物体需要开启 Alpha Blend 且关闭 ZTest和 ZWrite，同时如果我们绘制像 alpha=0 这种实际上不会产生效果的颜色上去，也同样有 Blend 操作，这是一种极大的浪费。

  >1.  减少UI重叠层级，隐藏处于底下被完全覆盖的UI面板。
  >
  >2. 对于需要暂时隐藏的UI，不要直接把Color属性的Alpha值改为0，UGUI中这样设置后仍然会渲染，应该用CanvasGroup组件把Alpha值置零。
  >
  >3.  需要响应Raycast事件时，不要使用空Image，可以自定义组件继承自MaskableGraphic，重写OnPopulateMesh把网格清空，这样可以响应Raycast而又不需要绘制Mesh。
  >
  >4. 打开全屏界面，关闭场景摄像机。对于一些非全屏但覆盖率较高的界面，在对场景动态表现要求不高的情况下，可以记录下打开UI时的画面，作为UI背景，然后关掉场景摄像机。
  >
  >5. 裁掉无用区域，镂空，对于Sliced类型的Image可以看情况取消Fill Center。
  >
  >6.  尺寸过大的图尽量不打进图集
  >
  >7.  尽量保持UI上的粒子特效简单，尽量使用序列帧实现。

Unity在Scene视图可以切换成Overdraw模式，观察填充率情况，根据UWA的提示，Editor下看到的不完全准确，需要真机测试。

## Raycast
UGUI创建的所有可点击组件都会默认开启RaycastTarget，当进行点击操作时，会对所有开启RaycastTarget的组件进行遍历检测和排序，实际上大部分的组件是不需要响应点击事件的，对于这些组件我们应该取消RaycastTarget属性，最好的方式是监听组件创建，在创建时直接赋值为False，对于需要响应事件的组件再手动开启。 
更多优化可以参考一下文章：
- [Unity UGUI优化：解决EventSystem耗时过长的问题 第一部分](https://blog.csdn.net/cyf649669121/article/details/83661023)
- [Unity UGUI优化：解决EventSystem耗时过长的问题 第二部分](https://blog.csdn.net/cyf649669121/article/details/83785539)
- [Unity UGUI优化：解决EventSystem耗时过长的问题 第三部分](https://blog.csdn.net/cyf649669121/article/details/86484168)
- [Unity UGUI 点击性能优化](https://zhuanlan.zhihu.com/p/55566751)

## Mask与RectMask2D

### Mask  
> - 裁剪依赖Image组件，可根据Image裁剪不规则形状。 
> 
> - 增加两个DrawCall 
> 
> - 增加两层Overdraw 
> 
> - Mask与内UI不能与外面的UI合批，但不同Mask内部的多个UI可以进行合批，但当Mask之间有相交时，会打断合批。


### RectMask2D
> - 裁剪范围按照RectTransform的Rect大小，不依赖Image，不能裁剪不规则形状。
>  
> - 不增加DrawCall 
> 
> - 不增加Overdraw 
> 
> - RectMask2D内UI不能与外面UI合批且多个RectMask2D之间也不能合批。 
> 
> - RectMask2D在SendWillRenderCanvases函数中每帧有计算裁剪区域的操作（非重建操作），当其里面的UI元素很多时，耗时也值得注意。参考UWA问答-RectMask2D 是不是会频繁触发 SendWillRenderCanvases。

### 小结
一般情况下，对于规则形状裁剪就可以考虑使用RectMask2D代替Mask，在DrawCall和Overdraw上有直接优势。但结合一个界面多个遮罩共存，遮罩区域是否相交、遮罩间内容是否能合批、遮罩内UI元素数量等，还需要进一步权衡。

## 其他
- 慎用自带组件Outlien和Shaow，都是通过重复绘制多个Mesh实现的，其中Showdow绘制为原文本Mesh的2倍，而Outline为5倍，对渲染面数、顶点数，BuildBatch和SendWillRenderCanvases的耗时，Overdraw都有影响。若对于某种字体每次出现都需要这两种效果，可以让美术同学直接把阴影和描边做到字体里。
- 隐藏界面时，可用CanvasGroup.Alpha=0，或者从Camera渲染层级里移除并将Canvas设置enable=false等方法隐藏，代替SetActive。 
- Test组件的Best Fit属性若非必要不要使用，它会使字号随着文本框大小而自动适配，一方面是适配本身在调整文本框大小时有CPU耗时开销，另一方面每个字号下新生成的字都会在Font Texture上占用一个字的大小，容易导致Font Texture过大（这个类似图集，当Font Texture当前大小放不下时才会占用更多内存）。
-  UI的图片一般关闭Read/Write和MipMaps。 尽量少使用Layout组件，会增加Canvas.SendWillRenderCanvases函数耗时，利用好RectTransform同样可实现简单布局。 
-  对于血条、飘字、小地图标记等频繁更新位置的UI，可尽量减低更新频率，如隔帧更新，并设定更新阈值，当位移大于一定数值时再赋值（一方面是位移小时可能表现上看不出位移，另一方面是就算是没有实际位移，重复赋相同的值，也会SetDirty触发重建），可减少BuildBatch耗时。 
-  界面上的特效不要直接挂在在UI的Prefab上，而是用动态加载的方式，有助于减小UI的Prefab体积以及加载时间。

## 总结
以上是UGUI较为常见而又不用大动干戈的优化方案，参考了不少资料，不得不说一下目前对于UGUI的优化网上存在大量过期教程和部分问题存在误导的文章，目前UGUI的C#源码已开源，建议大家还是多读源码，多动手多试验，实践是检验真理的唯一标准。

## 参考

- [Unity官方UGUI优化文档](https://learn.unity.com/tutorial/optimizing-unity-ui)

- [Unity UI优化小结](https://zhuanlan.zhihu.com/p/43111806)

- [UGUI深入理解--性能优化总结](https://zhuanlan.zhihu.com/p/343978391)

- [Unity3D ：关于UGUI的网格重建、动静分离](https://blog.csdn.net/cyf649669121/article/details/83142903#commentBox)

- [Unity UGUi优化分享及注意事项](https://zhuanlan.zhihu.com/p/98359112)

## 转载
- 原文:http://www.drflower.top/292/.html