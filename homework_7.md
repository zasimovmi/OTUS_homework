# Выпонение домашнего задания №7

## Установка VirtualBox на Windows
1. Скачать "Windows hosts" (ссыслка: [VirtualBox](https://www.virtualbox.org/wiki/Downloads/))
2. Устновить VirtualBox

## Подготовка ISO-образа Linux
1. Скачать ISO-образ Debian (ссылка: [Debian](https://www.debian.org/CD/netinst/))
***Комментарии:*** *использовал Netinst-образ, так как "весит" меньше и при установке всё "докачается"*

## Создание виртуальной машины в VirtualBox
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

## Создать правило для ssh подключения
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

## Проверить, установлен и запущен ли SSH-сервер
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

## Установка PosrgreSQl и проверка работы кластера
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

![Проверка пройдена успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/status_postgres.PNG)

## Настройка PostgreSQL
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

## Настройка контрольной точки раз в 30 секунд
1. Сделать бэкап исходного конфига
    ```bash
        sudo cp /etc/postgresql/17/main/postgresql.conf postgresql.conf.backup
    ```

2. Убедиться, что бэкап прошёл успешно
    ```bash
        ls -la postgresql.conf.backup
    ```

![Бэкап конфига успешно сделан](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/backup_config.PNG)

3. Открыть postgresql.conf
    ```bash
        sudo nano /etc/postgresql/17/main/postgresql.conf
    ```

4. Внести изменения (ниже приведены параметры, которые были изменены)
    ```text
	# WRITE-AHEAD LOG
    checkpoint_timeout = 30s          # Максимальный интервал между контрольными точками (ЧКП)
	max_wal_size = 2GB                # Задаем большое значение, чтобы ЧКП срабатывали только по таймауту, а не по размеру WAL
	min_wal_size = 1GB                

	# REPORTING AND LOGGING
	log_checkpoints = on              # Включим логирование информации о контрольных точках для анализа (у меня был включен)
    ```

![Меняем указанные параметры](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/postgresql_conf.PNG)

5. Сохранить файл и выйти в терминал

6. Перезагрузить сервер, чтобы изменения вступили в силу
    ```bash
        sudo systemctl restart postgresql
    ```

**Объяснение:** параметр checkpoint_timeout определяет максимальный интервал между контрольными точками (ЧКП), я установил его в 30 секунд.
max_wal_size намеренно установлен большим, чтобы ЧКП срабатывало именно по таймауту, а не из-за заполнения места под WAL-файлы.

## Нагрузочное тестирование и замер объема WAL
1. Инициализация базы данных для pgbench
    ```bash
        sudo -u postgres pgbench -i -s 50 test_db
        # -s 50 создаст масштаб БД в 50 раз больше стандартного (50 * 100000 = 5M строк)
    ```

![Инициализация прошла успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/pgbench_init.PNG)

2. Определим текущий LSN
***Это указатель на позицию в WAL. Я замерю разницу между начальным и конечным LSN.***
* Зайти в psql
    ```bash
        sudo -u postgres psql
    ```

* Выполнить запрос для определения текущей позиции
    ```sql
        SELECT pg_current_wal_lsn();
    ```

![Получаем первый LSN](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/first_LSN.PNG)

**Результат:** 0/28B9D7E8

* Выходим в терминал
    ```sql
        \q
    ```

3. Запускаем нагрузку на 10 минут (600 секунд)
    ```bash
        sudo -u postgres pgbench -T 600 -c 10 -j 2 test_db
        # -c 10 подключений, -j 2 потока, тест длится 600 сек
    ```

![Успешно отработано](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/pgbench_600_s.PNG)

4. По окончании теста снова определяем LSN
* Зайти в psql
    ```bash
        sudo -u postgres psql
    ```

* Выполнить запрос для определения текущей позиции
    ```sql
        SELECT pg_current_wal_lsn();
    ```

![Получаем второй LSN](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/second_LSN.PNG)

**Результат:** 0/A142FD08

5. Рассчитаем объем сгенерированных WAL-данных (в байтах)
    ```sql
        --Эта функция вычтет первый LSN из второго и вернет разницу в байтах
        SELECT pg_wal_lsn_diff('0/A142FD08', '0/28B9D7E8') AS wal_generated_bytes;
    ```

![Получаем результат в байтах](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/result_bytes.PNG)

**Результат:** 2022253856

**Расчёты:** за 600 секунд при интервале в 30 секунд должно было произойти 20 (600 / 30)  контрольных точек.
Разделим общий объем wal_generated_bytes (2022253856) на 20, чтобы получить средний объем на одну ЧКП - это, примерно, 101112693 (~96.5 Мегабайт).

**Вывод:** в результате измерений было установлено, что за 10 минут нагрузки с помощью pgbench было сгенерировано примерно 2 022 253 856 байт (~ 1.93 ГБ) WAL-данных.

При настройке checkpoint_timeout = 30s за 600 секунд теоретически должно было произойти 20 контрольных точек. Таким образом, в среднем на одну контрольную точку приходится ~96.5 МБ журнальных записей.

~96.5 МБ - это объем изменений ("грязных" данных), которые система накапливает в оперативной памяти за 30 секунд и затем обязана записать на диск во время следующей контрольной точки. Моя нагрузка генерировала в среднем ~3.2 МБ/с изменяемых данных.

## Проверка статистики контрольных точек
Проверяем логи сервера.
Поскольку мы включили log_checkpoints = on, вся информация есть в лог-файле PostgreSQL.

![Логи](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/logs.PNG)

**Вывод:** контрольные точки выполнялись не точно каждые 30 секунд, а с интервалом 32 секунды.

**Почему так произошло:** несмотря на то, что фактическое время записи контрольной точки составляло всего ~1.8 секунд, система PostgreSQL с параметром checkpoint_completion_target = 0.9 пытается распределить нагрузку записи на 27 секунд (90% от интервала). Вероятно, окончательная синхронизация и завершение контрольной точки занимали дополнительное время, что приводило к задержке начала следующей контрольной точки на 2 секунды.

Это нормальное поведение, которое показывает, что система балансирует между соблюдением расписания и равномерной нагрузкой на диск.

## Сравнение tps в синхронном и асинхронном режиме
1. Протестируем синхронный режим (по умолчанию)
    ```bash
        sudo -u postgres pgbench -T 30 -c 10 -j 2 test_db
    ```

![Первый tps (синхронный)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/first_tps.PNG)

**Результат:** 565 tps

2. Протестируем асинхронный режим
* Открыть postgresql.conf
    ```bash
        sudo nano /etc/postgresql/17/main/postgresql.conf
    ```

* Внести изменения (ниже приведены параметры, которые были изменены)
    ```text
	synchronous_commit = off
    ```

* Сохранить файл и выйти в терминал

* Перезагрузить сервер, чтобы изменения вступили в силу
    ```bash
        sudo systemctl restart postgresql
    ```

* Запускаем тест с теми же параметрами
    ```bash
        sudo -u postgres pgbench -T 30 -c 10 -j 2 test_db
    ```

![Второй tps (асинхронный)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/second_tps.PNG)

**Результат:** 565 tps

**Вывод:** TPS в асинхронном режиме будет значительно выше (в данном случе, примерно, в 3 раза). В синхронном режиме (on) сервер ждет подтверждения записи WAL на диск, прежде чем сообщить клиенту об успехе фиксации транзакции. Это гарантирует сохранность данных, но является "узким" местом производительности. В асинхронном режиме (off) сервер фиксирует транзакцию в памяти и немедленно сообщает клиенту об успехе, а запись на диск происходит позже. Это быстрее, но несет риск потери данных при сбое.

## Повреждение данных и проверка контрольных сумм
1. Создаём новый кластер с включенными контрольными суммами
    ```bash
        # Останавливаем текущий сервер
        sudo systemctl stop postgresql

        # Создаём новый каталог для данных
        sudo mkdir -p /var/lib/postgresql/test_data
        sudo chown postgres:postgres /var/lib/postgresql/test_data

        # Инициализируем новый каталог БД с включенными контрольными суммами
        sudo -u postgres /usr/lib/postgresql/17/bin/initdb -D /var/lib/postgresql/test_data --data-checksums
        # Ключевой момент здесь -- --data-checksums
    ```

![checksums](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/checksums.PNG)

2. Запустить новый кластер на другом порту, чтобы не конфликтовать с основным
    ```bash
        sudo -u postgres /usr/lib/postgresql/17/bin/pg_ctl -D /var/lib/postgresql/test_data -o "-p 5433" start
    ```

![Успешно подключились по другому порту](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/5433.PNG)

3. Создать таблицу и вставить данные
* Подключаемся по новому порту
    ```bash
        sudo -u postgres psql -p 5433 postgres
    ```

* Создаём таблицу и вставляем данные
    ```sql
        CREATE TABLE test_table (id serial, data text);
        INSERT INTO test_table (data) VALUES ('Very important data'), ('Another important record');
        SELECT * FROM test_table;
    ```

![Успешно всё создалось и вставилось](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/sql.PNG)

4. Посмотреть путь к файлу
    ```sql
        SELECT pg_relation_filepath('test_table');
        \q
    ```

![Получаем путь к файлу](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/path.PNG)

**Результат:** base/5/16389

5. Останавливаем кластер
    ```bash
        sudo -u postgres /usr/lib/postgresql/17/bin/pg_ctl -D /var/lib/postgresql/test_data -o "-p 5433" stop
    ```

6. Находим файл из шага 4

![Находим через mc](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/change.PNG)

7. Делаем резервную копию данного файла
    ```bash
        sudo cp /var/lib/postgresql/test_data/base/5/16389 /var/lib/postgresql/test_data/base/5/16389.backup
    ```

8. Создаем hex-дамп
    ```bash
        sudo hexdump -C /var/lib/postgresql/test_data/base/5/16389 > /tmp/file.hex
    ```

9. Смотрим содержимое
    ```bash
        sudo head -n 10 /tmp/file.hex
    ```

![Смотрим содержимое hex-дампа](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/hexdump.PNG)

10. Создаем измененную версию
* Выполнить команды
    ```bash
        sudo cp /tmp/file.hex /tmp/file_modified.hex
        sudo nano /tmp/file_modified.hex
    ```

* Изменим первую строку (00 на FF)
    ```text
        БЫЛО
        00000000  00 00 00 00 a8 f5 79 01

        СТАЛО
        00000000  FF 00 00 00 a8 f5 79 01
    ```

![Вносим изменения](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/hexdump_change.PNG)

11. Преобразовываем обратно в бинарный формат
* Устанавливаем xxd
    ```bash
        sudo apt update && sudo apt install xxd
    ```


* Делаем преобразование
    ```bash
        sudo xxd -r /tmp/file_modified.hex | sudo tee /var/lib/postgresql/test_data/base/5/16389 > /dev/null
    ```

12. Запускаем кластер
    ```bash
        sudo -u postgres /usr/lib/postgresql/17/bin/pg_ctl -D /var/lib/postgresql/test_data -o "-p 5433" start
    ```

![Сервер успешно запущен](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/start_server.PNG)

13. Проверим данные
    ```bash
        sudo -u postgres psql -p 5433 postgres -c "SELECT * FROM test_table;"
    ```

![Получаем ошибку](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/error_checksums.PNG)

14. Игнорируем ошибку
* Подключаемся к БД
    ```bash
        sudo -u postgres psql -p 5433 postgres
    ```

* Включаем игнорирование ошибок контрольных сумм
    ```sql
        SET ignore_checksum_failure = on;
    ```

* Делаем выборку
    ```sql
        SELECT * FROM test_table;
    ```

![Смотрим результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_7/incorrect.PNG)

**Вывод:** при включенных контрольных суммах страниц PostgreSQL обнаружил повреждение данных и вернул ошибку при попытке чтения таблицы. Это предотвратило возврат клиенту некорректных данных. С помощью параметра ignore_checksum_failure = on удалось проигнорировать ошибку и прочитать таблицу, но данные были частично повреждены. Это демонстрирует важность контрольных сумм для обеспечения целостности данных.