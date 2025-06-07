# Домашнее задание по теме: SQL и реляционные СУБД. Введение в PostgreSQL

## Цель:
1. Научиться работать в Яндекс Облаке
2. Научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

## Настройка Postgres в Яндекс Облаке

1. Создаем аккаунт https://yandex.cloud
2. Cоздаем каталог postgrees-catalog (с параметром "Создать сеть по умолчанию")
3. В созданном каталоге в разделе права доступа нажимаем "Настроить доступ", выдаем пользователю права  "vpc.user"
   и роль "managed-postgresql.editor"
4. Создаем PostgreSQL кластер с помощью "Managed Service for PostgreSQL". Сохраняем название бд (**db1**),
   имя пользователя (**user1**), пароль
5. В разделе "Облачные сети" в "Группы безопасности" добавляем правило сети для доступа извне
> Диапазон портов — 6432 <br/>
> Протокол — TCP <br/>
> Источник — CIDR <br/>
> CIDR блоки — 0.0.0.0/0 <br/>
6. Редактируем хост кластера устанавливаем признак "Публичный доступ"
7. Получаем ssl сертификат
> mkdir -p ~/.postgresql <br/>
> wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" --output-document ~/.postgresql/root.crt <br/>
> chmod 0655 ~/.postgresql/root.crt <br/>
8. Находим FQDN для обращения к кластеру (rc1a-tf9ugf9mq80334fv)
9. Подключаемся с помощью клиента postgres к созданному кластеру
> psql "host=rc1a-tf9ugf9mq80334fv.mdb.yandexcloud.net <br/>
> port=6432  <br/>
> sslmode=verify-full <br/>
> dbname=db1 <br/>
> user=user1 <br/>
> target_session_attrs=read-write" <br/>

## Управление уровнями изоляций PostgreSQL

Для изучения управления уровнями изоляций в PostgreSQL было открыто 2 терминала (Session1 и Session2)
с выключенным autocommit (SET AUTOCOMMIT OFF)

1. Cоздаем новую таблицу, наполняем ее данными <br/>
> **Session 1:** <br/>
> create table persons(id serial, first_name text, second_name text); <br/>
> begin transaction; <br/>
> insert into persons(first_name, second_name) values('ivan', 'ivanov'); <br/>
> insert into persons(first_name, second_name) values('petr', 'petrov'); <br/>
> commit; <br/>

2. Проверяем текущий уровень изоляции
> **Session 1:** <br/>
> show transaction isolation level;<br/>
> <br/>
> transaction_isolation <br/>
> ----------------------- <br/>
> read committed <br/>
> (1 row) <br/>


3. Начинаем новую транзакцию в обоих сессиях с не меняя уровень изоляции
> **Session 1:** <br/>
> begin transaction;

> **Session 2:** <br/>
> begin transaction;

4. В первой сессии добавляем новую запись
> **Session 1:** <br/>
> insert into persons(first_name, second_name) values('sergey', 'sergeev'); <br/>
> select * from persons; <br/>
> <br/>
> id | first_name | second_name <br/>
> ----+------------+------------- <br/>
> 1 | ivan       | ivanov <br/>
> 2 | petr       | petrov <br/>
> 3 | sergey     | sergeev <br/>
> (3 rows) <br/>

5. Выбираем все записи из таблицы persons во второй сессии
> **Session 2:** <br/>
> select * from persons; <br/>
>  <br/>
> id | first_name | second_name <br/>
> ----+------------+------------- <br/>
> 1 | ivan       | ivanov <br/>
> 2 | petr       | petrov <br/>
> (2 rows) <br/>

6. Видите ли вы новую запись и если да то почему? <br/>

Новая запись отсутствует, так как транзакция в которой она была добавлена не закомичена.
Так как текущий уровень изоляции read committed, то аномалию, которая позволила бы увидеть новую незакоммиченную запись (dirty read), мы не наблюдаем.

7. Завершаем первую транзакцию
> **Session 1:** <br/>
> commit; <br/>

8. Выбираем все записи из таблицы persons во второй сессии

> **Session 2:** <br/>
> select * from persons; <br/>
> <br/>
> id | first_name | second_name<br/>
> ----+------------+-------------<br/>
> 1 | ivan       | ivanov<br/>
> 2 | petr       | petrov<br/>
> 3 | sergey     | sergeev<br/>
> (3 rows)<br/>

9. Видите ли вы новую запись и если да то почему?<br/>

Новая запись видна.
Так как транзакция, в которой эта запись была добавлена, закоммичена после начала текущей транзакции,
это означает, что в данный момент мы наблюдаем аномалию phantom read.
Текущий уровень изоляции (read committed) не исключает подобные аномалии.
Операция select при данном уровне изоляции вернет все записи, что были закомичены до ее начала.
Чтобы избежать подобных аномалий следует использовать более строгий уровень изоляции Serializable.
Стоит отметить, что postgres использует механизм MVCC, который исключает аномалию phantom read уже на уровне изоляции
repeatable read.

10. Завершаем транзакцию во второй сессии

> **Session 2:** <br/>
> commit; <br/>

11. Начинаем новые транзакции, но уже с уровнем изоляции repeatable read
> **Session 1:** <br/>
> begin transaction; <br/>
> set transaction isolation level repeatable read; <br/>
> show transaction isolation level; <br/>
> <br/>
> transaction_isolation <br/>
> ----------------------- <br/>
> repeatable read <br/>
> (1 row) <br/>


> **Session 2:** <br/>
> begin transaction; <br/>
> set transaction isolation level repeatable read; <br/>
> show transaction isolation level; <br/>
> <br/>
> transaction_isolation <br/>
> ----------------------- <br/>
> repeatable read <br/>
> (1 row) <br/>

12. В первой сессии добавляем новую запись
> **Session 1:** <br/>
> insert into persons(first_name, second_name) values('sveta', 'svetova'); <br/>
> select * from persons; <br/>
> <br/>
> id | first_name | second_name <br/>
> ----+------------+------------- <br/>
> 1 | ivan       | ivanov <br/>
> 2 | petr       | petrov <br/>
> 3 | sergey     | sergeev <br/>
> 4 | sveta      | svetova <br/>
> (4 rows) <br/>

13. Выбираем все записи из таблицы persons во второй сессии
> **Session 2:** <br/>
> select * from persons; <br/>
> <br/>
> id | first_name | second_name <br/>
> ----+------------+------------- <br/>
> 1 | ivan       | ivanov <br/>
> 2 | petr       | petrov <br/>
> 3 | sergey     | sergeev <br/>
> (3 rows) <br/>

14. Видите ли вы новую запись и если да то почему?

Новая запись не наблюдается.
Новая запись отсутствует, так как транзакция в которой она была добавлена не закомичена.
Уровень изоляции repeatable read исключает аномалию  dirty read.

15. Завершаем первую транзакцию
> **Session 1:** <br/>
> сommit; <br/>

16. Выбираем все записи из таблицы persons во второй сессии
> **Session 2:** <br/>
> select * from persons; <br/>
> <br/>
> id | first_name | second_name <br/>
> ----+------------+------------- <br/>
> 1 | ivan       | ivanov <br/>
> 2 | petr       | petrov <br/>
> 3 | sergey     | sergeev <br/>
> (3 rows) <br/>

17. видите ли вы новую запись и если да то почему?

Новая запись не наблюдается.
Как было указано выше, phantom read - аномалия, которая позволяет нам видеть в результате запроса записи,
что были добавлены другими транзакциями, завершившимися после начала рассматриваемой транзакции.
Чтобы исключить подобную аномалию предусмотрен уровень изоляции Serializable.
Но postgres использует механизм MVCC, который исключает аномалию phantom read уже на уровне изоляции repeatable read.
Операция select при этом уровне изоляции вернет все записи, что были закомичены до начала транзакции.
Поэтому, новая запись не наблюдается.

18. Завершаем вторую транзакцию
> **Session 2:** <br/>
> сommit; <br/>

19. Выбираем все записи из таблицы persons во второй сессии
> **Session 2:** <br/>
> select * from persons; <br/>
> <br/>
> id | first_name | second_name <br/>
> ----+------------+------------- <br/>
> 1 | ivan       | ivanov <br/>
> 2 | petr       | petrov <br/>
> 3 | sergey     | sergeev <br/>
> 4 | sveta      | svetova <br/>
> (4 rows) <br/>
>  <br/>

20. Видите ли вы новую запись и если да то почему?

Запись присутствует.
Новая транзакция была начата после завершения транзакции, что добавила новую запись.
Если у новой транзакции будет уровень изоляции read commited, то мы увидим новую запись, так как она была закоммичена
до начала операции select.
Если мы изменим уровень изоляции новой транзакции на repeatable read, то мы все равно увидим новую запись,
так как она была закоммичена до начала транзакции.