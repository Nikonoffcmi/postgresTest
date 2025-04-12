# Ускорить запрос "semi-join", добиться времени выполнения < 10sec

``` sql
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
Изначальный запрос выполнялся слишком долгое время, так как in с подзапросом требует полного сканирования таблиц, также для каждой строки t2 происходит проверка условия in через вложенный цикл. Добавим индексы для id и day и поменяем запрос добавив exist вместо in для увеличение эффективности:
``` sql
create index t1_id_idx on t1(id);
create index t2_t_id_day_idx on t2(t_id, day);
```

Итоговый запрос:
``` sql
explain (analyze, buffers) select t2.day from t2 where exist (select 1 from t1 where t1.id = t2.t_id) and day > to_char(date_trunc('day', now() - '1 month'::interval), 'yyyymmdd');
```

Результат:
``` sql
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 Merge Semi Join  (cost=15.20..418045.54 rows=835255 width=9) (actual time=0.389..8044.247 rows=827772 loops=1)
   Merge Cond: (t2.t_id = t1.id)
   Buffers: shared hit=10 read=46479
   ->  Index Only Scan using t2_t_id_day_idx on t2  (cost=0.44..122864.99 rows=835255 width=13) (actual time=0.338..4616.606 rows=827772 loops=1)
         Index Cond: (day > to_char(date_trunc('day'::text, (now() - '1 mon'::interval)), 'yyyymmdd'::text))
         Heap Fetches: 0
         Buffers: shared hit=4 read=19157
   ->  Index Only Scan using t1_id_idx on t1  (cost=0.43..259749.43 rows=10000000 width=4) (actual time=0.037..2115.298 rows=9999993 loops=1)
         Heap Fetches: 0
         Buffers: shared hit=6 read=27322
 Planning:
   Buffers: shared hit=30 read=6
 Planning Time: 4.169 ms
 Execution Time: 8106.542 ms
(14 rows)
```