## Cassandra 的数据模型

### 行

Cassandra中一行的数据结构：

<img src=".\image\image-20241102111008335.png" alt="image-20241102111008335" style="zoom: 80%;" />

每一个列都是一个`key-value`对，一行中可以包含多个列。每一行的唯一标识称为 row key 或 primary key。

### 表

如果一行中的某些列没有值，不会为那些列存储null值，那会浪费空间。实际上，那一行根本不会存储那些列。

Cassandra的表是一个**稀疏**的多维数组结构：

<img src=".\image\image-20241102113800867.png" alt="image-20241102113800867" style="zoom:67%;" />

### 主键、分区、静态列

Cassandra中的主键是复合主键（Composite Key），它可以分成两部分。第一部分是**分区键（partition key）**，第二部分是**集群列（clustering column）**。

一个表中所有的行根据分区键可以被划分成多个**分区（partition）**。分区键就是用于确定一行数据存在哪个分区上。分区键本身也可以包含多个列。

集群列用于控制在一个分区内部，行数据是如何排序的。一个partition中行的顺序由clustering column确定。

包含在主键中的列都必须有值，因为这些列将共同唯一标识一行数据。

在表中增加新的一行时，必须为每一个主键列都提供一个值。不过对于非主键列则没有这个要求，不需要为每一个列都提供一个值。

Cassandra还支持**静态列（Static Column）**，静态列用于存储不属于主键但是由**一个分区内所有行共享**的那些数据。

![image-20241102142318219](.\image\image-20241102142318219.png)

### 键空间

**keyspace** 对应关系型数据库中的 database，它是表的容器。

### insert和update

由于Cassandra使用的是一种追加模型，所以insert与update操作之间并没有本质区别。

如果insert一行的主键与已存在的一行的主键完全相同，已存在的那一行就会被替换。

如果要update一行，而指定的主键不存在，则Cassandra就会创建那一行。

### 列的时间戳

每次向Cassandra写入数据时，会为新增或更新的**每个列**生成一个时间戳（单位是微秒）。

这个时间戳是用来解决并发写冲突的。

如果多个写操作试图修改同一个值，Cassandra在内部就使用那个列的时间戳来解决冲突，只保留最后写入的值，这也被称作**最后写获胜（last write win）**。

使用writetime函数获取一个列上的时间戳：

```CQL
select title, writetime(title) from user;
```

如果一个列上没有值，那时间戳也是null。

**不能查询主键列上的时间戳**，这也合理，因为主键列上的值不能被修改。

Cassandra允许客户在完成写操作时指定想要使用的时间戳，这要在update时使用可选的`USING TIMESTAMP`选项。手动设置的时间戳必须晚于列值当前的时间戳，否则写操作会被忽略。

```CQL
UPDATE user USING TIMESTAMP 1567886623298888 SET middle_initial = 'Q' 
WHERE first_name = 'Mary' AND last_name = 'RRRR';
```

### 生存时间TTL

TTL 是 Cassandra 为**各个列值**存储的一个值，用来指示这个值要保存多久。

TTL 值默认是null，这表示写入的数据永不过期。

用 TTL()  函数查看一个列值的 TTL 值：

```CQL
SELECT TTL(title) FROM user WHERE first_name = 'Mary' AND last_name = 'RRRR';
```

使用`USING TTL`选项将 middle_initial 列的TTL设置为1小时：

```CQL
UPDATE user USING TTL 3600 SET middle_initial = 'Z'
WHERE first_name = 'Mary' AND last_name = 'RRR';
```

如果一小时再查询这行记录，Mary的 middle_initial 就会被删除（设置成null）。

还可以在INSERT操作时设置TTL让整个一行数据过期：

```CQL
INSERT INTO user (first_name, last_name) values ('Jeff', 'Carpenter')
USING TTL 60;
```

这行记录被加入一分钟之后，这行记录会被自动删除。

## CQL 类型

### 数值类型

|      Cassandra类型       |    等价的Java类型    |
| :----------------------: | :------------------: |
|           int            |         int          |
|          bigint          |         long         |
|         smallint         |        short         |
| tinyint（8位有符号整数） |       tinyint        |
|          varint          | java.math.BigInteger |
|          float           |        float         |
|          double          |        double        |
|         decimal          | java.math.BigDecimal |

### 字符串类型

- text：和varchar是同义词，表示 UTF-8 字符串
- ascii：表示 ASCII 字符串

### 时间类型

#### timestamp

时间戳，可以编码为一个64位有符号整数，也可以以日期格式输入时间戳。

```java
2024-06-15 20:05-0700
```

#### date

只表示日期，不带时间。映射到Cassandra中的一个定制类型。

#### time

只表示时间，不带日期。映射到Java中的long，表示午夜以来的纳秒数。

### 标识数据类型

#### uuid

是一个128位的值。表示为短横线（-）分隔的十六进制数序列。

可以通过CQL中的 `uuid()` 函数生成一个UUID值，并在INSERT或UPDATE操作中使用这个值。

#### timeuuid

这类uuid在生成的时候嵌入了时间信息，基于时间戳和节点ID等信息生成。这使得timeuuid可以**按照时间顺序排序**。

适用于 日志记录 等对数据操作的先后顺序进行追踪和管理的场景

### 其它基本类型

#### boolean类型

true / false

#### blob类型

二进制大对象（binary large object）。可用于存储媒体或其它二进制文件。

#### inet

表示 IPv4 或 IPv6 或联网地址。

#### counter

对应一个64位有符号整数，它的值不能直接设置，而只能自增或自减。

counter常用于跟踪统计结果，如页面访问量，日志消息数等。

counter不能作为主键的一部分。如果使用了counter，那除了主键以外的所有其它列都必须是counter。

### cqlsh实践

对于keyspace、table 和 column的名字，无论输入的是大写还是小写字母，Cassandra都会将它们处理为**小写**字母。

所以通常这些名字都会采用小写字母加下划线（_）分隔的形式。

#### 创建keyspace：

```CQL
-- specify replication strategy and replication factor. 
CREATE KEYSPACE study_keyspace WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
```

#### 查看Cassandra有哪些keyspace：

```CQL
describe keyspaces
```

![image-20241103095019954](.\image\image-20241103095019954.png)

#### 查看单个keyspace的详细信息：

```CQL
DESCRIBE KEYSPACE study_keyspace
```

![image-20241103095710884](.\image\image-20241103095710884.png)

```CQL
CREATE KEYSPACE study_keyspace 
WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'}  
AND durable_writes = true;
```

多了一个 `durable_writes = true`。

#### 切换到目标keyspace：

这句后面必须要有 `;`。

```CQL
use study_keyspace;
```

![image-20241103100311618](.\image\image-20241103100311618.png)

#### 创建表：

```CQL
CREATE TABLE user
(
    first_name text,
    last_name  text,
    title      text,
    PRIMARY KEY ( last_name, first_name )
)
```

#### 查看表结构：

```CQL
-- 查看当前keyspace中包含的所有表
describe tables;
-- 查看一个表的详细定义信息
describe table user;
```

![image-20241103110923168](.\image\image-20241103110923168.png)

#### CRUD：

根 SQL 是一样的

```CQL
SELECT * FROM user;
INSERT INTO user (first_name, last_name, title) VALUES ('Bill', 'Nguyen', 'Mr.');
SELECT * FROM user WHERE first_name = 'Bill' AND last_name = 'Nguyen';
UPDATE user set title = 'Engineer' where first_name = 'Bill' AND last_name = 'Nguyen';
DELETE FROM user where first_name = 'Bill' AND last_name = 'Nguyen';
```

![image-20241103111804148](.\image\image-20241103111804148.png)

**查询必须包含分区键**，不包含分区键的查询会报错。但只包含分区键，不包含集群键的查询可以work：

![image-20241103112647015](.\image\image-20241103112647015.png)

DELETE 可以只删除**单独一列**中的值：

```CQL
DELETE title FROM user WHERE first_name = 'Bill' AND last_name = 'Nguyen';
```

![image-20241103114220415](.\image\image-20241103114220415.png)

如果 DELETE 命令后面不接任何列，那就是删除一整行。

#### Count

```CQL
SELECT COUNT(*) FROM user;
```

![image-20241103113531919](.\image\image-20241103113531919.png)

注意 `Warnings :Aggregation query used without partition key`

执行这个命令确实能得到正确的行数，但同时也会得到一个warning。这是因为，你要求Cassandra完成一个全表扫描操作。在一个可能有大量数据的多节点集群中，count 将是一个非常昂贵的操作。

#### 清除

```CQL
-- 删除 user 表中的全部数据
TRUNCATE user;
-- 完全删除 user 表
DROP TABLE user;
```

#### TTL

![image-20241103115252245](.\image\image-20241103115252245.png)

过了60秒之后，那一行就自动被删除了。

### 集合

需要在一个列中存储数目可变的元素时，集合类型会非常有用。

#### set

修改表结构，增加emails列，它的类型是 `set<text>`：

```CQL
ALTER TABLE user ADD emails set<text>;
```

![image-20241103143103382](.\image\image-20241103143103382.png)

增加email的值：

```CQL
-- 用一个全新的set替换原来的值
UPDATE user SET emails = { 'test@qq.com' } WHERE last_name = 'Nguyen' AND first_name = 'Bill';
-- 利用拼接操作向set中增加一个元素
UPDATE user SET emails = emails + { 'another_test@qq.com' } WHERE last_name = 'Nguyen' AND first_name = 'Bill';
```

![image-20241103143608543](.\image\image-20241103143608543.png)

set中的元素在查询时会以一种有序的顺序返回结果，如上面的例子是按照字母表顺序返回结果。set对应Java中的**TreeSet**。

用`+`向set增加元素，用`-`可以删除set中对应的元素：

```CQL
UPDATE user SET emails = emails - { 'another_test@qq.com' } WHERE last_name = 'Nguyen' AND first_name = 'Bill';
```

![image-20241103144503479](.\image\image-20241103144503479.png)

#### list

list中的值会按照加入的顺序排序：

```CQL
ALTER TABLE user ADD phone_numbers list<text>;
UPDATE user SET phone_numbers = ['123-456-789'] WHERE last_name = 'Nguyen' AND first_name = 'Bill';
```

![image-20241103144953919](.\image\image-20241103144953919.png)

list支持在其左右两边加入新的元素，并且可以加入重复的元素：

![image-20241103145259574](.\image\image-20241103145259574.png)

删除list中的一个元素：

```CQL
UPDATE user SET phone_numbers = phone_numbers - ['999-999-999'] WHERE last_name = 'Nguyen' AND first_name = 'Bill';
```

![image-20241103150124255](.\image\image-20241103150124255.png)

替换list中的一个元素：

```CQL
UPDATE user SET phone_numbers[0] = '999-999-999' WHERE last_name = 'Nguyen' AND first_name = 'Bill';
```

![image-20241103150407411](.\image\image-20241103150407411.png)

根据list中的元素索引删除元素：

```CQL
DELETE phone_numbers[0] FROM user WHERE last_name = 'Nguyen' AND first_name = 'Bill';
```

![image-20241103150720090](.\image\image-20241103150720090.png)

#### map

key 和 value 可以是除了 counter 以外的任何其它类型。

```CQL
ALTER TABLE user ADD login_session map<uuid, int>;
UPDATE user SET login_session = { uuid(): 13, uuid(): 18 } WHERE first_name = 'Bill' AND last_name = 'Nguyen';
```

![image-20241103161609034](.\image\image-20241103161609034.png)

向map中增加新的key-value对：

```CQL
UPDATE user SET login_session = login_session + { uuid(): 99, uuid(): 77 } WHERE first_name = 'Bill' AND last_name = 'Nguyen';
```

![image-20241103161901259](.\image\image-20241103161901259.png)

### 元组

一个定长的值集合。这些值可以有不同的类型。

```CQL
ALTER TABLE user ADD address tuple<text, text, text, int>;
UPDATE user SET address = ('aaa', 'bbb', 'ccc', 85715) WHERE first_name = 'Bill' AND last_name = 'Nguyen';
```

![image-20241103162831880](.\image\image-20241103162831880.png)

元组是不可变的，不能只更新元组中的单个字段，必须替换整个元组。

查询的时候也不能只获得元组中的单个元素值，必须获取整个元组的值。

删除 address 列：

```CQL
ALTER TABLE user DROP address;
```

### 自定义类型

```CQL
CREATE TYPE address (
	street text,
    city text,
    state text,
    zip_code int
);
```

**用户自定义类型（UDT）**的作用域仅限于定义这个UDT的**键空间**。

UDT 和 表 都成为键空间定义的一部分：

![image-20241103163808015](.\image\image-20241103163808015.png)

在表中使用address类型：

```CQL
ALTER TABLE user ADD addresses map<text, frozen<address>>;
```

注意frozen关键字，由于历史原因，需要加这个关键字。

```CQL
UPDATE user SET addresses = {'home': {street: 'aaa', city: 'bbb', state: 'AZ', zip_code: 12345}} where first_name = 'Bill' AND last_name = 'Nguyen';
```

![image-20241103165123011](.\image\image-20241103165123011.png)

```CQL
SELECT addresses['home'] FROM user;
```

![image-20241103165354640](.\image\image-20241103165354640.png)



 