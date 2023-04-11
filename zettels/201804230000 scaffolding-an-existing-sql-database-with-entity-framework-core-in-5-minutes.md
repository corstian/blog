---
title: "Scaffolding an existing SQL database with Entity Framework Core in 5 minutes"
slug: "scaffolding-an-existing-sql-database-with-entity-framework-core-in-5-minutes"
date: "2018-04-23"
summary: ""
references: 
---

#software-development #dotnet #sql #data-storage

Sometimes it's nice to get a break from 'legacy' software. In this case we would like to get started using Identity Server 4 with an existing database running on SQL. Wouldn't it be nice to get up and running in a few minutes? Hold on.

We're using the dotnet cli for speed, and cross platform usefulness (*OS X, Windows(?) and Linux*). We assume you've booted your favorite terminal and you are in your solution folder. Buckle up buddy!

```bash
mkdir <YOUR_PROJECT_FOLDER>
cd <YOUR_PROJECT_FOLDER>
dotnet new classlib

dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
```

After these packages have been installed we need to add the following two lines to the `.csproj` file in the current folder. These are required in order to use the Entity Framework tooling from the command line. Use your favorite text editor:

```xml
<ItemGroup>
  <DotNetCliToolReference Include="Microsoft.EntityFrameworkCore.Tools.DotNet" Version="2.0.0" />
  <DotNetCliToolReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Tools" Version="2.0.0" />
</ItemGroup>
```

*Ps. make sure these 4 lines are somewhere within the `<Project>` tags, or grouped with other `<DotNetCliToolReference>` tags eventually already in your file.*

In order to be able to run migrations from this project we will configure it to be able to act as being startup project. There should be a line which is:

```xml
<TargetFramework>netstandard2.0</TargetFramework>
```

Change it to:

```xml
<TargetFrameworks>netcoreapp2.0;netstandard2.0</TargetFrameworks>
```

Just copy paste it for your speed, and sanity. Now you're ready to scaffold your data model from the database. In order to do so:

```bash
dotnet ef dbcontext scaffold "<CONNECTION STRING>" Microsoft.EntityFrameworkCore.SqlServer
```

> In case you get an error about the framework versions you need to install, just determine the current version by running the `dotnet --info` command and grabbing the value from under the `Microsoft .NET Core Shared Framework Host` line. Next add the following tag just under the `<TargetFramework>`&nbsp;tag and you're good to go.
>
>
> ```
> <RuntimeFrameworkVersion>2.0.5</RuntimeFrameworkVersion>
> ```
>
> *See [https://github.com/aspnet/EntityFrameworkCore/issues/10298](https://github.com/aspnet/EntityFrameworkCore/issues/10298) for more info about this issue.*

Congratulations! You should now have your existing data model ready to use within your .NET Core application using Entity Framework Core :)
