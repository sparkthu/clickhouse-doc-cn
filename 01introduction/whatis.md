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
不同的存储方式会适应不同的场景。数据获取的场景是指请求如何发出，频率如何，比例如何。每种请求有多少数据被读出，包括按行，按列，按字节数；读取和更新数据之间的关系如何，活跃的数据有多大以及本地性如何；是否使用了事务，以及是否独立；对于数据持久性的要求如何，每种请求的延迟和吞吐量的要求如何等等。

整个系统的负载越高，针对场景定制化系统就越重要，定制化程度也越高。没有任何系统是能够很好的使用完全不同的各种场景的。如果一个系统适应很宽泛的场景需求，在负载很高的情况下，要么对每种场景的处理都很差，要么只在其中一种场景下工作的很好。

在OLAP(Online Analytical Processing)的场景下，具备以下特征：

- 绝大多数请求是读请求。
- 数据是批量更新的（每次1000行以上），而不是单条更新，或者从来不更新。
- 数据不会被修改。
- 读的时候，一次会读很多行，不过仅包括少量的列。
- 表很宽，也就是说有大量的列
- 请求频率相对不高（每台机器每秒小于几百个请求）
- 对一般的请求，50ms左右的延迟是允许的。
- 单列的数据很小，通常是数字或短字符串（比如60字节的URL信息）
- 处理单个请求的时候需要很大的吞吐量（每台机器每秒上十亿行）
- 没有事务要求
- 对数据一致性要求低
- 对每个请求来讲有一个表特别大，其他涉及的表都很小。
- 查询的结果比原始数据小得多，也就是说数据被过滤或聚合了。结果的数据量能够放入单台机器的内存中。

很容易看出来OLAP的场景和其他的流行的场景有很大不同（比如OLTP或KV读取的场景），因此完全没有必要去尝试使用OLTP数据库或KV数据库来处理分析类的请求。例如不要使用MongoDB或Elliptics做数据分析，相比于使用OLAP数据库性能会很差。

列式数据库会更适应OLAP的场景（至少1000倍的性能优势），有以下两个原因：

1. 从IO来讲：
    - 对于分析类的请求，每一行只有少量的几列需要读。对于列式数据库，你可以只读取你需要的数据，比如100列中你只需要5列，你就节省了20倍的IO。
    - 因为数据是按包读取的，更容易压缩，列式数据库能得到更好的压缩效果。
    - 因为IO数据量减少，系统的缓存能够缓存更多的数据。
  
  例如，“统计每个广告平台的记录的数量”这个请求需要读取“广告平台ID“这个列，未压缩时占用了1个字节。如果大部分的流量不是来自广告平台，你可以期待获得十倍的压缩。当使用一个快速的压缩算法时，每秒压缩几个G的数据是可能的。也就是说单台机器可以每秒处理数十亿行数据。而这个速度也实际达到了。
2. 从CPU来讲：（这部分没看懂）
  因为执行一条请求需要处理大量的行，将整个操作的vectors而不是单独的行进行分发会更合理，或者实现一个请求引擎来避免分发的开销。如果不这样，对于任何half-decent disk subsystem，请求解释器将会占满CPU。Since executing a query requires processing a large number of rows, it helps to dispatch all operations for entire vectors instead of for separate rows, or to implement the query engine so that there is almost no dispatching cost. If you don't do this, with any half-decent disk subsystem, the query interpreter inevitably stalls the CPU.
  
  因此将数据按列存储，并在可能的时候按列处理是很重要的。
  
  有两种方法可以做到这一点：
  
  1. 一个vector engine. 所有的操作为向量操作而写的，而不是单独的数据。这意味着你不需要经常调用操作，分发的开销就可以忽略不计。操作的代码包括了优化过的内部循环。A vector engine. All operations are written for vectors, instead of for separate values. This means you don't need to call operations very often, and dispatching costs are negligible. Operation code contains an optimized internal cycle.
  2. 代码生成。为请求而生成的代码包括了简介的操作。Code generation. The code generated for the query has all the indirect calls in it.

  在不同的数据库中并没有实现这些方法，因为对于简单的请求没有太大意义。然而也有例外，MemSQL使用了代码生成来降低处理SQL请求时的延迟。（相反，对于分析类的数据库更需要优化吞吐量而不是延迟）
  
  注意一点，为了提高CPU的效率，查询语句必须是声明式的（SQL或MDX），或至少是向量式的，请求中应该只包含隐式循环，方便优化。
