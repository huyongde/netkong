---
layout: post
title: mysql 两阶段提交
tags: mysql
---

# 什么是两阶段提交
两阶段提交是mysql在使用innodb引擎时，为了提供crash-safe 能力而设计的，两阶段提交涉及两个日志
server层的逻辑日志 bin log(归档日志) 和 innodb引擎层的物理日志redo log , redolog 存在prepare和commit两个状态
bin log和redo log有啥区别呢？ 总结如下三点：
* redo log是InnoDB引擎特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用。
* redo log是物理日志，记录的是“在某个数据页上做了什么修改”；binlog是逻辑日志，记录的是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1 ”。
* redo log是循环写的，空间固定会用完；binlog是可以追加写入的。“追加写”是指binlog文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。


基于这两个日志， mysql 的一条更新语句会存在两阶段提交， 一条更新语句的主要流程是：
1. server 执行器调用引擎层读接口找到要做更新的记录，如果对应的记录在引擎层的内存里，直接返回给执行器；否则，需要先从磁盘读入内存再返回
2. server 执行器拿到对应的记录后做更新并再调用引擎层的写接口
3. 引擎层将更新的数据写入内存中，并记录redo log, 并且redo log状态标记为prepare, 然后给执行器返回成功，可以做事务提交了
4. server 层执行器生成操作的bin log ,并把bin log写入磁盘， 然后调用引擎层的事务提交接口，引擎层把redolog 的状态从prepare 更新为commit， 更新语句执行完成

用下图来表达更新语句中的两阶段提交

![mysql](/image/mysql-two-phase-commit.jpg)






# 为什么两阶段提交可以提供crash safe 能力

关于为什么两阶段提交可以保证crash safe，主要以下原因
1. 如果在写redolog之前，就崩溃，那么redolog 和 binlog 中均未写入数据，数据可以保持一致性。
2. 如果在写入redolog之后，即prepare状态的redolog，
    a. 在写入binlog之前，系统崩溃。当系统恢复时，prepare状态的redolog中的事务ID，binlog中不能被找到，即直接回滚redo log
    b. 在写入binlog之后，标记redolog为commit之前，系统崩溃。当系统恢复时，redo log的事务ID，在binlog中是可以被找到的，那么直接提交数据，标记redolog从prepare状态至commit状态



