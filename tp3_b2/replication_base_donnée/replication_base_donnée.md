# Module 2 : Réplication de base de données

Il y a plein de façons de mettre en place de la réplication de base données de type MySQL (comme MariaDB).

MariaDB possède un mécanisme de réplication natif qui peut très bien faire l'affaire pour faire des tests comme les nôtres.

Une réplication simple est une configuration de type "master-slave". Un des deux serveurs est le *master* l'autre est un *slave*.

Le *master* est celui qui reçoit les requêtes SQL (des applications comme NextCloud) et qui les traite.

Le *slave* ne fait que répliquer les donneés que le *master* possède.

La [doc officielle de MariaDB](https://mariadb.com/kb/en/setting-up-replication/) ou encore [cet article cool](https://cloudinfrastructureservices.co.uk/setup-mariadb-replication/) expose de façon simple comment mettre en place une telle config.

Pour ce module, vous aurez besoin d'un deuxième serveur de base de données.

✨ **Bonus** : Faire en sorte que l'utilisateur créé en base de données ne soit utilisable que depuis l'autre serveur de base de données

- inspirez-vous de la création d'utilisateur avec `CREATE USER` effectuée dans le TP2

✨ **Bonus** : Mettre en place un setup *master-master* où les deux serveurs sont répliqués en temps réel, mais les deux sont capables de traiter les requêtes.


###Step 1

```
[archi@db2 ~]$ sudo dnf install -y mariadb-server
[...]
Complete!
```

```
[archi@db2 ~]$ sudo systemctl start mariadb
[sudo] password for archi:

[archi@db2 ~]$ sudo systemctl enable mariadb
Created symlink /etc/systemd/system/mysql.service → /usr/lib/systemd/system/mariadb.service.
Created symlink /etc/systemd/system/mysqld.service → /usr/lib/systemd/system/mariadb.service.
Created symlink /etc/systemd/system/multi-user.target.wants/mariadb.service → /usr/lib/systemd/system/mariadb.service.
```

```
[archi@db2 ~]$ sudo mysql_secure_installation

[archi@db2 ~]$ Enter current password for root (enter for none): Enter
[archi@db2 ~]$ Set root password? [Y/n] Y
[archi@db2 ~]$ Remove anonymous users? [Y/n] Y
[archi@db2 ~]$ Disallow root login remotely? [Y/n] Y
[archi@db2 ~]$ Remove test database and access to it? [Y/n] Y
[archi@db2 ~]$ Reload privilege tables now? [Y/n] Y
```

###Step 2

```
[archi@db ~]$ sudo cat /etc/my.cnf

[...]

[mariadb]

server-id              = 1
log_bin                = /var/log/mariadb/mariadb-bin.log
max_binlog_size        = 100M
relay_log = /var/log/mariadb/mariadb-relay-bin
relay_log_index = /var/log/mariadb/mariadb-relay-bin.index
```

```
[archi@db ~]$ sudo systemctl restart mariadb
```

```
[archi@db ~]$ sudo ss -antpl | grep mariadb
LISTEN 0      80                 *:3306            *:*    users:(("mariadbd",pid=1334,fd=116))
```

###Step 3

```
[archi@db ~]$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 4
Server version: 10.5.16-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

```
MariaDB [(none)]> CREATE USER 'replication'@'%' identified by 'toto';
Query OK, 0 rows affected (0.003 sec)

MariaDB [(none)]> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
Query OK, 0 rows affected (0.002 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> SHOW MASTER STATUS;
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000002 |      778 |              |                  |
+--------------------+----------+--------------+------------------+
1 row in set (0.000 sec)

MariaDB [(none)]> EXIT;
Bye
```

###Step 4

```
[archi@db2 ~]$ sudo cat /etc/my.cnf
[...]
[mariadb]

server-id              = 2
log_bin                = /var/log/mariadb/mariadb-bin.log
max_binlog_size        = 100M
relay_log = /var/log/mariadb/mariadb-relay-bin
relay_log_index = /var/log/mariadb/mariadb-relay-bin.index

```

```
[archi@db2 ~]$ sudo systemctl restart mariadb
```

```
[archi@db2 ~]$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 4
Server version: 10.5.16-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```

```
MariaDB [(none)]> STOP SLAVE;
Query OK, 0 rows affected, 1 warning (0.000 sec)

MariaDB [(none)]> CHANGE MASTER TO MASTER_HOST = '10.102.1.12', MASTER_USER = 'replication', MASTER_PASSWORD = 'toto', MASTER_LOG_FILE = 'mariadb-bin.000002', MASTER_LOG_POS = 778;
Query OK, 0 rows affected (0.021 sec)

MariaDB [(none)]> START SLAVE;
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> EXIT;
Bye
```

###step 5

```
[archi@db ~]$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 6
Server version: 10.5.16-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```

```
MariaDB [(none)]> CREATE DATABASE schooldb;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> USE schooldb
Database changed
MariaDB [schooldb]> CREATE TABLE students (id int, name varchar(20), surname varchar(20));
Query OK, 0 rows affected (0.012 sec)

MariaDB [schooldb]> INSERT INTO students VALUES (1,"hitesh","jethva");
Query OK, 1 row affected (0.004 sec)

MariaDB [schooldb]> INSERT INTO students VALUES (2,"jayesh","jethva");
Query OK, 1 row affected (0.002 sec)
```

```
MariaDB [schooldb]> SELECT * FROM students;
+------+--------+---------+
| id   | name   | surname |
+------+--------+---------+
|    1 | hitesh | jethva  |
|    2 | jayesh | jethva  |
+------+--------+---------+
2 rows in set (0.000 sec)
```

```
MariaDB [(none)]> SHOW SLAVE STATUS \G
*************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 10.102.1.12
                   Master_User: replication
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: mariadb-bin.000002
           Read_Master_Log_Pos: 1483
                Relay_Log_File: mariadb-relay-bin.000002
                 Relay_Log_Pos: 1262
         Relay_Master_Log_File: mariadb-bin.000002
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
               Replicate_Do_DB:
           Replicate_Ignore_DB:
            Replicate_Do_Table:
        Replicate_Ignore_Table:
       Replicate_Wild_Do_Table:
   Replicate_Wild_Ignore_Table:
                    Last_Errno: 0
                    Last_Error:
                  Skip_Counter: 0
           Exec_Master_Log_Pos: 1483
               Relay_Log_Space: 1573
               Until_Condition: None
                Until_Log_File:
                 Until_Log_Pos: 0
            Master_SSL_Allowed: No
            Master_SSL_CA_File:
            Master_SSL_CA_Path:
               Master_SSL_Cert:
             Master_SSL_Cipher:
                Master_SSL_Key:
         Seconds_Behind_Master: 0
 Master_SSL_Verify_Server_Cert: No
                 Last_IO_Errno: 0
                 Last_IO_Error:
                Last_SQL_Errno: 0
                Last_SQL_Error:
   Replicate_Ignore_Server_Ids:
              Master_Server_Id: 1
                Master_SSL_Crl:
            Master_SSL_Crlpath:
                    Using_Gtid: No
                   Gtid_IO_Pos:
       Replicate_Do_Domain_Ids:
   Replicate_Ignore_Domain_Ids:
                 Parallel_Mode: optimistic
                     SQL_Delay: 0
           SQL_Remaining_Delay: NULL
       Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
              Slave_DDL_Groups: 2
Slave_Non_Transactional_Groups: 0
    Slave_Transactional_Groups: 2
1 row in set (0.000 sec)
```

```
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| schooldb           |
+--------------------+
4 rows in set (0.000 sec)
```

```
MariaDB [(none)]> USE schooldb;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [schooldb]> SHOW TABLES;
+--------------------+
| Tables_in_schooldb |
+--------------------+
| students           |
+--------------------+
1 row in set (0.000 sec)
```

```
MariaDB [schooldb]> SELECT * FROM students;
+------+--------+---------+
| id   | name   | surname |
+------+--------+---------+
|    1 | hitesh | jethva  |
|    2 | jayesh | jethva  |
+------+--------+---------+
2 rows in set (0.000 sec)
```