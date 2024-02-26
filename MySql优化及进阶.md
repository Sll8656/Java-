# MySql优化及进阶
## 一、MySQL体系结构

- 连接层：是一些客户端和链接服务，包含本地sock 通信和大多数基于客户端/服务端工具实现的类似于 TCP/IP的通信
- 服务层：大多数的核心服务功能，如SQL接口，并完成缓存的查询，SQL的分析和优化，部 分内置函数的执行
- 引擎层：负责了MySQL中数据的存储和提取，服务器通过API和存储引擎进行通 信，其中索引 存储在该层中
- 存储层：主要是将数据(如: redolog、undolog、数据、索引、二进制日志、错误日志、查询 日志、慢查询日志等)存储在文件系统之上，并完成与存储引擎的交互

## 二、存储引擎

- InnoDB：默认的存储引擎 
   - 特点： 
      - 遵循ACID模型，**支持事务**
      - **行级锁**，提高并发访问性能
      - **支持外键**FOREIGN KEY约束，保证数据的完整性和正确性
   - 逻辑存储结构： 
      - 表空间 : InnoDB存储引擎逻辑结构的最高层，ibd文件其实就是表空间文件，在表空间中可以 包含多个Segment段。
      - 段 : 表空间是由各个段组成的， 常见的段有数据段、索引段、回滚段等。InnoDB中对于段的管 理，都是引擎自身完成，不需要人为对其控制，一个段中包含多个区。
      - 区 : 区是表空间的单元结构，每个区的大小为1M。 默认情况下， InnoDB存储引擎页大小为 16K， 即一个区中一共有64个连续的页。
      - 页 : 页是组成区的最小单元，页也是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默 认为 16KB。为了保证页的连续性，InnoDB 存储引擎每次从磁盘申请 4-5 个区。
      - 行 : InnoDB 存储引擎是面向行的，也就是说数据是按行进行存放的，在每一行中除了定义表时 所指定的字段以外，还包含两个隐藏字段(后面会详细介绍)。
- MyISAM： 
   - 特点： 
      - 不支持事务，不支持外键
      - 支持表锁，不支持行锁
      - 访问速度快

## 三、索引

> 索引（index）是帮助MySQL高效获取数据的**数据结构(有序)**。
>  
> - 数据结构支持：B+树、Hash索引   ，大部分
> - 数据结构演示网址：[B+ Tree Visualization (usfca.edu)](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)
> - B+树的结构特点： 
>    - 所有的数据都会出现在叶子节点。
>    - 叶子节点形成一个单向链表。
>    - 非叶子节点仅仅起到索引数据作用，具体的数据都是在叶子节点存放的。
>    - 增加一个指向相邻叶子节点 的链表指针，就形成了带有顺序指针的B+Tree，提高区间访问的性能，利于排序
> 
 


索引分类：根据索引的类型分类

| 分类 | 含义 | 特点 | 关键字 |
| --- | --- | --- | --- |
| 主键 索引 | 针对于表中主键创建的索引 | 默认自动创建, 只能 有一个 | PRIMARY |
| 唯一 索引 | 避免同一个表中某数据列中的值重复 | 可以有多个 | UNIQUE |
| 常规 索引 | 快速定位特定数据 | 可以有多个 |  |
| 全文 索引 | 全文索引查找的是文本中的关键词，而不是比 较索引中的值 | 可以有多个 | FULLTEXT |


根据索引的存储形式，又可以分为以下两种：聚集索引&二级索引（面试重点）

| 分类 | 含义 | 特点 |
| --- | --- | --- |
| 聚集索引(Clustered Index) | 将数据存储与索引放到了一块，索引结构的叶子 节点**保存了行数据** | 必须有,而且只 有一个 |
| 二级索引(Secondary Index) | 将数据与索引分开存储，索引结构的叶子节点**关联的是对应的主键** | 可以存在多个 |


> 选取规则：
>  
> - 如果存在主键，主键索引就是聚集索引。
> - 如果不存在主键，将使用第一个唯一（UNIQUE）索引作为聚集索引。
> - 如果表没有主键，或没有合适的唯一索引，则InnoDB会自动生成一个rowid作为隐藏的聚集索引。
> 
 
> 回表查询（面试重点）：当对非聚集索引进行查询的时候，首先根据二级索引找到值对应的主键id，此时返回的就是一个id，而不是该条数据的全部信息，然后还需要到聚集索引中根据id拿到所有的数据，这个操作叫做回表查询。
>  
> **如何解决回表查询**：使用联合索引，不会再次触发回表查询。


## 四、SQL性能分析

-  查看当前数据库的INSERT、UPDATE、DELETE、SELECT的访问频次：`SHOW GLOBAL STATUS LIKE 'Com_______'`   七个下划线 
-  慢查询日志：慢查询日志记录了所有执行时间超过指定参数（long_query_time，单位：秒，默认10秒）的所有 SQL语句的日志 
   -  查看是否开启慢查询日志：`show variables like 'slow_query_log'`，如果没有开启，需要在配置文件添加以下配置 
```sql
# 开启MySQL慢日志查询开关
slow_query_log=1

# 设置慢日志的时间为2秒，SQL语句执行时间超过2秒，就会视为慢查询，记录慢查询日志
long_query_time=2
```
 

   -  通过慢查询日志，就可以定位出执行效率比较低的SQL，从而有针对性的进行优化 
-  Profile详情： 能够在做SQL优化时帮助我们了解时间都耗费到哪里去了 
   -  查看当前MySql是否支持：`SELECT @@have_profiling;` ，开启：`SET profiling = 1;` 
   -  执行一系列的业务SQL的操作，然后通过如下指令查看指令的执行耗时： 
```sql
-- 查看每一条SQL的耗时基本情况
show profiles;

-- 查看指定query_id的SQL语句各个阶段的耗时情况
show profile for query query_id;

-- 查看指定query_id的SQL语句CPU的使用情况
show profile cpu for query query_id;
```
 

## 五、Explain关键字

> Explain用于获取 MySQL 如何执行 SELECT 语句的信息，包括在 SELECT 语句执行 过程中表如何连接和连接的顺序


语法：

```sql
-- 直接在select语句之前加上关键字 explain / desc

EXPLAIN SELECT 字段列表 FROM 表名 WHERE 条件 ;
```
| 字段 | 含义 |
| --- | --- |
| id | select查询的序列号，表示查询中执行select子句或者是操作表的顺序 (id相同，执行顺序从上到下；id不同，值越大，越先执行)。 |
| select_type | 表示 SELECT 的类型，常见的取值有 SIMPLE（简单表，即不使用表连接 或者子查询）、PRIMARY（主查询，即外层的查询）、 UNION（UNION 中的第二个或者后面的查询语句）、 SUBQUERY（SELECT/WHERE之后包含了子查询）等 |
| type | 表示连接类型，性能由好到差的连接类型为NULL、system、const、 eq_ref、ref、range、 index、all |
| possible_key | 显示可能应用在这张表上的索引，一个或多个。 |
| key | 实际使用的索引，如果为NULL，则没有使用索引。 |
| key_len | 表示索引中使用的字节数， 该值为索引字段最大可能长度，并非实际使用长 度，在不损失精确性的前提下， 长度越短越好 。 |
| rows | MySQL认为必须要执行查询的行数，在innodb引擎的表中，是一个估计值， 可能并不总是准确的。 |
| filtered | 表示返回结果的行数占需读取行数的百分比， filtered 的值越大越好。 |


## 六、索引的使用

### 1、索引失效情况

-  最左前缀法制：如果索引了多列（联合索引），要遵守最左前缀法则。最左前缀法则指的是查询从索引的**最左列开始**， 并且**不跳过索引中的列**。如果跳跃某一列，索引将会部分失效(后面的字段索引失效)。 
-  范围查询：联合索引中，出现范围查询(>,<)，范围查询右侧的列索引失效，而使用（>=  <=）不会让索引失效。 
-  索引列运算：不要在索引列上进行运算操作， 索引将失效。 
-  字符串不加引号：字符串类型字段使用时，不加引号，索引将失效。 
-  模糊查询：如果仅仅是尾部模糊匹配，索引不会失效。如果是头部模糊匹配，索引失效。 
-  or 连接条件：用or分割开的条件， 如果or前的条件中的列有索引，而后面的列中没有索引，那么涉及的索引都不会 被用到。 

> 数据分布影响： 如果MySQL评估使用索引比全表更慢，则不使用索引。


### 2、SQL提示

> 定义：是优化数据库的一个重要手段，简单来说，就是在SQL语句中加入一些人为的提示来达到优化操作的目的


```sql
#1、use index ： 建议MySQL使用哪一个索引完成此次查询（仅仅是建议，mysql内部还会再次进行评估）。
explain select * from tb_user use index(idx_user_pro) where profession = '软件工程';

#2、ignore index ： 忽略指定的索引。
explain select * from tb_user ignore index(idx_user_pro) where profession = '软件工程';

#3、force index ： 强制使用索引。
explain select * from tb_user force index(idx_user_pro) where profession = '软件工程';
```

### 3、覆盖索引

> 定义：指查询使用了索引，并且需要返回的列，在该索引中已经全部能够找到，减少使用select *

| Extra | 含义 |
| --- | --- |
| Using where; Using Index | 查找使用了索引，但是需要的数据都在索引列中能找到，所以不需 要回表查询数据 |
| Using index condition | 查找使用了索引，但是需要回表查询数据 |


> 思考题： 一张表, 有四个字段(id, username, password, status), 由于数据量大, 需要对 以下SQL语句进行优化, 该如何进行才是最优方案:
>  
> `select id,username,password from tb_user where username = 'admin';`
>  
> 答案: 针对于 username, password建立联合索引, sql为: create index idx_user_name_pass on tb_user(username,password); 这样可以避免上述的SQL语句，在查询的过程中，出现回表查询


### 4、前缀索引

> 定义：当字段类型为字符串（varchar，text，longtext等）时，有时候需要索引很长的字符串，这会让 索引变得很大，查询时，浪费大量的磁盘IO， 影响查询效率。此时可以只将字符串的一部分前缀，建 立索引，这样可以大大节约索引空间，从而提高索引效率。
>  
> 语法：`create index idx_xxxx on table_name(column(n)) ;`
>  
> 特性：前缀索引必须回表查询


前缀长度：可以根据索引的选择性来决定，而选择性是指不重复的索引值（基数）和数据表的记录总数的比值， 索引选择性越高则查询效率越高， 唯一索引的选择性是1，这是最好的索引选择性，性能也是最好的。

```sql
select count(distinct email) / count(*) from tb_user ;

select count(distinct substring(email,1,5)) / count(*) from tb_user;
```

### 5、使用规则及设计原则

-  使用规则 
   - 推荐使用联合索引，而不是单列索引
-  设计原则 
   - 针对于数据量较大，且查询比较频繁的表建立索引。
   - 针对于常作为查询条件（where）、排序（order by）、分组（group by）操作的字段建立索 引。
   - 尽量选择区分度高的列作为索引，尽量建立唯一索引，区分度越高，使用索引的效率越高。
   - 如果是字符串类型的字段，字段的长度较长，可以针对于字段的特点，建立前缀索引。
   - 尽量使用联合索引，减少单列索引，查询时，联合索引很多时候可以覆盖索引，节省存储空间， 避免回表，提高查询效率。
   - 要控制索引的数量，索引并不是多多益善，索引越多，维护索引结构的代价也就越大，会影响增 删改的效率。
   - 如果索引列不能存储NULL值，请在创建表时使用NOT NULL约束它。当优化器知道每列是否包含 NULL值时，它可以更好地确定哪个索引最有效地用于查询。

## 七、SQL优化

### 1、插入数据

-  insert优化 
   - 批量插入数据：`Insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');`
   - 手动控制事务：
```sql
start transaction;
insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
insert into tb_test values(4,'Tom'),(5,'Cat'),(6,'Jerry');
insert into tb_test values(7,'Tom'),(8,'Cat'),(9,'Jerry');
commit;
```

   - 主键顺序插入，性能要高于乱序插入。
-  大量数据插入：使用MySQL load指令进行插入操作，操作如下 
```sql
-- 客户端连接服务端时，加上参数 -–local-infile
mysql –-local-infile -u root -p

-- 设置全局参数local_infile为1，开启从本地加载文件导入数据的开关
set global local_infile = 1;

-- 执行load指令将准备好的数据，加载到表结构中
load data local infile '/root/sql1.log' into table tb_user fields
terminated by ',' lines terminated by '\n' ;
```
 

### 2、主键优化

- 满足业务需求的情况下，尽量降低主键的长度。
- 插入数据时，尽量选择顺序插入，选择使用AUTO_INCREMENT自增主键。
- 尽量不要使用UUID做主键或者是其他自然主键，如身份证号。
- 业务操作时，避免对主键的修改。

### 3、Order By优化

> MySQL的排序有两种方式，分别如下，对于以下的两种排序方式，Using index的性能高，而Using filesort的性能低，我们在优化排序 操作时，尽量要优化为 Using index


- Using filesort：通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区sort buffer中完成排序操作，所有不是通过索引直接返回排序结果的排序都叫 FileSort 排序。
- Using index :通过有序索引顺序扫描直接返回有序数据，这种情况即为 using index，不需要额外排序，操作效率高。

优化原则：

- A. 根据排序字段建立合适的索引，多字段排序时，也遵循最左前缀法则。
- B. 尽量使用覆盖索引，即使用索引的时候。
- C. 多字段排序, 一个升序一个降序，此时需要注意联合索引在创建时的规则（ASC/DESC）。
- D. 如果不可避免的出现filesort，大数据量排序时，可以适当增大排序缓冲区大小 sort_buffer_size(默认256k)。

### 4、Group By 优化

优化原则：

- A. 在分组操作时，可以通过索引来提高效率。
- B. 在分组操作时，索引的使用也是满足最左前缀法则的。

### 5、Limit 优化

优化原则：一般分页查询时，通过创建 覆盖索引 能够比较好地提高性能，可以通过覆盖索引加子查询形式进行优化。

```sql
explain select * from tb_sku t , (select id from tb_sku order by id limit 2000000,10) a where t.id = a.id;
```

### 6、count 优化

优化原则：按照效率排序的话，count(字段) < count(主键 id) < count(1) ≈ count(※)，所以尽量使用 count(*)

### 7、update优化

优化原则：InnoDB的行锁是针对索引加的锁，不是针对记录加的锁 ,并且该索引不能失效，否则会从行锁 升级为表锁 。所以update更新操作尽量对索引进行操作。


## 八、锁

锁的分类（按照锁的粒度分）：

-  全局锁：锁定数据库中的所有表。 
- 表级锁：每次操作锁住整张表。
-  行级锁：每次操作锁住对应的行数据。  
### 1、全局锁
定义： 全局锁就是对**整个数据库实例加锁**，加锁后整个实例就处于只读状态，后续的DML的写语句，DDL语 句，已经更新操作的事务提交语句都将被阻塞。  

应用场景： 做**_全库的逻辑备份_**，对所有的表进行锁定，从而获取一致性视图，保证数据的完整 性。  

语法：
1）、加全局锁：`flush tables with read lock;`
2）、数据备份：` mysqldump -uroot –proot test > test.sql `
3）、释放锁： `unlock tables ;`

注意： 在InnoDB引擎中，我们可以在备份时加上参数 `--single-transaction` 参数来完成**_不加锁的一致性数据备份_**  
```sql
 mysqldump --single-transaction -uroot –proot test > test.sql
```
### 2、表级锁
>  表级锁，每次操作锁住整张表。锁定粒度大，发生锁冲突的概率最高，并发度最低。应用在MyISAM、 InnoDB、BDB等存储引擎中。  


表级锁的分类：

- 表锁：
   -  表共享_**读锁**_（read lock）： _**不**_会**_影响_**客户端的_**读**_，但是会**_阻塞_**客户端的**_写_**    
   -  表独占**_写锁_**（write lock）： **_阻塞其他客户端的读和写  _**
```sql
-- 加锁：
lock tables 表名... read/write。

--释放锁：
unlock tables / 客户端断开连接 。
```

- 元数据锁（meta data lock，MDL）  
> MDL加锁过程是**系统自动控制，无需显式使用**，在访问一张表的时候会自动加上。MDL锁主要作用是维护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。**_为了避免DML与 DDL冲突，保证读写的正确性_**。  



- 意向锁
>  为了避免DML在执行时，加的行锁与表锁的冲突，在InnoDB中引入了意向锁，使得表锁不用检查每行 数据是否加锁，使用意向锁来减少表锁的检查 。

分类：

-  意向共享锁(IS):  由语句select ... lock in share mode添加 。与表锁共享锁 (read)兼容，与表锁排他锁(write)互斥。 

- 意向排他锁(IX):  由insert、update、delete、select...for update添加 。与表锁共 享锁(read)及排他锁(write)都互斥，意向锁之间不会互斥。  

**_ 一旦事务提交了，意向共享锁、意向排他锁，都会自动释放。  _**

```sql
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
performance_schema.data_locks;
```

### 3、行级锁
定义： 行级锁，**每次操作锁住对应的行数据**。锁定粒度最小，发生锁冲突的概率最低，并发度最高。**_应用在 InnoDB存储引擎中_**。  行锁是**通过对索引上的索引项加锁来实现的，而不是对记录加的锁**  

分类：

-  行锁（Record Lock）：锁定单个行记录的锁，防止其他事务对此行进行update和delete。在 RC、RR隔离级别下都支持。  

-  间隙锁（Gap Lock）：锁定索引记录间隙（不含该记录），确保索引记录间隙不变，防止其他事 务在这个间隙进行insert，产生幻读。在RR隔离级别下都支持。  

-  临键锁（Next-Key Lock）：行锁和间隙锁组合，同时锁住数据，并锁住数据前面的间隙Gap。 在RR隔离级别下支持。  

#### （1）、行锁

 InnoDB实现了以下两种类型的行锁：

-  共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁。 
- 排他锁（X）：允许获取排他锁的事务更新数据，阻止其他事务获得相同数据集的共享锁和排他锁。 

![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689755793880-f32515b3-1020-41ee-9482-b7ef8f58aa26.png#averageHue=%23e0b9b8&clientId=u64c4d1f9-41ab-4&from=paste&height=264&id=uc20010b6&originHeight=396&originWidth=1370&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=68049&status=done&style=none&taskId=u73e0992f-0441-4e9b-9407-e0fc894baba&title=&width=913.3333333333334)
 常见的SQL语句，在执行时，所加的行锁如下：  

|  SQL  | 行锁类型  | 说明   |
| --- | --- | --- |
|  INSERT ...   |  排他锁   |  自动加锁   |
|  UPDATE ...   |  排他锁   |  自动加锁   |
|  DELETE ...   |  排他锁   |  自动加锁   |
|  SELECT（正常）   |  不加任何锁   |  |
|  SELECT ... LOCK IN SHARE MODE   |  共享锁   |  需要手动在SELECT之后加LOCK IN SHARE MODE   |
|  SELECT ... FOR UPDATE   | 排他锁   |  需要手动在SELECT之后加FOR UPDATE   |


默认情况下，InnoDB在 **_REPEATABLE READ_**事务隔离级别运行，InnoDB使用 next-key 锁进行搜索和索引扫描，以防止幻读。 

- 针对**唯一索引**进行检索时，对已存在的记录进行等值匹配时，将会自动优化为行锁。
- InnoDB的**行锁是针对于索引加的锁**，**_不通过索引条件检索数据，那么InnoDB将对表中的所有记录加锁，此时就会升级为表锁_**。  


#### （2）、间隙锁、临键锁
 默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next-key 锁进行搜 索和索引扫描，以防止幻读。

-  索引上的等值查询(唯一索引)，给不存在的记录加锁时, 优化为间隙锁 。

-  索引上的等值查询(非唯一普通索引)，向右遍历时最后一个值不满足查询需求时，next-key lock 退化为间隙锁。 

- 索引上的范围查询(唯一索引)--会访问到不满足条件的第一个值为止。  

>  注意：间隙锁唯一目的是防止其他事务插入间隙。间隙锁可以共存，一个事务采用的间隙锁不会 阻止另一个事务在同一间隙上采用间隙锁。  


## 九、InnoDB引擎

 InnoDB的逻辑存储结构如下图所示:  
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689838490008-248f8bb5-77a6-4f8c-84f6-e394cfe780dd.png#averageHue=%2394cc5b&clientId=u03564a61-4e0a-4&from=paste&height=458&id=u28da12af&originHeight=687&originWidth=1372&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=168535&status=done&style=none&taskId=u1c18ae04-c1f4-4fd9-8c4f-7a2a8321f19&title=&width=914.6666666666666)

### 1、内存结构

![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689838801650-22a0e896-c089-4a0e-b458-3a831196c53b.png#averageHue=%23f2f1f0&clientId=u03564a61-4e0a-4&from=paste&height=466&id=u99cf32b6&originHeight=699&originWidth=347&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=84867&status=done&style=none&taskId=u29be32c3-2856-4aba-815f-6ae6a1868a5&title=&width=231.33333333333334)

 在左侧的内存结构中，主要分为： Buffer Pool、Change Buffer、Adaptive Hash Index、Log Buffer。  分别介绍如下：

- Buffer Pool：缓冲池，  是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增删改查操作时，_**先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频率刷新到磁盘，从而减少磁盘IO，加快处理速度**_。    

- Change Buffer： 更改缓冲区（针对于非唯一二级索引页），在执行DML语句时，如果这些数据Page 没有在Buffer Pool中，**_不会直接操作磁盘_**，而会将数据变更存在更改缓冲区 Change Buffer 中，在未来数据被读取时，再将数据合并恢复到Buffer Pool中，再将合并后的数据刷新到磁盘中。  

- Adaptive Hash Index： 自适应hash索引，用于优化对Buffer Pool数据的查询  

- Log Buffer： 日志缓冲区，用来保存要写入到磁盘中的log日志数据  
### 2、后台线程
 在InnoDB的后台线程中，分为4类，分别是：  

- Master Thread ： 核心后台线程，负责调度其他线程，还负责**将缓冲池中的数据异步刷新到磁盘中, 保持数据的一致性， 还包括脏页的刷新、合并插入缓存、undo页的回收** 。  

- IO Thread ： 在InnoDB存储引擎中大量使用了AIO来处理IO请求, 这样可以极大地提高数据库的性能，而IO Thread主要负责这些IO请求的回调。  

![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689839980940-30b6b98d-50ab-454e-ada1-3bc6dc824b5d.png#averageHue=%23fbfbfb&clientId=u03564a61-4e0a-4&from=paste&height=243&id=ub9ff3637&originHeight=365&originWidth=1038&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=58715&status=done&style=none&taskId=udc50b49e-f947-4d04-973e-fbd9fbe50f9&title=&width=692)

- Purge Thread ： 主要用于回收事务已经提交了的_**undo log**_，在事务提交之后，undo log可能不用了，就用它来回收。  

- Page Cleaner Thread ： 协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞。

### 3、事务原理 
> 定义 ： 是一组**操作的集合**，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系 统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败。  
> 特性： 
> - 原子性（Atomicity）：事务是不可分割的最小操作单元，要么全部成功，要么全部失败。  
> - 一致性（Consistency）：事务完成时，必须使所有的数据都保持一致状态。 
> - 隔离性（Isolation）：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环 境下运行。 
> -  持久性（Durability）：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的。  
> 
**_如何保证这四大特性的？_**
>  搭：1、_原子性、一致性、持久化 _这三个特性是由InnoDB中的两份日志来保证的，**_一份是redo log日志，一份是undo log日志  _**
>  2、_持久性_是通过**_数据库的锁， 加上MVCC来保证的_**  


#### 1）、redo log
> 定义：** 重做日志**，记录的是事务提交时数据页的物理修改，是用来实现事务的持久性。  
>  该日志文件由两部分组成：_重做日志缓冲_（redo log buffer）以及 _重做日志文件_（redo log file）,前者是在内存中，后者在磁盘中。**当事务提交之后会把所有修改信息都存到该日志文件中, 用 于在刷新脏页到磁盘,发生错误时, 进行数据恢复使用。**  


流程：
 有了redolog之后，当对缓冲区的数据进行增删改之后，会**_首先将操作的数据页的变化，记录在redo log buffer中_**。在**_事务提交时_**，会将redo log buffer中的数据刷新到redo log磁盘文件中。 过一段时间之后，如果刷新缓冲区的脏页到磁盘时，发生错误，此时就可以借助于redo log进行数据 恢复，这样就保证了事务的**持久性**。 而如果脏页成功刷新到磁盘 或 或者涉及到的数据已经落盘，此时redolog就没有作用了，就可以删除了，所以存在的两个redolog文件是循环写的。  

#### 2）、undo log

定义： **回滚日志**，用于**记录数据被修改前的信息** , 作用包含两个 : 提供回滚(保证事务的原子性) 和 MVCC(多版本并发控制) 。  （记录的是逻辑日志）

存储内容： 可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的 update记录。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。  （简而言之：_记录的数据是为了还原数据库_）

存储和销毁：

-  存储：undo log采用段的方式进行管理和记录，存放在前面介绍的 rollback segment 回滚段中，内部包含1024个undo log segment。  
- 销毁： undo log在事务执行时产生，事务提交时，并不会立即删除undo log，因为这些 日志可能还用于MVCC。  

### 4、MVCC（多版本并发控制）
先介绍下面两个定义，便于理解

> 1、**当前读**：读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。对于我们日常的操作，如：select ... lock in share mode(共享锁)，select ... for update、update、insert、delete(排他锁)都是一种当前读。  
> 2、**快照读**： 简单的select（不加锁）就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据， 不加锁，是非阻塞读。
> -  Read Committed：每次select，都生成一个快照读。 
> - Repeatable Read：开启事务后第一个select语句才是快照读的地方。 
> - Serializable：快照读会退化为当前读  


定义： 全称 Multi-Version Concurrency Control，多版本并发控制。**指维护一个数据的多个版本， 使得读写操作没有冲突**，快照读为MySQL实现MVCC提供了一个非阻塞读功能。MVCC的具体实现，还需要依赖于数据库记录中的**三个隐式字段、undo log日志、readView**。  

#### 1）、隐藏字段
在我们创建一个新的表的时候， InnoDB还会自动的给我们添加三个隐藏字段及其含义分别是：  
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689843347996-69edef4c-cbfc-4053-888d-d98794cfe849.png#averageHue=%23e0cab3&clientId=u03564a61-4e0a-4&from=paste&height=227&id=u1f73a063&originHeight=340&originWidth=1033&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=74357&status=done&style=none&taskId=u711b7c7e-f204-4b77-ae33-ba9c9780fda&title=&width=688.6666666666666)

#### 2）、undo log
版本链：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689843483511-0996dec3-c327-4387-9af0-fac96dfeb08a.png#averageHue=%23d1f0c9&clientId=u03564a61-4e0a-4&from=paste&height=285&id=u4ed8b007&originHeight=428&originWidth=1063&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=84020&status=done&style=none&taskId=u00e7f705-b3d9-4bfb-ab31-5e99ff4711a&title=&width=708.6666666666666)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689843509271-90e1326b-ad20-4341-ad1b-1f6fb742db33.png#averageHue=%23faf8f6&clientId=u03564a61-4e0a-4&from=paste&height=630&id=uce15beca&originHeight=945&originWidth=1043&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=242963&status=done&style=none&taskId=ufb893b05-d5fb-4a14-a33e-38f5817f4ce&title=&width=695.3333333333334)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689843524404-ea1f1bf5-9873-40d5-af08-47aaa597f0c3.png#averageHue=%23fbf8f7&clientId=u03564a61-4e0a-4&from=paste&height=400&id=ud32f6920&originHeight=600&originWidth=1046&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=191196&status=done&style=none&taskId=u4956bb37-42ba-4d89-9b00-002b6e969b7&title=&width=697.3333333333334)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689843541572-7d0b418e-a3a1-4e7b-9e65-1f6c012eb8d8.png#averageHue=%23f9eadd&clientId=u03564a61-4e0a-4&from=paste&height=229&id=u311cf50f&originHeight=344&originWidth=1049&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=62534&status=done&style=none&taskId=ue7d65eda-ac71-41d0-9253-83b46827a38&title=&width=699.3333333333334)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689843561727-edc395fa-cd74-4d91-b0c8-5ff9011bda4e.png#averageHue=%23faf4f2&clientId=u03564a61-4e0a-4&from=paste&height=622&id=u772fdf97&originHeight=933&originWidth=1063&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=252752&status=done&style=none&taskId=uea86a94c-641d-4114-9d1c-e2f4eff060f&title=&width=708.6666666666666)

> 结论： **_最终我们发现，不同事务或相同事务对同一条记录进行修改，会导致该记录的undolog生成一条记录版本链表，链表的头部是最新的旧记录，链表尾部是最早的旧记录_**。  


#### 3）、Redview
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689843684623-f7977e73-5495-44d5-9dc3-7d3b24f674c5.png#averageHue=%23fdfcfc&clientId=u03564a61-4e0a-4&from=paste&height=386&id=u2d167f11&originHeight=579&originWidth=1089&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=102989&status=done&style=none&taskId=u7d66af26-ba07-4db3-a293-49adbfeee99&title=&width=726)

 **不同的隔离级别，生成ReadView的时机不同**： 

- READ COMMITTED ：在事务中每一次执行快照读时生成ReadView。 	
- REPEATABLE READ：仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView。  


**MVCC的实现原理就是通过 InnoDB表的隐藏字段、UndoLog 版本链、ReadView来实现的。 而MVCC + 锁，则实现了事务的隔离性。 而一致性则是由redolog 与 undolog保证**。  


## 十、日志

> MySQL中日志分为以下四个类型的日志

- 错误日志： 错误日志是 MySQL 中最重要的日志之一，它记录了当 mysqld 启动和停止时，以及服务器在运行过 程中发生任何严重错误时的相关信息。当数据库出现任何故障导致无法正常使用时，建议首先查看此日志。

- 二进制日志： 二进制日志（BINLOG）记录了所有的 DDL（数据定义语言）语句和 DML（数据操纵语言）语句，但 不包括数据查询（SELECT、SHOW）语句。 作用：①. 灾难时的数据恢复；②. MySQL的主从复制。    

- 查询日志： 查询日志中记录了客户端的所有操作语句，而二进制日志不包含查询数据的SQL语句。默认情况下， 查询日志是未开启的  

- 慢查询日志： 慢查询日志记录了所有执行时间超过参数 long_query_time 设置值并且扫描记录数不小于 min_examined_row_limit 的所有的SQL语句的日志，默认未开启。long_query_time 默认为 10 秒，最小为 0， 精度可以到微秒。  

## 十一、主从复制

> 概念： 是指将**主数据库的 DDL 和 DML 操作通过二进制日志传到从库服务器中**，然后在从库上对这 些日志重新执行（也叫重做），从而使得从库和主库的数据保持同步。 MySQL支持一台主库同时向多台从库进行复制， 从库同时也可以作为其他从服务器的主库，实现链状 复制。  
> 
> 优点：
> -  主库出现问题，可以快速切换到从库提供服务。 
> - 实现读写分离，降低主库的访问压力。 
> - 可以在从库中执行备份，以避免备份期间影响主库服务。  


![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689920927019-9f9185aa-a00c-4461-b3ef-6fa7540f3bd0.png#averageHue=%23fcfbfa&clientId=u79a519ae-9d3e-4&from=paste&height=515&id=u3b64a4af&originHeight=773&originWidth=1106&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=171536&status=done&style=none&taskId=ua73eb058-7149-4ed6-886b-1b92019480d&title=&width=737.3333333333334)
### 1、主库配置

- 1、修改配置文件： /etc/my.cnf  
```yaml
#mysql 服务ID，保证整个集群环境中唯一，取值范围：1 – 232-1，默认为1
server-id=1

#是否只读,1 代表只读, 0 代表读写
read-only=0

#忽略的数据, 指不需要同步的数据库
#binlog-ignore-db=mysql
#指定同步的数据库
#binlog-do-db=db01
```

- 2、重启MySQL数据库： `systemctl restart mysqld`  

- 3、登录mysql，创建远程连接的账号，并授予主从复制权限  
```sql
-- 创建itcast用户，并设置密码，该用户可在任意主机连接该MySQL服务
CREATE USER 'itcast'@'%' IDENTIFIED WITH mysql_native_password BY 'Root@123456';

-- 为 'itcast'@'%' 用户分配主从复制权限
GRANT REPLICATION SLAVE ON *.* TO 'itcast'@'%';
```

- 4、 通过指令，查看二进制日志坐标：   `show master status  `

字段含义说明： 

   - file : 从哪个日志文件开始推送日志文件 
   - position ： 从哪个位置开始推送日志 
   - binlog_ignore_db : 指定不需要同步的数据库  

### 2、从库配置

- 1、修改配置文件：` /etc/my.cnf  `
```sql
-- mysql 服务ID，保证整个集群环境中唯一，取值范围：1 – 2^32-1，和主库不一样即可
server-id=2

-- 是否只读,1 代表只读, 0 代表读写
read-only=1
```

- 2、重启MySQL服务： `systemctl restart mysqld `

- 3、登录MySQL，设置主库配置
```sql
CHANGE REPLICATION SOURCE TO SOURCE_HOST='192.168.200.200', 
SOURCE_USER='itcast',
SOURCE_PASSWORD='Root@123456', SOURCE_LOG_FILE='binlog.000004',
SOURCE_LOG_POS=663;
```
 上述是8.0.23中的语法。如果mysql是 8.0.23 之前的版本，执行如下SQL：  
```sql
CHANGE MASTER TO MASTER_HOST='192.168.200.200', 
MASTER_USER='itcast',
MASTER_PASSWORD='Root@123456', MASTER_LOG_FILE='binlog.000004',
MASTER_LOG_POS=663;
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1689922764112-6452a09d-7f78-4190-a6f5-211c165f0a8c.png#averageHue=%23fcfbfb&clientId=u79a519ae-9d3e-4&from=paste&height=290&id=u2d13fe21&originHeight=435&originWidth=1039&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=70654&status=done&style=none&taskId=u1c487445-59a3-4080-85d0-59a394cb87f&title=&width=692.6666666666666)

- 4、开启同步操作：   `start replica ; #8.0.22之后 `

`start slave ; #8.0.22之前  `

- 5、查看主从同步状态： `show replica status ; #8.0.22之后 `

    ` show slave status ; #8.0.22之前  `

> 主要查看两个字段是否为YES：Replica_IO_Running 、Replica_SQL_Running ，两个字段都为YES的话，则证明主从复制已经开启。


## 十二、分库分表

>  分库分表的中心思想都是**将数据分散存储，使得单一数据库/表的数据量变小来缓解单一数据库的性能 问题，从而达到提升数据库性能的目的**。  


### 1、拆分策略
 分库分表的形式，主要是两种：垂直拆分和水平拆分。而拆分的粒度，一般又分为分库和分表，所以组 成的拆分策略最终如下：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690167282572-941cbba1-fc41-4c53-8328-151fbd94335b.png#averageHue=%23fdfdfd&clientId=u82749110-536b-4&from=paste&height=289&id=udcfbbeb7&originHeight=433&originWidth=1027&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=61317&status=done&style=none&taskId=u17c7f7de-f834-4bb7-b778-5ac81c9aaa3&title=&width=684.6666666666666)

![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690167676917-bb288a98-8755-4a71-9db7-77e122dc7034.png#averageHue=%23f6f5f5&clientId=u82749110-536b-4&from=paste&height=481&id=u6a60d55f&originHeight=721&originWidth=1608&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=394274&status=done&style=none&taskId=ufb90583d-1e62-4722-87ca-1d01bddc94d&title=&width=1072)

![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690167720699-049538df-24bf-4efc-9bbc-a7cc3f0ec9e7.png#averageHue=%23f7f5f4&clientId=u82749110-536b-4&from=paste&height=507&id=u7767c84d&originHeight=760&originWidth=1603&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=376513&status=done&style=none&taskId=uc4a6546f-e869-4f78-adcd-82c6a52bbf1&title=&width=1068.6666666666667)
实现技术： 

- shardingJDBC：基于AOP原理，在应用程序中对本地执行的SQL进行拦截，解析、改写、路由处 理。需要自行编码配置实现，只支持java语言，性能较高。

- MyCat：数据库分库分表中间件，不用调整代码即可实现分库分表，支持多种语言，性能不及前者


### 2、MyCat
> 介绍： Mycat是开源的、活跃的、基于Java语言编写的MySQL数据库中间件。可以像使用mysql一样来使用 mycat，对于开发人员来说根本感觉不到mycat的存在。 开发人员只需要连接MyCat即可，而具体底层用到几台数据库，每一台数据库服务器里面存储了什么数 据，都无需关心。 具体的分库分表的策略，只需要在MyCat中配置即可。    
> 启动：`mycat start `，停止：`mycat stop`

MyCat的整体结构中，分为两个部分：上面的逻辑结构，下面的物理结构，示意如下：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690168956932-0ef66c1d-a213-403b-9e9d-089dcca1ce51.png#averageHue=%23ecebd7&clientId=u82749110-536b-4&from=paste&height=396&id=u8649c789&originHeight=594&originWidth=1273&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=125828&status=done&style=none&taskId=u4802a170-2691-4e50-a0f6-ccffb124c22&title=&width=848.6666666666666)

#### 1）、配置
#####  schema.xml  配置文件
>  在schema.xml中配置逻辑库、逻辑表、数据节点、节点主机等相关信息。具体的配置如下：  

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
<!-- 	核心配置：schema 标签用于定义 MyCat实例中的逻辑库 , 一个MyCat实例中, 可以有多个逻辑库 , 可以通
过 schema 标签来划分不同的逻辑库。MyCat中的逻辑库的概念，等同于MySQL中的database概念
, 需要操作某个逻辑库下的表时, 也需要切换逻辑库(use xxx)。 -->
<!-- 	schema 的核心属性：
	1、name：指定自定义的逻辑库库名
	2、checkSQLschema：在SQL语句操作时指定了数据库名称，执行时是否自动去除；true：自动去
除，false：不自动去除
	3、sqlMaxLimit：如果未指定limit进行查询，列表查询模式查询多少条记录
-->
	<schema name="DB01" checkSQLschema="true" sqlMaxLimit="100">
<!-- table 标签定义了MyCat中逻辑库schema下的逻辑表 , 所有需要拆分的表都需要在table标签中定义 
		核心属性：
		1、name：定义逻辑表表名，在该逻辑库下唯一
		2、dataNode：定义逻辑表所属的dataNode，该属性需要与dataNode标签中name对应；多个dataNode逗号分隔
		3、rule：分片规则的名字，分片规则名字是在rule.xml中定义的
		4、primaryKey：逻辑表对应真实表的主键
		5、type：逻辑表的类型，目前逻辑表只有全局表和普通表，如果未配置，就是普通表；全局表，配置为 global
-->
		<table name="TB_ORDER" dataNode="dn1,dn2,dn3" rule="auto-sharding-long"/>
	</schema>

<!-- 	dataNode核心属性：
      1、name：定义数据节点名称
			2、dataHost：数据库实例主机名称，引用自 dataHost 标签中name属性
			3、database：定义分片所属数据库 -->
	<dataNode name="dn1" dataHost="dhost1" database="db01" />
	<dataNode name="dn2" dataHost="dhost2" database="db01" />
	<dataNode name="dn3" dataHost="dhost3" database="db01" />


<!-- 	该标签在MyCat逻辑库中作为底层标签存在, 直接定义了具体的数据库实例、读写分离、心跳语句。
	核心属性：
		1、name：唯一标识，供上层标签使用
		2、maxCon/minCon：最大连接数/最小连接数
		3、balance：负载均衡策略，取值 0,1,2,3
		4、writeType：写操作分发方式（0：写操作转发到第一个writeHost，第一个挂了，切换到第二个；1：写操作随机分发到配置的writeHost）
		5、dbDriver：数据库驱动，支持 native、jdbc
-->
	<dataHost name="dhost1" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"
		slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="master" url="jdbc:mysql://192.168.200.210:3306?
			useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
			user="root" password="1234" />
	</dataHost>
	<dataHost name="dhost2" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"
		slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="master" url="jdbc:mysql://192.168.200.213:3306?
			useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
			user="root" password="1234" />
	</dataHost>
</mycat:schema>
```

##### rule.xml 配置文件
>  rule.xml中**定义所有拆分表的规则**, 在使用过程中可以灵活的使用分片算法, 或者对同一个分片算法 使用不同的参数, 它让分片过程可配置化。主要包含两类标签：tableRule、Function  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
	<tableRule name="rule1">
		<rule>
			<columns>id</columns>
			<algorithm>func1</algorithm>
		</rule>
	</tableRule>

	<tableRule name="rule2">
		<rule>
			<columns>user_id</columns>
			<algorithm>func1</algorithm>
		</rule>
	</tableRule>

	<tableRule name="sharding-by-intfile">
		<rule>
			<columns>sharding_id</columns>
			<algorithm>hash-int</algorithm>
		</rule>
	</tableRule>
	<tableRule name="auto-sharding-long">
		<rule>
	<!-- 	表明根据ID进行分片  -->
			<columns>id</columns>
			<algorithm>rang-long</algorithm>
		</rule>
	</tableRule>
	<tableRule name="mod-long">
		<rule>
			<columns>id</columns>
			<algorithm>mod-long</algorithm>
		</rule>
	</tableRule>
	<tableRule name="sharding-by-murmur">
		<rule>
			<columns>id</columns>
			<algorithm>murmur</algorithm>
		</rule>
	</tableRule>
	<tableRule name="crc32slot">
		<rule>
			<columns>id</columns>
			<algorithm>crc32slot</algorithm>
		</rule>
	</tableRule>
	<tableRule name="sharding-by-month">
		<rule>
			<columns>create_time</columns>
			<algorithm>partbymonth</algorithm>
		</rule>
	</tableRule>
	<tableRule name="latest-month-calldate">
		<rule>
			<columns>calldate</columns>
			<algorithm>latestMonth</algorithm>
		</rule>
	</tableRule>

	<tableRule name="auto-sharding-rang-mod">
		<rule>
			<columns>id</columns>
			<algorithm>rang-mod</algorithm>
		</rule>
	</tableRule>

	<tableRule name="jch">
		<rule>
			<columns>id</columns>
			<algorithm>jump-consistent-hash</algorithm>
		</rule>
	</tableRule>

	<function name="murmur"class="io.mycat.route.function.PartitionByMurmurHash">
		<!-- 默认是0 -->
		<property name="seed">0</property>
		<!-- 要分片的数据库节点数量，必须指定，否则没法分片 -->
		<property name="count">2</property>
		
		<property name="virtualBucketTimes">160</property>
<!-- 一个实际的数据库节点被映射为这么多虚拟节点，默认是160倍，也就是虚拟节点数是物理节点数的160倍 
		 <property name="weightMapFile">weightMapFile</property> 节点的权重，没有指定权重的节点默认是1。以properties文件的格式填写，以从0开始到count-1的整数值也就是节点索引为key，以节点权重值为值。所有权重值必须是正整数，否则以1代替 
		 <property name="bucketMapPath">/etc/mycat/bucketMapPath</property> 
		用于测试时观察各物理节点与虚拟节点的分布情况，如果指定了这个属性，会把虚拟节点的murmur hash值与物理节点的映射按行输出到这个文件，没有默认值，如果不指定，就不会输出任何东西 
-->
	</function>

	<function name="crc32slot"
		class="io.mycat.route.function.PartitionByCRC32PreSlot">
	</function>
	<function name="hash-int"
		class="io.mycat.route.function.PartitionByFileMap">
		<property name="mapFile">partition-hash-int.txt</property>
	</function>
	<function name="rang-long"
		class="io.mycat.route.function.AutoPartitionByLong">
		<property name="mapFile">autopartition-long.txt</property>
	</function>
	<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
		<!-- how many data nodes -->
		<property name="count">3</property>
	</function>

	<function name="func1" class="io.mycat.route.function.PartitionByLong">
		<property name="partitionCount">8</property>
		<property name="partitionLength">128</property>
	</function>
	<function name="latestMonth"
		class="io.mycat.route.function.LatestMonthPartion">
		<property name="splitOneDay">24</property>
	</function>
	<function name="partbymonth"
		class="io.mycat.route.function.PartitionByMonth">
		<property name="dateFormat">yyyy-MM-dd</property>
		<property name="sBeginDate">2015-01-01</property>
	</function>

	<function name="rang-mod" class="io.mycat.route.function.PartitionByRangeMod">
		<property name="mapFile">partition-range-mod.txt</property>
	</function>

	<function name="jump-consistent-hash" class="io.mycat.route.function.PartitionByJumpConsistentHash">
		<property name="totalBuckets">3</property>
	</function>
</mycat:rule>
```
##### server.xml 配置文件
>  server.xml配置文件包含了MyCat的系统配置信息，主要有两个重要的标签：system、user。  

1、system 标签

详解请看链接：
[MyCat配置文件详解--server.xml_Fighter168的博客-CSDN博客](https://blog.csdn.net/fighterandknight/article/details/77885792)

2、user 标签
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690185208996-89d2118b-0a4f-4785-9f3f-0e81c80b37f2.png#averageHue=%23fdfafa&clientId=u82749110-536b-4&from=paste&height=468&id=u2d152fad&originHeight=702&originWidth=1375&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=273016&status=done&style=none&taskId=u62d32d6c-0c83-4b03-9515-28c11f76608&title=&width=916.6666666666666)

#### 2）、分片

##### 垂直分库及测试
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690189994801-9b24ab89-afb7-4dd9-8ee2-3f5161926d6f.png#averageHue=%23fdedec&clientId=u82749110-536b-4&from=paste&height=491&id=uc85bd74b&originHeight=737&originWidth=988&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=216240&status=done&style=none&taskId=u4533ef6c-00e1-4cfa-bf3d-e4ccead9ead&title=&width=658.6666666666666)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690190007301-646049cf-11ce-4bbc-af4a-f521e0d19a15.png#averageHue=%23fcfcfb&clientId=u82749110-536b-4&from=paste&height=491&id=ub9469dc4&originHeight=737&originWidth=769&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=138526&status=done&style=none&taskId=udc3da049-3b3c-4a0e-83a7-488453af5c9&title=&width=512.6666666666666)
```xml
<schema name="SHOPPING" checkSQLschema="true" sqlMaxLimit="100">
	<table name="tb_goods_base" dataNode="dn1" primaryKey="id" />
	<table name="tb_goods_brand" dataNode="dn1" primaryKey="id" />
	<table name="tb_goods_cat" dataNode="dn1" primaryKey="id" />
	<table name="tb_goods_desc" dataNode="dn1" primaryKey="goods_id" />
	
	<table name="tb_goods_item" dataNode="dn1" primaryKey="id" />
	<table name="tb_order_item" dataNode="dn2" primaryKey="id" />
	<table name="tb_order_master" dataNode="dn2" primaryKey="order_id" />
	<table name="tb_order_pay_log" dataNode="dn2" primaryKey="out_trade_no" />

	
	<table name="tb_user" dataNode="dn3" primaryKey="id" />
	<table name="tb_user_address" dataNode="dn3" primaryKey="id" />
	<table name="tb_areas_provinces" dataNode="dn3" primaryKey="id"/>
	<table name="tb_areas_city" dataNode="dn3" primaryKey="id"/>
	<table name="tb_areas_region" dataNode="dn3" primaryKey="id"/>
</schema>


<dataNode name="dn1" dataHost="dhost1" database="shopping" />
<dataNode name="dn2" dataHost="dhost2" database="shopping" />
<dataNode name="dn3" dataHost="dhost3" database="shopping" />
<dataHost name="dhost1" maxCon="1000" minCon="10" balance="0"
	writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"
	slaveThreshold="100">
	<heartbeat>select user()</heartbeat>
	<writeHost host="master" url="jdbc:mysql://192.168.200.210:3306?
		useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
		user="root" password="1234" />
</dataHost>
<dataHost name="dhost2" maxCon="1000" minCon="10" balance="0"
	writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"
	slaveThreshold="100">
	<heartbeat>select user()</heartbeat>
	<writeHost host="master" url="jdbc:mysql://192.168.200.213:3306?
		useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
		user="root" password="1234" />
</dataHost>
<dataHost name="dhost3" maxCon="1000" minCon="10" balance="0"
	writeType="0" dbType="mysql" dbDriver="jdbc" switchType="1"
	slaveThreshold="100">
	<heartbeat>select user()</heartbeat>
	<writeHost host="master" url="jdbc:mysql://192.168.200.214:3306?
		useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
		user="root" password="1234" />
</dataHost>
```

> 可能会遇到的情况：执行某条SQL语句的时候，由于不在同一个服务器，执行SQL语句会报错，而解决的办法是设置一个全局表可以解决这个问题。
> **解决：将多个业务中都会使用到的表，在每一个服务的数据库中都添加上该表。具体操作如下： 修改schema.xml中的逻辑表的配置，修改 tb_areas_provinces、tb_areas_city、 tb_areas_region 三个逻辑表，增加 type 属性，配置为global，就代表该表是全局表，就会在 所涉及到的dataNode中创建给表。对于当前配置来说，也就意味着所有的节点中都有该表了。  **

```xml
<table name="tb_areas_provinces" dataNode="dn1,dn2,dn3" primaryKey="id"
	type="global"/>
<table name="tb_areas_city" dataNode="dn1,dn2,dn3" primaryKey="id"
	type="global"/>
<table name="tb_areas_region" dataNode="dn1,dn2,dn3" primaryKey="id"
	type="global"/>

```
##### 水平分表
```xml
<schema name="ITCAST" checkSQLschema="true" sqlMaxLimit="100">
	<table name="tb_log" dataNode="dn4,dn5,dn6" primaryKey="id" rule="mod-long" />
</schema>
<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />

```
 tb_log表最终落在3个节点中，分别是 dn4、dn5、dn6 ，而具体的数据分别存储在 dhost1、 dhost2、dhost3的itcast数据库中。  

##### 分片规则
###### 1、范围分片
定义：根据指定的字段及其配置的范围与数据节点的对应情况， 来决定该数据属于哪一个分片，对应的配置如下
```xml
<table name="TB_ORDER" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />

<dataNode name="dn1" dataHost="dhost1" database="db01" />
<dataNode name="dn2" dataHost="dhost2" database="db01" />
<dataNode name="dn3" dataHost="dhost3" database="db01" />
```
```xml
<tableRule name="auto-sharding-long">
	<rule>
		<columns>id</columns>
		<algorithm>rang-long</algorithm>
	</rule>
</tableRule>
<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
	<property name="mapFile">autopartition-long.txt</property>
	<property name="defaultNode">0</property>
</function>
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690253052487-924ec4ce-8016-47ad-8420-b723cfbd6482.png#averageHue=%23e1cbb1&clientId=u437768e5-8bfe-4&from=paste&height=579&id=ub11fb34f&originHeight=869&originWidth=817&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=132472&status=done&style=none&taskId=u5e1460fc-55c7-4454-96c7-5d74ce6d5de&title=&width=544.6666666666666)
###### 2、取模分片
定义： 根据指定的字段值与节点数量进行求模运算，根据运算结果， 来决定该数据属于哪一个分片 
```xml
<table name="tb_log" dataNode="dn4,dn5,dn6" primaryKey="id" rule="mod-long" />

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```
```xml
<tableRule name="mod-long">
	<rule>
		<columns>id</columns>
		<algorithm>mod-long</algorithm>
	</rule>
</tableRule>


<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
	<property name="count">3</property>
</function>
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690253338118-09085da6-5a61-49fe-8db0-3c0fae3a1116.png#averageHue=%23fdfcfb&clientId=u437768e5-8bfe-4&from=paste&height=368&id=uf6f0a78b&originHeight=552&originWidth=946&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=79738&status=done&style=none&taskId=u34b0dead-d270-4d8e-aa36-891422502ba&title=&width=630.6666666666666)
###### 3、一致性hash分片
定义： 所谓一致性哈希，相同的哈希因子计算值总是被划分到相同的分区表中，不会因为分区节点的增加而改 变原来数据的分区位置，有效的解决了分布式数据的拓容问题。  
```xml
<!-- 一致性hash -->
<table name="tb_order" dataNode="dn4,dn5,dn6" rule="sharding-by-murmur" />

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```
```xml
<tableRule name="sharding-by-murmur">
	<rule>
		<columns>id</columns>
		<algorithm>murmur</algorithm>
	</rule>
</tableRule>
<function name="murmur" class="io.mycat.route.function.PartitionByMurmurHash">
	<property name="seed">0</property><!-- 默认是0 -->
	<property name="count">3</property>
	<property name="virtualBucketTimes">160</property>
</function>
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690254953656-529e39c3-a2ed-4246-b367-f938277cd718.png#averageHue=%23fbfafa&clientId=u437768e5-8bfe-4&from=paste&height=606&id=u2495c4c6&originHeight=909&originWidth=928&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=226251&status=done&style=none&taskId=u1e7c7bad-a4d1-473d-89bf-382c938e23b&title=&width=618.6666666666666)

###### 4、枚举分片
定义： 通过在配置文件中配置可能的枚举值, 指定数据分布到不同数据节点上, 本规则适用于按照省份、性 别、状态拆分数据等业务 。  
```xml
<!-- 枚举 -->
<table name="tb_user" dataNode="dn4,dn5,dn6" rule="sharding-by-intfile-enumstatus"
/>

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```
```xml
<tableRule name="sharding-by-intfile">
	<rule>
		<columns>sharding_id</columns>
		<algorithm>hash-int</algorithm>
	</rule>
</tableRule>
<!-- 自己增加 tableRule -->
<tableRule name="sharding-by-intfile-enumstatus">
	<rule>
		<columns>status</columns>
		<algorithm>hash-int</algorithm>
	</rule>
</tableRule>
<function name="hash-int" class="io.mycat.route.function.PartitionByFileMap">
	<property name="defaultNode">2</property>
	<property name="mapFile">partition-hash-int.txt</property>
</function>
```
 partition-hash-int.txt ，内容如下 :   1=0，2=1，3=2  ，指定的规则字段为1的话，放入第一张表，规则字段为2的话，放入第二张表；规则字段为3的话，放入第三张表
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690264576799-e0648d0c-d06a-4241-80b9-296b5ad0980f.png#averageHue=%23fbfbfa&clientId=u437768e5-8bfe-4&from=paste&height=401&id=u03094bf8&originHeight=601&originWidth=926&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=115968&status=done&style=none&taskId=u03e8252c-8c23-4580-98ae-4ae48d76f9c&title=&width=617.3333333333334)
###### 5、应用指定算法
定义：运行阶段由应用自主决定路由到那个分片 , 直接根据字符子串（必须是数字）计算分片号。  
```xml
<!-- 应用指定算法 -->
<table name="tb_app" dataNode="dn4,dn5,dn6" rule="sharding-by-substring" />

<dataNode name="dn4" dataHost="dhost1" database="itcast" />
<dataNode name="dn5" dataHost="dhost2" database="itcast" />
<dataNode name="dn6" dataHost="dhost3" database="itcast" />
```
```xml
<tableRule name="sharding-by-substring">
	<rule>
		<columns>id</columns>
		<algorithm>sharding-by-substring</algorithm>
	</rule>
</tableRule>


<function name="sharding-by-substring"
	class="io.mycat.route.function.PartitionDirectBySubString">
	<property name="startIndex">0</property> <!-- zero-based -->
	<property name="size">2</property>
	<property name="partitionCount">3</property>
	<property name="defaultPartition">0</property>
</function>
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690265521074-58c81dc5-ff54-480c-8191-8786f5dd37cd.png#averageHue=%23fcfcfc&clientId=u437768e5-8bfe-4&from=paste&height=418&id=ud33f8658&originHeight=627&originWidth=933&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=88373&status=done&style=none&taskId=ufe3b31a9-83c0-4a60-978f-cac7791e03e&title=&width=622)
 示例说明 : id=05-100000002 , 在此配置中代表根据id中从 startIndex=0，开始，截取siz=2位数字即 05，05就是获取的分区，如果没找到对应的分片则默认分配到defaultPartition 。  

###### 6、固定分片hash算法
定义： 该算法类似于十进制的求模运算，但是为二进制的操作，例如，取 id 的二进制低 10 位 与 1111111111 进行位 & 运算，位与运算最小值为 0000000000，最大值为1111111111，转换为十 进制，也就是位于0-1023之间。  

特点：

-  如果是求模，连续的值，分别分配到各个不同的分片；但是此算法会将连续的值可能分配到相同的 分片，降低事务处理的难度。
-  可以均匀分配，也可以非均匀分配。 
- 分片字段必须为数字类型。  

```xml
<tableRule name="sharding-by-long-hash">
	<rule>
		<columns>id</columns>
		<algorithm>sharding-by-long-hash</algorithm>
	</rule>
</tableRule>
<!-- 分片总长度为1024，count与length数组长度必须一致； -->
<function name="sharding-by-long-hash"
	class="io.mycat.route.function.PartitionByLong">
	<property name="partitionCount">2,1</property>
	<property name="partitionLength">256,512</property>
</function>
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690266152686-08ced75c-5d07-45d0-bfe0-a0f17d2d5b49.png#averageHue=%23fdfdfc&clientId=u437768e5-8bfe-4&from=paste&height=460&id=u122cfc59&originHeight=690&originWidth=944&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=91368&status=done&style=none&taskId=u03359e1b-ac2a-439c-aac0-e2803959667&title=&width=629.3333333333334)

###### 7、字符串hash解析算法
定义：截取字符串中的指定位置的子字符串，进行hash算法，算出分片。

```xml
<tableRule name="sharding-by-stringhash">
	<rule>
		<columns>name</columns>
		<algorithm>sharding-by-stringhash</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-stringhash"
	class="io.mycat.route.function.PartitionByString">
	<property name="partitionLength">512</property> <!-- zero-based -->
	<property name="partitionCount">2</property>
	<property name="hashSlice">0:2</property>
</function>
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690266671990-a19fe62b-9866-4f37-91a1-ea1936f56fba.png#averageHue=%23fcfbfa&clientId=u437768e5-8bfe-4&from=paste&height=498&id=u20aa1279&originHeight=747&originWidth=949&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=125270&status=done&style=none&taskId=u1b856732-80ec-41d8-ba37-4ef527fca4d&title=&width=632.6666666666666)

###### 8、按（天）日期分片
定义： 按照日期及对应的时间周期来分片。  
```xml
<tableRule name="sharding-by-date">
	<rule>
		<columns>create_time</columns>
		<algorithm>sharding-by-date</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-date"
	class="io.mycat.route.function.PartitionByDate">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2022-01-01</property>
	<property name="sEndDate">2022-01-30</property>
	<property name="sPartionDay">10</property>
</function>
<!--
从开始时间开始，每10天为一个分片，到达结束时间之后，会重复开始分片插入
配置表的 dataNode 的分片，必须和分片规则数量一致，例如 2022-01-01 到 2022-12-31 ，每
10天一个分片，一共需要37个分片。
-->
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690272400198-72c6f94d-8a22-42a6-be19-f90d89c11025.png#averageHue=%23fcfcfb&clientId=u437768e5-8bfe-4&from=paste&height=416&id=ue0766530&originHeight=624&originWidth=928&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=97557&status=done&style=none&taskId=u608aad7e-02f7-4274-b089-60a8fb1dd11&title=&width=618.6666666666666)
###### 9、按自然月分片
定义： 使用场景为按照月份来分片, 每个自然月为一个分片。
```xml
<tableRule name="sharding-by-month">
	<rule>
		<columns>create_time</columns>
		<algorithm>partbymonth</algorithm>
	</rule>
</tableRule>


<function name="partbymonth" class="io.mycat.route.function.PartitionByMonth">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2022-01-01</property>
	<property name="sEndDate">2022-03-31</property>
</function>
<!--
从开始时间开始，一个月为一个分片，到达结束时间之后，会重复开始分片插入
配置表的 dataNode 的分片，必须和分片规则数量一致，例如 2022-01-01 到 2022-12-31 ，一
共需要12个分片。
-->
```
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690273815193-79866555-7f90-48f0-a94c-a1b4a8bd486d.png#averageHue=%23fbfbfa&clientId=u437768e5-8bfe-4&from=paste&height=331&id=u1c02135c&originHeight=497&originWidth=928&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=73645&status=done&style=none&taskId=u0e1d720c-052f-4841-8f0f-e45d7275041&title=&width=618.6666666666666)


## 十三、读写分离
 
定义： 读写分离,**简单地说是把对数据库的读和写操作分开,以对应不同的数据库服务器**。主数据库提供写操 作，从数据库提供读操作，这样能有效地减轻单台数据库的压力。 通过MyCat即可轻易实现上述功能，不仅可以支持MySQL，也可以支持Oracle和SQL Server。  
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690276731418-37e5b0ec-658f-4426-b7bc-396883b5a188.png#averageHue=%23faf9f8&clientId=u437768e5-8bfe-4&from=paste&height=273&id=uf4752dea&originHeight=410&originWidth=996&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=74095&status=done&style=none&taskId=ud25498a0-4089-428f-b714-5884414f9bd&title=&width=664)
### 1、一主一从
原理：MySQL的主从复杂，是基于二进制文件binlog 实现的

```xml
<!-- 配置逻辑库 -->
<schema name="ITCAST_RW" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn7">
</schema>


<dataNode name="dn7" dataHost="dhost7" database="itcast" />



<dataHost name="dhost7" maxCon="1000" minCon="10" balance="1" writeType="0"
	dbType="mysql" dbDriver="jdbc" switchType="1" slaveThreshold="100">
	<heartbeat>select user()</heartbeat>
	<writeHost host="master1" url="jdbc:mysql://192.168.200.211:3306?
		useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
		user="root" password="1234" >
		<readHost host="slave1" url="jdbc:mysql://192.168.200.212:3306?
			useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
			user="root" password="1234" />
	</writeHost>
</dataHost>
```
 writeHost代表的是写操作对应的数据库，readHost代表的是读操作对应的数据库。 所以我们要想 **实现读写分离**，就得配置writeHost关联的是主库，readHost关联的是从库。 而仅仅配置好了writeHost以及readHost还不能完成读写分离，还需要配置一个非常重要的负责均衡 的参数 balance，取值有4种，具体含义如下：  

| 参数值 | 含义 |  |
| --- | --- | --- |
| 0 |  不开启读写分离机制 , 所有读操作都发送到当前可用的writeHost上   |  |
| 1 |  全部的readHost 与 备用的writeHost 都参与select 语句的负载均衡（主要针对 于双主双从模式）   |  |
| 2 |  所有的读写操作都随机在writeHost , readHost上分发   |  |
| 3 |  所有的读请求随机分发到writeHost对应的readHost上执行, writeHost不负担读压 力   |  |

> 存在的问题： 当主节点Master宕机之后，业务系统就只能够读，而不能写入数据了。  解决办法：通过双主双从的主从复制结构来实现


### 2、双主双从
定义：
 一个主机 Master1 用于处理所有写请求，它的从机 Slave1 和另一台主机 Master2 还有它的从 机 Slave2 负责所有读请求。当 Master1 主机宕机后，Master2 主机负责写请求，Master1 、 Master2 互为备机。架构图如下:  
![image.png](https://cdn.nlark.com/yuque/0/2023/png/34919960/1690336387711-e2d7e30b-8ad7-4313-b1f3-31bf091aa295.png#averageHue=%23f9f8f6&clientId=u1c83f9d3-db18-4&from=paste&height=295&id=ua3871784&originHeight=442&originWidth=1064&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=91313&status=done&style=none&taskId=u4733d096-3421-4c30-898b-e081721ee3a&title=&width=709.3333333333334)
配置：将两台从库关联到两个主库，同时两个主库也需要进行相互关联。

读写分离： MyCat控制后台数据库的读写分离和负载均衡由schema.xml文件datahost标签的balance属性控 制，通过writeType及switchType来完成失败自动切换的  配置如下：
```xml
<schema name="ITCAST_RW2" checkSQLschema="true" sqlMaxLimit="100" dataNode="dn7">
</schema>

<dataNode name="dn7" dataHost="dhost7" database="db01" />


<dataHost name="dhost7" maxCon="1000" minCon="10" balance="1" writeType="0"
	dbType="mysql" dbDriver="jdbc" switchType="1" slaveThreshold="100">
	<heartbeat>select user()</heartbeat>

	
	<writeHost host="master1" url="jdbc:mysql://192.168.200.211:3306?
		useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
		user="root" password="1234" >
		<readHost host="slave1" url="jdbc:mysql://192.168.200.212:3306?
			useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
			user="root" password="1234" />
	</writeHost>

	
	<writeHost host="master2" url="jdbc:mysql://192.168.200.213:3306?
		useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
		user="root" password="1234" >
		<readHost host="slave2" url="jdbc:mysql://192.168.200.214:3306?
			useSSL=false&amp;serverTimezone=Asia/Shanghai&amp;characterEncoding=utf8"
			user="root" password="1234" />
	</writeHost>
</dataHost>
```

属性说明：

-  balance="1" ：代表全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且 M1 与 M2 互为主备)，正常情况下， M2,S1,S2 都参与 select 语句的负载均衡 ;  
-  writeType  
   - 0： 写操作都转发到第1台writeHost, writeHost1挂了, 会切换到writeHost2上;  
   - 1： 所有的写操作都随机地发送到配置的writeHost上 ;  
-  switchType  
   - -1：不自动切换
   - 1：自动切换

