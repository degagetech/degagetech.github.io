---
layout: post
title: .NET Emit 实现动态 AOP 技术
subtitle: 动态 “附加” 特性，运行时方法拦截
comments: true
---


本文描述 .NET 平台下实现 AOP 的方式之一，运行时动态拦截，相比于编译时织入，性能低了一些，但是使用时更为灵活，编译时织入后续我会再写一篇博文阐述其原理。

基础知识不在啰嗦，请阅读我另一篇博文： [.NET 使用 Emit 构建接口的动态代理](http://blog.degage.tech/2018-05-23-emit-dynamic-proxy/)  。



从此文所描述的情景有两个目的：

**一.为现有接口定义附加 WCF 契约特性，以使得其可以作为契约被 WCF 框架使用。**

**二.对原有服务类的各个方法进行拦截（WCF 本身有类似机制，本文更便重于实现原理），并将控制权转移到指定类型代码中。**

****

有如下接口定义，无附加 WCF 契约特性：

```c#
   public interface IDuplexChat
    {
        Boolean Login(String userAccount,String password);
        Boolean SayHello(String name);
        DataTable GetData(String tableName);
    }
```

其对应服务端实现类如下：

```C#
  public class DuplexChatService : IDuplexChat
    {
        public DataTable GetData(String tableName)
        {
            DataTable table = new DataTable("UserTable");
            //...
         
            return table;
        }

        public Boolean Login(String userAccount, String password)
        {
            return true;
        }

        public Boolean SayHello(String name)
        {
            return false;
        }
    }
```

### 实现目的一

首先我们实现第一个目的 “附加” WCF 契约特性，注意 “附加” 一直都是打引号的。

在  .NET 类型体系里面，一个类型的无法被修改，其所有信息在被编译后都被存储到元数据中，我们能做的是，从这个类型中继承一个新的类型出来，并在定义这个新类型时，添加上我们的需求。

第一步在动态程序集中（更准确的说是在程序集的模块中）定义我们的继承接口类型， 程序集和模块的定义不在赘述。

```c#
  TypeBuilder typeBuilder = ContractWrapper.ModuleBuilder.DefineType(
             typeName,
             TypeAttributes.Abstract | TypeAttributes.Public | TypeAttributes.Interface,
            null, new Type[] { proxiedType }); //proxiedType 为被继承的接口的类型
```

这里需要注意的是  *[TypeAttributes](https://referencesource.microsoft.com/#mscorlib/system/reflection/typeattributes.cs,6ed31f1fb0ecdf25)*  参数的传入，抽象、公开、接口。

为了给我们的类型附加指定的特性信息，这里我们要获取特性类型的构造函数元数据信息，例如获取服务契约特性的：

```C#
  var servicContractType = typeof(ServiceContractAttribute);
  //此处使用默认无参的构造函数
  var serviceContractCtor = servicContractType.GetConstructor(new Type[] { });
```

然后通过此元数据在构造一个 *[CustomAttributeBuilder](https://referencesource.microsoft.com/#q=CustomAttributeBuilder)* 对象，如下：

```C#
 CustomAttributeBuilder serviceContractAttribuyeBuilder = new    
                                     CustomAttributeBuilder(
                                                 serviceContractCtor,                                                                      new Object[] { });
```

然后将其通过 *SetCustomAttribute* 附加到类型上：

```C#
 typeBuilder.SetCustomAttribute(serviceContractAttribuyeBuilder);
```

*TypeBuilder、MethodBuilder、ParameterBuilder*  均有提供此方法，但是它们并没有共同继承某类或接口。

在我们的例子中，需要为类型附加  *ServiceContractAttribute*，为其方法附加  *OperationContractAttribute* ，为对应方法的参数附加  *MessageParameterAttribute*  。

至此，新的类型接口算是生成完毕了，现在我们通过 WCF 代理工厂来创建通信代理：

```c#
//通过我们的类型包装器获取动态生成的新类型，其继承自原来的接口类型
var contractType = ContractWrapper<IDuplexChat>.Type;
ContractDescription description = ContractDescription.GetContract(contractType);
ServiceEndpoint endpoint = new ServiceEndpoint(description);

//在使用时获取通道对象
Type specificType = typeof(ChannelFactory<>).MakeGenericType(new Type[] { ContractWrapper<IDuplexChat>.WrappedType });
var factory = Activator.CreateInstance(specificType, new Object[] { endpoint });
var method = factory.GetType().GetMethod("CreateChannel", new Type[] { });
Object proxy= method.Invoke(factory, null);
```

但是在使用代理对象时请注意，假如如此使用：

```C#
 IDuplexChat proxy = ContractWrapper<IDuplexChat>.CreateProxy(endpoint);
 proxy.Login("JohnWang", "123");
```

得到的只能是错误：

```
未经处理的异常:  System.NotSupportedException: 方法 Login 在此代理中不受支持，如果未使用 OperationContractAttribute 标记方法或未使用 ServiceContractAttribute 标记接口类型，则会出现此情况。
```

这是因为我们动态创建的接口类型，其重新声明了与被继承接口相同的方法，所有特性也只是附加在这些新声明的方法上，**原有接口类型的任何信息我们都没有做改变，也无法改变。**

正确且简单的使用方式应该是：

```C#
 dynamic proxy = ContractWrapper<IDuplexChat>.CreateProxy(endpoint);
 proxy.Login("JohnWang", "123");
```

如此一来代理对象（*代理类型底层也使用类似技术动态生成类型*）会调用对动态接口类型的方法的实现，而不是原有接口类型（*IDuplexChat*）的的方法的实现。

目的一种的技术可能在实际情况中的应用不是很多，只能算是对.Net Emit 技术应用的一个学习，但实现目的二的技术在实际情况中就有很有用了。

****

### 实现目的二

首先明确实现的思路。

1. 动态生成一个继承自原有服务类的类型。
2. 覆盖原有类型的方法，在新方法中调用拦截器的代码，然后再将控制权转回原有类型的方法的实现。

中间的许多实现细节不再赘述，还是请看这篇文章   [.NET 使用 Emit 构建接口的动态代理](http://blog.degage.tech/2018-05-23-emit-dynamic-proxy/)  。



接下里我们来重点讲几个需要注意的点，请看下面的此动态类定义：

```C#
 TypeBuilder typeBuilder = ContractWrapper.ModuleBuilder.DefineType(
             typeName,
             TypeAttributes.Class | TypeAttributes.Public,
              parentType,//原有服务类的类型
              new Type[] { ContractWrapper<T>.WrappedType });
```

此处新的类型之所以还要继承这个包装原有接口的接口类型  *ContractWrapper<T>.WrappedType*   是因为在此例中WCF 双放通信，需要核对通信契约，客户端是将此包装类型作为契约与服务端通信的。

当我们成功拦截后，需要将控制权转回给原有方法的代码，而原有方法为了使其他人使用时能将控制权先转移到我们的新方法中，其在定义时已经被 new 关键词隐藏：

```C#
  MethodBuilder methodBuilder = typeBuilder.DefineMethod(
              methodInfo.Name,
              MethodAttributes.Public | MethodAttributes.Virtual |                                     MethodAttributes.NewSlot,  //隐藏
              CallingConventions.Standard,
              methodInfo.ReturnType,
              parameterTypeInfos);
```

所以如果此处不使用特殊的技术，始终无法将控制权返回给原有代码，更糟糕的是会形成递归调用，最终导致堆栈溢出。

正确的解决办法如下：

```c#
IntPtr ftn = methodInfo.MethodHandle.GetFunctionPointer();
var baseMethod = (Delegate)Activator.CreateInstance(methodGenericType,                                            proxy, ftn);
obj = baseMethod.DynamicInvoke(parameterValues);
```

我们需要获取原有方法的函数指针，以绕开运行时的调用机制，并通过此指针创建委托对象，最后执行，返回原有方法的执行结果。

在本例中我定义的方法拦截器接口如下：

```C#
 /// <summary>
    /// 用于处理对代理对象方法的调用
    /// </summary>
    public interface IMethodInterceptor
    {
        /// <summary>
        /// 处理对代理对象方法的调用
        /// </summary>
        /// <param name="proxiedType">被代理的类型</param>
        /// <param name="invokeMethod">调用的被代理的方法</param>
        /// <param name="parameterValues">方法参数值列表</param>
        /// <returns>方法的返回结果</returns>
        Object InovkeHandle(
          Object proxy,
            Type proxiedType,
            MethodInfo invokeMethod,
            Object[] parameterValues,
            MethodInterceptArgs args);
    }
```

又定义一个权限管理的拦截器类用于模拟实际应用（在真实情况下，希望业务方法在被调用前，执行一些权限的检查很常见）：

```C#
  public class PermissionMethodInterceptor : IMethodInterceptor
    {
        public Object InovkeHandle(Object proxy, Type proxiedType, MethodInfo invokeMethod, Object[] parameterValues, MethodInterceptArgs args)
        {

            IPEndPoint point = this.GetClientEndPoint();
            switch (invokeMethod.Name)
            {
                case nameof(IDuplexChat.Login):
                    {
                        //客户端登录操作
                        var sessionObj = (ISession)proxy;
                        sessionObj.Session["Account"] = parameterValues[0];
                        sessionObj.Session["Password"] = parameterValues[1];
                        sessionObj.Session["IsLogin"] = true;
                        Console.WriteLine($"登录操作，来自：{point.Address.ToString()} {point.Port}，账户:{  sessionObj.Session["Account"] } 密码：{     sessionObj.Session["Password"] }");
                    }
                    break;
                case nameof(IDuplexChat.GetData):
                    {
                        //判断是否来自本地        
                        if (point.Address == IPAddress.Loopback)
                        {
                            Console.WriteLine("来自本地的数据请求，不做权限验证！");
                        }
                        else
                        {
                            Console.WriteLine("来自外部的数据请求，判断是否登录以及账号密码是否有效");
                            var sessionObj = (ISession)proxy;
                            var isLogin = (Boolean)sessionObj.Session["IsLogin"];
                            if (!isLogin)
                            {
                                args.Handled = true;
                                Console.WriteLine("此外部连接未登录，不予返回数据！");
                                return null;
                            }

                            var account = sessionObj.Session["Account"].ToString();
                            var password = sessionObj.Session["Password"].ToString();

                            //账户、密码校验行为
                            //....
                        }
                    }
                    break;
            }

            return null;
        }
```

最后部署此服务类：

```C#
   var address = "127.0.0.1";
   var port = 20018;
   var contractType = ContractWrapper<IDuplexChat>.WrappedType;
   var imptType = ContractWrapper<IDuplexChat>.CreateImptType<DuplexChatService>    
              (typeof(PermissionMethodInterceptor));
    ServiceHost host = new ServiceHost(imptType);
    try
    {
        NetTcpBinding binding = new NetTcpBinding();
        binding.Security.Mode = SecurityMode.None;

        host.AddServiceEndpoint(
            contractType,
            binding,
            $"net.tcp://{address}:{port.ToString()}/Message");


        host.Opened += (s, e) =>
        {
            ConsoleHelper.ShowTextInfo("服务已打开。", ConsoleColor.Green);
        };
        host.Open();
        Console.ReadKey();
    }
    finally
    {
        host.Close();
    }
```

客户端完整调用代码如下：

```C#
var address = "127.0.0.1";
var port = 20018;
var contractType = ContractWrapper<IDuplexChat>.WrappedType;
ContractDescription description = ContractDescription.GetContract(contractType);
ServiceEndpoint endpoint = new ServiceEndpoint(description);

NetTcpBinding binding = new NetTcpBinding();
binding.Security.Mode = SecurityMode.None;

endpoint.Binding = binding;
endpoint.Address = new EndpointAddress($"net.tcp://{address}:{port.ToString()}/Message");

dynamic proxy = ContractWrapper<IDuplexChat>.CreateProxy(endpoint);

try
{
    var isLogin = proxy.Login("JohnWang", "123");
    if (isLogin)
    {
        ConsoleHelper.ShowTextInfo("登录成功！");
    }
    else
    {
        ConsoleHelper.ShowTextInfo("登录失败！");
    }
    var result = proxy.SayHello("JohnWang");
    ConsoleHelper.ShowTextInfo("Say Hello :" + result.ToString());

    var data = proxy.GetData("UserTable");
    if (data == null)
    {
        ConsoleHelper.ShowTextInfo("未能获取到数据", ConsoleColor.Yellow);
    }
    else
    {
        ConsoleHelper.ShowTextInfo("Get Data Table Data：", ConsoleColor.Green);

        foreach (DataRow item in data.Rows)
        {
            ConsoleHelper.ShowTextInfo("DataRow- Name: " + item["Name"].ToString(),               ConsoleColor.Yellow);
        }
    }

    Console.ReadKey();
}
finally
{
    IClientChannel channel = proxy as IClientChannel;
    if (channel != null && channel.State == CommunicationState.Opened)
    {
        try
        {
            channel.Close();
        }
        catch
        {
            channel.Abort();
        }
    }
}
```

两端执行结果如下：

客户端

![客户端](https://github.com/degagetech/degagetech.github.io/blob/master/img/posts/emit-dynamic-aop/client.png?raw=true)

服务端

![服务端](https://github.com/degagetech/degagetech.github.io/blob/master/img/posts/emit-dynamic-aop/server.png?raw=true)

您可以在[此处](https://github.com/degagetech/degagetech.github.io/blob/master/res/emit-dynamic-aop/code.rar?raw=true)获得完整的示例代码，如果有需要你可以通过[邮箱](mailto:932444208@qq.com)或者TIM(下方二维码)联系我。

<p style="float:left">
<img width='150'  src="https://raw.githubusercontent.com/degagetech/degage-platform-description/master/images/contact-tim.jpg" >
</p>