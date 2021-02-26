**如果ILRuntime使用时性能有问题，需要确认以下几点：**

1.发包前是否做了自动分析CLR绑定

2.是否注册了值类型绑定

3.热更DLL是否为release模式编译

4.发包前是否关闭了Development build

5.编辑器下是否定义了DISABLE_ILRUNTIME_DEBUG

**注意事项**

1.一般嵌套task或者foreach里多次await容易出现代码执行不到