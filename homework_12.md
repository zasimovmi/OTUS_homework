# Выпонение домашнего задания №12

# Триггеры, поддержка заполнения витрин

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

3. Перейдём в созданную базу данных
    ```sql
        \c test_db
    ```

4. Создадим схему
    ```sql
        DROP SCHEMA IF EXISTS pract_functions CASCADE;
        CREATE SCHEMA pract_functions;
    ```

5. Устанавливаем схему по умолчанию
    ```sql
        -- Была опечатка в слове "publi", исправил на верное значение
        SET search_path = pract_functions, public
    ```

6. Воссоздаём всю структуру и данные из домашнего задания (из файла `hw_triggers.sql`)
    ```sql
        -- Товары:
        CREATE TABLE goods (
            goods_id    integer PRIMARY KEY,
            good_name   varchar(63) NOT NULL,
            good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
        );

        -- Заполняем таблицу товаров (goods) данными
        -- Тут была опечатка в слове "хозайственные", исправил на верное значение
        INSERT INTO goods (goods_id, good_name, good_price)
        VALUES (1, 'Спички хозяйственные', .50), (2, 'Автомобиль Ferrari FXX K', 185000000.01);

        -- Продажи
        CREATE TABLE sales (
            sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
            good_id     integer REFERENCES goods (goods_id),
            sales_time  timestamp with time zone DEFAULT now(),
            sales_qty   integer CHECK (sales_qty > 0)
        );

        -- Заполняем таблицу продаж (sales) данными
        INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
    ```

![Структура полностью воссоздана (goods)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_12/table_goods.PNG)
![Структура полностью воссоздана (sales)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_12/table_sales.PNG)

### Анализируем, как работает отчёт
1. Исходный отчёт:
    ```sql
        SELECT G.good_name, sum(G.good_price * S.sales_qty)
        FROM goods G
        INNER JOIN sales S ON S.good_id = G.goods_id
        GROUP BY G.good_name;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_12/report.PNG)

**Выводы:** с увеличением объёма данных отчет стал создаваться медленно.
Поэтому принято решение денормализовать БД.  

`Денормализация` - это сознательное нарушение нормальных форм базы данных для повышения производительности

2. Создать таблицу
    ```sql
        -- Создаём таблицу-витрину
        CREATE TABLE good_sum_mart (
            good_name   varchar(63) NOT NULL,
            sum_sale    numeric(16, 2) NOT NULL
        );
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_12/table_good_sum_mart.PNG)

**Как это работает:**
    ```text
        Нормализованная БД (медленно):
        Таблица goods        Таблица sales
        goods_id ───────────── good_id
        good_name             sales_qty
        good_price

        Для отчёта нужно каждый раз:
        - Соединить (JOIN) две большие таблицы
        - Умножить price × qty (количество)
        - Группировать (GROUP BY)
        - Суммировать (SUM)

        Денормализованная БД (быстро):
        Таблица good_sum_mart
        good_name   sum_sale
        "Спички"    65.50
        "Ferrari"   185000000.01

        Для отчёта теперь:
        1. SELECT * FROM good_sum_mart (мгновенно!)
    ```

**Комментарии:** эта таблица будет хранить готовые результаты вместо их вычисления каждый раз

### Создадим триггерную функцию и сам триггер
1. Создать триггерную функцию
    ```sql
        /*
        Триггерная функция для поддержки денормализованной витрины good_sum_mart

        ЦЕЛЬ: 
        - Автоматически пересчитывать итоговую сумму продаж при любых изменениях в таблице sales
        - Исключить необходимость выполнения тяжёлого GROUP BY SUM() при каждом запросе отчёта

        ПРИНЦИП РАБОТЫ:
        1. Для DELETE и UPDATE: вычитаем сумму старой продажи из витрины
        2. Для INSERT и UPDATE: добавляем сумму новой продажи в витрину
        3. Используем MERGE (UPSERT) для атомарного обновления витрины
        
        ВАЖНЫЕ МОМЕНТЫ:
        - Функция работает с исходной структурой БД (не требует изменения таблиц)
        - Всегда актуальные цены из таблицы goods (цена берётся на момент операции)
        - Обрабатывает все три типа операций: INSERT, UPDATE, DELETE
        */
        CREATE OR REPLACE FUNCTION sales_after_changes()
        RETURNS TRIGGER AS $$
        BEGIN
            -- БЛОК 1: удаление или обновление (убираем старую сумму)
            -- При DELETE: полностью удаляем сумму продажи из витрины
            -- При UPDATE: вычитаем старую сумму перед добавлением новой (в блоке 2)
            IF TG_OP IN ('DELETE', 'UPDATE') THEN
                UPDATE good_sum_mart
                SET sum_sale = sum_sale - (
                    SELECT OLD.sales_qty * g.good_price
                    FROM goods g
                    WHERE g.goods_id = OLD.good_id
                )
                WHERE good_name = (
                    SELECT g.good_name 
                    FROM goods AS g 
                    WHERE g.goods_id = OLD.good_id
                );
                -- Примечание: если товар в витрине станет с нулевой суммой, 
                -- строка останется (можно добавить DELETE WHERE sum_sale = 0)
            END IF;

            -- БЛОК 2: вставка или обновление (добавляем новую сумму)
            -- При INSERT: добавляем новую продажу
            -- При UPDATE: добавляем обновлённую сумму после вычитания старой (из блока 1)
            IF TG_OP IN ('UPDATE', 'INSERT') THEN
                -- Используем MERGE (UPSERT) для атомарной вставки или обновления
                -- Преимущество MERGE: избегаем гонок условий при параллельных операциях
                MERGE INTO good_sum_mart AS gsm
                USING (
                    SELECT good_name, good_price
                    FROM goods
                    WHERE goods_id = NEW.good_id
                ) AS g
                ON gsm.good_name = g.good_name
                WHEN MATCHED THEN
                    -- Товар уже есть в витрине: увеличиваем сумму
                    UPDATE SET sum_sale = sum_sale + g.good_price * NEW.sales_qty
                WHEN NOT MATCHED THEN 
                    -- Товара нет в витрине: создаём новую запись
                    INSERT (good_name, sum_sale) 
                    VALUES (g.good_name, g.good_price * NEW.sales_qty);
            END IF;

            -- Для триггерных функций, выполняемых после операции AFTER, возвращаемое значение игнорируется системой
            -- но синтаксис требует, чтобы функция возвращала значение.
            -- Стандартная правктика - RETURN NULL
            RETURN NULL;
        END;
        $$ LANGUAGE plpgsql;
    ```

![Триггерная функция успешно создана](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_12/create_function.PNG)

2. Создать триггер
    ```sql
        -- Создаём триггер, который будет запускать функцию после каждой операции с продажами
        -- Триггер уровня строки (FOR EACH ROW) гарантирует обработку каждой изменённой записи
        CREATE TRIGGER sales_after_change_trigger 
        AFTER INSERT OR UPDATE OR DELETE ON sales
        FOR EACH ROW
        EXECUTE FUNCTION sales_after_changes();
    ```

![Триггер успешно создан](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_12/create_trigger.PNG)

### Тестирование работы триггера и витрины
Как мы должны были поступить в реальной жизни  

Наполнить витрину исходными данными (это делается **один раз** при внедрении системы)
   ```sql
        -- В реальной жизни: взять все исторические продажи и загрузить в витрину
        INSERT INTO good_sum_mart
        -- отчет
        SELECT G.good_name, sum(G.good_price * S.sales_qty)
        FROM goods G
        INNER JOIN sales S ON S.good_id = G.goods_id
        GROUP BY G.good_name;
   ```

Но в домашнем задании пойдём по другому

1. Выполним действия
   ```sql
        -- Очистим таблицу sales
        TRUNCATE sales;

        -- Заново вставим данные, чтобы сработал триггер и наполнил таблицу "good_sum_mart"
        INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

        -- Проверим что выдает отчет
        SELECT G.good_name, sum(G.good_price * S.sales_qty)
        FROM goods G
        INNER JOIN sales S ON S.good_id = G.goods_id
        GROUP BY G.good_name
        ORDER BY G.good_name;

        -- Результат:
                good_name         |     sum
        --------------------------+--------------
        Автомобиль Ferrari FXX K  | 185000000.01
        Спички хозяйственные      |        65.50
        (2 rows)

        -- Проверим что выдает витрина
        SELECT * FROM good_sum_mart ORDER BY good_name;

        -- Результат:
                good_name         |   sum_sale
        --------------------------+--------------
        Автомобиль Ferrari FXX K  | 185000000.01
        Спички хозяйственные      |        65.50
        (2 rows)
   ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_12/result.PNG)

**Вывод:** результат идентичен в двух случаях

2. Делаем более расширенное тестирование
  ```sql
        -- ТЕСТ 1: Добавление нового товара и продаж
        INSERT INTO goods (goods_id, good_name, good_price) VALUES 
            (3, 'Шариковая ручка', 25.00);

        -- Продажи ручек: 3 + 5 + 2 = 10 штук
        INSERT INTO sales (good_id, sales_qty) VALUES 
            (3, 3),  -- 3 * 25 = 75.00
            (3, 5),  -- 5 * 25 = 125.00 (итого: 200.00)
            (3, 2);  -- 2 * 25 = 50.00 (итого: 250.00)

        -- Проверяем витрину
        SELECT * FROM good_sum_mart ORDER BY good_name;
        
        -- Результат:
                good_name         |   sum_sale
        --------------------------+--------------
        Автомобиль Ferrari FXX K  | 185000000.01
        Спички хозяйственные      |        65.50
        Шариковая ручка           |       250.00
        (3 rows)

        -- ТЕСТ 2: UPDATE - изменение количества продаж
        -- Возврат 2 ручек (отмена продажи)
        UPDATE sales SET sales_qty = 1 WHERE sales_id = 9;  -- было 3, стало 1 (-50) и id_где_было_3 (в моём случае - это 9)
        UPDATE sales SET sales_qty = 10 WHERE sales_id = 10; -- было 5, стало 10 (+125) и id_где_было_5 (в моём случае - это 10)

        -- Проверяем: было 250, стало 250 - 50 + 125 = 325
        SELECT * FROM good_sum_mart WHERE good_name = 'Шариковая ручка';

        -- Результат:
            good_name    | sum_sale
        -----------------+----------
        Шариковая ручка  |   325.00
        (1 row)

        -- ТЕСТ 3: DELETE - полный возврат товара
        DELETE FROM sales WHERE good_id = 3;
        
        -- Витрина должна показать 0 для ручек (так как очистка не реализована)
        -- Если бы была реализована очистка, то строка удалилась
        SELECT * FROM good_sum_mart WHERE good_name = 'Шариковая ручка';

        -- Результат:
            good_name    | sum_sale
        -----------------+----------
        Шариковая ручка  |     0.00
        (1 row)
  ```

3. Критический тест: изменение цены товара
  ```sql
        -- Ситуация: цена ручки выросла с 25.00 до 35.00
        UPDATE goods SET good_price = 35.00 WHERE goods_id = 3;

        -- Добавляем новую продажу по новой цене
        INSERT INTO sales (good_id, sales_qty) VALUES (3, 10); -- 10 * 35 = 350.00

        -- Проверяем витрину
        SELECT * FROM good_sum_mart WHERE good_name = 'Шариковая ручка';
        
        -- Ожидаем: 350.00
        -- Фактический результат:
            good_name    | sum_sale
        -----------------+----------
        Шариковая ручка  |   350.00
        (1 row)

        -- А что с предыдущими продажами? Они пересчитались по новой цене? НЕТ!
        -- Наш триггер использует цену НА МОМЕНТ ОПЕРАЦИИ из таблицы goods

        -- Проверяем отчёт (он использует ТЕКУЩУЮ цену)
        SELECT G.good_name, sum(G.good_price * S.sales_qty)
        FROM goods G INNER JOIN sales S ON S.good_id = G.goods_id
        WHERE G.goods_id = 3
        GROUP BY G.good_name;

        -- Результат: 10 * 35 = 350.00
            good_name    |  sum
        -----------------+--------
        Шариковая ручка  | 350.00

        -- ВСЁ КОРРЕКТНО! Витрина и отчёт совпадают.
        -- Почему? Потому что старых продаж ручек больше нет (мы их удалили в тесте 3).
  ```

А что, если бы старые продажи остались?
  ```sql
        -- Воссоздаём проблему
        DELETE FROM sales WHERE good_id = 3;
        TRUNCATE good_sum_mart;

        -- Продаём 10 ручек по старой цене 35.00
        INSERT INTO sales (good_id, sales_qty) VALUES (3, 10);

        -- Меняем цену на 45.00
        UPDATE goods SET good_price = 45.00 WHERE goods_id = 3;

        -- Проверяем отчёт:
        SELECT G.good_name, sum(G.good_price * S.sales_qty)
        FROM goods G INNER JOIN sales S ON S.good_id = G.goods_id
        WHERE G.goods_id = 3
        GROUP BY G.good_name; 

        -- Результат:
            good_name    |  sum
        -----------------+--------
        Шариковая ручка  | 450.00
        (1 row) 

        -- Проверяем витрину:
            good_name    |  sum
        -----------------+--------
        Шариковая ручка  | 350.00
        (1 row) 
  ```

**Выводы:**
- Витрина (триггер использовал цену 35.00): 10 * 35 = 350.00
- Отчёт (использует текущую цену 45.00): 10 * 45 = 450.00
- РАСХОЖДЕНИЕ! 350 ≠ 450

**Обнаруженные ограничения и проблемы:**
- **Зависимость от цен в goods**: 
   - Триггер использует цену на момент операции
   - Если цена в `goods` изменится позже, исторические продажи в витрине **не** обновятся
   - Отчёт же будет считать по **текущим** ценам → возможно расхождение

- **Архитектурный недостаток**:
   - В реальной системе в таблице `sales` нужно хранить `sale_price` (цену на момент продажи)
   - Это решает проблему исторической точности данных

## Домашнее задание со звёздочкой (*)
***Чем схема (витрина+триггер) предпочтительнее отчёта, создаваемого "по требованию" (кроме производительности)?***

Самое главное - это историческая точность данных

| Аспект	             | Витрина + триггер	                        | Отчёт "по требованию"                                   |
|------------------------|----------------------------------------------|---------------------------------------------------------|
| Временная привязка 	 | Фиксирует данные на момент операции	        | Использует текущие значения на момент запроса           |
| Изменение цен	         | Сумма продажи "замораживается" при INSERT	| Все исторические продажи пересчитываются по новой цене  |
| Финансовая отчётность  | Показывает реальную выручку на дату продажи  | Может искажать фактическую прибыль                      |

Пример из моего тестирования:
- Продали 10 ручек по 35₽ → реальная выручка 350₽
- Через месяц цена стала 45₽
- Витрина: всё равно показывает 350₽ (историческая правда)
- Отчёт: покажет 450₽ (финансово некорректно!)

**Вывод:** Витрина сохраняет "финансовый снимок" на момент сделки, что критически важно для, как пример, бухгалтерского учёта, налоговых отчётов и анализа динамики продаж