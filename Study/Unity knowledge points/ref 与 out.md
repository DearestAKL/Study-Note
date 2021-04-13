## ref 与 out

​	CLR允许以传引用而非传值的方式传递参数。C#用关键字out或ref支持这个功能。两个关键字都告诉C#编译器生成的元数据是指明该参数的引用的。编译器将生成代码来传递参数的地址，而非传递参数本身。

​	CLR不区分out 和 ref，意味着无论用哪个关键字，都会生成相同的IL代码。另外，元数据机会完全一致，只有一个bit除外，它用于记录声明方法时指定的是out还是ref。但C#编译器是将这两个关键字区别对待的，而且这个区别决定了由哪个方法负责初始化所引用的对象。如果方法的参数用out来标记，表明不指望调用者再调用方法之前初始化好了对象。被调用的方法不能读取参数的值，而且在返回前必须向这个值写入。相反，如果方法用ref来标记，调用者就必须在调用该方法前初始化参数的值，被调用的方法可以读取值以及或者向值写入。

```c#
public sealed class Program
{
	public static void Main()
	{
		int x;					// x没有初始化 
		GetVal(out x);			 // x不必初始化
     	Console.WriteLine(x);	  // 显示"10"		
	}
	
    private static void GetVal(out int v)
    {
        v = 10;// 该方法初始化v
    }
}
```

```c#
public sealed class Program
{
	public static void Main()
	{
		int x = 5;					// x已经初始化 
		GetVal(ref x);				// x必须初始化
     	Console.WriteLine(x);		 // 显示"15"		
	}
	
    private static void GetVal(out int v)
    {
        v += 10;// 该方法可使用v已初始化的值
    }
}
```

