接口
==================
要测试系统的负载能力，下载数据到表中，手动查询，可以使用clickhouse-client客户端程序

##HTTP接口

HTTP接口让你能够使用任何语言与ClickHouse交互，我们使用JAVA，Perl甚至shell脚本进行工作。在其他部门还有Python，Go语言在使用HTTP接口，相比于原生接口，http接口限制较多，但是兼容性比较好。

默认情况下，clickhouse-server的HTTP端口在8123

如果你发送GET /请求，会返回字符串"Ok\n"，这可以用来做健康检查
   
    $ curl 'http://localhost:8123/'
    Ok.

query可以作为URL参数传入，使用GET或POST，或把请求的开头部分放在参数，剩余部分放在POST的内容中。成功情况下会返回200，body中是结果，如果发生错误，会收到500，body是错误原因。

使用GET方法时，'readonly'被设置，请求无法修改数据，POST则可以。示例：

    $ curl 'http://localhost:8123/?query=SELECT%201'
    1

    $ wget -O- -q 'http://localhost:8123/?query=SELECT 1'
    1

    $ GET 'http://localhost:8123/?query=SELECT 1'
    1

    $ echo -ne 'GET /?query=SELECT%201 HTTP/1.0\r\n\r\n' | nc localhost 8123
    HTTP/1.0 200 OK
    Connection: Close
    Date: Fri, 16 Nov 2012 19:21:50 GMT

    1
    
可以看到curl并不是很方便，因为空格需要URL编码。尽管wget处理好了，我们还是不建议用wget，因为当使用了keep-alive和Transfer-Encoding: chunked，它工作的有问题。

    $ echo 'SELECT 1' | curl 'http://localhost:8123/' --data-binary @-
    1 

    $ echo 'SELECT 1' | curl 'http://localhost:8123/?query=' --data-binary @-
    1

    $ echo '1' | curl 'http://localhost:8123/?query=SELECT' --data-binary @-
    1
    
当把请求的部分内容放在body时，两部分之间有回车

<pre>
$ echo 'ECT 1' | curl 'http://localhost:8123/?query=SEL' --data-binary @-
Code: 59, e.displayText() = DB::Exception: Syntax error: failed at position 0: SEL
ECT 1
, expected One of: SHOW TABLES, SHOW DATABASES, SELECT, INSERT, CREATE, ATTACH, RENAME, DROP, DETACH, USE, SET, OPTIMIZE., e.what() = DB::Exception
</pre>

默认情况下，返回的数据使用tab分割，可以使用FORMAT子句来指定格式：

<pre>
$ echo 'SELECT 1 FORMAT Pretty' | curl 'http://localhost:8123/?' --data-binary @-
┏━━━┓
┃ 1 ┃
┡━━━┩
│ 1 │
└───┘
</pre>

对于INSERT操作，POST是必须的，你可以把请求的前半部分放在URL参数中，数据放在body中。数据可以是MYSQL dump出的tab分割的文件。在这种情况，INSERT可以替代LOAD DATA LOCAL INFILE from MySQL

例如：

建表：
    
    echo 'CREATE TABLE t (a UInt8) ENGINE = Memory' | POST 'http://localhost:8123/'
    
插入：

    echo 'INSERT INTO t VALUES (1),(2),(3)' | POST 'http://localhost:8123/'
    
数据语句分开：

    echo '(4),(5),(6)' | POST 'http://localhost:8123/?query=INSERT INTO t VALUES'
    
你可以指定任何数据格式，Values的格式和INSERT INTO t VALUES的时候一样

    echo '(7),(8),(9)' | POST 'http://localhost:8123/?query=INSERT INTO t FORMAT Values'
    
要插入tab分割的dump文件，指定：

    echo -ne '10\n11\n12\n' | POST 'http://localhost:8123/?query=INSERT INTO t FORMAT TabSeparated'
    
读出数据，数据是乱序的因为并行插入的：

<pre>
$ GET 'http://localhost:8123/?query=SELECT a FROM t'
7
8
9
10
11
12
1
2
3
4
5
6
</pre>

删除表

    POST 'http://localhost:8123/?query=DROP TABLE t'
    
请求成功时返回body为空

传输数据时你可以压缩数据，压缩使用的非标准格式，你需要使用一个特殊的压缩程序来工作（sudo apt-get install compressor-metrika-yandex）

如果你在URL中指定'compress=1'， 服务器会把返回给你的数据做压缩。

如果你指定'decompress=1',服务器会解压POST中收到的数据。

这可以用来传输大量数据时降低网络流量，或者创建直接被压缩的dumps

使用database参数可以指定使用的数据库

<pre>
$ echo 'SELECT number FROM numbers LIMIT 10' | curl 'http://localhost:8123/?database=system' --data-binary @-
0
1
2
3
4
5
6
7
8
9
</pre>

默认情况下，请求使用默认的数据库，默认的数据库叫做‘default’，你可以使用表名前加数据库名和点的方式指定使用哪个数据库。

有两种方式可以配置用户名密码：

    echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-
    echo 'SELECT 1' | curl 'http://localhost:8123/?user=user&password=password' -d @-
    
如果没有指定用户名，默认为'default'，如果没有指定密码，默认为空。

你还可以在URL参数中设置各种参数，或者在配置文件中修改：

    http://localhost:8123/?profile=web&max_rows_to_read=1000000000&query=SELECT+1
    
更多信息参考"[Settings](https://clickhouse.yandex/reference_en.html#Settings)"章节。

<pre>
$ echo 'SELECT number FROM system.numbers LIMIT 10' | curl 'http://localhost:8123/?' --data-binary @-
0
1
2
3
4
5
6
7
8
9
</pre>

要看其他参数的信息，参考"[SET](https://clickhouse.yandex/reference_en.html#SET)"章节

和原生接口不同，HTTP接口不支持会话，会话设置等概念，也不允许打断一个请求（少部分情况下可以），不显示查询进度。解析数据格式等工作是在服务端进行，网络利用效率可能不高。

可选的参数：'query_id'，可以穿入任何字符串作为queryID，更多信息："[Settings, replace\_running\_query](https://clickhouse.yandex/reference_en.html#replace_running_query)"

可选参数：'quota_key'，传入配额信息，具体参考：[Quotas](https://clickhouse.yandex/reference_en.html#Quotas)

HTTP接口允许传入额外的数据（外部临时表），更多信息参考：[External data for query processing](https://clickhouse.yandex/reference_en.html#External data for query processing),[查询时使用外部数据](https://github.com/sparkthu/clickhouse-doc-cn/blob/master/02use/external.md)

## JDBC 驱动
有一个官方的JDBC驱动：[这里](https://github.com/yandex/clickhouse-jdbc)

## 第三方客户端
包括[Python](https://github.com/Infinidat/infi.clickhouse_orm), PHP([1](https://github.com/8bitov/clickhouse-php-client),[2](https://github.com/SevaCode/PhpClickHouseClient),[3](https://github.com/smi2/phpClickHouse)),[Go](https://github.com/roistat/go-clickhouse), Node.js([1](https://github.com/TimonKK/clickhouse), [2](https://github.com/apla/node-clickhouse), [Perl](https://github.com/elcamlost/perl-DBD-ClickHouse).

所有的库都没有经过官方测试，排序是随机的。

## 第三方GUI
有一个简单的WebUI：[SMI2](https://github.com/smi2/clickhouse-frontend)

## 原生接口（TCP）
原生接口是用来在clickhouse-client的命令行工具和服务器之间进行分布式查询的交互的，同样可以用来做C++程序接口。下面将仅介绍命令行工具：

### 命令行工具

<pre>
$ clickhouse-client
ClickHouse client version 0.0.26176.
Connecting to localhost:9000.
Connected to ClickHouse server version 0.0.26176.

:) SELECT 1
</pre>

clickhouse-client程序可用的参数如下：
--host, -h - server name, by default - 'localhost'.
IPv4或IPv6均可

--port - The port to connect to, by default - '9000'.
注意HTTP端口和TCP端口不同，这里要使用TCP端口

--user, -u - The username, by default - 'default'.

--password - The password, by default - empty string.

--query, -q - Query to process when using non-interactive mode.
非交互模式下使用

--database, -d - Select the current default database, by default - the current DB from the server settings (by default, the 'default' DB).要使用的DB名称

--multiline, -m - If specified, allow multiline queries (do not send request on Enter).如果指定了，则允许多行请求，回车时不发送请求

--multiquery, -n - If specified, allow processing multiple queries separated by semicolons.
Only works in non-interactive mode.如果有，可以执行分号连接的多个请求，只能在非交互模式下使用

--format, -f - Use the specified default format to output the result.指定返回数据的格式

--vertical, -E - If specified, use the Vertical format by default to output the result. This is the same as '--format=Vertical'. In this format, each value is printed on a separate line, which is helpful when displaying wide tables. 纵向展示，在展示宽表时很有用

--time, -t - If specified, print the query execution time to 'stderr' in non-interactive mode.在非交互模式下，将执行时间输出到stderr

--stacktrace - If specified, also prints the stack trace if an exception occurs.指定后，如果发生异常，打印出堆栈信息

--config-file - Name of the configuration file that has additional settings or changed defaults for the settings listed above.
指定额外的配置文件的文件名或修改上述配置的默认值，默认情况下，会搜索以下路径：
 
    ./clickhouse-client.xml
    ~/./clickhouse-client/config.xml
    /etc/clickhouse-client/config.xml
 
 只有第一个找到的文件会被使用
 
 同样还可以指定任何查询时会用到的设置，例如clickhouse-client --max_threads=1，更多信息参考'[Settings](https://clickhouse.yandex/reference_en.html#Settings)'章节
 
 客户端可以在交互模式或非交互模式（批量模式）下工作。要使用批量模式，指定'query'参数，或把数据传入stdin（会自动判断stdin不是terminal），或同时这两种。
 类似HTTP接口，当使用'query'参数并把数据传入'stdin'中时，请求会在'query'参数和stdin中的内容之间插入一个换行，对于大型的INSERT操作很有用。
 
 插入数据的示例：
 <pre>
 echo -ne "1, 'some text', '2016-08-14 00:00:00'\n2, 'some more text', '2016-08-14 00:00:01'" | clickhouse-client --database=test --query="INSERT INTO test FORMAT CSV";

cat <<_EOF | clickhouse-client --database=test --query="INSERT INTO test FORMAT CSV";
3, 'some text', '2016-08-14 00:00:00'
4, 'some more text', '2016-08-14 00:00:01'
_EOF

cat file.csv | clickhouse-client --database=test --query="INSERT INTO test FORMAT CSV";
</pre>

在批量模式下，默认的数据格式是tab分割的，可以在query中指定格式

默认情况下一个语句只能执行一条请求，要执行多条，可以使用multiquery参数，除了INSERT外的查询都可以。多条的查询请求会不加分隔的连续输出。

类似的，要执行大量的查询时，你也可以为每条请求执行一次clickhouse-client，每次程序启动大概需要花费几十ms

在交互模式下，会有一个命令行供输入请求。

默认情况下multiline未开启，此时要执行请求，输入Enter。末尾不需要分号。此时可以通过Enter前输入反斜杠 \ 来输入多行请求。

在multiline开启后，要执行请求输入分号并回车。如果行尾没有分号，会被要求继续输入下一行。任何分号后的内容会被忽略。

通过\G替代分号或加在分号后，可以使用纵向输出的格式，用来和MySQL CLI兼容。

命令行基于readline(以及history，或libedit），也就是说可以使用常见的快捷键，并保持输入历史，历史文件在/.clickhouse-client-history(可能是~/...?)

默认情况下，输出格式为PrettyCompact,可以通过FORMAT来改变。

要退出客户端，可以使用Ctrl+D,Ctrl+C或以下输入：
"exit", "quit", "logout", "учше", "йгше", "дщпщге", "exit;", "quit;", "logout;", "учшеж", "йгшеж", "дщпщгеж", "q", "й", "\q", "\Q", ":q", "\й", "\Й", "Жй"

当执行请求时，客户端会显示：
1. 进度信息，不超过每秒10次的刷新。对于快速的请求，可能没有时间显示
2. 解析后的格式化请求信息，用来调试
3. 指定格式的结果
4. 结果的行数，花费的时间，平均处理速度

要取消一个长时间的查询，使用Ctrl+C，然而需要等待一小段时间来打断改请求。在某些环节无法取消请求。如果等不及再次按下了Ctrl+C，客户端会退出。

客户端工具也可以传输额外的信息（外部临时表），更多信息参考："[External data for request processing](https://clickhouse.yandex/reference_en.html#External data for query processing),[查询时使用外部数据](https://github.com/sparkthu/clickhouse-doc-cn/blob/master/02use/external.md)"
