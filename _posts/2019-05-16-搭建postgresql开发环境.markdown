---
layout: post
title:  "搭建postgresql开发环境"
date:   2019-05-15 16:55:10
categories: postgresql
---

### 下载postgresql源码
```
cd YOUR_WORK_DIR
git clone https://github.com/postgres/postgres
```

### 配置构建参数
```
./configure --enable-cassert --enable-debug --enable-depend CFLAGS="-ggdb -Og -g3 -fno-omit-frame-pointer" --prefix=YOUR_INSTALL_DIR

```
- prefix应当设置为开发者指定的本地安装目录
- 参数相关说明参考 [link](https://wiki.postgresql.org/wiki/Developer_FAQ#Development_Process)
What debugging features are available?

### 构建安装postgres
```
make
make install
```

### 初始化数据库
```
cd YOUR_INSTALL_DIR/bin
./initdb -D YOUR_DATABASE_DIR
```
- -D参数用来指定数据库存储的位置

### 调试
```
cd YOUR_INSTALL_DIR/bin
./pg_ctl -D YOUR_DATABASE_DIR start -l logfile
```
输出：
```
2019-05-16 17:10:42.939 CST [5618] LOG:  starting PostgreSQL 12devel on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 5.4.0-6ubuntu1~16.04.11) 5.4.0 20160609, 64-bit
2019-05-16 17:10:42.946 CST [5618] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2019-05-16 17:10:42.950 CST [5618] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2019-05-16 17:10:42.965 CST [5624] LOG:  database system was shut down at 2019-05-15 20:16:07 CST
2019-05-16 17:10:42.972 CST [5618] LOG:  database system is ready to accept connections
```
- 查看postgres进程
`ps -ux | grep postgres` ：
```
binguo    4112  0.0  0.2 180136 17948 ?        Ss   13:52   0:00 /home/binguo/db_install/postgres/bin/postgres -D /home/binguo/db_install/postgres_data
binguo    4114  0.0  0.0 180136  2612 ?        Ss   13:52   0:00 postgres: checkpointer   
binguo    4115  0.0  0.0 180136  2612 ?        Ss   13:52   0:00 postgres: background writer   
binguo    4116  0.0  0.0 180136  2612 ?        Ss   13:52   0:00 postgres: walwriter   
binguo    4117  0.0  0.0 180700  5524 ?        Ss   13:52   0:00 postgres: autovacuum launcher   
binguo    4118  0.0  0.0  34556  2428 ?        Ss   13:52   0:00 postgres: stats collector   
binguo    4119  0.0  0.0 180560  5604 ?        Ss   13:52   0:00 postgres: logical replication launcher   
binguo    4122  0.0  0.0  21296   972 pts/18   S+   13:52   0:00 grep --color=auto postgres
```

- 客户端连接后端进程

```
./psql -h 127.0.0.1 -d postgres
psql (12devel)
Type "help" for help.

postgres=# 
```

- 再次查看postgres进程`ps -ux | grep postgres`
```
binguo    4114  0.0  0.0 180136  2612 ?        Ss   13:52   0:00 postgres: checkpointer   
binguo    4115  0.0  0.0 180136  4672 ?        Ss   13:52   0:00 postgres: background writer   
binguo    4116  0.0  0.1 180136  8228 ?        Ss   13:52   0:00 postgres: walwriter   
binguo    4117  0.0  0.0 180700  5524 ?        Ss   13:52   0:00 postgres: autovacuum launcher   
binguo    4118  0.0  0.0  34556  2428 ?        Ss   13:52   0:00 postgres: stats collector   
binguo    4119  0.0  0.0 180560  5604 ?        Ss   13:52   0:00 postgres: logical replication launcher   
binguo    4165  0.6  0.4 664892 38168 ?        Sl   13:53   0:00 gedit /home/binguo/db_install/postgres/bin/logfile
binguo    4207  0.0  0.0  37728  4476 pts/18   S+   13:54   0:00 ./psql -h 127.0.0.1 -d postgres
binguo    4208  0.0  0.1 181584  9972 ?        Ss   13:54   0:00 postgres: binguo postgres 127.0.0.1(53174) idle
binguo    4355  0.0  0.0  21296  1012 pts/19   S+   13:55   0:00 grep --color=auto postgres
```
- 在客户端交互界面查询后端进程ID

```
psql (12devel)
Type "help" for help.

postgres=# select pg_backend_pid();
 pg_backend_pid 
----------------
           4208
(1 row)
```

可以知道后端主进程创建了一个新进程用来与该客户端进行交互
- 利用gdb进行断点调试`sudo gdb attach 4208`
- 屏蔽无用中断信号

```
(gdb) handle SIGUSR1 nostop pass
Signal        Stop	Print	Pass to program	Description
SIGUSR1       No	Yes	Yes		User defined signal 1
```

- 设置断点

```
(gdb) b ExecResult
Breakpoint 1 at 0x67d2a6: file nodeResult.c, line 69.
```

- 客户端发起查询

```
postgres=# select 1 + 1;
```

- 程序在断点处停止

```
(gdb) c
Continuing.

Breakpoint 1, ExecResult (pstate=0x24e1390) at nodeResult.c:69
69	{
```

- 查看调用栈

```
(gdb) bt
#0  ExecResult (pstate=0x24e1390) at nodeResult.c:69
#1  0x0000000000659139 in ExecProcNodeFirst (node=0x24e1390) at execProcnode.c:445
#2  0x0000000000651e72 in ExecProcNode (node=0x24e1390) at ../../../src/include/executor/executor.h:239
#3  ExecutePlan (estate=estate@entry=0x24e1138, planstate=0x24e1390, use_parallel_mode=false, operation=operation@entry=CMD_SELECT, 
    sendTuples=sendTuples@entry=true, numberTuples=numberTuples@entry=0, direction=ForwardScanDirection, dest=0x2447710, execute_once=true)
    at execMain.c:1648
#4  0x0000000000652a72 in standard_ExecutorRun (queryDesc=0x24434d8, direction=ForwardScanDirection, count=0, execute_once=<optimized out>)
    at execMain.c:365
#5  0x0000000000652afa in ExecutorRun (queryDesc=queryDesc@entry=0x24434d8, direction=direction@entry=ForwardScanDirection, count=count@entry=0, 
    execute_once=<optimized out>) at execMain.c:309
#6  0x00000000007c7e54 in PortalRunSelect (portal=portal@entry=0x2486168, forward=forward@entry=true, count=0, count@entry=9223372036854775807, 
    dest=dest@entry=0x2447710) at pquery.c:929
#7  0x00000000007c94b9 in PortalRun (portal=portal@entry=0x2486168, count=count@entry=9223372036854775807, isTopLevel=isTopLevel@entry=true, 
    run_once=run_once@entry=true, dest=dest@entry=0x2447710, altdest=altdest@entry=0x2447710, completionTag=0x7ffd8118c8e0 "") at pquery.c:770
#8  0x00000000007c5a7d in exec_simple_query (query_string=query_string@entry=0x24212d8 "select 1 + 1;") at postgres.c:1215
#9  0x00000000007c778a in PostgresMain (argc=<optimized out>, argv=argv@entry=0x244c310, dbname=0x244c150 "postgres", username=<optimized out>)
    at postgres.c:4249
#10 0x000000000073f6c4 in BackendRun (port=port@entry=0x2443140) at postmaster.c:4430
#11 0x00000000007423fe in BackendStartup (port=port@entry=0x2443140) at postmaster.c:4121
#12 0x00000000007426de in ServerLoop () at postmaster.c:1704
#13 0x000000000074396d in PostmasterMain (argc=argc@entry=3, argv=argv@entry=0x241ba30) at postmaster.c:1377
#14 0x00000000006a33a0 in main (argc=3, argv=0x241ba30) at main.c:228
```
