# Домашнее задание по теме: Установка PostgreSQL

## Цель:

1. Установить PostgreSQL в Docker контейнере
2. Настроить контейнер для внешнего подключения

## Установка Docker

1. Создаем ВМ с Ubuntu 22.04 в VirtualBox
2. Устанавливаем Docker Engine
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \ 
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \ 
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
3. Создаем каталог /var/lib/postgres
4. Развертываем контейнер с сервером PostgreSQL 15
```
sudo docker network create postgres-network
sudo docker run --name postgres-server --network postgres-network -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
5. Развертываем контейнер с клиентом postgres и подключаемся к нему
```
sudo docker run -it --rm --name postgres-client --network postgres-network  postgres:15 psql -h postgres-server -U postgres
```
6. Создаем таблицу с парой строк
```
CREATE SCHEMA otus;
SET search_path TO otus;
CREATE TABLE dummy_table (id INTEGER, value VARCHAR(40), PRIMARY KEY(id));
INSERT INTO dummy_table(id, value) VALUES (1, 'value 1'), (2, 'value 2');
SELECT * FROM dummy_table;

 id |  value  
----+---------
  1 | value 1
  2 | value 2
(2 rows)
```
7. Подключаемся к контейнеру с сервером извне виртуальной машины на которой развернут docker

Предварительно настраиваем port forwarding в настройках виртуальной машины. Отображаем TCP порт 5432 на 5432 порт хоста.
```
psql -p 5432 -U postgres -h 127.0.0.1 -d postgres -W
SELECT * FROM otus.dummy_table;

 id |  value  
----+---------
  1 | value 1
  2 | value 2
(2 rows)
```
8. Удаляем контейнер с сервером
```
sudo docker stop postgres-server
sudo docker rm postgres-server
```
9. Развертываем контейнер с сервером заново
```
sudo docker run --name postgres-server --network postgres-network -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
10. Подключаемся снова из контейнера с клиентом к контейнеру с сервером
```
sudo docker run -it --rm --name postgres-client --network postgres-network  postgres:15 psql -h postgres-server -U postgres
```
11. Проверяем, что данные остались на месте
```
SELECT * FROM otus.dummy_table;

 id |  value  
----+---------
  1 | value 1
  2 | value 2
(2 rows)
```
