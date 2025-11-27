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
sudo -u postgres psql
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
### Часть 2.  Управление параметрами на уровне экземпляра
**Выполненные задачи:**
1.Изменение через ALTER SYSTEM: Используя команду ALTER SYSTEM, установил для параметра
work_mem новое значение. Убедился, что изменение записалось в файл postgresql.auto.conf
(используйте функцию pg_read_file). Применил изменение, перечитав конфигурацию (SELECT
pg_reload_conf();). Проверил новое значение параметра и его источник в pg_settings.
2. Изменение через дополнительный файл: Создал файл в каталоге, указанном в директиве
include_dir основного конфигурационного файла. Установил в этом файле значение для
параметра log_min_duration_statement. Применил изменение и проверьте его.
3. Ошибка в конфигурации: Намеренно внес синтаксическую ошибку в один из
конфигурационных файлов (например, invalid_value вместо числового значения). Попытался
перечитать конфигурацию. Изучил представление pg_file_settings, чтобы найти запись об
ошибке. Исправил ошибку и перечитал конфигурацию.

**КОМАНДЫ**
**Задача 1:**
```sql
ALTER SYSTEM SET work_mem = '16MB';
```
```sql
SELECT pg_read_file('postgresql.auto.conf');
```
```sql
SELECT pg_reload_conf();
```
```sql
SELECT name, setting, unit, source, sourcefile
FROM pg_settings
WHERE name = 'work_mem';
```
**Задача 2:**
```sql
\q
sudo nano /etc/postgresql/16/main/postgresql.conf
```
-- Найшел строку и убрал # include_dir = 'conf.d'
```sql
sudo mkdir -p /etc/postgresql/16/main/conf.d
sudo nano /etc/postgresql/16/main/conf.d/log_duration.conf
```
--вставил 
```sql
# Логирование всех SQL-запросов дольше 500 мс
log_min_duration_statement = 500
```

```sql
sudo systemctl reload postgresql
sudo systemctl restart postgresql
sudo -u postgres psql
SHOW log_min_duration_statement;
```
**Задача 3:**

```sql
cd /etc/postgresql/16/main/conf.d
```
```sql
sudo nano log_duration.conf
```
```text
вставил новый текст
log_min_duration_statement = asasasasasa 
```
```sql
sudo -u postgres psql
SELECT * FROM pg_file_settings WHERE sourcefile LIKE '%log_duration.conf%';
```
```sql
-- Вернулся исправить ошибки
sudo nano /etc/postgresql/16/main/conf.d/log_duration.conf
log_min_duration_statement = 500
sudo systemctl reload postgresql
```
```sql
sudo -u postgres psql
SHOW log_min_duration_statement;
```
**Фрагменты вывода:** 

**Задача 1:**
```text
ALTER SYSTEM
```
```text
pg_read_file                      
-------------------------------------------------------
 # Do not edit this file manually!                    +
 # It will be overwritten by the ALTER SYSTEM command.+
 work_mem = '16MB'                                    +
 
(1 row)

```
```text
pg_reload_conf 
----------------
 t
(1 row)

```
```text
   name   | setting | unit |       source       |                    sourcefile                    
----------+---------+------+--------------------+--------------------------------------------------
 work_mem | 16384   | kB   | configuration file | /var/lib/postgresql/16/main/postgresql.auto.conf
(1 row)
```
**Задача 2:**

```text
#------------------------------------------------------------------------------
# CONFIG FILE INCLUDES
#------------------------------------------------------------------------------

# These options allow settings to be loaded from files other than the
# default postgresql.conf.  Note that these are directives, not variable
# assignments, so they can usefully be given more than once.

include_dir = 'conf.d'          # include files ending in '.conf' from
                                        # a directory, e.g., 'conf.d'
#include_if_exists = '...'              # include file only if it exists
#include = '...'                        # include file
```
```text
student:~$ sudo mkdir -p /etc/postgresql/16/main/conf.d
student:~$ sudo nano /etc/postgresql/16/main/conf.d/log_duration.conf
student:~$ sudo systemctl reload postgresql
student:~$ sudo systemctl restart postgresql
student:~$ sudo -u postgres psql
psql (16.9 (Ubuntu 16.9-1.pgdg24.04+1))
Type "help" for help.

postgres=# SHOW log_min_duration_statement;
 log_min_duration_statement 
----------------------------
 500ms
(1 row)

postgres=# 

```

**Задача 3:**
```text
student:/etc/postgresql/16/main/conf.d$ 
```
```text
-- Види ошибку
                    sourcefile                    | sourceline | seqno |            name            |   setting   | applied |            error             
--------------------------------------------------+------------+-------+----------------------------+-------------+---------+------------------------------
 /etc/postgresql/16/main/conf.d/log_duration.conf |          1 |    25 | log_min_duration_statement | asasasasasa | f       | setting could not be applied
(1 row)
```
```text
psql (16.9 (Ubuntu 16.9-1.pgdg24.04+1))
Type "help" for help.

postgres=# SHOW log_min_duration_statement;
 log_min_duration_statement 
----------------------------
 500ms
(1 row)
```

###часть 3: Управление параметрами на уровне сеанса
1. Команда SET: В рамках сеанса изменил значение параметра work_mem с помощью SET.
Проверил новое значение. Завершил транзакцию с помощью ROLLBACK и проверил значение
параметра again. Объясните результат.
2. Команда SET LOCAL: Открыл транзакцию (BEGIN). Inside the transaction, use SET LOCAL to
change the work_mem parameter. Verify the change. After committing the transaction (COMMIT), check
the parameter value again. Explain the result.
3. Пользовательский параметр: Создал и установил значение для пользовательского
параметра (имя должно содержать точку, например, app.my_setting). Прочитал его значение
с помощью current_setting.

**ЗАДАЧА 1:**
**КОМАНДЫ И РЕЗУЬТАТЫ**
```sql
SHOW work_mem;
```
Проверил текущее значение work_mem
```text
 work_mem 
----------
 16MB
(1 row)
```
Изменил значение для текущего сеанса с помощью SET
```sql
SET work_mem = '64MB';
```
```text
SET
```
Проверил, что новое значение применилось
```sql
SHOW work_mem;
```
```text
 work_mem 
----------
 64MB
(1 row)
```
Проверка в транзакции
```sql
BEGIN;
SET work_mem = '128MB';
SHOW work_mem;
```
```text
 work_mem 
----------
 128MB
(1 row)
```
```sql
ROLLBACK;
SHOW work_mem;
```
```text
 work_mem 
----------
 64MB
(1 row)
```
**Выввод:**
SET изменяет параметр только для текущего сеанса, а если используется внутри транзакции, изменение действует до конца транзакции.
После ROLLBACK все изменения, сделанные в транзакции, отменяются, поэтому work_mem возвращается к значению, которое было установлено до транзакции (64MB в нашем примере).
Это не влияет на глобальные настройки в postgresql.conf.

**Задача 2:**
```sql
SHOW work_mem;
```
```text
 work_mem 
----------
 64MB
(1 row)
```
--Открываем транзакцию, Меняем значение с помощью SET LOCAL, Проверяем внутри транзакции
```sql
BEGIN;
SET LOCAL work_mem = '128MB';
SHOW work_mem;
```

```text
 work_mem 
----------
 128MB
(1 row)
```
-- Завершаем транзакцию, Проверяем значение снова
```sql
COMMIT;
```
```text
COMMIT
```
```sql
SHOW work_mem;
```
```text
postgres=# SHOW work_mem;
 work_mem 
----------
 64MB
(1 row
```
-- Вывод: Как только транзакция завершается (COMMIT или ROLLBACK), параметр автоматически возвращается к предыдущему значению.

**ЗАДАЧА 3:**
--Устанавливаем пользовательский параметр
```sql
sudo -u postgres psql
```
```text
SET
```
-- Считываем значение с помощью current_setting
```sql
SELECT current_setting('app.my_setting');
```
```text
 current_setting 
-----------------
 test_value
(1 row)
```

###котрольные вопросы###
Параметры с контекстом postmaster требуют перезапуска сервера, потому что влияют на процесс и память при старте. Параметры sighup применяются после reload без перезапуска. Параметры user можно менять в сеансе командой SET, они действуют сразу и локально.
ALTER SYSTEM меняет конфигурацию глобально и записывает её в postgresql.auto.conf, после reload действует для всех. SET меняет параметр только в текущем сеансе, SET LOCAL — только внутри транзакции.
Ошибку в конфигурации находят через reload и просмотр pg_file_settings, где видно файл, параметр и текст ошибки. После исправления файла выполняют повторный reload.

###ВЫВОД###
Изучил базовые компоненты архитектуры PostgreSQL (процессы, память) и получил практические навыки управления конфигурационными параметрами сервера на разных уровнях (экземпляр, сеанс). Освоил работу с основными и дополнительными файлами конфигурации, а также с представлениями pg_settings и pg_file_settings.
