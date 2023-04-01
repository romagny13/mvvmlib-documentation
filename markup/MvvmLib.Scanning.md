# MvvmLib.Scanning

> Allows to scan assemblies, filter types and resolve implementation types.

Example

```cs
var services = Scrutor.Default.Scan(scan => scan.FromCallingAssembly()
    .InExactNamespaceOf<IService>()).AsImplementedInterfaces();

var views = Scrutor.Default.AsSelfInExactNamespaceOf<Shell>();
```

Strategies availables for multiple implementations found (`ScanOptions`) : 

* All (default)
* First 
* Last
* PreferredImplementationAttribute (with attribute on the type)
* Custom
* Throw

PreferredImplementationAttribute Sample 

```cs
public interface IService
{ }

public class ServiceA : IService
{ }

[PreferredImplementation]
public class ServiceB : IService
{ }

// ...
var typeMaps = Scrutor.Default.Scan(scan => scan.FromAssemblyOf<IService>()).AsImplementedInterfaces(new ScanOptions
{
    MultipleImplementationTypeHandling = MultipleImplementationTypeHandling.PreferredImplementationAttribute
});
```

