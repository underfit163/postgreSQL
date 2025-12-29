## Работа с postgres из docker

psql -U user -d pg

\du

\conninfo

psql -d база -U роль -h узел -p порт

psql -h 127.0.0.1 -U user -d pg

![img.png](images/img.png)

![img_2.png](images/img_2.png)

* ? - список команд psql
* \help - список команд SQL
* \h - команда синтаксис команды SQL
* \q - выход(до 11 версии)

**Создать новый кластер Постгреса**

Из под пользователя Линукс postgres (если мы открыли новую консоль под student выполнить sudo su postgres)

pg_createcluster 14 main2

pg_ctlcluster 14 main2 start


**Подключиться из psql**

посмотрим список кластеров

pg_lsclusters

psql -p 5433


**Удалить созданный кластер**

pg_ctlcluster 14 main2 stop

pg_dropcluster 14 main2

#### Конфигуриование постгреса
show all;

Можно разделить параметры по контексту (полю context в запросе)

select name, context from pg_settings;

https://postgrespro.ru/docs/postgresql/14/view-pg-settings

* самые простые на сеанс имеют контекст user
* для сессии уже повыше уровень - backend и требуются соответствующие права
* есть уровень superuser - для суперпользователя
* остальные настройки уже более глобальные и обычно требуют перезапуска сервера

![img_4.png](images/img_4.png)

![img_5.png](images/img_5.png)

![img_6.png](images/img_6.png)

![img_7.png](images/img_7.png)

![img_9.png](images/img_9.png)
![img_10.png](images/img_10.png)

2 онлайн конфигуратора. Просто указывайте параметры вашего сервера и профиль нагрузки и вам предложат оптимальные настройки:

https://pgtune.leopard.in.ua/#/

http://pgconfigurator.cybertec.at/

Сконфигурируем нужные нам настройки, добавляем в конец конфигурационного файла и перезапускаем кластер PostgreSQL.

## Архитектура постгреса
![img_11.png](images/img_11.png)

![img_3.png](images/img_3.png)

![img_12.png](images/img_12.png)

![img_13.png](images/img_13.png)
![img_14.png](images/img_14.png)

## Вакуум
![img_15.png](images/img_15.png)

![img_16.png](images/img_16.png)

Как амазон решил проблему с автовакуумом:
https://aws.amazon.com/ru/blogs/database/a-case-study-of-tuning-autovacuum-in-amazon-rds-for-postgresql/

![img_17.png](images/img_17.png)

-- автовакуум

SELECT name, setting, context, short_desc FROM pg_settings WHERE category like '%Autovacuum%';

-- первый показатель показывает приближение проблемы

-- второй показывает когда запустится автоматический автофриз

WITH max_age AS (
    SELECT 2000000000 as max_old_xid
        , setting AS autovacuum_freeze_max_age
        FROM pg_catalog.pg_settings
        WHERE name = 'autovacuum_freeze_max_age' )
, per_database_stats AS (
    SELECT datname
        , m.max_old_xid::int
        , m.autovacuum_freeze_max_age::int
        , age(d.datfrozenxid) AS oldest_current_xid
    FROM pg_catalog.pg_database d
    JOIN max_age m ON (true)
    WHERE d.datallowconn ) 

SELECT max(oldest_current_xid) AS oldest_current_xid
    , max(ROUND(100*(oldest_current_xid/max_old_xid::float))) AS percent_towards_wraparound
    , max(ROUND(100*(oldest_current_xid/autovacuum_freeze_max_age::float))) AS percent_towards_emergency_autovac
FROM per_database_stats;

-- маскимальный возраст для наших таблиц

SELECT c.oid::regclass as table_name,
       greatest(age(c.relfrozenxid),age(t.relfrozenxid)) as age
FROM pg_class c
LEFT JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relkind IN ('r', 'm');

-- Проверка мертвых записей:

SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'persons';

## Буферный кэш

![img_18.png](images/img_18.png)

![img_19.png](images/img_19.png)
![img_20.png](images/img_20.png)
![img_21.png](images/img_21.png)
![img_22.png](images/img_22.png)

![img_23.png](images/img_23.png)

-- проверим размер кеша

SELECT setting, unit FROM pg_settings WHERE name = 'shared_buffers';

-- уменьшим количество буферов для наблюдения

ALTER SYSTEM SET shared_buffers = 200;

-- рестартуем кластер после изменений

pg_ctlcluster 14 main restart

CREATE DATABASE buffer_temp;

\c buffer_temp

CREATE TABLE test(i int);

-- сгенерируем значения

INSERT INTO test SELECT s.id FROM generate_series(1,100) AS s(id);

SELECT * FROM test limit 10;

-- создадим расширение для просмотра кеша

CREATE EXTENSION pg_buffercache;

\dx+

-- создадим представление для просмотра содержимого буферного кеша

```
CREATE VIEW pg_buffercache_v AS
SELECT bufferid,
       (SELECT c.relname FROM pg_class c WHERE  pg_relation_filenode(c.oid) = b.relfilenode ) relname,
       CASE relforknumber
         WHEN 0 THEN 'main'
         WHEN 1 THEN 'fsm'
         WHEN 2 THEN 'vm'
       END relfork,
       relblocknumber,
       isdirty,
       usagecount
FROM   pg_buffercache b
WHERE  b.relDATABASE IN (    0, (SELECT oid FROM pg_DATABASE WHERE datname = current_database()) )
AND    b.usagecount is not null;
SELECT * FROM pg_buffercache_v WHERE relname='test';
SELECT * FROM test limit 10;
UPDATE test set i = 2 WHERE i = 1;
```

-- увидим грязную страницу


SELECT * FROM pg_buffercache_v WHERE relname='test';

-- даст пищу для размышлений над использованием кеша -- usagecount > 3

```
SELECT c.relname,
count(*) blocks,
round( 100.0 * 8192 * count(*) / pg_TABLE_size(c.oid) ) "% of rel",
round( 100.0 * 8192 * count(*) FILTER (WHERE b.usagecount > 3) / pg_TABLE_size(c.oid) ) "% hot"
FROM pg_buffercache b
JOIN pg_class c ON pg_relation_filenode(c.oid) = b.relfilenode
WHERE  b.relDATABASE IN (
         0, (SELECT oid FROM pg_DATABASE WHERE datname = current_database())
       )
AND    b.usagecount is not null
GROUP BY c.relname, c.oid
ORDER BY 2 DESC
LIMIT 10;
```

-- прогрев кеша

```
sudo pg_ctlcluster 14 main restart

\c buffer_temp

SELECT * FROM pg_buffercache_v WHERE relname='test_text';

CREATE EXTENSION pg_prewarm;

SELECT pg_prewarm('test_text');

SELECT * FROM pg_buffercache_v WHERE relname='test_text';
```

## Журналы

![img_24.png](images/img_24.png)

![img_25.png](images/img_25.png)

![img_26.png](images/img_26.png)

\c buffer_temp

SELECT * FROM pg_ls_waldir() LIMIT 10;

CREATE EXTENSION pageinspect;

BEGIN;

-- текущая позиция lsn
SELECT pg_current_wal_insert_lsn();

-- посмотрим какой у нас wal file

SELECT pg_walfile_name('0/1670928');

-- после UPDATE номер lsn изменился

SELECT lsn FROM page_header(get_raw_page('test_text',0));

commit;

UPDATE test_text set t = '1' WHERE t = 'строка 1';

SELECT pg_current_wal_insert_lsn();

SELECT lsn FROM page_header(get_raw_page('test_text',0));

SELECT '0/1672DB8'::pg_lsn - '0/1670928'::pg_lsn;

/usr/lib/postgresql/14/bin/pg_waldump -p /var/lib/postgresql/14/main/pg_wal -s 0/1670928 -e 0/1672DB8 000000010000000000000001

## Блокировки

![img_27.png](images/img_27.png)

![img_28.png](images/img_28.png)

![img_29.png](images/img_29.png)

![img_30.png](images/img_30.png)

![img_31.png](images/img_31.png)

![img_32.png](images/img_32.png)

![img_33.png](images/img_33.png)

![img_34.png](images/img_34.png)

![img_35.png](images/img_35.png)

![img_36.png](images/img_36.png)

![img_37.png](images/img_37.png)

![img_38.png](images/img_38.png)

![img_39.png](images/img_39.png)

-- какие есть сейчас блокировки

SELECT * FROM pg_locks \gx

-- кто блокирует тот или иной процесс

SELECT pg_backend_pid();

SELECT pg_blocking_pids(25895);

-- настройка записывать блокировки в лог после определенного времени

SHOW log_lock_waits;

SELECT pg_backend_pid();

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 33394;

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks;

-- таймаут лока

-- https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-LOCK-TIMEOUT

-- PG_STAT_ACTIVITY

SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(14352)) \gx

-- логи через утилиту просмотра окончания файла tail

tail /var/log/postgresql/postgresql-14-main.log

#### Полезное про блокировки:
![img_41.png](images/img_41.png)
![img_42.png](images/img_42.png)
![img_40.png](images/img_40.png)
Совместимость блокировок(x не совместим)
![img_43.png](images/img_43.png)
![img_44.png](images/img_44.png)
![img_45.png](images/img_45.png)
![img_46.png](images/img_46.png)
![img_47.png](images/img_47.png)
![img_48.png](images/img_48.png)
![img_49.png](images/img_49.png)
![img_50.png](images/img_50.png)
![img_51.png](images/img_51.png)
![img_52.png](images/img_52.png)
![img_53.png](images/img_53.png)
![img_54.png](images/img_54.png)
![img_55.png](images/img_55.png)
![img_56.png](images/img_56.png)
![img_57.png](images/img_57.png)

## Логическое устройство. Базы данных и схемы

```-- database

-- system catalog

SELECT oid, datname, datistemplate, datallowconn FROM pg_database;

-- size

SELECT pg_size_pretty(pg_database_size('postgres'));

-- schema

\dn

-- current schema

SELECT current_schema();

-- view table

\d pg_database

select * from pg_database;


-- seach path

SHOW search_path;


-- SET search_path to .. - в рамках сессии

-- параметр можно установить и на уровне отдельной базы данных:

-- ALTER DATABASE otus SET search_path = public, special;

-- в рамках кластера в файле postgresql.conf

\dt

-- интересное поведение и search_path

\d pg_database

CREATE TABLE pg_database(i int);


-- все равно видим pg_catalog.pg_database

\d pg_database


-- чтобы получить доступ к толко что созданной таблице используем указание схемы

\d public.pg_database

SELECT * FROM pg_database limit 1;


-- в 1 схеме или разных?

CREATE TABLE t1(i int);

CREATE SCHEMA postgres;

CREATE TABLE t2(i int);

CREATE TABLE t1(i int);

\dt

\dt+

\dt public.*

SET search_path TO public, "$user";

\dt

SET search_path TO public, "$user", pg_catalog;

\dt

create temp table t1(i int);

\dt

SET search_path TO public, "$user", pg_catalog, pg_temp;

\dt


-- можем переносить таблицу между схемами - при этом меняется только запись в pg_class, физически данные на месте

ALTER TABLE t2 SET SCHEMA public;


-- чтобы не было вопросов указываем схему прямо при создании

CREATE TABLE public.t10(i int);
-- relations

SELECT relkind, count(*) FROM pg_class GROUP BY relkind;
relkind | count
---------+-------
таблицы
r       |    69
представления
v       |   137
индексы
i       |   158
тост сегменты
t       |    40
последовательности
S       |     1
```

## Табличное пространство
![img_58.png](images/img_58.png)

![img_59.png](images/img_59.png)

![img_60.png](images/img_60.png)

```-- перейдем в домашний каталог пользователя postgres в ОС линукс

cd ~

pwd

mkdir temp_tblspce


psql


CREATE TABLESPACE ts location '/var/lib/postgresql/temp_tblspce';

\db

CREATE DATABASE app TABLESPACE ts;

\c app

\l+ -- посмотреть дефолтный tablespace

CREATE TABLE test (i int);

CREATE TABLE test2 (i int) TABLESPACE pg_default;

SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';

ALTER TABLE test set TABLESPACE pg_default;

SELECT oid, spcname FROM pg_tablespace; -- oid унимальный номер, по кторому можем найти файлы

SELECT oid, datname,dattablespace FROM pg_database;


-- всегда можем посмотреть, где лежит таблица

SELECT pg_relation_filepath('test2');


-- Узнать размер, занимаемый базой данных и объектами в ней, можно с помощью ряда функций.

SELECT pg_database_size('app');


-- Для упрощения восприятия можно вывести число в отформатированном виде:

SELECT pg_size_pretty(pg_database_size('app'));


-- Полный размер таблицы (вместе со всеми индексами):

SELECT pg_size_pretty(pg_total_relation_size('test2'));


-- И отдельно размер таблицы...

SELECT pg_size_pretty(pg_table_size('test2'));


-- ...и индексов:

SELECT pg_size_pretty(pg_indexes_size('test2'));


-- !!! с дефолтным неймспейсом не все так просто !!!

SELECT count(*) FROM pg_class WHERE reltablespace = 0;
```

## Физическое устройство

![img_61.png](images/img_61.png)

```
Как посмотреть конфигурационные файлы?

show hba_file;

show config_file;

show data_directory;



\! nano /etc/postgresql/14/main/postgresql.conf



cd /var/lib/postgresql/14/main

ls -l



psql

CREATE TABLE test5(i int);


-- всегда можем посмотреть, где лежит таблица

SELECT pg_relation_filepath('test5');


-- посмотрим на файловую систему

cd /var/lib/postgresql/14/main/base

ls -l



cd 13797

ls -l

ls -l | grep



INSERT INTO test5 VALUES (1);



ls -l | grep



INSERT INTO test5 VALUES (2);



ls -l | grep
```

## Мониторинг

![img_62.png](images/img_62.png)

![img_63.png](images/img_63.png)

![img_64.png](images/img_64.png)

![img_65.png](images/img_65.png)

![img_66.png](images/img_66.png)

- Prometheus exporter + Grafana

- https://okmeter.io/

- https://www.zabbix.com/ru - обычно мониторим железо, но можно и Постгрес

- Percona monitoring and management (PMM)

Что ещё:

pgHero
https://habr.com/ru/company/domclick/blog/546910/

pgWatch2
https://github.com/cybertec-postgresql/pgwatch2

```
sudo top

sudo htop

sudo atop

sudo iotop

sudo iostat -x


-- pgtop

-- 2 окно

sudo -u postgres psql

CREATE TABLE test(i int);

INSERT INTO test SELECT s.id FROM generate_series(1,1000000000) AS s(id);


-- 1

pg_top


-- текст запроса Q #

-- план E

-- блокировки L

-- Мониторинг

-- что подключено в текущую секунду

sudo -u postgres psql


-- во 2 запустим нагрузку

SELECT * FROM pg_stat_activity;


-- Получаем активные запросы длительностью более 5 секунд:

SELECT now() - query_start as "runtime", usename, datname, state, wait_event_type, wait_event, query

FROM pg_stat_activity

WHERE now() - query_start > '5 seconds'::interval and state='active'

ORDER BY runtime DESC;


-- State = ‘idle’ тоже вызывают подозрения. Но хуже всего - idle in transaction!

Далее убиваем:

●	для active

○	SELECT pg_cancel_backend(procpid);

●	для idle

○	SELECT pg_terminate_backend(procpid);


-- ТОП по загрузке CPU:

SELECT pid, xact_start, now() - xact_start AS duration

FROM pg_stat_activity

WHERE state LIKE '%transaction%'

ORDER BY duration DESC;

CREATE EXTENSION pg_stat_statements;

alter system set shared_preload_libraries = 'pg_stat_statements';

exit

sudo pg_ctlcluster 14 main restart

sudo -u postgres psql

show shared_preload_libraries;


-- ТОП по времени выполнения:

SELECT substring(query, 1, 50) AS short_query, round(total_exec_time::numeric, 2) AS total_time,

	calls, rows, round(total_exec_time::numeric / calls, 2) AS avg_time,

	round((100 * total_exec_time / sum(total_exec_time::numeric) OVER ())::numeric, 2) AS percentage_cpu

FROM pg_stat_statements

ORDER BY total_time DESC LIMIT 20;
```

## Профилирование запросов

- Для pl/pgsql есть расширение https://github.com/bigsql/plprofiler

- для остальных запросов используется журнал сообщений сервера

- log_min_duration_statements = 0 — время и текст всех запросов, при увеличении порога срабатывания - только “толстые” запросы

- log_line_prefix — идентифицирующая информация

- включается для всего сервера

- большой объем

- анализ внешними средствами, например pgBadger

- для отслеживания вложенных запросов можно использовать расширение auto_explain

**Инструменты профилирования запросов:**

pg_profile https://github.com/zubkov-andrei/pg_profile
Анализ исторической нагрузки PostgreSQL

PoWA (PostgreSQL Workload Analyzer)
https://powa.readthedocs.io/en/latest/

PoWA like
https://habr.com/ru/post/345370/

pgFouine
https://highload.today/profilirovanie-v-postgresql/

```
-- профилирование

alter system set log_min_duration_statement = 0;

select pg_reload_conf();

show log_min_duration_statement;

select * from test;

exit


-- из под пользователя student

sudo apt install -y pgbadger lynx

tail /var/log/postgresql/postgresql-14-main.log

pgbadger /var/log/postgresql/postgresql-14-main.log

lynx out.html
```

## Роли, группы и пользователи

![img_67.png](images/img_67.png)

![img_68.png](images/img_68.png)

![img_69.png](images/img_69.png)

![img_70.png](images/img_70.png)

![img_71.png](images/img_71.png)

```
-- Users

SELECT usename, usesuper FROM pg_catalog.pg_user;

\du

CREATE USER test;

\du

SELECT * FROM pg_catalog.pg_user;


-- попробуем законнектиться

\c - test


-- посмотрим имеет ли юзер права на вход

SELECT rolname, rolcanlogin FROM pg_catalog.pg_roles;


-- почему не подключились?

-- peer аутентификация через unix socket, а в unix нет пользователя тест

cat /etc/postgresql/14/main/pg_hba.conf


-- зададим пароль, без него не пустит

\password test


-- или

ALTER USER test PASSWORD 'test$123';

sudo -u postgres psql -U test -h 127.0.0.1 -d postgres -W

SELECT * FROM test;


-- попробуем нового юзера создать из под test

CREATE USER test2 WITH PASSWORD 'test$123' NOLOGIN;


-- попробуем нового юзера создать из под postgres

CREATE USER test2 WITH PASSWORD 'test$123' NOLOGIN;

ALTER USER test2 PASSWORD 'test$123';

sudo -u postgres psql -U test2 -h 127.0.0.1 -d postgres -W 
```

## Привилегии

![img_72.png](images/img_72.png)

![img_73.png](images/img_73.png)

![img_74.png](images/img_74.png)

![img_75.png](images/img_75.png)

GRANT SELECT ON mytable TO PUBLIC;
GRANT SELECT, UPDATE, INSERT ON mytable TO admin;
GRANT SELECT (col1), UPDATE (col1) ON mytable TO miriam_rw;

```
-- привилегии

CREATE TABLE testa(i int);

INSERT INTO testa values (333), (444);


-- выдадим права на эту таблицу юзеру test

GRANT SELECT, UPDATE, INSERT ON testa TO test;

\dp testa

exit

sudo -u postgres psql -U test -h 127.0.0.1 -d postgres -W

\dt


-- протеструем дефолтные права на обзую схему public

CREATE TABLE testb(i int);

INSERT INTO testb values (333), (444);

SELECT * from testb;


-- отзовем права

REVOKE SELECT, UPDATE, INSERT ON testb FROM test;

SELECT * from testb;
```

## Типы данных

![img_76.png](images/img_76.png)

![img_77.png](images/img_77.png)

![img_78.png](images/img_78.png)

```
CREATE or REPLACE PROCEDURE p(inout x text)
as $$
DECLARE
i INTEGER DEFAULT 1;
d DECIMAL(10,4) DEFAULT 0;
f FLOAT DEFAULT 0;
BEGIN
LOOP  
d := d + 0.001;
f := f + 0.001E0;
i := i + 1;
if i > 10000
then exit;
end if;
END LOOP;
x = d || ' ' || f;
end;
$$
language plpgsql
;


-- как думаете, на выходе получим одинаковые значения?

call p('');


-- создадим таблицу товаров с автоинкрементным полем, видом товара, автоматической датой добавления, ценой

CREATE TABLE goods(

id SERIAL PRIMARY KEY,

kind TEXT,

created_at TIMESTAMP DEFAULT now(),

price DECIMAL(12,2)

);


-- добавим пару записей

INSERT INTO goods(kind, price) VALUES ('bananas',100.50),('apples', 75.01);


-- посмотрим, что у нас добавилось

SELECT * FROM goods;


-- преобразование типов

-- явное

SELECT 123::text;

SELECT 123::double precision;

SELECT 123::numeric(17,2);


-- на примере нашей таблицы

SELECT kind || price::text FROM goods;


-- неявное (можем в самый неожиданный момент получить баги)

SELECT kind || price FROM goods;


--JSON

CREATE TABLE tickets(

ticket_no TEXT,

contact_data JSONB

);

INSERT INTO tickets(ticket_no, contact_data) VALUES ('0005432000987','{"phone": ["+70582584031"], "email": "volkova.alina_03101973@postgrespro.ru"}'),('0005432000988', '{"phone": ["+666662584031"], "email": "admin@aristov.tech"}');

SELECT * FROM tickets;


-- есть много операций над элементами json

-- https://postgrespro.ru/docs/postgresql/14/functions-json

-- Например

SELECT  t.ticket_no

, t.contact_data

, contact_data->>'phone' as phone

, contact_data->>'email' as email

, contact_data - 'phone' as cd_without_phone

FROM tickets as t

WHERE contact_data->>'phone' like '%+7%';


-- кроме того, что каждый элемент JSON может быть JSON, также можно использовать массивы элементов

UPDATE tickets set contact_data = '{"phone": ["+80582584031", "+70582584031"], "email": "volkova.alina_03101973@postgrespro.ru"}' WHERE ticket_no = '0005432000987';


-- если больше 1 элемента в JSON, то мы дублируем основную строку + элемент json

SELECT 	t.ticket_no

		, t.contact_data

		, cd.*

FROM tickets as t, jsonb_each_text(t.contact_data) as cd;
```
![img_108.png](images/img_108.png)

![img_109.png](images/img_109.png)

## Нормализация

![img_79.png](images/img_79.png)

![img_80.png](images/img_80.png)

![img_81.png](images/img_81.png)

![img_82.png](images/img_82.png)

https://habr.com/ru/articles/254773/

```
-- создадим таблицу c марками и моделями

CREATE TABLE marka(

marka TEXT,

model TEXT

);


-- добавим пару записей

INSERT INTO marka(marka, model) VALUES ('Apple','Iphone13'),('Apple','Iphone14'),('Xiaomi', 'Mi11'),('Xiaomi', 'Mi12');


-- посмотрим, что у нас добавилось

SELECT * FROM marka;


-- создадим таблицу c дилерами и телефонами

CREATE TABLE dealer(

name TEXT,

phones TEXT

);


-- добавим пару записей

INSERT INTO dealer(name, phones) VALUES ('Svyaznoi','8-800'),('MTC','100-500');


-- посмотрим, что у нас добавилось

SELECT * FROM dealer;


-- создадим таблицу c марками и дилерами

CREATE TABLE marka_dealer(

marka TEXT,

dealer TEXT

);


-- добавим пару записей

INSERT INTO marka_dealer(marka, dealer) VALUES ('Apple','Svyaznoi'),('Apple','MTC'),('Xiaomi','MTC');


-- посмотрим, что у нас добавилось

SELECT * FROM marka_dealer;


-- создадим таблицу c марками и скидками

CREATE TABLE discount(

marka TEXT,

discount TEXT

);


-- добавим пару записей

INSERT INTO discount(marka, discount) VALUES ('Apple','5'),('Xiaomi','10');


-- посмотрим, что у нас добавилось

SELECT * FROM discount;


-- соберем вместе

SELECT m.marka, m.model, d.name, d.phones, ds.discount

FROM marka m

JOIN marka_dealer md

	ON m.marka = md.marka

JOIN dealer d

	ON md.dealer = d.name

JOIN discount ds

	ON m.marka = ds.marka;
```

![img_83.png](images/img_83.png)

## Индексы

![img_84.png](images/img_84.png)

![img_85.png](images/img_85.png)

![img_86.png](images/img_86.png)

![img_87.png](images/img_87.png)

CREATE [ UNIQUE ] INDEX [ CONCURRENTLY ] [ [ IF NOT EXISTS ] имя ] ON имя_таблицы [ USING метод ]
(
{ имя_столбца | ( выражение ) }
[ COLLATE правило_сортировки ] [ класс_операторов ] [ ASC | DESC ] [ NULLS { FIRST | LAST } ] [, ...] )
[ INCLUDE ( имя столбца [,...]) ] для Btree и Gist
[ WITH ( параметр_хранения = значение [, ... ] ) ]
[ TABLESPACE табл_пространство ]
[ WHERE предикат ]

![img_107.png](images/img_107.png)

![img_106.png](images/img_106.png)

![img_88.png](images/img_88.png)

![img_89.png](images/img_89.png)

![img_90.png](images/img_90.png)

![img_91.png](images/img_91.png)

![img_92.png](images/img_92.png)

![img_93.png](images/img_93.png)

![img_94.png](images/img_94.png)

```
-- полнотекстовый поиск

SELECT 'a fat cat sat on a mat and ate a fat rat'::tsvector @@ 'cat & rat'::tsquery;


-- разница функции to_tsvector и лексемы ::tsvector

SELECT to_tsvector('fat cats ate fat rats') @@ to_tsquery('fat & rat');

?column?

----------

t


-- Заметьте, что соответствие не будет обнаружено, если запрос записан как

SELECT 'fat cats ate fat rats'::tsvector @@ to_tsquery('fat & rat');

?column?

----------

f


-- так как слово rats не будет нормализовано. Элементами tsvector являются лексемы,

-- предположительно уже нормализованные, так что rats считается не соответствующим rat.

-- можем указывать язык для лексем

SELECT to_tsvector('english', 'a fat  cat sat on a mat - it ate a fat rats');

SELECT to_tsvector('толстая кошка') @@ to_tsquery('толст');

SELECT to_tsvector('толстая кошка')

SELECT to_tsvector('russian','толстая кошка')

SELECT to_tsvector('russian','толстая кошка') @@ to_tsquery('толст');
-- по полнотекстовому поиску пример индекса и запросов

drop table if exists test_fts;

create table test_fts(t text);

INSERT INTO test_fts VALUES ('лимон'),('лимонад'),('налим'),('толстая кошка'),('толстые кошки'),('кот полосатый'),('худые коты');

CREATE INDEX idx ON test_fts USING GIN (to_tsvector('russian',t));

SELECT count(*)

FROM test_fts

WHERE to_tsvector('russian',t) @@ to_tsquery('лимон');

SELECT count(*)

FROM test_fts

WHERE to_tsvector('russian',t) @@ to_tsquery('лим');

SELECT count(*)

FROM test_fts

WHERE to_tsvector('russian',t) @@ to_tsquery('кот');

SELECT count(*)

FROM test_fts

WHERE to_tsvector('russian',t) @@ to_tsquery('кошка');

SELECT count(*)

FROM test_fts

WHERE to_tsvector('russian',t) @@ to_tsquery('кошк');
-- используем язык и при преобразовании шаблона для поиска

SELECT count(*)

FROM test_fts

WHERE to_tsvector('russian',t) @@ to_tsquery('russian','кошка');

EXPLAIN

SELECT count(*)

FROM test_fts

WHERE to_tsvector('russian',t) @@ to_tsquery('кошк');
```

## Статистика

![img_95.png](images/img_95.png)

![img_96.png](images/img_96.png)

![img_97.png](images/img_97.png)

![img_98.png](images/img_98.png)

![img_99.png](images/img_99.png)

![img_100.png](images/img_100.png)
Взаимная статистика полей(на сколько поля зависят друг от друга, оптимизация, влияет на точность определения кардинальности)

![img_101.png](images/img_101.png)
Еще один способ увеличения точности кардинальности (тут уже по комбинации значений)

![img_102.png](images/img_102.png)
Статистики тут

![img_103.png](images/img_103.png)

![img_104.png](images/img_104.png)
Еще один способ определения зависимости между колонками

![img_105.png](images/img_105.png)
Для коррелированных столбцов

```
# stats

SELECT * FROM pg_stat_database WHERE datname = 'postgres';

CREATE TABLE t1 (

    a   int,

    b   int

);

INSERT INTO t1 SELECT i/100, i/500

                 FROM generate_series(1,1000000) s(i);

ANALYZE t1;

EXPLAIN ANALYZE SELECT * FROM t1 WHERE (a = 1) AND (b = 0);


-- Создадим коррелированную статистику и посмотрим разницу:

CREATE STATISTICS s1 (dependencies) ON a, b FROM t1;

ANALYZE t1;

EXPLAIN ANALYZE SELECT * FROM t1 WHERE (a = 1) AND (b = 0);

SELECT * FROM pg_stat_activity;

SELECT * FROM pg_stat_user_tables WHERE relname = 't1' \gx

SELECT * FROM pg_stat_user_indexes WHERE relname = 't1' \gx
```

## Обслуживание индексов

![img_110.png](images/img_110.png)

![img_111.png](images/img_111.png)

```
-- посмотреть раздутость индекса

psql -c "drop database if exists index_test;"

psql -c "create database index_test;"

pgbench -i -s 10 index_test

psql index_test -c "CREATE EXTENSION pgstattuple;"

psql index_test -c "\d+ pgbench_accounts";

psql index_test -c "SELECT * FROM pgstatindex('pgbench_accounts_pkey');"

psql index_test -c "update pgbench_accounts set bid = bid + 1000000;"

psql index_test -c "update pgbench_accounts set aid = aid + 1000000;"

psql index_test

SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'pgbench_accounts';

VACUUM pgbench_accounts;

SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx

VACUUM FULL pgbench_accounts; -- на хайлоаде невозможно - требует исключительной блокировки всей таблицы

SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx

REINDEX INDEX CONCURRENTLY pgbench_accounts_pkey;
-- и снова обновим все записи и посмотрим что будет

update pgbench_accounts set bid = bid + 1000000;

update pgbench_accounts set aid = aid + 1000000;

SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx

VACUUM pgbench_accounts;

update pgbench_accounts set bid = bid + 1000000;

SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx

VACUUM pgbench_accounts;

update pgbench_accounts set aid = aid + 1000000;

SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx

VACUUM pgbench_accounts;

SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx

REINDEX INDEX CONCURRENTLY pgbench_accounts_pkey;

SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx
```

## Join

![img_112.png](images/img_112.png)

![img_113.png](images/img_113.png)

## Резервное копирование

![img_114.png](images/img_114.png)

![img_115.png](images/img_115.png)

![img_116.png](images/img_116.png)

![img_117.png](images/img_117.png)

![img_118.png](images/img_118.png)

![img_119.png](images/img_119.png)

```
sudo mkdir /home/1 && sudo chmod 777 /home/1


sudo -u postgres psql


-- создадим табличку для тестов

CREATE DATABASE backup;

\c backup

SELECT current_database();

CREATE TABLE test (i int, col text);

INSERT INTO test VALUES (1, '123');

INSERT INTO test VALUES (2, '456');

INSERT INTO test VALUES (3, '789');


--скопируем данные

COPY test TO '/home/1/test.csv' CSV HEADER;

cat /home/1/test.csv


-- восстановим данные в другую табличку

-- ошибка!!!

COPY test2 FROM '/home/1/test.csv' CSV HEADER;


-- таблица должна быть создана заранее

CREATE TABLE test2 (i int, col text);

COPY test2 FROM '/home/1/test.csv' CSV HEADER;

SELECT * FROM test2;


-- архивируем БД через консольную утилиту

sudo -u postgres pg_dump -d backup --CREATE


-- опции для команды pg_dump должны быть в нижнем регистре

sudo -u postgres pg_dump -d backup --create > /home/1/1.sql


-- посмотрим содержимое через cat



-- восстановим БД

sudo -u postgres psql < /home/1/1.sql

sudo -u postgres psql

\dt

DROP DATABASE backup;

CREATE DATABASE backup;

-- echo "drop DATABASE backup;" | sudo -u postgres psql

-- echo "CREATE DATABASE backup;" | sudo -u postgres psql

sudo -u postgres psql < /home/1/1.sql

sudo -u postgres psql

\dt
```

![img_120.png](images/img_120.png)

![img_121.png](images/img_121.png)

![img_122.png](images/img_122.png)

![img_123.png](images/img_123.png)

```
sudo mkdir /home/1 && sudo chmod 777 /home/1


sudo -u postgres psql


-- создадим табличку для тестов

CREATE DATABASE backup;

\c backup

SELECT current_database();

CREATE TABLE test (i int, col text);

INSERT INTO test VALUES (1, '123');

INSERT INTO test VALUES (2, '456');

INSERT INTO test VALUES (3, '789');


-- физическая репликация

-- посмотрим параметры

SELECT name, setting FROM pg_settings WHERE name IN ('wal_level','max_wal_senders');

SELECT * FROM pg_replication_slots;


-- Необходимо настроить файервол в файле pg_hba.conf для получения доступа извне localhost

SELECT type, database, user_name, address, auth_method

FROM pg_hba_file_rules() WHERE DATABASE = '{replication}';


-- Создадим 2 кластер

sudo -u postgres pg_createcluster 14 main2


-- Удалим оттуда файлы

sudo -u postgres rm -rf /var/lib/postgresql/14/main2


-- Сделаем бэкап нашей БД

sudo -u postgres pg_basebackup -p 5432 -D /var/lib/postgresql/14/main2


-- Стартуем кластер

sudo -u postgres pg_ctlcluster 14 main2 start


-- Смотрим как стартовал

pg_lsclusters

sudo -u postgres psql --port 5433
```

## Физическая репликация

![img_124.png](images/img_124.png)

![img_125.png](images/img_125.png)

![img_126.png](images/img_126.png)

![img_127.png](images/img_127.png)

![img_128.png](images/img_128.png)

![img_129.png](images/img_129.png)

```
-- зайдем в консоль под пользователем postgres

sudo su postgres


-- Посмотрим текущее состояние

pg_lsclusters


-- если остался 2 кластер остановим и удалим его

pg_ctlcluster 14 main2 stop

pg_dropcluster 14 main2

pg_lsclusters


-- Создадим 2 кластер

pg_createcluster -d /var/lib/postgresql/14/main2 14 main2


-- Удалим оттуда файлы

rm -rf /var/lib/postgresql/14/main2


-- Сделаем бэкап нашей БД. Ключ -R создаст заготовку управляющего файла recovery.conf

pg_basebackup -p 5432 -R -D /var/lib/postgresql/14/main2


-- Стартуем кластер

pg_ctlcluster 14 main2 start


-- Смотрим как стартовал

pg_lsclusters


-- на 1 ноде создадим таблицу и добавим 2 записи

psql -c "CREATE TABLE testrepl(i int);"

psql -c "INSERT INTO testrepl VALUES (1),(2);"


-- на 2 ноде прочитаем данные из этой таблицы

psql -p 5433 -c "SELECT * FROM testrepl;"
```

## Логическая репликация

![img_130.png](images/img_130.png)

![img_131.png](images/img_131.png)

![img_132.png](images/img_132.png)

![img_133.png](images/img_133.png)

```
-- Логическая репликация

pg_ctlcluster 14 main2 promote


-- включим логический уровень журналирования на 1 сервере

psql

ALTER SYSTEM SET wal_level = logical;

\q


-- Рестартуем кластер

pg_ctlcluster 14 main restart


-- На первом сервере создаем публикацию:

psql

CREATE TABLE testlogical(i int);

CREATE PUBLICATION test_pub FOR TABLE testlogical;


-- посмотрим публикации

\dRp+


-- создадим подписку на втором экземпляре

-- DDL команды не реплицируются!!!!

psql -p 5433

CREATE TABLE testlogical(i int);


--создадим подписку к БД по Порту с Юзером и Паролем и Копированием данных=false

CREATE SUBSCRIPTION test_sub

CONNECTION 'host=localhost port=5432 user=postgres password=123 dbname=postgres'

PUBLICATION test_pub WITH (copy_data = false);


-- посмотрим подписки

\dRs

SELECT * FROM pg_stat_subscription \gx


-- добавим данные на 1 кластере

INSERT INTO testlogical VALUES (1);


-- посмотрим, что появилось на 2

SELECT * FROM testlogical;
```

## Резервное копирование BEST PRACTICE

![img_134.png](images/img_134.png)

![img_135.png](images/img_135.png)

![img_136.png](images/img_136.png)

![img_137.png](images/img_137.png)

![img_138.png](images/img_138.png)

![img_139.png](images/img_139.png)

![img_140.png](images/img_140.png)

![img_141.png](images/img_141.png)

![img_142.png](images/img_142.png)

![img_143.png](images/img_143.png)

## Планы запросов

![img_171.png](images/img_171.png)

#### доп информация

![img_144.png](images/img_144.png)

![img_145.png](images/img_145.png)

![img_146.png](images/img_146.png)

![img_147.png](images/img_147.png)

![img_148.png](images/img_148.png)

## Секционирование

![img_172.png](images/img_172.png)

```
DROP TABLE IF EXISTS sales;

CREATE TABLE sales(name TEXT, salesdate timestamp)

PARTITION BY RANGE (salesdate);


CREATE TABLE sales_y2006_y2022 PARTITION OF sales

    FOR VALUES FROM ('2000-01-01') TO ('2022-06-01');


CREATE TABLE sales_y2022_m06 PARTITION OF sales

    FOR VALUES FROM ('2022-06-01') TO ('2022-07-01');


CREATE TABLE sales_y2022_m07 PARTITION OF sales

    FOR VALUES FROM ('2022-07-01') TO ('2022-08-01');


CREATE TABLE sales_y2022_m08 PARTITION OF sales

    FOR VALUES FROM ('2022-08-01') TO ('2022-09-01');


CREATE TABLE sales_y2022_y_2030 PARTITION OF sales

    FOR VALUES FROM ('2022-09-01') TO ('2030-12-12');


INSERT INTO sales(name, salesdate) VALUES

('banana','20220801'),

('banana','20220701'),

('grape','20220615'),

('berry','20220805'),

('banana','20220910');


EXPLAIN SELECT * FROM sales WHERE salesdate = '2022-08-01';

    QUERY PLAN                                 
----------------------------------------------------------------------------

Seq Scan on sales_y2022_m08 sales (cost=0.00..25.00 rows=6 width=40)

Filter: (salesdate = '2022-08-01 00:00:00'::timestamp without time zone)
```

![img_149.png](images/img_149.png)

![img_150.png](images/img_150.png)

![img_151.png](images/img_151.png)

![img_152.png](images/img_152.png)

![img_153.png](images/img_153.png)

![img_154.png](images/img_154.png)

![img_155.png](images/img_155.png)
Прикрепление партиции(таблицы) к новой партициионированной таблице(констрейнт ускоряет прикрепление, не нужно делать лишних проверок на уровне базы)

![img_156.png](images/img_156.png)

![img_157.png](images/img_157.png)
![img_158.png](images/img_158.png)

![img_159.png](images/img_159.png)
![img_160.png](images/img_160.png)
![img_161.png](images/img_161.png)

![img_162.png](images/img_162.png)
![img_163.png](images/img_163.png)
![img_164.png](images/img_164.png)

![img_165.png](images/img_165.png)

![img_166.png](images/img_166.png)
![img_167.png](images/img_167.png)

## Оптимизации
![img_168.png](images/img_168.png)

![img_169.png](images/img_169.png)

## dblink
![img_170.png](images/img_170.png)

## CTE

![img_173.png](images/img_173.png)


## Миграция данных

![img_174.png](images/img_174.png)
