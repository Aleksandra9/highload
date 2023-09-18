<h1>Полусинхронная репликация</h1>

<ol>
<li>Поднимаем мастер</li>

```
docker run -dit -v "$PWD/volumes/pgmaster/:/var/lib/postgresql/data" -e POSTGRES_PASSWORD=pass -p "5432:5432" --restart=unless-stopped --network=pgnet --name=pgmaster postgres
```

<li>Меняем postgresql.conf на мастере</li>

```
ssl = off
wal_level = replica
max_wal_senders = 4 # expected slave num
```

<li>Создаем пользователя для репликации</li>

```
docker exec -it pgmaster su - postgres -c psql
create role replicator with login replication password 'pass';
exit
```

<li>Добавляем запись в pgmaster/pg_hba.conf</li>

```
host    replication     replicator      172.28.0.0/16           md5
```

<li>Применяем изменения, рестарт мастера</li>

```
docker restart pgmaster
```

<li>Делаем бэкап для реплики</li>

```
docker exec -it pgmaster bash
mkdir /pgslave
pg_basebackup -h pgmaster -D /pgslave -U replicator -v -P --wal-method=stream
exit
```

<li>Копируем директорию</li>

```
docker cp pgmaster:/pgslave volumes/pgslave/
```

<li>Создадем файл для реплика</li>

```
touch volumes/pgslave/standby.signal
```

<li>Меняем postgresql.conf на реплике pgslave</li>

```
primary_conninfo = 'host=pgmaster port=5432 user=replicator password=pass application_name=pgslave'
```

<li>Запускаем реплику pgslave</li>

```
docker run -dit -v "$PWD/volumes/pgslave/:/var/lib/postgresql/data" -e POSTGRES_PASSWORD=pass -p "15432:5432" --network=pgnet --restart=unless-stopped --name=pgslave postgres
```

<li>Проверяем, что реплика работают в асинхронном режиме на pgmaster</li>

```
docker exec -it pgmaster su - postgres -c psql
select application_name, sync_state from pg_stat_replication;
exit;
```

```
application_name | sync_state
------------------+------------
pgslave          | async
(1 row)
```

<li>Переключаем чтение на pgslave</li>
нагрузка до начала чтения

```
CONTAINER ID   NAME       CPU %     MEM USAGE / LIMIT     MEM %     NET I/O         BLOCK I/O        PIDS
ceb31c780906   pgmaster   0.00%     508.6MiB / 7.676GiB   6.47%     108MB / 902kB   4.94MB / 746MB   9
1e79d49b028e   pgslave    0.09%     39.3MiB / 7.676GiB    0.50%     120kB / 201kB   32.8kB / 4.1kB   15
```
в процессе чтения с pgslave

```
CONTAINER ID   NAME       CPU %     MEM USAGE / LIMIT     MEM %     NET I/O         BLOCK I/O        PIDS
ceb31c780906   pgmaster   0.00%     508.6MiB / 7.676GiB   6.47%     108MB / 904kB   4.94MB / 746MB   9
1e79d49b028e   pgslave    248.82%   184.1MiB / 7.676GiB   2.34%     556kB / 111MB   32.8kB / 4.1kB   15
```

<li>Создаем еще одну реплику на основе старого бэкапа</li>

Копируем директорию

```
docker cp pgmaster:/pgslave volumes/pgslavenew/
```

Создадем файл для реплика

```
touch volumes/pgslavenew/standby.signal
```

Меняем postgresql.conf на реплике pgslavenew

```
primary_conninfo = 'host=pgmaster port=5432 user=replicator password=pass application_name=pgslavenew'
```

Запускаем реплику pgslavenew

```
docker run -dit -v "$PWD/volumes/pgslavenew/:/var/lib/postgresql/data" -e POSTGRES_PASSWORD=pass -p "25432:5432" --network=pgnet --restart=unless-stopped --name=pgslavenew postgres
```

<li>Проверяем, что новая реплика добавилась</li>

```
docker exec -it pgmaster su - postgres -c psql
select application_name, sync_state from pg_stat_replication;
exit;
```

```
 application_name | sync_state 
------------------+------------
 pgslavenew       | async
 pgslave          | async
(2 rows)
```

<li>Включаем кворумную синхронную репликацию: изменяем конфиг pgmaster</li>

```
synchronous_commit = on
synchronous_standby_names = 'ANY 1 (pgslave, pgslavenew)'
```

<li>Включаем кворумную синхронную репликацию: перечитываем конфиг</li>

```
docker exec -it pgmaster su - postgres -c psql
select pg_reload_conf();
exit;
```

```
 application_name | sync_state 
------------------+------------
 pgslavenew       | quorum
 pgslave          | quorum
(2 rows)
```

<li>Делаем тестовую нагрузку на запись в таблицу</li>

```
create table if not exists test_data
(
    int_value bigserial,
    char_value varchar
);
```

<li>Убиваем одну из реплик</li>

```
docker stop pgslave
```

<li>Проверяем количество ошибок в записи</li>

```
Label	        Sample	    Average	    Error %	    Throughput
Thread Group	6613	    8	        0	        114.2319186
```

<li>Заканчиваем тестовую запись</li>

<li>Убиваем мастер pgmaster</li>

```
docker stop pgmaster
```

<li>Запромоутим реплику pgslavenew</li>

```
docker exec -it pgslavenew su - postgres -c psql
select pg_promote();
exit;
```

<li>Настраиваем репликацию на pgslavenew: меняем файл конфигурации</li>

```
synchronous_commit = on
synchronous_standby_names = 'ANY 1 (pgmaster, pgslave)'
```

```
docker exec -it pgslavenew su - postgres -c psql
select pg_reload_conf();
exit;
```

<li>Подключим вторую реплику pgslave к новому мастеру pgslavenew</li>

```
primary_conninfo = 'host=pgslavenew port=5432 user=replicator password=pass application_name=pgslave'
```

<li>Восстанавливаем реплику pgslave</li>

```
docker start pgslave
```

<li>Проверяем количество данных на репликах</li>

```
docker exec -it pgslavenew su - postgres -c psql
select count(*) from network.test_data;
exit;
```

pgslavenew - новый master
```
 count 
-------
 10435
(1 row)
```

pgslave - выключали во время записи
```
 count 
-------
 10435
(1 row)
```

pgmaster - старый master 
```
 count 
-------
 10435
(1 row)
```

</ol>
