# PostgreSQL Streaming Replication (потоковая репликация)

  Данное "руководство" будет использоваться для **PostgreSQL** версии **9.6**
  сервера под управлением **Ubuntu Bionic Beaver**

## Потоковая репликация (Master -> Replica)

    В нашем случае имеется два сервера:
    - master 192.168.1.1
    - replica 192.168.1.2
  
### Настраиваем Master

В файл */etc/postgresql/9.6/main/postgresql.conf* вносим следующие изменения

    # Указывает IP-адрес или имя компьютера, на котором сервер postgres принимает клиентские подключения по TCP/IP.
    # Значением может быть список адресов, разделённых запятыми, либо символ *, обозначающий все доступные интерфейсы.
    listen_addresses = '*'

    # Параметр wal_level определяет, как много информации записывается в WAL. Со 
    # значением minimal (по умолчанию) в журнал записывается только информация, 
    # необходимая для восстановления после сбоя или аварийного отключения. Вариант 
    # replica добавляет в него то, что необходимо для архивирования WAL, а также 
    # информацию, необходимую для выполнения запросов на резервном сервере в режиме 
    # «только чтение». Наконец, logical добавляет информацию, требуемую для поддержки 
    # логического декодирования.
    wal_level = hot_standby

    # Задаёт максимально допустимое число одновременных подключений резервных серверов
    # или клиентов потокового копирования (т. е. максимальное количество одновременно
    # работающих процессов передачи WAL). По умолчанию это значение равно нулю, то
    # есть репликация отключается. Передающие WAL процессы учитываются в общем числе
    # соединений, так что этот параметр не может превышать max_connections.
    max_wal_senders = 3

    # Задаёт минимальное число файлов прошлых сегментов журнала, которые будут сохраняться в каталоге pg_xlog,
    # чтобы ведомый сервер мог выбрать их при потоковой репликации. Обычно сегмент имеет размер 16 мегабайт.
    # Если ведомый сервер, подключённый к передающему, отстаёт больше чем на wal_keep_segments сегментов,
    # передающий удаляет сегменты WAL, всё ещё необходимые ведомому, и в этом случае соединение репликации прерывается.
    # В результате этого затем также будут прерваны зависимые соединения.
    wal_keep_segments = 128
    
    # Когда параметр archive_mode включён, полные сегменты WAL передаются в хранилище архива.
    archive_mode    = on
    archive_command = 'cp %p /var/lib/pg-archive/%f'

Создадим директорию для хранения WAL сигментов:
    
    mkdir /var/lib/pg-archive/
    chown postgres /var/lib/pg-archive/
    chmod 700 /var/lib/pg-archive/
    
Чтобы архив не занял всё место на диске, добавим в cron следующее:

    40 7 * * * /usr/bin/find /var/lib/pg-archive/ -type f -mtime +5 -exec rm {} \;

Чистим все сигменты старше 5 дней, каждый день в 7:40
 
Создаём пользователя для репликации, с помощью которого будем забирать данные:

    sudo -u postgres psql -c 'CREATE ROLE replication WITH REPLICATION PASSWORD 'tsoAYHhdrKmq+oP8dl7M' LOGIN;'
    
В файл */etc/postgresql/9.6/main/pg_hba.conf* добавляем следующее:

    # TYPE  DATABASE        USER            ADDRESS                         METHOD
    host    replication     replication     ip_address_slave_server         md5

Перезапускаем master сервер:
    
    systemctl restart postgresql && systemctl status postgresql
    
### Настраиваем Slave (Replica)

В файл */etc/postgresql/9.6/main/postgresql.conf* добавляем всё тоже самое что и на master сервере и добавляем:
    
    # Определяет, можно ли будет подключаться к серверу и выполнять запросы в процессе восстановления.
    # Значение по умолчанию — off (подключения не разрешаются). Задать этот параметр можно только при запуске сервера.
    # Данный параметр играет роль только в режиме ведомого сервера или при восстановлении архива.
    hot_standby = on

В нашем случае база на master сервере будет постоянно обновляться и дополняться, нам необходимо перенести на slave сервер начальное состояние.
Для начала удалим все данные на **SLAVE** сервере:

    rm -Rf /var/lib/postgresql/9.6/main/*
    
Копируем данные с **master** сервера, после выполнения команды надо будет ввести пароль, который мы задавали при создании роли **replication** на master:

    su postgres -c "pg_basebackup -h 192.168.1.1 -D /var/lib/postgresql/9.6/main -R -P -U replication --xlog-method=stream"

В результате выполнении команды будет создан файл **recovery.conf** с примерно следующим содержимым:

    standby_mode = 'on'
    primary_conninfo = 'user=replication password=tsoAYHhdrKmq+oP8dl7M host=192.168.1.1 port=5432 sslmode=prefer sslcompression=1 krbsrvname=postgres'

Добавляем в этот файл строку:

    trigger_file = '/var/lib/postgresql/9.6/main/trigger_file'
    
**trigger_file** нам нужен, чтобы переключить salve сервер в master, т.е. чтобы вывести slave сервер из режима только чтение.

Запускаем на salve сервер:
    
    systemctl start postgresql && systemctl status postgresql

Проверяем что репликация запущена и работает.

На **master**:
    
    ps wax|grep sender
    
получаем примерно такой вот ответ:

    28713 pts/0   S+    0:00 postgres: 9.6/main: wal sender process replication 
                          192.168.1.2(00000) streaming 0/150000000
                          
На **slave (replica)**:

    ps wax|grep receiver
    
получаем:

    1223 ?        Ss     0:00 postgres: 9.6/main: wal receiver process   streaming 0/150000000
    
Вроде всё работает :)

Проверим поглубже:

**master**:
    
    sudo -u postgres psql -c 'SELECT *,pg_xlog_location_diff(s.sent_location,s.replay_location) byte_lag FROM pg_stat_replication s;'
    
Получаем список серверов подлюкченных к **master**, обращаем внимание на колонку **byte_lag**, она покажет нам на сколько отстаёт **replica** от **master** в байтах.

**slave (replica)**:

    sudo -u postgres psql -c "SELECT now()-pg_last_xact_replay_timestamp();"
    
Получим когда последний раз была произведена сихронизация с **master**.

         ?column?     
    -----------------
     00:00:02.123112


Для того чтобы переключиться на новую базу, нам необходимо:

1. Изменяем DATABASE_URL приложения на master сервере
        
        vi qstn/current/.env
2. На новом сервере разрешаем подключение к базе, обновляем
        
        /etc/postgresql/9.6/main/pg_hba.conf
        host  all  all  master_server_ip/32 trust
3. Останавливаем приложение на master puma:stop, sidekiq:stop
4. Проверка что нет рассинхрона в данных с slave
    
        sudo -u postgres psql -c 'SELECT *,pg_xlog_location_diff(s.sent_location,s.replay_location) byte_lag FROM pg_stat_replication s;’
5. Как только в п. 3, в колонке byte_lag == 0. Останавливаем master postgres.
        
        systemctl stop postgresql
6. Переводим slave в master
      
        touch /var/lib/postgresql/9.6/main/trigger_file
7. Запускаем приложение на master сервере.
8. Проверяем что всё работает с базой на новом сервере
9. Переключаем DNS, запускаем приложение на новом сервере.

