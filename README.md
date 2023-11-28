# pgvector-dotnet

[pgvector](https://github.com/pgvector/pgvector) support for .NET (C# and F#)

Supports [Npgsql](https://github.com/npgsql/npgsql), [Npgsql.FSharp](https://github.com/Zaid-Ajaj/Npgsql.FSharp), [Dapper](https://github.com/DapperLib/Dapper), and [Entity Framework Core](https://github.com/dotnet/efcore)

[![Build Status](https://github.com/pgvector/pgvector-dotnet/workflows/build/badge.svg?branch=master)](https://github.com/pgvector/pgvector-dotnet/actions)

## Getting Started

Follow the instructions for your database library:

- [Npgsql](#npgsql)
- [Npgsql.FSharp](#npgsqlfsharp)
- [Dapper](#dapper)
- [Entity Framework Core](#entity-framework-core)

Or check out an example:

- [Embeddings](tests/Pgvector.Tests/OpenAITests.cs) with OpenAI

## Npgsql

Run

```sh
dotnet add package Pgvector
```

Create a connection

```csharp
var dataSourceBuilder = new NpgsqlDataSourceBuilder(connString);
dataSourceBuilder.UseVector();
await using var dataSource = dataSourceBuilder.Build();

var conn = dataSource.OpenConnection();
```

Enable the extension

```csharp
await using (var cmd = new NpgsqlCommand("CREATE EXTENSION IF NOT EXISTS vector", conn))
{
    await cmd.ExecuteNonQueryAsync();
}

conn.ReloadTypes();
```

Create a table

```csharp
await using (var cmd = new NpgsqlCommand("CREATE TABLE items (id serial PRIMARY KEY, embedding vector(3))", conn))
{
    await cmd.ExecuteNonQueryAsync();
}
```

Insert a vector

```csharp
await using (var cmd = new NpgsqlCommand("INSERT INTO items (embedding) VALUES ($1)", conn))
{
    var embedding = new Vector(new float[] { 1, 1, 1 });
    cmd.Parameters.AddWithValue(embedding);
    await cmd.ExecuteNonQueryAsync();
}
```

Get the nearest neighbors

```csharp
await using (var cmd = new NpgsqlCommand("SELECT * FROM items ORDER BY embedding <-> $1 LIMIT 5", conn))
{
    var embedding = new Vector(new float[] { 1, 1, 1 });
    cmd.Parameters.AddWithValue(embedding);

    await using (var reader = await cmd.ExecuteReaderAsync())
    {
        while (await reader.ReadAsync())
        {
            Console.WriteLine((Vector)reader.GetValue(0));
        }
    }
}
```

Add an approximate index

```csharp
await using (var cmd = new NpgsqlCommand("CREATE INDEX ON items USING ivfflat (embedding vector_l2_ops) WITH (lists = 100)", conn))
{
    await cmd.ExecuteNonQueryAsync();
}
```

Use `vector_ip_ops` for inner product and `vector_cosine_ops` for cosine distance

See a [full example](https://github.com/pgvector/pgvector-dotnet/blob/master/tests/Pgvector.Tests/NpgsqlTests.cs)

## Npgsql.FSharp

Run

```sh
dotnet add package Pgvector
```

Import the library

```fsharp
open Pgvector
```

Create a connection

```fsharp
let dataSourceBuilder = new NpgsqlDataSourceBuilder(connString)
dataSourceBuilder.UseVector()
use dataSource = dataSourceBuilder.Build()
```

Enable the extension

```fsharp
dataSource
|> Sql.fromDataSource
|> Sql.query "CREATE EXTENSION IF NOT EXISTS vector"
|> Sql.executeNonQuery
```

Create a table

```fsharp
dataSource
|> Sql.fromDataSource
|> Sql.query "CREATE TABLE items (id serial PRIMARY KEY, embedding vector(3))"
|> Sql.executeNonQuery
```

Insert a vector

```fsharp
let embedding = new Vector([| 1f; 1f; 1f |])
let parameter = new NpgsqlParameter("", embedding)

dataSource
|> Sql.fromDataSource
|> Sql.query "INSERT INTO items (embedding) VALUES (@embedding)"
|> Sql.parameters [ "embedding", Sql.parameter parameter ]
|> Sql.executeNonQuery
```

Get the nearest neighbors

```fsharp
type Item = {
    Id: int
    Embedding: Vector
}

dataSource
|> Sql.fromDataSource
|> Sql.query "SELECT * FROM items ORDER BY embedding <-> @embedding LIMIT 5"
|> Sql.parameters [ "embedding", Sql.parameter parameter ]
|> Sql.execute (fun read ->
    {
        Id = read.int "id"
        Embedding = read.fieldValue<Vector> "embedding"
    })
|> printfn "%A"
```

Add an approximate index

```fsharp
dataSource
|> Sql.fromDataSource
|> Sql.query "CREATE INDEX ON items USING hnsw (embedding vector_l2_ops)"
|> Sql.executeNonQuery
```

Use `vector_ip_ops` for inner product and `vector_cosine_ops` for cosine distance

See a [full example](https://github.com/pgvector/pgvector-dotnet/blob/master/tests/Pgvector.FSharp.Tests/NpgsqlFSharpTests.fs)

## Dapper

Run

```sh
dotnet add package Pgvector.Dapper
```

Import the library

```csharp
using Pgvector.Dapper;
```

Create a connection

```csharp
SqlMapper.AddTypeHandler(new VectorTypeHandler());

var dataSourceBuilder = new NpgsqlDataSourceBuilder(connString);
dataSourceBuilder.UseVector();
await using var dataSource = dataSourceBuilder.Build();

var conn = dataSource.OpenConnection();
```

Enable the extension

```csharp
conn.Execute("CREATE EXTENSION IF NOT EXISTS vector");
conn.ReloadTypes();
```

Define a class

```csharp
public class Item
{
    public int Id { get; set; }
    public Vector? Embedding { get; set; }
}
```

Create a table

```csharp
conn.Execute("CREATE TABLE items (id serial PRIMARY KEY, embedding vector(3))");
```

Insert a vector

```csharp
var embedding = new Vector(new float[] { 1, 1, 1 });
conn.Execute(@"INSERT INTO items (embedding) VALUES (@embedding)", new { embedding });
```

Get the nearest neighbors

```csharp
var embedding = new Vector(new float[] { 1, 1, 1 });
var items = conn.Query<Item>("SELECT * FROM items ORDER BY embedding <-> @embedding LIMIT 5", new { embedding });
foreach (Item item in items)
{
    Console.WriteLine(item.Embedding);
}
```

Add an approximate index

```csharp
conn.Execute("CREATE INDEX ON items USING ivfflat (embedding vector_l2_ops) WITH (lists = 100)");
// or
conn.Execute("CREATE INDEX ON items USING hnsw (embedding vector_l2_ops)");
```

Use `vector_ip_ops` for inner product and `vector_cosine_ops` for cosine distance

See a [full example](https://github.com/pgvector/pgvector-dotnet/blob/master/tests/Pgvector.Tests/DapperTests.cs)

## Entity Framework Core

Run

```sh
dotnet add package Pgvector.EntityFrameworkCore
```

The latest version works with .NET 8. For .NET 6 and 7, use version 0.1.2 and [this readme](https://github.com/pgvector/pgvector-dotnet/blob/efcore-v0.1.2/README.md#entity-framework-core).

Import the library

```csharp
using Pgvector.EntityFrameworkCore;
```

Enable the extension

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasPostgresExtension("vector");
}
```

Configure the connection

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseNpgsql("connString", o => o.UseVector());
}
```

Define a model

```csharp
public class Item
{
    public int Id { get; set; }

    [Column(TypeName = "vector(3)")]
    public Vector? Embedding { get; set; }
}
```

Insert a vector

```csharp
ctx.Items.Add(new Item { Embedding = new Vector(new float[] { 1, 1, 1 }) });
ctx.SaveChanges();
```

Get the nearest neighbors

```csharp
var embedding = new Vector(new float[] { 1, 1, 1 });
var items = await ctx.Items
    .OrderBy(x => x.Embedding!.L2Distance(embedding))
    .Take(5)
    .ToListAsync();

foreach (Item item in items)
{
    if (item.Embedding != null)
    {
        Console.WriteLine(item.Embedding);
    }
}
```

Also supports `MaxInnerProduct` and `CosineDistance`

Get the distance

```csharp
var items = await ctx.Items
    .Select(x => new { Entity = x, Distance = x.Embedding!.L2Distance(embedding) })
    .ToListAsync();
```

Get items within a certain distance

```csharp
var items = await ctx.Items
    .Where(x => x.Embedding!.L2Distance(embedding) < 5)
    .ToListAsync();
```

Add an approximate index

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Item>()
        .HasIndex(i => i.Embedding)
        .HasMethod("ivfflat")
        .HasOperators("vector_l2_ops")
        .HasStorageParameter("lists", 100);
    // or
    modelBuilder.Entity<Item>()
        .HasIndex(i => i.Embedding)
        .HasMethod("hnsw")
        .HasOperators("vector_l2_ops")
        .HasStorageParameter("m", 16)
        .HasStorageParameter("ef_construction", 64);
}
```

Use `vector_ip_ops` for inner product and `vector_cosine_ops` for cosine distance

See a [full example](https://github.com/pgvector/pgvector-dotnet/blob/master/tests/Pgvector.Tests/EntityFrameworkCoreTests.cs)

## History

- [Pgvector](https://github.com/pgvector/pgvector-dotnet/blob/master/src/Pgvector/CHANGELOG.md)
- [Pgvector.Dapper](https://github.com/pgvector/pgvector-dotnet/blob/master/src/Pgvector.Dapper/CHANGELOG.md)
- [Pgvector.EntityFrameworkCore](https://github.com/pgvector/pgvector-dotnet/blob/master/src/Pgvector.EntityFrameworkCore/CHANGELOG.md)

## Contributing

Everyone is encouraged to help improve this project. Here are a few ways you can help:

- [Report bugs](https://github.com/pgvector/pgvector-dotnet/issues)
- Fix bugs and [submit pull requests](https://github.com/pgvector/pgvector-dotnet/pulls)
- Write, clarify, or fix documentation
- Suggest or add new features

To get started with development:

```sh
git clone https://github.com/pgvector/pgvector-dotnet.git
cd pgvector-dotnet
createdb pgvector_dotnet_test
dotnet test
```
