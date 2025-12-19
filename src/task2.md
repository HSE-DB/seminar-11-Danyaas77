## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=1.191..1.194 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=1.041..1.042 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 3.454 ms
    Execution Time: 1.271 ms
    ```
    
    *Объясните результат:*
    Используется GIN индекс по `to_tsvector`, поэтому план строит Bitmap Index Scan и затем Bitmap Heap Scan с перепроверкой условия. Найдена 1 строка, доступ к heap минимальный.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.029..0.029 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.283 ms
     Execution Time: 0.072 ms
     ```
     
     *Объясните результат:*
     Поиск по первичному ключу использует Index Scan по `t_lookup_pk`. Условие точечное и возвращает одну строку, поэтому план простой и быстрый.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.051..0.051 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.488 ms
     Execution Time: 0.090 ms
     ```
     
     *Объясните результат:*
     План аналогичный: Index Scan по первичному ключу. Кластеризация по ключу почти не влияет на точечный поиск, время близкое.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.036..0.036 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.455 ms
     Execution Time: 0.088 ms
     ```
     
     *Объясните результат:*
     Используется индекс по `item_value`, поэтому выполняется Index Scan. Значение отсутствует, результат 0 строк и быстрое завершение без Seq Scan.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```text
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.037..0.037 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.519 ms
     Execution Time: 0.105 ms
     ```
     
     *Объясните результат:*
     Аналогичный план с индексом по `item_value`. Кластеризация по `item_key` не помогает этому поиску, время сопоставимо.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     В обоих случаях используется Index Scan по `item_value`, результат пустой, время почти одинаковое (около 0.09-0.10 мс). Кластеризация по ключу не дает заметного выигрыша для поиска по значению.
