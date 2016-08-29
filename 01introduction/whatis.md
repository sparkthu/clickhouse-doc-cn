什么是ClickHouse
===================
ClickHouse是一个列式的OLAP数据库管理系统
在通常的面向行的数据库中，数据是按照这样的顺序存储：

<pre> 
5123456789123456789     1       Eurobasket - Greece - Bosnia and Herzegovina - example.com      1       2011-09-01 01:03:02     6274717   1294101174      11409   612345678912345678      0       33      6       http://www.example.com/basketball/team/123/match/456789.html http://www.example.com/basketball/team/123/match/987654.html       0       1366    768     32      10      3183      0       0       13      0\0     1       1       0       0                       2011142 -1      0               0       01321     613     660     2011-09-01 08:01:17     0       0       0       0       utf-8   1466    0       0       0       5678901234567890123               277789954       0       0       0       0       0
5234985259563631958     0       Consulting, Tax assessment, Accounting, Law       1       2011-09-01 01:03:02     6320881   2111222333      213     6458937489576391093     0       3       2       http://www.example.ru/         0       800     600       16      10      2       153.1   0       0       10      63      1       1       0       0                       2111678 000       0       588     368     240     2011-09-01 01:03:17     4       0       60310   0       windows-1251    1466    0       000               778899001       0       0       0       0       0
...
</pre>

换句话说，一行的所有数据是连续存储的。MySQL，Postgres,SQL Server等都是典型的行式数据库。
在列式数据库中，数据是这样存储的：
<pre class="text-example" style="white-space: pre; ">
<b>WatchID:</b>    5385521489354350662     5385521490329509958     5385521489953706054     5385521490476781638     5385521490583269446     5385521490218868806     5385521491437850694   5385521491090174022      5385521490792669254     5385521490420695110     5385521491532181574     5385521491559694406     5385521491459625030     5385521492275175494   5385521492781318214      5385521492710027334     5385521492955615302     5385521493708759110     5385521494506434630     5385521493104611398
<b>JavaEnable:</b> 1       0       1       0       0       0       1       0       1       1       1       1       1       1       0       1       0       0       1       1
<b>Title:</b>      Yandex  Announcements - Investor Relations - Yandex     Yandex — Contact us — Moscow    Yandex — Mission        Ru      Yandex — History — History of Yandex    Yandex Financial Releases - Investor Relations - Yandex Yandex — Locations      Yandex Board of Directors - Corporate Governance - Yandex       Yandex — Technologies
<b>GoodEvent:</b>  1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1
<b>EventTime:</b>  2016-05-18 05:19:20     2016-05-18 08:10:20     2016-05-18 07:38:00     2016-05-18 01:13:08     2016-05-18 00:04:06     2016-05-18 04:21:30     2016-05-18 00:34:16     2016-05-18 07:35:49     2016-05-18 11:41:59     2016-05-18 01:13:32
...
</pre>

这些例子仅仅是用来展示数据是怎样排布的。
不同列的数据时分开存储的，而同一列的数据存储在一起。典型的列式数据库有： Vertica, Paraccel (Actian Matrix) (Amazon Redshift), Sybase IQ, Exasol, Infobright, InfiniDB, MonetDB (VectorWise) (Actian Vector), LucidDB, SAP HANA, Google Dremel, Google PowerDrill, Druid, kdb+ 等。
Different orders for storing data are better suited to different scenarios.
The data access scenario refers to what queries are made, how often, and in what proportion; how much data is read for each type of query - rows, columns, and bytes; the relationship between reading and updating data; the working size of the data and how locally it is used; whether transactions are used, and how isolated they are; requirements for data replication and logical integrity; requirements for latency and throughput for each type of query, and so on.

The higher the load on the system, the more important it is to customize the system to the scenario, and the more specific this customization becomes. There is no system that is equally well-suited to significantly different scenarios. If a system is adaptable to a wide set of scenarios, under a high load, the system will handle all the scenarios equally poorly, or will work well for just one of the scenarios.

We'll say that the following is true for the OLAP (online analytical processing) scenario:

- The vast majority of requests are for read access.
- Data is updated in fairly large batches (> 1000 rows), not by single rows; or it is not updated at all.
- Data is added to the DB but is not modified.
- For reads, quite a large number of rows are extracted from the DB, but only a small subset of columns.
- Tables are "wide," meaning they contain a large number of columns.
- Queries are relatively rare (usually hundreds of queries per server or less per second).
- For simple queries, latencies around 50 ms are allowed.
- Column values are fairly small -  numbers and short strings (for example, 60 bytes per URL).
- Requires high throughput when processing a single query (up to billions of rows per second per server).
- There are no transactions.
- Low requirements for data consistency.
- There is one large table per query. All tables are small, except for one.
- A query result is significantly smaller than the source data. That is, data is filtered or aggregated. The result fits in a single server's RAM.

It is easy to see that the OLAP scenario is very different from other popular scenarios (such as OLTP or Key-Value access). So it doesn't make sense to try to use OLTP or a Key-Value DB for processing analytical queries if you want to get decent performance. For example, if you try to use MongoDB or Elliptics for analytics, you will get very poor performance compared to OLAP databases.

Columnar-oriented databases are better suited to OLAP scenarios (at least 100 times better in processing speed for most queries), for the following reasons:

1. For I/O.
    -  For an analytical query, only a small number of table columns need to be read. In a column-oriented database, you can read just the data you need. For example, if you need 5 columns out of 100, you can expect a 20-fold reduction in I/O.
    - Since data is read in packets, it is easier to compress. Data in columns is also easier to compress. This further reduces the I/O volume.
    - Due to the reduced I/O, more data fits in the system cache.

 For example, the query "count the number of records for each advertising platform" requires reading one "advertising platform ID" column, which takes up 1 byte uncompressed. If most of the traffic was not from advertising platforms, you can expect at least 10-fold compression of this column. When using a quick compression algorithm, data decompression is possible at a speed of at least several gigabytes of uncompressed data per second. In other words, this query can be processed at a speed of approximately several billion rows per second on a single server. This speed is actually achieved in practice.
Show example

2. For CPU.

Since executing a query requires processing a large number of rows, it helps to dispatch all operations for entire vectors instead of for separate rows, or to implement the query engine so that there is almost no dispatching cost. If you don't do this, with any half-decent disk subsystem, the query interpreter inevitably stalls the CPU.
It makes sense to both store data in columns and process it, when possible, by columns.

There are two ways to do this:

1. A vector engine. All operations are written for vectors, instead of for separate values. This means you don't need to call operations very often, and dispatching costs are negligible. Operation code contains an optimized internal cycle.
2. Code generation. The code generated for the query has all the indirect calls in it.

This is not done in "normal" databases, because it doesn't make sense when running simple queries. However, there are exceptions. For example, MemSQL uses code generation to reduce latency when processing SQL queries. (For comparison, analytical DBMSs require optimization of throughput, not latency.)

Note that for CPU efficiency, the query language must be declarative (SQL or MDX), or at least a vector (J, K). The query should only contain implicit loops, allowing for optimization.