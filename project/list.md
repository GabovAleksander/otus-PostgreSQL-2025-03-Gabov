1. Создаем 4 виртуальных машины
   
3. Устанавливаем UBUNTU

4. Прописываем статические адреса на 3-х серверах кластера etc\netplan\50-cloud-init.yaml

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.3.230/24
      routes:
        - to: default
          via: 192.168.3.1
      nameservers:
        addresses: [192.168.3.1, 8.8.8.8]

```

```
    192.168.3.230 pg00.domain.local
    192.168.3.231 pg01.domain.local
    192.168.3.232 pg02.domain.local
    192.168.3.240 ha.domain.local
```

3. Отредактировать файл etc/hosts на всех трех серверах:

   <img width="733" height="353" alt="image" src="https://github.com/user-attachments/assets/f7f41fd6-1ee7-4a1c-8910-e53ea5b48661" />


```
    192.168.3.230 pg00.domain.local
    192.168.3.231 pg01.domain.local
    192.168.3.232 pg02.domain.local
    192.168.3.240 ha.domain.local
```

4. Установка Postgres
Производится на pg00, pg01, pg02. Выполняется из под пользователя root.

4.1.	Региструем репозитарий postgresql.org:

```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```
 
4.2. Устанавливаем Postgres 17:

```
apt install postgresql-17
```
Задаем пароль для роли postgres:

```
sudo su postgres
psql
\password
```

Далее два раза вводим пароль (в нашем случае — password).
После этого в той же утилите psql создаем пользователя replicator:
create user replicator replication login encrypted password 'password';
Создаем пользователя pgbouncer (если он будет использоваться):

```
create user pgbouncer login encrypted password 'password';
```

и выходим из psql:
\q
А затем возвращаемся на root:
Ctrl+D
Редактируем файл pg_hba.conf, чтобы разрешить удаленные подключения (обычные и для репликации):
```
mcedit /etc/postgresql/17/main/pg_hba.conf
```
и меняем строки 

```
host all all 127.0.0.1/32 scram-sha-256
host replication all 127.0.0.1/32 scram-sha-256
```
на 

```
host all all 0.0.0.0/0 scram-sha-256
host replication all 0.0.0.0/0 scram-sha-256
```

Редактируем файл postgresql.conf, чтобы разрешить прослушивание для всех IP-адресов:
```
mcedit /etc/postgresql/17/main/postgresql.conf
```

В строке 
```
#listen_address = 'localhost'
```

снимаем коментарий и ставим звездочку:
```
listen_address = '*'
```
После этого первую ноду (pg00) нужно перезапустить:
```
systemctl restart postgresql
```
(остальные необязательно).
Производится только на pg01 и pg02 (второй и третьей нодах):
На второй и третьей ноде нужно удалить содержимое каталога pgdata, поскольку оно будет отреплицировано с первой ноды при развертывании кластера:
```
systemctl stop postgresql
rm -rf /var/lib/postgresql/17/main/*
```
3. Установка ETCD
Производится на pg00, pg01, pg02. Выполняется от пользователя root.
Скачиваем дистрибутив etcd:

```
cd /tmp
wget https://github.com/etcd-io/etcd/releases/download/v3.5.5/etcd-v3.5.5-linux-amd64.tar.gz
```

Разархивируем:
```
tar xzvf etcd-v3.5.5-linux-amd64.tar.gz
```

Перемещаем исполняемые файлы etcd в usr/local/bin:
```
sudo mv /tmp/etcd-v3.5.5-linux-amd64/etcd* /usr/local/bin/
```

Создаем пользователя, от которого будет работать служба etcd:
```
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
```
Создаем каталоги etcd, меняем владельца и права:

```
mkdir /opt/etcd
mkdir /etc/etcd
mkdir /var/lib/etcd
chown -R etcd:etcd /opt/etcd /var/lib/etcd /etc/etcd
chmod -R 700 /opt/etcd/ /var/lib/etcd /etc/etcd
```

Далее создаем конфиги etcd. Для каждой ноды — свой отдельный.

```
mcedit /etc/etcd/etcd.conf
```

Текст конфига для pg00:

```
ETCD_NAME="pg00"
ETCD_LISTEN_CLIENT_URLS="http://192.168.3.230:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.3.230:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.3.230:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.3.230:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://192.168.3.230:2380,etcd2=http://192.168.3.231:2380,etcd3=http://192.168.3.232:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

Текст конфига для pg01:

```
ETCD_NAME="pg01"
ETCD_LISTEN_CLIENT_URLS="http://192.168.3.231:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.3.231:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.3.231:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.3.231:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://192.168.3.230:2380,etcd2=http://192.168.3.231:2380,etcd3=http://192.168.3.232:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

Текст конфига для pg02:

```
ETCD_NAME="pg02"
ETCD_LISTEN_CLIENT_URLS="http://192.168.3.232:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.3.232:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.3.232:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.3.232:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://192.168.3.230:2380,etcd2=http://192.168.3.231:2380,etcd3=http://192.168.3.232:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

Далее на каждой ноде делаем etcd службой (конфиг одинаковый):

```
mcedit /etc/systemd/system/etcd.service
```
Текст конфига:

```
[Unit]
Description=Etcd Server
Documentation=https://github.com/etcd-io/etcd
After=network.target
After=network-online.target
Wants=network-online.target
  
[Service]
User=etcd
Type=notify
#WorkingDirectory=/var/lib/etcd/
WorkingDirectory=/opt/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd"
Restart=on-failure
LimitNOFILE=65536
IOSchedulingClass=realtime
IOSchedulingPriority=0
Nice=-20
 
[Install]
WantedBy=multi-user.target
Далее настраиваем автозапуск службы etcd и ее запускаем:
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
И проверяем, что она запустилась:
systemctl status etcd
```

Если по какой-то причине служба не запускается, можно просмотреть информацию об ошибке при помощи команды
```
journalctl -e
```

Для проверки можно посмотреть информацию о нодах кластера (и узнать, кто лидер):
```
ETCDCTL_API=2 etcdctl member list
```
и проверить здоровье нод:
```
etcdctl endpoint health --cluster -w table
```


4. Установка Patroni
Устанавливаем пакеты для работы с Python:

```
apt -y install python3 python3-pip python3-dev python3-psycopg2 libpq-dev
```

Через PIP ставим пакеты Python:

```
pip3 install psycopg2 --break-system-packages
pip3 install psycopg2-binary --break-system-packages
pip3 install patroni --break-system-packages
pip3 install python-etcd --break-system-packages
```

Создаем каталог конфигов Patroni:
```
mkdir /etc/patroni/
```

Создаем файл конфигурации Patroni:

```
mcedit /etc/patroni/patroni.yml
```

Текст patroni.yml для первой ноды (pg00):


```
scope: postcluster # одинаковое значение на всех узлах
name: pg00 # разное значение на всех узлах
namespace: quarz # одинаковое значение на всех узлах
restapi:
  listen: 192.168.3.230:8008 # разное значение на всех узлах
  connect_address: 192.168.3.230:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: QwertyP

etcd:
  hosts: 192.168.3.230:2379, 192.168.3.231:2379, 192.168.3.232:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replicator scram-sha-256

postgresql:
  listen: 192.168.3.230,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.3.230:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: Qwerty2
    superuser:
      username: postgres
      password: Qwerty1
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false


  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```

Для второй ноды (pg01):

```
scope: postcluster # одинаковое значение на всех узлах
name: pg01 # разное значение на всех узлах
namespace: quarz # одинаковое значение на всех узлах

restapi:
  listen: 192.168.3.231:8008 # разное значение на всех узлах
  connect_address: 192.168.3.231:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: QwertyP

etcd:
  hosts: 192.168.3.230:2379, 192.168.3.231:2379, 192.168.3.232:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replicator scram-sha-256

postgresql:
  listen: 192.168.3.231,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.3.231:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: Qwerty2
    superuser:
      username: postgres
      password: Qwerty1
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```

На третьей ноде (pg02):

```
scope: postcluster # одинаковое значение на всех узлах
name: pg02 # разное значение на всех узлах
namespace: quarz # одинаковое значение на всех узлах

restapi:
  listen: 192.168.3.232:8008 # разное значение на всех узлах
  connect_address: 192.168.3.232:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: QwertyP

etcd:
  hosts: 192.168.3.230:2379, 192.168.3.231:2379, 192.168.3.232:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replicator scram-sha-256

postgresql:
  listen: 192.168.3.232,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.3.232:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: Qwerty2
    superuser:
      username: postgres
      password: Qwerty1
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```


Далее — назначаем права на каждой ноде:

```
chown postgres:postgres -R /etc/patroni
chmod 700 /etc/patroni
mkdir /var/lib/pgsql_stats_tmp
chown postgres:postgres /var/lib/pgsql_stats_tmp
```

Чтобы проверить, все ли сделано правильно, на всех нодах выполняем команду

```
sudo -u postgres patroni /etc/patroni/patroni.yml
```

Каждый из серверов должен запуститься и написать, он в роли лидера или в роли secondary.
Определяем Patroni как службу (на всех трех нодах одинаково):

```
mcedit /etc/systemd/system/patroni.service
```

Текст конфига службы может быть таким:

```
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres

# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf

# Start the patroni process
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=60

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
Переводим Patroni в автозапуск, стартуем и проверяем:
systemctl daemon-reload
systemctl enable patroni
systemctl start patroni
systemctl status patroni
Просмотреть состояние кластера можно командой
patronictl -c /etc/patroni/patroni.yml list
Для failover:
patronictl -c /etc/patroni/patroni.yml failover
```


5. Установка PGBouncer
Производится на всех нодах (pg00, pg01, pg02). Выполняется от пользователя root
Устанавливаем PGBouncer:

```
apt -y install pgbouncer
```

Сохраняем исходный вариант файла конфигурации:

```
mv /etc/pgbouncer/pgbouncer.ini /etc/pgbouncer/pgbouncer.ini.origin
```

Открываем его на редактирование:

```
mcedit /etc/pgbouncer/pgbouncer.ini
```

Вставляем текст (на каждой ноде одинаковый):

```
[databases]
postgres = host=127.0.0.1 port=5432 dbname=postgres
* = host=127.0.0.1 port=5432

[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
unix_socket_dir = /var/run/postgresql
auth_type = md5
#auth_type = trust
auth_file = /etc/pgbouncer/userlist.txt
auth_user = postgres
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
#admin_users = pgbouncer, postgres
admin_users = postgres
ignore_startup_parameters = extra_float_digits,geqo,search_path

pool_mode = session
#pool_mode = transaction
server_reset_query = DISCARD ALL
max_client_conn = 10000
#default_pool_size = 20
reserve_pool_size = 1
reserve_pool_timeout = 1
max_db_connections = 1000
#max_client_conn = 900
default_pool_size = 500
pkt_buf = 8192
listen_backlog = 4096
log_connections = 1
log_disconnections = 1
Создаем файл userlist.txt со списком пользователей для работы через PGBouncer:
mcedit /etc/pgbouncer/userlist.txt
```


Добавляем в него пользователей:

```
"postgres" "Qwerty1"
"pgbouncer" "Qwerty3"
```

Перезапускаем PGBouncer:

```
systemctl restart pgbouncer
```

Для проверки можно подключиться к серверу Postgres через PGBouncer (порт 6432):

```
su - postgres
psql -p 6432 -h 127.0.0.1 -U postgres postgres
```

Установка HAProxy
Производится на компьютере HA. Выполняется от пользователя root
Устанавливаем HAProxy:

```
apt -y install haproxy
```

Сохраняем исходный файл конфигурации:

```
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.origin
```

Создаем новый файл конфигурации:

```
mcedit /etc/haproxy/haproxy.cfg
```

И записываем в него текст:

```
global

        maxconn 10000
        log     127.0.0.1 local2

defaults
        log global
        mode tcp
        retries 2
        timeout client 30m
        timeout connect 4s
        timeout server 30m
        timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:7432
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.3.230:6432 maxconn 10000 check port 8008
    server node2 192.168.3.231:6432 maxconn 10000 check port 8008
    server node3 192.168.3.232:6432 maxconn 10000 check port 8008
```

Далее перезагружаем HAProxy:

```
sudo systemctl restart haproxy
```
и проверяем работоспособность:

```
sudo systemctl status haproxy
```

Для проверки можно подключиться через psql на IP-адрес HAProxy к порту 7432.

 


 
