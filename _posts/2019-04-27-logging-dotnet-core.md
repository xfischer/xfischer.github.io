---
published: true
layout: post
comments: true
title: Logging in a .Net Core Library
tags:
  - dotnet
  - netcore
  - logging
---


## Foreword

In this post, I'll try to explain how to setup logging (and dependency injection in the process) in a .Net Core library, and setup an application configuring listeners and filters as an example.

You can [Go directly to the Solution](#the-solution) if you want.

## The context, the mind process

My [DEM.Net project](https://github.com/dem-net) is a class library, intended to be used as a NuGet Package for any kind of .Net application (WebAPI, Console app, WinForms, other class library).

At the early development stage of this project, I was using good old ```Console.WriteLine()``` or ```Debug.WriteLine()``` statements to give output to me, as a developer inside Visual Studio.

Then when the classes were OK, I needed to write *only* warnings in specific cases. I still could use ```Console.WriteLine("WARNING: bla bla")```, but this is ugly I know you can feel it.

## Why should I log

> I usually log
>
>- For troubleshooting : progress of a task, cases encountered, dumps of in-memory structures : everything that will be useful if I need to know what was running and why.
>- To give the consuming code a basic knowledge of the progress of 
their demand. Anything that could prevent the " black box " effect : 
waiting for something to complete with no feedback.
>- At different levels (Trace, Debug, Info, Warn, Error, Critical) : 
it's very important to prioritize what's important and what's internal 
or critical.
>- With *categories*, allowing logical grouping of messages.
>

**The rule-of-thumb is : read the logs, and you should know exactly what is going on inside the code.**

So I have built a custom wrapper for different types of logs.

```csharp
using System.Diagnostics;

public static class Logger
{
  public static void Info(string message)
  {
    Trace.TraceInformation(message);
  }
  // ... plus other methods for each log lvel
}
```

I could emit logs from my code with a single line, and I knew I could migrate with no pain when I will find a solution for clean logging.

```csharp
Logger.Info($"Processing file {fileName}...");
```

## The problem

- Using ```System.Diagnostics.Trace```, library consumers are constrained to listen to Traces, and coded this way, there is no categories to enable them to filter out those trace logs.
- Depending on the consumer logging system, they may need to redirect my Trace output. That's a lot to ask, they may never do that.
- I need to enable logging **without actually logging anywhere**. The consumer can be able to choose where the traces go, either Console, Trace, a Database, Envent Logs, File, ...
- I need to find a solid solution, where there is no custom hand-coded class, I want to be pro.

## The solution

### The `ILogger<T>` interface

This interface resides in the NuGet package `Microsoft.Extensions.Logging.Abstractions`, which is part of [ASP.Net Extensions catalog](https://github.com/aspnet/Extensions).

>.NET APIs for commonly used programming patterns and utilities, such as dependency injection, logging, and configuration.

This is not tied to ASP.Net alone in any way. We can use it in our library!

This interface is simple and allows log calls to be made without having a concrete implementation. Don't dive into the internals, it's made for ease of use. Just remember that the generic type T is the type of the service for which the logging is made. Remember T as T*YourService*.

Let's setup :

- A library with a simple service. We want this service to log its activity.
- A Console app referencing the library and calling the service.

### 1. Starting point : a class library with your `MyService`

Create a new Class Library project `MyNetStandardLib` targeting .Net Standard 2.0.

Your service takes a param and simulates a long processing task for 1 second.

```csharp
using System;
using System.Threading;

namespace MyNetStandardLib
{
    public class MyService
    {
        public void DoSomething(string theParam)
        {
            // Simulate something we do for 1 second
            Thread.Sleep(1000);

            // done
        }
    }
}
```

### 2. Add package Microsoft.Extensions.Logging.Abstractions

You can do it by right clicking your project and choosing *Add NuGet Packages...* or via dotNet CLI:

```cli
nuget add Microsoft.Extensions.Logging.Abstractions
```

### 3. Add support for `ÌLogger<TYourService>`

Change the service class as follows:

- Add a constructor taking an `ILogger<MyService>`
- Add a backing field for the logger

```csharp
public class MyService
{
    // backing field
    private readonly ILogger<MyService> _logger;

    // constructor
    public MyService(ILogger<MyService> logger = null)
    {
        _logger = logger;
    }

    public void DoSomething(string theParam)
    {
        _logger?.LogInformation($"DoSometing with {theParam}...");

        // Simultate something me do for 1 second
        Thread.Sleep(1000);

        // Test logs at each level
        _logger?.LogTrace("DoSometing TRACE message");
        _logger?.LogDebug("DoSometing DEBUG message");
        _logger?.LogInformation("DoSometing INFO message");
        _logger?.LogWarning("DoSometing WARN message");
        _logger?.LogError("DoSometing ERROR message");
        _logger?.LogCritical("DoSometing CRITICAL message");
    }
}
```

At this point, apart from the new *Microsoft.Extensions.Logging.Abstractions* dependency, nothing has changed from a caller perspective.

- If a program was calling `MyService` it still can do it after recompilation because the `ILogger` is optional.
- If `null` is passed for the `ÌLogger`, the service still works thanks to the null check operator `_logger`**?**`.LogInformation()` which is equivalent to:

```csharp
if (_logger != null)  _logger.LogInformation(...);
```

Note that if you are migrating an existing service from your code that  already has a constructor with parameters, your could pass an optional ILogger as the last parameter.

The ILogger passed as a parameter is perfectly suited for dependency injection. With dependency injection, we can register a concrete implementation of the Logger on the caller side. Let's do it the .Net way!

### 4. The caller application, using dependency injection

So, now we have a class library that is logging. How can connect the wires ?

*ASP.Net Core has already a DI container and could call our library directly. But this is very well documented yet and falls outside the scope of this article.*

We are going to create a simple Console App. Follow those steps, in order to setup dependency injection and logging:

- Create a new Console App,
- Add a project reference to `MyNetStandardLib`,
- Add a NuGet package `Microsoft.Extensions.DependencyInjection`. This is the Microsoft DI container. There are plenty of others (AutoFac, StructureMap, Ninject among others), but I'll stick to the .Net Foundation stack.

So let's change the `Program.cs` file from this:

```csharp
namespace TestConsole
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
        }
    }
}
```

to that:

```csharp
using Microsoft.Extensions.DependencyInjection;
using System;

namespace TestConsole
{
    // Program with Dependency Injection
    // Still no logging support!
    class Program
    {
        static void Main(string[] args)
        {
            // Setting up dependency injection
            var serviceCollection = new ServiceCollection();
            ConfigureServices(serviceCollection);
            var serviceProvider = serviceCollection.BuildServiceProvider();

            // Get an instance of the service
            var myService = serviceProvider.GetService<MyNetStandardLib.MyService>();

            // Call the service (logs are made here)
            myService.DoSomething();

            Console.ReadLine();
        }

        private static void ConfigureServices(ServiceCollection services)
        {
            // Register service from the library
            services.AddTransient<MyNetStandardLib.MyService>();
        }
    }
}
```

Let's explain:

- We create the DI container `ServicesCollection` and configure it in a separate method for clarity.
- We register our `MyService` as a Transient (ie: short lived instance within the calling code variable's scope)
- When we call `var myService = serviceProvider.GetService<MyNetStandardLib.MyService>();` the DI container will instanciate the `MyService` and will check the dependency tree for the object and pass required instances that are registered within the container. As we have still not registered any `ILogger`, a `null` will be passed.
- We can use the `myService` variable as usual, calling the `DoSomething` service

### 5. Add Logging support

We will add **Console** and **Debug** loggers. For a Console App, this will be useful because we will see logs in the console and in the *Output>Debug* window.

Add the following NuGet packages:

- `Microsoft.Extensions.Logging`
- `Microsoft.Extensions.Logging.Console`
- `Microsoft.Extensions.Logging.Debug`

Add the following `using` statements at the top of `Program.cs`

```csharp
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Logging.Console;
using Microsoft.Extensions.Logging.Debug;
```

Change the `ConfigureServices` method as follows:

```csharp
private static void ConfigureServices(ServiceCollection services)
{
    services.AddLogging(config =>
    {
        config.AddDebug(); // Log to debug (debug window in Visual Studio or any debugger attached)
        config.AddConsole(); // Log to console (colored !)
    })
    .Configure<LoggerFilterOptions>(options =>
    {
        options.AddFilter<DebugLoggerProvider>(null /* category*/ , LogLevel.Information /* min level */);
        options.AddFilter<ConsoleLoggerProvider>(null  /* category*/ , LogLevel.Warning /* min level */);
    })
    .AddTransient<MyNetStandardLib.MyService>(); // Register service from the library
}
```

Very, **very** neat and powerful. Let's explain:

- `services.AddLogging` allows us to add *various* loggers to the DI container and configure their options. Here we register the *Console* and *Debug* loggers with `AddConsole()` and `AddDebug()`: two extensions methods provided by their respective NuGet packages.
- Then we configure (optionally) their filter level.
  - The first parameter is the category: not very well documented, it acts as a *StartsWith* or *Contains* filter on the type name `T` of `ILogger<T>`, allowing to filter which logs we want to listen or mute.
  - The second parameter takes a `LogLevel`, allowing to listen only when log have a lower greater or equal than the specified parameter.

This configuration can also be made via [configuration files](https://docs.microsoft.com/fr-fr/aspnet/core/fundamentals/configuration/options?view=aspnetcore-2.2).
Now, every time a `ILogger<T>` is required, as in our `MyService` constructor, an instance will be injected, and this instance will log to the destinations registered.

### 6. Test run

After the Console App launch, this is what we see in the Console window. Note that only Warn, Error and Critical logs are written, as configured:

![Console output]({{site.baseurl}}/images/2019-04-27-console.png)

And this is what we see in the Debug window, note that Info level is also written, as configured:

![Debug output]({{site.baseurl}}/images/2019-04-27-debug.png)


## Conclusion

We have seen how powerful Dependency Injection is, and how we can build a library that supports logging without constraining the calling application.

You can find all the source files for this project here (Tested on VS2019, both Windows and Mac) : [https://github.com/xfischer/CoreLoggingTests](https://github.com/xfischer/CoreLoggingTests).


## Sources

Those articles helped me a lot, and it's good to cite them:

- Official MS doc : [Logging in ASP.NET Core](https://docs.microsoft.com/aspnet/core/fundamentals/logging)
- [Logging and Monitoring with .NET/C#](https://microsoft.github.io/code-with-engineering-playbook/Engineering/DevOpsLoggingDetailsCSharp.html)