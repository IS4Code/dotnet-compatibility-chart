# .NET Feature Compatibility Chart
This document serves as an overview of compatibility differences between versions of .NET Framework and .NET Core (including .NET 5+).
Its purpose is to list the distinguishing features of both environments, and how they translate to the other one, if at all.

Note that this chart is not just about “missing APIs” (comprehensibly listed [here](https://learn.microsoft.com/en-us/dotnet/core/compatibility/fx-core) or [here](https://learn.microsoft.com/en-us/dotnet/core/compatibility/unsupported-apis)), but also major differences in approach to solving problems that may cover whole areas of features and technologies.
One may also use it to find workarounds to use when porting code over to either environment (or .NET Standard 2.0 which is, for the most part, intersection of both).

### Legend

<table>
<tr><th>Compatibility</th><th>Meaning</th></tr>
<tr>
<th>Nonexistent</th>
<td>There is no replacement of the feature as is, without significantly transforming existing code architecture.</td>
</tr>
<tr>
<th>Bare</th>
<td>The replacement covers only the simplest use cases of the original feature.</td>
</tr>
<tr>
<th>Adequate</th>
<td>The replacement works for the common use cases, but the rest is either impossible or requires effort to adapt.</td>
</tr>
<tr>
<th>Sufficient</th>
<td>The replacement works for most use cases without any additional considerations.</td>
</tr>
<tr>
<th>Full</th>
<td>The replacement is a one-to-one correspondence to the original feature.</td>
</tr>
</table>

## .NET Framework to .NET Core

<table>

<tr><th colspan="5"><h3>Code execution</h3></th></tr>
<tr><th>Feature</th><th>Replacement(s)</th><th>Supported from</th><th>Compatibility</th><th>Discussion</th></tr>

<tr>
<th rowspan="3">
<h4>Proxying</h4>
(<a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.remoting.proxies.realproxy"><code>RealProxy</code></a> and <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.remoting.proxies.proxyattribute"><code>ProxyAttribute</code></a>)
</th>
<td>
<a href="https://learn.microsoft.com/en-us/dotnet/api/system.reflection.dispatchproxy"><code>DispatchProxy</code></a>
</td>
<td>
.NET Core 1.0
</td>
<td rowspan="3" align="center">
Bare
</td>
<td rowspan="3" align="center">
dotnet/runtime#38873
</td>
</tr>
<tr>
<td>
<a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.idynamicinterfacecastable"><code>IDynamicInterfaceCastable</code></a>
</td>
<td>
.NET 5
</td>
</tr>
<tr>
<td>
<a href="https://github.com/dotnet/roslyn/blob/main/docs/features/interceptors.md">Interceptors (experimental)</a>
</td>
<td>
C# 12 (preview)
</td>
<tr>
<td colspan="5" align="justify">

Proxying allows the runtime to intercept calls performed through an interface or on a class deriving from <a href="https://learn.microsoft.com/en-us/dotnet/api/system.marshalbyrefobject"><code>MarshalByRefObject</code></a> and send them as messages to an instance of <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.remoting.proxies.realproxy"><code>RealProxy</code></a> which mimics the type that it is accessed through.
This includes even non-<code>virtual</code> or base methods like <code>GetType</code>.

There is no adequate replacement for this feature.
<a href="https://learn.microsoft.com/en-us/dotnet/api/system.reflection.dispatchproxy"><code>DispatchProxy</code></a> has a comparable usage, but only supports a single interface; this could theoretically be upgraded to any number of interfaces or a non-<code>sealed</code> class (as is possible with <code>System.Reflection.Emit</code>), but <code>virtual</code> non-<code>sealed</code> methods are the only members that can be intercepted (with <code>MarshalByRefObject</code>, even field access is intercepted).

<a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.idynamicinterfacecastable"><code>IDynamicInterfaceCastable</code></a> is a solid feature on its own, allowing to extend an object's set of supported interfaces at runtime.
This fully substitutes <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.remoting.iremotingtypeinfo"><code>IRemotingTypeInfo</code></a>, used by proxies for the same purpose, with the difference of the implementation being handled by the proxy itself &ndash; for the case of <code>IDynamicInterfaceCastable</code>, to be efficient, the implementation has to be defined through a specially marked interface with default interface implementation, but such can be generated at runtime with <code>System.Reflection.Emit</code>.
With these two combined, proxying for interfaces can be fully achieved.

Interceptors are a compile time-only feature, supported only for C#.
As such, they can work with potentially any method treated as interceptable by a user-provided source generator, but their functionality is limited to the code that uses them, hence any truly transparent runtime or reflection interoperability is impossible.
Additionally, there are currently limitations with the technology, such as inability to use interceptable methods as delegates or method pointers (but this is not a limitation in theory).

Similar runtime-based approaches include CIL rewriting or usage of the profiling API, but these work reliably only in limited cases.
</td>
</tr>

<tr><th>Feature</th><th>Replacement(s)</th><th>Supported from</th><th>Compatibility</th><th>Discussion</th></tr>

<tr>
<th rowspan="2">
<h4>Application domains</h4>
(<a href="https://learn.microsoft.com/en-us/dotnet/api/system.appdomain.createdomain?view=netframework-4.8.1"><code>AppDomain.CreateDomain</code></a>)
</th>
<td>
<a href="https://github.com/IS4Code/DotNetIsolator">DotNetIsolator</a>
</td>
<td>
<i>external package</i>
</td>
<td rowspan="2" align="center">
Adequate
</td>
<td rowspan="2"></td>
</tr>
<tr>
<td>
<a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.loader.assemblyloadcontext"><code>AssemblyLoadContext</code></a>
</td>
<td>
.NET Core 1.0 / <a href="https://www.nuget.org/packages/System.Runtime.Loader/"><i>external package</i></a>
</td>
</tr>
<td colspan="5" align="justify">

Application domains (AppDomains) are a way to isolate one .NET environment from another within the same process.
Such an environment has its own objects, static fields, and loaded assemblies, and can be unloaded on demand.
Communication between two application domains is established by sending messages back and forth from proxied calls.

There is no possibility of application domain-level isolation within .NET Core, and there will probably never be.
One of the reasons (and potential caveats of application domains) is that native code cannot be isolated at all, including things outside .NET such as environment variables, console, or the file system.
Code access security had been used to prevent such operations, but practice has shown it is practically impossible to protect against everything.

A lesser use of application domains is to be able to unload assemblies on demand.
This is achieved by a more convenient and lightweight approach, using <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.loader.assemblyloadcontext"><code>AssemblyLoadContext</code></a> to control how assemblies are resolved.
Unloading is not instantaneous however, and will happen only if there are no references to anything from the context.

A working alternative is to host the .NET Core runtime itself, but as there is no code access security, such would have the same access as the parent runtime.
A possible solution is the experimental <a href="https://github.com/IS4Code/DotNetIsolator">DotNetIsolator</a>, hosting .NET within an embedded WebAssembly environment.
This is secure as long as the WebAssembly host is secure, and allows for greater control over the execution, albeit at less convenience than what remoting allows between application domains.
</td>
</tr>

<tr><th>Feature</th><th>Replacement(s)</th><th>Supported from</th><th>Compatibility</th><th>Discussion</th></tr>

<tr>
<th>
<h4><a href="https://learn.microsoft.com/en-us/dotnet/api/system.threading.thread.abort"><code>Thread.Abort</code></a></h4>
</th>
<td>
<a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.controlledexecution.run"><code>ControlledExecution.Run</code></a>
</td>
<td>
.NET 7
</td>
<td align="center">
Sufficient
</td>
<td align="center">dotnet/runtime#41291</td>
</tr>
<td colspan="5" align="justify">

Non-cooperative cancellation has been traditionally performed in .NET Framework by aborting the thread running the uncontrollable code, resulting in <code>ThreadAbortException</code> being asynchronously thrown and propagated up the call stack.

<code>ControlledExecution.Run</code> does not allow aborting any arbitrary thread; only those currently executing the method.
It is also not possible to continue with the execution like could be done with <a href="https://learn.microsoft.com/en-us/dotnet/api/system.threading.thread.resetabort"><code>Thread.ResetAbort</code></a>.
Lastly, the .NET Core solution might be a little more dangerous, since the runtime does not have that many protections against this possibility compared to .NET Framework.
None of these differences are substantial enough however (as it was dangerous before too), making this feature sufficiently replaceable.
</td>
</tr>

<tr><th>Feature</th><th>Replacement(s)</th><th>Supported from</th><th>Compatibility</th><th>Discussion</th></tr>

<tr>
<th>
<h4><a href="https://learn.microsoft.com/en-us/previous-versions/dotnet/framework/code-access-security/code-access-security">Code access security</a>
</h4>
</th>
<td>
<a href="#application-domains">Isolation</a>
</td>
<td>
</td>
<td align="center">
Nonexistent
</td>
<td>
</td>
</tr>
<tr>
<td colspan="5" align="justify">

Code access security (CAS) was used to specify security permissions that have to be granted when executing certain methods in the runtime, both in a declarative and imperative style.
.NET Core and later don't include any such feature, as, in practice, it could not be used for full sandboxing and ensuring security when running untrusted assemblies.
Similarly, other security-related features are unavailable, such as <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.disableprivatereflectionattribute"><code>DisablePrivateReflectionAttribute</code></a>, not working after .NET Core 2.0.

There are no sufficient alternatives to CAS that would not require <a href="#application-domains">isolation</a>, specifically any code run within .NET Core has the same permissions to access runtime or system resources as any other code.
When a runtime is isolated, such as in another process, container, or a WebAssembly host, the hosting environment can limit access to resources better than the .NET runtime.

Outside of security, an alternative to permission-based architecture is following proper encapsulation, avoiding mutable global state, and using other techniques such as interfaces and dependency injection to decrease coupling, preventing code from getting access to more than what is needed.
</td>
</tr>

<tr><th>Feature</th><th>Replacement(s)</th><th>Supported from</th><th>Compatibility</th><th>Discussion</th></tr>

<tr>
<th>
<h4><a href="https://learn.microsoft.com/en-us/archive/msdn-magazine/2005/october/using-the-reliability-features-of-the-net-framework">Reliability</a></h4>
(<a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.runtimehelpers.prepareconstrainedregions"><code>Runtime&shy;Helpers.Prepare&shy;Constrained&shy;Regions</code></a>, <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.constrainedexecution.criticalfinalizerobject"><code>Critical&shy;Finalizer&shy;Object</code></a>, <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.exceptionservices.handleprocesscorruptedstateexceptionsattribute"><code>Handle&shy;Process&shy;Corrupted&shy;State&shy;Exceptions&shy;Attribute</code></a>)
</th>
<td>
<a href="#application-domains">Isolation</a>
</td>
<td>
</td>
<td align="center">
Nonexistent
</td>
<td>
</td>
</tr>
<tr>
<td colspan="5" align="justify">

In .NET Framework, it is possible to protect certain regions of code from being interrupted and possibly corrupting the global state as a result of asynchronous exceptions like <code>OutOfMemoryException</code>, <code>StackOverflowException</code>, and <code>ThreadAbortException</code>.
By introducing a <a href="https://learn.microsoft.com/en-us/dotnet/framework/performance/constrained-execution-regions">constrained execution region</a>, the runtime ensures that these so-called out-of-band exceptions will not happen in the middle of a “back-out” code (that is <code>catch</code>, <code>finally</code>, and so forth) usually intended to perform cleanup of native resources.
Any resources needed for the reliable execution of such code are prepared before the region is entered, and any operations that might endanger it are prohibited.
Additionally, correct execution is ensured even in case of a possible corruption, in which case the application domain is unloaded and a new one may be initialized, without affecting the consistency of the whole process.
Similarly, effort is made to run finalizers even in the case of forced unloading of the runtime, such as with <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.constrainedexecution.criticalfinalizerobject"><code>CriticalFinalizerObject</code></a>, and code itself may be run in the situation of a fatal exception by using <a href="https://learn.microsoft.com/en-us/dotnet/api/system.runtime.exceptionservices.handleprocesscorruptedstateexceptionsattribute"><code>HandleProcessCorruptedStateExceptionsAttribute</code></a>.

None of these mechanisms are supported in .NET Core, the philosophy being that in the case of a critical failure, the process is already corrupted beyond repair and any recovery has to be made externally.
As such, any alternative to this feature requires <a href="#application-domains">isolation</a> of the runtime, and relegating all reliability measures to the host.
</td>
</tr>

<tr><th colspan="5"><h3>Reflection</h3></th></tr>
<tr><th>Feature</th><th>Replacement(s)</th><th>Supported from</th><th>Compatibility</th><th>Discussion</th></tr>

<tr>
<th rowspan="2">
<h4><a href="https://learn.microsoft.com/en-us/dotnet/api/system.reflection.emit.assemblybuilder.save"><code>AssemblyBuilder.Save</code></a></h4>
</th>
<td>
<a href="https://github.com/Lokad/ILPack">ILPack</a>
</td>
<td>
<i>external package</i>
</td>
<td rowspan="2" align="center">
Sufficient
</td>
<td rowspan="2">
dotnet/runtime#15704
</td>
</tr>
<tr>
<td>
Re-added
</td>
<td>
.NET 9
</td>
</tr>
<tr>
<td colspan="5" align="justify">

The ability to save dynamically built assemblies has been a major feature absent from .NET Core, but it was re-added in .NET 9, with only a few missing parts which nevertheless don't prevent the feature from being sufficiently compatible.
</td>
</tr>

<tr><th>Feature</th><th>Replacement(s)</th><th>Supported from</th><th>Compatibility</th><th>Discussion</th></tr>

<tr>
<th>
<h4><a href="https://learn.microsoft.com/en-us/dotnet/api/system.reflection.assembly.reflectiononlyload"><code>Assembly.Reflection&shy;Only&shy;Load</code></a></h4>
</th>
<td>
<a href="https://learn.microsoft.com/en-us/dotnet/api/system.reflection.metadataloadcontext"><code>MetadataLoadContext</code></a>
</td>
<td>
<i>external package</i>
</td>
<td align="center">
Sufficient
</td>
<td>
</td>
</tr>
<tr>
<td colspan="5" align="justify">

In .NET Framework, assemblies could be loaded in a special reflection-only context that prevents them from being executed.
This context is not retained in .NET Core, but <a href="https://learn.microsoft.com/en-us/dotnet/api/system.reflection.metadataloadcontext"><code>MetadataLoadContext</code></a> is provided as a replacement, building on top of existing reflection abstractions without involving the runtime loader.

<code>MetadataLoadContext</code> exposes a similar API to <code>AssemblyLoadContext</code>, but it also has a few limitations at the moment &ndash; even the runtime assemblies have to be explicitly provided and loaded into the context, and missing types cannot be independently resolved. 
</td>
</tr>

</table>
