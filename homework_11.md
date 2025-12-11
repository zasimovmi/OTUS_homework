# Выпонение домашнего задания №11

# Секционирование таблицы

## Проделать основные шаги из других домашних заданий
Стандартные шаги:
- Установка VirtualBox на Windows
- Создание виртуальной машины в VirtualBox
- Создать правило для ssh подключения
- Первоначальная настройка ВМ
- Проверить, установлен и запущен ли SSH-сервер
- Установка PosrgreSQl и проверка работы кластера

## Подготовка базы данных и таблиц
1. Зайти в psql
    ```bash
        sudo -u postgres psql
    ```

2. Создаем базу данных для тестов
    ```sql
        CREATE DATABASE demo;
    ```

3. Скачиваем и устанавливаем демонстрационную базу данных авиаперевозки по России
    ```bash
        -- Сохраняем файл локально
        curl https://edu.postgrespro.ru/demo-20250901-3m.sql.gz -o demo-small.sql.gz
    ```

![Файл сохранён успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/curl.PNG)

**Комментарии:** параметр `-o` в curl означает "output" (выходной файл). Без этого параметра содержимое файла просто выведется в терминал.
С этим параметром:
- `-o demo-small.sql.gz` сохранит загруженный файл на диск под именем `demo-small.sql.gz`
- Без него данные "утекут" в терминал и будут потеряны

Если был отказ в выполнении команды `curl`, то её можно установить следующей командой:
    ```bash
        sudo apt update && sudo apt install curl -y
    ```

4. Выполнить действия
    ```bash
        -- Распаковываем
        gunzip demo-small.sql.gz
    ```

![Успешно получили файл](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/gunzip.PNG)

**Комментарии:** теперь у нас есть файл `demo-small.sql`

5. Импортируем
    ```bash
        sudo -u postgres psql demo < demo-small.sql
    ```

**Комментарии:**
- `sudo -u postgres` - запустить от имени пользователя postgres (стандартный пользователь БД в PostgreSQL)
- `psql demo` - запустить клиент psql и подключиться к базе данных demo
- `< demo_small.sql` - перенаправить содержимое файла demo_small.sql в psql (выполнить SQL-скрипт)

## Посмотрим размер таблиц и количество строк
1. Подключаемся к БД
    ```bash
        sudo -u postgres psql demo
    ```

2. Внутри psql устанавливаем search_path
    ```sql
        ALTER DATABASE demo SET search_path TO bookings, pg_catalog;
    ```

**Комментарии:** в демо-базе все таблицы в схеме `bookings`
Эта команда устанавливает, что при подключении к БД `demo` по умолчанию будет искать таблицы:
- В схеме `bookings`
- В системной схеме `pg_catalog`

Узнал я с помощью следующей команды:
```sql
        SELECT nspname AS schema_name
        FROM pg_namespace
        WHERE nspname NOT LIKE 'pg_%'   -- исключаем системные
        AND nspname != 'information_schema'
        ORDER BY nspname;
```

3. Переподключаемся, чтобы изменения вступили в силу
```sql
        \c demo
```

4. Проверяем таблицы
```sql
        \dt
```

![Проверка прошла успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/dt.PNG)

5. Просмотр размера таблиц
```sql
        SELECT
            t.tablename AS table_name,
            c.reltuples::bigint AS num_rows,
            pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
            pg_size_pretty(pg_table_size(c.oid)) AS data_size,
            pg_size_pretty(pg_indexes_size(c.oid)) AS index_size
        FROM
            pg_catalog.pg_tables AS t
            LEFT JOIN pg_catalog.pg_class AS c ON c.relname = t.tablename
        WHERE
            t.schemaname = 'bookings'
        ORDER BY pg_total_relation_size(c.oid) DESC;
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/size.PNG)

**Результат:**
`boarding_passes` - самая большая таблица (2463832 строк, 361 MB), но:
- У неё нет primary key (видно по `num_rows = -1` у `segments`)
- Это таблица-связка между tickets и flights
- Нет явного временного поля для секционирования

`bookings` - 1.29 млн строк, 92 MB - отличный кандидат для секционирования:
- Есть clear primary key (book_ref)
- Есть временное поле book_date (timestamp)
- Это корневая таблица бизнес-процесса

## Создание секционированной таблицы
Секционирование таблицы `bookings` по диапазону дат бронирования (`book_date`).
**Цель:** ускорить запросы, фильтрующие данные по времени, и упростить управление историческими данными

1. Анализ реального распределения данных
```sql
        SELECT 
            EXTRACT(YEAR FROM book_date) as year,
            EXTRACT(MONTH FROM book_date) as month,
            COUNT(*) as bookings_count,
            MIN(book_date) as first_booking,
            MAX(book_date) as last_booking
        FROM bookings.bookings
        GROUP BY EXTRACT(YEAR FROM book_date), EXTRACT(MONTH FROM book_date)
        ORDER BY year, month;
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/data_distribution.PNG)

**Результат:**
- Сентябрь 2025: 446,319 записей
- Октябрь 2025: 434,287 записей
- Ноябрь 2025: 410,680 записей
- Декабрь 2025: всего 1,607 записей (только за 1 декабря)

**Выводы:**
- Данные равномерно распределены по сентябрю-ноябрю (по ~400-450K каждая)
- Декабрь содержит минимальное количество данных
- Идеальная ситуация для месячного секционирования

2. Создаем родительскую секционированную таблицу
```sql
        -- Она определяет структуру, но не хранит данные напрямую.
        CREATE TABLE bookings_partitioned (
            book_ref character(6) NOT NULL,
            book_date timestamptz NOT NULL,
            total_amount numeric(10,2) NOT NULL
        ) PARTITION BY RANGE (book_date);
```

**Ожидаемый результат:** CREATE TABLE
Так как дальше создание таблиц, то ожидаемый результат будет одинаковый

3. Создание секций на основе фактического распределения
Границы секций точно соответствуют границам месяцев
**Важно:** границы должны быть непрерывными и не перекрывающимися

```sql
        -- 1. Секция за сентябрь 2025 (самая большая - 446K записей)
        -- Настраиваем параметры хранения для оптимальной производительности
        CREATE TABLE bookings_partition_2025_09 PARTITION OF bookings_partitioned
            FOR VALUES FROM ('2025-09-01 00:00:00+00') TO ('2025-10-01 00:00:00+00')
            WITH (
                fillfactor = 90,           -- 10% свободного места для обновлений
                autovacuum_enabled = true, -- Включить автоочистку
                toast.autovacuum_enabled = true
            );
        -- Комментарий: fillfactor 90 оптимален для таблиц со средним уровнем обновлений

        -- 2. Секция за октябрь 2025 (434K записей)
        CREATE TABLE bookings_partition_2025_10 PARTITION OF bookings_partitioned
            FOR VALUES FROM ('2025-10-01 00:00:00+00') TO ('2025-11-01 00:00:00+00')
            WITH (
                fillfactor = 90,
                autovacuum_enabled = true,
                toast.autovacuum_enabled = true
            );

        -- 3. Секция за ноябрь 2025 (410K записей)
        CREATE TABLE bookings_partition_2025_11 PARTITION OF bookings_partitioned
            FOR VALUES FROM ('2025-11-01 00:00:00+00') TO ('2025-12-01 00:00:00+00')
            WITH (
                fillfactor = 90,
                autovacuum_enabled = true,
                toast.autovacuum_enabled = true
            );

        -- 4. Секция за декабрь 2025 и все будущие данные
        -- Объединяем декабрь (1.6K) с будущими данными для экономии ресурсов
        CREATE TABLE bookings_partition_2025_12_future PARTITION OF bookings_partitioned
            FOR VALUES FROM ('2025-12-01 00:00:00+00') TO (MAXVALUE)
            WITH (
                fillfactor = 85,           -- Больше свободного места для частых вставок
                autovacuum_vacuum_threshold = 50, -- Чаще запускать vacuum для "горячей" секции
                autovacuum_analyze_threshold = 25
            );
        -- Комментарий: Для "активной" секции настраиваем более агрессивный autovacuum

        -- 5. Секция для любых исторических данных (если появятся)
        -- В текущих данных нет записей до сентября 2025, но создаем на будущее
        CREATE TABLE bookings_partition_historic PARTITION OF bookings_partitioned
            FOR VALUES FROM (MINVALUE) TO ('2025-09-01 00:00:00+00')
            WITH (
                fillfactor = 100,          -- Максимальная плотность для архивных данных
                autovacuum_enabled = false -- Архивные данные не меняются
            );
        -- Комментарий: fillfactor 100 и autovacuum disabled для read-only секции
```

![Таблицы успешно созданы](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/create_tables.PNG)

## Создание оптимизированных индексов
Стратегия индексирования зависит от объема и паттернов запросов

```sql
        -- 1. ОБЯЗАТЕЛЬНЫЕ ИНДЕКСЫ: по ключу секционирования
        -- Ускоряют partition pruning и запросы с фильтрацией по дате
        CREATE INDEX idx_bookings_2025_09_date ON bookings_partition_2025_09 (book_date);
        CREATE INDEX idx_bookings_2025_10_date ON bookings_partition_2025_10 (book_date);
        CREATE INDEX idx_bookings_2025_11_date ON bookings_partition_2025_11 (book_date);
        CREATE INDEX idx_bookings_2025_12_future_date ON bookings_partition_2025_12_future (book_date);
        CREATE INDEX idx_bookings_historic_date ON bookings_partition_historic (book_date);

        -- 2. БИЗНЕС-ИНДЕКСЫ: по book_ref для JOIN операций
        -- Учитывая частые JOIN с tickets, эти индексы важны
        CREATE INDEX idx_bookings_2025_09_ref ON bookings_partition_2025_09 (book_ref);
        CREATE INDEX idx_bookings_2025_10_ref ON bookings_partition_2025_10 (book_ref);
        CREATE INDEX idx_bookings_2025_11_ref ON bookings_partition_2025_11 (book_ref);
        CREATE INDEX idx_bookings_2025_12_future_ref ON bookings_partition_2025_12_future (book_ref);
        CREATE INDEX idx_bookings_historic_ref ON bookings_partition_historic (book_ref);

        -- 3. ОПТИОНАЛЬНЫЕ ИНДЕКСЫ: для аналитических запросов
        -- Если часто выполняются запросы "выручка по дням/неделям"
        CREATE INDEX idx_bookings_2025_09_date_amount ON bookings_partition_2025_09 (book_date, total_amount);
        CREATE INDEX idx_bookings_2025_10_date_amount ON bookings_partition_2025_10 (book_date, total_amount);
        CREATE INDEX idx_bookings_2025_11_date_amount ON bookings_partition_2025_11 (book_date, total_amount);
```

![Индексы успешно созданы](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/create_index_partition.PNG)

## Проверка структуры секционирования
1. Убеждаемся, что все секции созданы правильно
```sql
        SELECT 
            inhrelid::regclass AS partition_name,
            pg_get_expr(relpartbound, inhparent::regclass) AS partition_bound,
            pg_relation_size(inhrelid) AS size_bytes,
            pg_size_pretty(pg_relation_size(inhrelid)) AS size_pretty
        FROM pg_inherits 
        JOIN pg_class ON inhrelid = pg_class.oid
        WHERE inhparent = 'bookings_partitioned'::regclass
        ORDER BY partition_bound NULLS FIRST;
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/checking_sections.PNG)

**Результаты:**
Структура секционирования создана правильно:
- 5 секций создано
- Границы секций соответствуют плану:
    - historic → до 1 сентября 2025
    - 2025_09 → сентябрь 2025
    - 2025_10 → октябрь 2025
    - 2025_11 → ноябрь 2025
    - 2025_12_future → декабрь 2025 и далее

Критическая проблема: данные отсутствуют
- Все секции показывают 0 bytes (кроме historic - 8 bytes, это метаданные)
- Данные не перенесены из исходной таблицы

На данном этапе идёт всё хорошо, потому что миграции данных пока ещё не было.

## Миграция данных с расширенной валидацией
1. Предварительный анализ распределения
```sql
        WITH date_analysis AS (
            SELECT 
                book_date,
                CASE 
                    WHEN book_date < '2025-09-01' THEN 'historic'
                    WHEN book_date < '2025-10-01' THEN '2025-09'
                    WHEN book_date < '2025-11-01' THEN '2025-10'
                    WHEN book_date < '2025-12-01' THEN '2025-11'
                    ELSE '2025-12_future'
                END as expected_partition,
                COUNT(*) as count_per_day
            FROM bookings.bookings
            GROUP BY book_date, expected_partition
        )
        SELECT 
            expected_partition,
            COUNT(DISTINCT DATE(book_date)) as days_covered,
            SUM(count_per_day) as total_records,
            MIN(book_date) as earliest_date,
            MAX(book_date) as latest_date
        FROM date_analysis
        GROUP BY expected_partition
        ORDER BY expected_partition;
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/preliminary_distribution_analysis.PNG)

**Выводы:**
- Подтверждение правильности границ секций:
    - Данные идеально соответствуют моим секциям
    - Нет "потерянных" записей между секциями
    - Нет перекрытий (одна запись не может попасть в две секции)
- Равномерность распределения:
    - Сентябрь: 446K (~14.9K в день)
    - Октябрь: 434K (~14.0K в день)
    - Ноябрь: 410K (~13.7K в день)
    - ***Вывод:*** Нагрузка равномерная, ~14K бронирований в день
- Качество данных:
    - Данные охватывают полные месяцы (30-31 день)
    - Нет пропусков дней
    - Данные начинаются 1 сентября и заканчиваются 1 декабря

2. Миграция данных с контролем времени
```sql
        -- Замеряем время миграции для отчета
        \timing on

        -- Миграция основных данных
        INSERT INTO bookings_partitioned
        SELECT * FROM bookings.bookings;

        \timing off
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/time.PNG)

**Результат:** миграция успешно выполнена
- Все 1.29 млн записей перенесены
- Время миграции - 8.5 сек (приемлемо для production)
- Данные автоматически распределились по секциям

## Выполнить сравнение производительности
1. Тест 1: простой запрос за месяц
```sql
        -- ДО (на bookings.bookings)
        EXPLAIN (ANALYZE, BUFFERS, TIMING)
        SELECT COUNT(*), SUM(total_amount)
        FROM bookings.bookings
        WHERE book_date >= '2025-10-01' 
        AND book_date < '2025-11-01';
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/explain_before_1.PNG)

**Результат:**
```text
 Finalize Aggregate  (cost=18228.40..18228.41 rows=1 width=40) (actual time=66.674..69.273 rows=1 loops=1)
   Buffers: shared hit=351 read=7889
   ->  Gather  (cost=18228.17..18228.38 rows=2 width=40) (actual time=66.033..69.264 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=351 read=7889
         ->  Partial Aggregate  (cost=17228.17..17228.18 rows=1 width=40) (actual time=45.490..45.490 rows=1 loops=3)
               Buffers: shared hit=351 read=7889
               ->  Parallel Seq Scan on bookings  (cost=0.00..16320.58 rows=181518 width=6) (actual time=0.026..34.010 rows=144762 loops=3)
                     Filter: ((book_date >= '2025-10-01 00:00:00+03'::timestamp with time zone) AND (book_date < '2025-11-01 00:00:00+03'::timestamp with time zone))
                     Rows Removed by Filter: 286202
                     Buffers: shared hit=351 read=7889
 Planning Time: 0.053 ms
 Execution Time: 69.294 ms
```

2. Тест 1: простой запрос за месяц
```sql
-- ПОСЛЕ (на bookings_partitioned)  
EXPLAIN (ANALYZE, BUFFERS, TIMING)
SELECT COUNT(*), SUM(total_amount)
FROM bookings_partitioned
WHERE book_date >= '2025-10-01' 
  AND book_date < '2025-11-01';
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/explain_after_1.PNG)

**Результат:**
```text
 Finalize Aggregate  (cost=8666.99..8667.00 rows=1 width=40) (actual time=62.729..65.242 rows=1 loops=1)
   Buffers: shared hit=715 read=2379
   ->  Gather  (cost=8666.76..8666.97 rows=2 width=40) (actual time=59.882..65.232 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=715 read=2379
         ->  Partial Aggregate  (cost=7666.76..7666.77 rows=1 width=40) (actual time=35.593..35.595 rows=1 loops=3)
               Buffers: shared hit=715 read=2379
               ->  Parallel Append  (cost=0.00..6762.03 rows=180946 width=6) (actual time=0.024..25.806 rows=144762 loops=3)
                     Buffers: shared hit=715 read=2379
                     ->  Parallel Index Only Scan using idx_bookings_2025_09_date_amount on bookings_partition_2025_09 bookings_partitioned_1  (cost=0.42..63.80 rows=1014 width=6) (actual time=0.040..0.212 rows=1745 loops=1)
                           Index Cond: ((book_date >= '2025-10-01 00:00:00+03'::timestamp with time zone) AND (book_date < '2025-11-01 00:00:00+03'::timestamp with time zone))
                           Heap Fetches: 0
                           Buffers: shared hit=4 read=10
                     ->  Parallel Seq Scan on bookings_partition_2025_10 bookings_partitioned_2  (cost=0.00..5793.49 rows=180228 width=6) (actual time=0.015..17.506 rows=144181 loops=3)
                           Filter: ((book_date >= '2025-10-01 00:00:00+03'::timestamp with time zone) AND (book_date < '2025-11-01 00:00:00+03'::timestamp with time zone))
                           Rows Removed by Filter: 539
                           Buffers: shared hit=711 read=2369
 Planning:
   Buffers: shared hit=33 read=7
 Planning Time: 0.293 ms
 Execution Time: 65.280 ms
```

**Выводы:**
Тестирование производительности запроса за октябрь 2025 года показало:
- Сокращение операций ввода-вывода: Количество прочитанных буферов уменьшилось с 7,889 до 2,379 (на 70%)
- Механизм partition pruning: Частично работает, но требуется корректировка условий фильтрации с учетом временных зон
- Параллельное выполнение: В обоих случаях PostgreSQL эффективно использует parallel workers

3. Тест 2: JOIN запрос
```sql
        -- ДО
        EXPLAIN (ANALYZE, BUFFERS, TIMING)
        SELECT b.book_ref, t.passenger_name, b.book_date
        FROM bookings.bookings b
        JOIN bookings.tickets t ON b.book_ref = t.book_ref
        WHERE b.book_date >= '2025-09-15' 
        AND b.book_date < '2025-09-20'
        LIMIT 100;
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/explain_before_2.PNG)

**Результат:**
```text
 Limit  (cost=0.43..875.72 rows=100 width=28) (actual time=510.697..1120.890 rows=100 loops=1)
   Buffers: shared hit=1418743 read=8324
   ->  Nested Loop  (cost=0.43..1445958.92 rows=165197 width=28) (actual time=510.696..1120.876 rows=100 loops=1)
         Buffers: shared hit=1418743 read=8324
         ->  Seq Scan on tickets t  (cost=0.00..60484.50 rows=2974050 width=20) (actual time=0.035..20.469 rows=355847 loops=1)
               Buffers: shared read=3679
         ->  Index Scan using bookings_pkey on bookings b  (cost=0.43..0.47 rows=1 width=15) (actual time=0.003..0.003 rows=0 loops=355847)
               Index Cond: (book_ref = t.book_ref)
               Filter: ((book_date >= '2025-09-15 00:00:00+03'::timestamp with time zone) AND (book_date < '2025-09-20 00:00:00+03'::timestamp with time zone))
               Rows Removed by Filter: 1
               Buffers: shared hit=1418743 read=4645
 Planning:
   Buffers: shared hit=50 read=10
 Planning Time: 19.416 ms
 Execution Time: 1120.963 ms
```

4. Тест 2: JOIN запрос
```sql
        -- ПОСЛЕ
        EXPLAIN (ANALYZE, BUFFERS, TIMING)
        SELECT b.book_ref, t.passenger_name, b.book_date
        FROM bookings_partitioned b
        JOIN bookings.tickets t ON b.book_ref = t.book_ref
        WHERE b.book_date >= '2025-09-15' 
        AND b.book_date < '2025-09-20'
        LIMIT 100;
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_11/explain_after_2.PNG)

**Результат:**
```text
 Limit  (cost=0.42..633.44 rows=100 width=28) (actual time=7.727..81.479 rows=100 loops=1)
   Buffers: shared hit=109410 read=2036
   ->  Nested Loop  (cost=0.42..1432885.25 rows=226358 width=28) (actual time=7.726..81.467 rows=100 loops=1)
         Buffers: shared hit=109410 read=2036
         ->  Seq Scan on tickets t  (cost=0.00..60484.50 rows=2974050 width=20) (actual time=0.007..1.663 rows=27788 loops=1)
               Buffers: shared hit=7 read=287
         ->  Index Scan using idx_bookings_2025_09_ref on bookings_partition_2025_09 b  (cost=0.42..0.45 rows=1 width=15) (actual time=0.003..0.003 rows=0 loops=27788)
               Index Cond: (book_ref = t.book_ref)
               Filter: ((book_date >= '2025-09-15 00:00:00+03'::timestamp with time zone) AND (book_date < '2025-09-20 00:00:00+03'::timestamp with time zone))
               Rows Removed by Filter: 1
               Buffers: shared hit=109403 read=1749
 Planning:
   Buffers: shared hit=7
 Planning Time: 0.163 ms
 Execution Time: 81.520 ms
```


Метрика	        До секционирования	После секционирования	Ускорение
Execution Time	1120.963 ms	        81.520 ms	            13.7x быстрее
Buffers Read	8324	            2036	                4.1x меньше I/O
Buffers Hit	    1,418,743	        109,410	                13x меньше операций
Planning Time	19.416 ms	        0.163 ms	            119x быстрее планирование
Rows Processed	355,847	            27,788	                12.8x меньше строк

**Выводы:**
JOIN-запросы получают НАИБОЛЬШИЙ выигрыш от секционирования. 
Тестирование запроса с соединением таблиц bookings и tickets показало:
- Ускорение выполнения в 13.7 раз (1121 мс → 82 мс)
- Сокращение операций ввода-вывода в 4 раза
- Ускорение планирования запроса в 119 раз
- PostgreSQL эффективно применяет partition-wise join, исключая нерелевантные секции из обработки

Секционирование позволяет PostgreSQL оптимизировать порядок выполнения JOIN операций.
Вместо соединения со всей таблицей bookings (1.29 млн записей), оптимизатор понимает, что нужна только одна секция (сентябрь 2025, 446 тыс. записей), что кардинально сокращает объем обрабатываемых данных.