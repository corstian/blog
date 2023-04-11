---
title: "Rendering Razor views by supplying a model instance"
slug: "rendering-razor-views-by-supplying-a-model-instance"
date: "2019-05-27"
summary: ""
---

#software-development #dotnet #email

[*A repository containing a working sample is available over at GitHub.*](https://github.com/corstian/RenderRazorViewByModel)

---

That you even found this post means your on to something. I don't know what your use-case is, but you probably do not want to use the things which are being discussed on this page. With that aside, there are (imho) some legit use-cases from a technical perspective for using the approaches described in this post.


## Introduction

Asp.net makes it quite trivial to render and output a razor template over http. When you want to have razor based templating functionalities, one approach would be to create an API interface which could return HTML, or anything else over http. As I want to reduce the complexity and overhead which comes with developing and maintaining this additional piece of infrastructure, my goal is to be able to render razor templates within an arbitrary .net (core) process.

This has been difficult as the razor templating library itself has been tightly knit together with asp.net until .net core came along which redefined application architecture within the .net ecosystem. Since .net core the razor library has been refactored out of asp.net, and it's now possible to integrate the templating engine with your own applications.

The aim of this post is to use Razor solely as a templating engine, not tied to ASP.NET, nor any other web technology. Personally I believe referencing views (by strings) in business logic is messy, and would want the output of a templating engine to be based on a model I provide. From this philosophy I started to figure out ways to abstract Razor views away from my business logic.

## The idea: rendering templates from code

One of the biggest problems I personally have with templates is the coupling between code and the template itself. In the wordt case scenario one would construct their template by means of string concatenation. This approach tightly couples the template with code, is prone to errors, and does not encourage code reuse. From this point on it can only get better.

> Let's be honest. Composing templates is an unwelcome distraction often found in the middle of business logic. It breaks the flow of the code, and the templating logic itself usually has no relationship with the surrounding logic. Doing so promotes tight coupling, makes code less maintainable, and one would preferably focus on templates separately from the code which is used to invoke these templates.

Once I start to imagine what a reasonable good approach to template rendering would be I get stuck on two primary rules:

1. Templates should be reusable
2. Templates should not directly be referenced in code

The first one is fairly easy to implement. The second one however is where it gets difficult. In my imagination I would use a data model to supply a template with the information it needs in order to render succesfully. This could be achieved by two different methods again:

- Based on naming conventions
- By coupling these somewhere

In this article we will build a mechanism where we can invoke razor templates by providing a model which is coupled to a certain view.

From a code perspective something like this is what we will be aiming for:

```csharp
string result = Templates.Render(new Model {
  // properties ...
});
```

*But... But. There's no view defined?!! This is exactly what one of the goals we're aiming for.*

## Preparation

In order to be able to pre compile views during build or publish you will need the Razor SDK. The steps required to prepare your project for this can be found [here](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/sdk). Important aspects involve changing the project SDK and installing package references from NuGet.

## Razor views

Razor views are just bits of code. Ahwell, after they are compiled that is. Lets decompile an assembly containing razor views to see what has become of those templates, after compilation that is.

Check out the following razor view:

```csharp
@model SkyHop.Mail.Models.Example

Hello @Model.Name, welcome to Razor World!
```

Now note that you will see several dll's in your output folder. Razor views will be compiled into an additional dll which is your assembly name postfixed with `.Views.dll`. In my case I will find a `SkyHop.Mail.Views.dll` file.

> In case you did not notice yet, my main use-case involves generating mail templates via this approach.

When decompiling the `.Views.dll` file, we will find our razor file from above as follows:

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Razor;
using Microsoft.AspNetCore.Mvc.Razor.Internal;
using Microsoft.AspNetCore.Mvc.Rendering;
using Microsoft.AspNetCore.Mvc.ViewFeatures;
using Microsoft.AspNetCore.Razor.Hosting;
using SkyHop.Mail.Models;

namespace AspNetCore
{
    // Token: 0x02000004 RID: 4
    [RazorSourceChecksum("SHA1", "42f4a043d58abac240193d49037790cb8105b327", "/Views/Test.cshtml")]
    public class Views_Test : RazorPage<Example>
    {
        // Token: 0x0600001D RID: 29 RVA: 0x000022A8 File Offset: 0x000004A8
        public override async Task ExecuteAsync()
        {
            this.BeginContext(35, 8, true);
            this.WriteLiteral("\r\nHello ");
            this.EndContext();
            this.BeginContext(44, 10, false);
            this.Write(base.Model.Name);
            this.EndContext();
            this.BeginContext(54, 27, true);
            this.WriteLiteral(", welcome to Razor World!\r\n");
            this.EndContext();
        }

        // Token: 0x1700000C RID: 12
        // (get) Token: 0x0600001E RID: 30 RVA: 0x000022EF File Offset: 0x000004EF
        // (set) Token: 0x0600001F RID: 31 RVA: 0x000022F7 File Offset: 0x000004F7
        [RazorInject]
        public IModelExpressionProvider ModelExpressionProvider { get; private set; }

        // Token: 0x1700000D RID: 13
        // (get) Token: 0x06000020 RID: 32 RVA: 0x00002300 File Offset: 0x00000500
        // (set) Token: 0x06000021 RID: 33 RVA: 0x00002308 File Offset: 0x00000508
        [RazorInject]
        public IUrlHelper Url { get; private set; }

        // Token: 0x1700000E RID: 14
        // (get) Token: 0x06000022 RID: 34 RVA: 0x00002311 File Offset: 0x00000511
        // (set) Token: 0x06000023 RID: 35 RVA: 0x00002319 File Offset: 0x00000519
        [RazorInject]
        public IViewComponentHelper Component { get; private set; }

        // Token: 0x1700000F RID: 15
        // (get) Token: 0x06000024 RID: 36 RVA: 0x00002322 File Offset: 0x00000522
        // (set) Token: 0x06000025 RID: 37 RVA: 0x0000232A File Offset: 0x0000052A
        [RazorInject]
        public IJsonHelper Json { get; private set; }

        // Token: 0x17000010 RID: 16
        // (get) Token: 0x06000026 RID: 38 RVA: 0x00002333 File Offset: 0x00000533
        // (set) Token: 0x06000027 RID: 39 RVA: 0x0000233B File Offset: 0x0000053B
        [RazorInject]
        public IHtmlHelper<Example> Html { get; private set; }
    }
}
```

This is great! Two things; one could find this class based on the filename through the `RazorSourceChecksumAttribute` which contains the filename, or one could look for the `T` in `RazorPage<T>` with a little bit of reflection.

## Wiring some bits together

> **A word of caution first:** I do not know anything about the internals of the razor engine. I have wired some things together which I found on the internet, and which were fairly easy to use, but I have the feeling it all might be optimized a little further. For now the performance is 'good enough' for me, and I will keep it there. If anyone feels like optimizing this code, go ahead and be sure to let me know!

Articles about rendering razor views into strings often reference to a file called [`RazorViewToStringRenderer.cs`](https://github.com/aspnet/Entropy/blob/master/samples/Mvc.RenderViewToString/RazorViewToStringRenderer.cs) which resides in the aspnet/Entropy repository on GitHub. There are various articles available already about how to use this class (see [this](https://scottsauber.com/2018/07/07/walkthrough-creating-an-html-email-template-with-razor-and-razor-class-libraries-and-rendering-it-from-a-net-standard-class-library/) or [this](https://codeopinion.com/using-razor-in-a-console-application/) among [others](https://www.google.com/search?q=RazorViewToStringRenderer)). The point is that the `RenderViewToStringAsync` method accepts a view name and model, and returns a the rendered template as a string.

The following code is where this all comes together.

```csharp
public static class Extensions
{
    public static async Task<string> GetViewForModel<T>(this IRazorViewToStringRenderer renderer, T model)
    {
        string path = Path.Combine(Path.GetFullPath("."), $"{typeof(Extensions).Namespace}.Views.dll");
        var assembly = Assembly.LoadFrom(path);

        var identifier = assembly
            .ExportedTypes
            .Single(q => q.IsSubclassOf(typeof(RazorPage<T>)))
            .GetCustomAttribute<RazorSourceChecksumAttribute>()
            .Identifier;

        var result = await renderer.RenderViewToStringAsync(identifier, model);

        return result;
    }
}
```

Note that you should check whether path to the `.Views.dll` file resolves correctly. In my case I have this class in the root of my class library.

Usage of the above snippet is as follows:

```csharp
// Where _renderer implements IRazorViewToStringRenderer
string body = await _renderer.GetViewForModel(data);
```

At this point we have made an important improvement over string based templating from code, and that is that we now connect the template with code instead of connecting code with the template. The template has become just an additional asset for our application which makes it much easier to work with a template alone. But this approach comes with an important convention, and that is that **a single model can only be used in a single view**!

For me, right now, this is not a big deal, and it will facilitate extending this model in order to support localization, for example. But as I'm not there yet that'll be covered another time.

[***A sample project for the method outlined in this post can be found over at GitHub.***](https://github.com/corstian/RenderRazorViewByModel)
