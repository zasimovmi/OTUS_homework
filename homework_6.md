# Выпонение домашнего задания №6

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

![Проверка пройдена успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/status_postgres.PNG)

### Настройка и тестирование PostgreSQL
1. Зайти в psql
    ```bash
        sudo -u postgres psql
    ```

2. Создаем базу данных для тестов
    ```sql
        CREATE DATABASE test_db;
    ```

3. Выходим в терминал
    ```sql
        \q
    ```

4. Инициализируем pgbench
    ```bash
        sudo -u postgres pgbench -i test_db
    ```

5. Запускаем первый тест pgbench
    ```bash
        sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres test_db
    ```

**Комментарии:** 
- Параметр `-c8` - это количество клиентских подключений (8 одновременных подключений к БД)
- Параметр `-P 6` - это интервал отчетов о прогрессе (выводит отчет каждые 6 секунд)
- Параметр `-T 60` - это продолжительность теста (1 минута)

![Получаем первый результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/pgbench_first_test.PNG)

**Результат:** ~1420 tps

### Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
1. Сделать бэкап исходного конфига
    ```bash
        sudo cp /etc/postgresql/17/main/postgresql.conf postgresql.conf.backup
    ```

2. Убедиться, что бэкап прошёл успешно
    ```bash
        ls -la postgresql.conf.backup
    ```

![Бэкап конфига успешно сделан](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/backup_config.PNG)

3. Открыть postgresql.conf
    ```bash
        sudo nano /etc/postgresql/17/main/postgresql.conf
    ```

4. Внести изменения (ниже приведены параметры, которые были изменены)
    ```text
        # CONNECTIONS AND AUTHENTICATION
        max_connections = 40

        # RESOURCE USAGE (except WAL)
        shared_buffers = 1GB
        work_mem = 6553kB
        maintenance_work_mem = 512MB
        effective_io_concurrency = 2

        # WRITE AHEAD LOG
        wal_buffers = 16MB
        min_wal_size = 4GB
        max_wal_size = 16GB
        checkpoint_completion_target = 0.9

        # QUERY TUNING
        random_page_cost = 4
        effective_cache_size = 3GB
        default_statistics_target = 500
    ```

5. Сохранить файл и выйти в терминал

### Запуск pgbench после изменений в postgresql.conf
1. Перезапускаем PostgreSQL для применения настроек
    ```bash
        sudo systemctl restart postgresql
    ```

2. Запустить второй тест pgbench
    ```bash
        sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres test_db
    ```

![Получаем второй результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/pgbench_second_test.PNG)

**Результат:** ~885 tps

**Выводы:** Ожидаемое и логичное поведение при таких вводных. 
Это не ошибка, а прямое следствие применения настроек, которые категорически не подходят для данного "железа".
Можно сказать, что полученный результат - это наглядная демонстрация принципа "Плохие настройки хуже настроек по умолчанию"
Самый главный "вредитель" - это параметр `random_page_cost = 4` (эта настройка верна для HDD, а не для SSD)

### Создание и наполнение таблицы
1. Подключиться к БД `test_db`
    ```bash
        sudo -u postgres psql -d test_db
    ```

2. Создать таблицу
    ```sql
        CREATE TABLE test_table(
            id serial PRIMARY KEY,
            text_data text
        );
    ```

![Таблица создана успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/create_table_test_table.PNG)

3. Заполнить таблицу 1 млн строк случайными данными
    ```sql
        INSERT INTO test_table(text_data)
        SELECT md5(random()::text) -- Генерируем случайную MD5-хэш строку
        FROM generate_series(1, 1000000);
    ```

![Вставка строк произошла успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/insert_1_mln.PNG)

### Проверить размер файла таблицы
1. Выполнить команду
    ```sql
        SELECT pg_size_pretty(pg_relation_size('test_table'));
    ```

![Просмотр результатов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/first_size_test_table.PNG)

**Результат:** 65MB

### Первые пять обновлений, мёртвые строки и автовакуум
1. Обновить 5 раз все строки, добавляя символ `А`
    ```sql
        DO $$
        BEGIN
		FOR i IN 1..5 LOOP
			UPDATE test_table SET text_data = text_data || 'A';
		END LOOP;
	END $$;
    ```

2. Смотрим количество мертвых строк и время последнего автовакуума
    ```sql
	SELECT
  		relname,
  		n_live_tup,
  		n_dead_tup,
  		last_autovacuum
	FROM pg_stat_user_tables
	WHERE relname = 'test_table';
    ```

**Результат:** видим большое число n_dead_tup и NULL в last_autovacuum, так как автовакуум еще не сработал.

3. Ждем и периодически проверяем, пришел ли автовакуум
    ```sql
	-- Выполняем этот запрос несколько раз с интервалом в 10-20 секунд
	SELECT
  		relname,
  		n_live_tup,
  		n_dead_tup,
  		last_autovacuum
	FROM pg_stat_user_tables
	WHERE relname = 'test_table';
    ```

![Результат с автовакуумом](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/first_loop_5.PNG)

**Результат:** n_dead_tup "обнулился", а в last_autovacuum появилась timestamp. 
Это значит автовакуум отработал.

### Вторые пять обновлений и проверка размера таблицы
1. Обновить 5 раз все строки, добавляя символ `B`
    ```sql
        DO $$
        BEGIN
		FOR i IN 1..5 LOOP
			UPDATE test_table SET text_data = text_data || 'B';
		END LOOP;
	END $$;
    ```

2. Смотрим размер файла таблицы после автовакуума
    ```sql
        SELECT pg_size_pretty(pg_relation_size('test_table'));
    ```

![Просмотр результатов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/second_size_test_table.PNG)

**Результат:** 438MB
Размер таблицы увеличился (с 65 MB до 438 MB).
Это произошло потому, что каждое обновление создает новую версию строки. 
Автовакуум очистил старые ("мертвые") версии, но новые версии строк, которые мы только что создали, являются "живыми" и занимают место.
Кроме того, из-за добавления символов сами строки стали длиннее.

### Отключаем автовакуум для конкретной таблицы
1. Выполнить команду
    ```sql
        ALTER TABLE test_table SET (autovacuum_enabled = false);
    ```

2. Проверить, что автовакуум выключен для конкретной таблицы
	```sql
		SELECT relname, reloptions 
		FROM pg_class 
		WHERE relname = 'test_table';
	```

![Автовакуум выключен для test_table](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/autovacuum_false.PNG)

### Обновить десять раз все строки, добавляя символ `C`
1. Выполнить команду
	```sql
        DO $$
        BEGIN
		FOR i IN 1..10 LOOP
			UPDATE test_table SET text_data = text_data || 'C';
		END LOOP;
	END $$;
	```

![Заканчивается место](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/error_memory.PNG)

**Выводы:** 
- Я отключил автовакуум (autovacuum_enabled = false)
- Запустил 10 циклов обновления. Каждое обновление создавало новую версию для каждой из 1 млн строк.
- Вместо того чтобы очищать старые, ненужные версии строк (что и делает автовакуум), PostgreSQL был вынужден записывать все новые и новые данные на диск.
- Размер таблицы стремительно рос
- Диск заполнился до предела
- Когда процесс PostgreSQL попытался записать очередную порцию данных, ему не хватило места. Это критическая ошибка, которая привела к аварийному завершению этого процесса
- Чтобы избежать повреждения данных, главный процесс PostgreSQL (postmaster) принудительно завершил все соединения с базой данных и перевел кластер в режим восстановления

### Дальнейшие шаги
1. Перезойти в терминал

2. Посмотреть, где занято место
	```bash
		df -h
	```

![Просмотр свободного места](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/df_h.PNG)

3. Ввести команду
	```bash
		sudo -u postgres /usr/lib/postgresql/17/bin/postgres --single -D /var/lib/postgresql/17/main -c config_file=/etc/postgresql/17/main/postgresql.conf test_db
	```
---
**Объяснение:** мне необходим было запустить VACUUM в обход системы.
Так как обычный psql не подключится, то я должен запустить PostgreSQL в однопользовательском режиме. 
Это специальный режим для восстановления, который запускает только ядро СУБД без фоновых процессов и позволяет выполнить критически важные команды.

По хорошему я должен выполнить данную команду:
	```bash
		sudo -u postgres postgres --single -D /var/lib/postgresql/17/main test_db
	```

Но будет ошибка (будет жаловаться на второе "postgres"), так как система не сможет найти исполняемый файл postgres.
Это стандартная утилита для запуска PostgreSQL, и она должна быть в одной из директорий, указанных в переменной PATH.
Мне необходимо сразу указать полный путь к бинарному файлу.
Для этого я выаолнил сразу команду:
	```bash
		sudo find / -name postgres -type f -executable 2>/dev/null
	```

У меня был один мне нужный путь `/usr/lib/postgresql/17/bin/postgres`
Так как файл конфигурации у меня лежит не в /var/lib/, а в /etc/, то я сразу прописал к нему путь

---

![Успех](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/single_session.PNG)

4. Вводим команду
	```sql
		VACUUM FULL VERBOSE test_table;
	```

5. Выйти в терминал (сочетание клавиш `Ctrl+D`)

6. Подключиться к БД `test_db`
    ```bash
        sudo -u postgres psql -d test_db
    ```

![Успех](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/enter.PNG)

7. Проверить опять размер таблицы
    ```sql
        SELECT pg_size_pretty(pg_relation_size('test_table'));
    ```

![Смотрим результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/after_vacuum.PNG)

**Результат:** 78MB
Ожидаемый результат

8. Включаем автовакуум для test_table обратно
    ```sql
        ALTER TABLE test_table SET (autovacuum_enabled = true);
    ```

### Итоговое объяснение результата
**Без автовакуума:** при многократном обновлении данных в таблице накапливается огромное количество "мертвых" строк (старых версий).
Это приводит к сильному "раздуванию" размера таблицы на диске, замедлению запросов (им приходится читать больше данных) и общему падению производительности СУБД.

**С автовакуумом:** процесс автовакуума автоматически находит такие таблицы и очищает их, освобождая пространство, занимаемое "мертвыми" строками.
Однако, он не возвращает пространство операционной системе (если не используется VACUUM FULL), а лишь помечает его как готовое для переиспользования внутри таблицы.
Поэтому после цикла обновлений и работы автовакуума размер таблицы все равно будет больше исходного, но не катастрофически.

## Домашнее задание со звёздочкой (*)
    ```sql
        -- Анонимная процедура с выводом номера шага
	DO $$
	BEGIN
  		FOR update_cycle IN 1..10 LOOP
    			RAISE NOTICE 'Цикл обновления №%', update_cycle;
    			UPDATE test_table SET text_data = text_data || 'Z';
  		END LOOP;
	END $$;
    ```


![Смотрим результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_6/do.PNG)
