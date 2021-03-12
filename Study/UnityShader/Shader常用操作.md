### TRANSFORM_TEX 与 SAMPLE_TEXTURE2D

​	TRANSFORM_TEX主要作用是拿顶点的uv去和材质球的tiling和offset做运算，确保材质球里的缩放和偏移设置是正确的

> ```c
> 	TRANSFORM_TEX(i.texcoord,_MainTex)
> ```

​	SAMPLE_TEXTURE2D 进行纹理取样

> ```c
>     TEXTURE2D(_MainTex);
>     SAMPLER(sampler_MainTex);
>     ...
>     SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.texcoord)
> ```

​	需要定义_MainTex_ST 才能对 _MainTex进行纹理取样，其中ST的意思就是Sampler Texture的意思，且 _MainTex_ST.xy中是tiling， _MainTex_ST.zw中是offset。

​	如果Tiling 和Offset你留的是默认值，即Tiling为（1，1） Offset为（0，0）的时候，可以不用

> o.uv = TRANSFORM_TEX(v.texcoord,_MainTex);

​	换成

> o.uv = i.texcoord.xy;

​	也是能正常显示的；i相当于Tiling 为（1，1）Offset为（0，0），但是自己填的Tiling值和Offset值就不起作用了