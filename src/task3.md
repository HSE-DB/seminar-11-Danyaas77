## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=12.241..85.151 rows=500584 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=8334
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=11.117..11.117 rows=500584 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.719 ms
    Execution Time: 97.238 ms
    ```
    
    *Объясните результат:*
    Используется Bitmap Index Scan и Bitmap Heap Scan, так как условие возвращает около половины таблицы. Данные разбросаны, поэтому читается много heap-блоков (8334), что даёт заметное время выполнения.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```text
    workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
    [2025-12-19 12:29:13] completed in 912 ms
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on test_cluster  (cost=5584.59..20176.93 rows=500667 width=39) (actual time=9.129..65.660 rows=500584 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=4172
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5459.43 rows=500667 width=0) (actual time=8.752..8.752 rows=500584 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.856 ms
    Execution Time: 76.589 ms
    ```
    
    *Объясните результат:*
    План остался bitmap scan, но после кластеризации записи одной категории лежат ближе. Поэтому в heap читается вдвое меньше блоков (4172), и время выполнения уменьшается.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    После кластеризации Execution Time снизилось примерно с 97.2 ms до 76.6 ms, а число heap-блоков упало с 8334 до 4172. План тот же, но локальность данных улучшила скорость.
