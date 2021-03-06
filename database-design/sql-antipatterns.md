---
layout: page
title: SQL Antipatterns
tags: [database-design]
---

This page contains a collection of SQL antipatterns, mostly collected from [the book called SQL Antipatterns](https://www.oreilly.com/library/view/sql-antipatterns/9781680500073/).

{% comment %}
Chapter 16 from the book.
{% endcomment %}
# Random selection

Frequently, we need a SQL query that returns a random result.
It is preferable to query the database for these random samples than to fetch all of the data and obtain the random samples in application code.
A very naive way of doing this is to sort randomly and pick the first row.

```sql
SELECT * FROM integer ORDER BY random() LIMIT 1
```

As expected, this cannot benefit from any index.
If the query planner does not understand the query intention, it might go as far as sorting all the rows of the table only to preserve a single row.

A possible simple solution that works when the rows have a contiguous key is to obtain a pseudo-random value in the range of this key and take the row whose key is equal to this value.

If there are missing key values, using the next higher value to the selected pseudo-random number is a bad solution, as it is likely to result in a non-uniform random selection.

Another bad solution is to fetch all primary keys, select one at random in application code, and fetch the corresponding row.
This solution is specially bad as the selected row might have been deleted between the two queries.

The proper solution is to rely on the nonstandard `LIMIT` clause supported by some database systems or in the `ROW_NUMBER` window function supported in some other database systems.

```sql
SELECT * FROM integer ORDER BY id
  LIMIT 1 OFFSET floor(random() * (SELECT count(*) FROM integer))
```

It is worth pointing out that some database brands, such as Microsoft SQL Server and Oracle, have proprietary solutions for this use case.

## Performance benchmarking

Some microbenchmarking shows that the problem is not so bad as the author of SQL Antipatterns points out. At least not in PostgreSQL 11.

The following table shows the results I obtained with [this script]({{ site.baseurl }}/assets/database-design/sql-antipatterns/random-selection.py), which uses a table of integers.
The difference might be more substantial for larger tables and tables with more data per row.

Rows       | Random sort | Random offset
----------:| -----------:|-------------:
        10 |    0.143 ms |      0.175 ms
       100 |    0.166 ms |      0.190 ms
     1,000 |    0.358 ms |      0.279 ms
    10,000 |    2.399 ms |      1.865 ms
   100,000 |   22.059 ms |     16.074 ms
 1,000,000 |  208.594 ms |    112.629 ms
10,000,000 | 2264.827 ms |   1393.280 ms

"Some queries cannot be optimized; take a different approach."

{% comment %}
Chapter 17 from the book.
{% endcomment %}
# Full-text search

Most applications that store and keep track of text eventually needs to search for words or phrases within the text.

The principle of SQL which dictates that values in columns are atomic goes against this requirement.

SQL provides some pattern-matching such as the LIKE predicate. However, many database systems also support regular expressions in a non-standard way.
It is worth noting, however, that SQL-99 defines the SIMILAR TO predicate for regular expressions, even if it is seldom implemented.

The first big problem with using these operators is that they perform full table scans. Furthermore, these queries will often return unwanted matches. For instance,

```sql
LIKE '%ONE%'
```

will match `MONEY`, `PRONE`, `ALONE`, and many other words.
The solution for this is to only look for matches in word boundaries, which can be difficult and even more expensive.

This pattern also suffers from poor scalability, so in new websites and applications the performance issues might not be evident from the start.

However, it is worthwhile to point out that in some cases these queries might be so infrequent that doing them in SQL using pattern matching might be the most cost-effective way of solving the problem.

MySQL, for instance, has full-text indexes. Oracle has also supported text-indexing features since its very early stages. SQL Server also supports full-text searching.
PostgreSQL has the `TSVECTOR` type to improve text search performance. SQLite has options for full-text search indexing.

The most straightforward and recommended solution is to use something other than SQL to perform full-text searches, as SQL was not designed with this in mind.
There are also independent products, such as Sphinx and Lucene, which can be used to perform full-text search.
However, keeping them up-to-date with the database can be a challenge.

## Rolling your own

You can also roll your own solution for this problem. This is usually done in the form of an inverted index, which is implemented with a keywords table and a table that relates this keywords to the texts they appear on.
Triggers can be used to guarantee that the keywords table and the inverted index are always up-to-date.

"You don't have to use SQL to solve every problem."

{% comment %}
Chapter 18 from the book.
{% endcomment %}
# Decreasing the number of SQL queries

A common mistake is trying to do too much with a single query.
It is not uncommon to run into Cartesian products from joins when trying to solve a complex problem in a single step.

Besides the fact that you can get wrong results because getting a complex query right is difficult, it's important to consider that these queries are simply hard to write, hard to modify, and hard to debug.

Writing simpler queries and computing their union and writing queries that build upon CTEs are a better way to perform complex queries.

"Although SQL makes it seem possible to solve a complex problem in a single line of code, don't be tempted to build a house of cards."
