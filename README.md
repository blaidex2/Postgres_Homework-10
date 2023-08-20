## Домашнее задание №10

Развернуты 4 ВМ с Ubuntu 22.04: 2 ГБ ОЗУ, 1 ЦП, 15 ГБ. На каждой установлен PostrgeSQL 15.

1. На 1 ВМ создаем базу данных, таблицы «test» для записи и «test2» для запросов на чтение, установим параметр «wal_level = logical», создаем публикацию таблицы «test».
   
>alter user postgres password 'qwerty123';
>
>create database test_db;

--переключаемся в базу test_db

>create table test (id serial, str varchar(100));
>
>create table test2 (id serial, str varchar(100));
>
>insert into test (str) select md5(random()::text) from generate_series(1, 100);
>
>alter system set wal_level = logical;
>
--перезагружаем постгре

>create publication test_publication for table test;

 ![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/217f5ef3-2189-4498-95c1-ef1d90e6a07a)


3.	На 2 ВМ создаем таблицы «test2» для записи и «test» для запросов на чтение, установим параметр «wal_level = logical», создаем публикацию таблицы «test2».
   
>alter user postgres password 'qwerty123';
>
>create database test_db;
>
--переключаемся в базу test_db
>create table test (id serial, str varchar(100));
>
>create table test2 (id serial, str varchar(100));
>
>insert into test2 (str) select md5(random()::text) from generate_series(1, 100);
>
>alter system set wal_level = logical;
>
--перезагружаем постгре

>create publication test_publication2 for table test2;

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/9bb7ec9f-64a4-4a0e-8fd9-d4617ad1eabb)

 

4. На ВМ №1 и №2 необходимо отредактировать 2 файла:
   
>sudo nano /etc/postgresql/15/main/pg_hba.conf

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/a55ee305-9441-45b2-8cdf-2042008afda1)

 
>sudo nano /etc/postgresql/15/main/postgresql.conf

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/61c7f380-d552-4bda-9407-e79ef397ab1a)

 

5.	Подписываемся на публикацию таблицы «test» с ВМ №2.

>create subscription test_subscription connection 'host=10.100.110.138 port=5432 user=postgres password=qwerty123 dbname=test_db' publication test_publication with (copy_data=true);


![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/2522883c-51e0-4da7-9ef2-092b7a1befe6)

 
На ВМ №2 видны записи из таблицы «test» с ВМ №1.
Если на ВМ №2 добавить запись в таблицу «test», новая запись не отобразится.

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/24c97a21-9c49-411d-b95d-42638fd2238c)


6.	Подписываемся на публикацию таблицы «test2» с ВМ №1.

>create subscription test_subscription2 connection 'host=10.100.110.196 port=5432 user=postgres password=qwerty123 dbname=test_db' publication test_publication2 with (copy_data=true);

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/7a6157ab-9b75-40be-87e4-efbde8680f1c)


ВМ №3 использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).

>alter user postgres password 'qwerty123';
>
>create database test_db;
--переключаемся в базу test_db
>
>create table test (id serial, str varchar(100));
>
>create table test2 (id serial, str varchar(100));

>create subscription subscription1 connection 'host=10.100.110.138 port=5432 user=postgres password=qwerty123 dbname=test_db' publication test_publication with (copy_data=true);

>create subscription subscription2 connection 'host=10.100.110.196 port=5432 user=postgres password=qwerty123 dbname=test_db' publication test_publication2 with (copy_data=true);

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/cc0a1025-4d20-44a3-b8d4-848e36c604b3)
 

Задача под звездочкой: реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

На ВМ 3:
1. Отредактируем файл pg_hba.conf
   
>sudo nano  /etc/postgresql/15/main/pg_hba.conf
>

TYPE      |  DATABASE         |      USER             |      ADDRESS               |      METHOD    |

host      |  replication      |     postgres          |    10.100.110.162/32       |      md5       |



2. Отредактируем файл postgresql.conf
   
>sudo nano /etc/postgresql/15/main/postgresql.conf

listen_addresses = 'localhost,10.100.110.127'

wal_level = 'hot_standby'

archive_mode = on

archive_command = 'cd.'

max_wal_senders = 2

hot_standby = on


3. Перезагрузим ВМ №3
   
>sudo systemctl restart postgresql

На ВМ №4:
1. Остановим ВМ №4
   
>sudo systemctl stop postgresql

3. Отредактируем файл pg_hba.conf
   
>sudo nano /etc/postgresql/15/main/pg_hba.conf
>
TYPE      DATABASE            USER                ADDRESS                       METHOD

host      replication         postgres            10.100.110.127/32              md5
    
3. Отредактируем файл postgresql.conf
   
>sudo nano /etc/postgresql/15/main/postgresql.conf

listen_addresses = 'localhost,10.100.110.162'

wal_level = 'hot_standby'

archive_mode = on

archive_command = 'cd .'

max_wal_senders = 2

hot_standby = on


4. Удалим каталог с дефолтной БД и снова его создадим, но уже пустой:
   
>cd /var/lib/postgresql/15/
>
>sudo rm -rf main
>
>sudo mkdir main
>
>sudo chmod go-rwx main
>

>sudo pg_basebackup -P -R -X stream -c fast -h 10.100.110.127 -U postgres -D ./main

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/f796eb26-df11-4a02-b661-802d5a5a80cc)


5. Запустим постгре:
   
>sudo systemctl start postgresql

7. Подключимся к базе test_db и посмотрим на имеющиеся данные в таблице «test», после чего на ВМ 1 добавим строку в таблицу test. Видим, что записи появились:

![image](https://github.com/blaidex2/Postgres_Homework-10/assets/130083589/ba35fc53-fb96-467b-8bfa-aa7c56330aa0)

