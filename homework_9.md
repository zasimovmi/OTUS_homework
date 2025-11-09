# Выпонение домашнего задания №9

## Основное домашнее задание

### Проделать основные шаги из других домашних заданий
Стандартные шаги:
- Установка VirtualBox на Windows
- Создание виртуальной машины в VirtualBox
- Создать правило для ssh подключения
- Первоначальная настройка ВМ
- Проверить, установлен и запущен ли SSH-сервер
- Установка PosrgreSQl и проверка работы кластера

### Подготовка базы данных и таблиц
1. Зайти в psql
    ```bash
        sudo -u postgres psql
    ```

2. Создаем базу данных для тестов
    ```sql
        CREATE DATABASE test_db;
    ```

3. Создаём таблицы
Создадим базу данных для интернет-магазина.
У нас будут таблицы:
- customers (покупатели),
- products (товары),
- orders (заказы),
- order_items (позиция товара).

    ```sql
        -- Таблица покупателей
        -- Хранение информации о зарегистрированных пользователях магазина
        CREATE TABLE customers (
        customer_id SERIAL PRIMARY KEY,                                            -- Уникальный идентификатор покупателя (первичный ключ)
        name VARCHAR(100) NOT NULL,                                                -- Полное имя покупателя
        email VARCHAR(100) UNIQUE,                                                 -- Электронная почта покупателя
        registration_date DATE DEFAULT CURRENT_DATE,                               -- Дата регистрации пользователя
        status VARCHAR(20) CHECK (status IN ('active', 'inactive', 'premium'))     -- Статус клиента: active (активный), inactive (неактивный), premium (премиум)
        );

        -- Таблица товаров
        -- Каталог товаров магазина с ценами и остатками
        CREATE TABLE products (
        product_id SERIAL PRIMARY KEY,                                             -- Уникальный идентификатор товара (артикул)
        product_name VARCHAR(200) NOT NULL,                                        -- Наименование товара
        category VARCHAR(100),                                                     -- Категория товара: laptop (ноутбук), smartphone (телефон), headphones (наушники) и т.д.
        price DECIMAL(10, 2) CHECK (price > 0),                                    -- Розничная цена товара в рублях
        stock_quantity INT DEFAULT 0                                               -- Текущий остаток на складе
        );

        -- Таблица заказов
        -- Информация о заказах и их статусах
        CREATE TABLE orders (
        order_id SERIAL PRIMARY KEY,                                               -- Уникальный номер заказа
        order_date DATE NOT NULL,                                                  -- Дата оформления заказа
        customer_id INT REFERENCES customers(customer_id),                         -- Идентификатор покупателя (связь с таблицей customers)
        status VARCHAR(20) DEFAULT 'processing'                                    -- Статус заказа: processing (в обработке), completed (выполнен), cancelled (отменен)
        );

        -- Таблица позиции заказа
        -- Детализация состава заказов (что и в каком количестве заказано)
        CREATE TABLE order_items (
        order_item_id SERIAL PRIMARY KEY,                                          -- Уникальный идентификатор позиции в заказе
        order_id INT REFERENCES orders(order_id),                                  -- Ссылка на заказ (связь с таблицей orders)
        product_id INT REFERENCES products(product_id),                            -- Ссылка на товар (связь с таблицей products)
        quantity INT CHECK (quantity > 0),                                         -- Количество товара в заказе для расчета итоговой суммы
        unit_price DECIMAL(10, 2)                                                  -- Цена товара в момент заказа (историческая стоимость)
        );
    ```

![Таблицы успешно созданы](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/create_tables.PNG)

4. Наполняем данными наши таблицы
    ```sql
        -- Покупатели с разной активностью
        INSERT INTO customers (name, email, registration_date, status) VALUES
        ('Иван Иванов', 'ivan@tech.com', '2024-01-15', 'premium'),
        ('Петр Петров', 'petr@mail.com', '2024-02-20', 'active'),
        ('Мария Сидорова', 'maria@store.com', '2024-03-10', 'active'),
        ('Анна Козлова', 'anna@domain.com', '2024-04-05', 'inactive'),
        ('Сергей Смирнов', NULL, '2024-05-12', 'active'); -- Новый активный пользователь без email

        -- Товары по категориям
        INSERT INTO products (product_name, category, price, stock_quantity) VALUES
        ('MacBook Pro 16"', 'laptop', 250000.00, 5),
        ('iPhone 15 Pro', 'smartphone', 120000.00, 15),
        ('Samsung Galaxy S24', 'smartphone', 80000.00, 10),
        ('Dell XPS 13', 'laptop', 150000.00, 3),
        ('AirPods Pro', 'headphones', 25000.00, 20),
        ('Sony WH-1000XM5', 'headphones', 30000.00, 8);

        -- Заказы с разными статусами
        INSERT INTO orders (order_date, customer_id, status) VALUES
        ('2024-10-01', 1, 'completed'),
        ('2024-10-02', 2, 'completed'),
        ('2024-10-03', 1, 'completed'),
        ('2024-10-05', 3, 'processing'),
        ('2024-10-06', 5, 'completed'); -- Заказ от нового клиента

        -- Детали заказов (корзины)
        INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
        (1, 1, 1, 250000.00), -- Иван купил MacBook
        (1, 5, 1, 25000.00),  -- и AirPods
        (2, 2, 1, 120000.00), -- Петр купил iPhone
        (3, 4, 1, 150000.00), -- Иван купил Dell
        (4, 3, 2, 80000.00),  -- Мария заказала 2 Samsung
        (5, 6, 1, 30000.00);  -- Сергей купил Sony
    ```

![Данные успешно добавлены](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/insert_success.PNG)

### Написание запросов с соединениями
1. **INNER JOIN**: Анализ завершенных заказов с детализацией
    ```sql
        -- Комментарий: Бизнес-анализ завершенных заказов. INNER JOIN гарантирует, что мы видим только успешные транзакции.
        -- Цель: Анализ покупательской активности и выручки по категориям товаров.
        SELECT 
            c.name AS "Клиент",
            c.status AS "Статус клиента",
            o.order_date AS "Дата заказа",
            p.product_name AS "Товар",
            p.category AS "Категория",
            oi.quantity AS "Кол-во",
            oi.unit_price AS "Цена",
            (oi.quantity * oi.unit_price) AS "Сумма"
        FROM orders o
        INNER JOIN customers c ON o.customer_id = c.customer_id
        INNER JOIN order_items oi ON o.order_id = oi.order_id
        INNER JOIN products p ON oi.product_id = p.product_id
        WHERE o.status = 'completed'
        ORDER BY "Сумма" DESC;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/inner_join.PNG)

**Результат:** видим, что премиум-клиенты делают самые крупные покупки, а электроника — самая доходная категория.

2. **LEFT JOIN**: Полный "портрет" клиентской базы с метриками
    ```sql
        -- Комментарий: LEFT JOIN показывает полную картину по клиентам, включая тех, кто еще не совершал покупок.
        -- Цель: Расчет LTV (Lifetime Value или иначе говоря - пожизненная ценность клиента) и выявление неактивных клиентов.
        SELECT 
            c.customer_id AS "ID",
            c.name AS "Клиент",
            c.registration_date AS "Регистрация",
            c.status AS "Статус",
            COUNT(o.order_id) AS "Всего заказов",
            COUNT(DISTINCT oi.product_id) AS "Уникальных товаров",
            COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS "Общий LTV",
            CASE 
                WHEN COUNT(o.order_id) = 0 THEN 'Нет покупок'
                WHEN COUNT(o.order_id) = 1 THEN 'Однократный'
                ELSE 'Постоянный'
            END AS "Тип клиента"
        FROM customers c
        LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
        LEFT JOIN order_items oi ON o.order_id = oi.order_id
        GROUP BY c.customer_id, c.name, c.registration_date, c.status
        ORDER BY "Общий LTV" DESC;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/left_join.PNG)

**Результат:** четкая сегментация клиентов по покупательной активности, видна ценность каждого клиента.

3. **CROSS JOIN**: План кросс-продаж и рекомендаций
    ```sql
        -- Комментарий: CROSS JOIN создает матрицу "всех клиентов vs все товары" для анализа потенциала кросс-продаж.
        -- Цель: Выявление персональных рекомендаций товаров для каждого клиента.
        SELECT 
            c.name AS "Клиент",
            p.product_name AS "Рекомендуемый товар",
            p.category AS "Категория",
            p.price AS "Цена",
            -- Бизнес-логика рекомендаций:
            CASE 
                WHEN p.category = 'laptop' AND c.status = 'premium' THEN 'ТОП-рекомендация'
                WHEN p.price < 50000 THEN 'Бюджетный вариант'
                ELSE 'Стандартная рекомендация'
            END AS "Приоритет"
        FROM customers c
        CROSS JOIN products p
        -- Исключаем товары, которые уже есть на складе или клиент уже покупал:
        WHERE p.stock_quantity > 0 
        AND p.product_id NOT IN (
            SELECT oi.product_id 
            FROM orders o 
            JOIN order_items oi ON o.order_id = oi.order_id 
            WHERE o.customer_id = c.customer_id AND o.status = 'completed'
        )
        ORDER BY c.name, "Приоритет" DESC;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/cross_join.PNG)

**Результат:** автоматическая система рекомендаций на основе статуса клиента и истории покупок.

4. **FULL JOIN**: Аудит всей системы заказов
    ```sql
        -- Комментарий: FULL JOIN выявляет аномалии данных: клиенты без заказов И заказы без клиентов (если есть).
        -- Цель: Полный аудит данных и выявление проблем в бизнес-процессах.
        SELECT 
            COALESCE(c.name, '--- КЛИЕНТ УДАЛЕН ---') AS "Клиент",
            COALESCE(c.status, 'N/A') AS "Статус клиента", 
            COALESCE(o.order_id::TEXT, '--- НЕТ ЗАКАЗОВ ---') AS "Номер заказа",
            o.order_date AS "Дата заказа",
            o.status AS "Статус заказа",
            COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS "Сумма заказа"
        FROM customers c
        FULL JOIN orders o ON c.customer_id = o.customer_id
        LEFT JOIN order_items oi ON o.order_id = oi.order_id
        GROUP BY c.customer_id, c.name, c.status, o.order_id, o.order_date, o.status
        ORDER BY "Сумма заказа" DESC NULLS LAST;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/full_join.PNG)

**Результат:** выявление "зависших" заказов, клиентов без активности, проблем целостности данных.

5. **Комбинированный анализ**: Дашборд эффективности категорий
    ```sql
        -- Комментарий: Многоуровневый анализ с разными типами JOIN для построения дашборда эффективности.
        -- Цель: Анализ продаж по категориям с учетом покупательского поведения.
        WITH category_stats AS (
            -- Продажи по категориям (INNER JOIN для точных данных)
            SELECT 
                p.category AS "Категория",
                COUNT(DISTINCT o.customer_id) AS "Уникальные покупатели",
                SUM(oi.quantity) AS "Продано единиц",
                SUM(oi.quantity * oi.unit_price) AS "Общая выручка"
            FROM products p
            INNER JOIN order_items oi ON p.product_id = oi.product_id
            INNER JOIN orders o ON oi.order_id = o.order_id AND o.status = 'completed'
            GROUP BY p.category
        ),
        category_potential AS (
            -- Потенциал категорий (CROSS JOIN + LEFT JOIN для анализа охвата)
            SELECT 
                p.category AS "Категория",
                COUNT(DISTINCT c.customer_id) AS "Всего клиентов",
                COUNT(DISTINCT o.customer_id) AS "Клиенты с покупками",
                ROUND(
                    (COUNT(DISTINCT o.customer_id) * 100.0 / NULLIF(COUNT(DISTINCT c.customer_id), 0)), 
                    2
                ) AS "Проникновение %"
            FROM products p
            CROSS JOIN customers c
            LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'
            LEFT JOIN order_items oi ON o.order_id = oi.order_id AND oi.product_id = p.product_id
            GROUP BY p.category
        )
        -- Финальный дашборд (FULL JOIN для полной картины)
        SELECT 
            COALESCE(cs."Категория", cp."Категория") AS "Категория",
            cs."Общая выручка",
            cs."Продано единиц", 
            cs."Уникальные покупатели",
            cp."Всего клиентов",
            cp."Проникновение %",
            CASE 
                WHEN cp."Проникновение %" < 20 THEN 'Низкий охват'
                WHEN cp."Проникновение %" BETWEEN 20 AND 50 THEN 'Средний охват' 
                ELSE 'Высокий охват'
            END AS "Оценка охвата"
        FROM category_stats cs
        FULL JOIN category_potential cp ON cs."Категория" = cp."Категория"
        ORDER BY cs."Общая выручка" DESC NULLS LAST;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/combined_approach.PNG)

**Результат:** полная картина эффективности товарных категорий, выявление точек роста.

## Домашнее задание со звёздочкой (*)
1. **Метрика 1**: Индекс здоровья клиентской базы
    ```sql
        -- Композитный индекс здоровья клиентской базы
        SELECT 
            status AS "Сегмент клиентов",           -- Группировка по статусу (active, premium, inactive)
            COUNT(*) AS "Кол-во клиентов",          -- Общее количество клиентов в сегменте
            ROUND(AVG(order_count), 2) AS "Ср. кол-во заказов",  -- Средняя активность клиентов
            ROUND(AVG(ltv), 2) AS "Ср. LTV",        -- Lifetime Value - общая ценность клиента
            ROUND((COUNT(*) FILTER (WHERE order_count > 0) * 100.0 / COUNT(*)), 2) AS "Конверсия %",
            -- Композитная оценка здоровья (0-100) - ИНТЕГРАЛЬНЫЙ ПОКАЗАТЕЛЬ
            ROUND(
                (AVG(order_count) * 25 +           -- 25% веса: активность (количество заказов)
                AVG(ltv) / 1000 * 25 +            -- 25% веса: денежная ценность (нормализованная)
                (COUNT(*) FILTER (WHERE order_count > 0) * 100.0 / COUNT(*)) * 0.5),  -- 50% веса: конверсия
                2
            ) AS "Health Score"
        FROM (
            -- Подзапрос: расчет метрик для каждого клиента
            SELECT 
                c.customer_id,
                c.status,
                COUNT(o.order_id) AS order_count,   -- Сколько заказов сделал клиент
                COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS ltv  -- Общая сумма всех покупок клиента
            FROM customers c
            LEFT JOIN orders o ON c.customer_id = o.customer_id AND o.status = 'completed'  -- Только завершенные заказы
            LEFT JOIN order_items oi ON o.order_id = oi.order_id
            GROUP BY c.customer_id, c.status
        ) AS customer_metrics
        GROUP BY status
        ORDER BY "Health Score" DESC;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/chs.PNG)

**Назначение**: 
- Комплексная оценка "здоровья" разных сегментов клиентской базы.
- Позволяет сравнивать эффективность работы с разными группами клиентов.

**Интерпретация результатов**:
- Health Score 80-100: Отличное состояние - клиенты активны, платят много, большинство совершают покупки
- Health Score 60-79: Хорошее состояние - есть потенциал для роста
- Health Score 40-59: Требует внимания - нужно разбираться в проблемах
- Health Score 0-39: Критическое состояние - срочные меры по удержанию клиентов

**Действия на основе метрики**:
- Premium клиенты с высоким score: развивать программу лояльности
- Active клиенты с низким score: запускать реактивационные кампании
- Inactive клиенты: анализировать причины ухода

2. **Метрика 2**: Матрица эффективности товаров
    ```sql
        -- Матрица эффективности товаров (BCG-матрица)
        SELECT 
            product_name AS "Товар",
            category AS "Категория",
            total_revenue AS "Выручка",           -- Общая выручка от товара
            units_sold AS "Продажи",              -- Количество проданных единиц
            customer_count AS "Покупатели",       -- Сколько уникальных клиентов купили товар
            ROUND(total_revenue / NULLIF(units_sold, 0), 2) AS "Ср. чек",  -- Средняя цена продажи
            CASE 
                WHEN total_revenue > 100000 AND units_sold > 1 THEN 'Звезда'        -- Высокие продажи и спрос
                WHEN total_revenue > 50000 AND customer_count > 1 THEN 'Дойная корова' -- Стабильный доход от нескольких клиентов
                WHEN total_revenue < 30000 AND units_sold = 1 THEN 'Темная лошадка' -- Потенциал (пока мало данных)
                ELSE 'Вопрос'                      -- Требует дополнительного анализа
            END AS "BCG-статус"
        FROM (
            -- Подзапрос: агрегация данных по товарам
            SELECT 
                p.product_id,
                p.product_name,
                p.category,
                SUM(oi.quantity * oi.unit_price) AS total_revenue,  -- Суммарная выручка
                SUM(oi.quantity) AS units_sold,                     -- Общее количество продаж
                COUNT(DISTINCT o.customer_id) AS customer_count     -- Уникальные покупатели
            FROM products p
            LEFT JOIN order_items oi ON p.product_id = oi.product_id
            LEFT JOIN orders o ON oi.order_id = o.order_id AND o.status = 'completed'  -- Только успешные продажи
            GROUP BY p.product_id, p.product_name, p.category
        ) AS product_performance
        ORDER BY total_revenue DESC;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/ppm.PNG)

**Назначение**: 
Аналог BCG-матрицы для товаров. Классифицирует товары на 4 категории:
- Звезды: высокие продажи и рост
- Дойные коровы: стабильный доход
- Темные лошадки: потенциал для роста  
- Вопросы: требуют анализа

**Интерпретация результатов**:
- **Звёзды (Stars)**:
    - Максимальная выручка и количество продаж
    - Стратегия: инвестировать в рекламу, увеличивать наличие

- **Дойные коровы (Cash Cows)**:
    - Стабильный доход, несколько постоянных покупателей  
    - Стратегия: поддерживать наличие, минимальные инвестиции

- **Тёмные лошадки (Dark Horses)**:
    - Мало данных, но есть потенциал
    - Стратегия: тестовые рекламные кампании, анализ спроса

- **Вопросы (Question Marks)**:
    - Непонятная динамика
    - Стратегия: глубокий анализ, A/B тестирование цен и промо

**Действия на основе метрики**:
- Увеличивать закупки "Звезд"
- Оптимизировать логистику "Дойных коров" 
- Экспериментировать с "Темными лошадками"
- Анализировать "Вопросы" на предмет исключения из ассортимента

3. **Метрика 3**: Дашборд операционной эффективности
    ```sql
        -- Дашборд операционной эффективности
        SELECT 
            'Конверсия в покупку' AS "Метрика",    -- Название метрики
            ROUND(
                (SELECT COUNT(DISTINCT customer_id) FROM orders WHERE status = 'completed') * 100.0 / 
                NULLIF((SELECT COUNT(*) FROM customers), 0),  -- Процент клиентов, сделавших хотя бы одну покупку
                2
            ) AS "Значение %",
            'Чем выше, тем лучше' AS "Интерпретация"  -- Как понимать этот показатель

        UNION ALL  -- Объединение с следующей метрикой

        SELECT 
            'Средний чек',                         -- Средняя сумма заказа
            ROUND(
                (SELECT COALESCE(SUM(oi.quantity * oi.unit_price), 0) FROM order_items oi
                JOIN orders o ON oi.order_id = o.order_id AND o.status = 'completed') / 
                NULLIF((SELECT COUNT(*) FROM orders WHERE status = 'completed'), 0), 
                2
            ),
            'Рост = увеличение LTV'                -- Рост среднего чека увеличивает пожизненную ценность клиента

        UNION ALL

        SELECT 
            'Глубина корзины',                     -- Среднее количество товаров в заказе
            ROUND(
                (SELECT COALESCE(SUM(oi.quantity), 0) FROM order_items oi
                JOIN orders o ON oi.order_id = o.order_id AND o.status = 'completed') / 
                NULLIF((SELECT COUNT(*) FROM orders WHERE status = 'completed'), 0), 
                2
            ),
            'Кол-во товаров в заказе'              -- Показатель кросс-продаж и рекомендаций
            
        UNION ALL

        SELECT 
            'Охват категорий',                     -- Насколько разнообразны покупки
            ROUND(
                (SELECT COUNT(DISTINCT category) FROM products p
                JOIN order_items oi ON p.product_id = oi.product_id) * 100.0 /
                NULLIF((SELECT COUNT(DISTINCT category) FROM products), 0), 
                2
            ),
            '% проданных категорий от всех';       -- Диверсификация покупок
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_9/oed.PNG)

**Назначение**:
- Ключевые показатели эффективности (KPI) бизнеса в одном месте.
- Мониторинг основных бизнес-процессов.

**Интерпретация результатов**:
- **Конверсия в покупку**:
    - 0-20%: Низкая - проблемы с привлечением или удержанием
    - 21-40%: Средняя - есть потенциал роста
    - 41%+: Высокая - эффективная воронка продаж

- **Средний чек**:
    - Сравнивать с конкурентами в нише
    - Рост = успех upsell и cross-sell (перекрёсные продажи) стратегий

- **"Глубина" корзины**:
    - 1.0: Каждый заказ = 1 товар (плохо)
    - 1.5+: Хорошие кросс-продажи
    - 2.0+: Отличная работа рекомендательной системы

- **Охват категорий**:
    - <50%: Клиенты покупают узкий ассортимент
    - 70%+: Хорошая диверсификация покупок
    - 90%+: Отличное использование всего каталога

**Действия на основе метрики**:
- Низкая конверсия: улучшать UX, упрощать покупку
- Низкий средний чек: внедрять рекомендации "с этим покупают"
- Малая глубина корзины: улучшать рекомендательную систему
- Узкий охват категорий: проводить тематические акции