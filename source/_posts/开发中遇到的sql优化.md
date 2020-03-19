---
title: 开发中遇到的sql优化
date: 2020-03-15 16:14:08
tags:
---

## 表结构
```sql
-- 每个namespace 50w条bills, 100w条details
-- 1个bill对应2个details
create table bills(
    id serial8 PRIMARY KEY,
    namespace_id int8,
    value1 text
);
create table details(
    id serial8 PRIMARY KEY,
    namespace_id int8,
    bill_id int8,
    value2 text
);

create index bill_namespace_id on bills(namespace_id);
create index detail_namespace_id on details(namespace_id);
create index detail_bill_id on details(bill_id);
```

```
select count(*) from bills;
  count
----------
 10000000
(1 row)
```
```
select count(*) from details;

  count
----------
 20000000
(1 row)
```

## 减少Join条数

```sql
-- 第一种
select count(*)
from bills
join details on bills.id = details.bill_id
where bills.namespace_id = 1
```
```
 Finalize Aggregate  (cost=375270.26..375270.27 rows=1 width=8) (actual time=6121.541..6121.541 rows=1 loops=1)
   ->  Gather  (cost=375270.05..375270.26 rows=2 width=8) (actual time=6116.538..6128.755 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=374270.05..374270.06 rows=1 width=8) (actual time=6088.944..6088.945 rows=1 loops=3)
               ->  Hash Join  (cost=25459.43..373254.77 rows=406111 width=0) (actual time=395.746..6038.096 rows=333333 loops=3)
                     Hash Cond: (details.bill_id = bills.id)
                     ->  Parallel Seq Scan on details  (cost=0.00..258910.33 rows=8333333 width=8) (actual time=0.040..2076.963 rows=6666667 loops=3)
                     ->  Hash  (cost=17463.76..17463.76 rows=487333 width=8) (actual time=390.275..390.275 rows=500000 loops=3)
                           Buckets: 131072  Batches: 8  Memory Usage: 3465kB
                           ->  Index Scan using bill_namespace_id on bills  (cost=0.43..17463.76 rows=487333 width=8) (actual time=0.078..205.115 rows=500000 loops=3)
                                 Index Cond: (namespace_id = 1)
 Planning time: 0.288 ms
 Execution time: 6128.963 ms
```

```sql
-- 第二种
select count(*)
from bills
join details on bills.id = details.bill_id
where bills.namespace_id = 1 and details.namespace_id = 1
```
```
 Finalize Aggregate  (cost=63325.79..63325.80 rows=1 width=8) (actual time=986.875..986.876 rows=1 loops=1)
   ->  Gather  (cost=63325.58..63325.79 rows=2 width=8) (actual time=986.860..986.921 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=62325.58..62325.59 rows=1 width=8) (actual time=963.775..963.775 rows=1 loops=3)
               ->  Hash Join  (cost=25459.86..62276.03 rows=19818 width=0) (actual time=392.876..916.619 rows=333333 loops=3)
                     Hash Cond: (details.bill_id = bills.id)
                     ->  Parallel Index Scan using detail_namespace_id on details  (cost=0.44..30667.10 rows=406667 width=8) (actual time=0.045..198.221 rows=333333 loops=3)
                           Index Cond: (namespace_id = 1)
                     ->  Hash  (cost=17463.76..17463.76 rows=487333 width=8) (actual time=391.705..391.706 rows=500000 loops=3)
                           Buckets: 131072  Batches: 8  Memory Usage: 3465kB
                           ->  Index Scan using bill_namespace_id on bills  (cost=0.43..17463.76 rows=487333 width=8) (actual time=0.063..211.667 rows=500000 loops=3)
                                 Index Cond: (namespace_id = 1)
 Planning time: 0.353 ms
 Execution time: 987.010 ms
```


每个namespace的数据是独立的，所以无论加不加`details.namespace_id = 1`的限制，查询结果是一样的
从查询过程来看，都是`Hash Join`，但是第二种有`details.namespace_id = 1`的限制，details的条数要少很多，所以join就快了很多

## Semi Join
```sql
select count(distinct bills.id)
from bills
join details on bills.id = details.bill_id
where bills.namespace_id = 1 and details.namespace_id = 1
```
```
 Aggregate  (cost=68147.31..68147.32 rows=1 width=8) (actual time=1904.399..1904.399 rows=1 loops=1)
   ->  Gather  (cost=26459.86..68028.43 rows=47553 width=8) (actual time=620.552..1492.368 rows=1000000 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Hash Join  (cost=25459.86..62273.13 rows=19814 width=8) (actual time=612.397..1206.006 rows=333333 loops=3)
               Hash Cond: (details.bill_id = bills.id)
               ->  Parallel Index Scan using detail_namespace_id on details  (cost=0.44..30664.46 rows=406572 width=8) (actual time=0.044..268.003 rows=333333 loops=3)
                     Index Cond: (namespace_id = 1)
               ->  Hash  (cost=17463.76..17463.76 rows=487333 width=8) (actual time=610.177..610.178 rows=500000 loops=3)
                     Buckets: 131072  Batches: 8  Memory Usage: 3465kB
                     ->  Index Scan using bill_namespace_id on bills  (cost=0.43..17463.76 rows=487333 width=8) (actual time=0.216..416.889 rows=500000 loops=3)
                           Index Cond: (namespace_id = 1)
 Planning time: 0.434 ms
 Execution time: 1904.698 ms
```
```sql
select count(*)
from bills
where bills.namespace_id = 1 and exists (
    select 1 from details where namespace_id = 1 and bill_id = bills.id
)
```
```
 Finalize Aggregate  (cost=107180.12..107180.13 rows=1 width=8) (actual time=1443.874..1443.874 rows=1 loops=1)
   ->  Gather  (cost=107179.91..107180.12 rows=2 width=8) (actual time=1443.859..1443.917 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=106179.91..106179.92 rows=1 width=8) (actual time=1417.344..1417.344 rows=1 loops=3)
               ->  Hash Semi Join  (cost=52366.06..106130.37 rows=19814 width=0) (actual time=956.872..1392.469 rows=166667 loops=3)
                     Hash Cond: (bills.id = details.bill_id)
                     ->  Parallel Index Scan using bill_namespace_id on bills  (cost=0.43..14620.99 rows=203055 width=8) (actual time=0.076..93.284 rows=166667 loops=3)
                           Index Cond: (namespace_id = 1)
                     ->  Hash  (cost=36356.46..36356.46 rows=975773 width=8) (actual time=950.513..950.513 rows=1000000 loops=3)
                           Buckets: 131072  Batches: 16  Memory Usage: 3467kB
                           ->  Index Scan using detail_namespace_id on details  (cost=0.44..36356.46 rows=975773 width=8) (actual time=0.180..523.307 rows=1000000 loops=3)
                                 Index Cond: (namespace_id = 1)
 Planning time: 2.131 ms
 Execution time: 1444.183 ms
```

第一种是`Hash Join`，第二种是`Hash Semi Join`

## 多表联合查询+分页

场景：要将两个(或多个)表联合起来，得到结果再与其他表关联，最后分页

构建另一个联合的表(bill2)
```sql
-- 也插入1000w数据，每个namespace 50w
create table bills2(
    id serial8 PRIMARY KEY,
    namespace_id int8,
    value1 text
);
create index bill2_namespace_id on bills2(namespace_id);
```

```sql
select * from (
    (select * from bills where namespace_id = 1)
    union all
    (select * from bills2 where namespace_id = 1)
) tmp
join details on tmp.id = details.bill_id
order by value1 asc
limit 20
```

```
 Limit  (cost=1164304.65..1164304.70 rows=20 width=71) (actual time=2991.903..2991.909 rows=20 loops=1)
   ->  Sort  (cost=1164304.65..1179449.08 rows=6057772 width=71) (actual time=2991.902..2991.903 rows=20 loops=1)
         Sort Key: bills.value1
         Sort Method: top-N heapsort  Memory: 27kB
         ->  Merge Join  (cost=164947.82..1003109.52 rows=6057772 width=71) (actual time=732.101..2023.895 rows=2000000 loops=1)
               Merge Cond: (details.bill_id = bills.id)
               ->  Index Scan using detail_bill_id on details  (cost=0.44..694870.54 rows=19995340 width=40) (actual time=0.023..250.777 rows=1000001 loops=1)
               ->  Materialize  (cost=164947.38..169820.71 rows=974666 width=31) (actual time=732.074..1123.807 rows=1999999 loops=1)
                     ->  Sort  (cost=164947.38..167384.05 rows=974666 width=31) (actual time=732.070..895.457 rows=1000000 loops=1)
                           Sort Key: bills.id
                           Sort Method: external merge  Disk: 40000kB
                           ->  Append  (cost=0.43..44674.18 rows=974666 width=31) (actual time=0.026..317.829 rows=1000000 loops=1)
                                 ->  Index Scan using bill_namespace_id on bills  (cost=0.43..17463.76 rows=487333 width=31) (actual time=0.026..103.220 rows=500000 loops=1)
                                       Index Cond: (namespace_id = 1)
                                 ->  Index Scan using bill2_namespace_id on bills2  (cost=0.43..17463.76 rows=487333 width=31) (actual time=0.021..95.693 rows=500000 loops=1)
                                       Index Cond: (namespace_id = 1)
 Planning time: 0.490 ms
 Execution time: 2999.380 ms
 ```

```sql
select * from (
    (select * from bills where namespace_id = 1 order by value1 asc limit 20)
    union all
    (select * from bills2 where namespace_id = 1 order by value1 asc limit 20)
) tmp
join details on tmp.id = details.bill_id
order by value1 asc limit 20
```

```
 Limit  (cost=60863.48..60891.15 rows=20 width=71) (actual time=1347.949..1349.095 rows=20 loops=1)
   ->  Nested Loop  (cost=60863.48..61207.95 rows=249 width=71) (actual time=1347.948..1349.088 rows=20 loops=1)
         ->  Merge Append  (cost=60863.05..60863.85 rows=40 width=31) (actual time=1347.935..1347.958 rows=10 loops=1)
               Sort Key: bills.value1
               ->  Limit  (cost=30431.52..30431.57 rows=20 width=31) (actual time=686.947..686.949 rows=6 loops=1)
                     ->  Sort  (cost=30431.52..31649.85 rows=487333 width=31) (actual time=686.946..686.947 rows=6 loops=1)
                           Sort Key: bills.value1
                           Sort Method: top-N heapsort  Memory: 26kB
                           ->  Index Scan using bill_namespace_id on bills  (cost=0.43..17463.76 rows=487333 width=31) (actual time=0.036..104.420 rows=500000 loops=1)
                                 Index Cond: (namespace_id = 1)
               ->  Limit  (cost=30431.52..30431.57 rows=20 width=31) (actual time=660.985..660.987 rows=5 loops=1)
                     ->  Sort  (cost=30431.52..31649.85 rows=487333 width=31) (actual time=660.984..660.985 rows=5 loops=1)
                           Sort Key: bills2.value1
                           Sort Method: top-N heapsort  Memory: 27kB
                           ->  Index Scan using bill2_namespace_id on bills2  (cost=0.43..17463.76 rows=487333 width=31) (actual time=0.016..99.822 rows=500000 loops=1)
                                 Index Cond: (namespace_id = 1)
         ->  Index Scan using detail_bill_id on details  (cost=0.44..8.54 rows=6 width=40) (actual time=0.111..0.111 rows=2 loops=10)
               Index Cond: (bill_id = bills.id)
 Planning time: 0.453 ms
 Execution time: 1349.165 ms
```

第二个sql在联合之前先做一次`order + limit`，这样后面join details表就会少很多数据

如果查询第二页的话
```sql
select * from (
    (select * from bills where namespace_id = 1 order by value1 asc limit 40)
    union all
    (select * from bills2 where namespace_id = 1 order by value1 asc limit 40)
) tmp
join details on tmp.id = details.bill_id
order by value1 asc offset 20 limit 20
```
因为我们不知道第一页里bills，bill2表占了多少数据，所以需要`limit 40`，同理第三页就需要`limit 60`


## 忽略大小写匹配

```sql
select count(*) from bills where value1 ilike 'VaLue-%';
```

```
  count
----------
 10000000
(1 row)

Time: 6589.374 ms (00:06.589)
```
```sql
select count(*) from bills where upper(value1) like upper('VaLue-%')
```

```
  count
----------
 10000000
(1 row)

Time: 4730.915 ms (00:04.731)
```

第一种使用`ilike`
第二种把查询内容和字段值都转换为大写，再来`like`匹配

## 冗余字段

因为数据库执行Join操作并不是很快，增加冗余字段，减少Join的表也可以减少查询时间