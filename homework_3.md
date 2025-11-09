# Выпонение домашнего задания №3

# Установка и настройка PostgreSQL (+ работа с виртуальными дисками)

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

### Установка PosrgreSQl и проверка работы кластера
1. Установить PostgreSQL
    1. Выполнить команду в терминале
    ```bash
        sudo apt install -y postgresql-common
    ```  

    2. Выполнить команды в терминале
    ```bash
        apt install postgresql-17
        sudo systemctl status postgresql
        # Статус: active
    ```

2. Проверить работу кластера
    ```bash
        sudo -u postgres pg_lsclusters
    ```

![Проверка пройдена успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/pg_lsclusters.PNG)

### Создать тестовую таблицу
1. Зайти в psql
    ```bash
        sudo -u postgres psql
    ```

2. Выполнить команды
    ```sql
        CREATE TABLE test(c1 text);
        INSERT INTO test VALUES('1');
    ```

3. Проверить, что даные появились в таблице
    ```sql
        SELECT * FROM test;
    ```

![Данные отобразились](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/check_test.PNG)

4.  Выходим в терминал
    ```sql
        \q
    ```

### Остановить PostgreSQL
1. Выполнить команду
    ```bash
        sudo -u postgres pg_ctlcluster 17 main stop
    ```

**Комментарии:** Система сообщила, что рекомендует выполнить остановку PostgreSQL дургой командой

![Предупреждение](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/warning_postgresql.PNG)

2. Выполнить команду, которую предлодила система
    ```bash
        sudo systemctl stop postgresql@17-main
    ```

3. Проверить статус
    ```bash
        sudo systemctl status postgresql@17-main
    ```

![PostgreSQL остановлен](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/postgresql_stop.PNG)

### Добавить новый виртуальный диск
1. Выбрать нужну ВМ
2. Нажать клавишу `Настроить`
3. Перейти в раздел "Носители"
4. В списке контроллеров ("SATA Controller") нажмать `Добавить новый диск` (значок + с надписью "Добавить жёсткий диск")
5. Нажать клавишу `Создать`
6. Установить имя: new_debian-postgres.vdi
7. Установить размер жёсткого диска - 10GB
8. Нажать клавишу `Готово`
9. Нажать клавишу `Выбрать`
10. Нажать клавишу `OK`

### Инициализация диска
1. Проверить доступные диски
    ```bash
        lsblk
    ```

2. Найти новый диск (обычно /dev/sdb)
    ```bash
        sudo fdisk /dev/sdb
    ```

![Команда успешно отработала](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/fdisk.PNG)

3. Ввести команды
    ```text
        n (новая партиция)
        p (primary)
        1 (номер партиции)
        Enter (принимаем значения по умолчанию для первого и последнего сектора)
        w (записываем изменения)
    ```

![Настройки применены](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/initialization.PNG)

### Создать файловую систему и монтирование
1. Создать файловую систему ext4
    ```bash
        sudo mkfs.ext4 /dev/sdb1
    ```

2. Создать точку монтирования
    ```bash
        sudo mkdir /mnt/data
    ```

3. Монтируем диск
    ```bash
        sudo mount /dev/sdb1 /mnt/data
    ```

4. Проверяем
    ```bash
        df -h
    ```

### Настройка автоматического монтирования
1. Получаем UUID диска
    ```bash
        sudo blkid /dev/sdb1
    ```

**Комментарии:** копируем UUID

2. Редактируем /etc/fstab
    ```bash
        sudo nano /etc/fstab
    ```

3. Добавляем строку (заменив UUID на фактический):
    ```text
        UUID=скопированный-uuid /mnt/data ext4 defaults 0 2
    ```

### Перезагрузка и проверка
1. Выполнить
    ```bash
        sudo reboot
        # После перезагрузки проверяем
        df -h
    ```

![Диск остался примонтированным](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/reboot_check.PNG)

### Настройка прав
1. Выполнить
    ```bash
        sudo chown -R postgres:postgres /mnt/data/
    ```

### Перенос данных
1. Выполнить
    ```bash
        sudo systemctl stop postgresql
        sudo mv /var/lib/postgresql/17 /mnt/data/
        sudo ln -s /mnt/data/17 /var/lib/postgresql/17
    ```

### Попытка запуска
1. Выполнить
    ```bash
        sudo systemctl start postgresql@17-main
    ```

**Комментарии:** ошибки нет, потому что была прописана следующая команда:
    ```bash
        sudo ln -s /mnt/data/17 /var/lib/postgresql/17
    ```

- Эта команда создала символическую ссылку в старом расположении (/var/lib/postgresql/17), которая указывает на новое (/mnt/data/17)
- PostgreSQL, пытаясь прочитать данные из /var/lib/postgresql/17, автоматически перенаправляется в /mnt/data/17 благодаря ссылке

**Без ссылки кластер не смог бы стартовать, пока не измениться data_directory в /etc/postgresql/17/main/postgresql.conf на /mnt/data/17/main**

![Ошибки не возникло](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/no_error.PNG)

2. Проверка
    ```bash
        sudo -u postgres psql -c "SELECT * FROM test;"
    ```

![Вернулась созданная ранее запись](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/check_after_s.PNG)

## Домашнее задание со звёздочкой (*)

### Создать новую ВМ
1. Проделать шаги, описанные выше:
    - Создание виртуальной машины в VirtualBox
    **Комментарии:** имя указать - debian-postgres-1
    - Первоначальная настройка ВМ

### Установить PostgreSQL
1. Устанавить ключ репозитория
    ```bash
        wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /usr/share/keyrings/postgresql.gpg
        # Нажать 'y' и Enter в ответ на запрос о перезаписи
    ```

2. Проверить результат
    ```bash
        ls -l /usr/share/keyrings/postgresql.gpg
    ```

**Комментарии:** должен показать файл размером ~3-4KB

3. Добавить репозиторий
    ```bash
        echo "deb [arch=amd64 signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list
    ```

4. Обновляем пакеты
    ```bash
        sudo apt update
    ```

5. Устанавливаем PostgreSQL 17
    ```bash
        sudo apt install -y postgresql-17
    ```

6. Выполнить проверку
```bash
sudo -u postgres pg_lsclusters
```

![Видим работающий кластер PostgreSQL 17](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/check_install_postgresql.PNG)

### Подготовить вторую ВМ
1. Останавить PostgreSQL
    ```bash
        sudo systemctl stop postgresql
    ```

2. Удалить данные
    ```bash
        sudo rm -rf /var/lib/postgresql/17
    ```

### Подключить диск от первой ВМ
1. Выбрать нужну ВМ
2. Нажать клавишу `Настроить`
3. Перейти в раздел "Носители"
4. В списке контроллеров ("SATA Controller") нажмать `Добавить новый диск` (значок + с надписью "Добавить жёсткий диск")
5. Выбрать ранее созданный жёсткий диск от первой ВМ (new_debian-postgres.vdi)
6. Нажать клавишу `Выбрать`
7. Нажать клавишу `OK`

### Монтирование и настройка
1. Создать точку монтирования
    ```bash
        sudo mkdir /mnt/data
    ```

2. Монтируем диск
    ```bash
        sudo mount /dev/sdb1 /mnt/data
    ```

3. Проверить содержимое
    ```bash
        sudo ls /mnt/data/17/main
    ```

4. Отредактировать конфигурацию
    ```bash
        sudo nano /etc/postgresql/17/main/postgresql.conf
    ```

5. Установить занение
    ```text
        data_directory = '/mnt/data/17/main'
    ```

### Настройка прав
1. Выполнить
    ```bash
        sudo chown -R postgres:postgres /mnt/data/
    ```

### Запуск PostgreSQL
1. Выполнить
    ```bash
        sudo systemctl start postgresql@17-main
    ```

### Проверить доступность данных
```bash
sudo -u postgres psql -c "SELECT * FROM test;"
```

**Комментарии:** видим ту же таблицу с данными, что создавали на первой ВМ


![Данные видны](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/homework_3/final_check.PNG)
