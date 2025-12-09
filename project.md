# Выпонение проектной работы

# Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni

## Установка VirtualBox на Windows
1. Скачать "Windows hosts" (ссыслка: [VirtualBox](https://www.virtualbox.org/wiki/Downloads/))
2. Устновить VirtualBox

## Подготовка ISO-образа Linux
1. Скачать ISO-образ Ubuntu (ссылка: [Ubuntu](https://ubuntu.com/download/server#manual-install))

## Создание виртуальной машины №1 в VirtualBox
1. Запустить VirtualBox
2. Нажать клавишу `Создать`
3. Заполнить поля:
    - **Имя:** pg-node-1
    - **Образ ISO:** указал путь в скаченному iso-образу Ubuntu
4. Нажать клавишу `Далее`
5. Задать "Имя пользователя"
6. Задать и подтвердить пароль
7. Нажать клавишу `Далее`
8. В разделе "Оборудование" установить
    - **RAM** - 2GB
    - **Процессоры** - 2
9. Нажать клавишу `Далее`
10. Размер диска - 7GB
11. Нажать клавишу `Далее`
12. Нажать клавишу `Готово`
13. Дождаться установки
14. Выбрать созданную виртуальную машину (далее - ВМ)
15. Нажать клавишу `Настроить`
16. Перейти в раздел "Носители"
17. Найти "Контроллер: IDE"
18. Проверить, что диск пустой
19. Зайти в раздел "Сеть"
20. Убедиться, что тип подключения установлен как **Сетевой мост**
21. Нажать клавишу `ОК`

## Создание виртуальной машины №2 в VirtualBox
- Повторить все действия из шага "Создание виртуальной машины №1 в VirtualBox"
- Изменить наименование на pg-node-2

## Создание виртуальной машины №3 в VirtualBox
- Повторить все действия из шага "Создание виртуальной машины №1 в VirtualBox"
- Изменить наименование на pg-node-3

## IP адреса каждой ВМ
**Комментарий:** проброс портов ssh не нужно делать, так как я использую сетевой мост (ВМ получает реальный IP в моей сети).
Я смогу подключаться напрямую по этому IP.

1. На каждой ВМ проделываю команду:
```bash
        ip addr show
```
2. Получаю список IP адресов
```text
        192.168.0.30 # pg-node-1
        192.168.0.31 # pg-node-2
        192.168.0.32 # pg-node-3
```

## Проверить связь с хостовой машиной
**Комментарий:** на реальном компьютере (не в ВМ) открыть PowerShell и ввести поочерёдно команды:
    ```powershell
        ping 192.168.0.30
        ping 192.168.0.31
        ping 192.168.0.32
    ```

![Проверка пройдена успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/ping_shell.PNG)

## Первоначальная настройка ВМ
**Комментарий:** действия проделываются на каждой ВМ

1. Выбрать ВМ
2. Нажать клавишу `Запустить`
3. Следовать мастеру устновки
4. Установка произошла успешно
5. Войти под созданным пользователем
6. Зайти в терминал
6. Создаем пользователя patroni и добавляем в группу sudo:
    1. Ввести комманду
```bash
        sudo useradd -m -s /bin/bash patroni
```
    2. Ввести пароль
    3. Выполнить команду
```bash
        echo "patroni:patroni" | sudo chpasswd  # пароль тоже patroni
```
    4. Выполнить команду
```bash
        sudo usermod -aG sudo patroni
```
    5. Проверить, что пользователь patroni добавлен в группу
```bash
        getent group sudo
        # Получил подтверждение: sudo:x:27:user1, patroni
```
    6. Сменить пользователя
```bash
        su - patroni
```

![Проверка пройдена успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/user_patroni.PNG)

## Проверить, установлен и запущен ли SSH-сервер
**Комментарий:** действия проделываются на каждой ВМ

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

## Настройка VS Code
1. Отркыть PowerShell
2. Выполнить команды
```powershell
        cd C:\Users\<Пользователь>
        mkdir .ssh # создать папку .ssh, если её не было. Иначе пропустить шаг.
        cd .ssh
        notepad config
```

3. Вставить содержимое
```ini
        Host pg-node-1
            HostName 192.168.0.30
            User patroni
            Port 22

        Host pg-node-2
            HostName 192.168.0.31
            User patroni
            Port 22

        Host pg-node-3
            HostName 192.168.0.32
            User patroni
            Port 22
```

4. Сохранить изменения
5. Открыть VS Code
6. Открыть раздел `Extensions` (Ctrl+Shift+X)
7. Установить расширение `Remote SSH`
8. После устновки нажать клавишу `F1`
9. Ввести в верхнее поле `Remote-SSH`
10. Найти `Remote-SSH: Connect to Host...`
11. Должно отобразиться три ноды

![Список из трёх нод](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/nodes_list.PNG)

12. Выберать pg-node-1 из списка
13. Ввести пароль: patroni
14. Повторить в новом окне VS Code для остальных нод

**Комментарий:** так как у меня кириллица, то имя пользователя корректно не прописывалось (вместо Максим подставлялось аксим).
Я не мог корректно создать файл `config` в папке `.ssh`, так как не мог указать корректный путь.

**Решение:** получилось обойти, использовав переменную `USERPROFILE`
Я сделал следующие действия:
- Создал файл `config.txt` в папке `.ssh`
- Заполнил данными
- Выполнил следующие команды:
```powershell
        # Перейти в папку .ssh
        cd $env:USERPROFILE\.ssh

        # Посмотреть, что есть
        dir

        # Вижу config.txt, поэтому переименовываю:
        Rename-Item config.txt config

        # Проверяю
        dir
        # Теперь просто config (без .txt)
```

**Результат:** теперь у меня следующая структура
Мой компьютер (Windows)
├── VirtualBox
│   ├── ВМ1: pg-node-1 (Ubuntu, IP: 192.168.0.30)
│   ├── ВМ2: pg-node-2 (Ubuntu, IP: 192.168.0.31)
│   └── ВМ3: pg-node-3 (Ubuntu, IP: 192.168.0.32)
│
└── VS Code (3 окна)
    ├── Окно 1: SSH к 192.168.0.30 (пользователь patroni)
    ├── Окно 2: SSH к 192.168.0.31 (пользователь patroni)
    └── Окно 3: SSH к 192.168.0.32 (пользователь patroni)

## Установка ПО на все узлы
На каждой ВМ (во всех трёх окнах VS Code) выполнить:
```bash
        # 1. Обновление
        sudo apt update && sudo apt upgrade -y

        # 2. Установка необходимых пакетов
        sudo apt install -y wget curl git python3-pip python3-psycopg2 \
        postgresql-client postgresql-common postgresql-16 postgresql-server-dev-16 \
        vim net-tools

        # 3. Остановить встроенный PostgreSQL (он нам мешает)
        # PostgreSQL надо останавливать, потому что Patroni будет управлять своим собственным экземпляром PostgreSQL
        sudo systemctl stop postgresql
        sudo systemctl disable postgresql

        # 4. Установить Patroni
        pip3 install --user patroni[etcd] psycopg2-binary
```

![Получаю ошибку](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/error_pip.PNG)

**Комментарий**: это новая "фича" Ubuntu 24.04 - они заблокировали глобальную установку pip пакетов для защиты системы.
В Ubuntu 24.04 разработчики решили защитить системные пакеты Python. Раньше была проблема:
- Устанавливаешь пакет через pip install
- Он перезаписывает системные библиотеки
- Ломаются системные программы, которые зависят от этих библиотек

Поэтому было решение - изоляция (через виртуальное окружение).

Примерная структура:
Система Ubuntu:
├── Системный Python (python3.12)
│   ├── Библиотеки для системы
│   └── Библиотеки для apt-пакетов
│
└── Виртуальное окружение patroni-venv
    ├── Свой Python (копия)
    ├── Patroni
    ├── psycopg2
    └── Другие зависимости


Необходимо проделать следующие шаги:
```bash
        # 5. Установить необходимые системные пакеты
        sudo apt install -y python3.12-venv python3-pip

        # 6. Создать виртуальное окружение
        python3 -m venv ~/patroni-venv

        # 7. Установливаем patroni
        ~/patroni-venv/bin/pip install patroni[etcd] psycopg2-binary

        # 8. Настраиваем PATH для пользователя patroni
        echo 'export PATH="$HOME/patroni-venv/bin:$PATH"' >> ~/.bashrc
        source ~/.bashrc

        # 9. Проверьте
        patroni --version
        patronictl version
```

![Установка patroni прошла успешно](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/patroni_version.PNG)

Далее устанавливаем etcd:
```bash
        # 10. Установить etcd
        ETCD_VER=v3.5.11
        wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
        tar xvf etcd-${ETCD_VER}-linux-amd64.tar.gz
        sudo mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
        rm -rf etcd-${ETCD_VER}-linux-amd64*
```

## Настройка etcd на каждой ноде
1. На **pg-node-1**:
```bash
        # 1. Создаем конфиг для etcd1
cat > ~/etcd1.conf <<- 'EOF'
name: etcd1
data-dir: /home/patroni/etcd-data1
listen-client-urls: http://0.0.0.0:2379
advertise-client-urls: http://192.168.0.30:2379
listen-peer-urls: http://0.0.0.0:2380
initial-advertise-peer-urls: http://192.168.0.30:2380
initial-cluster: etcd1=http://192.168.0.30:2380,etcd2=http://192.168.0.31:2380,etcd3=http://192.168.0.32:2380
initial-cluster-token: etcd-cluster-1
initial-cluster-state: new
EOF

        # 2. Создаем директорию для данных
mkdir -p ~/etcd-data1

        # 3. Запускаем etcd (в фоне)
nohup etcd --config-file ~/etcd1.conf > ~/etcd.log 2>&1 &
       
        # 4. Проверяем, что работает
ps aux | grep etcd
```

2. На **pg-node-2**:
```bash
        # 1. Создаем конфиг для etcd2
cat > ~/etcd2.conf <<- 'EOF'
name: etcd2
data-dir: /home/patroni/etcd-data2
listen-client-urls: http://0.0.0.0:2379
advertise-client-urls: http://192.168.0.31:2379
listen-peer-urls: http://0.0.0.0:2380
initial-advertise-peer-urls: http://192.168.0.31:2380
initial-cluster: etcd1=http://192.168.0.30:2380,etcd2=http://192.168.0.31:2380,etcd3=http://192.168.0.32:2380
initial-cluster-token: etcd-cluster-1
initial-cluster-state: new
EOF

        # 2. Создаем директорию для данных
mkdir -p ~/etcd-data2

        # 3. Запускаем etcd (в фоне)
nohup etcd --config-file ~/etcd2.conf > ~/etcd2.log 2>&1 &
       
        # 4. Проверяем, что работает
ps aux | grep etcd
```

3. На **pg-node-3**:
```bash
        # 1. Создаем конфиг для etcd3
cat > ~/etcd3.conf <<- 'EOF'
name: etcd3
data-dir: /home/patroni/etcd-data3
listen-client-urls: http://0.0.0.0:2379
advertise-client-urls: http://192.168.0.32:2379
listen-peer-urls: http://0.0.0.0:2380
initial-advertise-peer-urls: http://192.168.0.32:2380
initial-cluster: etcd1=http://192.168.0.30:2380,etcd2=http://192.168.0.31:2380,etcd3=http://192.168.0.32:2380
initial-cluster-token: etcd-cluster-1
initial-cluster-state: new
EOF

        # 2. Создаем директорию для данных
mkdir -p ~/etcd-data3

        # 3. Запускаем etcd (в фоне)
nohup etcd --config-file ~/etcd3.conf > ~/etcd3.log 2>&1 &
       
        # 4. Проверяем, что работает
ps aux | grep etcd
```

4. Проверка etcd кластера
На любой ноде (возьмём, например, pg-node-1) выполним:
```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=http://192.168.0.30:2379,http://192.168.0.31:2379,http://192.168.0.32:2379 \
  member list
```

![Получаем успешный результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/etcd_status.PNG)

**Комментарии:** команда из четвёртого пункта нужна для проверки состояния кластера etcd.
Эту команду можно запускать с любой ноды (даже не входящей в кластер etcd), если есть сетевой доступ к :2379.

Часть команды   | Описание                                                                                                                                                             |
----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
ETCDCTL_API=3   | Устанавливает версию API etcdctl в v3 (актуальную). Без этого по умолчанию может использоваться устаревший v2.                                                       |
----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
etcdctl         | CLI-утилита для управления и диагностики etcd                                                                                                                        |
----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
--endpoints=... | Список адресов клиентских портов etcd-нод (2379 - клиентский порт). Указываются все ноды кластера: так etcdctl сможет подключиться даже если одна из нод недоступна. |
----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
member list     | Подкоманда: запрашивает список участников (members) кластера etcd и их статус                                                                                        |

Полезные ссылки:
- [Мануал Ubuntu](https://manpages.ubuntu.com/manpages/noble/man1/etcdctl.1.html)
- [Статья про etcd](https://rudimartinsen.com/2020/12/30/backup-restore-etcd/)

## Создание конфигов Patroni
После того как etcd запущен, создаём конфиги для Patroni

1. На **pg-node-1**:
```bash
cat > ~/patroni1.yml <<- 'EOF'
scope: pg_cluster
namespace: /service/
name: node1

restapi:
    listen: 192.168.0.30:8008
    connect_address: 192.168.0.30:8008
    auth: 'api_user:api_password'

etcd3:
    - 192.168.0.30:2379
    - 192.168.0.31:2379
    - 192.168.0.32:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true
            parameters:
                max_connections: 100
                shared_buffers: 256MB
                wal_level: replica
                hot_standby: "on"
                wal_keep_size: 128MB
                max_wal_senders: 10
                max_replication_slots: 10
    initdb:
        - encoding: UTF8
        - data-checksums
    pg_hba:
        - host replication replicator 0.0.0.0/0 md5
        - host all all 0.0.0.0/0 md5

postgresql:
    listen: 192.168.0.30:5432
    connect_address: 192.168.0.30:5432
    data_dir: /home/patroni/pgdata1
    bin_dir: /usr/lib/postgresql/16/bin
    pgpass: /tmp/pgpass
    authentication:
        replication:
            username: replicator
            password: rep_password
        superuser:
            username: postgres
            password: admin123

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
EOF
```

**Комментарии:** разберём общую структуру для yaml-файла (конфига).
Для все остальных нод набор параметров будет индентичным.

**scope, namespace, name:**
```yaml
        scope: pg_cluster
        namespace: /service/
        name: node1
```

- `scope`: имя кластера PostgreSQL. Должно быть одинаковым на всех нодах.
- `namespace`: путь в etcd, где Patroni хранит состояние кластера (/service/pg_cluster/...)
- `name`: уникальное имя этой ноды. На других нодах будет node2, node3.

**restapi:**
```yaml
        restapi:
            listen: 192.168.0.30:8008
            connect_address: 192.168.0.30:8008
            auth: 'api_user:api_password'
```

- `listen`: интерфейс и порт для HTTP API Patroni (health-check, управление через curl, интеграция с HAProxy и т.д.)
- `connect_address`: адрес, по которому другие ноды могут достучаться до этого API
- `auth`: базовая аутентификация. Обязательна, если API доступен не только локально. (*для продакшена рекомендуется использовать TLS (certfile, keyfile)*)

**etcd3:**
```yaml
        etcd3:
            hosts:
                - 192.168.0.30:2379
                - 192.168.0.31:2379
                - 192.168.0.32:2379
```

- Список клиентских эндпоинтов etcd, где Patroni будет хранить и читать состояние кластера (leader, config, failover и т.д.)

**bootstrap → dcs:**
```yaml
        bootstrap:
            dcs:
                ttl: 30
                loop_wait: 10
                retry_timeout: 10
                maximum_lag_on_failover: 1048576
                postgresql:
                    use_pg_rewind: true
                    parameters:
                        max_connections: 100
                        shared_buffers: 256MB
                        wal_level: replica
                        hot_standby: "on"
                        wal_keep_size: 128MB
                        max_wal_senders: 10
                        max_replication_slots: 10
```

Это глобальные параметры кластера, которые будут записаны в etcd при первом запуске (bootstrap). Применяются ко всем нодам.

**bootstrap → pg_hba:**
```yaml
            initdb:
                - encoding: UTF8
                - data-checksums
```

- Параметры для initdb при первом создании кластера
- `data-checksums` - включает контрольные суммы страниц, критически важно для pg_rewind

**bootstrap → initdb:**
```yaml
            pg_hba:
                - host replication replicator 0.0.0.0/0 md5
                - host all all 0.0.0.0/0 md5
```

- Репликация с любого IP (0.0.0.0/0) по паролю (md5)
- Разрешено любому подключаться к любой БД от любого пользователя - тоже по паролю

**postgresql:**
```yaml
        postgresql:
            listen: 192.168.0.30:5432
            connect_address: 192.168.0.30:5432
            data_dir: /home/patroni/pgdata1
            bin_dir: /usr/lib/postgresql/16/bin
            pgpass: /tmp/pgpass
            authentication:
                replication:
                    username: replicator
                    password: rep_password
                superuser:
                    username: postgres
                    password: admin123
```

- `listen` - интерфейс, на котором PostgreSQL принимает подключения
- `connect_address` - адрес, по которому другие ноды Patroni подключаются к этой PostgreSQL-ноде для репликации и health-check
- `bin_dir` - путь к postgres, pg_ctl, pg_rewind и т.д.
- `pgpass` - файл с паролями
- `authentication` - учётные данные:
    - `replication`: пользователь для потоковой репликации
    - `superuser`: суперпользователь для операций (создание ролей, pg_rewind, initdb и т.д.).

**tags:**
```yaml
        tags:
            nofailover: false
            noloadbalance: false
            clonefrom: false
            nosync: false
```

- `nofailover: false` - нода может стать лидером
- `noloadbalance: false` - нода может использоваться для чтения (в HAProxy - в бэкендах read-only)
- `clonefrom: false` - не приоритетный источник для pg_basebackup (можно true на лидере или standby с синхронной репликацией)
- `nosync: false` - может участвовать в синхронной репликации

**Проблема:** при вставке такой команды в терминал могут добавляться лишние символы или теряться пробелы
**Решение:** использовал команду `nano ~/patroni1.yml` (для каждой ноды номер своего yml-файла) и вставил содержимое

2. На **pg-node-2**:
```bash
cat > ~/patroni2.yml <<- 'EOF'
scope: pg_cluster
namespace: /service/
name: node2

restapi:
    listen: 192.168.0.31:8008
    connect_address: 192.168.0.31:8008
    auth: 'api_user:api_password'

etcd3:
    - 192.168.0.30:2379
    - 192.168.0.31:2379
    - 192.168.0.32:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true
            parameters:
                max_connections: 100
                shared_buffers: 256MB
                wal_level: replica
                hot_standby: "on"
                wal_keep_size: 128MB
                max_wal_senders: 10
                max_replication_slots: 10
    initdb:
        - encoding: UTF8
        - data-checksums
    pg_hba:
        - host replication replicator 0.0.0.0/0 md5
        - host all all 0.0.0.0/0 md5

postgresql:
    listen: 192.168.0.31:5432
    connect_address: 192.168.0.31:5432
    data_dir: /home/patroni/pgdata2
    bin_dir: /usr/lib/postgresql/16/bin
    pgpass: /tmp/pgpass
    authentication:
        replication:
            username: replicator
            password: rep_password
        superuser:
            username: postgres
            password: admin123

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
EOF
```

3. На **pg-node-3**:
```bash
cat > ~/patroni3.yml <<- 'EOF'
scope: pg_cluster
namespace: /service/
name: node3

restapi:
    listen: 192.168.0.32:8008
    connect_address: 192.168.0.32:8008
    auth: 'api_user:api_password'

etcd3:
    - 192.168.0.30:2379
    - 192.168.0.31:2379
    - 192.168.0.32:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true
            parameters:
                max_connections: 100
                shared_buffers: 256MB
                wal_level: replica
                hot_standby: "on"
                wal_keep_size: 128MB
                max_wal_senders: 10
                max_replication_slots: 10
    initdb:
        - encoding: UTF8
        - data-checksums
    pg_hba:
        - host replication replicator 0.0.0.0/0 md5
        - host all all 0.0.0.0/0 md5

postgresql:
    listen: 192.168.0.32:5432
    connect_address: 192.168.0.32:5432
    data_dir: /home/patroni/pgdata3
    bin_dir: /usr/lib/postgresql/16/bin
    pgpass: /tmp/pgpass
    authentication:
        replication:
            username: replicator
            password: rep_password
        superuser:
            username: postgres
            password: admin123

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
EOF
```

**Проблема:** не мог долго запустить кластер.
Была проблема в том, что я прописывал etcd в yaml-файле вот так:
```yaml
        etcd:
            hosts: 192.168.0.30:2379,192.168.0.31:2379,192.168.0.32:2379
```

`etcd: hosts: host1:port,host2:port` - устаревший формат.
Этот синтаксис работал в Patroni ≤ 1.6, но больше не поддерживается.

hosts как строка с запятыми игнорируется или парсится некорректно.
Patroni не находит корректный DCS → не может записать/прочитать состояние кластера → зависает, выдаёт ошибки

Успешный запуск кластера был с такими настройками etcd:
```yaml
        etcd3:
            hosts:
                - 192.168.0.30:2379
                - 192.168.0.31:2379
                - 192.168.0.32:2379
```

## Запуск Patroni
1. На **pg-node-1**:
```bash 
    patroni ~/patroni1.yml
```

**Результат:** ошибка
```text 
    FATAL: could not create lock file "/var/run/postgresql/.s.PGSQL.5432.lock": Permission denied
```

PostgreSQL при запуске хочет создать lock-файл (файл-блокировку) в /var/run/postgresql/, но не может, потому что принадлежит не тому пользователю (user1), и patroni (запущенный от пользователя patroni) не имеет прав на запись.

`.s.PGSQL.5432.lock` - это служебный lock-файл, создаваемый PostgreSQL при запуске unix-сокета.

Подробнее:
- PostgreSQL слушает подключения не только по TCP/IP (192.168.0.30:5432), но и через unix-сокет - специальный файл в файловой системе.
- По умолчанию сокет создаётся в /var/run/postgresql/, и его имя:
    - `/.s.PGSQL.5432` - сам сокет
    - `/.s.PGSQL.5432.lock` - lock-файл, нужен для:
        - предотвращения запуска двух экземпляров PostgreSQL на одном порту/сокете
        - корректного определения, занят ли сокет (чтобы другие процессы не перезаписали его)

Без этого файла PostgreSQL откажется стартовать - отсюда FATAL.

**Решение:** на каждой ноде выполнить команды:
```bash 
    sudo mkdir -p /var/run/postgresql
    sudo chown patroni:patroni /var/run/postgresql
    sudo chmod 755 /var/run/postgresql
```

Действие               |  Что дало                                                                        |
-----------------------|----------------------------------------------------------------------------------|
mkdir                  |  Создал каталог → теперь есть куда писать сокет и lock-файл                      |
-----------------------|----------------------------------------------------------------------------------|
chown patroni:patroni  |  Пользователь patroni получил право записи → может создать `.s.PGSQL.5432.lock`  |
-----------------------|----------------------------------------------------------------------------------|
chmod 755              |  Доступ: rwxr-xr-x - достаточно для запуска                                      |

Ещё раз выполняю команду из первого шага для того, чтобы запустить систему Patroni, которая автоматически инициализирует кластер PostgreSQL, регистрирует его в распределённом хранилище конфигураций (DCS) и запускает HTTP-API.

2. На **pg-node-2**:
```bash 
    patroni ~/patroni2.yml
```

3. На **pg-node-3**:
```bash 
    patroni ~/patroni3.yml
```

4. На **pg-node-1**:
```bash 
    # В другом терминале на pg-node-1
    patronictl -c ~/patroni1.yml list
```

![Получаем успешный результат](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/patroni_run.PNG)

**Результат:** кластер полностью работает

## Настройка HAProxy
Все действия будут выполняться на На **pg-node-1**

1. Установить HAProxy
```bash 
    sudo apt install -y haproxy
```

2. Проверить версию HAProxy
```bash 
    haproxy -v
```

![Получаем версию HAProxy](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/haproxy_version.PNG)

3. Создать backup оригинального конфига HAProxy
```bash 
    sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.backup
```

4. Создать новый конфиг
```bash
sudo tee /etc/haproxy/haproxy.cfg <<- 'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# Frontend для клиентов - ЗАПИСЬ (только мастер)
frontend pg_write
    bind *:5000
    mode tcp
    option httpchk OPTIONS /master
    default_backend pg_master

# Frontend для клиентов - ЧТЕНИЕ (реплики + мастер)
frontend pg_read
    bind *:5001
    mode tcp
    option httpchk OPTIONS /replica
    default_backend pg_replicas

# Backend для мастера (только запись)
backend pg_master
    mode tcp
    balance roundrobin
    option httpchk OPTIONS /master
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node-1 192.168.0.30:5432 maxconn 100 check port 8008
    server pg-node-2 192.168.0.31:5432 maxconn 100 check port 8008
    server pg-node-3 192.168.0.32:5432 maxconn 100 check port 8008

# Backend для реплик (только чтение)
backend pg_replicas
    mode tcp
    balance roundrobin
    option httpchk OPTIONS /replica
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node-1 192.168.0.30:5432 maxconn 100 check port 8008
    server pg-node-2 192.168.0.31:5432 maxconn 100 check port 8008
    server pg-node-3 192.168.0.32:5432 maxconn 100 check port 8008

# Статистика HAProxy (веб-интерфейс)
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 5s
    stats realm HAProxy\ Statistics
    stats auth admin:adminpassword
EOF
```

5. Проверить синтаксис конфига
```bash 
    sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

![Конфигурационный файл HAProxy валиден](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/haproxy_valid.PNG)


6. Выполнить команды
```bash 
    # Перезапустите службу
    sudo systemctl restart haproxy

    # Включите автозагрузку
    sudo systemctl enable haproxy

    # Проверьте статус
    sudo systemctl status haproxy
```

![Статус HAProxy](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/haproxy_active.PNG)

**Результат:** active

7. Проверить, что HAProxy слушает порты
```bash 
    sudo ss -tulpn | grep haproxy
```

![Проверка успешно пройдена](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/haproxy_tulpn.PNG)

**Результат:** HAProxy успешно запущен и слушает все нужные порты
- Порт 8404 → Мониторинг
- Порт 5000 → Запись (только мастер)
- Порт 5001 → Чтение (реплики)

8. Открыть в браузере: http://192.168.0.30:8404/stats
- Логин: admin
- Пароль: adminpassword

![Статистика успешно открылась](https://github.com/zasimovmi/OTUS_homework/blob/main/screenshot/project/haproxy_statistics.PNG)