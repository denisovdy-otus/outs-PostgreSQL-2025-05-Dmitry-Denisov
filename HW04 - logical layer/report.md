# Домашнее задание по теме: Физический уровень PostgreSQL

## Цель:

1. Cоздание новой базы данных, схемы и таблицы
2. Cоздание роли для чтения данных из созданной схемы созданной базы данных
3. Cоздание роли для чтения и записи из созданной схемы созданной базы данных

## Шаги выполнеия задания

1. Создаем новый кластер PostgresSQL 14
```
docker run --name postgres_container -p 5432:5432 -e POSTGRES_PASSWORD=postgres -d postgres:14
```
2. Заходим в созданный кластер под пользователем postgres
```
psql -h 127.0.0.1 -p 5432 -U postgres
```
3. Cоздаем новую базу данных testdb
```
CREATE DATABASE testdb;
```
4. Заходим в созданную базу данных под текущим пользователем postgres
```
\c testdb;
```
5. Создаём новую схему testnm
```
CREATE SCHEMA testnm;
```
6. Создаём новую таблицу t1 с одной колонкой c1 типа INTEGER
```
CREATE TABLE t1 (c1 INTEGER);
```
7. Вставляем строку со значением c1=1
```
INSERT INTO t1 (c1) VALUES (1);
```
8. Создаем новую роль readonly
```
CREATE ROLE readonly;
```
9. Выдаем для новой роли право на подключение к базе данных testdb
```
GRANT CONNECT ON DATABASE testdb TO readonly;
```
10. Выдаем для новой роли право на использование схемы testnm
```
GRANT USAGE ON SCHEMA testnm TO readonly;
```
11. Выдаем для новой роли право на SELECT для всех таблиц схемы testnm
```
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
```
12. Создаем пользователя testread с паролем test123
```
CREATE USER testread PASSWORD 'test123';
```
13. Выдаем роль readonly пользователю testread
```
GRANT readonly TO testread;
```
14. Заходим под пользователем testread в базу данных testdb
```
\c testdb testread;
```
15. Выполняем SELECT * FROM t1;
```
SELECT * FROM t1;

ERROR:  permission denied for TABLE t1
```
16. Запрос выполнить не получилось.
Таблица t1 создана в схеме public, так как мы явно не указали схему и search_path указывает на public.
У роли readonly привилегии SELECT для схемы public нет, лишь для схемы testnm. 
Поэтому у пользователя testread нет возможности выполнить SELECT запрос на t1.

17. Возвращаемся в базу данных testdb под пользователем postgres
```
\c testdb postgres;
```
18. Удаляем таблицу t1
```
DROP TABLE t1;
```
19. Создаем ее заново, но уже с явным указанием имени схемы testnm
```
CREATE TABLE testnm.t1 (c1 INTEGER);
```
20. Вставляем строку со значением c1=1
```
INSERT INTO testnm.t1 (c1) VALUES (1);
```
21. Заходим под пользователем testread в базу данных testdb
```
\c testdb testread;
```
22. Выполняем SELECT * FROM testnm.t1;
```
SELECT * FROM testnm.t1;

ERROR:  permission denied for TABLE t1
```
23. Запрос выполнить не получилось.
У пользователя testread нет ни одного разрешения для таблицы testnm.t1, 
так как на момент выдачи привилегий она отсутствовала.
Для того, чтобы задать привилегии, которые будут применяться к объетам созданным в будущем,
необходимо воспользоваться командой ALTER DEFAULT PRIVILEGIES.
```
\c testdb postgres;

ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;

\c testdb testread;
```
24. Выполняем SELECT * FROM testnm.t1;
```
SELECT * FROM testnm.t1;

ERROR:  permission denied for TABLE t1
```
25. Запрос выполнить не получилось.
Команда ALTER DEFAULT PRIVILEGES позволяет определять привилегии для обьектов, что будут созданы в будущем.
Тогда как testnm.t1 была создана до применения ALTER DEFAULT PRIVILEGES, то привилегий у роли readonly 
и пользователя testread у нее нет.
Для того чтобы исправить ситуацию окончательно требуется выдать привилегии на уже созданные объекты, в нашем случае на 
testnm.t1, либо пересоздать ее и привилегии, объявленные с помощью ALTER DEFAULT PRIVILEGES, 
будут добавлены по умолчанию.
```
\c testdb postgres;

GRANT SELECT ON testnm.t1 TO readonly;

\c testdb testread;
```
26. Выполняем SELECT * FROM testnm.t1;
```
SELECT * FROM testnm.t1;

 c1 
----
  1
(1 row)
```
27. Выполняем CREATE TABLE t2(c1 INTEGER); INSERT INTO t2 VALUES (2);
```
CREATE TABLE t2(c1 INTEGER); INSERT INTO t2 VALUES (2);

CREATE TABLE
INSERT 0 1
```
Создание таблицы и добавление в нее записи прошло успешно. 
Так как мы не указали схему, то таблицы была создана в схеме public. 
Так как роль public включает себя все роли (соответственно, и роль readonly) и имеет привилегии на создание таблиц
и внесение в них данных, то и пользователь testread имеет привилегии на создание и добавление данных в таблицы схемы 
public.

28. Чтобы убрать права на создание таблиц в схеме public требуется отнять привилегию CREATE у роли public, 
отнять привилегию у роли readonly недостаточно, так как пользователь testread получит привилегию через роль public.
```
\c testdb postgres;

REVOKE CREATE on SCHEMA public FROM public; 
 
\c testdb testread;

CREATE TABLE t3(c1 INTEGER); 

ERROR:  permission denied for SCHEMA public
LINE 1: CREATE TABLE t3(c1 INTEGER);
```
29. Уберем привилегии на добавление записей в таблицы схемы public. 

НЕПОНЯТНО!!! почему для INSERT достаточно отнять права у readonly, а для того, чтобы отнять привилегию CREATE, 
требуется ее отнимать у public???
```
\c testdb postgres;

REVOKE INSERT ON ALL TABLES IN SCHEMA public FROM readonly;
```
30. Проверим отстутствие привилегии, для нового пользователя с ролью readonly:
```
CREATE USER testread_new PASSWORD 'test123';
GRANT readonly TO testread_new;

\c testdb testread_new;

INSERT INTO t2 VALUES (2);

ERROR:  permission denied for TABLE t2
```
30. Проверим отстутствие привилегии, для пользователя создавшего таблицу t2:
```
\c testdb testread;

INSERT INTO t2 VALUES (2);

INSERT 0 1
```
Важно отметить, что пользователь testread владелец t2 таблицы и отнять привилегии на нее возможно лишь сменив владельца.
