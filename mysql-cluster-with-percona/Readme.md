# MySQL GTID-Based Replication Cluster Setup

This README explains how to set up a **MySQL master-slave replication cluster** with GTID, where the master’s existing data is copied to the slave and kept in sync.

---

## 1. Install MySQL on both nodes

```bash
sudo apt update
sudo apt install -y mysql-server
mysql --version
```

## 2. Configure MySQL users

```bash
sudo mysql
```

```sql
-- Root user with remote access
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY 'RootPass123!';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

-- Ensure localhost root uses native password
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'RootPass123!';
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'RootPass123!';
FLUSH PRIVILEGES;

-- Replication user for slave connection
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'ReplPass123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

## 3. Configure MySQL server (Master)

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

```ini
bind-address = 0.0.0.0
mysqlx-bind-address = 0.0.0.0

[mysqld]
server-id=1
log_bin=mysql-bin
binlog_format=ROW
gtid_mode=ON
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
log_slave_updates=ON
binlog_transaction_dependency_tracking=WRITESET
binlog_checksum=NONE
```

Restart MySQL:

```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

## 4. Install Percona XtraBackup

```bash
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update
sudo percona-release enable-only tools release
sudo apt install percona-xtrabackup-80 -y
```

## 5. Take a backup of Master data

```bash
sudo mkdir -p /backup/mysql
sudo chown -R mysql:mysql /backup/mysql
sudo xtrabackup --backup --target-dir=/backup/mysql --user=root --password='RootPass123!'
```

## 6. Transfer backup to Slave

Compress backup:

```bash
cd /backup
sudo tar -czvf mysql-backup.tar.gz mysql
```

Transfer via SCP:

```bash
scp -i /home/ubuntu/mysql.pem /backup/mysql-backup.tar.gz ubuntu@<slave_ip>:/tmp/
```

## 7. Restore backup on Slave

```bash
sudo systemctl stop mysql
sudo
```
