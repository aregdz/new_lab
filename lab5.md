### Лабораторная работа №05: Надежность: Журнал предзаписи (WAL)
### Цель работы: Изучить работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. Получить практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL и исследования процессов восстановления после сбоев.
### Теоретическая часть (краткое содержание):
**Буферный кеш: Область общей памяти для кэширования страниц данных, считываемых с диска.
Измененные ("грязные") буферы периодически сбрасываются на диск.
Контрольная точка (Checkpoint): Процесс принудительной записи всех "грязных" буферов на
диск. Ограничивает объем WAL, необходимый для восстановления.**
**Журнал предзаписи (WAL): Циклический журнал, в который записываются все изменения
данных перед тем, как они попадут в основные файлы данных. Обеспечивает надежность и
возможность восстановления после сбоя.**
**Восстановление: Процесс применения WAL-записей, созданных после последней контрольной
точки, к данным на диске для приведения их в согласованное состояние.**


## Практическая часть
###Модуль 1: Процессы и режимы остановки
**1. Поиск процессов: Средствами ОС (например, ps aux | grep postgres) найшел процессы, отвечающие за работу буферного кеша (checkpointer, background writer) и журнала WAL
(walwriter).**
Ищем процессы:
checkpointer — отвечает за контрольные точки.
background writer — пишет грязные страницы из буферного кеша на диск.
walwriter — сбрасывает записи WAL на диск.
```sql
ps aux | grep postgres
```
```text
postgres     766  0.0  3.1 225448 30876 ?        Ss   19:10   0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres     777  0.0  3.0 225808 29684 ?        Ss   19:10   0:00 postgres: 16/main: checkpointer 
postgres     778  0.0  0.8 225592  8856 ?        Ss   19:10   0:00 postgres: 16/main: background writer 
postgres     796  0.0  1.0 225448 10572 ?        Ss   19:10   0:00 postgres: 16/main: walwriter 
postgres     797  0.0  0.9 227044  9420 ?        Ss   19:10   0:00 postgres: 16/main: autovacuum launcher 
postgres     798  0.0  0.8 227016  8344 ?        Ss   19:10   0:00 postgres: 16/main: logical replication launcher 
student     1272  0.0  0.9  26088  9112 pts/0    T    19:12   0:00 /usr/lib/postgresql/16/bin/psql -d mydb -U student
postgres    1273  0.0  2.0 229000 20028 ?        Ss   19:12   0:00 postgres: 16/main: student mydb [local] idle
root        1375  0.0  0.7  19540  7624 pts/0    S+   19:29   0:00 sudo -i -u postgres
root        1376  0.0  0.2  19540  2732 pts/1    Ss   19:29   0:00 sudo -i -u postgres
postgres    1377  0.0  0.5  11268  5292 pts/1    S    19:29   0:00 -bash
postgres    2456  0.0  0.4  13496  4328 pts/1    R+   22:53   0:00 ps aux
postgres    2457  0.0  0.2   9284  2264 pts/1    S+   22:53   0:00 grep postgres
postgres@course:~$ 
```

**2. Остановка Fast: Остановил PostgreSQL в режиме fast (sudo pg_ctlcluster 16 main stop).
Запустил сервер. Просмотрите журнал сообщений сервера (/var/log/postgresql/postgresql-16-main.log). Найшел записи о контрольной точке,
выполненной при завершении работы.**
```sql
sudo pg_ctlcluster 16 main stop -m fast
sudo pg_ctlcluster 16 main start
```
```text
var/log/postgresql/postgresql-16-main.log
```
В логах /var/log/postgresql/postgresql-16-main.log указана запись о том, что сделана контрольная точка перед завершением работы:
database system is shut down

**3. Остановка Immediate:
Остановил PostgreSQL в режиме immediate (sudo pg_ctlcluster 16 main stop -m
immediate). Запустил сервер. Просмотрел журнал сообщений. Найшел записи о восстановлении
после сбоя (recovery). Сравнил с предыдущим случаем.**
```sql
sudo pg_ctlcluster 16 main stop -m immediate
sudo pg_ctlcluster 16 main start
less /var/log/postgresql/postgresql-16-main.log
```
```text
025-06-23 05:48:45.194 MSK [2266] LOG:  starting PostgreSQL 16.9 (Ubuntu 16.9-1
.pgdg24.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24
.04) 13.3.0, 64-bit
2025-06-23 05:48:45.195 MSK [2266] LOG:  listening on IPv4 address "127.0.0.1", 
port 5432
2025-06-23 05:48:45.209 MSK [2266] LOG:  listening on Unix socket "/var/run/post
gresql/.s.PGSQL.5432"
2025-06-23 05:48:45.241 MSK [2269] LOG:  database system was shut down at 2025-06-23 05:48:37 MSK
2025-06-23 05:48:45.259 MSK [2266] LOG:  database system is ready to accept connections
2025-06-23 05:48:55.184 MSK [2266] LOG:  received fast shutdown request
2025-06-23 05:48:55.194 MSK [2266] LOG:  aborting any active transactions
2025-06-23 05:48:55.198 MSK [2266] LOG:  background worker "logical replication launcher" (PID 2272) exited with exit code 1
2025-06-23 05:48:55.200 MSK [2267] LOG:  shutting down
2025-06-23 05:48:55.210 MSK [2267] LOG:  checkpoint starting: shutdown immediate
2025-06-23 05:48:56.959 MSK [2267] LOG:  checkpoint complete: wrote 935 buffers (5.7%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.043 s, sync=1.653 s, total=1.759 s; sync files=308, longest=0.059 s, average=0.006 s; distance=4280 kB, estimate=4280 kB; lsn=0/1922028, redo lsn=0/1922028
2025-06-23 05:48:56.982 MSK [2266] LOG:  database system is shut down
2025-06-23 05:48:58.733 MSK [2610] LOG:  starting PostgreSQL 16.9 (Ubuntu 16.9-1:
```


### Модуль 2: Буферный кеш и контрольные точки
**1. Анализ размера: Создал таблицу wal_test (id INT, data TEXT). Вставил в нее достаточное количество
строк.Определил, сколько страниц на диске занимает таблица (например, с помощью
pg_relation_size(...) / current_setting('block_size')::int).Определил, сколько буферов занимает таблица в кеше (запрос к pg_buffercache).**



**2. Грязные буферы и контрольная точка: Узнал общее количество "грязных" буферов в кеше (запрос к pg_stat_bgwriter или
pg_buffercache). Выполнил команду CHECKPOINT;. Снова проверил количество "грязных" буферов. Объяснил результат.**



**3. Предварительное чтение (pg_prewarm): Подключил расширение pg_prewarm. Загрузил свою таблицу в кеш с помощью pg_prewarm(...). Перезапустил сервер. Проверил, осталась ли таблица в кеше. Проанализировал эффективность метода.**
