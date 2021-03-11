# Proj 3 DDL

> Date: 2021/03/11 - 2021/03/XX

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

### 2.2 实现

### 2.3 结果