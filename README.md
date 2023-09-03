# ДЗ № 3
##### Описание/Пошаговая инструкция выполнения домашнего задания:

- создать виртуальную машину c Ubuntu 20.04/22.04 LTS

```
Distributor ID: Ubuntu`
Description:    Ubuntu 22.04.3 LTS
Release:        22.04
Codename:       jammy
```

- поставьте на нее PostgreSQL 15 через sudo apt
```
-- 15 версия
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15


```
- проверьте что кластер запущен через sudo -u postgres pg_lsclusters
```
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

- зайдите из под пользователя postgres в psql и сделайте произвольную 
таблицу с произвольным содержимым
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
\q
```
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q

```
- остановите postgres например через
  sudo pg_ctlcluster 15 main stop
```
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
- создайте новый диск к ВМ размером 10GB <br> смотрим что сесть
```
sergt@ubuntu1:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
fd0                         2:0    1    4K  0 disk
sda                         8:0    0   10G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  8.2G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0  8.2G  0 lvm  /
sr0                        11:0    1 1024M  0 rom
sergt@ubuntu1:~$ one of the only occasional exceptions.
```
добавил диск 10гиг система дала ему имя **sdb**
```
sergt@ubuntu1:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
fd0                         2:0    1    4K  0 disk
sda                         8:0    0   10G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0  8.2G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0  8.2G  0 lvm  /
sdb                         8:16   0   10G  0 disk
sr0                        11:0    1 1024M  0 rom
```
- добавьте свеже-созданный диск 
к виртуальной машине - надо зайти в режим ее редактирования и 
дальше выбрать пункт attach existing disk
проинициализируйте диск согласно инструкции и подмонтировать 
файловую систему, только не забывайте менять имя диска на актуальное, 
в вашем случае это скорее всего будет /dev/sdb <br>
доп инфа по разметке дисков [читаем как это сделать](https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux) <br>
Размечаем форматируем назначаем метку. 


```
sudo parted /dev/sdb mklabel gpt
sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L partPostgreSQL /dev/sdb1
sergt@ubuntu1:~$ lsblk -fs
NAME                  FSTYPE      FSVER    LABEL          UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
fd0
sda1
└─sda
sda2                  ext4        1.0                     551c8f85-0645-4cdc-bd12-0753cab92d00      1.3G    15% /boot
└─sda
sdb1                  ext4        1.0      partPostgreSQL 413d6144-baeb-4de0-b91d-be7b847169f1      9.2G     0% /mnt/data
└─sdb
sr0
ubuntu--vg-ubuntu--lv ext4        1.0                     ccdc196a-1fd8-4533-bc5c-407b5ba4b8d4      4.3G    41% /
└─sda3                LVM2_member LVM2 001                HIPW01-f2aB-maiM-jyuX-uZv5-NkPc-mBGbZ0
  └─sda
```
Раздел на диске появился и смонтировать как в /etc/fstab
- перезагрузите инстанс и убедитесь, что диск остается примонтированным
Да остался 
(если не так смотрим в сторону **fstab**)
```
dev/disk/by-uuid/551c8f85-0645-4cdc-bd12-0753cab92d00 /boot ext4 defaults 0 1
/dev/disk/by-uuid/413d6144-baeb-4de0-b91d-be7b847169f1 /mnt/data ext4 defaults 0    
```
- сделайте пользователя postgres владельцем 
/mnt/data - chown -R postgres:postgres /mnt/data/
```
sergt@ubuntu1:~$ ll /mnt/data/
total 28
drwxr-xr-x 3 postgres postgres  4096 Aug 31 07:07 ./
drwx------ 2 postgres postgres 16384 Aug 31 05:41 lost+found/
-rw-r--r-- 1 postgres postgres     8 Aug 31 07:07 test_file
```
- перенесите содержимое /var/lib/postgres/15 
в /mnt/data - mv /var/lib/postgresql/15/mnt/data
- попытайтесь запустить кластер -
  sudo pg_ctlcluster 15 main start
- напишите получилось или нет и почему <br>
  **Не получилось. Выдает ошибку т.к. не находит свои данные.**
```
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
- задание: найти конфигурационный параметр в файлах раположенных в 
/etc/postgresql/15/main который надо поменять и поменяйте его
- напишите что и почему поменяли
```
файл /etc/postgresql/15/main/postgresql.conf
строка 
data_directory = '/mnt/data/15/main'          # use data in another directory
```
**этот параметр отвечает за расположение данных экземпляра кластера**
- попытайтесь запустить кластер - 
sudo -u postgres pg_ctlcluster 15 main start
- напишите получилось или нет и почему

**кластер запустился т.к. на этот раз по указанному пути лежат данные кластера**
- зайдите через через psql и проверьте содержимое ранее созданной таблицы
```
postgres=# \d
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# select * from test;
 c1
----
 1
(1 row)
```
**Всё на месте и таблица и данные**

- задание со звездочкой *: не удаляя существующий инстанс 
ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres,
перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко 
второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, 
расскажите как вы это сделали и что в итоге получилось.


После подключения диска к новому экземпляру виртуальной машины дис определился в ОС так же как и в
первой машине с тем же UUID
```

NAME                  FSTYPE      FSVER    LABEL          UUID                                   FSAVAIL FSUSE% MOUN
fd0
sda1
└─sda
sda2                  ext4        1.0                     551c8f85-0645-4cdc-bd12-0753cab92d00      1.3G    15% /boo
└─sda
sdb1                  ext4        1.0      partPostgreSQL 413d6144-baeb-4de0-b91d-be7b847169f1
└─sdb
sr0
ubuntu--vg-ubuntu--lv ext4        1.0                     ccdc196a-1fd8-4533-bc5c-407b5ba4b8d4      4.4G    40% /
└─sda3                LVM2_member LVM2 001                HIPW01-f2aB-maiM-jyuX-uZv5-NkPc-mBGbZ0
  └─sda
```
Логично будет сделать точно такие же настройки файловой системы и конфигурационного фала PostgreSQL
правим /etc/fstab и /etc/postgresql/15/main/postgresql.conf

Да всё получилось данные на месте .