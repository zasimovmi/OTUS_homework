# Выпонение домашнего задания №14

# Репликация

## Основное домашнее задание

### Проделать основные шаги из других домашних заданий (на 4-х ВМ)
Стандартные шаги:
- Установка VirtualBox на Windows
- Создание виртуальной машины в VirtualBox
- Создать правило для ssh подключения
- Первоначальная настройка ВМ
- Проверить, установлен и запущен ли SSH-сервер
- Установка PosrgreSQl и проверка работы кластера

**Не забыть:** сеть на каждой ВМ изменить на "Cетевой мост"

### Предварительная подготовка
Действия выполняются на всех 4-х ВМ  
1. Отредактировать `postgresql.conf`
```bash
        sudo nano /etc/postgresql/17/main/postgresql.conf
```

2. Внести изменения (ниже приведены параметры, которые были изменены)
```text
        listen_addresses = '*'         # Разрешаем прослушивать все адреса
        port = 5432                    # Для всех ВМ!
        wal_level = logical            # КРИТИЧЕСКИ важно для логической репликации (ВМ1, ВМ2, ВМ3). Для ВМ4 можно replica, но оставим logical для гибкости.
        max_replication_slots = 10     # Минимум по одному слоту на подписку + для физической репликации
        max_wal_senders = 10
        shared_preload_libraries = 'pg_stat_statements'
```

3. Сохранить файл и выйти в терминал

4. Отредактировать `pg_hba.conf`
Разрешаем аутентификацию по паролю при подключении с любого адреса
```bash
        sudo nano /etc/postgresql/17/main/pg_hba.conf
```

5. Внести изменения (в самый конец файла)
```text
        host    all             all             0.0.0.0/0               scram-sha-256
        host    all             all             ::/0                    scram-sha-256
```

6. Сохранить файл и выйти в терминал

7. Создать пользователя для репликации (на ВМ1, ВМ2, ВМ3)
```bash
        sudo -u postgres psql -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator';"
        sudo -u postgres psql -c "GRANT pg_monitor TO replicator;"
```

**Комментарии:**  
`CREATE ROLE replicator`
- `CREATE ROLE` - создаёт роль (пользователя) в PostgreSQL
- `replicator` - имя пользователя для репликации
- `WITH REPLICATION` - ***ВАЖНО:*** даёт право на репликацию
- `LOGIN` - позволяет входить в БД (не все роли могут логиниться)
- `PASSWORD 'replicator'` - пароль
Что даёт `WITH REPLICATION`:
- Чтение WAL (Write-Ahead Log) файлов
- Подключение в режиме replication
- Создание/использование слотов репликации
`GRANT pg_monitor TO replicator`
- `pg_monitor` - системная роль для мониторинга
- Даёт права на чтение системных представлений:
    - `pg_stat_activity` - кто подключен, какие запросы выполняет
    - `pg_stat_replication` - статус репликации, лаги
    - `pg_stat_database` - статистика по БД
- Полезно для отладки: можно подключаться как `replicator` и смотреть статус репликации

8. Выполнить перезагрузку PostgreSQL
```bash
        # Обязательно для внесения изменений
        sudo systemctl restart postgresql
```

**Комментарии:**
- На каждой ВМ я сделал команду `hostname -I`
- Получил ip (для дальнейших подключений)
    - ВМ1 - `192.168.0.41`
    - ВМ2 - `192.168.0.42`
    - ВМ3 - `192.168.0.43`
    - ВМ4 - `192.168.0.44`

### Создание таблиц и публикаций на ВМ1
1. Создание таблиц
```sql
        -- Подключаемся к PostgreSQL
        sudo -u postgres psql

        -- Создаем базу данных
        CREATE DATABASE appdb;

        \c appdb

        -- Таблица для записи (публикуем)
        CREATE TABLE test (
            id SERIAL PRIMARY KEY,
            data TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );

        -- Таблица для чтения (будем подписываться из ВМ2)
        CREATE TABLE test2 (
            id SERIAL PRIMARY KEY,
            data TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );

        -- Проверяем
        \dt
```

![Структура ВМ1](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/dt_vm_1.PNG)

2. Создание публикации для таблицы `test`
```sql
        -- Создаем публикацию только для таблицы test
        CREATE PUBLICATION pub_test FOR TABLE test;

        -- Проверяем публикацию
        SELECT * FROM pg_publication;
```

![Публикация успешно создана](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_publication_vm_1.PNG)

**Результат:** создана публикация `pub_test` для таблицы `test`. Теперь изменения в этой таблице можно реплицировать на другие серверы.

### Создание таблиц и публикаций на ВМ2
1. Создание таблиц
```sql
        -- Подключаемся к PostgreSQL
        sudo -u postgres psql

        -- Создаем базу данных
        CREATE DATABASE appdb;

        \c appdb

        -- Таблица для записи (публикуем)
        CREATE TABLE test2 (
            id SERIAL PRIMARY KEY,
            data TEXT,
            source VARCHAR(50),
            created_at TIMESTAMP DEFAULT NOW()
        );

        -- Таблица для чтения (будем подписываться из ВМ1)
        CREATE TABLE test (
            id SERIAL PRIMARY KEY,
            data TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );
            
        -- Проверяем
        \dt
```

![Структура ВМ2](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/dt_vm_2.PNG)

2. Создание публикации для таблицы `test2`
```sql
        CREATE PUBLICATION pub_test2 FOR TABLE test2;
        SELECT * FROM pg_publication;  
```

![Публикация успешно создана](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_publication_vm_2.PNG)

**Результат:** создана публикация `pub_test2` для таблицы `test2`. Теперь изменения в этой таблице можно реплицировать на другие серверы.

### Взаимная подписка между ВМ1 и ВМ2
1. **На ВМ1**: Подписка на публикацию ВМ2 (`test2`)
```sql
        -- На ВМ1 выполняем:
        \c appdb

        -- Создаем подписку на ВМ2
        -- Указываем port от ВМ2
        CREATE SUBSCRIPTION sub_from_vm2
        CONNECTION 'host=192.168.0.42 port=5432 dbname=appdb user=replicator password=replicator'
        PUBLICATION pub_test2;

        -- Проверяем статус подписки
        SELECT subname, subenabled, subconninfo FROM pg_subscription;
        SELECT * FROM pg_stat_subscription;
```

![Подписка на публикацию ВМ успешна](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_subscription_vm1.PNG)  
![Подписка на публикацию ВМ успешна](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_stat_subscription_vm1.PNG)

2. **На ВМ2**: Подписка на публикацию ВМ1 (`test`)
```sql
        -- На ВМ1 выполняем:
        \c appdb

        -- Создаем подписку на ВМ1
        -- Указываем port от ВМ1
        CREATE SUBSCRIPTION sub_from_vm1
        CONNECTION 'host=192.168.0.41 port=5432 dbname=appdb user=replicator password=replicator'
        PUBLICATION pub_test;

        -- Проверяем статус подписки
        SELECT subname, subenabled, subconninfo FROM pg_subscription;
        SELECT * FROM pg_stat_subscription;
```

![Подписка на публикацию ВМ успешна](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_subscription_vm2.PNG)  
![Подписка на публикацию ВМ успешна](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_stat_subscription_vm2.PNG)

**Объяснение:**
- Теперь у нас двунаправленная логическая репликация
- Когда мы вставляем данные в `test` на ***ВМ1***, они реплицируются в `test` на ***ВМ2***
- Когда вставляю в `test2` на ***ВМ2***, они реплицируются в `test2` на ***ВМ1***

### Настройка ВМ3 как центра для чтения и бэкапов
1. Создать таблицы на ВМ3 (структура должна совпадать)
```sql
        -- Подключаемся к PostgreSQL
        sudo -u postgres psql

        -- Создаем базу данных
        CREATE DATABASE appdb;

        \c appdb

        -- Таблица для записи (публикуем)
        CREATE TABLE test (
            id SERIAL PRIMARY KEY,
            data TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );

        -- Таблица для чтения (будем подписываться из ВМ2)
        CREATE TABLE test2 (
            id SERIAL PRIMARY KEY,
            data TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );

        -- Проверяем
        \dt
```

![Структура ВМ3](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/dt_vm_3.PNG)

2. Подписка на обе публикации
```sql
        -- Подписка на ВМ1 (публикация test)
        -- Имена должны быть уникальными
        CREATE SUBSCRIPTION sub_vm3_from_vm1
        CONNECTION 'host=192.168.0.41 port=5432 dbname=appdb user=replicator password=replicator'
        PUBLICATION pub_test;

        -- Подписка на ВМ2 (публикация test2)
        -- Имена должны быть уникальными
        CREATE SUBSCRIPTION sub_vm3_from_vm2
        CONNECTION 'host=192.168.0.42 port=5432 dbname=appdb user=replicator password=replicator'
        PUBLICATION pub_test2;

        -- Проверяем обе подписки
        SELECT subname, subenabled, subconninfo FROM pg_subscription;
        SELECT * FROM pg_stat_subscription;
```

![Подписка на публикацию ВМ успешна](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_subscription_vm3.PNG)  
![Подписка на публикацию ВМ успешна](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_stat_subscription_vm3.PNG)


**Вывод:** ВМ3 теперь получает данные с обоих серверов и может использоваться для:
- Чтения объединенных данных
- Выполнения аналитических запросов
- Создания бэкапов без нагрузки на основные серверы

## Домашнее задание со звёздочкой (*)
***Настройка горячей физической репликации на ВМ4 (источник - ВМ3)***  
  
**На ВМ3**  
1. Открываем `postgresql.conf`:
```bash
        sudo nano /etc/postgresql/17/main/postgresql.conf
```

2. Изменяем параметры
```text
        wal_level = replica  # Меньше нагрузки чем logical, достаточно для физической репликации
        hot_standby = on
```

3. Сохранить файл и выйти в терминал

4. Отредактировать `pg_hba.conf`
```bash
        sudo nano /etc/postgresql/17/main/pg_hba.conf
```

5. Внести изменения (в самый конец файла)
```text
        host    replication     physical_replicator      192.168.0.44/32        scram-sha-256
```

6. Сохранить файл и выйти в терминал

7. Перезапускаем PostgreSQL
```bash
        sudo systemctl restart postgresql
```

8. Создаем пользователя для физической репликации
```bash
        sudo -u postgres psql -c "CREATE ROLE physical_replicator WITH REPLICATION LOGIN PASSWORD 'phys_repl_pass';"
```

**На ВМ4**  
1. Останавливаем PostgreSQL и очищаем data directory
```bash
        sudo systemctl stop postgresql
        sudo -u postgres rm -rf /var/lib/postgresql/17/main/*
```

2. Выполняем базовую копию с ВМ3
```bash
        sudo -u postgres pg_basebackup -h 192.168.0.43 -U physical_replicator -D /var/lib/postgresql/17/main -P -R -X stream --slot=slot_vm4 --create-slot
```

![Успех](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_basebackup.PNG)

**Объяснение параметров:**
- `-R`: Автоматически создать файл `standby.signal` и добавить `primary_conninfo` в `postgresql.auto.conf`
- `-X stream`: Потоковая передача WAL во время бэкапа
- `-C -S slot_vm4`: Создает слот репликации на источнике
- `-P`: Прогресс в реальном времени

3. Открываем `postgresql.conf`:
```bash
        # Настраиваем параметры реплики
        sudo nano /etc/postgresql/17/main/postgresql.conf
```

4. Запускаем PostgreSQL на ВМ4
```bash
        sudo systemctl start postgresql
```

5. Проверка физической репликации
```bash
        sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
        # Должно вернуть 't' (true) - сервер в режиме recovery/реплики
```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_14/pg_is_in_recovery_vm4.PNG)

## Тестирование всей системы
1. Тест логической репликации
```sql
        -- На ВМ1:
        INSERT INTO test (data) VALUES ('Данные с ВМ1');

        -- На ВМ2 проверяем:
        SELECT * FROM test;

        -- Результат:
         id |     data     |        created_at
        ----+--------------+---------------------------
          1 | Данные с ВМ1 | 2025-12-18 10:23:44.98369
        (1 row)

        -- На ВМ2:
        INSERT INTO test2 (data, source) VALUES ('Данные с ВМ2', 'vm2');

        -- На ВМ1 проверяем:
        SELECT * FROM test2;

        -- Результат:
         id |     data     | source |         created_at
        ----+--------------+--------+----------------------------
          1 | Данные с ВМ2 | vm2    | 2025-12-18 10:41:45.110948
        (1 row)

        -- На ВМ3 проверяем обе таблицы:
        SELECT * FROM test;
        SELECT * FROM test2;

        -- Результат: данные в таблицах есть
```

2. Тест физической репликации
```sql
        -- На ВМ3:
        CREATE TABLE test_replica_check (id SERIAL, check_time TIMESTAMP);
        INSERT INTO test_replica_check (check_time) VALUES (NOW());

        -- На ВМ4 (должна быть та же таблица и данные):
        SELECT * FROM test_replica_check;

        -- Результат:
         id |         check_time
        ----+----------------------------
          1 | 2025-12-18 10:52:54.775839
        (1 row)
```

**Вывод:** всё работает корректно