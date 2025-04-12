# Ускорить простой запрос, добиться времени выполнения < 10ms

Изначальный результат быстродействия запроса:
``` sql 
explain (analyze, buffers) select name from t1 where id = 50000;
```
``` sql 
                                             QUERY PLAN
-----------------------------------------------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..208396.61 rows=1 width=30) (actual time=9.798..1946.181 rows=1 loops=1)
   Filter: (id = 50000)
   Rows Removed by Filter: 9999999
   Buffers: shared hit=2432 read=80960
 Planning Time: 0.081 ms
 Execution Time: 1946.202 ms
(6 rows)
```

Как видно PostgreSQL выполняет полное сканирование таблицы, что требует чтения всех 10 миллионов строк. Это занимает приличное время.
Чтобы исправить это можно создать индекс на столбец id, который создаст B-дерево для быстрого поиска по id.

``` sql
create index t1_id_idx on t1 (id);
```
Решение с индексом:
Запрос:
``` sql
explain (analyze, buffers) select name from t1 where id = 50000;
```
Результат:
``` sql 
                                                  QUERY PLAN
---------------------------------------------------------------------------------------------------------------
 Index Scan using t1_id_idx on t1  (cost=0.43..8.45 rows=1 width=30) (actual time=0.130..0.132 rows=1 loops=1)
   Index Cond: (id = 50000)
   Buffers: shared read=4
 Planning:
   Buffers: shared hit=19 read=1
 Planning Time: 1.831 ms
 Execution Time: 0.164 ms
(7 rows)
```

