# Выполнение домашнего задания №4

## Установка VirtualBox на Windows
1. Скачать "Windows hosts" (ссыслка: [VirtualBox](https://www.virtualbox.org/wiki/Downloads/))
2. Устновить VirtualBox

## Подготовка ISO-образа Linux
1. Скачать ISO-образ Ubuntu (ссылка: [Ubuntu](https://ubuntu.com/download/server#manual-install))

## Создание виртуальной машины в VirtualBox
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

## Первоначальная настройка ВМ
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
## Обновить пакеты и установить Docker Engine
1. Обновить пакеты:
    ```bash
        sudo apt update
    ```

![Обновление пакетов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/apt_update.PNG)

2. Установка Docker и включение его в автоматический запуск сервиса при старте системы
    ```bash
        sudo apt install docker.io
        sudo systemctl enable --now docker
    ```

3. Проверить версию Docker
    ```bash
        docker --version
    ```

![Проверка, что Docker установлен](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/docker_version.PNG)

## Подготовить каталог для PostgreSQL
1. Создать каталог для данных БД:
    ```bash
        sudo mkdir -p /var/lib/postgresql/data
        sudo chmod 777 /var/lib/postgresql/data  # Упроcтил права для теста
    ```

![Создание каталога для данных БД](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/mkdir_postgresql_chmod_777.PNG)

## Задать дополнительные параметры для стабильной работы Docker'а

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

![Вывод списка зеркал](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/mirrors.PNG)

## Запустить контейнер PostgreSQL
1. Скачать образ postgresql:17
    ```bash
        sudo docker pull postgres:17
    ```

![Скачивание успешно завершено](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/pull_postgresql_17.PNG)

2. Создать именованный volume Docker (так не будет конфликтов с системной postgres-папкой)
    ```bash
        sudo docker volume create pgdata
    ```

![Создан именованный volume Docker](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/volume_docker.PNG)

3. Запустить контейнер с монтированием каталога
    ```bash
        sudo docker run --name postgres-server -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=testdb -v pgdata:/var/lib/postgresql/data -p 5432:5432 -d postgres:17
    ```

![Запуск произошёл успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/conteiner_docker.PNG)

## Развернуть контейнер с клиентом PostgreSQL и подключиться к серверу
1. Запуск клиента с подключением к серверу
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U postgres -d testdb
    ```

![Запуск произведён успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/start_client.PNG)

## Создать схему testnm
1. Выполнить команду
    ```sql
        -- Создание схемы
        CREATE SCHEMA testnm;
    ```

![Схема успешно создалась](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/create_schema_testnm.PNG)

## Создать таблицу t1 и наполнить данными
1. Выполнить команды
    ```sql
        CREATE TABLE t1 (c1 integer);
        INSERT INTO t1 VALUES (1);
    ```

![Таблица создана и наполнена данными](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/create_table_t1_insert.PNG)

## Создать роль readonly
1. Выполнить команду
    ```sql
        CREATE ROLE readonly;
    ```

![Роль создана](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/create_role_readonly.PNG)

## Назначить права роли readonly
1. Выполнить команды
    ```sql
        GRANT CONNECT ON DATABASE testdb TO readonly;
        GRANT USAGE ON SCHEMA testnm TO readonly;
        GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
    ```

![Права выданы](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/grant_readonly.PNG)

## Создать пользователя testread
1. Выполнить команды
    ```sql
        CREATE USER testread WITH PASSWORD 'test123';
        GRANT readonly TO testread;
    ```

![Пользователь создан](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/create_role_testread.PNG)

## Подключиться под testread
1. Выйти в терминал
    ```sql
        \q
    ```

2. Подключиться под пользователем testread
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U testread -d testdb
    ```

![Подключение произошло под пользователем testread](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/psql_testread_first_enter.PNG)

## Попытка SELECT
    ```sql
        SELECT * FROM t1;
    ```

![Ошибка](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/error_permission_denied.PNG)

**Результат:** ошибка - таблица t1 не найдена.
**Объяснение:** таблица t1 была создана без явного указания схемы, поэтому она попала в схему public, а не testnm. Права мы давали только для схемы testnm.

## Просмотр списка таблиц
1. Выполнить команду
    ```sql
        \dt
    ```

![Список таблиц](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/dt.PNG)

**Комментарии:** видим, что таблица t1 находится в схеме public, а не testnm.

## Удаление и пересоздание таблицы с указанием схемы
1. Выйти в терминал
    ```sql
        \q
    ```

2. Подключиться под пользователем postgres
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U postgres -d testdb
    ```

3. Выполнить команды
    ```sql
        DROP TABLE t1;
        CREATE TABLE testnm.t1 (c1 integer);
        INSERT INTO testnm.t1 VALUES (1);
    ```

![Таблица создалась и наполнена данными](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/drop_t1_and_create_t1.PNG)

## Повторная попытка SELECT
1. Выдать права повторно
    ```sql
        GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
    ```

**Комментарии:** эта команда работает только для таблиц, существующих на момент выполнения команды. Она не применяется автоматически к новым таблицам, созданным позже.

2. Выйти в терминал
    ```sql
        \q
    ```

3. Подключиться под пользователем testread
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U testread -d testdb
    ```

4. Выполнить команду
    ```sql
        SELECT * FROM testnm.t1; 
    ```

![Таблица доступна пользователю testread](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/select_t1_schema_testnm.PNG)

**Результат:** успешно, возвращается строка со значением 1.
**Объяснение:** теперь таблица находится в правильной схеме, к которой у пользователя testread есть права.

## Настройка прав по умолчанию
1. Выйти в терминал
    ```sql
        \q
    ```

2. Подключиться под пользователем postgres
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U postgres -d testdb
    ```

3. Выполнить команду 
    ```sql
        ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
    ```


![Права выданы по умолчанию](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/default_grant.PNG)

**Комментарии:** это гарантирует, что все новые таблицы в схеме testnm автоматически получат права SELECT для роли readonly.

## Попытка создания таблицы под testread
1. Выйти в терминал
    ```sql
        \q
    ```

2. Подключиться под пользователем testread
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U testread -d testdb
    ```

3. Выполнить команды
    ```sql
        CREATE TABLE t2(c1 integer); 
        INSERT INTO t2 VALUES (2);
    ```

**Результат:** успешно, хотя роль readonly не должна иметь таких прав.
**Объяснение:** по умолчанию в PostgreSQL все пользователи имеют права CREATE и USAGE в схеме public. Так как мы не указали схему при создании таблицы, то она по умолчанию создалась в схеме public.

## Отзыв прав на public
1. Выйти в терминал
    ```sql
        \q
    ```

2. Подключиться под пользователем postgres
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U postgres -d testdb
    ```

3. Выполнить команды
    ```sql
        REVOKE ALL ON SCHEMA public FROM public;
        REVOKE ALL ON DATABASE testdb FROM public;
    ```

**Комментарии:** теперь пользователи по умолчанию не смогут создавать объекты в public.

## Повторная попытка создания таблицы
1. Выйти в терминал
    ```sql
        \q
    ```

2. Подключиться под пользователем testread
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U testread -d testdb
    ```

3. Выполнить команды
    ```sql
        CREATE TABLE t3(c1 integer); 
    ```

![Таблица не создалась](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_4/error_after_revoke.PNG)

**Результат:**оОшибка - нет прав на создание таблицы в схеме public.
**Объяснение:** после отзыва прав из public пользователь testread больше не может создавать таблицы.