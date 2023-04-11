---
title: "How to send emails from asp.net core using Razor templates?"
slug: "sending-transactional-emails-from-asp-net-core"
date: "2020-01-07"
summary: ""
references: 
---

#software-development #dotnet #email

> The code produced in this blog post is [published on GitHub](https://github.com/skyhop/Mail) as being a stand-alone library you can include in your project. The library itself can handle most of the email delivery needs, while you can bring your own views and view-models. Samples are available within the repository.
>
> You can find the package on NuGet [over here](https://www.nuget.org/packages/Skyhop.Mail).

## Introduction

Sending transactional email from my applications has always been one of my weakest points. Developing applications is easy. Writing api's is just an everyday task, and even utilizing raw sockets to implement some obscure real-time api is considered trivial when it comes to sending transactional emails from an application. I have been actively writing software for about nine years now. Professionally for about five. But email? Email is still one of the things I actively try to avoid. Emails are a true pain in the ass, at least for me. Doing a quick google search on sending emails with .NET yields plenty of results and blog posts, but none of them really usefull. They are all about how to use the SMTP protocol from within C#, and that just does not cut it for me. The solutions are not reusable, promote a shattered software architecture, and are in no way maintainable, nor usable in the first place.


In my professional career I have only written just a few email reports, and it seems as if these few email reports have the highest user interaction of everything I have ever built! How come? These reports contain highly specific information targeted towards the end users. These people rely on these messages for their day-to-day job. They have a busy life, and they just don't sit in front of their screens all day long. Having the information they need in their mail boxes when they need it is the best way to interact with these people and facilitate them in their day-to-day life in the least invasive manner. Think about emails what you want, but that's just how it is.

But there are plenty of reasons not to go this way:

- You can build a mobile app
- It drives traffic away from your main platform
- Mails are difficult

And for each of these arguments a counter argument exists:

- Mobile apps take lots of resources (mainly money and time) to get them right, and require an ongoing maintenance effort which just doesn't cut it for many small businesses. Besides that, yet another mobile app might also distract the end-user.
- I strongly believe software should exist to facilitate the end-user, and not to strike the ego's of their developers or owners, so changing traffic flows should not be an issue.
- Mails are indeed difficult, but can be incredibly powerfull with a little thought.

My mailbox has a chronic overflow, and I have to try my best to direct my attention to where it belongs. There is a clear distinction in the mail I get:

- **[Automated]** Crap.
- **[Automated]** Stuff I'm interested in. I open this, read a bit, and delete it because my mailbox is not supposed to be a storage for knowledge.
- **[Personal]** Stuff I'm not interested in. (Friend message me via more personal media anyway.)
- **[Personal]** Things I have to do.

The goal is to facilitate the end-user, and provide them with the information they want to have in their mailbox. Of all the automated emails I have ever gotten there is one message in particular which I have remembered:

![Email from freepik telling me I have been unsubscribed because I did not read their emails](/uploads/Annotation_2019_04_06_111019_67fd6c8d38.png)
*This email from freepik telling me I have been unsubscribed because I did not read their emails*

It's this seemingly small gesture which shows respect towards the people, and their inboxes which I really liked. It's not about flooding one's mailbox for views, but about facilitating end-users here, and I appreciate that. What is needed to achieve this level of quality?

Now there's just a small thing to note. Most ordinary computer users do not realize that companies can track when their emails are read and by whom. From this point of view I see that companies tend to be somewhat careful to expose their users to the knowledge that they (the company) has this kind of information about them. On the other hand I absolutely appreciate transparency. This practice happens. Companies do have that kind of information, so why not use it in a transparent way in which the user actually benefits from it?

In this article I want to explore the kind of system you need in order to get (real) good at sending emails.

## Technical overview

Sending emails is a total nightmare from a technical point of view. There are so many aspects which can make sending emails so incredibly difficult. To name some of these:

- You will need personalized information in each email
- At some point you'll probably want some kind of localization
- Emails need to be nicely formatted and look well across a wide range of devices and mail clients
- You need an unsubscribe system
- It must be easy to send emails from any place in your code base in order to send great emails from the places it matters the most.

This article will not cover all of these points. I will only try to give a few pointers into the right direction. You will need to decide for yourself where you integrate this kind of system, you have to provide the templates and content for the emails yourself, and I am not going to build the unsubscribe interface for you but I will try to guide you to solve some of these fundamental problems. What I will focus on are persionalization and integration. 

Localization is something which I will not cover in this post, but probably will tackle in a future blog post. Right now I have no incentive to implement localization within my own systems, but probably will do so some time in the future.

## Integration

The way how we send emails from our code base is one of the most important aspects. It's not about how to use the SMPT procol, or calling some API. We have to think about from what places in our code-base we want to send emails, and how we want to process this further.

One of the inherent properties of transactional emails is that it is most certainly tightly bound to a business process, and indirectly (*if you modelled your code after your business process*) to a technical process.

In the past I have already explored several options when it comes to rendering templates. One of the results of this work is documented [over here](/blog/2019-05-27/rendering-razor-views-by-supplying-a-model-instance), and I will build on that.

The core essence of this approach is that I do not want to have anything related to the template anywhere near the business logic which needs to send an email. The only thing which is reasonable is to fill a data model with the required data, and send that off. The previously mentioned blog-post is usefull as it describes a way to resolve a view for a given view-model, hence removing the requirement to specify a view which needs to be rendered.

> Even though a minor distraction, I have found myself creating bugs every time I have to do something with a hardcoded path, such as specifying a view. I think such tedious work is better left to a computer, so that humans can reason about what's going on from a more abstract point of view.

The resulting code with which I can send an email looks as follows:

```csharp
await _mailDispatcher.SendMail(
    data: new InvoiceModel
    {
        // Fill the viewmodel with appropriate data
    },
    to: new[] { new MimeKit.MailboxAddress($"{user.FirstName} {user.LastName}", user.Email) });
```


### Implementation

I don't feel like building my own email stack. There are excellent libraries available which abstract away some or most of the logic already. One of them is [MailKit](https://github.com/jstedfast/MailKit), which is based on MimeKit. It features a custom built SMTP client which in some way is better than the default SMTP client found within the .NET framework.

The true reasons however for using MailKit is their MessageBuilder class, with which one can reasonably compose a mail message. This is a part which I want to access both from the logic sending the message, as well as the template rendering the email itself. In order to achieve this I use a base model which I use for all the view-models meant for rendering a template. It's use is twofold as it allows both calling code as well as the template to access the `BodyBuilder` class, and it makes scanning assemblies for compiled templates easier. More on template compilation later.

This base class is also a convenient place to put the `MimeMessage` instance, which is basically the email we're about to send. The base class I use is as follows:

```csharp
public class MailBase
{
    public MimeMessage MailMessage { get; } = new MimeMessage();
    public BodyBuilder BodyBuilder { get; } = new BodyBuilder();
}
```

I will put hte MailMessage and BodyBuilder toghether in a processing step just before sending the message. Other things I have put in the 'pipeline' are:

- [PreMailer.NET](https://github.com/milkshakesoftware/PreMailer.Net), for inlining styles
- [HtmlAgilityPack](https://html-agility-pack.net/), for generating a lazy plain-text preview of the HTML email version. Not doing this might be a reason why email ends up in spam.

The code which puts this all together is as follows;

```csharp
public class MailDispatcher
{
    private readonly IRazorViewToStringRenderer _renderer;
    private readonly MailDispatcherOptions _options;

    public MailDispatcher(
        IRazorViewToStringRenderer renderer,
        MailDispatcherOptions options)
    {
        _renderer = renderer;
        _options = options;
    }

    public async Task SendMail<T>(
        T data,
        MailboxAddress from = default,
        MailboxAddress[] to = default,
        MailboxAddress[] cc = default,
        MailboxAddress[] bcc = default,
        MimeEntity[] attachments = default) where T : MailBase
    {
        // The GetViewForModel extension method is documented [here](/blog/2019-05-27/rendering-razor-views-by-supplying-a-model-instance)
        var body = await _renderer.GetViewForModel(data);

        // Inline some CSS
        body = PreMailer.Net.PreMailer
            .MoveCssInline(body)
            .Html;

        data.BodyBuilder.HtmlBody = body;

        HtmlDocument doc = new HtmlDocument();
        doc.LoadHtml(body);

        // Generate a text-only mail by stripping all html elements
        data.BodyBuilder.TextBody = string.Join(" ", doc.DocumentNode.SelectNodes("//text()").Select(q => q.InnerText));

        if (attachments != default && attachments.Any())
        {
            foreach (var attachment in attachments)
            {
                data.BodyBuilder.Attachments.Add(attachment);
            }
        }

        data.MailMessage.Body = data.BodyBuilder.ToMessageBody();

        if (from != default || _options.DefaultFromAddress != default) data.MailMessage.From.Add(from ?? _options.DefaultFromAddress);

        if (to != default && to.Any()) data.MailMessage.To.AddRange(to);
        if (cc != default && cc.Any()) data.MailMessage.Cc.AddRange(cc);
        if (bcc != default && bcc.Any()) data.MailMessage.Bcc.AddRange(bcc);

        _options.MailSender.Invoke(data.MailMessage);
    }
}

public class MailDispatcherOptions
{
    public MailboxAddress DefaultFromAddress { get; set; }
    public Action<MimeMessage> MailSender { get; set; }
}
```

### Registration
While you can choose to instantiate a new `MailDispatcher` instance every time you need one, I preferably register one instance with the dependency container as a singleton instance. This will prevent the quite heavy instantiation costs of the `RazorViewToStringRenderer`, which makes use of it's own dependency container so that it can be used with the generic host model used since .NET Core 3. For more information on that, see [this post ("Using the RazorViewToStringRenderer with Asp.Net Core 3")](/blog/2019-12-25/using-the-razorviewtostringrenderer-with-asp-net-core-3).

During registration it's possible to configure it's behaviour when it comes to the default sender address, and the email delivery method, for which you can bring your own. You can choose yourself whether you would like to use your own SMTP server, or would like to use a service such as SendGrid.

> In a prior life I have always had trouble delivering emails over SMTP. Whatever happened in between, I wasn't able to get it right. Currently I'm using the [`docker-mailserver` image](https://github.com/tomav/docker-mailserver) which contains everything required to self-host a fully functional mail server, and so far it works surprisingly good. If you're looking for something bigger, but still self-hosted, you might want to take a look at [Postal](https://github.com/postalhq/postal) for your email delivery needs.

Registering the `MailDispatcher` class with the DI container can be as simple as this;

```csharp
services.AddMailDispatpcher(builder =>
{
    builder.DefaultFromAddress = new MimeKit.MailboxAddress("Email Support", "support@example.tld");

    builder.MailSender = async message =>
    {
        using (var client = new SmtpClient())
        {
            await client.ConnectAsync("mail.example.tld", 587, false);
            await client.AuthenticateAsync("support@example.tld", "**ExamplePassword**");
            await client.SendAsync(message);
            await client.DisconnectAsync(true);
        }
    };
});
```


## Personalization

Now that the basic functionality is set up by which we can easily send emails, it's time to foucs on personalization. The great thing is that it's possible to do without too much distraction. The only things to focus on in this part are the views and the viewmodels, and thankfully context switches between these two aren't too heavy to digest. The limited mental capacity herein is part of what makes this approach so nice to use.

Given the email templates one would like to send vary heavily in between, I'll just try to provide some helpful advice for small bits and pieces which may be helpful.


### Runtime or compile time compilation?
As said before we'll build the templates using the Razor templating engine. This is great for two reasons, as we can recycle existing knowledge of the C# language, use the Visual Studio debugger right within our templates, and we can pre-compile them for runtime use. Note that there isn't the option to have runtime compilation with the approach documented in this post, though it is definitely possible to build. For a few starters on runtime compilation, check out the [introduction to this post](/blog/2019-12-25/using-the-razorviewtostringrenderer-with-asp-net-core-3) where I give a few pointers to the right resources.

Personally the lack of runtime compilation isn't as bad as it might sound. Especially not with the scope limited to transactional emails, which are mostly tightly related to code either way. As such, a change of email templates means a new deployment any way. Yet another side effect is that also the mail templates are tracked by version control, and there's just no way to shoot yourself in the foot with that.

### Project structure
Currently my approach to mail templates is to have a single project which contains all the views and viewmodels used for mail generation. This is primarilly due to some weird project mechanics which makes it difficult to add Razor views and compile-time compilation to any arbitrary .NET project. More on configuration later.

It is possible to use a layout just the same way as you would do with any other Razor project. As such, you can create a `Shared` folder with a `_Layout.cshtml` file, which contains your boilerplate code.

My project looks somewhat like the following folder structure: 

```
MailProject
|->Assets
|  |->Icon.png
|->Models
|  |->WelcomeMailModel.cs
|->Views
|  |->Shared
|  |  |->_Layout.cshtml
|  |->WelcomeMail.cshtml
```

### Creating a view?
There aren't really any requirements for a view, besides that I would recommend to use a model (how else would you resolve a view based on a viewmodel??), and the generic layout file. With this the basic structure looks as follows:

```csharp
@model WelcomeMailModel

@{
    Layout = "_Layout";
}

<span>Your email contents 😊</span>
```

### Embedding images?
Images can be displayed within an email in several different ways. One of them is to host the images on the web, and reference them. Changes are however, that in this situation images are not always displayed by default. If you create a custom endpoint to deliver images you can even track open rates, but hey, use this knowledge responsibly!

There are various ways to display images within emails, and [this article](https://blog.mailtrap.io/embedding-images-in-html-email-have-the-rules-changed/) neatly describes what approaches are possible, and how most mail clients may react to a certain approach.

In order to display images most of the time, an image needs to be embedded within the email itself, which can be done by adding it as an linked resource. This is something which I do within the template itself.

```csharp
var icon = Model.BodyBuilder.LinkedResources.Add("./Assets/Icon.png");

icon.ContentId = MimeUtils.GenerateMessageId();
```

After which the image can be used from markup as follows:

```html
<img src="cid:@icon.ContentId" />
```

## Installation
In order to save myself time setting up this logic every time I want to send mails using this approach I have abstracted most of this logic away in a small library [which is available on GitHub in the `skyhop/Mail` repository](https://github.com/skyhop/Mail) (find it on NuGet [over here](https://www.nuget.org/packages/Skyhop.Mail)). 

You can use the following commands, to make your life easier:

**Using the NuGet Package Manager**
```
Install-Package Skyhop.Mail
```

**Using the .NET CLI**
```
dotnet add package Skyhop.Mail
```


There are however a few gotchas you need to take care of when installing this library:

- You will need to create a separate project which contains your templates due to the requirement to use the `Microsoft.NET.Sdk.Razor` SDK target.
- You will need to add the `AddRazorSupportForMvc` element to your project file.
- It is assumed that the compiled assembly containing the Razor views has the default naming convention, which is `*.Views.dll`.

These requirements are also documented within the repository's readme file.


## What's next?
There's still one quite important topic to tackle, which is localization. As I personally do not have a high demand for localization on emails, I'm postponing this topic until later. Once I have a need for this I will try to find a way by which I can reuse existing localization methods, most are familiar with from use with asp.net. Given I have never used localization with asp.net, this would be a completely new topic to jump into.

For now I hope this approach might be beneficial to you, and please share your favourite techniques for generating content on the back-end!
