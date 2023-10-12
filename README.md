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

## Задача 5

Получите полную информацию по выполнению запроса выдачи всех пользователей из задачи 4 
(используя директиву EXPLAIN).

Приведите получившийся результат и объясните, что значат полученные значения.

## Задача 6

Создайте бэкап БД test_db и поместите его в volume, предназначенный для бэкапов (см. задачу 1).

Остановите контейнер с PostgreSQL, но не удаляйте volumes.

Поднимите новый пустой контейнер с PostgreSQL.

Восстановите БД test_db в новом контейнере.

Приведите список операций, который вы применяли для бэкапа данных и восстановления. 

---

### Как cдавать задание

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
