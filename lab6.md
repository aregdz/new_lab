# Лабораторная работа №06: Блокировки и мониторинг
##Цель работы: Изучить систему блокировок в PostgreSQL и методы мониторинга активности сервера.
Получить практические навыки анализа статистики, диагностики блокировок и взаимоблокировок,
использования инструментов мониторинга.
**Дата:** 2025-11-28
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Джараян Арег Александрович
## Теоретическая часть (краткое содержание):
**Блокировки: Механизм, обеспечивающий согласованность данных при параллельном доступе.**
**Бывают разных уровней: объекты, строки, буферы в памяти.**
**Мониторинг: Набор представлений и функций для отслеживания активности сервера, статистики
использования объектов и блокировок.**
**Взаимоблокировка (Deadlock): Ситуация, когда две или более транзакции ожидают друг друга,
освобождения ресурсов.**

# ПРАКТИЧЕСКАЯ ЧАСТЬ
##Модуль 1: Мониторинг активности
**1. Статистика таблиц: В новой базе данных создалйте таблицу monitor_test (id INT). Вставил несколько строк,
затем удалил все. Изучил статистику обращений к таблице в pg_stat_all_tables (n_tup_ins, n_tup_del, n_live_tup, n_dead_tup). Сопоставил с действиями. Выполнил VACUUM. Снова проверил статистику. Объяснил изменения.**
```bash
ps -ef | grep -E "post(gres|master)|checkpointer|writer|wal" | grep -v grep
```
-- Фрагменты вывода:
```text
/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres     797     787  0 08:34 ?        00:00:00 postgres: 16/main: checkpointer 
postgres     798     787  0 08:34 ?        00:00:00 postgres: 16/main: background writer 
postgres     977     787  0 08:34 ?        00:00:00 postgres: 16/main: walwriter 
postgres     978     787  0 08:34 ?        00:00:02 postgres: 16/main: autovacuum launcher 
postgres     979     787  0 08:34 ?        00:00:00 postgres: 16/main: logical replication launcher 
student     6148    1601  0 08:46 pts/0    00:00:00 /usr/lib/postgresql/16/bin/psql
```


**2. Взаимоблокировка:
Создал ситуацию взаимоблокировки двух транзакций (например, изменение двух строк в
разном порядке). Изучил, какая информация записывается в журнал сообщений сервера при обнаружении взаимоблокировки.**
```bash
sudo pg_ctlcluster 16 main stop -m fast
sudo pg_ctlcluster 16 main start
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```
-- Фрагменты вывода:
```text
2025-10-09 08:53:20.243 MSK [797] LOG:  checkpoint starting: shutdown immediate
2025-10-09 08:53:20.250 MSK [797] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.009 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=22877 kB; lsn=0/64E8C0F0, redo lsn=0/64E8C0F0
```


**3. Расширение pg_stat_statements (Практика+):
Установил и настроил расширение pg_stat_statements.
Выполнил несколько произвольных запросов.
Изучил информацию в представлении pg_stat_statements (топ запросов, время
выполнения и т.д.).**
```bash
sudo pg_ctlcluster 16 main stop -m immediate
sudo pg_ctlcluster 16 main start
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```
-- Фрагменты вывода:
```text
2025-10-09 10:06:08.573 MSK [38667] LOG:  database system was not properly shut down; automatic recovery in progress
2025-10-09 10:06:08.786 MSK [38667] LOG:  redo starts at 0/5837B058
2025-10-09 10:06:08.984 MSK [38667] LOG:  invalid record length at 0/5837B058: expected at least 24, got 0
2025-10-09 10:06:09.345 MSK [38667] LOG:  redo done at 0/5837B058 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
2025-10-09 10:06:09.563 MSK [38666] LOG:  checkpoint starting: end-of-recovery immediate wait
2025-10-09 10:06:12.351 MSK [39813] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.003 s, sync=0.002 s, total=0.013 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB; lsn=0/5837B058, redo lsn=0/5837B058
```

Модуль 2: Блокировки объектов
**1. Блокировки при чтении: На уровне изоляции Read Committed прочитал одну строку таблицы по первичному
ключу. Изучил удерживаемые блокировки в pg_locks. Объяснил, какие блокировки и почему были захвачены.**
```sql
CREATE DATABASE lab05_db;
\c lab05_db
CREATE TABLE wal_test(id int PRIMARY KEY, data text);
INSERT INTO wal_test
SELECT g, repeat('x', 200)
FROM generate_series(1, 100000) AS g;
SELECT pg_relation_size('wal_test');
SELECT current_setting('block_size')::int;
WITH rel AS (
  SELECT oid, pg_relation_filenode(oid) AS fn
  FROM pg_class WHERE relname = 'wal_test'
)
SELECT
  count(*) AS buffers
FROM pg_buffercache b
JOIN rel r
  ON b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
 AND b.relfilenode = r.fn
 AND b.relforknumber = 0;

```
-- фрагмент кода
```text
CREATE TABLE
INSERT 0 100000
 pg_relation_size 
------------------
         24100864
(1 row)

 current_setting 
-----------------
            8192
(1 row)

 buffers 
---------
    2942
(1 row)

```

**2. Повышение уровня блокировок: Воспроизвел ситуацию автоматического повышения уровня предикатных блокировок
при чтении строк по индексу. Показал, что это может привести к ложной ошибке сериализации.**
```sql
SELECT count(*) FROM pg_buffercache WHERE isdirty IS TRUE;
CHECKPOINT;
SELECT count(*) FROM pg_buffercache WHERE isdirty IS TRUE;
```
-- фрагмент кода
```text
 count 
-------
  3291
(1 row)

CHECKPOINT

 count 
-------
     0
(1 row)

```

**3. Логирование долгих ожиданий: Настроил запись в журнал сообщений о ожиданиях блокировок > 100 мс (log_lock_waits
= on, deadlock_timeout = 100ms). Создалситуацию длительного ожидания блокировки. Убедил, что сообщение
появилось в логе.**
```sql
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('wal_test');
```
```bash
sudo pg_ctlcluster 16 main restart
```
```sql
WITH rel AS (
  SELECT oid, pg_relation_filenode(oid) AS fn
  FROM pg_class WHERE relname = 'wal_test'
)
SELECT count(*) AS buffers
FROM pg_buffercache b
JOIN rel r
  ON b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
 AND b.relfilenode = r.fn
 AND b.relforknumber = 0;
```
-- фрагмент кода
```text
 pg_prewarm 
------------
       2942
(1 row)
```
```text
 buffers 
---------
       0
```
