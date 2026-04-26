# Домашнее задание к занятию «Репликация и масштабирование. Часть 1» - Стрельников Александр

---

### Задание 1

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

*Ответить в свободной форме.*

### Решение

Master-Slave (ведущий-ведомый): Репликация работает в одном направлении — с одного главного сервера (master) на один или несколько подчинённых (slave), где slave предназначены только для чтения и выступают как резервные копии или для распределения нагрузки при SELECT-запросах. Master-Master (ведущий-ведущий): Оба сервера одновременно являются master’ами и реплицируют изменения друг другу, что позволяет писать данные на любой из узлов, но требует более сложной обработки конфликтов (например, при автоинкременте или одновременном изменении одной строки).

---

### Задание 2

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

*Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.*

### Решение

1. Для настройки Master-Slave репликации были созданы два контейнера MySQL с помощью Docker Compose. Мастер (mysql_master) настроен с server-id=1, включённым бинарным логом и базой repl_db для репликации. Слейв (mysql_slave) настроен с server-id=2, включённым релейным логом и режимом read-only=1.

Файл docker-compose.yml
```
services:
  master:
    image: mysql:8.0
    container_name: mysql_master
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: repl_db
      MYSQL_USER: repl_user
      MYSQL_PASSWORD: repl_pass
    ports:
      - "3306:3306"
    volumes:
      - ./master_data:/var/lib/mysql
      - ./master_my.cnf:/etc/mysql/conf.d/my.cnf
    networks:
      - mysql_network

  slave:
    image: mysql:8.0
    container_name: mysql_slave
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
    ports:
      - "3307:3306"
    volumes:
      - ./slave_data:/var/lib/mysql
      - ./slave_my.cnf:/etc/mysql/conf.d/my.cnf
    networks:
      - mysql_network
    depends_on:
      - master

networks:
  mysql_network:
    driver: bridge
```

файл конфигурации мастера master_my.cnf
```
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-do-db = repl_db
binlog-format = ROW
```

файл конфигурации слейва slave_my.cnf
```
[mysqld]
server-id = 2
relay-log = relay-log
read-only = 1
log-bin = mysql-bin
```

2. На мастере создан пользователь repl_user с правами REPLICATION SLAVE. На слейве выполнена команда CHANGE MASTER TO с указанием хоста мастера, логина/пароля и координат бинарного лога (файл mysql-bin.000003, позиция 549), после чего запущена репликация командой START SLAVE.

Настройка мастера
```
CREATE USER 'repl_user'@'%' IDENTIFIED WITH mysql_native_password BY 'repl_pass';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;
```
![config_master](https://github.com/Stvrrow/sdb-hw-12-06/blob/main/img/img2.png)

Настройка слейва
```
CHANGE MASTER TO
  MASTER_HOST = 'master',
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'repl_pass',
  MASTER_LOG_FILE = 'mysql-bin.000003',   -- из SHOW MASTER STATUS
  MASTER_LOG_POS = 549,                   -- из SHOW MASTER STATUS
  GET_MASTER_PUBLIC_KEY = 1;

START SLAVE;

SHOW SLAVE STATUS\G
```

![config_slave](https://github.com/Stvrrow/sdb-hw-12-06/blob/main/img/img3.png)

3. Проверка статусов показала Slave_IO_Running: Yes и Slave_SQL_Running: Yes, что подтверждает корректную работу репликации. Также выполнена проверка записью - данные, добавленные на мастере, успешно появились на слейве.

На мастере создадим тестовую таблицу

```
USE repl_db;
CREATE TABLE test (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO test VALUES (1, 'hello');
```

![master_table](https://github.com/Stvrrow/sdb-hw-12-06/blob/main/img/img4.png)

Проверим, что таблица появилась на слейве:

![slave_table](https://github.com/Stvrrow/sdb-hw-12-06/blob/main/img/img5.png)


Скриншот состояния и режимы работы серверов.

![config](https://github.com/Stvrrow/sdb-hw-12-06/blob/main/img/img6.png)
