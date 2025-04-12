# Ускорить запрос "max + left join", добиться времени выполнения < 10ms

Изначальный результат быстродействия запроса:
``` sql 
explain (analyze, buffers) select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
``` sql 
                                                           QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=385682.77..385682.78 rows=1 width=32) (actual time=14232.707..14232.711 rows=1 loops=1)
   Buffers: shared hit=4673 read=110591, temp read=17849 written=17849
   ->  Hash Left Join  (cost=218335.76..373182.96 rows=4999923 width=9) (actual time=3224.838..9686.997 rows=5000000 loops=1)
         Hash Cond: (t2.t_id = t1.id)
         Buffers: shared hit=4673 read=110591, temp read=17849 written=17849
         ->  Seq Scan on t2  (cost=0.00..81871.23 rows=4999923 width=13) (actual time=0.074..1409.722 rows=5000000 loops=1)
               Buffers: shared hit=2112 read=29760
         ->  Hash  (cost=208392.00..208392.00 rows=606061 width=4) (actual time=3222.388..3222.390 rows=624041 loops=1)
               Buckets: 262144  Batches: 4  Memory Usage: 7554kB
               Buffers: shared hit=2561 read=80831, temp written=1368
               ->  Seq Scan on t1  (cost=0.00..208392.00 rows=606061 width=4) (actual time=0.038..2976.113 rows=624041 loops=1)
                     Filter: (name ~~ 'a%'::text)
                     Rows Removed by Filter: 9375959
                     Buffers: shared hit=2561 read=80831
 Planning:
   Buffers: shared hit=8
 Planning Time: 0.270 ms
 Execution Time: 14234.542 ms
(18 rows)
```

Создадим индекс, который позволит быстро найти максимальное значение в столбце day без полного сканирования таблицы.

``` sql
create index idx_t2_day on t2(day);
```

Так как left join не влияет на результат max(t2.day) (все строки t2 сохраняются, а t2.day не зависит от t1), запрос можно упростить, удалив ненужный join.

Итоговый запрос:
``` sql 
explain (analyze, buffers) select max(day) from t2;
                                                                      QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------
 Result  (cost=0.45..0.46 rows=1 width=32) (actual time=0.034..0.035 rows=1 loops=1)
   Buffers: shared hit=4
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.43..0.45 rows=1 width=9) (actual time=0.031..0.031 rows=1 loops=1)
           Buffers: shared hit=4
           ->  Index Only Scan Backward using idx_t2_day on t2  (cost=0.43..104780.44 rows=5000000 width=9) (actual time=0.029..0.029 rows=1 loops=1)
                 Index Cond: (day IS NOT NULL)
                 Heap Fetches: 0
                 Buffers: shared hit=4
 Planning Time: 0.172 ms
 Execution Time: 0.058 ms
(11 rows)
```