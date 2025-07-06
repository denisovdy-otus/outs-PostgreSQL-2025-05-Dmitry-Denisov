# Домашнее задание по теме: Физический уровень PostgreSQL

## Цель:

1. Cоздавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
2. Переносить содержимое базы данных PostgreSQL на дополнительный диск
3. Переносить содержимое БД PostgreSQL между виртуальными машинами

## Шаги выполнеия задания

1. Создаем VM с Ubuntu 22.04 в VirtualBox
2. Устанавливаем на VM PostgreSQL 15 через sudo apt
```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

sudo apt install postgresql-15
```
3. Проверяем, что кластер запущен 
```
sudo -u postgres pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
4. Заходим из под пользователя postgres в psql и создаем произвольную таблицу с произвольным содержимым
```
sudo -u postgres psql
```
Создаем произвольную таблицу с произвольным содержимым
```
create table test(c1 text);
insert into test values('1');

select * from test;

 c1 
----
 1
(1 row)

\q
```
6. Останавливаем postgres 
```
sudo -u postgres pg_ctlcluster 15 main stop

sudo -u postgres pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
7. Создаем и добавляем новый диск к VM размером 10GB
    1. Останавливаем VM c Postgres
    2. В настройках VM в разделе "Носители"->"Устройства"->"Контроллер: SATA" создаем и добавляем жесткий диск размером 10GB
8. Инициализируем диск
```
sudo fdisk /dev/sdb
> n - add a new partition
> p - primary partition type (0 primary, 0 extended, 4 free)
> default - Partition number (1-4, default 1)
> default - First sector (2048-20971519, default 2048)
> default - Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519)
> w - write table to disk and exit
```
Проверяем созданный диск и партишен
```
lsblk /dev/sdb

NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0  10G  0 disk 
└─sdb1   8:17   0  10G  0 part 
```
Форматируем новый партишен
```
sudo mkfs.ext4 /dev/sdb1
```
Монтируем партишен
```
sudo mkdir /mnt/data
sudo mount -t ext4 -o rw /dev/sdb1 /mnt/data
echo '/dev/sdb1 /mnt/data ext4 rw 0 0' | sudo tee -a /etc/fstab
```
9. Перезагружаем инстанс
```
sudo reboot
```
Убеждаемся, что диск остается примонтированным
```
lsblk /dev/sdb

NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0  10G  0 disk 
└─sdb1   8:17   0  10G  0 part /mnt/data
```
10. Делаем пользователя postgres владельцем /mnt/data
```
sudo chown -R postgres:postgres /mnt/data/
```
11. Переносим содержимое /var/lib/postgres/15 в /mnt/data
```
sudo mv /var/lib/postgresql/15 /mnt/data
```
12. Стартуем кластер
```
sudo -u postgres pg_ctlcluster 15 main start

Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
Кластер не запустился.

Причина этого - неактуальная настройка "data_directory" в postgresql.conf кластера main.

В файле настроек /etc/postgresql/15/main/postgresql.conf параметр data_directory = '/var/lib/postgresql/15/main'.

На предыдущем шаге содержимое /var/lib/postgres/15 было перенесено в /mnt/data, соответственно файлы для кластера main отсутствуют в /var/lib/postgresql/15/main и кластер не запускается.

13. Меняем параметр data_directory в файле настроек /etc/postgresql/15/main/postgresql.conf на '/mnt/data/15/main'
```
...
data_directory = '/mnt/data/15/main' 
...
```
14. Стартуем кластер
```
sudo -u postgres pg_ctlcluster 15 main start
```
Проверяем состояние кластера
```
sudo -u postgres pg_lsclusters

Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```
Кластер запущен и в статусе 'online', так как параметр "data_directory" теперь указывает на актуальное расположение данных кластера ('/mnt/data/15/main').

15. Проверяем содержимое ранее созданной таблицы
```
sudo -u postgres psql -c 'select * from test;'

 c1 
----
 1
(1 row)
```

## Задание со звездочкой

1. Создаем новый инстанс VM, выполняем шаги 1 и 2.
Проверяем состояние кластера
```
sudo -u postgres pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
2. Останавливаем postgres
```
sudo -u postgres pg_ctlcluster 15 main stop

sudo -u postgres pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
3. Удаляем файлы с данными /var/lib/postgresql
```
sudo rm -r /var/lib/postgresql
```
4. Останавливаем первую VM. 

5. В настройках первой VM в разделе "Носители"->"Устройства"->"Контроллер: SATA" удаляем внешний диск

6. Останавливаем вторую VM. 

7. В настройках второй VM в разделе "Носители"->"Устройства"->"Контроллер: SATA" добавляем внешний диск

8. Запускаем вторую VM. 

9. Монтируем внешний диск в /var/lib/postgresql
```
mkdir /var/lib/postgresql 

sudo mount -t ext4 -o rw /dev/sdb1 /var/lib/postgresql 

echo '/dev/sdb1 /var/lib/postgresql ext4 rw 0 0' | sudo tee -a /etc/fstab
```
10. Стартуем кластер
```
sudo -u postgres pg_ctlcluster 15 main start
```
11. Проверяем, что ранее созданная таблица доступна
```
sudo -u postgres psql -c 'select * from test;'

 c1 
----
 1
(1 row)
```