---
title: "Rolling back migrations to a previous version of Entity Framework Core"
slug: "rolling-back-migrations-to-a-previous-version-of-entity-framework-core"
date: "2021-02-02"
summary: "I discovered there was a dependency incompatibility between EF Core 5 and SqlKata. I weighted my options, and decided it would be easier to revert back to EF Core 3, than to solve this dependency compatibility issue in another way."
---

#software-development #dotnet

During a month long refactor session on the Skyhop back-end I had upgraded all of my dependencies to their latest versions. Later during this process I discovered there was a dependency incompatibility between EF Core 5 and SqlKata. I weighted my options, and decided it would be easier to revert back to EF Core 3, than to solve this dependency compatibility issue in another way.

## The process

The process to do so had been surprisingly pleasant due to the following factors;

1. I use Git. Without it I would most certainly have been screwed.
2. Each time I create a new migration, I put the model changes and the migration code in different commits, which are easy to identify separately.
3. [I'm using a solution wide `Directory.Build.targets` for dependency version management.](https://www.strathweb.com/2018/07/solution-wide-nuget-package-version-handling-with-msbuild-15/)
4. I had not yet touched the production database, thus not requiring a rollback over there

It would be important to note that I was unable to roll back to a previous version due to changes to the API's between EF Core 3.x and 5.x, resulting in errors in some migrations, as well as errors in the model snapshot. As such, the process to do so;

1. Remove migrations which are scaffolded using the 5.x version
2. Grab the model snapshot from version control which is in the commit that also contains the current last available migration.
3. Change the global EF Core version number in the solution wide `Directory.Build.targets` back to the latest available version. (This would be a pain to do without)
4. Create a new migration using the current configuration.

## Additional findings
Despite having upgraded to .NET Core 5, it seems like EF Core 3, still works together nicely with this setup. For now I'll be waiting until this dependency issue is resolved before I try to upgrade once again.

