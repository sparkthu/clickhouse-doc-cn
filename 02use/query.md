查询语言
=============

#语法
ClickHouse中有两种解析器，一个是全SQL解析器（递归下降分析器），一个数据格式解析器（快速流解析器），第二个仅用于INSERT语句。INsert语句使用了两种解析器

INSERT INTO t VALUES (1, 'Hello, world'), (2, 'abc'), (3, 'def')


INSERT INTO t VALUES部分使用全解析器，数据部分使用流解析器。

数据可以使用任何格式，接收到请求后，服务器将不超过’max_query_size’的部分放在RAM中解析（默认为1MB），剩余部分使用流解析器。因此和MySQL一样，对于很大的INSERT语句也不会有任何问题。

在INSERT语句中使用使用Values格式时，数据看上去可以和SELECT语句的表达式进行相同的解析，然而并不是。Values的格式限制多很多。
接下来我们要讲全解析器，对于解析的更多信息，参考Formats章节(https://clickhouse.yandex/reference_en.html#Formats)
 。

###空白

语法结构之间和前后可以有任意数量的空白字符，包括空格，tab，换行，CR，form feed.

###注释

SQL和C风格的注释都是允许的，SQL风格： 从—到行尾，—之后的空格可以省略
C风格：从/*到*/，允许多行。

###关键字

类似SELECT这种的关键字是大小写不敏感的，但和标准SQL不同，其他任何词（包括列名，方法名等）都是大小写敏感的。关键字使用的单词不是保留的，可以在其他命名中使用，他们仅在合适的上下文中被解析为关键字。

###标识符
标识符（列名，方法名，数据类型）可以使用引号，也可以不用。

不使用引号的标识符首字符必须是拉丁字符或下划线，后面可以是拉丁字符，下划线或数字，也就是满足这个正则表达式：<pre>^[a-zA-Z_][0-9a-zA-Z_]*$</pre>
例如这些是合法的：x, _1, X_y__Z123_

引号中的标识符：和MySQL一样，使用`id`的形式，中间可以使用任何字符，特殊字符（比如`符号本身）可以使用反斜杠转义。转义规则和字符串常量相同。我们推荐使用无需括号的标识符。

###常量

包括数字常量，字符串常量和混合常量

####数字常量

数字常量的解析方式顺序：

1. 首先使用64位有符号数，使用'strtoull'方法
2. 如果不成功，使用64位无符号数，'strtoll'方法
3. 还不成功，使用浮点数，'strtod'方法
4. 再不成功则返回错误

响应数据则使用结果能够适应的最小类型，例如1使用Uint8,而256使用Uint16,更多信息参考[Data types](https://clickhouse.yandex/reference_en.html#Data types)

####字符串常量

字符串常量必须使用单引号包裹。其中的特殊字符可以使用反斜杠转义。这些转义字符有特殊含义：

    \b, \f, \r, \n, \t, \0, \a, \v, \xHH
    
其他情况下，对于任意字符c,\c被转成字符c本身，也就是\\表示\，\'表示'字符。

####混合常量

数组中支持混合常量，例如：[1, 2, 3]，还有元组，例如：(1, 'Hello, world!', 2)

事实上他们并不是常量，而是使用了数组构造函数或元组构造函数的表达式。更多信息参考[Operators](https://clickhouse.yandex/reference_en.html#Operators2)

数组至少包含一个元素，元组至少包含两个元素。元组是专门用于匹配IN从句中的SELECT请求结果的。通常情况下不能被存储到数据库中，（Memory表除外）

### 函数

函数写起来就是一个标识符，后面跟着一串（可能为空）的用小括号括起来的参数。和标准的SQL不同，即使是空的参数列表，括号也是必须的，比如now().

函数分为普通函数和聚合函数。某些聚合函数可以跟两组参数，比如quantile(0.9)(x)。这种聚合函数被称为参数化函数，第一组参数就是参数化方法中的参数。对于没有带参数的聚合方法语法就和普通方法一样。

### 运算符

运算符在解析请求时会被转换为对应的函数，同时考虑运算符的优先级和结合性，例如 1+2+3+4被转换为 plus(plus(1, multiply(2, 3)), 4)，更多内容请参考[Operators](https://clickhouse.yandex/reference_en.html#Operators2)

### 数据类型和表引擎

在CREATE语句中，数据类型和表引擎写法和标识符及函数一致。也就是说他们可能含有参数列表，也可能不含。更多信息参考[Data types](https://clickhouse.yandex/reference_en.html#Data types), "Table engines"以及"create"

### 别名

在SELECT请求中，表达式可以使用AS语法指定别名。表达式在AS左边，别名在AS右边。和标准SQL不同，别名不仅仅可以在表达式顶层指定，例如一下是合法的：

    SELECT (1 AS n) + 2, n
    
和标准SQL不同，别名可以用在任何语句，而不仅仅可以用在SELECT中。

###Asterisk

###表达式

表达式可以是函数，标识符，常量，运算符应用，包在括号中，放在子查询中，Asterisk中等。表达式可以包含别名。多个表达式使用逗号分隔。函数和运算符也可以使用表达式作为参数。

# 请求
###建库

    CREATE DATABASE [IF NOT EXISTS] db_name

创建一个名为db_name的数据库，数据库仅仅是一系列表的目录。指定了 IF NOT EXISTS后，如果数据库已经存在，语句不会返回错误。

###建表

建表请求有多种格式

<pre>
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] [db.]name
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = engine
</pre>

在db数据库中简历一个name的表，如果没有指定db则使用当前数据库。结构是括号中指定的格式，引擎使用指定的引擎。表结构是所有列描述的列表。如果引擎支持索引，可以在引擎的参数中指定。

列描述最简单的情况是"name type",例如RegionID UInt32，也可以用表达式指定默认值,下面会有详细说明。

    CREATE [TEMPORARY] TABLE [IF NOT EXISTS] [db.]name AS [db2.]name2 [ENGINE = engine]
    
这个语句使用另一个表的结构建表，如果没有指定引擎，则会使用db2.name2的引擎。

    CREATE [TEMPORARY] TABLE [IF NOT EXISTS] [db.]name ENGINE = engine AS SELECT ...
    
上面的语句使用SELECT的结果作为结构建表，同时把SELECT结果的数据插入表中，引擎可以单独指定。

在所有上面的方法中，IF NOT EXISTS可以发挥作用。

#### 默认值

列可以使用下面的语法指定一个默认值：DEFAULT expr, MATERIALIZED expr, ALIAS expr。 例如URLDomain String DEFAULT domain(URL)

如果没有指定默认值，对于数字会默认为0，字符串为空字符串，数组为空数组，日期为0000-00-00，时间为0000-00-00 00:00:00，不支持NULL

如果指定了默认值，类型可以省略，这种情况下使用默认值的类型，例如EventDate DEFAULT toDate(EventTime)，EventDate的类型会被设置为Date。如果明确指定了类型，默认值的会被转换为目标类型。
例如Hits UInt32 DEFAULT 0 和 Hits UInt32 DEFAULT toUInt32(0)是完全相同的。

默认值的表达式中也可以使用表常量和其他列。建表和修改表结构时，会检查是否包含循环，INSERT语句中，会检查是否表达式中的所有列都已经传入了。

"DEFAULT expr"表示普通的默认值，如果INSERT语句没有指定相应列，会自动使用表达式的值填充

"MATERIALIZED expr"表示“物化列”，这样的列不允许在INSERT中指定，因为它必须计算得来。对于没有指定列的INSERT语句，这些列不会被考虑在内，使用SELECT *时，这些列也不会返回，因为将SELECT * 的结果使用不指定列的INSERT语句插入式很常见的行为，会引发问题。

"ALIAS expr"是指别名，这样的列根本不会被存储起来。不能够使用INSERT插入数据，也不会包括在SELECT * 的返回值中，如果别名是在请求解析阶段展开的，可以使用在SELECT语句中。

当使用ALTER请求来添加新列时，这些列的老数据不会被写入。当读取的老数据没有这些新列的值时，默认情况下会在线计算。然而如果这些表达式使用了没有在SELECT语句中出现的列，这些列会被读取到，当然仅在需要的情况下。

如果向表中添加了新列，但是后来又修改了默认表达式，老数据的值将会更新。当运行后台合并操作时，原来没有值的列会在合并后被写入。

不允许为嵌套的数据结构设置默认值。

####临时表

如果指定了TEMPORARY，将会创建临时表，临时表的特征：

- 连接结束或中断时，临时表会消失。
- 临时表只能使用内存引擎，其它引擎不支持。
- 临时表不能指定数据库，他们是在数据库外创建的
- 如果临时表和普通表的名字相同，请求没有指定db的名字时，临时表会被使用
- 在分布式处理中，一个请求用到的临时表会传给远端server。

通常情况下，临时表不是手动建立的，而是使用外部数据，或使用分布式的IN语句时被自动创建。
