# Домашнее задание к занятию 2. «SQL», Лебедев А.И., FOPS-10

## Введение

Перед выполнением задания вы можете ознакомиться с 
[дополнительными материалами](https://github.com/netology-code/virt-homeworks/blob/virt-11/additional/README.md).

## Задача 1

Используя Docker, поднимите инстанс PostgreSQL (версию 12) c 2 volume, 
в который будут складываться данные БД и бэкапы.

Приведите получившуюся команду или docker-compose-манифест.  

### Решение:  

- Для разворота docker + docker-compose, я воспользуюсь одним из своих ansible-хостов, которым я, как раз, поднимал эти две службы используя роли (таску из роли положу в папку ansible).

- Подготовим docker-compose.yaml, который будет соответствовать требованиям задачи:

```
version: "3"
services:
  PostgreSQL:
    image: postgres:12-bullseye
    container_name: postgres12
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    ports:
      - '5432:5432'
    volumes:
      - db:/var/lib/postgresql/data
      - backup:/var/lib/postgresql/backup
volumes:
  db:
    driver: local
  backup:
    driver: local
```

- Я специально выбрал такой тип маунта волумов. Посмотрим, куда доккер их монтирует на локальной машине по дефолту:

```
root@pg1:/home/vagrant/docker-compose# docker volume ls
DRIVER    VOLUME NAME
local     docker-compose_backup
local     docker-compose_db
```

```
root@pg1:/home/vagrant/docker-compose# docker volume inspect docker-compose_db
[
    {
        "CreatedAt": "2023-10-12T10:32:04Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "docker-compose",
            "com.docker.compose.version": "1.24.0",
            "com.docker.compose.volume": "db"
        },
        "Mountpoint": "/var/lib/docker/volumes/docker-compose_db/_data",
        "Name": "docker-compose_db",
        "Options": null,
        "Scope": "local"
    }
]
```

- Ok. Здесь меня все устраивает. Не устраивает меня тот момент, что, после входа в контейнер, используя команду **psql -U potgres**, меня пускает в PGSQL без пароля. Поправим несколько файлов - **pg_hba.conf** и **postgresql.conf**:

В **postgresql.conf** раскомментируем данную строку:  

```
#authentication_timeout = 1min          # 1s-600s
password_encryption = md5               # md5 or scram-sha-256
#db_user_namespace = off
```

В **pg_hba.conf** заменим trust на md5:  

```
# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

- Перезанрузим контейнер и попробуем зайти:

```
root@pg1:/home/vagrant/docker-compose# docker exec -it postgres12 /bin/sh
# psql -U postgres
Password for user postgres:
```

- Отлично. Идем дальше.

---

  

## Задача 2

В БД из задачи 1: 

- создайте пользователя test-admin-user и БД test_db;
- в БД test_db создайте таблицу orders и clients (спeцификация таблиц ниже);
- предоставьте привилегии на все операции пользователю test-admin-user на таблицы БД test_db;
- создайте пользователя test-simple-user;
- предоставьте пользователю test-simple-user права на SELECT/INSERT/UPDATE/DELETE этих таблиц БД test_db.

Таблица orders:

- id (serial primary key);
- наименование (string);
- цена (integer).

Таблица clients:

- id (serial primary key);
- фамилия (string);
- страна проживания (string, index);
- заказ (foreign key orders).

Приведите:

- итоговый список БД после выполнения пунктов выше;
- описание таблиц (describe);
- SQL-запрос для выдачи списка пользователей с правами над таблицами test_db;
- список пользователей с правами над таблицами test_db.

### Решение:  

- Последовательно выполним все необходимые действия:

```
root@pg1:/home/vagrant/docker-compose# docker exec -it postgres12 /bin/sh
# psql -U postgres
Password for user postgres:
psql (12.16 (Debian 12.16-1.pgdg110+1))
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

postgres=# CREATE USER test-admin-user WITH PASSWORD 'strongpassword';
ERROR:  syntax error at or near "-"
LINE 1: CREATE USER test-admin-user WITH PASSWORD 'strongpassword';
                        ^
postgres=# CREATE USER test_admin_user WITH PASSWORD 'strongpassword';
CREATE ROLE
postgres=# CREATE DATABASE test_db;
CREATE DATABASE
postgres=# \c test_db
You are now connected to database "test_db" as user "postgres".
test_db=# CREATE TABLE orders (
    id serial PRIMARY KEY,
    order_name varchar(200) NOT NULL,
    price integer
);
CREATE TABLE
test_db=# CREATE TABLE clients (
    id serial PRIMARY KEY,
    lastname varchar(255),
    country varchar(255),
    order integer,
    FOREIGN KEY (order) REFERENCES orders (id)
);
ERROR:  syntax error at or near "order"
LINE 5:     order integer,
            ^
test_db=# CREATE TABLE clients (
    id serial PRIMARY KEY,
    lastname varchar(255),
    country varchar(255),
    purchase integer,
    FOREIGN KEY (purchase) REFERENCES orders (id)
);
CREATE TABLE
test_db=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO test_admin_user;
GRANT
test_db=# create user test_simple_user with password 'simplepassword';
CREATE ROLE
test_db=# grant select, insert, update, delete on all tables in schema public to test_simple_user;
GRANT
test_db=# \l+
                                                                   List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+------------+------------+-----------------------+---------+------------+--------------------------------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |                       | 7969 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +| 7825 kB | pg_default | unmodifiable empty database
           |          |          |            |            | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +| 7825 kB | pg_default | default template for new databases
           |          |          |            |            | postgres=CTc/postgres |         |            |
 test_db   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |                       | 8097 kB | pg_default |
(4 rows)

test_db=# \dt orders
         List of relations
 Schema |  Name  | Type  |  Owner
--------+--------+-------+----------
 public | orders | table | postgres
(1 row)

test_db=# \dt clients
          List of relations
 Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
 public | clients | table | postgres
(1 row)

test_db=# create index index_country on clients (country);
CREATE INDEX
test_db=# \dt+ clients
                       List of relations
 Schema |  Name   | Type  |  Owner   |    Size    | Description
--------+---------+-------+----------+------------+-------------
 public | clients | table | postgres | 8192 bytes |
(1 row)

test_db=# SELECT grantee, table_name, privilege_type
FROM information_schema.table_privileges
WHERE table_catalog = 'test_db' AND table_schema = 'public';
     grantee      | table_name | privilege_type
------------------+------------+----------------
 postgres         | orders     | INSERT
 postgres         | orders     | SELECT
 postgres         | orders     | UPDATE
 postgres         | orders     | DELETE
 postgres         | orders     | TRUNCATE
 postgres         | orders     | REFERENCES
 postgres         | orders     | TRIGGER
 test_admin_user  | orders     | INSERT
 test_admin_user  | orders     | SELECT
 test_admin_user  | orders     | UPDATE
 test_admin_user  | orders     | DELETE
 test_admin_user  | orders     | TRUNCATE
 test_admin_user  | orders     | REFERENCES
 test_admin_user  | orders     | TRIGGER
 test_simple_user | orders     | INSERT
 test_simple_user | orders     | SELECT
 test_simple_user | orders     | UPDATE
 test_simple_user | orders     | DELETE
 postgres         | clients    | INSERT
 postgres         | clients    | SELECT
 postgres         | clients    | UPDATE
 postgres         | clients    | DELETE
 postgres         | clients    | TRUNCATE
 postgres         | clients    | REFERENCES
 postgres         | clients    | TRIGGER
 test_admin_user  | clients    | INSERT
 test_admin_user  | clients    | SELECT
 test_admin_user  | clients    | UPDATE
 test_admin_user  | clients    | DELETE
 test_admin_user  | clients    | TRUNCATE
 test_admin_user  | clients    | REFERENCES
 test_admin_user  | clients    | TRIGGER
 test_simple_user | clients    | INSERT
 test_simple_user | clients    | SELECT
 test_simple_user | clients    | UPDATE
 test_simple_user | clients    | DELETE
(36 rows)
```

- Вставил весь свой листинг не удаляя ошибок.

- Комментарии следующие - PGSQL не дает использовать символ **-** в имени пользователя, не дает создать поле order, т.к. order является зарезервированным в SQL словом, в какой-то момент я понял, что можно не писать большими буквами команды :), в какой-то момент вспомнил, что нужно было добавить индекс к полю **страна проживания**.

---






## Задача 3

Используя SQL-синтаксис, наполните таблицы следующими тестовыми данными:

Таблица orders

|Наименование|цена|
|------------|----|
|Шоколад| 10 |
|Принтер| 3000 |
|Книга| 500 |
|Монитор| 7000|
|Гитара| 4000|

Таблица clients

|ФИО|Страна проживания|
|------------|----|
|Иванов Иван Иванович| USA |
|Петров Петр Петрович| Canada |
|Иоганн Себастьян Бах| Japan |
|Ронни Джеймс Дио| Russia|
|Ritchie Blackmore| Russia|

Используя SQL-синтаксис:
- вычислите количество записей для каждой таблицы.

Приведите в ответе:

    - запросы,
    - результаты их выполнения.  

### Решение:  

- Заполним таблицы в соответствие с задачей:

```
test_db=# INSERT INTO orders (order_name, price)
VALUES
    ('Шоколад', 10),
    ('Принтер', 3000),
    ('Книга', 500),
    ('Монитор', 7000),
    ('Гитара', 4000);
INSERT 0 5
test_db=# INSERT INTO clients (lastname, country)
VALUES
    ('Иванов Иван Иванович', 'USA'),
    ('Петров Петр Петрович', 'Canada'),
    ('Иоганн Себастьян Бах', 'Japan'),
    ('Ронни Джеймс Дио', 'Russia'),
    ('Ritchie Blackmore', 'Russia');
INSERT 0 5

test_db=# SELECT * FROM clients;
 id |       lastname       | country | purchase
----+----------------------+---------+----------
  1 | Иванов Иван Иванович | USA     |
  2 | Петров Петр Петрович | Canada  |
  3 | Иоганн Себастьян Бах | Japan   |
  4 | Ронни Джеймс Дио     | Russia  |
  5 | Ritchie Blackmore    | Russia  |
(5 rows)

test_db=# SELECT * FROM orders;
 id | order_name | price
----+------------+-------
  1 | Шоколад    |    10
  2 | Принтер    |  3000
  3 | Книга      |   500
  4 | Монитор    |  7000
  5 | Гитара     |  4000
(5 rows)


test_db=# SELECT COUNT(*) AS entries_amount_orders FROM orders;
 entries_amount_orders
-----------------------
                     5
(1 row)

test_db=# SELECT COUNT(*) AS entries_amount_orders FROM clients;
 entries_amount_orders
-----------------------
                     5
(1 row)
```

- Здесь комментариев, вроде как, нет.

---


## Задача 4

Часть пользователей из таблицы clients решили оформить заказы из таблицы orders.

Используя foreign keys, свяжите записи из таблиц, согласно таблице:

|ФИО|Заказ|
|------------|----|
|Иванов Иван Иванович| Книга |
|Петров Петр Петрович| Монитор |
|Иоганн Себастьян Бах| Гитара |

Приведите SQL-запросы для выполнения этих операций.

Приведите SQL-запрос для выдачи всех пользователей, которые совершили заказ, а также вывод этого запроса.
 
Подсказка: используйте директиву `UPDATE`.  

### Решение:  

- Во втором задании при формировании полей в таблице clients, мы уже создали FOREIGN KEY, который связывает поле *purchase* таблицы **clients** с полем *id* таблицы **orders**. Попробуем использовать его:

```
test_db=# UPDATE clients
SET purchase = (SELECT id FROM orders WHERE order_name = 'Книга')
WHERE lastname = 'Иванов Иван Иванович';

UPDATE clients
SET purchase = (SELECT id FROM orders WHERE order_name = 'Монитор')
WHERE lastname = 'Петров Петр Петрович';

UPDATE clients
SET purchase = (SELECT id FROM orders WHERE order_name = 'Гитара')
WHERE lastname = 'Иоганн Себастьян Бах';
UPDATE 1
UPDATE 1
UPDATE 1
```

- Используя JOIN выведем всех клиентов, которые совершили заказ. JOIN здесь основная команда, которая сопоставляет строки из clients.purchase м orders.id. Для наглядности:

```
test_db=# SELECT * FROM clients;
 id |       lastname       | country | purchase
----+----------------------+---------+----------
  4 | Ронни Джеймс Дио     | Russia  |
  5 | Ritchie Blackmore    | Russia  |
  1 | Иванов Иван Иванович | USA     |        3
  2 | Петров Петр Петрович | Canada  |        4
  3 | Иоганн Себастьян Бах | Japan   |        5
(5 rows)
```


```
test_db=# SELECT clients.id, clients.lastname, clients.country, orders.order_name, orders.price
FROM clients
JOIN orders ON clients.purchase = orders.id;
 id |       lastname       | country | order_name | price
----+----------------------+---------+------------+-------
  1 | Иванов Иван Иванович | USA     | Книга      |   500
  2 | Петров Петр Петрович | Canada  | Монитор    |  7000
  3 | Иоганн Себастьян Бах | Japan   | Гитара     |  4000
(3 rows)
```

---



## Задача 5

Получите полную информацию по выполнению запроса выдачи всех пользователей из задачи 4 
(используя директиву EXPLAIN).

Приведите получившийся результат и объясните, что значат полученные значения.  

### Решение:    

- Выполним SQL-запрос:

```
test_db=# EXPLAIN SELECT clients.id, clients.lastname, clients.country, orders.order_name, orders.price
FROM clients
JOIN orders ON clients.purchase = orders.id;
                               QUERY PLAN
------------------------------------------------------------------------
 Hash Join  (cost=11.57..24.61 rows=70 width=1458)
   Hash Cond: (orders.id = clients.purchase)
   ->  Seq Scan on orders  (cost=0.00..11.70 rows=170 width=426)
   ->  Hash  (cost=10.70..10.70 rows=70 width=1040)
         ->  Seq Scan on clients  (cost=0.00..10.70 rows=70 width=1040)
(5 rows)
```

- Hash Join - использует операцию хеш-соединения (Hash Join) для объединения данных из таблиц orders и clients + примерная "стоимость" запроса (это очень сложная для моего понимания штука, типа, сколько постгри затратит ресурсов на выполнение) + сколько должно получиться строк и их ширина;

- Hash Cond - условие соединения, которое описывает какие столбцы используются для сравнения записей из таблиц;

- Далее идут операции сканирования и хэширования используемых таблиц и оценка их "стоимости"

---




## Задача 6

Создайте бэкап БД test_db и поместите его в volume, предназначенный для бэкапов (см. задачу 1).

Остановите контейнер с PostgreSQL, но не удаляйте volumes.

Поднимите новый пустой контейнер с PostgreSQL.

Восстановите БД test_db в новом контейнере.

Приведите список операций, который вы применяли для бэкапа данных и восстановления.   

### Решение:  

- Это, казалось бы, простое задание съело мне весь мозг.

- Начнем с того, что если мы останавливаем контейнер командой **docker compose down**, то и второй мой замонтированный волум сохраняется, соответственно, сохраняется и test_db, потому что в db-волуме лежат базы данных.

- В общем, бэкап я сделал вот так: **pg_dump -U postgres test_db > /var/lib/postgresql/backup/test_db_backup.sql**. Это был бэкап в директорию, которая связана с моим волумом.

- Потом я, просто, скопировал его в каталог в котором работаю и снёс к чертям контейнер со всеми волумами - **docker-compose down --volumes**. А после поднятия нового контейнера, закинул бэкап через волум обратно.

- Затем, создал пустую базу test_db, вышел из оснастки в контейнер и выполнил: **psql -U postgres -d test_db < /var/lib/postgresql/backup/test_db_backup.sql**

- Пользователей он не вытянул, но с базой все ок:

```
ERROR:  role "test_admin_user" does not exist
ERROR:  role "test_simple_user" does not exist
ERROR:  role "test_admin_user" does not exist
ERROR:  role "test_simple_user" does not exist

postgres=# \l+
                                                                   List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   |  Size   | Tablespace |                Description
-----------+----------+----------+------------+------------+-----------------------+---------+------------+--------------------------------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |                       | 7969 kB | pg_default | default administrative connection database
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +| 7825 kB | pg_default | unmodifiable empty database
           |          |          |            |            | postgres=CTc/postgres |         |            |
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +| 7825 kB | pg_default | default template for new databases
           |          |          |            |            | postgres=CTc/postgres |         |            |
 test_db   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |                       | 8145 kB | pg_default |
(4 rows)

postgres=# \c test_db
You are now connected to database "test_db" as user "postgres".
test_db=# SELECT * FROM clients;
 id |       lastname       | country | purchase
----+----------------------+---------+----------
  4 | Ронни Джеймс Дио     | Russia  |
  5 | Ritchie Blackmore    | Russia  |
  1 | Иванов Иван Иванович | USA     |        3
  2 | Петров Петр Петрович | Canada  |        4
  3 | Иоганн Себастьян Бах | Japan   |        5
(5 rows)
```



---

