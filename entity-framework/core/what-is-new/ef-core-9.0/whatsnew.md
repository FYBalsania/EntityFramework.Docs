---
title: What's New in EF Core 9
description: Overview of new features in EF Core 9
author: ajcvickers
ms.date: 05/02/2024
uid: core/what-is-new/ef-core-9.0/whatsnew
---

# What's New in EF Core 9

EF Core 9 (EF9) is the next release after EF Core 8 and is scheduled for release in November 2024. See [_Plan for Entity Framework Core 9_](xref:core/what-is-new/ef-core-9.0/plan) for details.

EF9 is available as [daily builds](https://github.com/dotnet/efcore/blob/main/docs/DailyBuilds.md) which contain all the latest EF9 features and API tweaks. The samples here make use of these daily builds.

> [!TIP]
> You can run and debug into the samples by [downloading the sample code from GitHub](https://github.com/dotnet/EntityFramework.Docs). Each section below links to the source code specific to that section.

EF9 targets .NET 8, and can therefore be used with either [.NET 8 (LTS)](https://dotnet.microsoft.com/download/dotnet/8.0) or a [.NET 9 preview](https://dotnet.microsoft.com/download/dotnet/9.0).

> [!TIP]
> The _What's New_ docs are updated for each preview. All the samples are set up to use the [EF9 daily builds](https://github.com/dotnet/efcore/blob/main/docs/DailyBuilds.md), which usually have several additional weeks of completed work compared to the latest preview. We strongly encourage use of the daily builds when testing new features so that you're not doing your testing against stale bits.

<a name="cosmos"></a>

## Azure Cosmos DB for NoSQL

We are working on significant updates in EF9 to the EF Core database provider for Azure Cosmos DB for NoSQL.

### Role-based access

Azure Cosmos DB for NoSQL includes a [built-in role-based access control (RBAC) system](/azure/cosmos-db/role-based-access-control). This is now supported by EF9 for both management and use of containers. No changes are required to application code. See [Issue #32197](https://github.com/dotnet/efcore/issues/32197) for more information.

### Synchronous access blocked by default

> [!TIP]
> The code shown here comes from [CosmosSyncApisSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9.Cosmos/CosmosSyncApisSample.cs).

Azure Cosmos DB for NoSQL does not support synchronous (blocking) access from application code. Previously, EF masked this by default by blocking for you on async calls. However, this both encourages sync use, which is bad practice, and [may cause deadlocks](https://blog.stephencleary.com/2012/07/dont-block-on-async-code.html). Therefore, starting with EF9, an exception is thrown when synchronous access is attempted. For example:

```output
System.InvalidOperationException: An error was generated for warning 'Microsoft.EntityFrameworkCore.Database.SyncNotSupported':
 Azure Cosmos DB does not support synchronous I/O. Make sure to use and correctly await only async methods when using
 Entity Framework Core to access Azure Cosmos DB. See https://aka.ms/ef-cosmos-nosync for more information.
 This exception can be suppressed or logged by passing event ID 'CosmosEventId.SyncNotSupported' to the 'ConfigureWarnings'
 method in 'DbContext.OnConfiguring' or 'AddDbContext'.
   at Microsoft.EntityFrameworkCore.Diagnostics.EventDefinition.Log[TLoggerCategory](IDiagnosticsLogger`1 logger, Exception exception)
   at Microsoft.EntityFrameworkCore.Cosmos.Diagnostics.Internal.CosmosLoggerExtensions.SyncNotSupported(IDiagnosticsLogger`1 diagnostics)
   at Microsoft.EntityFrameworkCore.Cosmos.Storage.Internal.CosmosClientWrapper.DeleteDatabase()
   at Microsoft.EntityFrameworkCore.Cosmos.Storage.Internal.CosmosDatabaseCreator.EnsureDeleted()
   at Microsoft.EntityFrameworkCore.Infrastructure.DatabaseFacade.EnsureDeleted()
```

As the exception says, sync access can still be used for now by configuring the warning level appropriately. For example, in `OnConfiguring` on your `DbContext` type:

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder.ConfigureWarnings(b => b.Ignore(CosmosEventId.SyncNotSupported));
```

Note, however, that we plan to fully remove sync support in EF11, so start updating to use async methods like `ToListAsync` and `SaveChangesAsync` as soon as possible!

### Enhanced primitive collections

> [!TIP]
> The code shown here comes from [CosmosPrimitiveTypesSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9.Cosmos/CosmosPrimitiveTypesSample.cs).

The Cosmos DB provider has supported primitive collections in a limited form since EF Core 6. This is support is being enhanced in EF9, starting with consolidation of the metadata and API surfaces for primitive collections in document databases to align with primitive collections in relational databases. This means that primitive collections can now be explicitly mapped using the model building API, allowing for facets of the element type to be configured. For example, to map a list of required (i.e. non-null) strings:

<!--
            modelBuilder.Entity<Book>()
                .PrimitiveCollection(e => e.Quotes)
                .ElementType(b => b.IsRequired());
-->
[!code-csharp[ConfigureCollection](../../../../samples/core/Miscellaneous/NewInEFCore9.Cosmos/CosmosPrimitiveTypesSample.cs?name=ConfigureCollection)]

See [What's new in EF8: primitive collections](xref:core/what-is-new/ef-core-8.0/whatsnew#primitive-collections) for more information on the model building API.

## AOT and pre-compiled queries

As mentioned in the introduction, there is a lot of work going on behind the scenes to allow EF Core to run without just-in-time (JIT) compilation. Instead, EF compile ahead-of-time (AOT) everything needed to run queries in the application. This AOT compilation and related processing will happen as part of building and publishing the application. At this point in the EF9 release, there is not much available that can be used by you, the app developer. However, for those interested, the completed issues in EF9 that support AOT and pre-compiled queries are:

- [Compiled model: Use static binding instead of reflection for properties and fields](https://github.com/dotnet/efcore/issues/24900)
- [Compiled model: Generate lambdas used in change tracking](https://github.com/dotnet/efcore/issues/24904)
- [Make change tracking and the update pipeline compatible with AOT/trimming](https://github.com/dotnet/efcore/issues/29761)
- [Use interceptors to redirect the query to precompiled code](https://github.com/dotnet/efcore/issues/31331)
- [Make all SQL expression nodes quotable](https://github.com/dotnet/efcore/issues/33008)
- [Generate the compiled model during build](https://github.com/dotnet/efcore/issues/24894)
- [Discover the compiled model automatically](https://github.com/dotnet/efcore/issues/24893)
- [Make ParameterExtractingExpressionVisitor capable of extracting paths to evaluatable fragments in the tree](https://github.com/dotnet/efcore/issues/32999)
- [Generate expression trees in compiled models (query filters, value converters)](https://github.com/dotnet/efcore/issues/29924)
- [Make LinqToCSharpSyntaxTranslator more resilient to multiple declaration of the same variable in nested scopes](https://github.com/dotnet/efcore/issues/32716)
- [Optimize ParameterExtractingExpressionVisitor](https://github.com/dotnet/efcore/issues/32698)

Check back here for examples of how to use pre-compiled queries as the experience comes together.

## LINQ and SQL translation

The team is working on some significant architecture changes to the query pipeline in EF Core 9 as part of our continued improvements to JSON mapping and document databases. This means we need to get **people like you** to run your code on these new internals. (If you're reading a "What's New" doc at this point in the release, then you're a really engaged part of the community; thank you!) We have over 120,000 tests, but it's not enough! We need you, people running real code on our bits, in order to find issues and ship a solid release!

<a name="groupby-complex-types"></a>

### GroupBy complex types

> [!TIP]
> The code shown here comes from [ComplexTypesSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/ComplexTypesSample.cs).

EF9 supports grouping by a complex type instance. For example:

<!--
        var groupedAddresses = await context.Stores
            .GroupBy(b => b.StoreAddress)
            .Select(g => new { g.Key, Count = g.Count() })
            .ToListAsync();
-->
[!code-csharp[GroupByComplexType](../../../../samples/core/Miscellaneous/NewInEFCore9/ComplexTypesSample.cs?name=GroupByComplexType)]

EF translates this as grouping by each member of the complex type, which aligns with the semantics of complex types as value objects. For example, on Azure SQL:

```sql
SELECT [s].[StoreAddress_City], [s].[StoreAddress_Country], [s].[StoreAddress_Line1], [s].[StoreAddress_Line2], [s].[StoreAddress_PostCode], COUNT(*) AS [Count]
FROM [Stores] AS [s]
GROUP BY [s].[StoreAddress_City], [s].[StoreAddress_Country], [s].[StoreAddress_Line1], [s].[StoreAddress_Line2], [s].[StoreAddress_PostCode]
```

<a name="prune"></a>

### Prune columns passed to OPENJSON's WITH clause

> [!TIP]
> The code shown here comes from [JsonColumnsSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/JsonColumnsSample.cs).

EF9 removes unnecessary columns when calling `OPENJSON WITH`. For example, consider a query that obtains a count from a JSON collection using a predicate:

<!--
        #region PruneJSON
        var results = await context.Posts
            .Where(p => p.Metadata!.Updates.Count(e => e.UpdatedOn >= date) == 1)
            .ToListAsync();
-->
[!code-csharp[PruneJSON](../../../../samples/core/Miscellaneous/NewInEFCore9/JsonColumnsSample.cs?name=PruneJSON)]

In EF8, this query generates the following SQL when using the Azure SQL database provider:

```sql
SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata]
FROM [Posts] AS [p]
WHERE (
    SELECT COUNT(*)
    FROM OPENJSON([p].[Metadata], '$.Updates') WITH (
        [PostedFrom] nvarchar(45) '$.PostedFrom',
        [UpdatedBy] nvarchar(max) '$.UpdatedBy',
        [UpdatedOn] date '$.UpdatedOn',
        [Commits] nvarchar(max) '$.Commits' AS JSON
    ) AS [u]
    WHERE [u].[UpdatedOn] >= @__date_0) = 1
```

Notice that the `UpdatedBy`, and `Commits` are not needed in this query. Starting with EF9, these columns are now pruned away:

```sql
SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata]
FROM [Posts] AS [p]
WHERE (
    SELECT COUNT(*)
    FROM OPENJSON([p].[Metadata], '$.Updates') WITH (
        [PostedFrom] nvarchar(45) '$.PostedFrom',
        [UpdatedOn] date '$.UpdatedOn'
    ) AS [u]
    WHERE [u].[UpdatedOn] >= @__date_0) = 1
```

In some scenarios, this results in complete removal of the `WITH` clause. For example:

<!--
        #region PruneJSONPrimitive
        var tagsWithCount = await context.Tags.Where(p => p.Text.Length == 1).ToListAsync();
-->
[!code-csharp[PruneJSONPrimitive](../../../../samples/core/Miscellaneous/NewInEFCore9/JsonColumnsSample.cs?name=PruneJSONPrimitive)]

In EF8, this query translates to the following SQL:

```sql
SELECT [t].[Id], [t].[Text]
FROM [Tags] AS [t]
WHERE (
    SELECT COUNT(*)
    FROM OPENJSON([t].[Text]) WITH ([value] nvarchar(max) '$') AS [t0]) = 1
```

In EF9, this has been improved to:

```sql
SELECT [t].[Id], [t].[Text]
FROM [Tags] AS [t]
WHERE (
    SELECT COUNT(*)
    FROM OPENJSON([t].[Text]) AS [t0]) = 1
```

<a name="greatest"></a>

### Translations involving GREATEST/LEAST

> [!TIP]
> The code shown here comes from [LeastGreatestSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/LeastGreatestSample.cs).

Several new translations have been introduced that use the `GREATEST` and `LEAST` SQL functions.

> [!IMPORTANT]
> The `GREATEST` and `LEAST` functions were [introduced to SQL Server/Azure SQL databases in the 2022 version](https://techcommunity.microsoft.com/t5/azure-sql-blog/introducing-the-greatest-and-least-t-sql-functions/ba-p/2281726). Visual Studio 2022 installs SQL Server 2019 by default. We recommend installing [SQL Server Developer Edition 2022](https://www.microsoft.com/sql-server/sql-server-downloads) to try out these new translations in EF9.

For example, queries using `Math.Max` or `Math.Min` are now translated for Azure SQL using `GREATEST` and `LEAST` respectively. For example:

<!--
        #region MathMin
        var walksUsingMin = await context.Walks
            .Where(e => Math.Min(e.DaysVisited.Count, e.ClosestPub.Beers.Length) > 4)
            .ToListAsync();
-->
[!code-csharp[MathMin](../../../../samples/core/Miscellaneous/NewInEFCore9/LeastGreatestSample.cs?name=MathMin)]

This query is translated to the following SQL when using EF9 executing against SQL Server 2022:

```sql
SELECT [w].[Id], [w].[ClosestPubId], [w].[DaysVisited], [w].[Name], [w].[Terrain]
FROM [Walks] AS [w]
INNER JOIN [Pubs] AS [p] ON [w].[ClosestPubId] = [p].[Id]
WHERE LEAST((
    SELECT COUNT(*)
    FROM OPENJSON([w].[DaysVisited]) AS [d]), (
    SELECT COUNT(*)
    FROM OPENJSON([p].[Beers]) AS [b])) >
```

`Math.Min` and `Math.Max` can also be used on the values of a primitive collection. For example:

<!--
        #region MathMinPrimitive
        var pubsInlineMax = await context.Pubs
            .SelectMany(e => e.Counts)
            .Where(e => Math.Max(e, threshold) > top)
            .ToListAsync();
-->
[!code-csharp[MathMinPrimitive](../../../../samples/core/Miscellaneous/NewInEFCore9/LeastGreatestSample.cs?name=MathMinPrimitive)]

This query is translated to the following SQL when using EF9 executing against SQL Server 2022:

```sql
SELECT [c].[value]
FROM [Pubs] AS [p]
CROSS APPLY OPENJSON([p].[Counts]) WITH ([value] int '$') AS [c]
WHERE GREATEST([c].[value], @__threshold_0) > @__top_1
```

Finally, `RelationalDbFunctionsExtensions.Least` and `RelationalDbFunctionsExtensions.Greatest` can be used to directly invoke the `Least` or `Greatest` function in SQL. For example:

<!--
        #region Least
        var leastCount = await context.Pubs
            .Select(e => EF.Functions.Least(e.Counts.Length, e.DaysVisited.Count, e.Beers.Length))
            .ToListAsync();
-->
[!code-csharp[Least](../../../../samples/core/Miscellaneous/NewInEFCore9/LeastGreatestSample.cs?name=Least)]

This query is translated to the following SQL when using EF9 executing against SQL Server 2022:

```sql
SELECT LEAST((
    SELECT COUNT(*)
    FROM OPENJSON([p].[Counts]) AS [c]), (
    SELECT COUNT(*)
    FROM OPENJSON([p].[DaysVisited]) AS [d]), (
    SELECT COUNT(*)
    FROM OPENJSON([p].[Beers]) AS [b]))
FROM [Pubs] AS [p]
```

<a name="parameterization"></a>

### Force or prevent query parameterization

> [!TIP]
> The code shown here comes from [QuerySample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs).

Except in some special cases, EF Core parameterizes variables used in a LINQ query, but includes constants in the generated SQL. For example, consider the following query method:

<!--
        #region DefaultParameterization
        async Task<List<Post>> GetPosts(int id)
            => await context.Posts
                .Where(
                    e => e.Title == ".NET Blog" && e.Id == id)
                .ToListAsync();
-->
[!code-csharp[DefaultParameterization](../../../../samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs?name=DefaultParameterization)]

This translates to the following SQL and parameters when using Azure SQL:

```output
info: 2/5/2024 15:43:13.789 RelationalEventId.CommandExecuted[20101] (Microsoft.EntityFrameworkCore.Database.Command) 
      Executed DbCommand (1ms) [Parameters=[@__id_0='1'], CommandType='Text', CommandTimeout='30']
      SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata]
      FROM [Posts] AS [p]
      WHERE [p].[Title] = N'.NET Blog' AND [p].[Id] = @__id_0
```

Notice that EF created a constant in the SQL for ".NET Blog" because this value will not change from query to query. Using a constant allows this value to be examined by the database engine when creating a query plan, potentially resulting in a more efficient query.

On the other hand, the value of `id` is parameterized, since the same query may be executed with many different values for `id`. Creating a constant in this case results in pollution of the query cache with lots of queries that differ only in parameter values. This is very bad for overall performance of the database.

Generally speaking, these defaults should not be changed. However, EF Core 8.0.2 introduces an `EF.Constant` method which forces EF to use a constant even if a parameter would be used by default. For example:

<!--
        #region ForceConstant
        async Task<List<Post>> GetPostsForceConstant(int id)
            => await context.Posts
                .Where(
                    e => e.Title == ".NET Blog" && e.Id == EF.Constant(id))
                .ToListAsync();
-->
[!code-csharp[ForceConstant](../../../../samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs?name=ForceConstant)]

The translation now contains a constant for the `id` value:

```output
info: 2/5/2024 15:43:13.812 RelationalEventId.CommandExecuted[20101] (Microsoft.EntityFrameworkCore.Database.Command) 
      Executed DbCommand (1ms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata]
      FROM [Posts] AS [p]
      WHERE [p].[Title] = N'.NET Blog' AND [p].[Id] = 1
```

EF9 introduces the `EF.Parameter` method to do the opposite. That is, force EF to use a parameter even if the value is a constant in code. For example:

<!--
        #region ForceParameter
        async Task<List<Post>> GetPostsForceParameter(int id)
            => await context.Posts
                .Where(
                    e => e.Title == EF.Parameter(".NET Blog") && e.Id == id)
                .ToListAsync();
-->
[!code-csharp[ForceParameter](../../../../samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs?name=ForceParameter)]

The translation now contains a parameter for the ".NET Blog" string:

```output
info: 2/5/2024 15:43:13.803 RelationalEventId.CommandExecuted[20101] (Microsoft.EntityFrameworkCore.Database.Command)
      Executed DbCommand (1ms) [Parameters=[@__p_0='.NET Blog' (Size = 4000), @__id_1='1'], CommandType='Text', CommandTimeout='30']
      SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata]
      FROM [Posts] AS [p]
      WHERE [p].[Title] = @__p_0 AND [p].[Id] = @__id_1
```

<a name="inlinedsubs"></a>

### Inlined uncorrelated subqueries

> [!TIP]
> The code shown here comes from [QuerySample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs).

In EF8, an IQueryable referenced in another query may be executed as a separate database roundtrip. For example, consider the following LINQ query:

<!--
        #region InlinedSubquery
        var dotnetPosts = context
            .Posts
            .Where(p => p.Title.Contains(".NET"));

        var results = dotnetPosts
            .Where(p => p.Id > 2)
            .Select(p => new { Post = p, TotalCount = dotnetPosts.Count() })
            .Skip(2).Take(10)
            .ToArray();
-->
[!code-csharp[InlinedSubquery](../../../../samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs?name=InlinedSubquery)]

In EF8, the query for `dotnetPosts` is executed as one round trip, and then the final results are executed as second query. For example, on SQL Server:

```sql
SELECT COUNT(*)
FROM [Posts] AS [p]
WHERE [p].[Title] LIKE N'%.NET%'

SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata]
FROM [Posts] AS [p]
WHERE [p].[Title] LIKE N'%.NET%' AND [p].[Id] > 2
ORDER BY (SELECT 1)
OFFSET @__p_1 ROWS FETCH NEXT @__p_2 ROWS ONLY
```

In EF9, the `IQueryable` in the `dotnetPosts` is inlined, resulting in a single round trip:

```sql
SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata], (
    SELECT COUNT(*)
    FROM [Posts] AS [p0]
    WHERE [p0].[Title] LIKE N'%.NET%')
FROM [Posts] AS [p]
WHERE [p].[Title] LIKE N'%.NET%' AND [p].[Id] > 2
ORDER BY (SELECT 1)
OFFSET @__p_0 ROWS FETCH NEXT @__p_1 ROWS ONLY
```

<a name="hashsetasync"></a>

### New `ToHashSetAsync<T>` methods

> [!TIP]
> The code shown here comes from [QuerySample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs).

The <xref:System.Linq.Enumerable.ToHashSet%2A?displayProperty=nameWithType> methods have existed since .NET Core 2.0. In EF9, the equivalent async methods have been added. For example:

<!--
        #region ToHashSetAsync
        var set1 = await context.Posts
            .Where(p => p.Tags.Count > 3)
            .ToHashSetAsync();

        var set2 = await context.Posts
            .Where(p => p.Tags.Count > 3)
            .ToHashSetAsync(ReferenceEqualityComparer<Post>.Instance);
-->
[!code-csharp[ToHashSetAsync](../../../../samples/core/Miscellaneous/NewInEFCore9/QuerySample.cs?name=ToHashSetAsync)]

This enhancement was contributed by [@wertzui](https://github.com/wertzui). Many thanks!

## ExecuteUpdate and ExecuteDelete

<a name="executecomplex"></a>

### Allow passing complex type instances to ExecuteUpdate

> [!TIP]
> The code shown here comes from [ExecuteUpdateSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/ExecuteUpdateSample.cs).

The `ExecuteUpdate` API was introduced in EF7 to perform immediate, direct updates to the database without tracking or `SaveChanges`. For example:

<!--
        #region NormalExecuteUpdate
        await context.Stores
            .Where(e => e.Region == "Germany")
            .ExecuteUpdateAsync(s => s.SetProperty(b => b.Region, "Deutschland"));
-->
[!code-csharp[NormalExecuteUpdate](../../../../samples/core/Miscellaneous/NewInEFCore9/ExecuteUpdateSample.cs?name=NormalExecuteUpdate)]

Running this code executes the following query to update the `Region` to "Deutschland":

```sql
UPDATE [s]
SET [s].[Region] = N'Deutschland'
FROM [Stores] AS [s]
WHERE [s].[Region] = N'Germany'
```

In EF8 `ExecuteUpdate` can also be used to update values of complex type properties. However, each member of the complex type must be specified explicitly. For example:

<!--
            #region UpdateComplexTypeByMember
            var newAddress = new Address("Gressenhall Farm Shop", null, "Beetley", "Norfolk", "NR20 4DR");

            await context.Stores
                .Where(e => e.Region == "Deutschland")
                .ExecuteUpdateAsync(
                    s => s.SetProperty(b => b.StoreAddress.Line1, newAddress.Line1)
                        .SetProperty(b => b.StoreAddress.Line2, newAddress.Line2)
                        .SetProperty(b => b.StoreAddress.City, newAddress.City)
                        .SetProperty(b => b.StoreAddress.Country, newAddress.Country)
                        .SetProperty(b => b.StoreAddress.PostCode, newAddress.PostCode));
-->
[!code-csharp[UpdateComplexTypeByMember](../../../../samples/core/Miscellaneous/NewInEFCore9/ExecuteUpdateSample.cs?name=UpdateComplexTypeByMember)]

Running this code results in the following query execution:

```sql
UPDATE [s]
SET [s].[StoreAddress_PostCode] = @__newAddress_PostCode_4,
    [s].[StoreAddress_Country] = @__newAddress_Country_3,
    [s].[StoreAddress_City] = @__newAddress_City_2,
    [s].[StoreAddress_Line2] = NULL,
    [s].[StoreAddress_Line1] = @__newAddress_Line1_0
FROM [Stores] AS [s]
WHERE [s].[Region] = N'Deutschland'
```

In EF9, the same update can be performed by passing the complex type instance itself. That is, each member does not need to be explicitly specified. For example:

<!--
            #region UpdateComplexType
            var newAddress = new Address("Gressenhall Farm Shop", null, "Beetley", "Norfolk", "NR20 4DR");

            await context.Stores
                .Where(e => e.Region == "Germany")
                .ExecuteUpdateAsync(s => s.SetProperty(b => b.StoreAddress, newAddress));
-->
[!code-csharp[UpdateComplexType](../../../../samples/core/Miscellaneous/NewInEFCore9/ExecuteUpdateSample.cs?name=UpdateComplexType)]

Running this code results in the same query execution as the previous example:

```sql
UPDATE [s]
SET [s].[StoreAddress_City] = @__complex_type_newAddress_0_City,
    [s].[StoreAddress_Country] = @__complex_type_newAddress_0_Country,
    [s].[StoreAddress_Line1] = @__complex_type_newAddress_0_Line1,
    [s].[StoreAddress_Line2] = NULL,
    [s].[StoreAddress_PostCode] = @__complex_type_newAddress_0_PostCode
FROM [Stores] AS [s]
WHERE [s].[Region] = N'Germany'
```

Multiple updates to both complex type properties and simple properties can be combined in a single call to `ExecuteUpdate`. For example:

<!--
        #region UpdateMultipleComplexType
        await context.Customers
            .Where(e => e.Name == name)
            .ExecuteUpdateAsync(
                s => s.SetProperty(
                        b => b.CustomerInfo.WorkAddress, new Address("Gressenhall Workhouse", null, "Beetley", "Norfolk", "NR20 4DR"))
                    .SetProperty(b => b.CustomerInfo.HomeAddress, new Address("Gressenhall Farm", null, "Beetley", "Norfolk", "NR20 4DR"))
                    .SetProperty(b => b.CustomerInfo.Tag, "Tog"));
-->
[!code-csharp[UpdateMultipleComplexType](../../../../samples/core/Miscellaneous/NewInEFCore9/ExecuteUpdateSample.cs?name=UpdateMultipleComplexType)]

Running this code results in the same query execution as the previous example:

```sql
UPDATE [c]
SET [c].[CustomerInfo_Tag] = N'Tog',
    [c].[CustomerInfo_HomeAddress_City] = N'Beetley',
    [c].[CustomerInfo_HomeAddress_Country] = N'Norfolk',
    [c].[CustomerInfo_HomeAddress_Line1] = N'Gressenhall Farm',
    [c].[CustomerInfo_HomeAddress_Line2] = NULL,
    [c].[CustomerInfo_HomeAddress_PostCode] = N'NR20 4DR',
    [c].[CustomerInfo_WorkAddress_City] = N'Beetley',
    [c].[CustomerInfo_WorkAddress_Country] = N'Norfolk',
    [c].[CustomerInfo_WorkAddress_Line1] = N'Gressenhall Workhouse',
    [c].[CustomerInfo_WorkAddress_Line2] = NULL,
    [c].[CustomerInfo_WorkAddress_PostCode] = N'NR20 4DR'
FROM [Customers] AS [c]
WHERE [c].[Name] = @__name_0
```

## Migrations

<a name="temporal-migrations"></a>

### Improved temporal table migrations

The migration created when changing an existing table into a temporal table has been reduced in size for EF9. For example, in EF8 making a single existing table a temporal table results in the following migration:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AlterTable(
        name: "Blogs")
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "BlogsHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart");

    migrationBuilder.AlterColumn<string>(
        name: "SiteUri",
        table: "Blogs",
        type: "nvarchar(max)",
        nullable: false,
        oldClrType: typeof(string),
        oldType: "nvarchar(max)")
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "BlogsHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart");

    migrationBuilder.AlterColumn<string>(
        name: "Name",
        table: "Blogs",
        type: "nvarchar(max)",
        nullable: false,
        oldClrType: typeof(string),
        oldType: "nvarchar(max)")
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "BlogsHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart");

    migrationBuilder.AlterColumn<int>(
        name: "Id",
        table: "Blogs",
        type: "int",
        nullable: false,
        oldClrType: typeof(int),
        oldType: "int")
        .Annotation("SqlServer:Identity", "1, 1")
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "BlogsHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart")
        .OldAnnotation("SqlServer:Identity", "1, 1");

    migrationBuilder.AddColumn<DateTime>(
        name: "PeriodEnd",
        table: "Blogs",
        type: "datetime2",
        nullable: false,
        defaultValue: new DateTime(1, 1, 1, 0, 0, 0, 0, DateTimeKind.Unspecified))
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "BlogsHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart");

    migrationBuilder.AddColumn<DateTime>(
        name: "PeriodStart",
        table: "Blogs",
        type: "datetime2",
        nullable: false,
        defaultValue: new DateTime(1, 1, 1, 0, 0, 0, 0, DateTimeKind.Unspecified))
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "BlogsHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart");
}
```

In EF9, the same operation now results in a much smaller migration:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AlterTable(
        name: "Blogs")
        .Annotation("SqlServer:IsTemporal", true)
        .Annotation("SqlServer:TemporalHistoryTableName", "BlogsHistory")
        .Annotation("SqlServer:TemporalHistoryTableSchema", null)
        .Annotation("SqlServer:TemporalPeriodEndColumnName", "PeriodEnd")
        .Annotation("SqlServer:TemporalPeriodStartColumnName", "PeriodStart");

    migrationBuilder.AddColumn<DateTime>(
        name: "PeriodEnd",
        table: "Blogs",
        type: "datetime2",
        nullable: false,
        defaultValue: new DateTime(1, 1, 1, 0, 0, 0, 0, DateTimeKind.Unspecified))
        .Annotation("SqlServer:TemporalIsPeriodEndColumn", true);

    migrationBuilder.AddColumn<DateTime>(
        name: "PeriodStart",
        table: "Blogs",
        type: "datetime2",
        nullable: false,
        defaultValue: new DateTime(1, 1, 1, 0, 0, 0, 0, DateTimeKind.Unspecified))
        .Annotation("SqlServer:TemporalIsPeriodStartColumn", true);
}
```

## Model building

<a name="auto-compiled-models"></a>

### Auto-compiled models

> [!TIP]
> The code shown here comes from the [NewInEFCore9.CompiledModels](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9.CompiledModels/) sample.

Compiled models can improve startup time for applications with large models--that is entity type counts in the 100s or 1000s. In previous versions of EF Core, a compiled model had to be generated manually, using the command line. For example:

```dotnetcli
dotnet ef dbcontext optimize
```

After running the command, a line like, `.UseModel(MyCompiledModels.BlogsContextModel.Instance)` must be added to `OnConfiguring` to tell EF Core to use the compiled model.

Starting with EF9, this `.UseModel` line is no longer needed when the application's `DbContext` type is in the same project/assembly as the compiled model. Instead, the compiled model will be detected and used automatically. This can be seen by having EF log whenever it is building the model. Running a simple application then shows EF building the model when the application starts:

```output
Starting application...
>> EF is building the model...
Model loaded with 2 entity types.
```

The output from running `dotnet ef dbcontext optimize` on the model project is:

```output
PS D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\Model> dotnet ef dbcontext optimize

Build succeeded in 0.3s

Build succeeded in 0.3s
Build started...
Build succeeded.
>> EF is building the model...
>> EF is building the model...
Successfully generated a compiled model, it will be discovered automatically, but you can also call 'options.UseModel(BlogsContextModel.Instance)'. Run this command again when the model is modified.
PS D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\Model> 
```

Notice that the log output indicates that the _model was built when running the command_. If we now run the application again, after rebuilding but without making any code changes, then the output is:

```output
Starting application...
Model loaded with 2 entity types.
```

Notice that the model was not built when starting the application because the compiled model was detected and used automatically.

### MSBuild integration

With the above approach, the compiled model still needs to be regenerated manually when the entity types or `DbContext` configuration is changed. However, EF9 ships with MSBuild and targets package that can automatically update the compiled model when the model project is built! To get started, install the [Microsoft.EntityFrameworkCore.Tasks](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Tasks/) NuGet package. For example:

```dotnetcli
dotnet add package Microsoft.EntityFrameworkCore.Tasks --version 9.0.0-preview.4.24205.3
```

> [!TIP]
> Use the package version in the command above that matches the version of EF Core that you are using.

Then enable the integration by setting the `EFOptimizeContext` property to your `.csproj` file. For example:

```xml
<PropertyGroup>
    <EFOptimizeContext>true</EFOptimizeContext>
</PropertyGroup>
```

There are additional, optional, MSBuild properties for controlling how the model is built, equivalent to the options passed on the command line to `dotnet ef dbcontext optimize`. These include:

| MSBuild property   | Description                                                                                                                                                                                                     |
|--------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| EFOptimizeContext  | Set to `true` to enable auto-compiled models.                                                                                                                                                                   |
| DbContextName      | The DbContext class to use. Class name only or fully qualified with namespaces. If this option is omitted, EF Core will find the context class. If there are multiple context classes, this option is required. |
| EFStartupProject   | Relative path to the startup project. Default value is the current folder.                                                                                                                |
| EFTargetNamespace  | The namespace to use for all generated classes. Defaults to generated from the root namespace and the output directory plus CompiledModels.                                                                     |

In our example, we need to specify the startup project:

```xml
<PropertyGroup>
  <EFOptimizeContext>true</EFOptimizeContext>
  <EFStartupProject>..\App\App.csproj</EFStartupProject>
</PropertyGroup>
```

Now, if we build the project, we can see logging at build time indicating that the compiled model is being built:

```output
Optimizing DbContext...
dotnet exec --depsfile D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\App\bin\Release\net8.0\App.deps.json
  --additionalprobingpath G:\packages 
  --additionalprobingpath "C:\Program Files (x86)\Microsoft Visual Studio\Shared\NuGetPackages" 
  --runtimeconfig D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\App\bin\Release\net8.0\App.runtimeconfig.json G:\packages\microsoft.entityframeworkcore.tasks\9.0.0-preview.4.24205.3\tasks\net8.0\..\..\tools\netcoreapp2.0\ef.dll dbcontext optimize --output-dir D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\Model\obj\Release\net8.0\ 
  --namespace NewInEfCore9 
  --suffix .g 
  --assembly D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\Model\bin\Release\net8.0\Model.dll --startup-assembly D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\App\bin\Release\net8.0\App.dll 
  --project-dir D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\Model 
  --root-namespace NewInEfCore9 
  --language C# 
  --nullable 
  --working-dir D:\code\EntityFramework.Docs\samples\core\Miscellaneous\NewInEFCore9.CompiledModels\App 
  --verbose 
  --no-color 
  --prefix-output 
```

And running the application shows that the compiled model has been detected and hence the model is not built again:

```output
Starting application...
Model loaded with 2 entity types.
```

Now, whenever the model changes, the compiled model will be automatically rebuilt as soon as the project is built.

> [NOTE!]
> We are working through some performance issues with changes made to the compiled model in EF8 and EF9. See [Issue 33483#](https://github.com/dotnet/efcore/issues/33483) for more information.

<a name="read-only-primitives"></a>

### Read-only primitive collections

> [!TIP]
> The code shown here comes from [PrimitiveCollectionsSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/PrimitiveCollectionsSample.cs).

EF8 introduced support for [mapping arrays and mutable lists of primitive types](xref:core/what-is-new/ef-core-8.0/whatsnew#primitive-collections). This has been expanded in EF9 to include read-only collections/lists. Specifically, EF9 supports collections typed as `IReadOnlyList`, `IReadOnlyCollection`, or `ReadOnlyCollection`. For example, in the following code, `DaysVisited` will be mapped by convention as a primitive collection of dates:

```csharp
public class DogWalk
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ReadOnlyCollection<DateOnly> DaysVisited { get; set; }
}
```

The read-only collection can be backed by a normal, mutable collection if desired. For example, in the following code, `DaysVisited` can be mapped as a primitive collection of dates, while still allowing code in the class to manipulate the underlying list.

```csharp
    public class Pub
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public IReadOnlyCollection<string> Beers { get; set; }

        private List<DateOnly> _daysVisited = new();
        public IReadOnlyList<DateOnly> DaysVisited => _daysVisited;
    }
```

These collections can then be used in queries in the normal way. For example, this LINQ query:

<!--
        var walksWithADrink = await context.Walks.Select(
            w => new
            {
                WalkName = w.Name,
                PubName = w.ClosestPub.Name,
                Count = w.DaysVisited.Count(v => w.ClosestPub.DaysVisited.Contains(v)),
                TotalCount = w.DaysVisited.Count
            }).ToListAsync();
-->
[!code-csharp[WalksWithADrink](../../../../samples/core/Miscellaneous/NewInEFCore9/PrimitiveCollectionsSample.cs?name=WalksWithADrink)]

Which translates to the following SQL on SQLite:

```sql
SELECT "w"."Name" AS "WalkName", "p"."Name" AS "PubName", (
    SELECT COUNT(*)
    FROM json_each("w"."DaysVisited") AS "d"
    WHERE "d"."value" IN (
        SELECT "d0"."value"
        FROM json_each("p"."DaysVisited") AS "d0"
    )) AS "Count", json_array_length("w"."DaysVisited") AS "TotalCount"
FROM "Walks" AS "w"
INNER JOIN "Pubs" AS "p" ON "w"."ClosestPubId" = "p"."Id"
```

<a name="sequence-caching"></a>

### Specify caching for sequences

> [!TIP]
> The code shown here comes from [ModelBuildingSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/ModelBuildingSample.cs).

EF9 allows setting the [caching options for database sequences](/sql/t-sql/statements/create-sequence-transact-sql) for any relational database provider that supports this. For example, `UseCache` can be used to explicitly turn on caching and set the cache size:

<!--
            #region UseCache
            modelBuilder.HasSequence<int>("MyCachedSequence")
                .HasMin(10).HasMax(255000)
                .IsCyclic()
                .StartsAt(11).IncrementsBy(2)
                .UseCache(20);
-->
[!code-csharp[UseCache](../../../../samples/core/Miscellaneous/NewInEFCore9/ModelBuildingSample.cs?name=UseCache)]

This results in the following sequence definition when using SQL Server:

```sql
CREATE SEQUENCE [MyCachedSequence] AS int START WITH 11 INCREMENT BY 2 MINVALUE 10 MAXVALUE 255000 CYCLE CACHE 3;
```

Similarly, `UseNoCache` explicitly turns off caching:

<!--
            #region UseNoCache
            modelBuilder.HasSequence<int>("MyUncachedSequence")
                .HasMin(10).HasMax(255000)
                .IsCyclic()
                .StartsAt(11).IncrementsBy(2)
                .UseNoCache();
-->
[!code-csharp[UseNoCache](../../../../samples/core/Miscellaneous/NewInEFCore9/ModelBuildingSample.cs?name=UseNoCache)]

```sql
CREATE SEQUENCE [MyUncachedSequence] AS int START WITH 11 INCREMENT BY 2 MINVALUE 10 MAXVALUE 255000 CYCLE NO CACHE;
```

If neither `UseCache` or `UseNoCache` are called, then caching is not specified and the database will use whatever its default is. This may be a different default for different databases.

This enhancement was contributed by [@bikbov](https://github.com/bikbov). Many thanks!

<a name="fill-factor"></a>

### Specify fill-factor for keys and indexes

> [!TIP]
> The code shown here comes from [ModelBuildingSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/ModelBuildingSample.cs).

EF9 supports specification of the [SQL Server fill-factor](/sql/relational-databases/indexes/specify-fill-factor-for-an-index) when using EF Core Migrations to create keys and indexes. From the SQL Server docs, "When an index is created or rebuilt, the fill-factor value determines the percentage of space on each leaf-level page to be filled with data, reserving the remainder on each page as free space for future growth."

The fill-factor can be set on a single or composite primary and alternate keys and indexes. For example:

<!--
            #region FillFactor
            modelBuilder.Entity<User>()
                .HasKey(e => e.Id)
                .HasFillFactor(80);

            modelBuilder.Entity<User>()
                .HasAlternateKey(e => new { e.Region, e.Ssn })
                .HasFillFactor(80);

            modelBuilder.Entity<User>()
                .HasIndex(e => new { e.Name })
                .HasFillFactor(80);

            modelBuilder.Entity<User>()
                .HasIndex(e => new { e.Region, e.Tag })
                .HasFillFactor(80);
-->
[!code-csharp[FillFactor](../../../../samples/core/Miscellaneous/NewInEFCore9/ModelBuildingSample.cs?name=FillFactor)]

When applied to existing tables, this will alter the tables to the fill-factor to the constraint:

```sql
ALTER TABLE [User] DROP CONSTRAINT [AK_User_Region_Ssn];
ALTER TABLE [User] DROP CONSTRAINT [PK_User];
DROP INDEX [IX_User_Name] ON [User];
DROP INDEX [IX_User_Region_Tag] ON [User];

ALTER TABLE [User] ADD CONSTRAINT [AK_User_Region_Ssn] UNIQUE ([Region], [Ssn]) WITH (FILLFACTOR = 80);
ALTER TABLE [User] ADD CONSTRAINT [PK_User] PRIMARY KEY ([Id]) WITH (FILLFACTOR = 80);
CREATE INDEX [IX_User_Name] ON [User] ([Name]) WITH (FILLFACTOR = 80);
CREATE INDEX [IX_User_Region_Tag] ON [User] ([Region], [Tag]) WITH (FILLFACTOR = 80);
```

This enhancement was contributed by [@deano-hunter](https://github.com/deano-hunter). Many thanks!

<a name="conventions"></a>

### Make existing model building conventions more extensible

> [!TIP]
> The code shown here comes from [CustomConventionsSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/CustomConventionsSample.cs).

Public model building conventions for applications were [introduced in EF7](xref:core/modeling/bulk-configuration#Conventions). In EF9, we have made it easier to extend some of the existing conventions. For example, [the code to map properties by attribute in EF7](xref:core/what-is-new/ef-core-7.0/whatsnew#model-building-conventions) is this:

```csharp
public class AttributeBasedPropertyDiscoveryConvention : PropertyDiscoveryConvention
{
    public AttributeBasedPropertyDiscoveryConvention(ProviderConventionSetBuilderDependencies dependencies)
        : base(dependencies)
    {
    }

    public override void ProcessEntityTypeAdded(
        IConventionEntityTypeBuilder entityTypeBuilder,
        IConventionContext<IConventionEntityTypeBuilder> context)
        => Process(entityTypeBuilder);

    public override void ProcessEntityTypeBaseTypeChanged(
        IConventionEntityTypeBuilder entityTypeBuilder,
        IConventionEntityType? newBaseType,
        IConventionEntityType? oldBaseType,
        IConventionContext<IConventionEntityType> context)
    {
        if ((newBaseType == null
             || oldBaseType != null)
            && entityTypeBuilder.Metadata.BaseType == newBaseType)
        {
            Process(entityTypeBuilder);
        }
    }

    private void Process(IConventionEntityTypeBuilder entityTypeBuilder)
    {
        foreach (var memberInfo in GetRuntimeMembers())
        {
            if (Attribute.IsDefined(memberInfo, typeof(PersistAttribute), inherit: true))
            {
                entityTypeBuilder.Property(memberInfo);
            }
            else if (memberInfo is PropertyInfo propertyInfo
                     && Dependencies.TypeMappingSource.FindMapping(propertyInfo) != null)
            {
                entityTypeBuilder.Ignore(propertyInfo.Name);
            }
        }

        IEnumerable<MemberInfo> GetRuntimeMembers()
        {
            var clrType = entityTypeBuilder.Metadata.ClrType;

            foreach (var property in clrType.GetRuntimeProperties()
                         .Where(p => p.GetMethod != null && !p.GetMethod.IsStatic))
            {
                yield return property;
            }

            foreach (var property in clrType.GetRuntimeFields())
            {
                yield return property;
            }
        }
    }
}
```

In EF9, this can be simplified down to the following:

<!--
    #region AttributeBasedPropertyDiscoveryConvention
    public class AttributeBasedPropertyDiscoveryConvention(ProviderConventionSetBuilderDependencies dependencies)
        : PropertyDiscoveryConvention(dependencies)
    {
        protected override bool IsCandidatePrimitiveProperty(
            MemberInfo memberInfo, IConventionTypeBase structuralType, out CoreTypeMapping? mapping)
        {
            if (base.IsCandidatePrimitiveProperty(memberInfo, structuralType, out mapping))
            {
                if (Attribute.IsDefined(memberInfo, typeof(PersistAttribute), inherit: true))
                {
                    return true;
                }

                structuralType.Builder.Ignore(memberInfo.Name);
            }

            mapping = null;
            return false;
        }
    }
-->
[!code-csharp[AttributeBasedPropertyDiscoveryConvention](../../../../samples/core/Miscellaneous/NewInEFCore9/CustomConventionsSample.cs?name=AttributeBasedPropertyDiscoveryConvention)]

<a name="non-pub-apply"></a>

### Update ApplyConfigurationsFromAssembly to call non-public constructors

In previous versions of EF Core, the `ApplyConfigurationsFromAssembly` method only instantiated configuration types with a public, parameterless constructors. In EF9, we have both [improved the error messages generated when this fails](https://github.com/dotnet/efcore/pull/32577), and also enabled instantiation by non-public constructor. This is useful when co-locating configuration in a private nested class which should never be instantiated by application code. For example:

```csharp
public class Country
{
    public int Code { get; set; }
    public required string Name { get; set; }

    private class FooConfiguration : IEntityTypeConfiguration<Country>
    {
        private FooConfiguration()
        {
        }

        public void Configure(EntityTypeBuilder<Country> builder)
        {
            builder.HasKey(e => e.Code);
        }
    }
}
```

As an aside, some people think this pattern is an abomination because it couples the entity type to the configuration. Other people think it is very useful because it co-locates configuration with the entity type. Let's not debate this here. :-)

## SQL Server HierarchyId

> [!TIP]
> The code shown here comes from [HierarchyIdSample.cs](https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore9/HierarchyIdSample.cs).

<a name="hierarchyid-path-generation"></a>

### Sugar for HierarchyId path generation

First class support for the SQL Server `HierarchyId` type was [added in EF8](xref:core/what-is-new/ef-core-8.0/whatsnew#hierarchyid). In EF9, a sugar method has been added to make it easier to create new child nodes in the tree structure. For example, the following code queries for an existing entity with a `HierarchyId` property:

<!--
        #region HierarchyIdQuery
        var daisy = await context.Halflings.SingleAsync(e => e.Name == "Daisy");
-->
[!code-csharp[HierarchyIdQuery](../../../../samples/core/Miscellaneous/NewInEFCore9/HierarchyIdSample.cs?name=HierarchyIdQuery)]

This `HierarchyId` property can then be used to create child nodes without any explicit string manipulation. For example:

<!--
        #region HierarchyIdParse1
        var child1 = new Halfling(HierarchyId.Parse(daisy.PathFromPatriarch, 1), "Toast");
        var child2 = new Halfling(HierarchyId.Parse(daisy.PathFromPatriarch, 2), "Wills");
-->
[!code-csharp[HierarchyIdParse1](../../../../samples/core/Miscellaneous/NewInEFCore9/HierarchyIdSample.cs?name=HierarchyIdParse1)]

If `daisy` has a `HierarchyId` of `/4/1/3/1/`, then, `child1` will get the `HierarchyId` "/4/1/3/1/1/", and `child2` will get the `HierarchyId` "/4/1/3/1/2/".

To create a node between these two children, an additional sub-level can be used. For example:

<!--
        #region HierarchyIdParse2
        var child1b = new Halfling(HierarchyId.Parse(daisy.PathFromPatriarch, 1, 5), "Toast");
-->
[!code-csharp[HierarchyIdParse2](../../../../samples/core/Miscellaneous/NewInEFCore9/HierarchyIdSample.cs?name=HierarchyIdParse2)]

This creates a node with a `HierarchyId` of `/4/1/3/1/1.5/`, putting it bteween `child1` and `child2`.

This enhancement was contributed by [@Rezakazemi890](https://github.com/Rezakazemi890). Many thanks!

## Tooling

<a name="fewer-rebuilds"></a>

### Fewer rebuilds

The [`dotnet ef` command line tool](xref:core/cli/dotnet) by default builds your project before executing the tool. This is because not rebuilding before running the tool is a common source of confusion when things don't work. Experienced developers can use the `--no-build` option to avoid this build, which may be slow. However, even the `--no-build` option could cause the project to be re-built the next time it is built outside of the EF tooling.

We believe a [community contribution](https://github.com/dotnet/efcore/pull/32860) from [@Suchiman](https://github.com/Suchiman) has fixed this. However, we're also conscious that tweaks around MSBuild behaviors have a tendency to have unintended consequences, so we're asking people like you to try this out and report back on any negative experiences you have.