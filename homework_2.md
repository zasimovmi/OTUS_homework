# Выполнение домашнего задания №2

# Установка и настройка PostgreSQL в контейнере Docker

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

![Обновление пакетов](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/apt_update.PNG)

2. Установка Docker и включение его в автоматический запуск сервиса при старте системы
    ```bash
        sudo apt install docker.io
        sudo systemctl enable --now docker
    ```

3. Проверить версию Docker
    ```bash
        docker --version
    ```

![Проверка, что Docker установлен](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/docker_version.PNG)

## Подготовить каталог для PostgreSQL
1. Создать каталог для данных БД:
    ```bash
        sudo mkdir -p /var/lib/postgresql/data
        sudo chmod 777 /var/lib/postgresql/data  # Упроcтил права для теста
    ```

![Создание каталога для данных БД](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/mkdir_postgresql_chmod_777.PNG)

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

![Вывод списка зеркал](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/mirrors.PNG)

## Запустить контейнер PostgreSQL
1. Скачать образ postgresql:17
    ```bash
        sudo docker pull postgres:17
    ```

![Скачивание успешно завершено](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/pull_postgresql_17.PNG)

2. Создать именованный volume Docker (так не будет конфликтов с системной postgres-папкой)
    ```bash
        sudo docker volume create pgdata
    ```

![Создан именованный volume Docker](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/volume_docker.PNG)

3. Запустить контейнер с монтированием каталога
    ```bash
        sudo docker run --name postgres-server -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=testdb -v pgdata:/var/lib/postgresql/data -p 5432:5432 -d postgres:17
    ```

![Запуск произошёл успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/conteiner_docker.PNG)

## Развернуть контейнер с клиентом PostgreSQL и подключиться к серверу
1. Запуск клиента с подключением к серверу
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U postgres -d testdb
    ```

![Запуск произведён успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/start_client.PNG)

2. После подключения выполнить в psql
    ```sql
        -- Создание таблицы
        CREATE TABLE test_table (
        id SERIAL PRIMARY KEY,
        message TEXT
        );

        -- Добавление данных
        INSERT INTO test_table (message) VALUES 
        ('hello'),
        ('world');

        -- Проверка
        SELECT * FROM test_table;
    ```

![Таблица создана и наполнена данными](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/select_test_table_after_insert.PNG)

3. Выйти в терминал
    ```sql
        \q
    ```

## Подключиться с хоста (через dbeaver)
1. Узнаnm IP ВМ
    ```bash
        ip a | grep inet
    ```

![Видим список IP](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/ip.PNG)

2. Сделать проброс порта в VirtualBox
    1. Выбрать ВМ
    2. Нажать клавишу `Настроить`
    3. Зайти в раздел "Сеть"
    4. Нажать клавишу `Проброс портов`
    5. Установить новое правило:
    - **Name:** PostgreSQL
    - **Protocol:** TCP
    - **Host IP:** 127.0.0.1
    - **Host Port:** 5432
    - **Guest IP:** 10.0.2.15
    - **Guest Port:** 5432
    6. Сохранть ОК → ОК

3. Подключиться через DBeaver

![Подключение через DBeaver произведено успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/dbeaver.PNG)

## Удалить и восстановить контейнер с сохранением данных
1. Сделать остановку контейнера
    ```bash
        sudo docker stop postgres-server
    ```

2. Удалить контейнер
    ```bash
        sudo docker rm postgres-server
    ```

3. Проверить, что данные остались в volume
    ```bash
        sudo docker volume inspect pgdata
    ```

![Проверка произошла успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/check_pgdata.PNG)

4. Запустить новый контейнер с теми же данными
    ```bash
        sudo docker run --name postgres-server -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=testdb -v pgdata:/var/lib/postgresql/data -p 5432:5432 -d postgres:17
    ```

## Проверка сохранности данных
1. Запуск клиента с подключением к серверу
    ```bash
        sudo docker run -it --rm --network host postgres:17 psql -h 127.0.0.1 -U postgres -d testdb
    ```

2. Выполнить
    ```sql
        SELECT * FROM test_table;
    ```


![Отображаются ранее созданные записи](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_2/check_final.PNG)
