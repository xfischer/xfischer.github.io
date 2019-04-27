---
published: true
layout: post
title: Logging from a Library in .Net Core
comments: true
tags:
  - dotnet - core - logging - console
---
In this post, I'll try to explain how to setup logging (and dependency injection in the process) in a .Net Core library, and setup an application configuring listeners and filters as an example.

You'll see al lot of tutorials for ASP.Net Core, but it's hard to distinguish between what is bringed via the ASP.Net Core infrastructure (abeit much lighter than ASP.Net in the Full Framework), and what is natively inside the .Net Core framework itself.

There are also a lot of good logging frameworks available (like SeriLog, NLog, Log4Net) which gracefully integrates with the .Net Core stack.

# Why logging ?

## I usually log :

- For troubleshooting : I trace a progress of a task, cases encountered, dumps of in-memory structures : everything that will be useful if I need to know what was running and why.
- To give the consuming code a basic knowledge of the progress of their demand. Anything that could prevent the " black box " effect : waiting for something to complete with no feedback.
- At different levels (Trace, Debug, Info, Warn, Error, Critical) : it's very important to prioritize what's important and what's internal or critical.
- With *categories*, allowing logical grouping of messages.

**The rule-of-thumb is : read the logs, and you should know exactly what is going on inside the code.**

## Wait. This would be a LOT of logs !

If no one is listening, logging is harmless. You can setup  **log listeners**.

A listener can be registered and configured to listen to logs, with category and level filters. This listener then propagates log messages to a destination (Console, Debug window, ) Writing logs is one thing. Reading logs is something else.

# Show me the code

Let's setup :

- A library with a simple service. We want this service to log its activity.
- A Console app calling the library which configures logging listeners.

## 1) Create the class library *MyLibrary*

From Visual Studio, choose *New Project > Class Library*,
or with .Net CLI :
```
dotnet new classlib -n MyLibrary
```
This will create a base project.

# The problem
In the old .Net days, I have used a lot the ```System.Diagnostics.Trace``` class.

I have struggled for hours to setup this and I think it's good to share and transform the hours spent trying again and again in something useful to others.

# How logging works ?


{% github xfischer/CoreLoggingTests %}
