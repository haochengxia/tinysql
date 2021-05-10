# Proj 3 DDL

> Date: 2021/03/11 - 2021/03/15

## 1. 基础概念
区分DML，DDL，DCL，TCL：
1. DML: Data Manipulation Language, 操作对象为**数据记录**。实现对数据库中数据记录的增删改查包括SELECT、UPDATE、INSERT、DELETE。
2. DDL: Data Definition Language, 操作对象为**表**。用在定义或改变表的结构，数据类型，表之间的链接和约束等初始化工作(大多在建表时使用)，有CREATE、ALTER、DROP等。
3. DCL: Data Control Language,操作对象即为数据库，用来设置或更改数据库用户或角色权限，包括grant,deny,revoke等语句。在默认状态下，只有`sysadmin`,`dbcreator`,`db_owner`或`db_securityadmin`等可以执行。
4. TCL: Transaction Control Language
   | Op | Description |
   | - | - |
   |COMMIT | 保存已完成的工作|
   |SAVEPOINT | 在事务中设置保存点，可以回滚到此处|
   |ROLLBACK | 回滚| 
   |SET TRANSACTION | 改变事务选项|

## 2. Homework

### 2.1 分析

修改`onDropColumn`函数，基本逻辑可以按照`onAddColumn`逆推。

`colInfo.State`的经历以下四个阶段，起始为`StatePublic`：
1. StatePublic
   将状态转变为StateWriteOnly，并更新`tblInfo.Columns`将删除的列移动至最后；
2. StateWriteOnly
   将状态转变为StateDeleteOnly；
3. StateDeleteOnly
   将状态转变为StateDeleteReorganization；
4. StateDeleteReorganization → finish job
   更新`tblInfo.Columns`删除最后的行，结束任务。

### 2.2 实现

见文件[column.go](../../ddl/column.go#L217-L268).

### 2.3 结果

```bash
$ go test . -check.f TestDropColumn #-v
ok      github.com/pingcap/tidb/ddl     0.721s
$ go test . -check.f TestColumnChange #-v
ok      github.com/pingcap/tidb/ddl     0.631s
```