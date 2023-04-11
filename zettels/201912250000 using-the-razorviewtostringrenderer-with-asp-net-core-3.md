---
title: "Using the RazorViewToStringRenderer with Asp.Net Core 3"
slug: "using-the-razorviewtostringrenderer-with-asp-net-core-3"
date: "2019-12-25"
summary: ""
---

#software-development #dotnet #email

There are several articles detailing how one can render Razor views without the whole ASP.NET Core web hosting infrastructure. Some of these articles are detailing how to use the `RazorViewToStringRenderer` approach as first seen [in this repo](https://github.com/aspnet/Entropy/blob/master/samples/Mvc.RenderViewToString/RazorViewToStringRenderer.cs). Another approach is to manually use the `RazorProjectEngine` to compile templates on runtime. Both approaches have their down and upsides.

The main distinction between these two approaches is the way they deal with templates. In this regard the `RazorViewToStringRenderer` is ideally used to render pre-compiled razor templates, while the `RazorProjectEngine` approach is best used in situations where dynamic compilation is required. Think for example about templates loaded from a database or another dynamic data-source.

Additionally the `RazorViewToStringRenderer` supports rendering complex templates more easily because the dependencies are properly compiled at compile time (no pun intended). Having missing dependencies using the `RazorProjectEngine` method is one of the more tricky issues to solve consistently.

This post details the instantiation of the `RazorViewToStringRenderer` using Asp.Net Core 3. Excellent samples for both approaches can be found in the answers to [this question on StackOverflow](https://stackoverflow.com/questions/38247080/using-razor-outside-of-mvc-in-net-core/38253402).


## Implementing the `RazorViewToStringRenderer`

There are a few subtle differences when migrating from .NET Core 2.x to .NET Core 3.x which make it so that the `RazorViewToStringRenderer` does not work anymore. Issues which might be rising is that certain (DI) dependencies are missing, or that the views cannot be found.

The best way to use the `RazorViewToStringRenderer` is to register it with the DI container, and let the DI container resolve all dependencies. The most straightforward way to achieve this is to use the `WebHostBuilder`. See the following code:

```csharp
private static RazorViewToStringRenderer GetRenderer()
{
    var services = new ServiceCollection();
    
    var appDirectory = Directory.GetCurrentDirectory();

    var webhostBuilder = new WebHostBuilder().ConfigureServices(services =>
    {
        services.Configure<MvcRazorRuntimeCompilationOptions>(options =>
        {
            options.FileProviders.Add(new PhysicalFileProvider(appDirectory));
        });

        services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();

        var diagnosticSource = new DiagnosticListener("Microsoft.AspNetCore");
        services.AddSingleton<DiagnosticListener>(diagnosticSource);
        services.AddSingleton<DiagnosticSource>(diagnosticSource);

        services.AddLogging();

        services.AddRazorPages();

        services.AddSingleton<RazorViewToStringRenderer>();
    }).UseStartup<Startup>().Build();

    return webhostBuilder.Services.GetRequiredService<RazorViewToStringRenderer>();
}

internal class Startup
{
    public void Configure()
    {

    }
}
```

The alternative to the `GetRenderer` method is to register the `RazorViewToStringRenderer` within the standard DI container of your app, but this only works when you have a web application due to a dependency on an `IWebHostEnvironment` implementation. I have not looked into ways around this, and for portability reasons I construct my own `IWebHost` instance from which I pull the required dependency. For performance reasons I make sure that one instance to the `RazorViewToStringRenderer` instance is cached.


### About the changes
One problem when migrating to .NET Core 3 is that the `IRazorViewEngineOptions` class does not exist anymore. It has been pointed out [on GitHub](https://github.com/aspnet/AspNetCore.Docs/issues/14593#issuecomment-538659633) that instead of the `IRazorViewEngineOptions`, the `MvcRazorRuntimeCompilationOptions` class should be used to register file providers.

The other problem there was related to the environment, which has changed with the hosting model to the generic host. These changes have been reflected in the sample code up above.


## Application

One of my favourite applications for the `RazorViewToStringRenderer` is for use with model-to-view approach as outlined [in this post (called "*Rendering Razor views by supplying a model instance*")](/blog/2019-05-27/rendering-razor-views-by-supplying-a-model-instance). The essence there is that I use reflection to dig through the compiled view assemblies to be able to automagically render a template based on the supplied viewmodel.


Happy hacking!
