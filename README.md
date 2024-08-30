
# .NET 8 Moq mock GetRequiredKeyedService Setup报错


项目代码里有地方用到`IServiceProvider.GetRequiredKeyedService`来解析服务，在写单元测试时需要Mock它，本以为像下面这样写就可以了：



```


|  | var serviceProvider = new Mock<IServiceProvider>(); |
| --- | --- |
|  | serviceProvider.Setup(x => x.GetRequiredKeyedService<AAA>(It.IsAny<BBB>())).Returns(new CCC()); |


```

没想到报错了:



```


|  | Test method threw exception: |
| --- | --- |
|  | System.NotSupportedException: Unsupported expression: x => x.GetRequiredKeyedService(It.IsAny<Type>(), It.IsAny()) |
|  | Extension methods (here: ServiceProviderKeyedServiceExtensions.GetRequiredKeyedService) may not be used in setup / verification expressions. |
|  |  |
|  | Stack Trace: |
|  | Guard.IsOverridable(MethodInfo method, Expression expression) line 87 |
|  | MethodExpectation.ctor(LambdaExpression expression, MethodInfo method, IReadOnlyList`1 arguments, Boolean exactGenericTypeArguments, Boolean skipMatcherInitialization, Boolean allowNonOverridable) line 236 |
|  | ExpressionExtensions.g__Split|5_0(Expression e, Expression& r, MethodExpectation& p, Boolean assignment, Boolean allowNonOverridableLastProperty) line 256 |
|  | ExpressionExtensions.Split(LambdaExpression expression, Boolean allowNonOverridableLastProperty) line 170 |
|  | Mock.SetupRecursive[TSetup](Mock mock, LambdaExpression expression, Func`4 setupLast, Boolean allowNonOverridableLastProperty) line 728 |
|  | Mock.Setup(Mock mock, LambdaExpression expression, Condition condition) line 562 |
|  | Mock`1.Setup[TResult](Expression`1 expression) line 645 |


```

有点奇怪，难道`GetRequiredKeyedService`不是接口方法？查看.NET源代码，果然，`GetRequiredKeyedService`是IServiceProvider的扩展方法，而我们知道Moq是不支持Setup扩展方法的。



```


|  | /// |
| --- | --- |
|  | /// Get service of type  from the . |
|  | /// |
|  | /// The type of service object to get. |
|  | /// The  to retrieve the service object from. |
|  | /// An object that specifies the key of service object to get. |
|  | /// A service object of type . |
|  | /// There is no service of type . |
|  | public static T GetRequiredKeyedService<T>(this IServiceProvider provider, object? serviceKey) where T : notnull |
|  | { |
|  | ThrowHelper.ThrowIfNull(provider); |
|  |  |
|  | return (T)provider.GetRequiredKeyedService(typeof(T), serviceKey); |
|  | } |
|  |  |


```

原因找到就好办了，翻看源码，一步步找到IServiceProvider.GetRequiredKeyedService最终调用的接口方法，然后再mock即可。


首先看下requiredServiceSupportingProvider.GetRequiredKeyedService(serviceType, serviceKey)调的是什么方法



```


|  | /// |
| --- | --- |
|  | /// IKeyedServiceProvider is a service provider that can be used to retrieve services using a key in addition |
|  | /// to a type. |
|  | /// |
|  | public interface IKeyedServiceProvider : IServiceProvider |
|  | { |
|  | /// |
|  | /// Gets the service object of the specified type. |
|  | /// |
|  | /// An object that specifies the type of service object to get. |
|  | /// An object that specifies the key of service object to get. |
|  | ///  A service object of type serviceType. -or- null if there is no service object of type serviceType. |
|  | object? GetKeyedService(Type serviceType, object? serviceKey); |
|  |  |
|  | /// |
|  | /// Gets service of type  from the  implementing |
|  | /// this interface. |
|  | /// |
|  | /// An object that specifies the type of service object to get. |
|  | /// The  of the service. |
|  | /// A service object of type . |
|  | /// Throws an exception if the  cannot create the object. |
|  | object GetRequiredKeyedService(Type serviceType, object? serviceKey); |
|  | } |
|  |  |


```

可以看到IKeyedServiceProvider也是继承了IServiceProvider接口，这就更好办了，我们直接Mock IKeyedServiceProvider再Setup即可，将用到IServiceProvider的地方，换成IKeyedServiceProvider。


代码如下：



```


|  | var serviceProvider = new Mock<IKeyedServiceProvider>(); |
| --- | --- |
|  | serviceProvider.Setup(x => x.GetRequiredKeyedService(It.IsAny<AAA>(), It.IsAny<BBB>())).Returns(new CCC()); |


```

运行测试，完美。


# 总结


解决这个问题并不困难，但是如果.Net不开源，看不到源代码，还是有点头疼。


 本博客参考[豆荚加速器](https://yirou.org)。转载请注明出处！
