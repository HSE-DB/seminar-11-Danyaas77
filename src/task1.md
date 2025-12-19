# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```text
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.034..0.035 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.024..0.024 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.720 ms
   Execution Time: 0.168 ms
   ```
   
   *Объясните результат:*
   Используется BRIN индекс и bitmap heap scan, но индекс даёт только кандидатов, поэтому требуется recheck на куче. Так как `category` в данных не содержит `NULL`, после перепроверки получаем 0 строк.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```text
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=11.474..11.475 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1225
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.060..0.060 rows=12250 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.619 ms
   Execution Time: 11.573 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   План использует bitmap scan по BRIN индексу `category`, но он «грубый» (lossy), поэтому перепроверяются блоки в heap и отбрасываются все строки. Условие по `author` применено как фильтр, индекс по автору не выбран планировщиком (слишком дорогая/малополезная комбинация), поэтому результата нет.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```text
   Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=25.629..25.630 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=25.587..25.588 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.003..8.635 rows=150000 loops=1)
   Planning Time: 0.578 ms
   Execution Time: 25.774 ms
   ```
   
   *Объясните результат:*
   План делает полный Seq Scan, затем HashAggregate для `DISTINCT`, и отдельно сортирует из-за `ORDER BY`. Индекс не помогает, так как нужно просканировать все строки и агрегировать значения.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```text
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=11.107..11.108 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=11.104..11.104 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 0.991 ms
   Execution Time: 11.200 ms
   ```
   
   *Объясните результат:*
   Используется последовательное сканирование, потому что условие `LIKE 'S%'` не селективно для BRIN и значения авторов не отсортированы; в данных нет строк на `S`, поэтому все 150000 строк отфильтрованы.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE LOWER(title) LIKE 'o%';
   ```
   
   *План выполнения:*
   ```text
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=22.430..22.431 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=22.426..22.427 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 1.847 ms
   Execution Time: 22.510 ms
   ```
   
   *Объясните результат:*
   План всё равно выбирает Seq Scan. Обычный btree-индекс по `LOWER(title)` не используется для `LIKE 'o%'` при стандартной локали; для индексируемого префиксного поиска нужен `text_pattern_ops`/коллация `C` или trigram-индекс. Поэтому происходит полное сканирование и находится 1 строка.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
```sql
CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
USING brin(category, author);
```

14. Повторите запрос из шага 7:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```text
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=1.016..1.016 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8847
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.061..0.061 rows=730 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.613 ms
   Execution Time: 1.109 ms
   ```
   
   *Объясните результат:*
   Композитный BRIN индекс учитывает оба условия, поэтому bitmap меньше и лосси-блоков значительно меньше. В итоге меньше перепроверок и быстрее выполнение (~1.1 мс против 11.6 мс).
