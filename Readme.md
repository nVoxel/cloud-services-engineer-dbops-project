# dbops-project
Исходный репозиторий для выполнения проекта дисциплины "DBOps"

## Создание БД `store`

```sh
psql "host=<host> port=<port> dbname=store_default user=<user>"
```

```postgresql
CREATE DATABASE store;
\c store
```

## Создание нового пользователя
```postgresql
CREATE USER test_user WITH PASSWORD 'test_password';
GRANT ALL PRIVILEGES ON DATABASE store TO test_user;
GRANT USAGE ON SCHEMA public TO test_user;
GRANT CREATE ON SCHEMA public TO test_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO test_user;
```

## Время выполнения перед добавлением индексов

```postgresql
SELECT o.date_created, SUM(op.quantity)
FROM orders AS o
JOIN order_product AS op ON o.id = op.order_id
WHERE o.status = 'shipped' AND o.date_created > NOW() - INTERVAL '7 DAY'
GROUP BY o.date_created;
```

```
 date_created |  sum  
--------------+-------
 2026-02-25   | 10974
 2026-02-26   | 11219
 2026-02-27   | 11599
 2026-02-28   | 11475
 2026-03-01   | 12290
 2026-03-02   | 11322
 2026-03-03   |  9310
(7 rows)

Time: 38.098 ms
```

```
                                                                          QUERY PLAN                                                                           
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=8828.52..8840.35 rows=91 width=12) (actual time=43.105..46.289 rows=7 loops=1)
   Group Key: o.date_created
   ->  Gather Merge  (cost=8828.52..8838.98 rows=91 width=12) (actual time=43.098..46.278 rows=14 loops=1)
         Workers Planned: 1
         Workers Launched: 1
         ->  Sort  (cost=7828.51..7828.74 rows=91 width=12) (actual time=41.237..41.240 rows=7 loops=2)
               Sort Key: o.date_created
               Sort Method: quicksort  Memory: 25kB
               Worker 0:  Sort Method: quicksort  Memory: 25kB
               ->  Partial HashAggregate  (cost=7824.64..7825.55 rows=91 width=12) (actual time=41.221..41.225 rows=7 loops=2)
                     Group Key: o.date_created
                     Batches: 1  Memory Usage: 24kB
                     Worker 0:  Batches: 1  Memory Usage: 24kB
                     ->  Parallel Hash Join  (cost=3663.65..7802.61 rows=4406 width=8) (actual time=15.115..40.627 rows=3690 loops=2)
                           Hash Cond: (op.order_id = o.id)
                           ->  Parallel Seq Scan on order_product op  (cost=0.00..3675.71 rows=176471 width=12) (actual time=0.004..9.934 rows=150000 loops=2)
                           ->  Parallel Hash  (cost=3626.94..3626.94 rows=2937 width=12) (actual time=14.873..14.874 rows=2484 loops=2)
                                 Buckets: 8192  Batches: 1  Memory Usage: 352kB
                                 ->  Parallel Seq Scan on orders o  (cost=0.00..3626.94 rows=2937 width=12) (actual time=0.017..14.321 rows=2484 loops=2)
                                       Filter: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
                                       Rows Removed by Filter: 97516
 Planning Time: 0.204 ms
 Execution Time: 46.345 ms

```

## Время выполнения после добавления индексов

```postgresql
SELECT o.date_created, SUM(op.quantity)
FROM orders AS o
JOIN order_product AS op ON o.id = op.order_id
WHERE o.status = 'shipped' AND o.date_created > NOW() - INTERVAL '7 DAY'
GROUP BY o.date_created;
```

```
 date_created |  sum  
--------------+-------
 2026-02-25   | 10974
 2026-02-26   | 11219
 2026-02-27   | 11599
 2026-02-28   | 11475
 2026-03-01   | 12290
 2026-03-02   | 11322
 2026-03-03   |  9310
(7 rows)

Time: 24.169 ms
```

```
                                                                                    QUERY PLAN                                                                                    
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=6605.92..6617.75 rows=91 width=12) (actual time=31.971..35.362 rows=7 loops=1)
   Group Key: o.date_created
   ->  Gather Merge  (cost=6605.92..6616.39 rows=91 width=12) (actual time=31.964..35.353 rows=14 loops=1)
         Workers Planned: 1
         Workers Launched: 1
         ->  Sort  (cost=5605.91..5606.14 rows=91 width=12) (actual time=29.627..29.631 rows=7 loops=2)
               Sort Key: o.date_created
               Sort Method: quicksort  Memory: 25kB
               Worker 0:  Sort Method: quicksort  Memory: 25kB
               ->  Partial HashAggregate  (cost=5602.04..5602.95 rows=91 width=12) (actual time=29.598..29.603 rows=7 loops=2)
                     Group Key: o.date_created
                     Batches: 1  Memory Usage: 24kB
                     Worker 0:  Batches: 1  Memory Usage: 24kB
                     ->  Parallel Hash Join  (cost=1441.06..5580.01 rows=4406 width=8) (actual time=2.510..28.938 rows=3690 loops=2)
                           Hash Cond: (op.order_id = o.id)
                           ->  Parallel Seq Scan on order_product op  (cost=0.00..3675.71 rows=176471 width=12) (actual time=0.006..10.153 rows=150000 loops=2)
                           ->  Parallel Hash  (cost=1404.34..1404.34 rows=2937 width=12) (actual time=2.264..2.266 rows=2484 loops=2)
                                 Buckets: 8192  Batches: 1  Memory Usage: 352kB
                                 ->  Parallel Bitmap Heap Scan on orders o  (cost=71.60..1404.34 rows=2937 width=12) (actual time=0.295..1.795 rows=2484 loops=2)
                                       Recheck Cond: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
                                       Heap Blocks: exact=1209
                                       ->  Bitmap Index Scan on idx_orders_status_date_created  (cost=0.00..70.36 rows=4993 width=0) (actual time=0.372..0.373 rows=4968 loops=1)
                                             Index Cond: (((status)::text = 'shipped'::text) AND (date_created > (now() - '7 days'::interval)))
 Planning Time: 0.346 ms
 Execution Time: 35.431 ms
```

## Вывод

После того, как мы добавили индексы, время выполненияз запроса сократилось в ~1.5 раза, с 38.098 ms до 24.169 ms
