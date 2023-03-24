# MvvmLib.Scanning

Allows to scan assemblies to filter types and resolve implementation types.

Example

```cs
var services = Scrutor.Default.Scan(scan => scan.FromCallingAssembly().InExactNamespaceOf<IService>()).AsImplementedInterfaces();
var views = Scrutor.Default.AsSelfInExactNamespaceOf<Shell>();
```