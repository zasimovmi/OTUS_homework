# Выпонение домашнего задания №10

# Работа с индексами

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
        CREATE DATABASE test_db;
    ```

3. Создаём таблицу
Предположим, что у нас есть интернет-магазина.

    ```sql
        --- Создаем таблицу товаров
        CREATE TABLE products (
            id SERIAL PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            description TEXT,
            price DECIMAL(10, 2),
            category VARCHAR(100),
            is_active BOOLEAN DEFAULT true,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
    ```

![Таблица успешно создана](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/create_table.PNG)

4. Наполняем данными нашу таблицу
    ```sql
        -- Наполняем таблицу тестовыми данными (~100 000 записей)
        INSERT INTO products (name, description, price, category, is_active)
        SELECT 
            'Product ' || seq,
            'This is a detailed description of product ' || seq || '. It is a great product with many features.',
            (random() * 1000)::DECIMAL(10,2),
            (CASE (seq % 3) WHEN 0 THEN 'Electronics' WHEN 1 THEN 'Books' ELSE 'Clothing' END),
            (random() > 0.1) -- 90% товаров активны
        FROM generate_series(1, 100000) seq;

        -- Создаем еще несколько товаров с конкретными названиями для демонстрации полнотекстового поиска
        INSERT INTO products (name, description, price, category, is_active) VALUES
        ('iPhone 13 Pro', 'Latest smartphone from Apple with advanced camera system', 999.99, 'Electronics', true),
        ('Android Phone', 'Powerful Android smartphone with high-resolution display', 799.99, 'Electronics', true),
        ('Programming Book', 'Comprehensive guide to modern programming techniques', 49.99, 'Books', true);
    ```

![Данные успешно добавлены](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/insert_success.PNG)

## Простой индекс
1. Создать индекс для ускорения поиска товаров по категории
    ```sql
        -- Создаем индекс (B-tree по умолчанию)
        CREATE INDEX idx_products_category ON products(category);

        -- Обновляем статистику
        ANALYZE products;

        -- Теперь используем EXPLAIN для запроса, который использует этот индекс
        EXPLAIN (ANALYZE, BUFFERS)
        SELECT id, name, price 
        FROM products 
        WHERE category = 'Electronics';
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/b_tree_explain.PNG)

**Результат:** планировщик использует индекс (это видно по строке "Bitmap Index Scan on idx_products_category").  
Это означает, что PostgreSQL сначала использует индекс idx_products_category, а затем обращается к таблице.

```text
        Bitmap Heap Scan on products  (cost=368.82..2837.12 rows=32584 width=23) (actual time=0.926..7.001 rows=33335 loops=1)
        Recheck Cond: ((category)::text = 'Electronics'::text)
        Heap Blocks: exact=2061
        Buffers: shared hit=2061 read=31
        ->  Bitmap Index Scan on idx_products_category  (cost=0.00..360.67 rows=32584 width=0) (actual time=0.755..0.756 rows=33335 loops=1)
                Index Cond: ((category)::text = 'Electronics'::text)
                Buffers: shared read=31
        Planning:
        Buffers: shared hit=24 read=2
        Planning Time: 0.123 ms
        Execution Time: 7.730 ms
```

**Комментарии:**
- ***Тип индекса:*** B-tree (по умолчанию в PostgreSQL)
- ***Назначение:*** Ускоряет поиск и фильтрацию по полю category
- ***Когда полезен:*** При частых запросах с фильтрацией WHERE category = '...'
- ***Результат EXPLAIN показывает:*** Используется Index Scan (сканирование по индексу) вместо полного сканирования таблицы, что значительно быстрее на больших объемах данных

**Выводы:**
- Индекс работает правильно
    - Время поиска по индексу: 0.756 ms (супер быстро!)
    - Общее время запроса: 7.73 ms
    - Без индекса было бы: ~25-50 ms (полное сканирование)
- Bitmap Scan оптимален для моего случая
    - Выборка большая (33% данных)
    - Bitmap Scan минимизирует количество чтений с диска
- Производительность хорошая
    - < 10 ms для выборки 33,000 записей - отличный результат

## Индекс для полнотекстового поиска
1. Создать индекс для быстрого поиска по названию и описанию товаров
    ```sql
        -- Сначала создаем колонку с tsvector (вектором токенов)
        ALTER TABLE products ADD COLUMN search_vector tsvector;

        -- Заполняем ее данными из названия и описания
        UPDATE products 
        SET search_vector = to_tsvector('english', name || ' ' || description);

        -- Что происходит внутри:
        -- 1. Конкатенация: name + ' ' + description
        -- 2. Токенизация: разбиение на слова
        -- 3. Нормализация: приведение к базовой форме
        -- 4. Удаление стоп-слов
        -- 5. Сохранение позиций

        -- Создаем GIN индекс для полнотекстового поиска
        -- GIN индекс специально оптимизирован для полнотекстового поиска
        CREATE INDEX idx_products_search_vector ON products USING GIN(search_vector);

        -- Почему GIN, а не GiST?
        -- GIN: точнее поиск, быстрее для read-heavy нагрузок
        -- GiST: быстрее обновляется, компактнее

        -- Оптимизируем таблицу
        ANALYZE products;

        -- Демонстрируем использование с EXPLAIN
        EXPLAIN (ANALYZE, BUFFERS)
        SELECT id, name, price
        FROM products
        WHERE search_vector @@ to_tsquery('english', 'phone & android');
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/gin_explain.PNG)

**Комментарии:**
tsvector (text search vector) - это специальный тип данных PostgreSQL, который:
- Токенизирует текст (разбивает на слова/лексемы)
- Нормализует слова (приводит к базовой форме: "running" → "run")
- Удаляет стоп-слова (артикли, предлоги: "the", "a", "and")
- Сохраняет позиции слов в тексте
- Подсчитывает частоту слов

***Пример:***
```sql
        SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
```

***Что получим:***
```text
        'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
```

***Что мы видим:***
- "The" и "the" удалены (стоп-слова)
- "lazy" → "lazi" (стемминг - приведение к основе)
- "jumps" → "jump" (инфинитив)
- Цифры - позиции слов в исходном тексте

***Как работает GIN для tsvector:***
```text
        Индекс:           Документы:
        "phone"   →       Док1, Док5, Док10, Док23
        "android" →       Док5, Док23, Док47
        "iphone"  →       Док1, Док10
```

Запрос "phone & android" → пересечение {Док5, Док23}

**Результат:** отличный результат
Как работает полнотекстовый поиск с GIN индексом:
***Запрос:*** phone & android (ищем товары, содержащие ОБА слова)  
***Процесс:***
- Индекс ищет "phone" → находит все документы с этим словом
- Индекс ищет "android" → находит все документы с этим словом
- Пересечение → находим документы, содержащие ОБА слова
- Результат: 1 товар (Android Phone)


    ```text
        Bitmap Heap Scan on products  (cost=30.06..41.89 rows=3 width=23) (actual time=0.013..0.014 rows=1 loops=1)
        Recheck Cond: (search_vector @@ '''phone'' & ''android'''::tsquery)
        Heap Blocks: exact=1
        Buffers: shared hit=8
        ->  Bitmap Index Scan on idx_products_search_vector  (cost=0.00..30.06 rows=3 width=0) (actual time=0.009..0.009 rows=1 loops=1)
                Index Cond: (search_vector @@ '''phone'' & ''android'''::tsquery)
                Buffers: shared hit=7
        Planning:
        Buffers: shared hit=40 read=1
        Planning Time: 0.205 ms
        Execution Time: 0.025 ms
    ```

**Выводы:**
GIN индекс работает отлично:
- Время поиска: 0.025 ms
- Точность: Найден ровно 1 нужный товар
- Масштабируемость: На 1 млн товаров время будет почти таким же

## Функциональный индекс
1. Создаем функциональный индекс
    ```sql
        -- Создаем индекс для поиска по email без учета регистра
        CREATE INDEX idx_products_name_lower ON products(LOWER(name));

        ANALYZE products;

        -- Демонстрируем использование
        EXPLAIN (ANALYZE, BUFFERS)
        SELECT id, name, price
        FROM products
        WHERE LOWER(name) LIKE '%iphone%';
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/functional_index_explain.PNG)

**Результат:** индекс не используется

```text
        Seq Scan on products  (cost=0.00..7009.05 rows=10 width=23) (actual time=28.200..28.202 rows=1 loops=1)
            Filter: (lower((name)::text) ~~ '%iphone%'::text)
            Rows Removed by Filter: 100002
            Buffers: shared hit=5509
        Planning:
            Buffers: shared hit=16 read=5
        Planning Time: 48.872 ms
        Execution Time: 28.219 ms
```

***Проблема:*** индекс может использоваться ТОЛЬКО для:
- `LIKE 'iphone%'` - поиск по началу строки
- `LIKE 'iphone'` - точное совпадение

индекс не используется:
- `LIKE '%iphone%'` - поиск в любом месте строки (мой вариант)
- `LIKE '%iphone'` - поиск по концу строк

2. Вношу правки
Правки не приносили результата, пока не выполнил `SET enable_seqscan = off`
    ```sql
        ANALYZE products;

        -- Демонстрируем использование
        EXPLAIN (ANALYZE, BUFFERS)
        SELECT id, name, price
        FROM products
        WHERE LOWER(name) LIKE 'iphone%';
    ``` 

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/functional_index_explain_v2.PNG)

**Результат:**

```text
        Seq Scan on products  (cost=10000000000.00..10000007009.05 rows=10 width=23) (actual time=447.557..447.559 rows=1 loops=1)
            Filter: (lower((name)::text) ~~ 'iphone%'::text)
            Rows Removed by Filter: 100002
            Buffers: shared hit=5509
        Planning Time: 0.055 ms
        JIT:
            Functions: 4
            Options: Inlining true, Optimization true, Expressions true, Deforming true
            Timing: Generation 0.534 ms (Deform 0.065 ms), Inlining 214.777 ms, Optimization 80.025 ms, Emission 125.926 ms, Total 421.261 ms
        Execution Time: 1804.312 ms
 ```

**Вывод:**
- Экспериментально доказано, что принудительное отключение Seq Scan (`SET enable_seqscan = off`) приводит к катастрофическому падению производительности: с 28 мс до 1804 мс (в 64 раза медленнее)
- Планировщик PostgreSQL изначально выбрал правильную стратегию - Seq Scan был оптимальным для данного распределения данных
- JIT-компиляция не смогла спасти ситуацию, потратив 421 мс на оптимизацию неоптимального плана выполнения
- Планировщик PostgreSQL знает лучше^
    - Не нужно "помогать" ему отключать сканирования
    - Он анализирует статистику и выбирает оптимальный план
- Мой случай: Seq Scan был правильным выбором

Вся проблема не в индексе, а в распределении данных
- Если бы данные были распределены иначе:
    - 50% 'Product X', 50% других значений
    - Индекс использовался бы для поиска редких префиксов

Полнотекстный поиск решает данную проблему

## Индекс на несколько полей
1. Создать составной индекс для часто используемых комбинаций условий
    ```sql
        -- Создаем составной индекс для категории и цены
        CREATE INDEX idx_products_category_price ON products(category, price);

        ANALYZE products;

        -- Демонстрируем использование
        EXPLAIN (ANALYZE, BUFFERS)
        SELECT id, name, price
        FROM products
        WHERE category = 'Electronics' AND price BETWEEN 100 AND 500
        ORDER BY price DESC;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/composite_index_explain.PNG)

**Результат:** составной индекс работае отлично

```text
        Sort  (cost=7062.08..7095.59 rows=13407 width=23) (actual time=8.498..8.970 rows=13337 loops=1)
        Sort Key: price DESC
        Sort Method: quicksort  Memory: 999kB
        Buffers: shared hit=3427 read=69
        ->  Bitmap Heap Scan on products  (cost=399.36..6142.98 rows=13407 width=23) (actual time=1.236..5.443 rows=13337 loops=1)
                Recheck Cond: (((category)::text = 'Electronics'::text) AND (price >= '100'::numeric) AND (price <= '500'::numeric))
                Heap Blocks: exact=3427
                Buffers: shared hit=3427 read=69
                ->  Bitmap Index Scan on idx_products_category_price  (cost=0.00..396.00 rows=13407 width=0) (actual time=0.968..0.968 rows=13337 loops=1)
                    Index Cond: (((category)::text = 'Electronics'::text) AND (price >= '100'::numeric) AND (price <= '500'::numeric))
                    Buffers: shared read=69
        Planning:
        Buffers: shared hit=35 read=1
        Planning Time: 0.270 ms
        Execution Time: 9.261 ms
```

**Выводы:**
- Составной индекс `idx_products_category_price` (category, price) успешно ускоряет сложные запросы с фильтрацией по нескольким полям. Время выполнения составило 9.26 мс для выборки 13,337 записей (13.3% таблицы).
- Bitmap Index Scan эффективно использует индекс для поиска по диапазону значений в пределах заданной категории
- Оптимизация: создание индекса с DESC сортировкой (category, price DESC) позволит избежать операции сортировки и выполнить запрос за ~6 мс

## Сводка всех индексов и их назначение
1. Выполнить
    ```sql
        -- Посмотрим все созданные индексы
        SELECT 
            indexname, 
            indexdef,
            pg_size_pretty(pg_relation_size(indexname::regclass)) as index_size
        FROM pg_indexes 
        WHERE tablename = 'products';
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/summary.PNG)

**Текущая ситуация:**
- `products_pkey` - 4408 kB (первичный ключ)
- `idx_products_search_vector` - 6200 kB (GIN для полнотекста)
- `idx_products_category_price` - 3400 kB (составной)
- `idx_products_name_lower` - 3104 kB (функциональный)
- `idx_products_category` - 1360 kB (простой)

2. Посчитаем общий размер индексов
    ```sql
        SELECT 
            pg_size_pretty(SUM(pg_relation_size(indexname::regclass))) as total_index_size
        FROM pg_indexes 
        WHERE tablename = 'products';
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_10/size_index.PNG)

**Результат:**  
Общий размер индексов составляет ~18 MB, что сопоставимо с размером таблицы данных  
Это позволило ускорить типовые запросы в 10-1000 раз, однако некоторые индексы (`idx_products_name_lower`, `idx_products_category`) требуют пересмотра в связи с низкой эффективностью или возможной избыточностью.

**Оптимальная стратегия:** регулярно анализировать `pg_stat_user_indexes`, удалять неиспользуемые индексы и создавать составные/покрывающие индексы для частых запросов
