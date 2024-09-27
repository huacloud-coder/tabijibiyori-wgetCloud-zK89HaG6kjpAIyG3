
# 一、事件总线设计方案


### 1\.1、事件总线的概念


* 事件总线是一个事件管理器，负责统一处理系统中所有事件的发布和订阅。
* 事件总线模式通过提供一种松耦合的方式来促进系统内部的业务模块之间的通信，从而增强系统的灵活性和可维护性。


### 1\.2、实现的功能目标


* 注入事件总线服务到DI容器，自动注入整个程序集的事件；
* 每个事件处理程序能够自动依赖注入;
* 通过特性标注事件消息模型、事件处理器；
* 事件总线服务提供一个发布事件的方法，根据消息模型，自动找到并触发对应的事件处理程序，并传递事件参数。


# 二、使用案例


### 2\.1、事件消息模型


* 需要继承 EventArgs



```
public class UserTestEventArgs : EventArgs
{
    public string UserId { get; set; }
    public string UserName { get; set; }
}

```

### 2\.2、事件处理程序


* 该事件模型，触发的事件处理程序，会自动依赖注入



```
[LocalEventHandler(typeof(UserTestEventArgs), 1)]
public class UserTest1EventHandler(SingletonTestService singletonService, ScopeTestService scopeService, TransientTestService transientService) : ILocalEventHandler
{
    public Task OnEventHandlerAsync(object sender, UserTestEventArgs e)
    {
        Console.WriteLine($"事件1被'{sender.GetType().Name}'触发，参数：" + JsonUtils.ToJson(e));
        try
        {
            singletonService.Test();
            scopeService.Test();
            transientService.Test();
        }
        catch (Exception ex)
        {
        }
        return Task.CompletedTask;
    }
}

```

### 2\.3、注入事件总线服务到DI


* 在Startup.cs 或 Program.cs 中，注入服务



```
builder.Services.AddLocalEventBus(typeof(UserTest1EventHandler).Assembly); // 注入事件总线服务，自动注册这个程序集内的所有事件处理器。

```

### 2\.4、使用事件总线服务，触发事件


* 通过构造函数依赖注入，拿到事件总线服务 ILocalEventBus
* 调用事件总线服务，发布事件消息，触发事件处理程序 eventBus.PublishAsync(this, args);



```
// 事件总线测试控制器
public class EventTestController(
    ILocalEventBus eventBus, // 主构造函数，依赖注入事件总线服务
    IUserService userService // 测试服务
    ) : ControllerBase
{
    [HttpPost]
    public Task Test(UserTestEventArgs args) // UserTestEventArgs 事件消息模型
    {
        var users = userService.GetUsers();
        return eventBus.PublishAsync(this, args); // 发布事件消息，触发事件处理程序。 this：触发事件的对象  args：事件消息
    }
}

```

# 三、事件总线功能开发


![](https://img2024.cnblogs.com/blog/2750888/202409/2750888-20240924102601127-229954610.png)


### 3\.1、本地事件总线 服务接口


* 事件的发布方法设计，基于 .Net 标准事件模式 思想。
* 这里需要泛型参数：事件消息模型类型，以便在触发事件时可以找到注册的该消息模型对应的事件处理器。



```
/// 
/// 本地事件总线
/// 
public interface ILocalEventBus
{
    /// 
    /// 触发对应 事件消息模型 对应的 事件处理程序
    /// 
    /// 事件消息模型类型
    /// 触发事件的对象
    /// 事件消息模型
    Task PublishAsync<TEventArgs>(object sender, TEventArgs args) where TEventArgs : EventArgs;
}

```

### 3\.2、事件处理器 泛型接口



```
/// 
/// 本地事件 事件处理程序接口
/// 
/// 事件消息模型
public interface ILocalEventHandler<TEventArgs> where TEventArgs : EventArgs
{
    /// 
    /// 事件处理程序方法
    /// 
    /// 事件触发者
    /// 事件消息
    Task OnEventHandlerAsync(object sender, TEventArgs e);
}


```

### 3\.3、本地事件处理程序 特性


* 本地事件处理程序 特性 ：用于在事件处理器上标注。
* 【MessageType】 指定该消息处理器，接受的消息模型类型。 在事件触发时，通过消息类型，找到该消息处理器，并调用。
* 【Sort】如果多个消息处理器，声明接受同一个类型的消息模型。那么当这个类型的消息发布时，会触发这些多个事件处理程序，会通过指定的该触发顺序挨个触发。其中一个事件处理器执行报错，不会影响其他的。



```
/// 
/// 本地事件处理程序 特性
/// 
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public class LocalEventHandlerAttribute : Attribute
{
    /// 
    /// 事件消息模型类，需要继承EventArgs
    /// 
    public Type MessageType { get; set; }
    /// 
    /// 触发顺序 正序
    /// 
    public int Sort { get; set; }

    /// 
    /// 构造函数
    /// 
    /// 事件消息类型
    /// 触发顺序 正序
    public LocalEventHandlerAttribute(Type messageType, int sort = 0)
    {
        if(!messageType.IsSubclassOf(typeof(EventArgs)))
        {
            throw new Exception($"【LocalEventBus】The MessageType '{messageType.Name}' can not assignable from '{nameof(EventArgs)}'");
        }
        MessageType = messageType;
        Sort = sort;
    }
}

```

### 3\.4、事件处理程序信息



```
/// 
/// 事件处理程序模型
/// 
public class LocalEventHandlerModel
{
    /// 
    /// 触发顺序
    /// 
    public int Sort { get; set; }
    /// 
    /// 事件处理程序类型
    /// 
    public Type HandlerType { get; set; }
}

```

### 3\.5、事件总线服务



```
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using System.Collections.Concurrent;

namespace Singer.Framework.LocalEventBus;

/// 
/// 本地事件总线
/// 
public sealed class LocalEventBus(IHttpContextAccessor httpContextAccessor) : ILocalEventBus
{
    /// 
    /// 事件消息类型 - 对应的事件处理程序的类型集合
    /// 
    private ConcurrentDictionary> Events = new ConcurrentDictionary>();

    /// 
    /// 触发对应 事件消息模型 对应的 事件处理程序
    /// 
    /// 事件消息模型类型
    /// 触发事件的对象
    /// 事件消息模型
    public async Task PublishAsync<TEventArgs>(object sender, TEventArgs args)
        where TEventArgs : EventArgs
    {
        var argType = typeof(TEventArgs);
        if (!Events.TryGetValue(argType, out ConcurrentBag? handlers))
            return;
        if (handlers == null || handlers.Count == 0)
            return;
        foreach (var handlerModel in handlers.OrderBy(x => x.Sort))
        {
            try
            {
                // 在此时 通过 DI 和事件处理程序类型 获取到 事件处理程序实例
                var handlerInstance = httpContextAccessor?.HttpContext?.RequestServices?.GetService(handlerModel.HandlerType);
                if (handlerInstance != null)
                {
                    var method = handlerModel.HandlerType.GetMethod("OnEventHandlerAsync");
                    Task? task = (Task?)method?.Invoke(handlerInstance, [sender, args]);
                    if (task != null)
                        await task;
                }
            }
            catch (Exception)
            {
            }
        }
    }

    /// 
    /// 向事件总线中添加多个事件处理程序
    /// 
    /// 事件消息类型 - 对应的事件处理程序模型列表 字典
    public void AddHandlers(Dictionary> handlerDic)
    {
        if (handlerDic == null || handlerDic.Count == 0)
            return;
        foreach (var item in handlerDic)
        {
            if (Events.TryGetValue(item.Key, out ConcurrentBag<LocalEventHandlerModel> handlerModels))
            {
                foreach (var value in item.Value)
                {
                    handlerModels.Add(value);
                }
            }
            else
            {
                Events.TryAdd(item.Key, new ConcurrentBag(item.Value));
            }
        }
    }

}


```

### 3\.6、事件总线服务 依赖注入处理



```
/// 
/// 本地事件总线 服务拓展
/// 
public static class LocalEventBusServiceExtensions
{
    /// 
    /// 注册事件总线
    /// 将整个程序集中带有EventHandler特性的事件处理程序类都注册到事件总线中
    /// 事件处理程序所在的程序集
    /// 
    public static void AddLocalEventBus(this IServiceCollection services, Assembly handlerAssembly)
    {
        var handlerTypes = handlerAssembly.GetTypes().Where(x => x.GetCustomAttribute() != null);
        Dictionary> handlerDic = new();
        foreach (Type handlerType in handlerTypes)
        {
            services.Replace(new ServiceDescriptor(handlerType, handlerType, ServiceLifetime.Scoped)); // 将事件处理程序注入到容器中，这样事件处理程序也可以像普通服务一样使用依赖注入
            var attribute = handlerType.GetCustomAttribute();
            if (attribute == null)
                continue;

            var handlerModel = new LocalEventHandlerModel() { Sort = attribute.Sort, HandlerType = handlerType };
            if (handlerDic.ContainsKey(attribute.MessageType))
            {
                handlerDic[attribute.MessageType].Add(handlerModel);
            }
            else
            {
                handlerDic.Add(attribute.MessageType, new List() { handlerModel });
            }
        }
        LocalEventBus? eventBus = services.BuildServiceProvider().GetService() as LocalEventBus; // 此方法可能多次调用，单例处理

        services.AddHttpContextAccessor(); // 依赖Http上下文，通过HttpContext拿到每次事件触发时的 服务作用域
        services.AddSingleton(sp => // 将事件总线 注册为 单例服务
        {
            IHttpContextAccessor httpContextAccessor = sp.GetRequiredService();
            eventBus = eventBus ?? new LocalEventBus(httpContextAccessor);
            eventBus.AddHandlers(handlerDic); // 将解析出来的事件处理程序添加到事件总线中
            return eventBus;
        });
    }
}


```

 本博客参考[westworld加速](https://tianchuang88.com)。转载请注明出处！
