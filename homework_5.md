# Выполнение домашнего задания №5

## Основное домашнее задание

### Установка VirtualBox на Windows
1. Скачать "Windows hosts" (ссыслка: [VirtualBox](https://www.virtualbox.org/wiki/Downloads/))
2. Устновить VirtualBox

### Подготовка ISO-образа Linux
1. Скачать ISO-образ Ubuntu (ссылка: [Ubuntu](https://ubuntu.com/download/server#manual-install))

### Создание виртуальной машины в VirtualBox
1. Запустить VirtualBox
2. Нажать клавишу `Создать`
3. Заполнить поля:
    - **Имя:** ubuntu-postgres
    - **Образ ISO:** указал путь в скаченному iso-образу Ubuntu
4. Нажать клавишу `Далее`
5. Задать "Имя пользователя"
6. Задать и подтвердить пароль
7. Нажать клавишу `Далее`
8. В разделе "Оборудование" ничего не менял
9. Нажать клавишу `Далее`
10. Размер диска оставил "по умолчанию" - 20GB
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
### Обновить пакеты и установить Docker Engine
1. Обновить пакеты:
    ```bash
        sudo apt update
    ```

![Обновление пакетов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/apt_update.PNG)

2. Установка Docker и включение его в автоматический запуск сервиса при старте системы
    ```bash
        sudo apt install docker.io
        sudo systemctl enable --now docker
    ```

3. Проверить версию Docker
    ```bash
        docker --version
    ```

![Проверка, что Docker установлен](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/docker_version.PNG)

### Подготовить каталог для PostgreSQL
1. Создать каталог для данных БД:
    ```bash
        sudo mkdir -p /var/lib/postgresql/data
        sudo chmod 777 /var/lib/postgresql/data  # Упроcтил права для теста
    ```

![Создание каталога для данных БД](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/mkdir_postgresql_chmod_777.PNG)

### Задать дополнительные параметры для стабильной работы Docker'а

***Комментарии:*** этот раздел появился, так как постоянно получал при запуске контейнера ошибку **"TLS handshake timeout"**

1. Открыть конфигурационный файл docker'а
    ```bash
        sudo nano /etc/docker/daemon.json
    ```

2. Внести исправления:
```json
    {
    "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry-1.docker.io",
        "https://mirror.gcr.io",
        "https://dockerhub.azk8s.cn"
    ],
    "max-concurrent-downloads": 6,
    "max-concurrent-uploads": 4,
    "storage-driver": "overlay2"
    }
```

3. Применить изменения
    ```bash
        sudo systemctl daemon-reload
        sudo systemctl restart docker
    ```

4. Добавить пользователя в группу docker
    ```bash
        sudo usermod -aG docker user1
    ```

5. Применить изменения группы (без перезагрузки)
    ```bash
        newgrp docker
    ```

6. Проверить работу:
    ```bash
        docker info | grep -A 10 Mirrors
    ```

![Вывод списка зеркал](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/mirrors.PNG)

### Создать volume Docker
1. Скачать образ postgresql:17
    ```bash
        sudo docker pull postgres:17
    ```

![Скачивание успешно завершено](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/pull_postgresql_17.PNG)

2. Создать именованный volume Docker (так не будет конфликтов с системной postgres-папкой)
    ```bash
        sudo docker volume create pgdata
    ```

![Создан именованный volume Docker](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/volume_docker.PNG)

### Создадать директорию для конфига на ВМ
1. Выполнить команду
    ```bash
        mkdir -p ~/postgres-config
    ```

### Создадать и отредактировать файл конфигурации
1. Открыть файл конфигурации для редактирования
    ```bash
        sudo nano ~/postgres-config/postgresql.conf
    ```

2. Вставить следующее содержимое в открывшийся редактор

**Комментарии:** параметры подбирал исходя от моей ВМ

 ```text
    # --------------
    # CUSTOM CONFIG
    # --------------

    # CONNECTIONS AND AUTH
    listen_addresses = '*'          # Разрешаем подключение к СУБД с любого сетевого интерфейса, а не только с localhost. Это необходимо для тестирования извне контейнера/ВМ (например, через pgbench).
    max_connections = 100           # Pgbench может использовать много одновременных подключений (так как у меня RAM ~2ГБ)

    # RESOURCE USAGE (except WAL)
    shared_buffers = 512MB          # Есть правило - 25% от RAM (у меня 2048MB). Данная настройка позволит хранить большую часть активного набора данных в RAM, минимизируя медленные чтения с диска.
    huge_pages = off                # Упрощаем, выключаем сложную работу с huge pages. Читал, что для кратковременного теста проще и надежнее отключить, чтобы избежать потенциальных проблем.
    work_mem = 16MB                 # Память на операции сортировки/хэширования для одного запроса. Значение подобрано так, чтобы несколько таких операций могли выполняться параллельно, не исчерпывая всю память.
    maintenance_work_mem = 256MB    # Память на операции техобслуживания (VACUUM, CREATE INDEX). Эти операции начинают выполняться в разы быстрее, так как могут проходить большие объемы данных в памяти.
    autovacuum = off                # ВЫКЛЮЧАЕМ autovacuum! Это чисто тестовая настройка.
                                    # Autovacuum критически важен для работы реальной БД, но во время теста он потребляет ресурсы и может помешать замерам.

    # WRITE-AHEAD LOG - НАСТРОЙКИ, ДАЮЩИЕ МАКСИМУМ ПРОИЗВОДИТЕЛЬНОСТИ
    fsync = off                     # САМОЕ ГЛАВНОЕ. Отключает принудительную запись данных на диск. Операционная система будет сама решать, когда записывать данные. 
                                    # Это должно дать большой прирост производительности, но при любом сбое питания или ОС все данные, не сброшенные на диск, будут утеряны безвозвратно.
    synchronous_commit = off        # Транзакция считается завершенной ДО реальной записи в WAL, т.е. сразу после записи в буфер в RAM.
    full_page_writes = off          # Отключает защиту от порчи данных при сбое во время записи. Это еще один уровень защиты, который замедляет операции записи.
    wal_buffers = 16MB              # Буфер для WAL. Увеличиваем. Больший буфер позволяет накапливать больше данных перед синхронизацией.
    max_wal_size = 4GB              # Макс. размер WAL-файлов.
    min_wal_size = 1GB              # Мин. размер WAL-файлов.

    # BACKGROUND WRITER
    bgwriter_delay = 10ms           # Чаще отдавать "грязные" страницы на запись.
    bgwriter_lru_maxpages = 500     # Макс. число страниц, которые bgwriter может записать за раз.
    bgwriter_lru_multiplier = 5.0   # Более агрессивная работа bgwriter.
                                    # Параметры bgwriter_lru_maxpages и bgwriter_lru_multiplier делает работу фонового писателя более агрессивной. Он будет стараться выгружать больше страниц за один цикл, активнее помогая основному процессу с записью данных.

    # QUERY TUNING
    random_page_cost = 1.1          # Понижаем т.к. используем SSD (в виртуальной среде). Понижение значения заставляет планировщик запросов чаще выбирать планы, использующие индексы и случайный доступ, что обычно правильнее для быстрых дисков.
    effective_io_concurrency = 200  # Подразумеваем, что подсистема диска может выполнять множество операций параллельно (как SSD).
    max_worker_processes = 2        # Маскимальное число фоновых процессов.
    max_parallel_workers_per_gather = 1 # Отключаем параллелизм для экономии RAM.
    max_parallel_workers = 1        # Отключаем параллелизм.

    # LOGGING
    logging_collector = off         # Упрощаем, выключаем сбор логов в файлы для экономии IO.
    log_statement = 'none'          # Не логируем запросы.
    log_duration = off
    log_lock_waits = off
 ```

![Редактируем config-файл](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/custom_config.PNG)

### Запуск контейнера PostgreSQL 17 с отреддактированной конфигурацией
1. Запускаем новый контейнер, монтируя наш конфиг и volume с данными
    ```bash
        sudo docker run --name postgres-server -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=testdb -v pgdata:/var/lib/postgresql/data -v /home/user1/postgres-config/postgresql.conf:/etc/postgresql/postgresql.conf -p 5432:5432 -d postgres:17 -c 'config_file=/etc/postgresql/postgresql.conf'
    ```

**Комментарии:** ключевые моменты здесь - это монтирование тома с данными (pgdata) для сохранности между перезапусками и монтирование отредактированного конфига. Параметр `-c` в конце команды указывает запущенному PostgreSQL использовать наш отредактированный файл вместо стандартного.

![Запуск произошёл успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/conteiner_docker_custom_config.PNG)

**Комментарии:** если бы был запущен другой контейнер на ВМ, то его необходимо было остановить и удалить
   ```bash
        # Останавливаем и удаляем старый контейнер
        sudo docker stop postgres-server 2>/dev/null || true
        sudo docker rm postgres-server 2>/dev/null || true
   ```

2. Проверить, что контейнер запустился
    ```bash
        sudo docker ps
    ```

![Видим запущенный контейнер](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/sudo_docker_ps.PNG)

### Подготовка тестовой базы данных для pgbench

**Комментарии:** необходимо создать отдельную базу и инициализировать в ней схему, которую будет использовать pgbench.

1. Создаем БД с именем `benchmark`
    ```bash
        sudo docker exec -it postgres-server psql -U postgres -c "CREATE DATABASE benchmark;"
    ```

![База данных успешно создалась](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/create_db_benchmark.PNG)

2. Инициализируем схему `pgbench` в этой БД с масштабом 100

**Комментарии:** это должно создать ~15ГБ данных

```bash
        # -i = инициализация
        # -s 100 = коэффициент масштабирования (100 примерно = 15ГБ)
        sudo docker exec -it postgres-server pgbench -U postgres -i -s 100 benchmark
 ```

![Получаем рузельтат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/result_i.PNG)

**Комментарии:** масштаб -s 100 выбран, чтобы данные не помещались в оперативную память и частично считывались с диска, что делает тест, как мне кажется, более реалистичным. Время инициализации (~79 секунд) — это достаточно хороший результат для такого объема.

### Проведение нагрузочного теста и фиксация результата TPS
1. Запускаем основной тест. Будем использовать 10 виртуальных клиентов, 2 рабочих потока и длительность теста 60 секунд.
    ```bash
        sudo docker exec -it postgres-server pgbench -U postgres -T 60 -c 10 -j 2 benchmark
    ```

![Получаем финальный рузельтат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/result_final.PNG)

**Результат:** 3908 tps

## Домашнее задание со звёздочкой

### Шаг 1. Заходим в контейнер
   ```bash
        sudo docker exec -it postgres-server bash
   ```

![Зашли в контейнер](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/root.PNG)

### Шаг 2. Устанавливаем sysbench (внутри контейнера)
   ```bash
        apt update && apt install -y curl gnupg2
        curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | bash
        apt install -y sysbench
   ```

### Шаг 3. Выходим из контейнера
   ```bash
        exit
   ```

### Шаг 4. Проверяем, что sysbench установлен
   ```bash
        sudo docker exec -it postgres-server bash -c "sysbench --version"
   ```

![Отобразилась версия sysbench](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/sysbench_version.PNG)

### Шаг 4. Создаем базу данных для sysbench
   ```bash
        sudo docker exec -it postgres-server psql -U postgres -c "CREATE DATABASE tpcc;"
   ```

### Шаг 5. Готовим схему (внутри контейнера)

**Комментарии:** это займет время

   ```bash
        sudo docker exec -it postgres-server bash -c "
        sysbench oltp_read_write \
        --db-driver=pgsql \
        --pgsql-user=postgres \
        --pgsql-db=tpcc \
        --pgsql-password=postgres \
        --table-size=1000000 \
        --tables=10 \
        --threads=4 \
        prepare"
   ```

### Шаг 6. Запускаем тест на 60 секунд
   ```bash
        sudo docker exec -it postgres-server bash -c "
        sysbench oltp_read_write \
        --db-driver=pgsql \
        --pgsql-user=postgres \
        --pgsql-db=tpcc \
        --pgsql-password=postgres \
        --table-size=1000000 \
        --tables=10 \
        --threads=10 \
        --time=60 \
        --report-interval=2 \
        run"
   ```

![Получаем финальный рузельтат sysbench](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_5/sysbench_result_final.PNG)

**Результат:** transactions: 660 (это и есть tps)
sysbench показал меньше результат, чем pgbench - это ожидаемый результат
3908 TPS (pgbench) ÷ 660 TPS (sysbench) ≈ 6 раз

Это полностью укладывается в ожидаемые рамки, так как обычно разница составляет от 5 до 10 раз
