---
title: "Accessing the filesystem with an asp.net core app run on Docker"
slug: "accessing-the-file-system-with-asp-net-core-and-docker"
date: "2021-01-31"
summary: "I ran a bunch of experiments to figure out how to access the filesystem from a .NET app running on Docker."
---

#software-development #dotnet #data-storage

A recurring question when it comes to dealing with filesystems is "where do I put this file, and where can I find it?". This even more so when dealing with an app which is to be dockerized. How can I ensure these files can be accessed both during debug sessions, as well as when this app is running in Docker? To answer this question I have been fiddling around with an example project which you can find here, and the findings of which I describe in this post.


> A proof of concept of the contents I describe in this post can be found [in this GitHub repository](https://github.com/corstian/DockerFileSystemAccess).

## Accessing the build directory
Assuming I need to access a few files I have copied to my output directory I like to use the following code snippet;

```csharp
var path = Path.Combine(
    Path.GetDirectoryName(System.Reflection.Assembly.GetExecutingAssembly().Location),
    "files");
```

This path will most certainly return the build folder, which is `bin/Debug/netcoreapp3.1/files/`. And so it does when run in Docker, where the absolute path is `/app/bin/Debug/netcoreapp3.1/files/`. Though we'd be perfectly able to mount a folder over there, I'm reluctant to do so. The primary reason for which is that I'd break my app once I do a framework upgrade. Changing volume mounts is the last thing I'd think about in such situation.

## Using the `SpecialFolders` enum

There is a special enum which links to all kinds of special folders, which is the `Environment.SpecialFolders` enum. This enum can be resolved using the `Environment.GetFolderPath()` method. One thing to consider is that most of these enums do not resolve to directories when ran within Docker. The few which do are;

- `SpecialFolders.MyDocuments` (resolves to `/root` on Docker, `C:\Users\{userProfile}\Documents` on Windows)
- `SpecialFolders.CommonApplicationData` (resolves to `/usr/share` on Docker, `C:\ProgramData` on Windows)
- `SpecialFolders.UserProfile` (resolves to `/root` on Docker, `C:\Users\{userProfile}` on Windows)

Out of these folders I suppose the `CommonApplicationData` folder is the most appropriate one to use for storing data in a somewhat cross-platform compatible way. When checking the contents of this folder on Windows, it becomes evident each vendor neatly groups their stuff within their own folders, something which I'd suggest you to do, too.

## Mounting for shared access

When debugging stuff, it'd be nice to have access to the same contents, both when debugging locally, as well as when debugging on Docker. In order to do so I'd suggest mounting your application folder within the `ProgramData` directory to the `/usr/share` directory on Docker. Thankfully there are a few project properties we can configure in order to direct Visual Studio's container tools to run our Docker image with a few additional arguments. [Documentation for these properties can be found here.](https://docs.microsoft.com/en-us/visualstudio/containers/container-msbuild-properties?view=vs-2019).

The property we'd want to add is `<DockerfileRunArguments>` with the command to mount a local directory as follows ([see the documentation here](https://docs.docker.com/storage/bind-mounts/)):

```
--mount type=bind,source="C:/ProgramData/appName",target="/usr/share/appName"
```

Therefore the *.csproj file contains a section like this;

```
<PropertyGroup>
  <TargetFramework>netcoreapp3.1</TargetFramework>
  <DockerDefaultTargetOS>Linux</DockerDefaultTargetOS>
  <DockerfileRunArguments>--mount type=bind,source="C:/ProgramData/appName",target="/usr/share/appName"</DockerfileRunArguments>
</PropertyGroup>
```

When doing this you do not have to worry about creating files and so on, as that's automatically dealt with. During runtime you can simply drop a file in this directory, and have it picked up by your app running in Docker during runtime.

