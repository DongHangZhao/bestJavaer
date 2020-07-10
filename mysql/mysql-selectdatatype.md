# MySQL 选择合适的数据类型

我们会经常遇见的一个问题就是，在建表时如何选择合适的数据类型，通常选择合适的数据类型能够提高性能、减少不必要的麻烦，下面我们就来一起探讨一下，如何选择合适的数据类型。

### CHAR 和 VARCHAR 的选择

char 和 varchar 是我们经常要用到的两个存储字符串的数据类型，char 一般存储定长的字符串，它属于固定长度的字符类型，比如下面

| 值         | char(5) | 存储字节 |
| ---------- | ------- | -------- |
| ''         | '     ' | 5个字节  |
| 'cx'       | 'cx   ' | 5个字节  |
| 'cxuan'    | 'cxuan' | 5个字节  |
| 'cxuan007' | 'cxuan' | 5个字节  |

可以看到，不管你的值写的是什么，一旦指定了 char 字符的长度，如果你的字符串长度不够指定字符的长度的话，那么就用空格来填补，如果超过字符串长度的话，只存储指定字符长度的字符。

>这里注意一点：如果 MySQL 使用了非 `严格模式`的话，上面表格最后一行是可以存储的。如果 MySQL 使用了 `严格模式` 的话，那么表格上面最后一行存储会报错。

如果使用了 varchar 字符类型，我们来看一下例子

| 值         | varchar(5) | 存储字节 |
| ---------- | ---------- | -------- |
| ''         | ''         | 1个字节  |
| 'cx'       | 'cx '      | 3个字节  |
| 'cxuan'    | 'cxuan'    | 6个字节  |
| 'cxuan007' | 'cxuan'    | 6个字节  |

可以看到，如果使用 varchar 的话，那么存储的字节将根据实际的值进行存储。你可能会疑惑为什么 varchar 的长度是 5 ，但是却需要存储 3 个字节或者 6 个字节，这是因为使用 varchar 数据类型进行存储时，默认会在最后增加一个字符串长度，占用1个字节（如果列声明的长度超过255，则使用两个字节）。varchar 不会填充空余的字符串。

一般使用 char 来存储定长的字符串，比如**身份证号、手机号、邮箱等**；使用 varchar 来存储不定长的字符串。由于 char 长度是固定的，所以它的处理速度要比 VARCHAR 快很多，但是缺点是浪费存储空间，但是随着 MySQL 版本的不断演进，varchar 数据类型的性能也在不断改进和提高，所以在许多应用中，VARCHAR 类型更多的被使用。

在 MySQL 中，不同的存储引擎对 CHAR 和 VARCHAR 的使用原则也有不同

* MyISAM：建议使用固定长度的数据列替代可变长度的数据列，也就是 CHAR
* MEMORY：使用固定长度进行处理、CHAR 和 VARCHAR 都会被当作 CHAR 处理
* InnoDB：建议使用 VARCHAR 类型

### TEXT 与 BLOB

一般在保存较少的文本的时候，我们会选择 CHAR 和 VARCHAR，在保存大数据量的文本时，我们往往选择 TEXT 和 BLOB；TEXT 和 BLOB 的主要差别是 BLOB 能够保存`二进制数据`；而 TEXT 只能保存`字符数据`，TEXT 往下细分有

* TEXT
* MEDIUMTEXT
* LONGTEXT

BLOB 往下细分有

* BLOB
* MEDIUMBLOB
* LONGBLOB

三种，它们最主要的区别就是存储文本长度不同和存储字节不同，用户应该根据实际情况选择满足需求的最小存储类型，下面主要对 BLOB 和 TEXT 存在一些问题进行介绍

TEXT 和 BLOB 在删除数据后会存在一些性能上的问题，为了提高性能，建议使用 `OPTIMIZE TABLE` 功能对表进行碎片整理。

也可以使用合成索引来提高文本字段（BLOB 和 TEXT）的查询性能。合成索引就是根据大文本（BLOB 和 TEXT）字段的内容建立一个散列值，把这个值存在对应列中，这样就能够根据散列值查找到对应的数据行。一般使用散列算法比如 md5() 和 SHA1() ，如果散列算法生成的字符串带有尾部空格，就不要把它们存在 CHAR 和 VARCHAR 中，下面我们就来看一下这种使用方式

首先创建一张表，表中记录 blob 字段和 hash 值

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132811453-1444792840.png)

向 cxuan005 中插入数据，其中 hash 值作为 info 的散列值。

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132817165-601198698.png)

然后再插入两条数据

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132823617-1807595218.png)

插入一条 info 为 cxuan005 的数据

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132833167-1066282615.png)

如果想要查询 info 为 cxuan005 的数据，可以通过查询 hash 列来进行查询

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132842239-285583278.png)

这是合成索引的例子，如果要对 BLOB 进行模糊查询的话，就要使用前缀索引。

其他优化 BLOB 和 TEXT 的方式：

* 非必要的时候不要检索 BLOB 和 TEXT 索引
* 把 BLOB 或 TEXT 列分离到单独的表中。

### 浮点数和定点数的选择

浮点数指的就是含有小数的值，浮点数插入到指定列中超过指定精度后，浮点数会四舍五入，MySQL 中的浮点数指的就是 `float` 和 `double`，定点数指的是 `decimal`，定点数能够更加精确的保存和显示数据。下面通过一个示例讲解一下浮点数精确性问题

首先创建一个表 cxuan006 ，只为了测试浮点数问题，所以这里我们选择的数据类型是 float

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132849801-480771977.png)

然后分别插入两条数据

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132856319-580952636.png)

然后执行查询，可以看到查询出来的两条数据执行的舍入不同

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132906606-150784557.png)

为了清晰的看清楚浮点数与定点数的精度问题，再来看一个例子

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132919095-147664610.png)

先修改 cxuan006 的两个字段为相同的长度和小数位数

然后插入两条数据

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132925328-173481375.png)

执行查询操作，可以发现，浮点数相较于定点数来说，会产生误差

![](https://img2020.cnblogs.com/blog/1515111/202007/1515111-20200706132931982-1014481538.png)

### 日期类型选择

在 MySQL 中，用来表示日期类型的有 **DATE、TIME、DATETIME、TIMESTAMP**，在

[138 张图带你 MySQL 入门](https://mp.weixin.qq.com/s?__biz=MzU2NDg0OTgyMA==&mid=2247488824&idx=1&sn=fd7a3bfb1840fb303edfe71e2047992d&chksm=fc45e8cbcb3261dd0862d298a062956283ac0f9e564134356e3396245c5945515b029b6a2ac5&token=392068071&lang=zh_CN#rd)

这篇文中介绍过了日期类型的区别，我们这里就不再阐述了。下面主要介绍一下选择

* TIMESTAMP 和时区相关，更能反映当前时间，如果记录的日期需要让不同时区的人使用，最好使用 TIMESTAMP。
* DATE 用于表示年月日，如果实际应用值需要保存年月日的话就可以使用 DATE。
* TIME 用于表示时分秒，如果实际应用值需要保存时分秒的话就可以使用 TIME。
* YEAR 用于表示年份，YEAR 有 2 位（最好使用4位）和 4 位格式的年。 默认是4位。如果实际应用只保存年份，那么用 1 bytes 保存 YEAR 类型完全可以。不但能够节约存储空间，还能提高表的操作效率。
