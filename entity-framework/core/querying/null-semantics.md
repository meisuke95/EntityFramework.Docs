---
title: Query null semantics in EF Core
description: Information on how Entity Framework Core handles null comparisons in queries 
author: maumar
ms.date: 11/11/2020
uid: core/querying/null-semantics
---
# Query null semantics

## Introduction

SQL databases operate on 3-valued logic (`true`, `false`, `null`) when performing comparisons, as opposed to boolean logic of C#. When translating LINQ queries to SQL, EF Core tries to compensate for the difference by introducing additional null checks for some elements of the query.
To illustrate this lets define the following entity:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/NullSemanticsEntity.cs#Entity)]

and issue several queries:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/Program.cs#BasicExamples)]

First two queries produce simple comparison. In case of the first query both columns are non-nullable so null checks are not needed. In case of the second query `NullableInt` could contain null, however `Id` is non-nullable. Comparing `null` to non-null yields `null` as a result, which would be filtered out by `WHERE` operation. No additional terms need to be added either.

```sql
SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE [e].[Id] = [e].[Int]

SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE [e].[Id] = [e].[NullableInt]
```

Third query introduces the null check. When `NullableInt` is `null` the comparison `Id <> NullableInt` yields `null`, which would be filtered out by `WHERE` operation. However from the boolean logic perspective this case should be returned as part of the result. Hence EF Core adds the necessary check to ensure that.

```sql
SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE ([e].[Id] <> [e].[NullableInt]) OR [e].[NullableInt] IS NULL
```

Queries four and five show the pattern when both columns are nullable. It's worth noting that `<>` operation produces significantly more complicated (and slower) query than equality operation.

```sql
SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE ([e].[String1] = [e].[String2]) OR ([e].[String1] IS NULL AND [e].[String2] IS NULL)

SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE (([e].[String1] <> [e].[String2]) OR ([e].[String1] IS NULL OR [e].[String2] IS NULL)) AND ([e].[String1] IS NOT NULL OR [e].[String2] IS NOT NULL)
```

## Null semantics in functions

Many functions in SQL can only return null result if some of their arguments are null. EF Core takes advantage of this to produce more efficient queries.
The query below illustrates the optimization:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/Program.cs#Functions)]

The generated SQL is as follows (we don't need to evaluate the `SUBSTRING` function):

```sql
SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE [e].[String1] IS NULL OR [e].[String2] IS NULL
```

This optimization can also be used for user defined functions. This can be done by adding `PropagatesNullability()` call to relevant function parameters model configuration.
To illustrate this, define two user functions inside the `DbContext`:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/NullSemanticsContext.cs#UdfBody)]

The model configuration (inside `OnModelCreating` method) is as follows:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/NullSemanticsContext.cs#UdfModelConfiguration)]

First function is configured in a standard way and the second function is configured to take advantage of the nullability propagation optimization.

When issuing the following queries:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/Program.cs#UdfExamples)]

We get this SQL:

```sql
SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE [dbo].[ConcatStrings]([e].[String1], [e].[String2]) IS NOT NULL

SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE [e].[String1] IS NOT NULL AND [e].[String2] IS NOT NULL
```

Just like with built-in `Substring` function, the second query doesn't need to evaluate the function itself to test it's nullability.

> [!NOTE]
> This optimization should only be used if the function can only return `null` becuase it's parameters are `null`.

## Optimizations and considerations

- Comparing non-nullable columns is simpler and faster than comparing nullable columns. Consider marking columns as non-nullable wherever it is possible.

- `Equal` comparison is simpler and faster than `NotEqual`, because query doesn't need to distinguish between `null` and `false` result. Consider using equality comparison whenever it is possible.

> [!NOTE]
> Wrapping `Equal` comparison around `Not` is effectively `NotEqual`. `Equal` comparison is only faster/simpler if it is not negated.

- In some cases it is possible to simplify a complex comparison by filtering out `null` values from a column explicitly - e.g. when no `null` values are present or these values are not relevant in the result. Consider the following example:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/Program.cs#ManualOptimization)]

These queries produce the following SQL:

```sql
SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE ((([e].[String1] <> [e].[String1]) OR [e].[String1] IS NULL) AND [e].[String1] IS NOT NULL) OR ((CAST(LEN([e].[String1]) AS int) = CAST(LEN([e].[String1]) AS int)) OR [e].[String1] IS NULL)

SELECT [e].[Id], [e].[Int], [e].[NullableInt], [e].[String1], [e].[String2]
FROM [Entities] AS [e]
WHERE [e].[String1] IS NOT NULL AND (CAST(LEN([e].[String1]) AS int) = CAST(LEN([e].[String1]) AS int))
```

In the second query, `null` results are filtered out from `String1` column explicitly. EF Core can safely treat the `String1` column as non-nullable when performing comparison, which results is simpler query.

## Using relational null semantics

It is possible to disable the null comparison compensation and use relational null semantics directly. This can be done by calling `UseRelationalNulls(true)` method on the options builder inside `OnConfiguring` method:

[!code-csharp[Main](../../../samples/core/Querying/NullSemantics/NullSemanticsContext.cs#UseRelationalNulls)]
