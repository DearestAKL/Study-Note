### Queue

在3D引擎中，我们经常要对 透明和不透明物体进行排序。先渲染不透明再渲染透明物体，Unity使用 **Queue**标签，分别放入不同的渲染队列中。

1. **Background** - 背景，一般天空盒之类的使用这个标签，最早被渲染
2. **Genmetry(default)** - 适用于大部分不透明的物体
3. **AlphaTest** - 如果Shader要使用AlphaTest功能 使用这个队列性能更高
4. **Transparent** - 这个渲染队列在AlphaTest之后,Shader中有用到Alpha Blend的，或者深入不写入的都应该在这个队列。
5. **Overlay** - 最后的渲染队列，全屏幕后期使用这个进行处理

### RenderType

全局替换渲染

1. **Opaque** - 用于大多数着色器(法线着色器、自发光着色器、反射着色器以及地形着色器)。
2. **Transparent** - 用于半透明着色器(透明着色器、粒子着色器、字体着色器、地形额外通道着色器)。
3. **TransparentCutout** - 蒙皮透明着色器（Transparent Cutout，两个通道的植被着色器）。
4. **Breakground** - Skybox shaders. 天空盒着色器。
5. **Overlay** - GUITexture,Halo,Flare shaders. 光晕着色器、闪光着色器。
6. **TreeOpaque** - Terrain engine tree bark. 地形引擎中的树皮。
7. **TreeTransparentCutout** - terrain engine tree leaves. 地形引擎中的树叶。
8. **TreeBillboard** - terrain engine billboarded trees. 地形引擎中的广告牌树。
9. **Grass** - terrain engine grass. 地形引擎中的草。
10. **GrassBillboard** - terrain engine billboarded grass. 地形引擎何中的广告牌草。

### **DisableBatching**

当进行一些批处理操作时，有可能会导致物体的模型空间（Object Space）中的信息丢失，使用此项可保存相关信息

1. **True** - 针对此Shader总是使批处理操作无效
2. **False** - 不取消批处理操作（默认选项）
3. **LODFading** - 仅在LOD减弱时禁用批处理，多用于树木ForceNoShadowCasting