# DI \(Dependency Injection\) with Injector

> The injector allows to **resolve all types and types for implemented interfaces automatically** (views, view models, services) 

* **Auto discovery** of types and impletation types for interfaces not registered
* Allows to **register** and **resolve types, instances, factories**
* **injection parameters** and **properties**
* **Lifetimes**: Transcient, Singleton, InstancePerScope, InstancePerMatchingScope
* **Scopes** and **child scopes**
* **Interceptors** allow to intercept value resolution
* Custom value resolvers
* Attributes: **PreferredConstructor**, **PreferredImplementation** (for interfaces), **Dependency** (for properties and parameters) 

**Require to be registered :**

* **Singletons**
* Registrations with name
* Registration with injection parameters/ properties
* Use the **PreferredImplementation attribute** with many implementation of interfaces.


## Comparison

_For example with Autofac_

**Autofac**

<img src="https://res.cloudinary.com/romagny13/image/upload/v1559055307/comp_autofac_bsognl.png" />

**Injector**

<img src="https://res.cloudinary.com/romagny13/image/upload/v1565189594/ioc-1_vi1oy3.png" />


Namespace: required for extensions methods

```cs
using MvvmLib.IoC;
```

## Registration

> The injector allows to **resolve all types and types for implemented interfaces automatically** but it's possible to register "manually" the services.

_Types_

```cs
var injector = new Injector();

injector.RegisterType(typeof(MyItem));
injector.RegisterType(typeof(MyItem), "MyName"); // with name
injector.RegisterType(typeof(IMyService), typeof(MyService));
injector.RegisterType(typeof(IMyService), typeof(MyService), "MyName"); // with name
```

With extension methods

```cs
using MvvmLib.IoC;
// ...

var injector = new Injector();

injector.RegisterType<MyItem>();
injector.RegisterType<MyItem>("MyName"); // with name
injector.RegisterType<IMyService, MyService>();
injector.RegisterType<IMyService, MyService>("MyName"); // with name
```

_Instances_

```cs
var injector = new Injector();

injector.RegisterInstance(typeof(MyItem), new MyItem());
injector.RegisterInstance(typeof(MyItem), new MyItem(), "MyName"); // with name
```

With extension methods

```cs
using MvvmLib.IoC;
// ...

var injector = new Injector();

injector.RegisterInstance<MyItem>(new MyItem());
injector.RegisterInstance<MyItem>(new MyItem(), "MyName", ); // with name
```

_Factories_

```cs
var injector = new Injector();

injector.RegisterFactory(typeof(MyItem), () => new MyItem());
injector.RegisterFactory(typeof(MyItem), () => new MyItem(), "MyName"); // with name
```

With extension methods

```cs
using MvvmLib.IoC;
// ...

var injector = new Injector();

injector.RegisterFactory<MyItem>(() => new MyItem());
injector.RegisterFactory<MyItem>(() => new MyItem(), "MyName"); // with name
```

### Lifetimes

* Transcient by default (TranscientLifetime): get always a new instance
* Singleton (SingletonLifetime): get always the same instance
* InstancePerScope (InstancePerScopeLifetime): get always the same instance per scope
* InstancePerMatchingScope (InstancePerMatchingScopeLifetime): get the same instance per matching scope and child scopes.

With **Fluent Api**

_Singleton_

Examples:

```cs
injector.RegisterType<MyItem>().AsSingleton(); // type
injector.RegisterFactory<MyItem>(() => new MyItem()).AsSingleton(); // factory
// or
injector.RegisterType(typeof(MyItem)).AsSingleton(); // type
injector.RegisterFactory(typeof(MyItem), () => new MyItem()).AsSingleton(); // factory
```

_InstancePerScope_

Examples:

```cs
injector.RegisterType<MyItem>().InstancePerScope(); // type
injector.RegisterFactory<MyItem>(() => new MyItem()).InstancePerScope(); // factory
// or
injector.RegisterType(typeof(MyItem)).InstancePerScope(); // type
injector.RegisterFactory(typeof(MyItem), () => new MyItem()).InstancePerScope(); // factory
// or
injector.RegisterScoped<MyItem>();
injector.RegisterScoped<MyItem>(() => new MyItem());
```

_InstancePerMatchingScope_

Examples:

```cs
injector.RegisterType<MyItem>().InstancePerMatchingScope("MyScope"); // with the scope name to match
// or
injector.RegisterType(typeof(MyItem)).InstancePerMatchingScope("MyScope"); // with the scope name to match
```

## Registering Type for all implemented interfaces

```cs
public interface ILookupDataServiceType1 { }

public interface ILookupDataServiceType2 { }

public interface ILookupDataServiceType3 { }

public class LookupDataService : ILookupDataServiceType1, ILookupDataServiceType2, ILookupDataServiceType3 
{ }
```

```cs
var injector = new Injector();
var options = injector.RegisterTypeForImplementedInterfaces<LookupDataService>();
```

... Avoid to do:

```cs
injector.RegisterTypeForImplementedInterfaces<ILookupDataServiceType1, LookupDataService>();
injector.RegisterTypeForImplementedInterfaces<ILookupDataServiceType2, LookupDataService>();
injector.RegisterTypeForImplementedInterfaces<ILookupDataServiceType3, LookupDataService>();
```

It's possible to change the lifetime for one registration 

```cs
var options2 = options[typeof(ILookupDataServiceType2)];
options2.AsSingleton();
```

## Registering Singleton for all implemented interfaces

Allows to get the same instance for all "services".

```cs
var injector = new Injector();
injector.RegisterSingletonForImplementedInterfaces<LookupDataService>();
```

## Registering Scoped for all implemented interfaces

Allows to get the same instance for all "services" by scope.

```cs
var injector = new Injector();
injector.RegisterScopedForImplementedInterfaces<LookupDataService>();
```

## PreferredImplementationAttribute

The injector resolves not registered types for **interfaces**. For many implementations use the **PreferredImplementationAttribute** Example:

```cs
public interface IMyService
{ }

public class MyService1 : IMyService
{ }

[PreferredImplementation]
public class MyService2 : IMyService
{ }

public class MyService3 : IMyService
{ }
```

Change the option

```cs
var injector = new Injector(new ContainerOptions
{
    AutoDiscoveryOptions = new ScanOptions
    {
        MultipleImplementationTypeHandling = MultipleImplementationTypeHandling.PreferredImplementationAttribute
    }
});
```

Change the lifetime On discovery (Transcitent by default)

```cs
var injector = new Injector();
injector.Context.AutoDiscoverer.LifetimeOnDiscovery = LifetimeOnDiscovery.Singleton;
```

Add additional types for resolution (for services registered in another assembly)

```cs
injector.Context.AutoDiscoverer.AdditionalTypes.AddRange(typeof(IService).Assembly.ExportedTypes); 
```

## Injection Parameters

1 - Allow to provide values for value types, string, enumerables...

Example

```cs
public class MyItem
{
    public MyItem(string myString, int myInt, List<string> myList)
    {

    }
}
```

Inject values

```cs
var injector = new Injector();

injector.RegisterType<MyItem>()
    .WithInjectionParameters(new InjectionParameter("myString", value: "My value"), 
                             new InjectionParameter("myInt", value: 100), 
                             new InjectionParameter("myList", value: new List<string> { "A", "B" }));
```

2 - Other example allow to select a named registration

```cs
public class MyInnerItem { }

public class MyItem
{
    public MyItem(MyInnerItem myInnerItem)
    {

    }
}
```

```cs
var injector = new Injector();

injector.RegisterType<MyInnerItem>("Key1"); 

injector.RegisterType<MyItem>()
    .WithInjectionParameters(new InjectionParameter("myInnerItem", name: "Key1")); // provide the name of the registration to use
```

_Alternative with Dependency Attribute_

```cs
public class MyInnerItem { }

public class MyItem
{
    public MyItem([Dependency(Name ="Key1")] MyInnerItem myInnerItem)
    {

    }
}
```

```cs
var injector = new Injector();

injector.RegisterType<MyInnerItem>("Key1"); 

injector.RegisterType<MyItem>();
```

## Resolve

```cs
var injector = new Injector();

// resolve type, instance, factory ... with the lifetime
var value = injector.Resolve(typeof(MyItem));
var value = injector.Resolve(typeof(MyItem), "MyName"); // with name
var value = injector.Resolve(typeof(IMyService)); 
var value = injector.Resolve(typeof(IMyService), "MyName"); // with name
```

With extension methods

```cs
using MvvmLib.IoC;
// ...

var injector = new Injector();

var value = injector.Resolve<MyItem>();
var value = injector.Resolve<MyItem>("MyName"); // with name
var value = injector.Resolve<IMyService>();
var value = injector.Resolve<IMyService>("MyName"); // with name
```

_InstancePerScope_


```cs
var injector = new Injector();
injector.RegisterScoped<MyItem>();

using (var scope1 = injector.CreateScope())
{
    var instance1 = scope1.Resolve<MyItem>(); 
    var instance2 = scope1.Resolve<MyItem>(); // instance1 equals instance2
}

using (var scope2 = injector.CreateScope())
{
    var instance3 = scope2.Resolve<MyItem>(); 
    var instance4 = scope2.Resolve<MyItem>(); // instance3 equals instance4
}
```

Named scope:

```cs
using (var scope1 = injector.CreateScope("MyScope"))
{
}
```

_InstancePerMatchingScope_

```cs
var injector = new Injector();
injector.RegisterType<MyItem>().InstancePerMatchingScope("MyScope");

using (var scope = injector.CreateScope("MyScope"))
{
    var instance1 = scope.Resolve<MyItem>();
    using (var scope1 = scope.CreateScope())
    {
        var instance2 = scope1.Resolve<MyItem>();
        using (var scope2 = scope1.CreateScope())
        {
            var instance3 = scope2.Resolve<MyItem>(); // instance1, instance2 and instance3 are equals
        }
    }
}

using (var scope = injector.CreateScope("MyScope"))
{
    var instance4 = scope.Resolve<MyItem>();
    using (var scope1 = scope.CreateScope())
    {
       var instance5 = scope1.Resolve<MyItem>(); // instance4 equals instance5 but instance1 (...) and instance4 (...) are not equals
    }
}
```

## Func

Sample 1

```cs
var injector = new Injector();
var resolver = injector.Resolve<Func<MyItem>>();

MyItem item = resolver();
```

Sample 2

```cs
public class MyItemWithFunc
{
    public MyItemWithFunc(Func<MyItem> resolver)
    {
        MyItem item = resolver();
    }
}
```

```cs
var injector = new Injector();

injector.RegisterType<MyItem>();
injector.RegisterType<MyItemWithFunc>();

var instance = injector.Resolve<MyItemWithFunc>();
```

Sample 3

With `Func<string,T>`. Useful for conditional resolution in the constructor.

```cs
var injector = new Injector();
injector.RegisterType<IMyService, MyService1>("Key1");
injector.RegisterType<IMyService, MyService2>("Key2");
injector.RegisterType<MyItem>();

var instance1 = injector.Resolve<MyItem>();
```

```cs
public interface IMyService { }
public class MyService1 : IMyService { }
public class MyService2 : IMyService { }

public class MyItem
{
    public MyItem(Func<string, IMyService> resolver)
    {
        var name = /* condition */ ? "Key1": "Key2";
        var instance = resolver(name); // resolve by name
    }
}
```

## Lazy

Sample 1

```cs
var injector = new Injector();
var lazy = injector.Resolve<Lazy<MyItem>>();

MyItem item = lazy.Value;
```

Sample 2

```cs
public class MyItemWithFuncAndLazy
{
    public MyItemWithFuncAndLazy(Func<MyItem> func1, Lazy<MyItem> lazy1)
    {
        MyItem MyItem1 = func1();
        MyItem MyItem2 = lazy1.Value;
    }
}
```

Note: dependency attribute and injection parameter are availables for constructor parameters with name. 


## Property Injection

1 - define properties to include

_With InjectionProperty_

```cs
var injector = new Injector();
injector.RegisterType<MyInnerItem>();
injector.RegisterType<MyItem>().WithInjectionProperties(new InjectionProperty("MyInner")); // propertyName

var instance = injector.BuildUp<MyItem>();
```

_Alternative with the Dependency Attribute_

```cs
public class MyInnerItem { }

public class MyItem
{
    public string MyString { get; set; } // not included

    [Dependency]
    public MyInnerItem MyInner { get; set; } // included
}
```


2 - Provide a value with InjectionProperty

```cs
var injector = new Injector();

injector.RegisterType<MyItem>().WithInjectionProperties(new InjectionProperty("MyString", value: "My value")); // propertyName, value

var instance = injector.BuildUp<MyItem>();
```

```cs
public class MyItem
{
    public string MyString { get; set; }
}
```

3 - Provide the name the name of the registration to use

_With InjectionProperty_

```cs
var injector = new Injector();

injector.RegisterType<MyInnerItem>("Key1");

injector.RegisterType<MyItem>().WithInjectionProperties(new InjectionProperty("MyInner", name: "Key1")); // propertyName, name

var instance = injector.BuildUp<MyItem>();
```

```cs
public class MyInnerItem { }

public class MyItem
{
    public MyInnerItem MyInner { get; set; }
}
```
_Alternative with the Dependency Attribute_

```cs
public class MyInnerItem { }

public class MyItem
{
    [Dependency(Name ="Key1")]
    public MyInnerItem MyInner { get; set; }
}
```

4 - BuildUp with an existing instance

```cs
var item = new MyItem();
var instance = injector.BuildUp<MyItem>(item);
```

## OnResolved Callback

Allows to handle circular references for example.


```cs
public class CircularPropertyItem
{
    public CircularPropertyItem Item { get; set; } // circular reference

    // other properties
    public string MyString { get; set; }
    public SubItem MySubItem { get; set; }
}
```

```cs
injector.RegisterType<CircularPropertyItem>().OnResolved((registration, instance) =>
{
    var item = instance as CircularPropertyItem;
    item.Item = item; // circular reference

    // other properties
    item.MyString = "My value";
    item.MySubItem = injector.Resolve<SubItem>();
});

var instance = injector.Resolve<CircularPropertyItem>();
```

## Resolve All

Resolves and returns all values for a type.

Example:

```cs
var injector = new Injector();

injector.RegisterInstance<MyItem>(new MyItem());
injector.RegisterInstance<MyItem>("Key1", new MyItem());

var items = injector.ResolveAll<MyItem>();
```

## AutoDiscovery

By default, non registered types are resolved. To change this behavior:

```cs
injector.AutoDiscoverer.AutoDiscovery = false;
```

## NonPublicConstructors

By default, non public constructors/ properties are found. To change this behavior:

```cs
var injector = new Injector(new ContainerOptions { NonPublicConstructors = false });
```

## PreferredConstructorAttribute

Example

```cs
public class MyViewModel
{
    public MyViewModel()
    {

    }

    public MyViewModel(IMyService1 service1)
    {

    }

    [PreferredConstructor]
    public MyViewModel(IMyService1 service1, IMyService2 service2)
    {

    }
}
```

## Constructor Selector

It's possible to create a custom selector (for example a MostParametersConstructorSelector).


```cs
public class MyConstructorSelector : IConstructorSelector
{
    public ConstructorInfo ResolveConstructor(Type implementationType, bool nonPublic)
    {
        // implements ...
    }
}
```

```cs
var injector = new Injector(new ContainerOptions { ConstructorSelector = new MyConstructorSelector() });
```

## DelegateFactory

The **ExpressionDelegateFactory** is used by **default** to create instances (with Expressions **Linq**).

Change to ReflectionDelegateFactory (less performant)

```cs
var injector = new Injector(new ContainerOptions { DelegateFactory = new ReflectionDelegateFactory() });
```

Note: It's possible to create a custom DelegateFactory (implements IDelegateFactory interface). For example with `System.Reflection.Emit`. 

## Interceptors

Create an interceptor

```cs
public class MyInterceptor1 : InterceptorBase
{
    protected override Task Process(InterceptionContext input, Func<InterceptionContext, bool, Task> next)
    {     
        // do things before
        next(input, true); // <= Cancels next interceptors execution with false
        // do things after
        return Task.FromResult<object>(null);


        // - or - 
        // do things before
        // return next(input, true);
    }
}
```

Register the interceptors

```cs
injector.RegisterType<MyItem>().WithInterceptors(new MyInterceptor1(), new MyInterceptor2()); // ...

var instance = injector.Resolve<MyItem>(); // => interceptors executed
```

Note: the instance is resolved by the **last interceptor** (`ResolveInterceptor`). `input.Result` contains the instance **after resolution**. So at beginning the result is null, it's possible to provide the result and cancel next interceptors execution.

Note 2: interceptors with MvvmLib.IoC are not invoked on method calls (unlike Unity for example).


_Async_

```cs
public class MyInterceptor2 : InterceptorBase
{
    protected override async Task Process(InterceptionContext input, Func<InterceptionContext, bool, Task> next)
    {
        // do things before

        await next(input, true);

        // do things after
    }
}
```

## Resolvers

Default resolvers:

* **ValueResolver**: for `ValueType` and `string`, returns default values
* **EnumerableTypeResolver**: for `IEnumerable`, returns default values
* **FuncResolver**: for `Func<T>`
* **FuncWithNameResolver**: for `Func<string,T>` used to resolve a type by name. Useful for conditional resolution in the constructor.
* **LazyResolver**: for `Lazy<T>`
* **TypeResolver**: the default resolver used to resolve values for Types

It's possible to create **custom resolvers**


```cs
public class MyItemResolver : IValueResolver
{
    /// <summary>
    /// Checks if a registration is required.
    /// </summary>
    public bool RequireRegistration
    {
        get { return true; }
    }

    public bool CanResolve(Type type)
    {
        // used to determine if the resolver can be used
        return type == typeof(MyItem);
    }

    public object Resolve(Type type, string name)
    {
        // the method used to resolve the value
        return new MyItem();
    }
}
```

Register and use the custom resolver

```cs
var injector = new Injector();

// register the custom resolver
injector.RootScope.ResolverProvider.AddCustomResolver(new MyItemResolver());

var item = injector.Resolve<MyItem>(); // the custom resolver is used
```

Note: the custom resolvers are copied on new scope creation.


## Events

* Registering
* Registered
* Resolved

