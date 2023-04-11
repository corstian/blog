---
title: "Solution wide config files with .NET Core"
slug: "solution-wide-config-files-with-dotnet-core"
date: "2020-01-15"
summary: ""
references: 
---

#software-development #dotnet

Configuration is difficult, and confusing, and one of the first things to become a difficult thing to deal with once a project really starts to grow. Before you know it you have to store connection strings, api keys for multiple dependencies, logging settings and so on.

Comparing the current situation with how it was back in the days of the .NET Framework, I think it has really improved. Now we have the `appsettings.json` files in which we can put everything we like, and which we can easily query by injecting the `IConfiguration` object.

What was not clear to me was how to share a single configuration file with all the individual projects within a solution, and that's exactly what this post is about!

> **Proper disclaimer:**
> I have no idea whether these are best practices. Personally I'm just looking for a way by which I can keep my configuration maintainable in a larger solution, where most of the projects have somewhat the same configuration. I haven't really worked with secret configurations, and I know that it's been discouraged to store secrets within configuration files, but that's what I've been up to now, and that's probably how I'll be doing it until someone points me to a fairly straightforward way to maintain it all.


## Structure

First of all I create a json file which I appropriately call `commonsettings.json`, in the root of the solution, after which I add this file as a solution item for quick and easy access.

## Linking the file to the projects

This is probably the most important aspect of this whole post; upon compilation we want to distribute the files to the output directory as being content, but the file itself does not live in the project folder. For this we can alter the `.csproj` file to include specific content items.

For me these so called linked files are declared in the following way:

```xml
<ItemGroup>
    <Content Include="$(SolutionDir)commonsettings.json" CopyToOutputDirectory="PreserveNewest" LinkBase="\" />
    <Content Include="$(SolutionDir)commonsettings.Development.json" CopyToOutputDirectory="PreserveNewest" LinkBase="\" />
    <Content Include="$(SolutionDir)commonsettings.DockerCompose.json" CopyToOutputDirectory="PreserveNewest" LinkBase="\" />
</ItemGroup>
```

From which we get to including the files.

## Registering the files with the configuration container

We can directly use the generic host builder to register these custom configuration files as follows:

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((hostingContext, config) =>
        {
            config
                .SetBasePath(Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location))
                .AddJsonFile("commonsettings.json", optional: true)
                .AddJsonFile($"commonsettings.{hostingContext.HostingEnvironment.EnvironmentName}.json", optional: true);
        });
```

One fairly important tidbit is to specify base path where these config files can be found. If not done, your files will not be found.

For usability's sake we add two variants. One file with is `commonsettings.json`, and one which is the `commonsettings.{environment}.json` variant. This last one can be used to apply a different configuration dependent on the environment. As such I personally like to run two different configuration files. One for development (local debug), and one for running my solution with docker-compose, as it requires different connection strings to find the required services. The docker compose configuration is one which could be deployed to production, if it weren't for the fact that I apply production secrets via environment variables, and don't run all production services via docker-compose.

It's not like the configuration itself became any less complex, it's more like the configuration files do not need to be copied in between projects anymore 😎.

## Selecting the correct environment via environment variables

One very interesting and unexpected thing I ran against was the way an environment is selected, and how this differs whether you're running an .net Core application, or you're running an asp.net Core application. The difference is in the environment variable you'd have to set:

**For asp.net Core:**
Set the `ASPNETCORE_ENVIRONMENT` variable to the environment you'd wish to use.

**For .NET Core apps:**
Set the `DOTNET_ENVIRONMENT` variable to the environment you'd wish to use.

