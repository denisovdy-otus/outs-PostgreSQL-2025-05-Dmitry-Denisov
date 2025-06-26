# Домашнее задание по теме: Установка PostgreSQL

## Цель:

1. Установить PostgreSQL в Docker контейнере
2. Настроить контейнер для внешнего подключения

## Установка Docker

1. Создаем ВМ с Ubuntu 22.04 в VirtualBox
2. Устанавливаем Docker Engine
> sudo apt-get update <br/>
> sudo apt-get install ca-certificates curl <br/>
> sudo install -m 0755 -d /etc/apt/keyrings <br/>
> sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc <br/>
> sudo chmod a+r /etc/apt/keyrings/docker.asc <br/>
>  <br/>
> echo \  <br/>
> "deb \[arch=\$\(dpkg --print-architecture\) signed-by=/etc/apt/keyrings/docker.asc\] https\://download.docker.com/linux/ubuntu \\ <br/>
> \$\(\. /etc/os-release \&\& echo "\$\{UBUNTU_CODENAME\:-\$VERSION_CODENAME\}"\) stable" \| \\  <br/>
> sudo tee /etc/apt/sources.list.d/docker.list \> /dev/null <br/>
>  <br/>
> sudo apt-get update <br/>
> sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin <br/>

3. Создаем каталог /var/lib/postgres
4. Развертываем контейнер с сервером PostgreSQL 15
> sudo docker network create postgres-network <br/>
> sudo docker run --name postgres-server --network postgres-network -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15 <br/>
5. Развертываем контейнер с клиентом postgres и подключаемся к нему
> sudo docker run -it --rm --name postgres-client --network postgres-network  postgres:15 psql -h postgres-server -U postgres <br/>
6. Создаем таблицу с парой строк
> CREATE SCHEMA otus; <br/>
> SET search_path TO otus; <br/>
> CREATE TABLE dummy_table (id INTEGER, value VARCHAR(40), PRIMARY KEY(id)); <br/>
> INSERT INTO dummy_table(id, value) VALUES (1, 'value 1'), (2, 'value 2'); <br/>
> SELECT * FROM dummy_table; <br/>
> <br/>
>  id |  value   <br/>
> ----+--------- <br/>
>   1 | value 1 <br/>
>   2 | value 2 <br/>
> (2 rows) <br/>
7. Подключаемся к контейнеру с сервером извне виртуальной машины на которой развернут docker
Предварительно настраиваем port forwarding в настройках виртуальной машины. Отображаем TCP порт 5432 на 5432 порт хоста.
> psql -p 5432 -U postgres -h 127.0.0.1 -d postgres -W <br/>
> SELECT * FROM otus.dummy_table; <br/>
> <br/>
>  id |  value   <br/>
> ----+--------- <br/>
>   1 | value 1 <br/>
>   2 | value 2 <br/>
> (2 rows) <br/>
8. Удаляем контейнер с сервером
> sudo docker stop postgres-server <br/>
> sudo docker rm postgres-server <br/>
9. Развертываем контейнер с сервером заново
> sudo docker run --name postgres-server --network postgres-network -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15 <br/>
10. Подключаемся снова из контейнера с клиентом к контейнеру с сервером
> sudo docker run -it --rm --name postgres-client --network postgres-network  postgres:15 psql -h postgres-server -U postgres <br/>
11. Проверяем, что данные остались на месте
> SELECT * FROM otus.dummy_table; <br/>
> <br/>
>  id |  value   <br/>
> ----+--------- <br/>
>   1 | value 1 <br/>
>   2 | value 2 <br/>
> (2 rows) <br/>
