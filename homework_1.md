# Подготовка к выполнению домашнего задания

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
    - **Имя:** debian-postgres
    - **Образ ISO:** указал путь в скаченному iso-образу Debian
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
## Настройка SSH
1. Выполнить следующие команды:
    ```bash
        sudo apt update
        sudo apt install -y openssh-server
        sudo systemctl enable --now ssh
        sudo systemctl status ssh
        # При выполнении последней команды получил статус: active (running)
    ```
## Сделать проброс порта в VirtualBox
1. Выбрать ВМ
2. Нажать клавишу `Настроить`
3. Зайти в раздел "Сеть"
4. Нажать клавишу `Проброс портов`
5. Установить новое правило:
    - **Name:** ssh
    - **Protocol:** TCP
    - **Host IP:** оставь пустым
    - **Host Port:** 2222
    - **Guest IP:** оставь пустым
    - **Guest Port:** 22
6. Сохранть ОК → ОК

## Запустить ssh-agent на Windows
1. Открыть PowerShell от **Администратора**
2. Выполнить следующие команды:
    ```powershell
        Get-Service ssh-agent | Set-Service -StartupType Automatic
        Start-Service ssh-agent
        ssh-add -l  
        # Результат: The agent has no identities
    ```

## Пробросить публичный ключ на Debian-ВМ (аналог "metadata")
1. Сделал одной командой из Powershell
    ```powershell
        type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 user1@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
    ```
2. Делаю проверку и ввожу команду
    ```powershell
        ssh -p 2222 user1@127.0.0.1
        # Результат: вход выполнен
    ```

## Установка PosrgreSQl
1. Открыть PowerShell от **Администратора**
2. Осуществляю вход
    ```powershell
        ssh -p 2222 user1@127.0.0.1
        # Результат: вход выполнен
    ``` 
3. Установить PostgreSQL
    1. Выполнить команду в терминале
    ```bash
        sudo apt install -y postgresql-common
    ```  
    2. Выполнить скрипт в PowerShell 
    ```powershell
        sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
    ```
    3. Выполнить команды в терминале
    ```bash
        apt install postgresql-17
        sudo systemctl status postgresql
        # Статус: active
    ```
## Отключение autocommit в двух сессиях
1. Зайти в psql от пользователя postgres (в обеих сессиях)
     ```bash
        sudo -iu postgres
        psql
    ``` 

![Вход в psql (две сессии)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/psql.PNG)

2. Выключить autocommit (в обеих сессиях)
    ```bash
        \set AUTOCOMMIT off
        \echo :AUTOCOMMIT
        # Результат: off
        ```

![Отключение autocommit (две сессии)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/autocommit_off.PNG)

# Выполнение домашнего задания

## Часть 1: Работа с уровнем изоляции по умолчанию (READ COMMITTED)

### Шаг 1. Создать таблицу и наполнить её данными (Сессия 1)
```sql
    CREATE TABLE persons(
        id SERIAL, 
        first_name TEXT, 
        second_name TEXT
    );
    INSERT INTO persons(first_name, second_name) VALUES('ivan', 'ivanov');
    INSERT INTO persons(first_name, second_name) VALUES('petr', 'petrov');
    COMMIT;
```

### Шаг 2. Проверить уровень изоляции в обмеих сессиях
```sql
    SHOW TRANSACTION ISOLATION LEVEL;
    -- Ожидаемый результат: read committed
```

![Создание таблицы в сессии 1 и проверка уровня изоляции (две сессии)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/create_s1_show_isolation_level_s1_s2.PNG)

### Шаг 3. Начать транзакции в обеих сессиях
```sql
    BEGIN;
```

![Начали транзакции (две сессии)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/begin_p1_s1_s2.PNG)

### Шаг 4. Добавить запись в первой сессии
```sql
    -- Сессия 1
    INSERT INTO persons(first_name, second_name) VALUES('sergey', 'sergeev');
```

### Шаг 5. Проверить данные во второй сессии
```sql
    -- Сессия 2
    SELECT * FROM persons;
```

![Доавление новой строки в сессии 1 и проверка в этот момент таблицы в сессии 2](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/insert_s1_select_s2.PNG)

**Результат:** видим только 2 записи (ivan и petr), новую запись (sergey) не видим.
Уровень изоляции READ COMMITTED не позволяет видеть незакоммиченные изменения других транзакций.

### Шаг 6. Сделать коммит в первой сессии
```sql
    -- Сессия 1
    COMMIT;
```

### Шаг 7. Проверить данные во второй сессии
```sql
    -- Сессия 2
    SELECT * FROM persons;
```

![Коммит в сессии 1 и проверка в этот момент таблицы в сессии 2](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/commit_s1_select_s2.PNG)

**Результат:** теперь видим 3 записи, включая "sergey".
После коммита изменения становятся видимыми для других транзакций на уровне READ COMMITTED.

### Шаг 8. Завершить транзакцию во второй сессии
```sql
    -- Сессия 2
    COMMIT;
```

## Часть 2: Работа с уровнем изоляции REPEATABLE READ

### Шаг 1. Начинать новые транзакции с уровнем REPEATABLE READ
```sql
    -- В обеих сессиях
    BEGIN;
    SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

![Начали новые транзакции (обе сессии) + проверка уровня изоляции (обе сессии)](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/show_isolation_repeatable_read_s1_s2.PNG)

### Шаг 2. Добавить запись в первой сессии
```sql
    -- Сессия 1
    INSERT INTO persons(first_name, second_name) VALUES('sveta', 'svetova');
```

### Шаг 3. Проверить данные во второй сессии
```sql
    -- Сессия 2
    SELECT * FROM persons;
```

![Вставили новую строчку в таблицу (сессия 1) и проверка в этот момент таблицы в сессии 2](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/insert_s1_select_s2_read.PNG)

**Результат:** видим только 3 записи (без sveta).
На уровне REPEATABLE READ транзакция работает с "снимком" данных на момент своего начала.

### Шаг 4. Сделать коммит в первой сессии
```sql
    -- Сессия 1
    COMMIT;
```

### Шаг 5. Проверить данные во второй сессии
```sql
    -- Сессия 2
    SELECT * FROM persons;
```

![Коммит в сессии 1 и проверка в этот момент таблицы в сессии 2](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/commit_s1_select_s2_readPNG.PNG)

**Результат:** так же видим только 3 записи (без sveta).
REPEATABLE READ гарантирует, что в течение транзакции будут видны только те данные, которые были закоммичены до её начала.

### Шаг 6. Завершить транзакцию во второй сессии
```sql
    -- Сессия 2
    COMMIT;
```

### Шаг 7. Проверить данные после завершения всех транзакций
```sql
    -- Сессия 2
    SELECT * FROM persons;
```

![Коммит в сессии 2 и проверка в этой же сессии таблицы](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_1/commit_s2_select_s2.PNG)

**Результат:** теперь видим все 4 записи.
После завершения всех транзакций мы видим актуальное состояние базы данных.

## Оформление в виде таблицы
| Действие          | Уровень изоляции | Сессия 1        | Сессия 2          | Видимость изменений | Объяснение                                                  |
|-------------------|------------------|-----------------|-------------------|---------------------|-------------------------------------------------------------|
| INSERT в сессии 1 | READ COMMITTED   | Добавлен sergey | SELECT → 2 записи | Не видны            | Незакоммиченные изменения не видны                          |
| COMMIT в сессии 1 | READ COMMITTED   | -               | SELECT → 3 записи | Видны               | После коммита изменения видны                               |
| INSERT в сессии 1 | REPEATABLE READ  | Добавлен sveta  | SELECT → 3 записи | Не видны            | Транзакция видит снимок на момент своего начала             |
| COMMIT в сессии 1 | REPEATABLE READ  | -               | SELECT → 3 записи | Не видны            | REPEATABLE READ сохраняет снимок данных                     |
| COMMIT в сессии 2 | -                | -               | SELECT → 4 записи | Видны               | После завершения всех транзакций видно актуальное состояние |
