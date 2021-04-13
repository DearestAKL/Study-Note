## Mask 与 Rect Mask2D

### 主要区别

 > #### 区别一
  >
  > - Mask主要处理不规则图形遮罩效果
  > - RectMask2D只能做矩形遮罩

>  #### 区别二
>
>   - Mask需要一个Image来当作遮罩区域,子节点在Image[渲染区域]才会显示
>   - RectMask2D以自身RectTransform为裁剪区域,子节点在[RectTransform区域]内显示
>

### Rect Mask2D

​	RectMask2D 是一个类似遮罩(Mask)控件的遮罩控件。遮罩将子元素限制未父元素的矩形。与标准遮罩控件不同，这个控件有一些限制，但也有许多性能优势。

##### 描述

局限性：

- 仅在2D空间中有效
- 不能正确覆盖不共面的元素

优势：

- 不使用模板缓冲区
- 无需而外绘制调用
- 无需更改材质
- 高速性能

> 如果只用固定矩形遮罩,不需要特殊形状遮罩的情况下,可以优先使用RectMask2D

### Mrask 与 RectMask2D 合批情况

#### Mrask 

1. **多个Mask之间可以进行合批(头和头合批, 子对象和子对象合批, 尾和尾合批),需要同渲染层级(depth), 同材质, 同图集**
2. **Mask内外不能进行合批**

#### RectMask2D 

1. **RectMask2D本身不产生drawcall**
2. **不同RectMask2D的子对象不能合批**