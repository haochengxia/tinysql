# Proj 1 Relational Algebra

> Date: 2021/02/19

## Part 1 SQL语法

省略SQL语法基础。

### 1.1 Homework

作业描述见[Link](https://github.com/tidb-incubator/tinysql/blob/course/courses/proj1-part1-README-zh_CN.md)

1. Revising the Select Query I
   ```SQL
   select * from CITY where POPULATION > 100000 and COUNTRYCODE = 'USA';
   ```

2. Revising the Select Query II
   ```SQL
   select NAME from CITY where POPULATION > 120000 and COUNTRYCODE = 'USA';
   ```

3. Revising Aggregations - The Count Function
    ```SQL
    select count(*) from CITY where POPULATION > 100000;
   ```

4. Revising Aggregations - The Sum Function
    ```SQL
    select sum(POPULATION) from CITY where DISTRICT = 'California';
   ```

5. Revising Aggregations - Averages
   ```SQL
    select avg(POPULATION) from CITY where DISTRICT = 'California';
   ```

6. African Cities
    ```SQL
    select CITY.NAME from CITY inner join COUNTRY on CITY.COUNTRYCODE = COUNTRY.CODE where CONTINENT = 'Africa';
   ```

7. Average Population of Each Continent
    ```SQL
   select CONTINENT, floor(avg(CITY.POPULATION)) from CITY inner join COUNTRY on CITY.COUNTRYCODE = COUNTRY.CODE group by CONTINENT;
   ```

8. Binary Tree Nodes
    ```SQL
   select
      N,
      (case when P is null then 'Root'
      when N in (select P from BST) then 'Inner'
      else 'Leaf' 
      end)
   from
      BST order by N;
   ```