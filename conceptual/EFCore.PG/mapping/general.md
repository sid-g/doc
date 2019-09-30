# Type mapping

The EF Core provider transparently maps the types supported by Npgsql at the ADO.NET level - see [the Npgsql ADO type mapping page](/doc/types/basic.html).

This means that you can use PostgreSQL-specific types, such as `inet` or `circle`, directly in your entities. Simply define your properties just as if they were a simple type, such as a `string`:

```c#
public class MyEntity
{
    public int Id { get; set; }
    public string Name { get; set; }
    public IPAddress IPAddress { get; set; }
    public NpgsqlCircle Circle { get; set; }
    public int[] SomeInts { get; set; }
}
```

Special types such as [arrays](array.md) and [enums](enum.md) have their own documentation pages with more details.

[PostgreSQL composite types](https://www.postgresql.org/docs/current/static/rowtypes.html), while supported at the ADO.NET level, aren't yet supported in the EF Core provider. This is tracked by [#22](https://github.com/npgsql/Npgsql.EntityFrameworkCore.PostgreSQL/issues/22).

# Explicitly specifying data types

In some cases, your .NET property type can be mapped to several PostgreSQL data types; a good example is a `string`, which will be mapped to `text` by default, but can also be mapped to `jsonb`. You can explicitly specify the PostgreSQL data type by adding the following to your model's `OnModelCreating`:

```c#
builder.Entity<Blog>()
       .Property(b => b.SomeStringProperty)
       .HasColumnType("jsonb");
```

Or, if you prefer annotations, use a `ColumnAttribute`:

```c#
[Column(TypeName="jsonb")]
public string SomeStringProperty { get; set; }
```

## Operation translation to SQL

Entity Framework Core allows providers to translate query expressions to SQL for database evaluation. For example, PostgreSQL supports [regular expression operations](http://www.postgresql.org/docs/current/static/functions-matching.html#FUNCTIONS-POSIX-REGEXP), and the Npgsql EF Core provider automatically translates .NET's `Regex.IsMatch()` to use this feature. Since evaluation happens at the server, table data doesn't need to be transferred to the client (saving bandwidth), and in some cases indexes can be used to speed things up. The same C# code on other providers will trigger client evaluation.

Below are some Npgsql-specific translations, many additional standard ones are supported as well. See the other pages in the mapping section for more supported types and operations.

| C# expression                                              | SQL generated by Npgsql |
|------------------------------------------------------------|-------------------------|
| `.Where(c => Regex.IsMatch(c.Name, "^A+")`                 | [`WHERE "c"."Name" ~ '^A+'`](http://www.postgresql.org/docs/current/static/functions-matching.html#FUNCTIONS-POSIX-REGEXP)
| `.Where(c => EF.Functions.Like(c.Name, "foo%")`            | [`WHERE "c"."Name" LIKE 'foo%'`](https://www.postgresql.org/docs/current/static/functions-matching.html#FUNCTIONS-LIKE)
| `.Where(c => EF.Functions.ILike(c.Name, "foo%")`           | [`WHERE "c"."Name" ILIKE 'foo%'`](https://www.postgresql.org/docs/current/static/functions-matching.html#FUNCTIONS-LIKE) (case-insensitive LIKE)
| `.Select(c => EF.Functions.ToTsVector("english", c.Name))` | [`SELECT to_tsvector('english'::regconfig, "c"."Name")`](https://www.postgresql.org/docs/current/static/textsearch-controls.html#TEXTSEARCH-PARSING-DOCUMENTS)
| `.Select(c => EF.Functions.ToTsQuery("english", "pgsql"))` | [`SELECT to_tsquery('english'::regconfig, 'pgsql')`](https://www.postgresql.org/docs/current/static/textsearch-controls.html#TEXTSEARCH-PARSING-QUERIES)
| `.Where(c => c.SearchVector.Matches("Npgsql"))`            | [`WHERE "c"."SearchVector" @@ 'Npgsql'`](https://www.postgresql.org/docs/current/static/textsearch-intro.html#TEXTSEARCH-MATCHING)
| `.Select(c => EF.Functions.ToTsQuery(c.SearchQuery).ToNegative())` | [`SELECT (!! to_tsquery("c"."SearchQuery"))`](https://www.postgresql.org/docs/current/static/textsearch-features.html#TEXTSEARCH-MANIPULATE-TSQUERY)
| `.Select(c => EF.Functions.ToTsVector(c.Name).SetWeight(NpgsqlTsVector.Lexeme.Weight.A))` | [`SELECT setweight(to_tsvector("c"."Name"), 'A')`](https://www.postgresql.org/docs/current/static/textsearch-features.html#TEXTSEARCH-MANIPULATE-TSVECTOR)