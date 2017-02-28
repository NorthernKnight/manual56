## 22.3 性能库运行时配置

性能库安装表包含监控配置相关的信息：

```
mysql> SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
       WHERE TABLE_SCHEMA = 'performance_schema'
       AND TABLE_NAME LIKE 'setup%';
+-------------------+
| TABLE_NAME        |
+-------------------+
| setup_actors 	    |
| setup_consumers   |
| setup_instruments |
| setup_objects     |
| setup_timers 	    |
+-------------------+
```

检查这些表的内容可以获得性能库监控特征相关的信息。如果有 UPDATE 权限，就能改变性能库的操作，方式是通过修改这些安装表来影响如何监控。关于安装表的更多细节，请参考 [22.10.02 性能库安装表](./22.10.02 Performance_Schema_Setup_Tables.md#22.10.02)  。

查询 setup_timers 表，可以看到选中的是哪些事件计时器。

```
mysql> SELECT * FROM setup_timers;
+-----------+-------------+
| NAME      | TIMER_NAME  |
+-----------+-------------+
| idle      | MICROSECOND |
| wait      | CYCLE       |
| stage     | NANOSECOND  |
| statement | NANOSECOND  |
+-----------+-------------+
```

NAME 列的值表明了计时器适用的监控类型， 而 TIMER_NAME 表明那个计时器应用在这些监控。计时器应用于那些名字以一个匹配 NAME 值的组件打头的监控。

通过修改 TIMER_NAME 列的值可以修改计时器。比如，为 wait 监控使用 NANOSECOND 计时器：

```
mysql> UPDATE setup_timers SET TIMER_NAME = 'NANOSECOND'
       WHERE NAME = 'wait';
mysql> SELECT * FROM setup_timers;
+-----------+-------------+
| NAME      | TIMER_NAME  |
+-----------+-------------+
| idle      | MICROSECOND |
| wait      | NANOSECOND  |
| stage     | NANOSECOND  |
| statement | NANOSECOND  |
+-----------+-------------+
```

关于计时器的讨论，请参考 **22.3.1 性能库事件计时**

表 setup_instruments 和 setup_consumers 分别列出了那些可以为之收集事件的监控，和事件信息被真正收集的各种消费者。其他安装表可以进一步修改监控配置。**22.3.2 性能库事件过滤** 讨论了如何通过修改这些表来影响事件收集。

如果性能库配置必须在运行时使用 SQL 语句修改，但又希望这些修改能在每次数据库服务启动的时候生效，那么可以把这些 SQL 语句存到一个文件里，在启动数据库服务的时候带着 —init-file=file_name 选项。如果有多个监控配置都被调整以产生不同类型的监控，这个方法在同样有用。比如临时的数据库服务健康监控、突发事件调查、应用行为诊断等等。将每个监控配置的语句存入它们各自的文件，在启动数据库服务的时候指定合适的文件作为 —init-file 参数。

### 22.3.1 性能库事件计时

MySQL 数据库源码中加入了各种监控来收集事件。性能库通过监控时间事件来提供事件的耗时。如果不收集计时信息，也可以通过配置监控实现。本章节将讨论可用的计时器及其特性，以及事件中说的时间计数是如何表示的。

#### 性能库计时器

性能库中的以下两张表提供了计时器信息：

* performance_timers 列出了可用的计时器及其特性。
* setup_timers 指明了监控及其用到计时器的对应关系。

表 setup_timers 中的每一行必须对应表 performance_timers 中的其中一个计时器。

这些计时器在开销的精度和数量上各有不同。通过查看表 performance_timers ，可以看到这些可用的计时器及其特性：

```
mysql> SELECT * FROM performance_timers;
+-------------+-----------------+------------------+----------------+
| TIMER_NAME  | TIMER_FREQUENCY | TIMER_RESOLUTION | TIMER_OVERHEAD |
+-------------+-----------------+------------------+----------------+
| CYCLE       | 2389029850      | 1                | 72             |
| NANOSECOND  | 1000000000      | 1                | 112            |
| MICROSECOND | 1000000         | 1                | 136            |
| MILLISECOND | 1036            | 1                | 168            |
| TICK        | 105             | 1                | 2416           |
+-------------+-----------------+------------------+----------------+
```

各列的含义如下：

* TIMER_NAME 表示可用计数器的名称。CYCLE 计时器基于 CPU 周期计数器。表 setup_timers 中的可用计数器是除了 TIMER_NAME 列之外的字段值都不为 NULL 的那些。如果计时器名称对应的其它字段值为 NULL ，说明在当前操作系统平台上不支持此计时器。
* TIMER_FREQUENCY 表示每秒计时器单位的数量。对一个周期计时器来讲，频率通常 CPU 速度相关。显示值是在一个配备 2.4GHz 处理器的系统上获取的。其它计数器是基于对秒的固定分割。对 TICK 计时器来讲，其频率可能会随操作系统平台的不同而变化（例如有的操作系统每秒 100 次滴答计数，有的则是每秒 1000 次）。
* TIMER_RESOLUTION 表示计时器的值单次计数会增加多少个计时器单位。假设计时器对应的解析度值为 10，那么计时器的值每次计时增加 10 个计时器单位。
* TIMER_OVERHEAD 是给定计时器单次计时所开销的最小周期数。由于计时器在事件的开始和结束时被调用，所以每个事件的开销是该值的两倍。

使用 setup_timers 这张表可以查看当前生效的计时器或者修改计时器：

```
mysql> SELECT * FROM setup_timers;
+-----------+-------------+
| NAME      | TIMER_NAME  |
+-----------+-------------+
| idle      | MICROSECOND |
| wait      | CYCLE       |
| stage     | NANOSECOND  |
| statement | NANOSECOND  |
+-----------+-------------+
mysql> UPDATE setup_timers SET TIMER_NAME = 'MICROSECOND'
       WHERE NAME = 'idle';
mysql> SELECT * FROM setup_timers;
+-----------+-------------+
| NAME      | TIMER_NAME  |
+-----------+-------------+
| idle      | MICROSECOND |
| wait      | CYCLE       |
| stage     | NANOSECOND  |
| statement | NANOSECOND  |
+-----------+-------------+
```

性能库默认对每个监控类型使用可用的最佳计数器，但也可以选择其它定时器。

对于时间等待事件，最重要的准则是减少开销，而且可能会以损失计时器精度为代价，所以使用 CYCLE 计时器是最好的选择。

执行一条语句或 stage 的耗时，按 **基数的顺序** 会比一次独立等待的耗时更长。对语句计时，最重要的准则是有精确且不被处理器频率的变化影响的度量，所以最好使用一个不基于 CPU 周期的计时器。语句默认的计时器是 NANOSECOND 。对比 CYCLE 计时器，额外的开销并不明显，因为与执行语句本身消耗的CPU时间相比，两次调用计时器导致的开销（语句开始与结束时各一次）在基数级别上更少。这种情况下使用 CYCLE 计时器只会有害无益。

周期计数器提供的精度取决于处理器速度。如果处理器以 1GHz （每秒10亿周期）或更高主频运行，周期计数器提供的精度为亚纳秒级。使用周期计数器的代价远小于获取当天的实际时间。举例来说，标准的 gettimeofday() 函数会花费数百个周期，对每秒可能发生数千次乃至数百万次的数据收集来说，这样的开销是不可接受的。

周期计数器也会有不足之处：

* 用户希望以时钟单位计时，比如以秒的分割计时。从周期转换为秒的分割代价昂贵。基于这个原因，实际的转换操作通过快速而相当粗略的相乘实现。
* 处理器周期率可能改变，比如当一个笔记本进入省电模式或当 CPU 变慢以减少发热时。如果处理器的周期率起伏不定，从周期到真实时间单位的转换常会存在误差。
* 受限于处理器或操作系统，周期计数器可能不可靠或不可用。例如对奔腾处理器来说，其指令不是 C 指令，而是一种名为 RDTSC 的汇编语言。而且操作系统在理论上可能会组织用户模式的程序使用周期计数器。
* 乱序执行或多处理器异步相关的处理器细节可能导致计数器看上去比真实情况或快或慢高达 1000 个周期。

MySQL 周期计数器在 x386（Windows, OS X, Linux, Solaris, 和其它类 Unix 系统），PowerPC, 和 IA-64 平台上运行正常。

 #### 性能库计时器在事件中的表示

在性能库里，存储当前事件和历史事件的表有三列表示计时信息：TIMER_START 和 TIMER_END 分别表示事件的起止时间，TIMER_WAIT 表示事件持续时间。

setup_instruments 表中有一名为 ENABLED 的列用来标记为监控启用事件收集。该表还有一列 TIMED 来标记被计时的监控。如果一个监控没被启用收集事件，那么就不会产生事件。如果启用事件收集的监控没被计时，该项监控产生的事件对应的 TIMER_START, TIMER_END, 和 TIMER_WAIT 这三个计时器都取 NULL 值。 这进而导致在概要表中对时间值计算求和、最小值、最大值和平均值的时候这些值都会被忽略。

在内部，事件时间以事件计时开始时生效的计时器提供的单位存储。为了展现何时从性能库的表里检索事件，时间会以 picosecond （万亿分之一秒）计，以便标准化到一个标准的时间单位。不需顾及查询了哪个计时器。

对 setup_timers 的修改会对监控立刻生效。正在运行的事件可以对事件开始时间使用最初的计时器，对事件结束时间使用新的计时器。为了避免在更改计时器之后出现不可预知的后果，需要使用 TRUNCATE TABLE 来重置性能库统计信息。 

数据库服务启动期间，在性能库初始化阶段会生成计时器基线（time zero 零时）。事件对应的 TIMER_START 和 TIMER_END 分别表示在事件开始和结束时，自这个时间基线起过了多少 picoseconds（万亿分之一秒）。TIMER_WAIT 的值表示事件持续了多少 picoseconds（万亿分之一秒）。

事件的 Picosecond 值是近似值，其精确度受限于与不同单位相互转换相关的误差的一般形式。如果应用 CYCLE 计时器而处理器速度会变化，那么可能产生漂移。基于以上原因，把事件的 TIMER_START 值作为自数据库启动起过去的精确时间是不合理的。另一方面，在 ORDER BY 子句中使用 TIMER_START 的值来对事件按开始时间排序或使用 TIMER_WAIT 的值来对事件按持续时间排序是合理的。

在事件中选用 picoseconds 而不是毫秒数这样的值是基于性能方面的考量。过去一个实现目标是以一种统一的事件单位来展现结果，而不管计时器。理想情况下，这个时间单位看起来像时钟单位一样而且相当精确。换言之，就像毫秒一样。但是为了将周期数或纳秒数转换成毫秒，必然要对每个监控执行除法操作。在很多平台上，除法操作都开销昂贵，故而现实中采用的是开销不大的乘法操作。因此时间单位是最合理的 TIMER_FREQUENCY 值的整数倍。时间单位使用一个足够大的乘数来确保没有大的精度损失。结果就是时间单位选用了 “picoseconds.”(万亿分之一秒) 这个单位并不精确，但这样确保了开销最小化。

MySQL 5.6.26 之前的版本在执行等待、stage 或语句事件时，这些事件各自的当前事件表会展示事件已填充了 TIMER_START 值，但 TIMER_END 和 TIMER_WAIT 的值为 NULL： 

```
events_waits_current
events_stages_current
events_statements_current
```

从 MySQL 5.6.26 开始，当前事件计时提供了更多信息。为了确定尚未完成的事件已经运行了多长时间，计时器的列会按以下方式设置：

* TIMER_START 会和之前的版本一样被填充值
* TIMER_END 被填充成当前计时器的值
* TIMER_WAIT 被填充成至今为止消耗的时间（TIMER_END − TIMER_START）

尚未完成的事件其 END_EVENT_ID 值为 NULL。通过查询 TIMER_WAIT 列可以查看预估的事件已运行时间。因此，为了识别尚未完成并且到目前为止已运行时长超过 N picoseconds 的事件，监控应用程序可以在查询中使用如下表达式：

```
WHERE END_EVENT_ID IS NULL AND TIMER_WAIT > N
```

上述事件识别假定相应的监控已经启用、TIMED 的值已置为 YES 而且相关的消费者也已启用。

### 22.3.2 性能库事件过滤

事件是以一种生产者／消费者的模式在处理：

* 监控代码作为事件的起源产生被收集的事件。setup_instruments 表列出可为之收集事件的监控，不管是否被启用，也不管对已启用的监控是否收集计时信息：

```
mysql> SELECT * FROM setup_instruments;
+------------------------------------------------------------+---------+-------+
| NAME                                                       | ENABLED | TIMED |
+------------------------------------------------------------+---------+-------+
...
| wait/synch/mutex/sql/LOCK_global_read_lock                 | YES     | YES   |
| wait/synch/mutex/sql/LOCK_global_system_variables          | YES     | YES   |
| wait/synch/mutex/sql/LOCK_lock_db                          | YES     | YES   |
| wait/synch/mutex/sql/LOCK_manager                          | YES     | YES   |
...
```

setup_instruments 表提供了对事件产生的最基本的控制。对基于对象类型或被监控的线程的事件产生来说， 为了进一步改善这些事件产生，可以使用 **22.3.3 事件预过滤** 中描述的表。

* 性能库的表作为事件的终点消费事件。事件信息可被发送到消费者，setup_consumers 表列出了这些消费者的类型，不管这些消费者是否已启用：

```
mysql> SELECT * FROM setup_consumers;
+--------------------------------+---------+
| NAME                           | ENABLED |
+--------------------------------+---------+
| events_stages_current          | NO      |
| events_stages_history          | NO      |
| events_stages_history_long     | NO      |
| events_statements_current      | YES     | 
| events_statements_history      | NO      |
| events_statements_history_long | NO      |
| events_waits_current           | NO      |
| events_waits_history           | NO      |
| events_waits_history_long      | NO      |
| global_instrumentation         | YES     |
| thread_instrumentation         | YES     |
| statements_digest              | YES     |
+--------------------------------+---------+
```

事件过滤可以在不同的性能监控阶段完成：

*  **预过滤**  通过性能库配置完成预过滤，这样只从生产者收集特定类型的事件，而且收集的事件只更新特定的消费者。这可以通过启用或禁用监控或消费者实现。预过滤通过性能库完成，在全局生效，也就是可以应用到全部用户。

  以下列出了使用预过滤的原因：

*   可以减少开销。即使在启用所有监控的情况下，性能库的开销也应是最小化的，但也许你想进一步降低开销。或者不需考虑计时事件并且想要禁用计时代码来消除计时开销。
*   避免使用你不感兴趣的事件填充当前事件或历史表。预过滤在这些表中为启用监控类型的记录预留了更多空间。如果只启用预过滤的文件监控，将不会对非文件监控收集记录。使用后过滤会收集非文件事件，也会为文件事件留下少量记录。
    * 避免维护某些类型的事件表。如果禁用一个消费者，数据库服务就不再花时间为这个消费者维护终点。例如，如果不在意事件历史，那么可以禁用历史表消费者来提高性能。

*   **后过滤** 为了指明你想看到哪个可用事件，涉及到从性能库的表里查询信息的查询语句中 WHERE 子句的使用。 后过滤对单个用户生效，这是因为单个用户选择其感兴趣的可用事件。

          使用后过滤有以下原因：

    * 用来避免为个别用户在其对哪个事件信息感兴趣方面做决定。
    * 用来在不可预知强行使用预过滤的限制时，使用性能库来调查性能问题。

后续的章节将提供更多关于预过滤的细节，也会对在过滤操作中命名监控或消费者做出指导。关于通过编写查询语句来检索信息（即后过滤），请参考 **22.4 性能库查询**。

### 22.3.3 事件预过滤

预过滤是通过性能库完成，其对所有用户全局生效。预过滤可以应用到事件处理的生产者或消费者阶段：

* 在生产者阶段配置预过滤，可以使用以下几个表：
  * setup_instruments 指明了可用监控。不论其他生产相关的安装表中的内容是怎样的，setup_instruments 表中已禁用的监控不产生事件。该表中已启用的监控可以产生事件，但这受到其他表中的内容的限制。
  * setup_objects 控制性能库是否监控特定的表对象。
  * threads 指示是否对每个数据库服务线程启用监控。
  * setup_actors 决定了新的前台线程最初的监控状态。
* 为了在消费者阶段配置预过滤，要修改 setup_consumers 表。该表决定了事件被发送到的终点，而且还会暗中影响事件生产。如果给定的是事件不会被发送到任何终点（也就是说此事件将不会被消费），那么性能库就不会生产该事件。

排除如下的例外，对上述表中任何一个的修改对监控的影响都立竿见影：

* 在 setup_instruments 表中，对部分监控的修改只有在数据库服务启动时有效，在运行期间修改不会生效。即便这对其他的监控也可能成立，但这主要影响到的是数据库服务的互斥量、条件、以及读写锁。
* 对 setup_actors 表的修改只影响修改之后创建的前台线程，修改前已存在的线程不受影响。

当修改监控配置时，性能库不刷新历史表。已经收集的事件在被新事件取代之前仍然存在于当前事件表和历史事件表。如果禁用监控，需要等到这些监控对应的事件被新的事件取代之后才会起效。或者也可以使用 TRUNCATE TABLE 来清空这些历史表。

对监控做修改之后，你可能想要清空概要表。一般来说，只需要将概要列的值重置为 0 或者 NULL ，而非删除记录。这样做使你可以清空已收集的值并重新聚合。比如在做了运行时配置修改之后，这可能会有用。这种清空行为的例外被记录在独立的聚合表章节。

以下章节将描述如何使用具体的表来控制性能库预过滤。

### 22.3.4 通过监控预过滤

setup_instruments 表列出了可用的监控：

```
mysql> SELECT * FROM setup_instruments;
+------------------------------------------------------------+---------+-------+
| NAME                                                       | ENABLED | TIMED |
+------------------------------------------------------------+---------+-------+
...
| wait/synch/mutex/sql/LOCK_global_read_lock                 | YES     | YES   |
| wait/synch/mutex/sql/LOCK_global_system_variables          | YES     | YES   |
| wait/synch/mutex/sql/LOCK_lock_db                          | YES     | YES   |
| wait/synch/mutex/sql/LOCK_manager                          | YES     | YES   |
...
| wait/synch/rwlock/sql/LOCK_grant                           | YES     | YES   |
| wait/synch/rwlock/sql/LOGGER::LOCK_logger                  | YES     | YES   |
| wait/synch/rwlock/sql/LOCK_sys_init_connect                | YES     | YES   |
| wait/synch/rwlock/sql/LOCK_sys_init_slave                  | YES     | YES   |
...
| wait/io/file/sql/binlog                                    | YES     | YES   |
| wait/io/file/sql/binlog_index                              | YES     | YES   |
| wait/io/file/sql/casetest                                  | YES     | YES   |
| wait/io/file/sql/dbopt                                     | YES     | YES   |
...
```

通过将表中的 ENABLED 列设置为 YES 或 NO 可以控制是否启用监控。配置是否收集计时信息，将 TIMED 列的值设置为 YES 或 NO 即可。如 **22.3.1 性能库事件计时** 中提到的那样，设置 TIMED 列会影响性能库中表的内容。

对 setup_instruments 表中绝大多数行的修改会对监控立即生效。对部分监控来讲，修改只有在数据库服务启动时有效，运行时修改不会生效。即便这对其他的监控也可能成立，但这主要影响到的是数据库服务的互斥量、条件、以及读写锁。

setup_instruments 表提供了对事件产生的最基本的控制。对基于对象类型或被监控的线程的事件产生来说， 为了进一步改善事件产生，可以使用 **22.3.3 事件预过滤** 中描述的那些表。

以下例子展示了可以对 setup_instruments 表进行的操作。这些修改像其他预过滤操作那样，会影响到所有用户。其中部分查询使用了 LIKE 操作符来匹配监控名称。关于在查询监控时指定模式的更多信息，请参考 **22.3.9 为过滤操作指明监控或消费者**。

- 禁用所有监控：

```
mysql> UPDATE setup_instruments SET ENABLED = 'NO';
```

执行以上语句后将不会收集任何事件。

- 禁用所有文件监控以将它们加入到当前已禁用的监控集：

```
mysql> UPDATE setup_instruments SET ENABLED = 'NO'
       WHERE NAME LIKE 'wait/io/file/%';
```

- 只禁用文件监控，并启用所有其他监控：

```
mysql> UPDATE setup_instruments
       SET ENABLED = IF(NAME LIKE 'wait/io/file/%', 'NO', 'YES');
```

- 只启用在 mysys 库中的所有监控：

```
mysql> UPDATE setup_instruments
       SET ENABLED = CASE WHEN NAME LIKE '%/mysys/%' THEN 'YES' ELSE 'NO' END;
```

- 禁用指定的监控：

```
mysql> UPDATE setup_instruments SET ENABLED = 'NO'
       WHERE NAME = 'wait/synch/mutex/mysys/TMPDIR_mutex';
```

- 反置 ENABLED 的值以切换给定监控的状态：

```
mysql> UPDATE setup_instruments
       SET ENABLED = IF(ENABLED = 'YES', 'NO', 'YES')
       WHERE NAME = 'wait/synch/mutex/mysys/TMPDIR_mutex';
```

- 禁用所有监控的计时：

```
mysql> UPDATE setup_instruments SET TIMED = 'NO';
```

### 22.3.5 通过对象预过滤

setup_objects 表控制性能库是否监控特定的表对象。最初的 setup_objects 表中的内容如下：

```
mysql> SELECT * FROM setup_objects;
+-------------+--------------------+-------------+---------+-------+
| OBJECT_TYPE | OBJECT_SCHEMA      | OBJECT_NAME | ENABLED | TIMED |
+-------------+--------------------+-------------+---------+-------+
| TABLE       | mysql              | %           | NO      | NO    |
| TABLE       | performance_schema | %           | NO      | NO    |
| TABLE       | information_schema | %           | NO      | NO    |
| TABLE       | %                  | %           | YES     | YES   |
+-------------+--------------------+-------------+---------+-------+
```

对 setup_objects 表的修改会影响对象监控，这种影响在修改后立即生效。

OBJECT_TYPE 列指示一行记录应用的对象类型。TABLE 过滤会影响表 I/O 事件（等待／IO／表／sql／句柄监控）和表锁事件（等待／锁／表／sql／句柄监控）。

OBJECT_SCHEMA 列的值应该是文字的库名或可以匹配任何库名的 '%'；OBJECT_NAME 列的值是文字的表名或可以匹配任何表名的 '%'。

ENABLED 列指明匹配的对象是否已启用监控。TIMED 列指明是否收集计时信息。设置 TIMED 列会影响性能库中表的内容，这在 **22.3.1 性能库事件计时** 中已有介绍。

默认的对象配置会监控所有那些不在 mysql, INFORMATION_SCHEMA, 和 performance_schema 库中的表。（不论 setup_objects 中做何设置， INFORMATION_SCHEMA 库中的表都不会被监控；information_schema.% 这一行明确了这一默认配置。）

当性能库在 setup_objects 表中检查匹配时，会尝试首先找出更多明确的匹配。对于匹配给定 OBJECT_TYPE 的那些记录，性能库会依以下顺序检查它们：

* 记录是否满足 OBJECT_SCHEMA='literal' 且 OBJECT_NAME='literal'（即对象库和对象的名字都是文字的）
* 记录是否满足 OBJECT_SCHEMA='literal' 且 OBJECT_NAME='%'（即对象库的名字为文字的，对象名则不作限制、值为 '%' 通配符）
* 记录是否满足  OBJECT_SCHEMA='%' 且 OBJECT_NAME='%'（即对象库和对象名均无限制、值都为 '%' 通配符）

例如，对于表 db1.t1 ，性能库在 OBJECT_TYPE='TABLE' 的那些记录中先查找满足 OBJECT_SCHEMA='db1' 且 OBJECT_NAME = 't1' 的记录，再查找 OBJECT_SCHEMA='db1' 且 OBJECT_NAME = '%' 的记录，最后查找 OBJECT_SCHEMA='%' 且 OBJECT_NAME = '%' 的记录。匹配顺序很重要，因为匹配到的不同记录的 ENABLED 和 TIMED 列的值可能相异。

对于表相关事件，性能库结合 setup_objects 和
setup_instruments 表中的内容来决定是否启用监控以及是否对已启用的监控计时：

* 对那些在 setup_objects 表中有一条记录与之匹配的表，表监控只有在 setup_objects 和 setup_instruments 表同时满足 ENABLED='YES' 时才会启用。
* setup_objects 和 setup_instruments 表中的 TIMED 值是结合在一起的，所以只有当两张表同时满足 TIMED='YES' 才会收集计时信息。

假设 setup_objects 表有如下应用到 db1, db2 和 db3 库的对象类型为 TABLE 的记录：

```
+-------------+---------------+-------------+---------+-------+
| OBJECT_TYPE | OBJECT_SCHEMA | OBJECT_NAME | ENABLED | TIMED |
+-------------+---------------+-------------+---------+-------+
| TABLE       | db1           | t1          | YES     | YES   |
| TABLE       | db1           | t2          | NO      | NO    |
| TABLE       | db2           | %           | YES     | YES   |
| TABLE       | db3           | %           | NO      | NO    |
| TABLE       | %             | %           | YES     | YES   |
+-------------+---------------+-------------+---------+-------+
```

如果 setup_instruments 表中的一个表相关的监控满足 ENABLED＝'NO'，那么这个对象不会被监控。如果 ENABLED='YES'，那么根据 setup_objects 表中记录的 ENABLED 值，对是否启用事件监控可做如下判断：

- db1.t1 事件监控已启用
- db1.t2 事件监控未启用
- db2.t3 事件监控已启用
- db3.t4 事件监控未启用
- db4.t5 事件监控已启用

类似的逻辑也适用于结合 setup_objects 和 setup_instruments 表中的 TIMED 列的值来确定是否收集事件计时信息。

如果一张持久化表和一张临时表名字相同，那么对 setup_objects 表的匹配也相同。不可能为这两张表中其中的一张启用监控另一张不启用。然而两张表时被分别监控的。

ENABLED 列是自 MySQL 5.6.3 开始才有的。5.6.3 之前的版本没有这一列，setup_objects 只是被用来对在该表中匹配到的对象启用监控。没有办法用该表来显示禁用监控。

### 22.3.6 通过线程预过滤

threads 表中对每个数据库服务线程都存有一条记录。每条记录包含了一个线程的信息，指明是否为这个线程启用监控。性能库要监控一个线程，须保证满足以下条件：

* 表 setup_consumers 中 thread_instrumentation 消费者须为 YES 。
* 列 threads.INSTRUMENTED 的值须为 YES 。
* 只监控那些为 setup_instruments 表中启用的监控产生的线程事件。


threads 表中的 INSTRUMENTED 列为每个线程指明了监控状态。对前台线程（由客户端连接生成），INSTRUMENTED 的初始值取决于用户帐号是否与 setup_actors 表中的记录匹配的线程关联。

对后台线程，没有用户与之关联，INSTRUMENTED 的值默认为 YES ， 也无需查询 setup_actors 表。

setup_actors 表中最初的内容如下：

```
mysql> SELECT * FROM setup_actors;
+------+------+------+
| HOST | USER | ROLE |
+------+------+------+
| %    | %    | %    |
+------+------+------+
```

HOST 列的值应该是一个文字描述的主机名或者通配符 '%' ；USER 列的值应该是一个文字描述的用户名或者通配符 '%'。

性能库使用 HOST 和 USER 列来匹配每个新建的前台线程（未使用 ROLE 列）。如果存在记录与之匹配，那么 INSTRUMENTED 列的值变成 YES ，否则变成 NO 。这使得监控能选择性地应用在每台主机、每个用户、或者主机和用户的每个组合。

假设按照如下方式修改 setup_actors 表：

```
TRUNCATE TABLE setup_actors;
```

现在 setup_actors 表已清空，表中没有能与正传入的连接匹配的记录。因而性能库将为所有新的前台线程将 INSTRUMENTED 列的值置为 NO 。

假设进一步按如下方式修改 setup_actors 表：

```
INSERT INTO setup_actors (HOST,USER,ROLE) VALUES('localhost','joe','%');
INSERT INTO setup_actors (HOST,USER,ROLE) VALUES('%','sam','%');
```

现在性能库按如下方式确定如何为新连接的线程设置 INSTRUMENTED 的值：

* 如果 joe 从本地主机连接，那么该连接可以与插入的第一条记录匹配。
* 如果 joe 从其他主机连接，没有记录与之匹配。
* sam 从任意主机连接，该连接都能与插入的第二条记录匹配。
* 对于其他的任何连接，没有记录与之匹配。

对 setup_actors 表的修改只对修改后创建的前台线程生效，对修改前已存在的线程没有影响。如需要对已存在的线程生效，需要修改 threads 表 INSTRUMENTED 列的值。

### 22.3.7 通过消费者预过滤

setup_consumers 表列出了已启用的可用消费者类型：

```
mysql> SELECT * FROM setup_consumers;
+--------------------------------+---------+
| NAME                           | ENABLED |
+--------------------------------+---------+
| events_stages_current          | NO      |
| events_stages_history          | NO      |
| events_stages_history_long     | NO      |
| events_statements_current      | YES     |
| events_statements_history      | NO      |
| events_statements_history_long | NO      |
| events_waits_current           | NO      |
| events_waits_history           | NO      |
| events_waits_history_long      | NO      |
| global_instrumentation         | YES     |
| thread_instrumentation         | YES     |
| statements_digest              | YES     |
+--------------------------------+---------+
```

修改 setup_consumers 表可以在消费者阶段影响预过滤并决定哪些事件被发送到终点。将 ENABLED 的值设置为  YES 或 NO 可以启用／禁用对应的消费者：

```
mysql> UPDATE setup_consumers
       SET ENABLED = 'NO' WHERE NAME LIKE '%history%';
```

在 setup_consumers 表中的消费者设置形成了一个从高到低的层次结构。以下原则适用：

* 与消费者关联的终点接收不到事件，除非性能库检查此消费者而且此消费者已启用。
* 一个消费者只有在其依赖的所有消费者（如果有的话）被启用的情况下才会被检查。
* 如果一个消费者没有被检查，或者已被检查但被禁用，那么其他依赖于它的消费者都不会被检查。
* 依赖其他消费者的消费者可以有依赖该消费者的消费者。
* 如果一个事件不会被发送到任何终点，那么性能库就不会生产它。

下面列出了可用的消费者。关于几个有代表性的消费者配置以及它们对监控的影响，请参考 **22.3.8 消费者配置示例** 。

**全局消费者与线程消费者**

* global_instrumentation 是最高级别的消费者。如果 global_instrumentation 的值为 NO ，那这样会禁用全局监控；所有其他设置都是更低级别的而且它们也不会被检查，此时不管将它们设置成什么值都不会起效；没有全局或线程信息会被维护，当前事件表或事件历史表中也不会收集单独的事件。如果 global_instrumentation 的值为 YES ，性能库会为全局状态维护信息，而且也会检查 thread_instrumentation 消费者。
* thread_instrumentation 消费者只有当 global_instrumentation 的值为 YES 时才会被检查。此外，如果此时 thread_instrumentation 的值为 NO ，这样就禁用了线程特有的监控，所有更低级别的设置也会被忽略；没有线程信息会被维护，当前事件表或事件历史表中也不会收集单独的事件。如果 thread_instrumentation 的值为 YES ，性能库会维护线程特有的信息，而且会检查名称符合 events_xxx_current 形式的消费者。 

**等待事件消费者**

这些消费者需要 global_instrumentation 和 thread_instrumentation 的值都为 YES ，否则它们将不被检查。如果被检查，那么它们将表现如下：

* events_waits_current 的值如果为 NO ，那么会禁用对 events_waits_current 表中的单独等待事件的收集；如果为 YES ，则会启用等待事件收集，性能库也会检查 events_waits_history 和 events_waits_history_long 消费者。
* events_waits_history 在 event_waits_current 为 NO 的情况下不会被检查。否则，events_waits_history 取 NO 或 YES 会禁用或启用对 events_waits_history 表中的等待事件的收集。
* events_waits_history_long 在 event_waits_current 为 NO 的情况下不会被检查。否则，events_waits_history_long 取 NO 或 YES 会禁用或启用对 events_waits_history_long 表中的等待事件的收集。

**暂存事件消费者**

这些消费者需要 global_instrumentation 和 thread_instrumentation 的值都为 YES ，否则它们将不被检查。如果被检查，那么它们将表现如下：

- events_stages_current 的值如果为 NO ，那么会禁用对 events_stages_current 表中的单独暂存事件的收集；如果为 YES ，则会启用暂存事件收集，性能库也会检查 events_stages_history 和 events_stages_history_long 消费者。
- events_stages_history 在 event_stages_current 为 NO 的情况下不会被检查。否则，events_stages_history 取 NO 或 YES 会禁用或启用对 events_stages_history 表中的暂存事件的收集。
- events_stages_history_long 在 event_stages_current 为 NO 的情况下不会被检查。否则，events_stages_history_long 取 NO 或 YES 会禁用或启用对 events_stages_history_long 表中的暂存事件的收集。

**语句事件消费者**

这些消费者需要 global_instrumentation 和 thread_instrumentation 的值都为 YES ，否则它们将不被检查。如果被检查，那么它们将表现如下：

- events_statements_current 的值如果为 NO ，那么会禁用对 events_statements_current 表中的单独语句事件的收集；如果为 YES ，则会启用语句事件收集，性能库也会检查 events_statements_history 和 events_statements_history_long 消费者。
- events_statements_history 在 event_statements_current 为 NO 的情况下不会被检查。否则，events_statements_history 取 NO 或 YES 会禁用或启用对 events_statements_history 表中的语句事件的收集。
- events_statements_history_long 在 event_statements_current 为 NO 的情况下不会被检查。否则，events_statements_history_long 取 NO 或 YES 会禁用或启用对 events_statements_history_long 表中的语句事件的收集。

**语句概要消费者**

这个消费者需要 global_instrumentation 的值为 YES ，否则它将不被检查。由于该消费者对语句事件消费者没有依赖，所以可以只获取每个概要的统计信息，而不必收集 events_statements_current 的统计信息。这样在开销方面是有益的。反之，也可以在 events_statements_current 中获得详细的没有概要的语句（此情形下，DIGEST 和 DIGEST_TEXT 列的值都为 NULL ）。

### 22.3.8 消费者配置示例

在 setup_consumers 表中的消费者设置形成了一个从高到低的层次结构。下面将描述消费者如何工作、展示特定的配置、以及当消费者设置被从高级到低级逐步启用后的影响。展示的消费者的值是有代表性的。这里描述的通用准则也可以应用到其他可用的消费者的值。

配置描述按照功能和开销逐步增加的顺序列出。如果不需要启用低级设置后生成的信息，那就禁用这些设置。这样性能库会为你执行更少的代码，你也将筛选更少的信息。

setup_consumers 表中的消费者的层次结构如下：

```
global_instrumentation
 thread_instrumentation
   events_waits_current
     events_waits_history
     events_waits_history_long
   events_stages_current
     events_stages_history
     events_stages_history_long
   events_statements_current
     events_statements_history
     events_statements_history_long
 statements_digest
```

[^备注]: 在消费者层次结构中，等待、暂存和语句消费者的层级相同。这有别于事件的嵌套层次关系，对事件来说，等待事件嵌在暂存事件中，暂存事件嵌在语句事件中。

如果给定的消费者设置为 NO ，性能库会禁用这个消费者相关的监控，并忽略所有更低级别的设置。如果给定设置为 YES ，性能库会启用相关的监控，并检查更低一级的设置。关于每个消费者的规则的描述，请参考 **22.3.7 通过消费者预过滤** 。

例如，如果 global_instrumentation 被启用，thread_instrumentation 会被检查。如果 thread_instrumentation 被启用，events_xxx_current 消费者会被检查。如果 events_waits_current 被启用， events_waits_history 和 events_waits_history_long 会被检查。

下面的每个配置描述指示了性能库检查哪个安装元素以及它维护哪个输出表（即为哪个表收集信息）。

**不启用监控**

数据库服务配置的状态：

```
mysql> SELECT * FROM setup_consumers;
+---------------------------+---------+
| NAME                      | ENABLED |
+---------------------------+---------+
| global_instrumentation    | NO      |
...
+---------------------------+---------+
```

如此配置，将不做任何监控。

被检查的安装元素：

* setup_consumers 表，global_instrumentation 消费者。

被维护的输出表：

* 无。

**只启用全局监控**

数据库服务配置的状态：

```
mysql> SELECT * FROM setup_consumers;
+---------------------------+---------+
| NAME                      | ENABLED |
+---------------------------+---------+
| global_instrumentation    | YES     |
| thread_instrumentation    | NO      |
...
+---------------------------+---------+
```

如此配置，将只对全局状态维护监控。按线程监控被禁用。

与上述配置相关的额外的被检查的安装元素如下：

- setup_consumers 表，thread_instrumentation 消费者。
- setup_instruments 表
- setup_objects 表
- setup_timers 表

与上述配置相关的被维护的输出表如下：

* mutex_instances
* rwlock_instances
* cond_instances
* file_instances
* users
* hosts
* accounts
* socket_summary_by_event_name
* file_summary_by_instance
* file_summary_by_event_name
* objects_summary_global_by_type
* table_lock_waits_summary_by_table

- table_io_waits_summary_by_index_usage
- table_io_waits_summary_by_table
- events_waits_summary_by_instance
- events_waits_summary_global_by_event_name
- events_stages_summary_global_by_event_name
- events_statements_summary_global_by_event_name

**只启用全局和线程监控**

数据库服务配置的状态：

```
mysql> SELECT * FROM setup_consumers;
+---------------------------+---------+
| NAME                      | ENABLED |
+---------------------------+---------+
| global_instrumentation    | YES     |
| thread_instrumentation    | YES     |
| events_waits_current      | NO      |
...
| events_stages_current     | NO      |
...
| events_statements_current | YES     |
...
+---------------------------+---------+
```

如此配置，将在全局范围和按线程维护监控。在当前事件和事件历史表中不会收集单独的事件。

与上述配置相关的额外的被检查的安装元素如下：

- setup_consumers 表，events_xxx_current 消费者，其中 xxx 可取 waits, stages, statements 。
- setup_actors 表
- threads.instrumented 列

与上述配置相关的被维护的输出表如下：

- events_xxx_summary_by_yyy_by_event_name ，其中 xxx 可取 waits, stages, statements，而 yyy 可取 thread, user, host, account 。


**启用全局、线程和当前事件监控**

数据库服务配置的状态：

```
mysql> SELECT * FROM setup_consumers;
+--------------------------------+---------+
| NAME                           | ENABLED |
+--------------------------------+---------+
| global_instrumentation         | YES     |
| thread_instrumentation         | YES     |
| events_waits_current           | YES     |
| events_waits_history           | NO      |
| events_waits_history_long      | NO      |
| events_stages_current          | YES     |
| events_stages_history          | NO      |
| events_stages_history_long     | NO      |
| events_statements_current      | YES     |
| events_statements_history      | NO      |
| events_statements_history_long | NO      |
...
+--------------------------------+---------+
```

如此配置，将在全局范围和按线程维护监控。单独的事件会收集在当前事件表而非事件历史表中。

与上述配置相关的额外的被检查的安装元素如下：

* events_xxx_history 消费者，其中 xxx 可取 waits, stages, statements
* events_xxx_history_long 消费者，其中 xxx 可取 waits, stages, statements

与上述配置相关的被维护的输出表如下：

* events_xxx_current ，其中 xxx 可取 waits, stages, statements

**启用全局、线程、当前事件和事件历史监控**

因为 events_xxx_history 和 events_xxx_history_long 消费者被禁用，前述的几种配置不会收集事件历史。这些消费者可以被分别获一起启用，以便按线程、在全局范围、或在全局范围按线程来收集事件历史。

以下配置按线程收集事件历史，但不在全局范围内收集。：

```
mysql> SELECT * FROM setup_consumers;
+--------------------------------+---------+
| NAME                           | ENABLED |
+--------------------------------+---------+
| global_instrumentation         | YES     |
| thread_instrumentation         | YES     |
| events_waits_current           | YES     |
| events_waits_history           | YES     |
| events_waits_history_long      | NO      |
| events_stages_current          | YES     |
| events_stages_history          | YES     |
| events_stages_history_long     | NO      |
| events_statements_current      | YES     |
| events_statements_history      | YES     |
| events_statements_history_long | NO      |
...
+--------------------------------+---------+
```

为以上配置维护的事件历史表如下：

- events_xxx_history ，其中 xxx 可取 waits, stages, statements

以下配置在全局范围收集事件历史，但不是按线程收集：

```
mysql> SELECT * FROM setup_consumers;
+--------------------------------+---------+
| NAME                           | ENABLED |
+--------------------------------+---------+
| global_instrumentation         | YES     |
| thread_instrumentation         | YES     |
| events_waits_current           | YES     |
| events_waits_history           | NO      |
| events_waits_history_long      | YES     |
| events_stages_current          | YES     |
| events_stages_history          | NO      |
| events_stages_history_long     | YES     |
| events_statements_current      | YES     |
| events_statements_history      | NO      |
| events_statements_history_long | YES     |
...
+--------------------------------+---------+
```

为以上配置维护的事件历史表如下：

- events_xxx_history_long ，其中 xxx 可取 waits, stages, statements

以下配置在全局范围按线程收集事件历史：

```
mysql> SELECT * FROM setup_consumers;
+--------------------------------+---------+
| NAME                           | ENABLED |
+--------------------------------+---------+
| global_instrumentation         | YES     |
| thread_instrumentation         | YES     |
| events_waits_current           | YES     |
| events_waits_history           | YES     |
| events_waits_history_long      | YES     |
| events_stages_current          | YES     |
| events_stages_history          | YES     |
| events_stages_history_long     | YES     |
| events_statements_current      | YES     |
| events_statements_history      | YES     |
| events_statements_history_long | YES     |
...
+--------------------------------+---------+
```

为以上配置维护的事件历史表如下：

- events_xxx_history ，其中 xxx 可取 waits, stages, statements
- events_xxx_history_long ，其中 xxx 可取 waits, stages, statements

### 22.3.9 为过滤操作指明监控或消费者

为过滤操作给定的名称可以如要求的那样是明确的或大致的。通过列出全名可以指示单一的监控或消费者：

```
mysql> UPDATE setup_instruments
       SET ENABLED = 'NO'
       WHERE NAME = 'wait/synch/mutex/myisammrg/MYRG_INFO::mutex';
mysql> UPDATE setup_consumers
       SET ENABLED = 'NO' WHERE NAME = 'events_waits_current';
```

匹配到群成员的模式，可用来指定一组监控或消费者：

```
mysql> UPDATE setup_instruments
       SET ENABLED = 'NO'
       WHERE NAME LIKE 'wait/synch/mutex/%';
mysql> UPDATE setup_consumers
       SET ENABLED = 'NO' WHERE NAME LIKE '%history%';
```

如果使用模式，应该选择能匹配到所有相关的项而非其他项的模式。例如，选择所有的文件 I/O 监控最好使用包含完整的监控名前缀的模式：

```
... WHERE NAME LIKE 'wait/io/file/%';
```

模式 '%/file/%' 将匹配到名称中含有 '/file/' 的其他监控。模式 '%file%' 则更不前档，因为它将匹配名称中包含 'file' 的所有监控，比如 wait/synch/mutex/sql/LOCK_des_key_file 。

可以进行简单的测试来检查一个模式能匹配到哪些监控或消费者名称：

```
mysql> SELECT NAME FROM setup_instruments WHERE NAME LIKE 'pattern';
mysql> SELECT NAME FROM setup_consumers WHERE NAME LIKE 'pattern';
```

关于支持的名称类型信息，请参考 **22.5 性能库监控命名规范**

### 22.3.10 确定监控内容

总能通过检查 setup_instruments 表来确定性能库包含了什么监控。例如，可使用如下查询来查看为 InnoDB 存储引擎监控了哪些文件相关的事件：

```
mysql> SELECT * FROM setup_instruments WHERE NAME LIKE 'wait/io/file/innodb/%';
+--------------------------------------+---------+-------+
| NAME                                 | ENABLED | TIMED |
+--------------------------------------+---------+-------+
| wait/io/file/innodb/innodb_data_file | YES     | YES   |
| wait/io/file/innodb/innodb_log_file  | YES     | YES   |
| wait/io/file/innodb/innodb_temp_file | YES     | YES   |
+--------------------------------------+---------+-------+
```

本文档中没有给出监控内容的精确详尽的描述，这基于以下原因：

* 监控内容是数据库服务代码。此代码经常变动，这也会影响监控集。
* 列出所有的数百个监控是不实际的。
* 如前所述，能通过查询 setup_instruments 表来确定性能库包含了什么监控。这些信息总是与使用的 MySQL 版本保持一致，也会包含核心服务之外的监控插件安装后增加的监控。这些监控内容可以供自动化工具使用。

### 22.16.1 使用性能库进行查询分析

下面的示例演示了如何使用性能库语句事件和暂存事件来检索 SHOW PROFILES 和 SHOW PROFILE 语句提供的分析信息。

在这个例子中，语句和暂存事件数据被收集在 events_statements_history_long 和 events_stages_history_long 表中。如果数据库服务繁忙、有很多活动等前台线程，历史表中的数据可能在你检索到要分析的信息之前就已过期。如果遇到这个问题，有以下选项：

* 在拥有更少前台线程活动的测试实例上运行查询。

* 通过对其他已存在的前端线程设置 threads.INSTRUMENTED = NO ，禁用对它们的监控。例如下面的语句为所有除了 test_user 线程之外的其他线程:

  ```
  mysql> UPDATE performance_schema.threads SET INSTRUMENTED = 'NO'
         WHERE TYPE='FOREGROUND' AND PROCESSLIST_USER NOT LIKE 'test_user';
  ```

  然而，要当心那些默认总是会被监控的新线程。

* 增加 events_statements_history_long 和 events_stages_history_long 表中的记录数。自 MySQL 5.6.6 起， performance_schema_events_statements_history_size 和 performance_schema_events_stages_history_size 这两个配置选项默认被自动赋值，但也能在数据库服务启动时显式设置。通过运行 SHOW VARIABLES 可以查看这两项配置的当前值。关于自动赋值的性能库参数，请参考 **22.2.2 性能库启动配置**

性能库以万亿分之一秒为单位展示事件计时器信息，这样就能将计时数据规范为以标准的时间单位展示。在下面的例子中， TIMER_WAIT 的值被除以 1000000000000 来以秒为单位展示数据。这些值也被截断成带有 6 个小数位以便展示数据时与 SHOW PROFILES 和 SHOW PROFILE 语句的格式相同。

1. 通过更新 setup_instruments 表来确保语句和暂存监控被启用。默认有些监控可能已经启用。

   ```
   mysql> UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'
          WHERE NAME LIKE '%statement/%';
   mysql> UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'
          WHERE NAME LIKE '%stage/%';
   ```

2.  确保 events_statements\_\* 和 events_stages\_\* 消费者被启用。默认有些消费者可能已经启用。

   ```
   mysql> UPDATE performance_schema.setup_consumers SET ENABLED = 'YES'
          WHERE NAME LIKE '%events_statements_%';
   mysql> UPDATE performance_schema.setup_consumers SET ENABLED = 'YES'
          WHERE NAME LIKE '%events_stages_%';
   ```

3. 运行想要分析的 SQL 语句。例如：

   ```
   mysql> SELECT * FROM employees.employees WHERE emp_no = 10001;
   +--------+------------+------------+-----------+--------+------------+
   | emp_no | birth_date | first_name | last_name | gender | hire_date  |
   +--------+------------+------------+-----------+--------+------------+
   | 10001  | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
   +--------+------------+------------+-----------+--------+------------+
   ```

   ​

4. 通过查询  events_statements_history_long 表可以确定 SQL 语句对应的 EVENT_ID 。这与运行 SHOW PROFILES 来确定 Query_ID 类似。下列查询产生的输出类似于 SHOW PROFILES 的输出：

   ```
   mysql> SELECT EVENT_ID, TRUNCATE(TIMER_WAIT/1000000000000,6) as Duration, SQL_TEXT
          FROM performance_schema.events_statements_history_long WHERE SQL_TEXT like '%10001%';
   +----------+----------+--------------------------------------------------------+
   | event_id | duration | sql_text                                               |
   +----------+----------+--------------------------------------------------------+
   | 31       | 0.028310 | SELECT * FROM employees.employees WHERE emp_no = 10001 |
   +----------+----------+--------------------------------------------------------+
   ```

5. 查询 events_stages_history_long 表可以检索语句的暂存事件。暂存事件和语句事件通过事件嵌套关联。每条暂存事件记录都有一个 NESTING_EVENT_ID 列，该列记录着父语句事件的 EVENT_ID 。

   ```
   mysql> SELECT event_name AS Stage, TRUNCATE(TIMER_WAIT/1000000000000,6) AS Duration
          FROM performance_schema.events_stages_history_long WHERE NESTING_EVENT_ID=31;
   +--------------------------------+----------+
   | Stage                          | Duration |
   +--------------------------------+----------+
   | stage/sql/starting             | 0.000080 |
   | stage/sql/checking permissions | 0.000005 |
   | stage/sql/Opening tables       | 0.027759 |
   | stage/sql/init                 | 0.000052 |
   | stage/sql/System lock          | 0.000009 |
   | stage/sql/optimizing           | 0.000006 |
   | stage/sql/statistics           | 0.000082 |
   | stage/sql/preparing            | 0.000008 |
   | stage/sql/executing            | 0.000000 |
   | stage/sql/Sending data         | 0.000017 |
   | stage/sql/end                  | 0.000001 |
   | stage/sql/query end            | 0.000004 |
   | stage/sql/closing tables       | 0.000006 |
   | stage/sql/freeing items        | 0.000272 |
   | stage/sql/cleaning up          | 0.000001 |
   +--------------------------------+----------+
   ```




