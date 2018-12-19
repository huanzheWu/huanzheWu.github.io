---
layout:     post
title:      Mysql数据库压测工具——mysqlslap
subtitle:   
date:       2018-05-06
author:     huanzhewu
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Mysql
---



### 一、业务背景
需求要对数据库表进行读操作。表数据量在一万左右，字段较多，依据业务逻辑，需要进行两次排序逻辑：

1. 先按照发布类型edu_flag进行排序，edu_flag取值0、1。edu_flag为1的优先。
2. 在1的排序结果下，按照价格term_origin_price从高到低排序

因此需要执行两次ORDER BY逻辑。这会进行全表扫描。而该操作为活动页面，活动发布的时候可能请求量会较大，在未使用缓存的前提下，先测试下数据库最大能够扛多大的请求量，来决定需不需要做缓存。
开始了解的压测工具是[sysbench ](https://github.com/akopytov/sysbench  "sysbench ")。发现它只能在数据库的所在机器上进行测试，没有client的测试方式。遂使用[mysqlslap](http://dev.mysql.com/doc/refman/5.6/en/mysqlslap.html "mysqlslap")，使用起来非常方便，也满足了我的压测需求。
### 二、工具说明
mysqlslap是MySQL5.1.4之后自带的基准测试工具，它可以模拟多个客户端，指定服务器ip、port，指定查询sql，执行查询次数等条件，向数据库服务器发出查询更新，给出性能测试数据并且提供多种引擎的性能比价。还有更多的功能，通过不同的命令选项来实现。这里贴上使用参数（来自文章[mysqlslap 使用总结](https://my.oschina.net/moooofly/blog/152547 "mysqlslap 使用总结")）：
```
The following options may be given as the first argument:
--print-defaults        Print the program argument list and exit.
--no-defaults           Don't read default options from any option file,
                        except for login file.
--defaults-file=#       Only read default options from the given file #.
--defaults-extra-file=# Read this file after the global files are read.
--defaults-group-suffix=#
                        Also read groups with concat(group, suffix)
--login-path=#          Read this path from the login file.
  -?, --help          Display this help and exit.
  -a, --auto-generate-sql 自动生成测试表和数据
                      Generate SQL where not supplied by file or command line.
  --auto-generate-sql-add-autoincrement 增加auto_increment一列
                      Add an AUTO_INCREMENT column to auto-generated tables.
  --auto-generate-sql-execute-number=# 自动生成的查询的个数
                      Set this number to generate a set number of queries to
                      run.
  --auto-generate-sql-guid-primary 增加基于GUID的主键
                      Add GUID based primary keys to auto-generated tables.
  --auto-generate-sql-load-type=name 测试语句的类型。取值包括：read，key，write，update和mixed(默认)
                      read:查询 write:插入 key:读主键 update:更新主键 mixed:一半插入一半查询
                      Specify test load type: mixed, update, write, key, or
                      read; default is mixed.
  --auto-generate-sql-secondary-indexes=# 增加二级索引的个数，默认是0
                      Number of secondary indexes to add to auto-generated
                      tables.
  --auto-generate-sql-unique-query-number=# 不同查询的数量，默认值是10
                      Number of unique queries to generate for automatic tests.
  --auto-generate-sql-unique-write-number=# 不同插入的数量，默认是100
                      Number of unique queries to generate for
                      auto-generate-sql-write-number.
  --auto-generate-sql-write-number=# 
                      Number of row inserts to perform for each thread (default
                      is 100).
  --commit=#          多少条DML后提交一次
                      Commit records every X number of statements.
  -C, --compress      如果服务器和客户端支持都压缩，则压缩信息传递
                      Use compression in server/client protocol.
  -c, --concurrency=name 模拟N个客户端并发执行select。可指定多个值，以逗号或者 --delimiter 参数指定的值做为分隔符
                      Number of clients to simulate for query to run.
  --create=name       指定用于创建表的.sql文件或者字串
                      File or string to use create tables.
  --create-schema=name 指定待测试的数据库名，MySQL中schema也就是database，默认是mysqlslap
                      Schema to run tests in.
  --csv[=name]        Generate CSV output to named file or to stdout if no file
                      is named.
  -#, --debug[=#]     This is a non-debug version. Catch this and exit.
  --debug-check       Check memory and open file usage at exit.
  -T, --debug-info    打印内存和CPU的信息
                      Print some debug info at exit.
  --default-auth=name Default authentication client-side plugin to use.
  -F, --delimiter=name 文件中的SQL语句使用分割符号
                      Delimiter to use in SQL statements supplied in file or
                      command line.
  --detach=#          每执行完N个语句，先断开再重新打开连接
                      Detach (close and reopen) connections after X number of
                      requests.
  --enable-cleartext-plugin 
                      Enable/disable the clear text authentication plugin.
  -e, --engine=name   创建测试表所使用的存储引擎，可指定多个
                      Storage engine to use for creating the table.
  -h, --host=name     Connect to host.
  -i, --iterations=#  迭代执行的次数
                      Number of times to run the tests.
  --no-drop           Do not drop the schema after the test.
  -x, --number-char-cols=name 自动生成的测试表中包含多少个字符类型的列，默认1
                      Number of VARCHAR columns to create in table if
                      specifying --auto-generate-sql.
  -y, --number-int-cols=name 自动生成的测试表中包含多少个数字类型的列，默认1
                      Number of INT columns to create in table if specifying
                      --auto-generate-sql.
  --number-of-queries=# 总的测试查询次数(并发客户数×每客户查询次数)
                      Limit each client to this number of queries (this is not
                      exact).
  --only-print        只输出模拟执行的结果，不实际执行
                      Do not connect to the databases, but instead print out
                      what would have been done.
  -p, --password[=name] 
                      Password to use when connecting to server. If password is
                      not given it's asked from the tty.
  --plugin-dir=name   Directory for client-side plugins.
  -P, --port=#        Port number to use for connection.
  --post-query=name   测试完成以后执行的SQL语句的文件或者字符串 这个过程不影响时间计算
                      Query to run or file containing query to execute after
                      tests have completed.
  --post-system=name  测试完成以后执行的系统语句 这个过程不影响时间计算
                      system() string to execute after tests have completed.
  --pre-query=name    测试执行之前执行的SQL语句的文件或者字符串 这个过程不影响时间计算
                      Query to run or file containing query to execute before
                      running tests.
  --pre-system=name   测试执行之前执行的系统语句 这个过程不影响时间计算
                      system() string to execute before running tests.
  --protocol=name     The protocol to use for connection (tcp, socket, pipe,
                      memory).
  -q, --query=name    指定自定义.sql脚本执行测试。例如可以调用自定义的一个存储过程或者sql语句来执行测试
                      Query to run or file containing query to run.
  -s, --silent        不输出
                      Run program in silent mode - no output.
  -S, --socket=name   The socket file to use for connection.
  --ssl               Enable SSL for connection (automatically enabled with
                      other flags).
  --ssl-ca=name       CA file in PEM format (check OpenSSL docs, implies
                      --ssl).
  --ssl-capath=name   CA directory (check OpenSSL docs, implies --ssl).
  --ssl-cert=name     X509 cert in PEM format (implies --ssl).
  --ssl-cipher=name   SSL cipher to use (implies --ssl).
  --ssl-key=name      X509 key in PEM format (implies --ssl).
  --ssl-crl=name      Certificate revocation list (implies --ssl).
  --ssl-crlpath=name  Certificate revocation list path (implies --ssl).
  --ssl-verify-server-cert 
                      Verify server's "Common Name" in its cert against
                      hostname used when connecting. This option is disabled by
                      default.
  -u, --user=name     User for login if not current user.
  -v, --verbose       输出更多的信息
                      More verbose output; you can use this multiple times to
                      get even more verbose output.
  -V, --version       Output version information and exit.
```



### 三、测试步骤
这里说下我的测试步骤。主要包括：
1. 创建测试数据库表（与业务数据库表结构一致）
2. 写存储过程来批量插入数据，这里根据业务评估大概数据量在1W左右
3. 利用mysqlslap对业务的sql进行测试
4. 分析测试结果

#### 1. 建表
根据业务实际需求，在测试数据库建立一张表，表结构如下：
![](http://pj05m6t8l.bkt.clouddn.com/5_1.png)



#### 2.插入数据
根据对业务之前的数据，预计表的数据量在一万左右。这里插入一万行数据用于测试。通过存储过程来批量循环插入一万行数据：
```
DROP PROCEDURE IF EXISTS proc_initData;
DELIMITER $
CREATE PROCEDURE proc_initData()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i<=5000 DO
        INSERT INTO
          t_partner_info_for_press_test(aid, cid, count,
                                                  bgtime, endtime, state,
                                                  f_update_time, bgpic, tid,
                                                  creatime, edu_flag, link_title,
                                                  link_wording, link_pic, partner_topic,
                                                  back_url, start_wording, complete_wording,
                                                  end_wording, bottom_topic, bottom_url,term_origin_price)
        VALUES(131,1000246,	12,1525104000	,
                   1525708740,0,	"2018-05-01 11:12:50",
                   "http://p.qpic.cn/qqconadmin/0/647c6cfed8354b8daff05f6ab03bd389/0",
                   1000018667,"2018-05-01 11:12:50",	0,
                  "为我助力让我0元上课好吗老铁","	腾讯课堂精选好课0元上	",
                    "//10.url.cn/qqke_course_info/ajNVdqHZLLAOSCibzGCJ0Mn7EwQZgzs4ZH6lMIgIFVr9bPykqj7hmzpyzsWoHFib4YZE3xzPYZbWo/",
                  "腾讯课堂•全面学习季	腾讯课堂精选好课0元上",
                  "我正在学习老铁帮我助力可好",
                  "我已经获免费上课资格，你也可以扫描0元上课噢",
                  "你来我一步啦，我想要的课程活动已经结束。不过还有很多好课你可以扫描查看"	,
                  "限时优惠精选课程","	http://ke.qq.com","	http://ke.qq.com",
                  floor(rand()*100)
);
        SET i = i+1;
    END WHILE;
END $
CALL proc_initData();
```
执行完存储过程，可以看到数据库表中产生的数据有五千行数据，如果需要更多数据再执行存储过程即可。
![](http://pj05m6t8l.bkt.clouddn.com/5_2.png)


####  3. 使用mysqlslap进行性能测试
sql的主要逻辑是按照两个字段来先后排序后，从数据库表中获取三个属性字段。根据业务需求，需要进行压测的语句是：

```
SELECT term_origin_price,cid,actid  from t_partner_info_for_press_test WHERE  endtime >= 1525247649     AND  bgtime <= 1525247649     AND  state =0        ORDER BY edu_flag DESC ,          term_origin_price DESC    limit 0,10
```

对该sql进行mysqlslap测试，命令如下：

```
 > mysqlslap -h10.198.30.120 -P4100 --concurrency=10 --iterations=1 --create-schema='ONLINE_EDU_TEST' --query='SELECT term_origin_price,cid,actid  from t_partner_info_for_press_test WHERE  endtime >= 1525247649     AND  bgtime <= 1525247649     AND  state =0        ORDER BY edu_flag DESC ,          term_origin_price DESC    limit 0,10'  --number-of-queries=1000 --debug-info -u**** -p****
```

该语句表示：指定对10.198.30.120：4100的ONLINE_EDU_TEST数据库t_partner_info_for_press_test表进行压测，对指定的sql模拟10个客户端并发进行10000次请求。

执行之后结果输出如下：
![](http://pj05m6t8l.bkt.clouddn.com/5_3.png)

从中可以看到执行语句的平均耗时、最大耗时、最小耗时和并发线程数等。

###  4. 结果分析
从测试报告可以看出，1000的数据容量，10并发数，3查询属性，有与查询条件，无法通过加单列索引或者多列索引解决，查询需要扫描全表，性能在600/s左右。而且可经过多次测试验证，性能会根据数据量的增加而递减，需要做缓存。

依据同样的方式，可以测试不同并发数下、不同数量的查询属性的性能。也能测试sql优化是否有效。



