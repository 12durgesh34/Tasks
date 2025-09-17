# 📘 MySQL InnoDB Cluster Setup (Step-by-Step Guide)

This document explains how to **install, configure, and manage a MySQL InnoDB Cluster** with multiple nodes.  
It covers **installation, configuration, cluster creation, adding nodes, verification, and testing replication**.  

---

## 🖥️ 1. Install MySQL on Each Node

Run the following commands on **every node** (Primary and Secondaries):

```bash
sudo apt update
sudo apt install -y mysql-server
mysql --version
```


## 🖥️ 2. Configure MySQL Root User
Login into MySQL:
```
sudo mysql
```

### Now run the following SQL commands:
```
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY 'RootPass123!';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

```
🔎 Explanation:

`root@'%' `→ Allows the root user to connect from any IP address (needed for cluster nodes).

`GRANT ALL PRIVILEGES` → Gives root user full permissions.

`FLUSH PRIVILEGES` → Refreshes MySQL permission system.

---

### Switch the root authentication method

```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'RootPass123!';
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'RootPass123!';
FLUSH PRIVILEGES;
```
🔎 Explanation:

By default, MySQL uses caching_sha2_password, which sometimes causes issues.

Switching to mysql_native_password ensures compatibility with MySQL Shell and Group Replication.

---

## 3. Configure MySQL (mysqld.cnf)

### Edit the MySQL config file:
```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

### Example configuration for Primary Node (IP: 172.31.100.15):

```
[mysqld]
user            = mysql
bind-address    = 0.0.0.0
mysqlx-bind-address = 0.0.0.0
binlog_transaction_dependency_tracking = WRITESET
# Basic tuning
key_buffer_size = 16M
max_binlog_size = 100M

# Replication & binlog
server-id = 1   # must be unique per node
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
master_info_repository = TABLE
relay_log_info_repository = TABLE
relay_log_recovery = ON
binlog_checksum = NONE
log_slave_updates = ON
read_only = OFF

# Group Replication
transaction_write_set_extraction = XXHASH64
loose-group_replication_bootstrap_group = OFF
loose-group_replication_start_on_boot = OFF   # <-- important change
loose-group_replication_local_address = "172.31.100.15:33061"
loose-group_replication_group_seeds = ""172.31.100.15:33061,172.31.100.51:33061,172.31.100.85:33061"
loose-group_replication_group_name = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
report_host = 172.31.100.15# <-- important fix

# Optional: increase connections for cluster
max_connections = 500

```

### Explanation:

`server-id` → Must be unique for each node (1,2,3...).

`binlog_format = ROW` → Required for replication to work correctly.

`gtid_mode & enforce_gtid_consistency` → Enables Global Transaction Identifiers for replication.

`group_replication_local_address` → Internal communication port (33061).

`group_replication_group_seeds` → All nodes in the cluster, used for discovery.

`report_host` → The actual IP of the node (must be correct).

---

## 4. Restart MySQL

```
sudo systemctl restart mysql

sudo systemctl status mysql

```

## 5. Verify Ports

```
sudo lsof -i :3306
sudo lsof -i :33061
```
#### 🔎 Explanation:

`3306` → Standard MySQL connections.

`33061` → Used for Group Replication.

Both must be open for the cluster to work.

---

7. Create Cluster on Primary 6. Install MySQL Shell

```
sudo apt install -y mysql-shell
```
#### 🔎 Explanation:

MySQL Shell (mysqlsh) is required to create and manage InnoDB Clusters.

---

## 7. Create Cluster on Primary

Connect to the primary node-
```
mysqlsh root@<primary-server-ip>:3306 -p

```
Inside MySQL Shell:
```
cluster = dba.create_cluster("myCluster")
```
Check cluster:
```
cluster = dba.get_cluster("myCluster")
cluster.status()
```
#### 🔎 Explanation:

`create_cluster()` → Initializes the InnoDB Cluster on the primary node.

`get_cluster()` → Retrieves the cluster object for management.

`cluster.status()` → Displays health, roles, and replication status.

---
## 8. Add Secondary Nodes

From the Primary node MySQL Shell:
```
cluster.add_instance('root@172.31.100.51:3306')
cluster.add_instance('root@172.31.100.69:3306')
cluster.status()
```
#### 🔎 Explanation:

Adds replicas to the cluster.

status() confirms all members are ONLINE.

---

## 9. Change Primary Node
>
>If you want to promote another node as Primary
>
```
cluster.set_primary_instance("172.31.100.16:3306")
```

#### 🔎 Explanation:

Ensures automatic failover and manual promotion are supported.

## 10. Verify Cluster Members
```
SELECT MEMBER_HOST, MEMBER_PORT, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```
#### 🔎 Explanation:

Displays all cluster members, their ports, and roles (PRIMARY / SECONDARY).

---

## 11. Data Generator Script (Test Replication)

Create a file called `datagenerator.sh:`

```
#!/bin/bash
PRIMARY=172.31.100.15
USER=root
PASS="RootPass123!"

# Ensure database and table exist
mysql -h $PRIMARY -u $USER -p$PASS -e "
CREATE DATABASE IF NOT EXISTS testdb;
USE testdb;
CREATE TABLE IF NOT EXISTS cluster_test (
  id INT AUTO_INCREMENT PRIMARY KEY,
  msg VARCHAR(255),
  ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
"

# Loop to keep inserting rows
while true; do
  MSG="Insert at $(date +%T)"
  mysql -h $PRIMARY -u $USER -p$PASS -e "USE testdb; INSERT INTO cluster_test (msg) VALUES ('$MSG');"
  echo "Inserted: $MSG"
  sleep 2
done

```

#### 🔎 Explanation:

Creates a testdb database and a cluster_test table.

Inserts a new row with a timestamp every 2 seconds.

Used to verify replication across all nodes.

