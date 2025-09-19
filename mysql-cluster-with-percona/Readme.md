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

### 3a. Configure MySQL server (Slave)

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

```ini
bind-address = 0.0.0.0
mysqlx-bind-address = 0.0.0.0

[mysqld]
server-id=2
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
sudo xtrabackup --prepare --target-dir=/backup/mysql
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
sudo rm -rf /var/lib/mysql
cd /tmp
sudo mkdir -p /var/lib/mysql
sudo chown -R mysql:mysql /var/lib/mysql
sudo chmod 750 /var/lib/mysql
sudo tar -xvzf mysql-backup.tar.gz
sudo mv mysql/* /var/lib/mysql/
sudo chown -R mysql:mysql /var/lib/mysql
```

### 7a. Recreate the binary log index to prevent startup errors

```bash
sudo -u mysql touch /var/lib/mysql/mysql-bin.index
sudo chown mysql:mysql /var/lib/mysql/mysql-bin.index
sudo systemctl start mysql
sudo systemctl status mysql
```

## 8. Configure Slave replication

```bash
sudo mysql -u root -p
```

```sql
STOP SLAVE;
RESET SLAVE ALL;

CHANGE MASTER TO
  MASTER_HOST='<master_ip>',
  MASTER_USER='repl',
  MASTER_PASSWORD='ReplPass123!',
  MASTER_AUTO_POSITION=1;

START SLAVE;
SHOW SLAVE STATUS\G
```

## 9. Verify replication

```sql
SHOW MASTER STATUS\G
SHOW SLAVE STATUS\G
```

Check that:

* `Slave_IO_Running: Yes`
* `Slave_SQL_Running: Yes`
* `Seconds_Behind_Master: 0`

## 10. Notes / Best Practices

* Open **port 3306** between master and slave.
* Set `bind-address=0.0.0.0` on both nodes.
* Use **GTID replication** for automatic failover.
* Ensure `/var/lib/mysql` exists and has correct ownership (`mysql:mysql`) before starting MySQL.
* If binary logs are enabled, always create `mysql-bin.index` manually if missing.
* Test backup and restore on slave before configuring replication.
