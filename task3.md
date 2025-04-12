# Ускорить запрос "anti-join", добиться времени выполнения < 10sec

``` sql
select day from t2 where t_id not in ( select t1.id from t1 );
```
Изначальный запрос выполнялся слишком долгое время, так как not in с подзапросом требует полного сканирования таблиц, также для каждой строки t2 происходит проверка всех строк t1. Добавим индексы для id и поменяем запрос добавив left join + is null вместо not in для увеличение эффективности:
``` sql
create index t1_id_idx on t1(id);
create index t2_t_id_idx on t2(t_id);
create index t2_t_id_day_idx on t2(t_id) INCLUDE (day);
```

Итоговый запрос:
``` sql
explain (analyze, buffers) select t2.day from t2 left join t1 on t2.t_id = t1.id where t1.id is null;
```
Полученный результат:
``` sql
                                                                     QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------
 Merge Anti Join  (cost=15.19..499261.87 rows=1 width=9) (actual time=5983.658..5983.659 rows=0 loops=1)
   Merge Cond: (t2.t_id = t1.id)
   Buffers: shared hit=11 read=46478
   ->  Index Only Scan using t2_t_id_day_idx on t2  (cost=0.43..152012.43 rows=5000000 width=13) (actual time=0.030..1312.939 rows=5000000 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=4 read=19157
   ->  Index Only Scan using t1_id_idx on t1  (cost=0.43..259749.43 rows=10000000 width=4) (actual time=0.019..2340.388 rows=9999999 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=7 read=27321
 Planning:
   Buffers: shared hit=29 read=6
 Planning Time: 3.128 ms
 Execution Time: 6005.301 ms
(13 rows)
```
