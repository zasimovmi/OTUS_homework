# Выпонение домашнего задания №8

## Основное домашнее задание

### Установка VirtualBox на Windows
1. Скачать "Windows hosts" (ссыслка: [VirtualBox](https://www.virtualbox.org/wiki/Downloads/))
2. Устновить VirtualBox

### Подготовка ISO-образа Linux
1. Скачать ISO-образ Debian (ссылка: [Debian](https://www.debian.org/CD/netinst/))
***Комментарии:*** *использовал Netinst-образ, так как "весит" меньше и при установке всё "докачается"*

### Создание виртуальной машины в VirtualBox
1. Запустить VirtualBox
2. Нажать клавишу `Создать`
3. Заполнить поля:
    - **Имя:** debian-postgres-3
    - **Образ ISO:** указал путь в скаченному iso-образу Debian
4. Нажать клавишу `Далее`
5. Задать "Имя пользователя"
6. Задать и подтвердить пароль
7. Нажать клавишу `Далее`
8. В разделе "Оборудование" установить
    - **RAM** - 4GB
    - **Процессоры** - 2
9. Нажать клавишу `Далее`
10. Размер диска оставил "по умолчанию" - 10GB
11. Нажать клавишу `Далее`
12. Нажать клавишу `Готово`
13. Дождаться установки
14. Выбрать созданную виртуальную машину (далее - ВМ)
15. Нажать клавишу `Настроить`
16. Перейти в раздел "Носители"
17. Найти "Контроллер: IDE"
18. Проверить, что диск пустой
19. Зайти в раздел "Сеть"
20. Убедиться, что тип подключения установлен как **NAT**
21. Нажать клавишу `ОК`

### Создать правило для ssh подключения
1. Нажать клавишу `Настроить`
2. Зайти в раздел "Сеть"
3. Нажать клавишу `Проброс портов`
4. Нажать `Добавить правило` (зелёный плюсик справа)
5. Установить следующие значения:
    - **Имя:** ssh-debian-3
    - **Протокол:** TCP
    - **Адрес хоста:** 127.0.0.1
    - **Порт хоста:** 6022
    - **Адрес гостя:** 10.0.2.15
    - **Порт гостя:** 22

### Первоначальная настройка ВМ
1. Выбрать ВМ
2. Нажать клавишу `Запустить`
3. Следовать мастеру устновки
4. Установка произошла успешно
5. Войти под созданным пользователем
6. Зайти в терминал
7. Доавление своего пользователя в группу sudo:
    1. Ввести комманду
    ```bash
        su -
    ```
    2. Ввести пароль
    3. Выполнить команду
    ```bash
        usermod -aG sudo user1
    ```
    4. Проверить, что мой user1 добавлен в группу
    ```bash
        getent group sudo
        # Получил подтверждение: sudo:x:27:user1
    ```
    5. Сменить пользователя
    ```bash
        su - user1
    ```

### Проверить, установлен и запущен ли SSH-сервер
1. Проверить, установлен ли openssh-server
    ```bash
        sudo systemctl status ssh
    ```

2. Если он не установлен (получим сообщение о том, что сервис не найден), установить его
    ```bash
        sudo apt update && sudo apt install -y openssh-server
    ```

3. После установки явно его запустить и добавь в автозагрузку
    ```bash
        sudo systemctl enable --now ssh
    ```

4. Еще раз проверить статус. 
    ```bash
        sudo systemctl status ssh
    ```

**Комментарии:** статус 'active (running)'

### Установка PosrgreSQl и проверка работы кластера
1. Установить PostgreSQL
    1. Выполнить команды
    ```bash
        sudo apt update && sudo apt upgrade -y
        sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
        wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
        sudo apt update
        sudo apt install -y postgresql-17 postgresql-contrib-17
    ```

    2. Проверяем установку
    ```bash
        sudo systemctl status postgresql
    ```

![Проверка пройдена успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/status_postgres.PNG)

### Настройка PostgreSQL
1. Зайти в psql
    ```bash
        sudo -u postgres psql
    ```

2. Создаем базу данных для тестов
    ```sql
        CREATE DATABASE test_db;
    ```

3. Создаём тестовую таблицу
    ```sql
        CREATE TABLE test_table (
            id SERIAL PRIMARY KEY,
            value INTEGER
        );
    ```

4. Наполняем данными тестовую таблицу
    ```sql
        INSERT INTO test_table (value) VALUES (10), (20), (30);
    ```

![Все действия выполнены успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/create_insert_success.PNG)

### Настройка логирования долгих блокировок
1. Проверка и настройка параметров
    ```sql
        SHOW log_lock_waits;
        SHOW deadlock_timeout;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/show_before.PNG)

**Результат:**
* log_lock_waits = `off`
* deadlock_timeout = `1s`

2. Включаем логирование и уменьшаем таймаут
Так как необходимо логировать блокировки > 200 мс нужно установить `deadlock_timeout` в значение больше 200 мс (этот параметр - минимальное время перед проверкой взаимоблокировки, и он же используется для log_lock_waits). Установим 200ms.

    ```sql
        ALTER SYSTEM SET log_lock_waits = on;
        ALTER SYSTEM SET deadlock_timeout = '200ms';
    ```

3. Применяем изменения
    ```sql
        -- Перезагружаем конфиг без перезапуска сервера
        SELECT pg_reload_conf();
    ```

4. Убедимся, что настройки применились
    ```sql
        SHOW log_lock_waits;
        SHOW deadlock_timeout;
    ```

![Получаем результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/show_after.PNG)

**Результат:**
* log_lock_waits = `on`
* deadlock_timeout = `200ms`

### Моделирование блокировок от трех UPDATE
1. Выполняем команды
***Открываем три окна (три сеанса)***

**Сеанс 1**
    ```sql
        BEGIN;
        UPDATE test_table SET value = value + 1 WHERE id = 1;
    ```

![Сессия 1](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_session_1.PNG)

**Результат:** запись заблокирована транзакцией 1.
UPDATE выполнился успешно - это нормально ожидаемый результат, но транзакция не завершена (нет COMMIT или ROLLBACK).
Это значит, что блокировки удерживаются.

**Сеанс 2**
    ```sql
        BEGIN;
        UPDATE test_table SET value = value + 1 WHERE id = 1;
    ```

![Сессия 2](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_session_2.PNG)

**Результат:** эта команда "зависла", ожидая завершения транзакции в Сеансе 1.

**Сеанс 3**
    ```sql
        BEGIN;
        UPDATE test_table SET value = value + 1 WHERE id = 1;
    ```

![Сессия 3](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_session_3.PNG)

**Результат:** эта команда также "зависла", встав в очередь за Сеансом 2.

![Все три сессии](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_session_1_2_3.PNG)

2. Анализ блокировок в pg_locks
***Откроем четвертое окно (Сеанс 4) и выполните запрос к представлению `pg_locks`.***

**Сеанс 4**
1) Выполнить запрос
    ```sql
        SELECT 
            l.pid,                -- ID процесса
            l.locktype,           -- Тип блокировки
            l.mode,               -- Режим блокировки
            l.granted,            -- Выдана ли блокировка (true/false)
            l.relation::regclass, -- Имя таблицы (если применимо)
            l.page,               -- Номер страницы (если применимо)
            l.tuple,              -- Номер строки (если применимо)
            l.virtualxid,         -- Виртуальный ID транзакции
            l.virtualtransaction, -- Виртуальная транзакция
            a.backend_xid as xid  -- ID транзакции (если применимо)
        FROM pg_locks l
        LEFT JOIN pg_database d ON l.database = d.oid
        LEFT JOIN pg_stat_activity a ON l.pid = a.pid
        WHERE d.datname = 'test_db' -- Смотрим только наши блокировки
            OR l.virtualxid IS NOT NULL -- И виртуальные транзакции
        ORDER BY l.pid, l.granted;
    ```

![Сессия 4](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_session_4.PNG)

2) Выполнить запрос для нашей таблицы
    ```sql
        SELECT 
            pid,
            locktype,
            mode,
            granted,
            relation::regclass as table_name
        FROM pg_locks 
        WHERE relation = 'test_table'::regclass;
    ```

![Сессия 4, таблица test_table](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_session_4_test_table.PNG)

**Вывод:** 
Все три процесса (1259, 1409, 1414) имеют блокировки RowExclusiveLock на таблицу test_table с granted = true. Это нормально - каждый UPDATE берет такую блокировку на таблицу.

Ключевой момент - блокировки на уровне строк (tuple):
* Процесс 1409: tuple ExclusiveLock с granted = true
  Это транзакция, которая удерживает блокировку на строку
* Процесс 1414: tuple ExclusiveLock с granted = false
  Это транзакция, которая ждет блокировку на ту же строку

***Что сейчас происходит***:
* Процесс 1409 выполнил UPDATE и удерживает блокировку строки.
* Процесс 1414 пытается выполнить UPDATE той же строки и ждет, пока 1409 освободит блокировку.
* Процесс 1259 тоже имеет транзакцию, но пока не пытается блокировать эту строку (или работает с другой строкой).

3) Проверим, кто что делает
    ```sql
        SELECT 
            pid,
            query,
            state,
            wait_event_type,
            wait_event
        FROM pg_stat_activity 
        WHERE pid IN (1259, 1409, 1414);
    ```

![Просмотр реультатов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_session_4_result.PNG)

**Вывод:**
***Цепочка блокировок:***
    ```text
        1259 (idle) ──╮
                      │
        1409 (active) │ Ждет 1259
                      │
        1414 (active) │ Ждет 1409 (который ждет 1259)
    ```

***Что произошло:***
* 1259 первым сделал UPDATE и "завис" в idle in transaction
* 1409 попытался сделать UPDATE той же строки, но встал в очередь за 1259
* 1414 попытался сделать UPDATE и встал в очередь за 1409


3. Проверка логов
* В Сеансе 4 выходим в треминал
    ```sql
        \q
    ```

* Открываем логи
    ```bash
        sudo nano /var/log/postgresql/postgresql-17-main.log
    ```

![Просмотр логов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_log.PNG)

**Вывод:** изучая журнал сообщений постфактум можно полностью разобраться в ситуации блокировок.

4. Завершим все транзакции
    ```sql
        --Во всех сеансах (1-3) выполнить команду поочерёдно, начиная с первого
        COMMIT;
    ```

![Во всех трёх сеансах update выполнен успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_final.PNG)

5. Закрываем все сеансы (кроме 1)

### Воспроизведение взаимоблокировки (Deadlock)
1. Вернем наши данные в исходное состояние
    ```sql
        UPDATE test_table SET value = 10 WHERE id = 1;
        UPDATE test_table SET value = 20 WHERE id = 2;
    ```

![UPDATE прошёл успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_rollback.PNG)

2. Запускаем три транзакции
***Открываем три окна (три сеанса)***

**Сеанс 1**
    ```sql
        BEGIN;
        UPDATE test_table SET value = 100 WHERE id = 1;
    ```

**Результат:** удерживает блокировку строки id=1

**Сеанс 2**
    ```sql
        BEGIN;
        UPDATE test_table SET value = 200 WHERE id = 2;
    ```

**Результат:** удерживает блокировку строки id=2

**Сеанс 3**
    ```sql
        BEGIN;
        UPDATE test_table SET value = 300 WHERE id = 3;
    ```

**Результат:** удерживает блокировку строки id=3

3. Создаем цикл ожидания
**Сеанс 1**
    ```sql
        UPDATE test_table SET value = 200 WHERE id = 2;
    ```

**Результат:** эта команда заблокировалась, ожидая завершения Сеанса 2.

**Сеанс 2**
    ```sql
        UPDATE test_table SET value = 300 WHERE id = 3;
    ```

**Результат:** эта команда заблокировалась, ожидая завершения Сеанса 3.

**Сеанс 3**
    ```sql
        UPDATE test_table SET value = 100 WHERE id = 1;
    ```

**Результат:** Сеанс 3 пытается взять блокировку, которую удерживает Сеанс 1.
Но Сеанс 1 ждет Сеанс 2, а Сеанс 2 ждет Сеанс 3.
Образуется цикл: 1 -> 2 -> 3 -> 1.

**Вывод:** через ~200 мс (наш deadlock_timeout) PostgreSQL обнаружил deadlock. Он выбрал одну из транзакций как "жертву" и прервал её.

![Цикл завершился ошибкой](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/loop_error.PNG)

4. Проверка логов
В новом Сеансе открываем логи
    ```bash
        sudo nano /var/log/postgresql/postgresql-17-main.log
    ```

![Просмотр логов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/deadlock_log.PNG)

**Вывод:** изучая журнал сообщений постфактум можно полностью разобраться в ситуации взаимоблокировок.

## Домашнее задание со звёздочкой (*)
***Тут же отвечу на вопрос: могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?***

**Сеанс 1**
    ```sql
        -- Транзакция A использует последовательное сканирование
        SET enable_indexscan = off;
        UPDATE deadlock_test SET value = value + 1;
    ```

**Сеанс 2**
    ```sql
        -- Транзакция B использует индексное сканирование  
        SET enable_seqscan = off;
        UPDATE deadlock_test SET value = value - 1;
    ```

![Видим ошибку](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_8/update_without_where.PNG)


**Вывод:** хотя команда UPDATE без WHERE сканирует и блокирует таблицу последовательно, взаимоблокировка возможна из-за особенностей порядка сканирования и конкурентных изменений. Вероятность такого события крайне мала на маленьких таблицах с последовательным сканированием, но теоретически возможна на больших таблицах со сложными планами запросов. Это показывает, что даже кажущиеся "атомарными" операции могут привести к взаимоблокировке при высокой конкуренции.
