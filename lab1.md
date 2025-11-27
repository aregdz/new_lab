# Отчет по лабораторной работе №1
# Архитектура СУБД и конфигурация

**Дата:** 2025-11-27
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Джараян Арег Александрович

## Цель работы
Изучить базовые компоненты архитектуры PostgreSQL (процессы, память) и получить
практические навыки управления конфигурационными параметрами сервера на разных уровнях
(экземпляр, сеанс). Освоить работу с основными и дополнительными файлами конфигурации, а также с
представлениями pg_settings и pg_file_settings.


## Теоретическая часть
**Файлы конфигурации:** postgresql.conf (основной), postgresql.auto.conf (управляемыйкомандой ALTER SYSTEM), файлы в каталогах, подключаемых через include_dir.
**Команды уровня сеанса:** SET (для текущего сеанса или транзакции). Параметры имеют разный контекст (internal, postmaster, sighup, user и др.), определяющий необходимые для их применения действия (перезагрузка сервера, перечитывание конфигурации или немедленное применение).

## Практическая часть

### Часть 1.Исследование параметров и файлов конфигурации

**Выполненные задачи:**
1. Текущая конфигурация: Подключился к серверу с помощью psql. Определилил расположение основного файла конфигурации (postgresql.conf) с помощью команды SHOW config_file;.
2. Анализ параметров: Изучил представление pg_settings. Нашел параметры, для изменениякоторых требуется перезагрузка сервера (context = 'postmaster'). Найшел 2-3 параметра сконтекстом sighup и user.
3. Анализ файлов: Изучил представление pg_file_settings. Определил, из каких файлов и скакими значениями были считаны текущие настройки параметров shared_buffers и work_mem.

**КОМАНДЫ**
**Задача 1:**
```sql
sudйo -u postgres psql
SHOW config_file;
```
**Задача 2:**
```sql
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE context = 'postmaster'
ORDER BY name
LIMIT 5;
```
```sql
SELECT name, setting, unit, short_desc FROM pg_settings WHERE context='sighup' ORDER BY name LIMIT 3;
```
```sql
SELECT name, setting, unit, short_desc FROM pg_settings WHERE context='user' ORDER BY name LIMIT 3;
```
**Задача 3:**
```sql
SELECT name, setting, sourcefile, sourceline, applied
FROM pg_file_settings
WHERE name IN ('shared_buffers', 'work_mem');

```

**Фрагменты вывода:** 

**Задача 1:**
```text
psql (16.9 (Ubuntu 16.9-1.pgdg24.04+1))
Type "help" for help.

postgres=#
```

```text
config_file               
-----------------------------------------
 /etc/postgresql/16/main/postgresql.conf
(1 row
```

**Задача 2:**
```text
            short_desc                                   
-------------------------------------+-----------+------+--------------------------------------------------------------------------------
 archive_mode                        | off       |      | Allows archiving of WAL files using archive_command.
 autovacuum_freeze_max_age           | 200000000 |      | Age at which to autovacuum a table to prevent transaction ID wraparound.
 autovacuum_max_workers              | 3         |      | Sets the maximum number of simultaneously running autovacuum worker processes.
 autovacuum_multixact_freeze_max_age | 400000000 |      | Multixact age at which to autovacuum a table to prevent multixact wraparound.
 bonjour                             | off       |      | Enables advertising the server via Bonjour.
(5 rows)
```
```text
          name           |  setting   | unit |                              short_desc                              
-------------------------+------------+------+----------------------------------------------------------------------
 archive_cleanup_command |            |      | Sets the shell command that will be executed at every restart point.
 archive_command         | (disabled) |      | Sets the shell command that will be called to archive a WAL file.
 archive_library         |            |      | Sets the library that will be called to archive a WAL file.
(3 rows)
```
```text

        name         | setting | unit |                                  short_desc                                  
---------------------+---------+------+------------------------------------------------------------------------------
 application_name    | psql    |      | Sets the application name to be reported in statistics and logs.
 array_nulls         | on      |      | Enable input of NULL elements in arrays.
 backend_flush_after | 0       | 8kB  | Number of pages after which previously performed writes are flushed to disk.
(3 rows)

```
**Задача 3:**
```text
----------------+---------+-----------------------------------------+------------+---------
 shared_buffers | 128MB   | /etc/postgresql/16/main/postgresql.conf |        130 | t
(1 row)
~
```

