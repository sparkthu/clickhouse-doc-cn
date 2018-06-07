\##ClickHouse 引擎

Clickhouse 提供了丰富的存储引擎，存储引擎的类型决定了数据如何存放、如何做备份、如何被检索、是否使用索引。不同的存储引擎在数据写入/检索方面做平衡，以满足不同业务需求。

Clickhouse 提供了十多种引擎，主要是用的是`MergeTree`  系表引擎。

\###TinyLog

这是最简单的表引擎，它将数据存储在磁盘上。每列都存储在一个单独的压缩文件中。写入时，数据被附加到文件的末尾。该类型引擎不支持索引 这种引擎没有并发数据访问控制：

- 同时对一张表进行读写操作，读操会错误
- 同时在多个查询中进行写入操作，数据将被破坏

使用此表的典型方法是一次写入：只需要一次写入数据，然后根据需要多次读取它。查询在单个流处理中执行，换句话说该引擎适用于相对较小的表格（官方推荐建议一百万行以内）。如果你有很多小表，使用这个表引擎是很有意义的，因为他比Log Engines（另一个引擎下边会介绍）更简单（需要代开的文件更少）。当你有大量读写效率很低的小表时，而且在与另一个DBMS一起工作时已经被使用了，你可能会发现切换到使用TinyLog类型的表更容易。 在Yandex.Metrica中，TinyLog表用于小批量处理的中间数据。

```
CREATE TABLE test.tinyLog_test ( id String,  name String) ENGINE = TinyLog

insert into test.Log_test (id, name) values ('1', 'first');
```

找到数据目录（{home}/clickhouse/data/data/test）数据在磁盘上的结构如下

```
[hc@dpnode04 test]$ tree  -CL 5 ./tinyLog_test/
./tinyLog_test/
├── id.bin
├── name.bin
└── sizes.json
```

a.bin 和 b.bin 是压缩过的对应的列的数据， sizes.json 中记录了每个 *.bin 文件的大小：

```
[hc@dpnode04 test]$ cat ./tinyLog_test/sizes.json 
{"yandex":{"id%2Ebin":{"size":"28"},"name%2Ebin":{"size":"32"}}}
```

\###Log

Log引擎跟TinyLog引擎的区别是增加一个小的标记文件（*marks.mrk )* 留在列文件中。这些文件记录了每个数据块的偏移量。这种做的一个用处就是可以准确的切分读的范围，从而使并发读成为可能。但是，它是不能支持并发写的，一个写操作会阻塞其他读操作。 Log引擎不支持索引，同时因为有一个_marks.mrk 冗余数据，所以在写入数据时，一旦出现问题，这个表就报废了。同TinyLog差不多Log引擎使用的场景也是那种一次下入后面都是读取的场景，比如测试环境或者演示场景。

```
CREATE TABLE test.Log_test ( id String,  name String) ENGINE = Log
insert into test.Log_test (id, name) values ('1', 'abc');
insert into test.Log_test (id, name) values ('2', 'q');
insert into test.Log_test (id, name) values ('3', 'w');
insert into test.Log_test (id, name) values ('4', 'e');
insert into test.Log_test (id, name) values ('5', 'r');
```

数据（{home}/clickhouse/data/data/test）在磁盘上的结构：

```
[hc@dpnode04 test]$ tree  -CL 5 ./Log_test/
./Log_test/
├── id.bin
├── __marks.mrk
├── name.bin
└── sizes.json
```

\###Memory

Memory引擎 数据以未压缩的形式存储在RAM中。数据的存储方式与读取时的接收的格式完全相同。（在很多情况下，MergeTree引擎的性能几乎一样高）锁是短暂的：读写操作不会彼此阻塞。不支持索引。支持并发访问数据库。 简单的查询最快（超过10GB/s）服务器重启数据就会消失。 一般用到它的地方不多，除了用来测试，就是在需要非常高的性能，同时数据量又不太大（上限大概 1 亿行）的场景。 内存引擎由系统用于具有外部查询数据的临时表 ###Merge

一个工具引擎，本身不保存数据，只用于把指定库中的指定多个表链在一起。这样，读取操作可以并发执行，同时也可以利用原表的索引，但是，此引擎不支持写操作。

指定引擎的同时，需要指定要链接的库及表，库名可以使用一个表达式，表名可以使用正则表达式指定（只用表达的方式创建成功了，而且表结构得保持一致）。

```
create table test.me1 (id String, name String) ENGINE=TinyLog;
create table test.me2 (id String, name String) ENGINE=TinyLog;
create table test.me3 (id String, name String) ENGINE=TinyLog;

insert into me1(id, name) values ('1', 'first');
insert into me2(id, name) values ('2', 'xxxx');
insert into me3(id, name) values ('12', 'i am in t3');

create table merge_test (id String, name String) ENGINE=Merge(currentDatabase(), 'me[1-9]\d*');
```

上面先建了 me1 ， me2 ， me3 ，三个表，然后用 Merge 引擎的merge_test表再把它们链接起来。

这样，查询的时候，就能同时取到三个表的数据了：![merge](https://note.youdao.com/yws/api/personal/file/3DC2D8C67EFB45E3AAD4985DDBDC87BC?method=download&shareKey=928852fb1bfa26246fabfbf84250bf9b)

select 中， "_table" 这个列，是因为使用了 Merge 多出来的一个的一个 虚拟列 ，它表示原始数据的来源表，它不会出现在 show table 的结果当中，同时， select * 不会包含它。

### Distributed

Distributed 引擎并不存储真实数据，而是来做分布式写入和查询，与其他引擎配合使用。比如：Distributed + MergeTree。并行执行查询操作。在查询期间，如果远程服务器上的表索引的话会被使用。数据不仅被读取，而且在远程服务器上进行部分处理（只要这是可能的）。例如，对于具有GROUP BY的查询，数据将在远程服务器上聚合，聚合函数的中间状态将发送到请求者服务器。然后数据将被进一步聚合。

示例：

```
Distributed(remote_group, database, table [, sharding_key])
eg. Distributed(logs, default, hits[, sharding_key])
```

- *remote_group* 是配置文件（默认在 `/etc/clickhouse-server/config.xml` ）中的 `remote_servers` 一节的配置信息。
- *database* 是各服务器中的库名。可以使用返回字符串的常量表达式来代替数据库名称。例如：currentDatabase（）。
- *table* 是表名。
- *sharding_key* 是一个 *寻址表达式* ，可以是一个列名，也可以是像 `rand()` 之类的函数调用，它与 `remote_servers` 中的 `weight` 共同作用，在写入时决定往哪个 *shard* 写。每个分片都可以在配置文件中定义一个weight 。默认情况下，weight 重等于1。数据以与shard weight成比例的量分布在shards中。例如，如果有两个碎片，第一个的weight 为9，而第二个的weight 为10，则第一个将发送9/19parts ，第二个将发送10/19。要判断一行数据发送到哪个shard ，通过分片表达式我们能得到一个数字，这个数字对总权重进行取余操作。余数命中哪个shard 的范围，这条数据就会在那个shard 上。例如，如果有两个碎片，第一个碎片的权重为9，其shard的范围是[0，9]，第二个碎片的碎片权重为10，shard的范围[9,19]。假设该行sharding expression 为22 则公式为 22 对 总权重19 取 余得3 应该在第一个分片上。分片表达式可以是返回一个整数的来自常量和表中列的任何表达式。例如，您可以使用表达式rand（）来随机分配数据，或使用'UserID'来分配剩余的用户ID（然后单个用户的数据将驻留在单个分片上，这简化了由用户运行IN和JOIN）。如果其中一列的分布不够均匀，则可以将其包装在散列函数中：intHash64（UserID）。划分的一个简单余数是分片的有限解决方案，并不总是合适的。它适用于大量数据（数十台服务器），但不适用于大量数据（数百台服务器或更多）。在后一种情况下，请使用主题区所需的分片方案，而不要使用分布式表中的条目。

SELECT操作被发送到所有分片，并且无论数据如何分布在分片中（它们可以完全随机分发）都可以工作。当你添加一个新的分片时，你不必将旧数据传送给它。您可以用较重的权重编写新数据 - 数据将稍微不均匀分布，但查询将正确有效地工作。

- host 远程服务器的地址。您可以使用域或IPv4或IPv6地址。如果指定域，则服务器在启动时发出DNS请求，并且只要服务器正在运行，结果就会被存储。如果DNS请求失败，服务器不会启动。如果您更改DNS记录，请重新启动服务器。
- port 信使活动的TCP端口（配置中的'tcp_port'，通常设置为9000）。不要将它与http_port混淆。
- user 连接到远程服务器的用户的名称。默认值：default。该用户必须有权访问指定的服务器。 Access在users.xml文件中配置。有关更多信息，请参阅“Access rights”部分。
- password 连接到远程服务器的密码（未加密）。默认值：空字符串

指定副本时，读取时将为每个分片选择一个可用副本。您可以配置算法以实现负载平衡（副本访问的首选项） - 请参阅“load_balancing”设置。如果与服务器的连接未建立，则会尝试连接一个短暂超时。如果连接失败，则将选择下一个副本，以此类推所有副本。如果连接尝试对所有副本都失败，则尝试将以相同的方式重复几次。这有助于恢复能力，但不提供完整的容错能力：远程服务器可能会接受连接，但可能无法工作，或工作效果不佳。

可以配置您想配置的集群数量（即：集群的个数没有限制），要查看您的群集，请使用'system.clusters'表

Distributed引擎允许像本地服务器一样使用群集。但是，群集是不可扩展的：您必须将其配置写入服务器配置文件。

不支持查看其他Distributed表的Distributed 表（除非分布式表只有一个分片）。作为替代方法，使Distributed 表查看“final”表。Distributed 引擎要求将clusters写入配置文件。无需重新启动服务器即可更新配置文件中的群集。

如果您需要每次向未知的分片和副本集发送查询，则不需要创建分布式表 - 而是使用“remote”表函数。请参阅“Table functions”一节。

- 有两种将数据写入群集的方法:

第一种，可以定义将哪些数据写入哪些服务器，并直接在每个分片上执行写入操作。也就是， 着眼于在分布式表中执行INSERT。这是最灵活的解决方案 - 您可以使用任何sharding scheme，由于所处领域的要求，这可能不是微不足道的。这也是最理想的解决方案，因为数据可以完全独立地写入不同的分片。（我理解的就是想往哪些服务器写哪些数据，直接通过分片去写入）

第二种，您可以在分布式表中执行INSERT。在这种情况下，表格将在服务器本身分配插入的数据。为了写入分布式表，它必须有一个sharding key（最后一个参数）。另外，如果只有一个分片，写入操作无需指定sharding key，因为它在这种情况下没有任何意义。

每个分片都可以在配置文件中定义'internal_replication'参数。 如果此参数设置为'true'，则写入操作会选择第一个健康副本并向其写入数据。然后各个replica之间通过zookeeper自动同步数据，类似于replicated表的数据同步模式。如果分布式表“looks at”复制表，请使用此选项。简单来说，后台会自动创建数据备份。 如果它设置为'false'（默认），则数据将写入所有副本。实际上，这意味着分布式表本身复制数据。这比使用复制表更糟糕，因为副本的一致性未被检查，并且随着时间的推移，它们将包含稍微不同的数据。

在以下情况下，您应该关注分片方案：

- 使用查询需要通过特定键连接数据（IN或JOIN）。如果数据被该键分割，则可以使用本地IN或JOIN而不是GLOBAL IN或GLOBAL JOIN，这样更有效。
- 使用大量服务器（数百个或更多）以及大量小型查询（查询单个客户端 - 网站，广告商或合作伙伴）。为了使小型查询不影响整个群集，在单个分片上定位单个客户端的数据是有意义的。或者，正如我们在Yandex.Metrica中所做的那样，您可以设置双层分片：将整个群集划分为“layers”（层），其中layers可以由多个分片组成。单个客户端的数据位于单个layers上，但必要时可将碎片添加到layers中，并且数据随机分布在其中。为每个layer创建分布式表，并为全局查询创建一个共享分布式表。

数据是异步写入的。对于分布式表的INSERT，数据块只写入本地文件系统。数据尽快发送到后台的远程服务器。通过检查表目录中的文件列表（等待发送的数据）：/ var / lib / clickhouse / data / database / table /来检查数据是否成功发送。

如果服务器在对分布式表进行INSERT后停止退出或暴力重启（例如，设备出现故障后），则插入的数据可能会丢失。如果在表目录中检测到损坏的数据部分，它将被转移到'broken'子目录并不再使用。 当启用max_parallel_replicas选项时，查询处理在单个分片中的所有副本上并行化。有关更多信息，请参阅“设置，max_parallel_replicas”部分。

### MergeTree

MergeTree 是 ClickHouse 中最重要的引擎，并由 MergeTree 衍生出了一系列的引擎，统称 MergeTree 系引擎。MergeTree 引擎提供了工根据日期进行索引和根据主键进行索引。MergeTree 是ClickHouse最先进的表引擎，不要跟merge 引擎混淆。

使用这个引擎的形式如下：

```
MergeTree(EventDate, (CounterID, EventDate), 8192)
eg. MergeTree(EventDate, intHash32(UserID), (CounterID, EventDate, intHash32(UserID)), 8192)
```

- `EventDate` 一个日期的列名。一个MergeTree 类型的表必须有一个包含date类型的列。必须是Date类型不允许是DateTime
- `intHash32(UserID)` 采样表达式。来伪随机的对主键里的CounterID和EventDate进行打散。换句话说，当使用了SAMPLE子句时，您会为一部分用户获得均匀的伪随机数据样本。
- `(CounterID, EventDate)` 主键组（里面除了列名，也支持表达式），也可以是一个表达式。 `8192` 主键索引的粒度。 这个表是由很多个part构成。每一个part 按照主键进行了排序，除此之外，每一个part含有一个最小日期和最大日期。当插入数据的时候，会创建一个新的sort part ，同时会在后台周期行的进行merge的过程，当merge的时候，很多个part会被选中，通常是一个最小的一些part，然后merge成为一个大的排序好的part。 简单来说，整个整个合并排序的过程是在数据插入表的时候进行的。这个merge会导致这个表总是由少量的排序好的part构成，而且这个merge 本身没有做特别多的工作。这些part 在进行合并的时候会有一个大小的阈值，所以不会有太长的merge过程。 对于每一个part会生成索引文件。这个索引文件储存了表里面每一个索引块数据的主键的value值，也就是这是这个part的小型索引 对于列来说，在每一个索引块的数据也写入了标记，从而让数据可以在明确的数值范围内被查找到 当读表里的数据时，select 查询会被转化为要使用那些索引。这些索引会被用在判断where 条件或者prewhere 条件中，来判断是否命中了这些索引区间。 因此能够快速查询一个或者多个主键范围的值，在下面的示例中，能够快速的查询一个明确的counter，执行范围的日期区间里的一个明确的counter，各种counter的集合等。

总结一下上面说的：

##### 特性

- 支持主键索和日期索引。
- 可以提供实时的数据更新。
- MergeTree 类型的表必须有一个 Date 类型列。因为默认情况下数据是按时间进行分区存放的。

##### 分区

- MergeTree 默认分区是以月为单位，同一个月的数据永远都不会被合并，从1.1.54310 版本以后，可以自定义。可以通过 system.parts 表查看表的分区情况
- 同一个分区的数据会被切割到不同的文件夹中。
- 当有新数据写入时，数据会被写入新的文件夹中，后台会有线程定时对这些文件夹进行合并。
- 每个文件夹中包含当前文件夹范围内的数据，数据按照主码排序，并且每个文件夹中有一个针对该文件夹中数据的索引文件。

##### 索引

- 每个数据分区的子文件夹都有一个独立索引。 对于日期索引，查询仅仅在包含这些数据的分区上执行。
- 当 where 子句中在索引列及 Date 列上做了“等于、不等于、>=、<=、>、<、IN、bool 判断”操作，索引就会起作用。
- Like 操作不会使用索引如下面的 SQL 将不会用到索引。eg.SELECT count()FROM table WHERE CounterID = 34 OR URL LIKE '%upyachka%'。
- 查询时最好指定主码，因为在一个子分区中，数据按照主码存储。所以，当定位到某天的数据文件夹时，如果这一天数据量很大，查询不带主码就会导致大量的数据扫描。

```
create table mergeTree_test (gmt  Date, id UInt16, name String, point UInt16) ENGINE=MergeTree(gmt, (id, name), 10);

insert into mergeTree_test(gmt, id, name, point) values ('2017-04-01', 1, 'zys', 10);
insert into mergeTree_test(gmt, id, name, point) values ('2017-06-01', 4, 'abc', 10);
insert into mergeTree_test(gmt, id, name, point) values ('2017-04-03', 5, 'zys', 11);
```

这个mergeTree_test 的gmt 只接受 yyyy-mm-dd 的格式

数据在磁盘中的结构

![MergeTree](https://note.youdao.com/yws/api/personal/file/29FF62B819B84F229B5D8A0DF65A9985?method=download&shareKey=928852fb1bfa26246fabfbf84250bf9b)

从上面看： 最外层的目录，是根据日期列的范围，做了切分的。目前看来，三条数据，并没有使系统执行 merge 操作（还是有三个目录），后面使用更多的数据看看表现。 最外层的目录：最小日期 - 最大日期 - 最小区块数量 - 最大块数 - 层级 这个是旧的结构不是最新的 detached ：如果系统检测到有损坏的数据部分（文件大小错误）或无法识别的部分（部分写入文件系统但未记录在ZooKeeper中），它会将它们移动到“detached”子目录（它们不会被删除） 目录内， primary.idx 应该就是主键组索引了。 目录内其它的文件，看起来跟 Log 引擎的差不多，就是按列保存，额外的 mrk 文件保存一下块偏移量。 使用 optimize table mergeTree_test 触发 merge 行为，三个目录会被合成两个目录，变成 20170401_20170403_2_6_1 和 20170601_20170601_4_4_0 了）

### ReplacingMergeTree

这个引擎是在 MergeTree 的基础上，添加了“处理重复数据”（根据主键去重）的功能，特别适合是在多维数据加工流程中，为“最新值”，“实时数据”场景。 表引擎的最后一个可选参数是版本列。合并时，它将具有相同主键值的所有行缩减为一行。如果指定了版本列，则会保留最高版本的行;否则，保留最后一行。版本列数据类型必须是 UInt 系、Date或者Datetime 中的一个 ReplacingMergeTree(EventDate, (OrderID, EventDate, BannerID, ...), 8192, ver) 注意，数据仅在合并期间进行重复数据删除。合并在未知的时间发生在后台，所以你不能为此计划。部分数据可能保持未处理状态。虽然可以使用OPTIMIZE操作运行未计划的合并，但不要指望使用它，因为OPTIMIZE操作将读取和写入大量数据 因此，ReplacingMergeTree适用于清除后台中的重复数据以节省空间，但不能保证一定不存在重复。这个引擎不在Yandex.Metrica中使用，但它已被应用于其他Yandex项目

```
create table rmt_test (gmt  Date, id UInt16, name String, point UInt16) ENGINE=ReplacingMergeTree(gmt, (name), 10, point);

insert into rmt_test (gmt, id, name, point) values ('2017-07-10', 1, 'a', 20);
insert into rmt_test (gmt, id, name, point) values ('2017-07-10', 1, 'a', 30);
insert into rmt_test (gmt, id, name, point) values ('2017-07-11', 1, 'a', 20);
insert into rmt_test (gmt, id, name, point) values ('2017-07-11', 1, 'a', 30);
insert into rmt_test (gmt, id, name, point) values ('2017-07-11', 1, 'a', 10);
```

数据在磁盘上的结构

![rmt_test](https://note.youdao.com/yws/api/personal/file/E0C429617986406E89F85408E0BD565D?method=download&shareKey=928852fb1bfa26246fabfbf84250bf9b)

结构和MergeTree是一样的 插入这些数据，用 optimize table rmt_test 手动触发一下 merge 行为，然后查询：

```
:) select * from rmt_test

SELECT *
FROM rmt_test 

┌────────gmt─┬─id─┬─name─┬─point─┐
│ 2017-07-11 │  1 │ a    │    30 │
└────────────┴────┴──────┴───────┘

1 rows in set. Elapsed: 0.003 sec. 
```

### SummingMergeTree

此引擎与MergeTree的不同之处在于它在合并时汇总数据。

```
SummingMergeTree(EventDate, (OrderID, EventDate, BannerID, ...), 8192)

SummingMergeTree(EventDate, (OrderID, EventDate, BannerID, ...), 8192, (Shows, Clicks, Cost, ...))
```

明确设置要总计的列（最后的几个参数 - Shows, Clicks, Cost, ...）。合并时，具有相同主键值的所有行将其值汇总在指定的列中。指定的列也必须是数字，并且不能是主键的一部分。可加列不能是主键中的列，并且如果某行数据可加列都是 `null` ，则这行会被删除。 对于不属于主键的其他部分，合并时会选择第一个出现的值。 另外，一个表格可以嵌套以特殊方式处理的数据结构。如果嵌套表的名称以'Map'结尾，并且它至少包含两列满足以下条件的列

1. 表的第一列是数字类（Uint ，Date ，DateTime），我们将它称之为key；
2. 其他列是支持运算的（(U)IntN, Float32/64），我们将它称之为values 然后，这个嵌套表被解释为key =>（values ...）的映射，并且在merging时，两个数据集的元素通过'key'合并，相应的（值...） 合并求和。 Examples： [(1, 100)] + [(2, 150)] -> [(1, 100), (2, 150)] key 不相同 (1, 100)] + [(1, 150)] -> [(1, 250)]	key 相同 [(1, 100)] + [(1, 150), (2, 150)] -> [(1, 250), (2, 150)] [(1, 100), (2, 150)] + [(1, -100)] -> [(2, 150)]

对于Map的聚合，使用函数sumMap（key，value） 对于嵌套数据结构，不需要将列指定为总计列的列表。 这个表引擎不是特别有用。请记住，只保存预先汇总的数据时，会损失系统的某些优势

数据在磁盘上的结构和MergeTree一样

### AggregatingMergeTree

此引擎与MergeTree的不同之处在于，具有相同主键值的行merge 通过表中存储的聚合函数的逻辑 为此，它使用AggregateFunction数据类型，以及用于集合函数的-State和-Merge修饰符。 让我们仔细研究一下，有一个AggregateFunction数据类型。它是一种参数数据类型。作为参数，传递聚集函数的名称，然后是其参数的类型。 Examples

```
CREATE TABLE t
(
    column1 AggregateFunction(uniq, UInt64),
    column2 AggregateFunction(anyIf, String, UInt8),
    column3 AggregateFunction(quantiles(0.5, 0.9), UInt64)
) ENGINE = ...
```

这种类型的列存储聚合函数的逻辑。要获得这种类型的值，请使用具有状态后缀的聚合函数。 例如: uniqState(UserID), quantilesState(0.5, 0.9)(SendTiming) 与相应的uniq和quantiles函数相反，这些函数返回状态，而不是准备好的值。也就是说，他们返回一个AggregateFunction类型的值。（这里我理解的就像是Scala这种语言一样只是一些逻辑的声明，并没有去按照逻辑去计算，也可能放了一些中间值） AggregateFunction类型值不能以Pretty formats输出。在其他格式中，这些类型的值将作为特定于实现的二进制数据输出。 在一次转换中AggregateFunction类型值不用于输出或保存 使用AggregateFunction类型值可以做的唯一有用的事情是将状态组合起来并获得结果，这实际上意味着完成聚合。具有“Merge并”后缀的聚合函数用于此目的。 例如：uniqMerge（UserIDState），其中UserIDState具有AggregateFunction类型。 简单来说，具有“Merge并”后缀的聚合函数会采用一组状态，将它们组合起来并返回结果。举个例子，这两个查询返回相同的结果：

```
SELECT uniq(UserID) FROM table
SELECT uniqMerge(state) FROM (SELECT uniqState(UserID) AS state FROM table GROUP BY RegionID)
```

AggregatingMergeTree引擎。它在合并期间的工作是 将来表中相同主键的行按照的聚集函数的逻辑组合在一起。 不能使用普通INSERT在包含AggregateFunction列的表中插入行，因为无法显式定义AggregateFunction值。只能使用INSERT SELECT和-State集合函数插入数据。 使用AggregatingMergeTree表中的SELECT，通过'-Merge'修饰符使用GROUP BY和聚合函数来完成数据聚合 您可以使用AggregatingMergeTree表进行增量数据聚合，包括聚合物化视图。 Example：

```
CREATE MATERIALIZED VIEW test.basic
ENGINE = AggregatingMergeTree(StartDate, (CounterID, StartDate), 8192)
AS SELECT
    CounterID,
    StartDate,
    sumState(Sign)    AS Visits,
    uniqState(UserID) AS Users
FROM test.visits
GROUP BY CounterID, StartDate;
```

在test.visits表中插入数据。数据也将被插入到视图中，并汇总在该视图中： INSERT INTO test.visits ... 使用GROUP BY从视图执行SELECT以完成数据聚合：

```
SELECT
    StartDate,
    sumMerge(Visits) AS Visits,
    uniqMerge(Users) AS Users
FROM test.basic
GROUP BY StartDate
ORDER BY StartDate;
```

你可以像这样创建一个物化视图并为其分配一个普通的视图来完成数据聚合。 请注意，在大多数情况下，使用AggregatingMergeTree是没有效果的，因为直接查询也很快。这个看的也是很迷糊

### CollapsingMergeTree

这个引擎的适用场景是实时查询当前应用上的实时数据，可以类比为变形版的窗口函数。

要搞明白它，以及什么场景下用它，为什么用它，需要先行了解一些背景。

首先，在 *clickhouse* 中，数据是不能改，更不能删的，其实在好多数仓的基础设施中都是这样。前面为了数据的“删除”，还专门有一个 *ReplacingMergeTree* 引擎嘛。在这个条件之下，想要处理“终态”类的数据，比如大部分的状态数据都是这类，就有些麻烦了。

试想，假设每隔 10 秒时间，你都能获取到一个当前在线人数的数据，把这些数据一条一条存下，大概就是这样：

| 时间点 | 在线人数 |
| ------ | -------- |
| 10     | 123      |
| 20     | 101      |
| 30     | 98       |
| 40     | 88       |
| 50     | 180      |

现在问你，“当前有多少人在线？”，这么简单的问题，怎么回答？

在这种存数机制下，“当前在线人数”显然是不能把 *在线人数* 这一列聚合起来取数的嘛。也许，能想到的是，“取最大的时间”的那一行，即先 `order by` 再 `limit 1` ，这个办法，在这种简单场景下，好像可行。那我们再把维度加一点：

| 时间点 | 频道 | 在线人数 |
| ------ | ---- | -------- |
| 10     | a    | 123      |
| 10     | b    | 29       |
| 10     | c    | 290      |
| 20     | a    | 101      |
| 20     | b    | 181      |
| 20     | c    | 31       |
| 30     | a    | 98       |
| 30     | b    | 18       |
| 30     | c    | 56       |
| 40     | a    | 88       |
| 40     | b    | 9        |
| 40     | c    | 145      |

这时，如果想看每个频道的当前在线人数，查询就不像之前那么好写了，硬上的话，你可能需要套子查询。好了，我们目的不是讨论 SQL 语句怎么写。

回到开始的数据：

| 时间点 | 在线人数 |
| ------ | -------- |
| 10     | 123      |
| 20     | 101      |
| 30     | 98       |
| 40     | 88       |
| 50     | 180      |

如果我们的数据，是在关心一个最终的状态，或者说最新的状态的话，考虑在业务型数据库中的作法，我们会不断地更新确定的一条数据， OLAP 环境我们不能改数据，但是，我们可以通过“运算”的方式，去抹掉旧数据的影响，把旧数据“减”去即可，比如：

| 符号 | 时间点 | 在线人数 |
| ---- | ------ | -------- |
| +    | 10     | 123      |
| -    | 10     | 123      |
| +    | 20     | 101      |

当我们在添加 20 时间点的数据前，首先把之前一条数据“减”去，以这种“以加代删”的增量方式，达到保存最新状态的目的。

当然，起初的数据存储，我们可以以 `+1` 和 `-1` 表示符号，以前面两个维度的数据的情况来看（我们把 “时间gmt，频道point” 作为主键）：

| sign | gmt  | name | point |
| ---- | ---- | ---- | ----- |
| +1   | 10   | a    | 123   |
| +1   | 10   | b    | 29    |
| +1   | 10   | c    | 290   |
| -1   | 10   | a    | 123   |
| +1   | 20   | a    | 101   |
| -1   | 10   | b    | 29    |
| +1   | 20   | b    | 181   |
| -1   | 10   | c    | 290   |
| +1   | 20   | c    | 31    |

如果想看每个频道的当前在线人数：

```
select name, sum(point * sign) from t group by name;
```

就可以得到正确结果了：

```
┌─name─┬─sum(multiply(point, sign))─┐
│ b    │                        181 │
│ c    │                         31 │
│ a    │                        101 │
└──────┴────────────────────────────┘
```

神奇。考虑数据可能有错误的情况（`-1` 和 `+1` 不匹配），我们可以添加一个 `having` 来把错误的数据过滤掉，比如再多一条类似这样的数据：

```
insert into t (sign, gmt, name, point) values (-1, '2017-07-11', 'd', 10),
```

再按原来的 SQL 查，结果是：

```
┌─name─┬─sum(multiply(point, sign))─┐
│ b    │                        181 │
│ c    │                         31 │
│ d    │                        -10 │
│ a    │                        101 │
└──────┴────────────────────────────┘
```

加一个 `having` ：

```
select name, sum(point * sign) from t group by name having sum(sign) > 0;
```

就可以得到正确的数据了：

```
┌─name─┬─sum(multiply(point, sign))─┐
│ b    │                        181 │
│ c    │                         31 │
│ a    │                        101 │
└──────┴────────────────────────────┘
```

这种增量方式更大的好处，是它与指标本身的性质无关的，不管是否是可加指标，或者是像 UV 这种的去重指标，都可以处理。

相较于其它一些变通的处理方式，比如对于可加指标，我们可以通过“差值”存储，来使最后的 `sum` 聚合正确工作，但是对于不可加指标就无能为力了。

上面的东西如果都明白了，我们也就很容易理解 *CollapsingMergeTree* 引擎的作用了。

“以加代删”的增量存储方式，带来了聚合计算方便的好处，代价却是存储空间的翻倍，并且，对于只关心最新状态的场景，中间数据都是无用的。 *CollapsingMergeTree* 引擎的作用，就是针对主键，来帮你维护这些数据，它会在 *merge* 期，把中间数据删除掉。

前面的数据，如果我们存在 *MergeTree* 引擎的表中，那么通过 `select * from t` 查出来是：

```
┌─sign─┬────────gmt─┬─name─┬─point─┐
│    1 │ 2017-07-10 │ a    │   123 │
│   -1 │ 2017-07-10 │ a    │   123 │
│    1 │ 2017-07-10 │ b    │    29 │
│   -1 │ 2017-07-10 │ b    │    29 │
│    1 │ 2017-07-10 │ c    │   290 │
│   -1 │ 2017-07-10 │ c    │   290 │
│    1 │ 2017-07-11 │ a    │   101 │
│    1 │ 2017-07-11 │ b    │   181 │
│    1 │ 2017-07-11 │ c    │    31 │
│   -1 │ 2017-07-11 │ d    │    10 │
└──────┴────────────┴──────┴───────┘
```

如果换作 *CollapsingMergeTree* ，那么直接就是：

```
┌─sign─┬────────gmt─┬─name─┬─point─┐
│    1 │ 2017-07-11 │ a    │   101 │
│    1 │ 2017-07-11 │ b    │   181 │
│    1 │ 2017-07-11 │ c    │    31 │
│   -1 │ 2017-07-11 │ d    │    10 │
└──────┴────────────┴──────┴───────┘
```

*CollapsingMergeTree* 在创建时与 *MergeTree* 基本一样，除了最后多了一个参数，需要指定 *Sign* 位（必须是 `Int8` 类型）：

```
create table t(sign Int8, gmt Date, name String, point UInt16) ENGINE=CollapsingMergeTree(gmt, (gmt, name), 8192, sign);
```

讲明白了 *CollapsingMergeTree* 可能有人会问，如果只是要“最新状态”，用 *ReplacingMergeTree* 不就好了么？

这里，即使不论对“日期维度”的特殊处理（ *ReplacingMergeTree* 不会对日期维度做特殊处理，但是 *CollapsingMergeTree* 看起来是最会保留最新的），更重要的，是要搞明白， 我们面对的数据的形态，不一定是 *merge* 操作后的“完美”形态，也可能是没有 *merge* 的中间形态，所以，即使你知道最后的结果对于每个主键只有一条数据，那也只是 *merge* 操作后的结果，你查数据时，聚合函数还是得用的，当你查询那一刻，可能还有很多数据没有做 *merge* 呢。

明白了一点，不难了解，对于 *ReplacingMergeTree* 来说，在这个场景下跟 *MergeTree* 其实没有太多区别的，如果不要 `sign` ，那么结果就是日期维度在那里，你仍然不能以通用方式聚合到最新状态数据。如果要 `sign` ，当它是主键的一部分时，结果就跟 *MergeTree* 一样了，多存很多数据。而当它不是主键的一部分，那旧的 `sign` 会丢失，就跟没有 `sign` 的 *MergeTree* 一样，不能以通用方式聚合到最新状态数据。结论就是， *ReplacingMergeTree* 的应用场景本来就跟 *CollapsingMergeTree* 是两回事。

*ReplacingMergeTree* 的应用，大概都是一些 `order by limit 1` 这种。而 *CollapsingMergeTree* 则真的是 `group by` 了。

官方的例子： Yandex.Metrica有普通日志（如命中日志）和更改日志。更改日志用于增量计算不断变化的数据统计信息。例如会话更改的日志或用户历史更改的日志。Session连接在Yandex.Metrica中不断变化。例如，每个会话的点击次数增加。我们将任何对象的变化称为一对（old values, new values）。如果创建对象，旧值可能会丢失。如果对象被删除，新的值可能会丢失。如果对象已更改，但以前存在且未被删除。在更改日志中，每个更改都会创建一个或两个条目。每个条目都包含对象所具有的所有属性，以及用于区分旧值和新值的特殊属性。当对象发生变化时，只有新条目被添加到更改日志中，而现有的条目未被触及。

更改日志可以逐步计算几乎所有的统计数据。为此，我们需要考虑带有加号的“new”行和带有减号的“old”行。换句话说，所有统计量的代数结构都包含用于取元素反转的操作。大多数统计数据都是如此。我们还可以计算“幂等”统计信息，如独特访问者的数量，因为在更改会话时，唯一访问者不会被删除。 这是允许Yandex.Metrica实时工作的主要概念。CollapsingMergeTree接受一个附加参数 - 包含行的“Sign”的Int8类型列的名称。 Example CollapsingMergeTree(EventDate, (CounterID, EventDate, intHash32(UniqID), VisitID), 8192, Sign) 在这里，Sign是一个包含-1表示“old”值和1表示“new”值的列。 merge时，每组连续相同的主键值（用于排序数据的列）被减少为不超过一行，列值为'sign_column = -1'（“负行”），并且不超过一行列值'sign_column = 1'（“正行”）。换句话说，来自更改日志的条目已折叠。 如果正数行和负数行匹配，则写入第一个负行和最后一个正数行。如果正数行比负数行多一个，则只写入最后一个正数行。如果负行比正行多一个，则只写入第一个负行。否则，将会出现逻辑错误，并且不会写入任何行。如果日志的同一部分意外插入多次，则会发生逻辑错误，错误仅记录在服务器日志中，merge继续。）

因此，collapsing 不应改变计算统计的结果。随着collapsed的进行每个对象最后几乎只剩下最后的一个值。与MergeTree相比，CollapsingMergeTree引擎可以使数据量减少数倍 有几种方法可以从CollapsingMergeTree表中获取完全“collapsed”的数据： 1、用GROUP BY和聚合函数编写一个查询来解释sign。例如，要计算数量，请写'sum（Sign）'而不是'count（）'。要计算某些东西的总和，请写'sum（Sign * x）'而不是'sum（x）'，依此类推，并添加'HAVING sum（Sign）> 0'。并非所有的金额都可以这样计算。例如，集合函数'min'和'max'不能被重写。 2、如果您必须提取没有聚合的数据（例如，要检查是否存在最新值符合特定条件的行），则可以对FROM子句使用FINAL修饰符。这种方法效率显着较低·

### GraphiteMergeTree

该引擎专为汇总（细化和聚合/平均）Graphite data而设计。（Graphite 介绍)对于想要将ClickHouse用作Graphite的数据存储的开发人员可能会有帮助 没有细看

### Data replication

复制仅支持MergeTree系列中的表，下面穷举出来了： ReplicatedMergeTree ReplicatedSummingMergeTree ReplicatedReplacingMergeTree ReplicatedAggregatingMergeTree ReplicatedCollapsingMergeTree ReplicatedGraphiteMergeTree

复制适用于单个表的级别，而不是整个服务器。服务器可以同时存储复制表和非复制表 复制不依赖分片。每个分片都有自己的独立复制。 为INSERT和ALTER查询复制压缩数据（请参阅ALTER操作的说明）。 CREATE，DROP，ATTACH，DETACH和RENAME查询在单个服务器上执行，不会被复制：

- CREATE TABLE 操作将在运行查询的服务器上创建一个新的可复制表。如果该表已经存在于其他服务器上，它将添加一个新的副本。
- DROP TABLE 操作将删除位于运行查询的服务器上的副本。
- RENAME 操作会重命名其中一个副本上的表。就是说，复制表可以在不同副本上具有不同的名称

要使用复制，请在配置文件中设置ZooKeeper群集的地址。例：

```
<zookeeper>
    <node index="1">
        <host>example1</host>
        <port>2181</port>
    </node>
    <node index="2">
        <host>example2</host>
        <port>2181</port>
    </node>
    <node index="3">
        <host>example3</host>
        <port>2181</port>
    </node>
</zookeeper>
```

使用ZooKeeper 3.4.5或更高版本。 您可以指定任何现有的ZooKeeper集群，并且系统将使用其上的目录作为自己的数据（该目录在创建可复制表时指定） 如果ZooKeeper未在配置文件中设置，则无法创建复制表，并且任何现有的复制表都将为只读 在SELECT查询中不使用ZooKeeper，因为复制不会影响SELECT的性能，查询的运行速度与对非复制表的速度一样快。查询分布式复制表时，ClickHouse行为由设置max_replica_delay_for_distributed_queries和fallback_to_stale_replicas_for_distributed_queries控制。 对于每个INSERT操作，通过多个事务向ZooKeeper添加约十个条目。（更确切地说，这是针对每个插入的数据块; 一个INSERT操作包含一个块或者每个max_insert_block_size = 1048576行）。与非复制表相比，INSERT的延迟时间稍长。但是，如果按照建议以每秒不超过一次批量的INSERT数据，则不会产生任何问题。使用ZooKeeper集群的协调整个ClickHouse集群每秒总共有数百个INSERT。数据插入的吞吐量（每秒的行数）与非复制数据一样高。

对于非常大的群集，可以针对不同的分片使用不同的ZooKeeper群集。但是，这在Yandex.Metrica集群（大约300台服务器）上证明是没有必要的。

Replication 是异步和多主机的。INSERT操作（以及ALTER）可以发送到任何可用的服务器。将数据插入运行查询的服务器上，然后将其复制到其他服务器。由于它是异步的，因此最近插入的数据会在其他副本上出现一些延迟。如果部分副本不可用，则数据在可用时被写入。如果副本可用，则延迟是通过网络传输压缩数据块所需的时间。

默认情况下，INSERT操作只等待确认写入一个副本的数据。如果数据只写入一个副本，并且具有此副本的服务器不再存在，则存储的数据将会丢失。 Tp能够从多个副本获取数据写入的确认，请使用insert_quorum选项。 每个数据块都是以原子方式写入的。 INSERT操作被分成块，最多为max_insert_block_size = 1048576行。或者这样说，如果INSERT操作的行数小于1048576，则它是一个原子操作。

数据块中重复数被据删除。对于同一数据块的多次写入（相同大小的数据块包含相同顺序的相同行），该块只写入一次。这是因为客户端应用程序不知道数据是否写入数据库时发生网络故障，因此可以简单地重复INSERT查询。使用相同的数据发送哪些副本INSERT并不重要。 INSERT是幂等的。重复数据消除参数由merge_tree服务器设置控制。

在复制期间，只有要插入的源数据通过网络传输。进一步的数据转换（merging）是以相同的方式在所有副本上进行协调和执行的。这可以最大限度地减少网络使用量，这意味着当副本驻留在不同的数据中心时复制效果良好。 （请注意，复制不同数据中心中的数据是复制的主要目标。）

您可以拥有相同数据的任意数量的副本。 Yandex.Metrica在生产中使用双重复制。每个服务器在某些情况下使用RAID-5或RAID-6和RAID-10。这是一个相对可靠和方便的解决方案。

系统监控副本上的数据同步性，并能够在发生故障后进行恢复。故障转移是自动的（对于数据中的小差异）或半自动的（当数据差异太大时，这说明可能配置错误）。

\####创建复制表

Replicated的前缀被添加到表引擎名称。例如：ReplicatedMergeTree。 参数列表的开始处还添加了两个参数 - ZooKeeper中的表的路径以及ZooKeeper中的副本名称 Example

```
ReplicatedMergeTree('/clickhouse/tables/{layer}-{shard}/hits', '{replica}', EventDate, intHash32(UserID), (CounterID, EventDate, intHash32(UserID), EventTime), 8192)
```

如示例所示，这些参数可以在大括号中包含替换。替换值取自配置文件的“macros”部分(理解为变量值)。例 <macros> <layer>05</layer> <shard>02</shard> <replica>example05-02-1.yandex.ru</replica> </macros> ZooKeeper中表的路径对于每个复制表都应该是唯一的。不同碎片上的表应该有不同的路径。在这种情况下，路径由以下部分组成： /clickhouse/tables/ 是常见的前缀。官方建议使用这个。 {layer}-{shard} 是碎片标识符。在本例中它由两部分组成，因为Yandex.Metrica集群使用双层分片。对于大多数任务，可以只留下{shard}替换，该替换将扩展为碎片标识符。 hits 是ZooKeeper中表的节点名称。将它与表名称相同是一个好主意。它是明确定义的，因为与表名相反，它在RENAME操作后不会更改。

副本名称标识保持唯一，可以使用服务器名称作为示例。

应该明确定义参数，而不是使用替换。这可能对测试和配置小型集群很方便。但是，在这种情况下，在这种情况下不能使用分布式DDL查询（ON CLUSTER）。就是说不用前面提到的在macros标签中定义的变量

在处理大型集群时，我们建议使用替换，因为它们会降低出错的可能性。 在每个副本上运行CREATE TABLE操作。此操作将创建一个新的复制表，或者将新副本添加到现有复制表中。

如果在表已包含其他副本上的某些数据后添加新的副本，则在运行查询后，数据将从其他副本复制到新副本。换句话说，新副本与其他节点同步。

要删除副本，请运行DROP TABLE。但是，只有一个副本被删除 - 驻留在运行查询的服务器上的副本。

\####Recovery after failures

如果ZooKeeper在服务器启动时不可用，则复制表将切换到只读模式。系统会定期尝试连接到ZooKeeper 如果ZooKeeper在INSERT期间不可用，或者在与ZooKeeper交互时发生错误，则会抛出异常。 连接到ZooKeeper后，系统会检查本地文件系统中的数据集是否与预期的数据集（ZooKeeper存储此信息）匹配。如果存在较小的不一致性，则系统通过与副本同步数据来解决这些问题。 如果系统检测到有损坏的数据部分（文件大小错误）或无法识别的部分（部分写入文件系统但未记录在ZooKeeper中），它会将它们移动到“detached”子目录（它们不会被删除）。任何缺少的部分都从副本中复制。 请注意，ClickHouse不会执行任何破坏性操作，例如：自动删除大量数据。 当服务器启动（或者与ZooKeeper建立一个新的会话）时，它只检查所有文件的数量和大小。如果文件大小匹配但字节在中间的某个位置发生了更改，则不会立即检测到这种情况，而只是在尝试读取SELECT查询的数据时才会检测到。该查询将引发有关非匹配校验或者压缩块大小的异常。在这种情况下，数据部分将添加到验证队列中，并在必要时从副本中复制。

如果本地数据集与预期数据差别太大，则会触发安全机制。服务器在日志中输入并拒绝启动。造成这种情况的原因是，这种情况表明可能存在配置错误，例如shard 上的副本被意外配置为不同分片上的副本。但是，此机制的阈值设置得相当低，并且正常故障恢复期间可能会发生这种情况。在这种情况下，通过“pushing a button”数据被半自动地恢复。

开始恢复，在zookeeper上使用任何内容创建节点/path_to_table/replica_name/flags/force_restore_data或者运行该命令以恢复所有复制表： sudo -u clickhouse touch /var/lib/clickhouse/flags/force_restore_data 然后重新启动服务器。在开始时，服务器删除这些标志并开始恢复。

\####Recovery after complete data loss

如果所有数据和元数据都从其中一台服务器中消失，请按照以下步骤进行恢复： 1、在服务器上安装ClickHouse。如果有需要，则在包含碎片标识符和副本的配置文件中使用正确定义替换。（我理解的就是正确安装ClickHouse和正确的配置，最好就是复制一台正在良好运行的ClickHouse节点然后改配置） 2、如果您有必须在服务器上手动复制的未复制表，请从副本（在目录中）复制其数据， (in the directory /var/lib/clickhouse/data/db_name/table_name/). 3、复制位于/ var / lib / clickhouse / metadata /中副本的表定义。如果在表定义中明确定义了分片或副本标识符，请对其进行更正以使其与此副本相对应。 （或者，启动服务器并进行应该位于/ var / lib / clickhouse / metadata /中的.sql文件中的所有ATTACH TABLE操作。） 4、开始恢复，使用任何内容创建ZooKeeper节点/path_to_table/replica_name/flags/force_restore_data或运行该命令来恢复所有复制表 sudo -u clickhouse touch /var/lib/clickhouse/flags/force_restore_data

然后启动服务器（重新启动，如果它已经在运行）。数据将从副本下载。

备用恢复选项是从ZooKeeper（/ path_to_table / replica_name）删除有关已丢失副本的信息，然后再次创建副本，如"Creating replicatable tables".中所述 请注意，如果您要一次恢复多个副本，恢复期间网络带宽没有限制。

\####从MergeTree转换到ReplicatedMergeTree

我们用MergeTree来代指MergeTree系列中的所有表引擎，与ReplicatedMergeTree相同。

如果您有一个需要手动复制的MergeTree表，则可以将其转换为可复制表。如果您已经在MergeTree表中收集了大量数据并且现在想要启用复制，则可能需要执行此操作。

如果各种副本上的数据不同，请首先对其进行同步，或者保留一个副本。

重命名现有的MergeTree表，然后使用旧名称创建一个ReplicatedMergeTree表。使用新表数据（/ var / lib / clickhouse / data / db_name / table_name /）将旧表中的数据移动到目录内的“detached”子目录中。然后在其中一个副本上运行ALTER TABLE ATTACH PARTITION，将这些数据parts 添加到工作集中。

当ZooKeeper集群中的元数据丢失或损坏时恢复

使用不同的名称创建MergeTree表。将具有ReplicatedMergeTree表数据的目录中的所有数据移动到新表的数据目录。然后删除ReplicatedMergeTree表并重新启动服务器。

如果要在不启动服务器的情况下删除 replicatedmergetree 表, 请执行以下操作: 删除元数据目录（/ var / lib / clickhouse / metadata /）中相应的.sql文件。 删除ZooKeeper中的相应路径（/ path_to_table / replica_name）。

在此之后，您可以启动服务器，创建MergeTree表，将数据移动到其目录，然后重新启动服务器。

当ZooKeeper集群中的元数据丢失或损坏时恢复 如果ZooKeeper中的数据丢失或损坏，您可以通过将数据移动到未复制的表格来保存数据，如上所述。 如果其他副本上存在完全相同的部分，则会将它们添加到其工作集中。如果没有，则从具有它们的副本中进行下载。

### Buffer

Buffer 引擎，像是 Memory 存储的一个上层应用似的（磁盘上也是没有相应目录的）。它的行为是一个缓冲区，写入的数据先被放在缓冲区，达到一个阈值后，这些数据会自动被写到指定的另一个表中。 Buffer 是接在其它表前面的一层，对它的读操作，也会自动应用到后面表，但是因为前面说到的限制的原因，一般我们读数据，就直接从源表读就好了，缓冲区的这点数据延迟，只要配置得当，影响不大的。 Buffer 后面也可以不接任何表，这样的话，当数据达到阈值，就会被丢弃掉。

```
Buffer(database, table, num_layers, min_time, max_time, min_rows, max_rows, min_bytes, max_bytes)
--  先创建源表
create table source_table (gmt  Date, id UInt16, name String, point UInt16) ENGINE=MergeTree(gmt, (id, name), 10);
-- 在创建buffer表
create table source_buffer as t ENGINE=Buffer(default, source_table, 16, 3, 20, 2, 10, 1, 10000)
```

引擎参数：

- database：数据库，
- table：表 - 将数据刷新到的表。您可以使用一个返回string.num_layers - Parallelism图层的常量表达式来代替数据库名称。数据库和表名的单引号中设置空字符串。这代表没有目标表。在这种情况下，当达到数据刷新条件时，缓冲区将被清除。这对于在内存中保存数据窗口可能很有用。
- `num_layers` 是类似“分区”的概念，每个分区的后面的 `min / max` 是独立计算的，官方推荐的值是 `16` 。为每个'num_layers'缓冲区分别计算刷新数据的条件。例如，如果num_layers = 16且max_bytes = 100000000，则最大RAM消耗为1.6 GB。
- `min / max` 这组配置荐，就是设置阈值的，分别是 时间（秒），行数，空间（字节）。

阈值的规则，是“所有的 min 条件都满足， 或 至少一个 max 条件满足”。 如果按上面我们的建表来说，所有的 min 条件就是：过了 3秒，2条数据，1 Byte。一个 max 条件是：20秒，或 10 条数据，或有 10K 。

关于 Buffer 的其它一些点：

- 如果一次写入的数据太大或太多，超过了 max 条件，则会直接写入源表。
- 删源表或改源表的时候，建议 Buffer 表删了重建。
- “友好重启”时， Buffer 数据会先落到源表，“暴力重启”， Buffer 表中的数据会丢失。
- 即使使用了 Buffer ，多次的小数据写入，对比一次大数据写入，也 慢得多 （几千行与百万行的差距）。

从Buffer表读取数据时，Buffer和目标表（如果有的话）都会处理数据。请注意，Buffer表不支持索引。换句话说，Buffer中的数据已被完全扫描，对于大型Buffer可能会很慢。（对于从属表中的数据，将使用它支持的索引。）

如果Buffer表中的一组列与下属表中的一组列不匹配，则会插入两个表中存在的列的子集

如果这些类型与Buffer表和其下一个表中的一列不匹配，则会在服务器日志中输入错误消息，并清除缓冲区。如果在刷新缓冲区时从属表不存在，也会发生同样的情况。

如果需要为下级表和缓冲区表运行ALTER，我们建议首先删除Buffer表，为下级表运行ALTER，然后再次创建Buffer表。

PREWHERE，FINAL和SAMPLE对缓冲区表无法正常工作。这些条件传递到目标表，但不用于处理缓冲区中的数据。因此，我们建议只使用Buffer表进行写入，同时从目标表读取数据。

将数据添加到Buffer时，其中一个Buffer被锁定。如果从表中同时执行读取操作，则会导致延迟

插入到Buffer表中的数据可能以不同的顺序在不同的块中结束于从属表。因此，Buffer表很难正确写入CollapsingMergeTree。为了避免问题，您可以将'num_layers'设置为1。

如果复制目标表，则写入缓冲区表时，复制表的某些预期特性会丢失。随机更改数据部分的行和大小顺序会导致重复数据删除操作退出，这意味着无法对复制表进行可靠的“精确一次”写入。

由于这些缺点，我们只能推荐在极少数情况下使用Buffer表。

请注意，一次插入一行数据是无意义的，即使对于缓冲区表也是如此。这只会产生每秒几千行的速度，而插入较大的数据块每秒可产生超过一百万行（请参见“Performance”一节）。