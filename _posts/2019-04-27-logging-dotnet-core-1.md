---
published: false
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

### 1. Starting point : a class with your `SimpleService`

Your service takes a param and simulates a long processing task for 1 second.

```csharp
using System;
using System.Threading;

namespace MyLibrary
{
    public class MySimpleService
    {
        public void DoSomething(string theParam)
        {
            // Simultate something me do for 1 second
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

- Add a constructor taking an `ILogger<SimpleService>`
- Add a backing field for the logger

```csharp
public class MySimpleService
{
    // backing field
    private readonly ILogger<MyService> _logger;
    
    // constructor
    public MySimpleService(ILogger<MyService> logger = null)
    {
      _logger = logger;
    }

    // ... DoSomething omitted for brievety
}
```

At this point nothing has changed in the implementation. If a program was calling `MyService`it still can do it after recompilation because the ILogger is optional.

If the service had already a constructor, your could pass the ILogger as the last parameter.

## Sources

Those articles helped me a lot, and it's good to cite them:

- Official MS doc : [Logging in ASP.NET Core](https://docs.microsoft.com/aspnet/core/fundamentals/logging)
- [Logging and Monitoring with .NET/C#](https://microsoft.github.io/code-with-engineering-playbook/Engineering/DevOpsLoggingDetailsCSharp.html)
