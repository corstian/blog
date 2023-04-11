
---
title: "Converting .NET Ticks to MSSQL DateTime and back"
slug: "converting-net-ticks-to-mssql-datetime-and-back"
date: "2018-09-21"
summary: ""
---

#software-development #dotnet #sql

Given my background with .NET technology, and me being stupid enough to store the DateTime struct's Ticks value in a SQL database I found the need to convert these Ticks to a DateTime again, in order to visualize the stuff going on in the database.

Let me just drop the code here. **It's not mine, but I'm having trouble finding it on the internet every time I need it.**

**The script to convert ticks (long value) to datetime2:**

```sql
SET ANSI_NULLS ON
GO
CREATE FUNCTION dbo.ToDateTime2 ( @Ticks bigint )
    RETURNS datetime2
AS
BEGIN
    DECLARE @DateTime datetime2 = '00010101';
    SET @DateTime = DATEADD( DAY, @Ticks / 864000000000, @DateTime );
    SET @DateTime = DATEADD( SECOND, ( @Ticks % 864000000000) / 10000000, @DateTime );
    RETURN DATEADD( NANOSECOND, ( @Ticks % 10000000 ) * 100, @DateTime );
END
GO
```

After you created the function you can use it as:

```sql
SELECT dbo.ToDateTime2(`Table`.`Ticks`) AS `TimeStamp` FROM `Table`
```

**The script to convert a datetime2 value to ticks again:**

```sql
SET ANSI_NULLS ON
GO

CREATE FUNCTION [dbo].[Ticks] (@dt DATETIME)
    RETURNS BIGINT
WITH SCHEMABINDING
AS
BEGIN
    DECLARE @year INT = DATEPART(yyyy, @dt)
    DECLARE @month INT = DATEPART(mm, @dt)
    DECLARE @day INT = DATEPART(dd, @dt)
    DECLARE @hour INT = DATEPART(hh, @dt)
    DECLARE @min INT = DATEPART(mi, @dt)
    DECLARE @sec INT = DATEPART(ss, @dt)

    DECLARE @days INT =
        CASE @month - 1
            WHEN 0 THEN 0
            WHEN 1 THEN 31
            WHEN 2 THEN 59
            WHEN 3 THEN 90
            WHEN 4 THEN 120
            WHEN 5 THEN 151
            WHEN 6 THEN 181
            WHEN 7 THEN 212
            WHEN 8 THEN 243
            WHEN 9 THEN 273
            WHEN 10 THEN 304
            WHEN 11 THEN 334
            WHEN 12 THEN 365
        END

    IF  @year % 4 = 0 AND (@year % 100  != 0 OR (@year % 100 = 0 AND @year % 400 = 0)) AND @month > 2 BEGIN
        SET @days = @days + 1
    END
    RETURN CONVERT(bigint,
        ((((((((@year - 1) * 365) + ((@year - 1) / 4)) - ((@year - 1) / 100)) + ((@year - 1) / 400)) + @days) + @day) - 1) * 864000000000) +
        ((((@hour * 3600) + CONVERT(bigint, @min) * 60) + CONVERT(bigint, @sec)) * 10000000) + (CONVERT(bigint, DATEPART(ms, @dt)) * CONVERT(bigint,10000));

END
GO
```

Which can be used in the following way again:

```sql
SELECT dbo.Ticks(`Table`.`TimeStamp`) AS `Ticks` FROM `Table`
```

 

***In case you figure out the source of these scripts, please pass me a message and I'll give proper credit***
