# Выпонение домашнего задания №13

# Бэкапы

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

2. Создаем базу данных, схему и таблицы
    ```sql
        -- Создаём БД 'backup_db'
        CREATE DATABASE backup_db;

        -- Подключаемся к новой БД
        \c backup_db

        -- Создаем схему 'test'
        CREATE SCHEMA test;

        -- Устанавливаем путь поиска для удобства
        SET search_path TO test, public;

        -- Создаем первую таблицу 'table1'
        CREATE TABLE test.table1 (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            random_value INTEGER
        );

        -- Создаем вторую таблицу 'table2' с такой же структурой для будущего восстановления
        CREATE TABLE test.table2 (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            random_value INTEGER
        );

        -- Проверяем создание
        \dt test.*
    ```

![Всё успешно создано](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/dt_test.PNG)

**Результат:**  
Создана:
- база данных backup_db
- схема test
- две таблицы:
    - table1 (основная)
    - table2 (для восстановления через COPY)

## Заполнить таблицы автосгенерированными записями (100 штук)
1. Выполнить команды
    ```sql
        -- Заполняем table1 100 записями
        INSERT INTO test.table1 (name, random_value)
        SELECT 
            'User_' || i,
            (random() * 1000)::INTEGER
        FROM generate_series(1, 100) AS i;

        -- Проверяем заполнение
        SELECT COUNT(*) FROM test.table1;
        SELECT * FROM test.table1 LIMIT 5;
    ```

![Данные успешно вставлены](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/count_table1.PNG)
![Данные успешно вставлены](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/insert_table1.PNG)

**Результат:** была использована функция `generate_series` и случайные данные.  
Таблица `table1` успешно заполнена 100 записями со случайными значениями.

## Создание каталога для бэкапов
1. Выходим из `psql` (команда `\q`) и в терминале под пользователем `postgres` создаем каталог
```bash
        sudo -u postgres bash  # Или остаемся под sudo -u postgres
        mkdir -p /var/lib/postgresql/backups
        cd /var/lib/postgresql/backups

        # Проверяем
        pwd
        ls -la
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/ls_la.PNG)

**Результат:** каталог `/var/lib/postgresql/backups/` создан. Он будет использоваться для хранения файлов резервных копий. Важно, что владельцем является `postgres`.

## Логический бэкап с помощью COPY
1. Используем SQL-команду `COPY` для экспорта данных в текстовый файл
```bash
        # Подключаемся к БД и выполняем COPY
        # 'sudo -u postgres' не используем, так как мы вошли как пользователь 'postgres' шагом ранее
        psql -d backup_db -c "COPY test.table1 TO STDOUT WITH CSV HEADER DELIMITER ',';" > /var/lib/postgresql/backups/table1_backup.csv
```

**Разбор команды:**
- `-c` - выполнить команду и выйти
- `COPY ... TO STDOUT` - отправить данные таблицы в стандартный вывод
- `WITH CSV HEADER DELIMITER ','` - формат CSV с заголовком столбцов
- `>` - перенаправление вывода в файл

2. Проверяем бэкап
```bash
        # Выведем первые пять строк из файла
        head -5 /var/lib/postgresql/backups/table1_backup.csv
```

![Бэкап успешно создан>](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/copy_backup.PNG)

**Результат:** создан файл `table1_backup.csv` с 101 строкой (заголовок + 100 данных). Данные сохранены в удобочитаемом формате.

## Восстановление данных из CSV-бэкапа во вторую таблицу
1. Используем `COPY` для импорта. Сначала очистим `table2` (она пустая, но на будущее).
```bash
        psql -d backup_db -c "COPY test.table2 FROM '/var/lib/postgresql/backups/table1_backup.csv' WITH CSV HEADER DELIMITER ',';"
```

2. Проверяем восстановление
```bash
        psql -d backup_db -c "SELECT COUNT(*) FROM test.table2;"
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/copy_count.PNG)

**Вывод:** данные из `table1_backup.csv` успешно восстановлены в таблицу `table2`. В ней теперь также 100 записей.  
Самое важное, что последовательности (SERIAL) не сбрасывались, но значения ID были импортированы как данные.  

В CSV-файл были включены значения id и при импорте загрузили эти значения "как есть", а не позволили PostgreSQL сгенерировать новые.  

Последовательность (sequence), связанная с SERIAL, не была обновлена.  

Это приводит к проблеме: если вставить новое значение в таблицу `table2`, то PostgreSQL попытается использовать `nextval('table2_id_seq')`, которое, возможно, меньше, чем максимальный id, уже существующий в таблице.  
Как результат - ошибка уникальности (`duplicate key`), потому что `nextval` выдаст, скажем, 100, а id=100 уже есть.

Можно исправить данную проблему:  
После восстановления данных с явными id нужно синхронизировать последовательность  
```bash
        # Обновить последовательность для table2.id
        psql -d backup_db -c "SELECT setval(pg_get_serial_sequence('test.table2', 'id'),(SELECT MAX(id) FROM test.table2));;"
```

Эта команда:
- находит имя SEQUENCE, связанной с колонкой id в table2
- устанавливает её текущее значение на MAX(id)
- следующий INSERT без id получит MAX(id) + 1 → конфликта не будет

## Бэкап двух таблиц с помощью pg_dump в кастомном сжатом формате
1. Используем формат `-Fc`(custom), который поддерживает сжатие и выборочное восстановление
```bash
        pg_dump -d backup_db -n test -t test.table1 -t test.table2 -Fc -f /var/lib/postgresql/backups/two_tables_backup.dump -v
```

**Разбор ключей (опций):**
- `-d backup_db` - указание БД
- `-n test` - бэкапить только объекты из схемы `test`
- `-t test.table1` - указание конкретной таблицы. Можно несколько `-t`.
- `-Fc` - формат вывода: custom (архив, сжатый)
- `-f` - файл для вывода
- `-v` - подробный вывод (`verbose`)

2. Проверяем содержимое дампа
```bash
        pg_restore -l /var/lib/postgresql/backups/two_tables_backup.dump | head -20
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/pg_dump_backup.PNG)

**Результат:** создан сжатый архив `two_tables_backup.dump`. В нем содержатся определения схемы, таблиц (`CREATE TABLE`), индексы (`PRIMARY KEY`) и сами данные двух таблиц. Формат `-Fc` не читается текстовым редактором, но `pg_restore` может извлекать из него информацию.

## Восстановление только второй таблицы в новую БД с помощью pg_restore
1. Создадим новую чистую БД
```bash
        psql -c "CREATE DATABASE backup_db_restored;"
        psql -d backup_db_restored -c "CREATE SCHEMA test;"
```

2. Теперь восстановим только `table2` из архива в новую БД. Нам нужно исключить все объекты, кроме `table2`.
```bash
        # 1. Создаем файл-листинг содержимого дампа
        pg_restore -l /var/lib/postgresql/backups/two_tables_backup.dump > /var/lib/postgresql/backups/list.txt

        # 2. Редактируем листинг, оставляя только table2 и связанные объекты (например, ее первичный ключ).
        #    Можно сделать это вручную в редакторе (nano) или через sed.
        #    Самый надежный способ для задания: создать новый файл списка с нужными номерами строк.
        #    Но для автоматизации отфильтруем строки с table2.
        pg_restore -l /var/lib/postgresql/backups/two_tables_backup.dump | grep -E '(TABLE |TABLE DATA|INDEX|CONSTRAINT).*table2' > /var/lib/postgresql/backups/table2_list.txt

        # 3. Восстанавливаем в новую БД, используя отфильтрованный листинг
        pg_restore -d backup_db_restored -L /var/lib/postgresql/backups/table2_list.txt -v /var/lib/postgresql/backups/two_tables_backup.dump
```

3. Проверяем восстановление
```bash
        psql -d backup_db_restored -c "\dt test.*"
        psql -d backup_db_restored -c "SELECT COUNT(*) FROM test.table2;"
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/pg_restore_dt.PNG)
![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_13/pg_restore_count.PNG)

**Результат:** в новую базу данных `backup_db_restored` успешно восстановлена только таблица `table2` (ее структура и данные) из комплексного дампа `two_tables_backup.dump`  

Когда мы делаем дамп в кастомном формате (`-Fc`), `pg_dump` сохраняет не просто SQL, а структурированный архив, содержащий множество объектов:
- таблицы (DDL)
- данные (TABLE DATA)
- индексы
- ограничения (CONSTRAINT, PRIMARY KEY, FOREIGN KEY)
- последовательности
- права доступа и т.д.

**Комментарии:**
- `pg_restore -l backup.dump` (листинг) - выводит список всех объектов в этом архиве в специальном формате  
- `TABLE |TABLE DATA|INDEX|CONSTRAINT` - особое внимание на пробел после `TABLE`, чтобы не поймать `TABLE DATA` дважды

**Выводы:**  
В ходе работы были освоены два принципиально разных подхода к логическому резервному копированию в PostgreSQL:
- Утилита `COPY`: 
    - Идеальна для быстрого экспорта/импорта данных конкретных таблиц в человекочитаемом формате (CSV). Не требует дополнительных прав и проста в использовании из SQL-сессии
    - ***Недостаток:*** не переносит метаинформацию (индексы, триггеры, владельцев) в рамках одной команды
    - Подходит для обмена данными
- Утилиты `pg_dump` и `pg_restore`:
    - Профессиональный инструмент для резервного копирования
    - Формат `-Fc` (custom) предоставляет:
        - Сжатие данных
        - Выборочное восстановление отдельных объектов (как было продемонстрировано на примере восстановления только `table2`)
        - Параллельное выполнение (опция `-j`)
        - Сохранение всей объектной модели БД