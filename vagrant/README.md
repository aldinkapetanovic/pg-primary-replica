---
marp: false
---

## Primary
```sh
vagrant up
```
```sh
vagrant ssh primary
```
```sh
vagrant ssh replica
```
---

```sh
sudo vim /etc/postgresql/14/main/postgresql.conf
```
```
listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
hot_standby = on
```
```sh
sudo vim /etc/postgresql/14/main/pg_hba.conf
```
```
host replication all 192.168.56.0/24 md5
```
```sh
sudo systemctl restart postgresql
```
```sh
sudo -u postgres psql
```
```sql
CREATE USER replication_user WITH REPLICATION ENCRYPTED PASSWORD 'replication_pass';
```
## Replica

```sh
vagrant ssh replica
```
```sh
sudo systemctl stop postgresql
```
```sh
sudo rm -rf /var/lib/postgresql/14/main/*
```
```sh
sudo -u postgres pg_basebackup -h 192.168.56.3 -D /var/lib/postgresql/14/main -U replication_user -v -P --password
```
```sh
sudo touch /var/lib/postgresql/14/main/standby.signal
```
```sh
sudo vim /etc/postgresql/14/main/postgresql.conf
```
```
primary_conninfo = 'host=192.168.56.3 port=5432 user=replication_user password=replication_pass'
```
```sh
sudo systemctl start postgresql
```

## Verify Replication

### primary

```sh
sudo -u postgres psql -c "CREATE DATABASE test_db;"
```
```sh
sudo -u postgres psql -d test_db -c "CREATE TABLE test_table (id SERIAL PRIMARY KEY, name VARCHAR(100));"
```
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    age INT,
    address TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
```sql
-- Insert 5 users with additional information
INSERT INTO users (name, email, age, address) VALUES
    ('Alice Smith', 'alice.smith@example.com', 30, '123 Maple Street, Springfield'),
    ('Bob Johnson', 'bob.johnson@example.com', 25, '456 Oak Avenue, Shelbyville'),
    ('Charlie Brown', 'charlie.brown@example.com', 35, '789 Pine Road, Capital City'),
    ('David White', 'david.white@example.com', 40, '101 Birch Lane, Smalltown'),
    ('Eva Green', 'eva.green@example.com', 28, '202 Cedar Blvd, Rivertown');
```
```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    product_name VARCHAR(100),
    quantity INT,
    price DECIMAL(10, 2),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
```sql
-- Insert 5 orders
INSERT INTO orders (user_id, product_name, quantity, price) VALUES
    (1, 'Laptop', 1, 999.99),
    (2, 'Smartphone', 2, 499.49),
    (3, 'Headphones', 3, 79.99),
    (4, 'Tablet', 1, 329.99),
    (5, 'Smartwatch', 1, 199.99);
```

### replica

```sh
sudo -u postgres psql -d test_db -c "SELECT * FROM test_table;"
```

## Backup
```sh
postgres@replica:~$ pg_dump -U postgres -F c -b -v -f test_db_replica_backup.dump test_db
```
### (SQL Format)
```sh
pg_dump -U postgres -h 127.0.0.1 -p 5432 -F p -v -f /path/to/backup.sql mydatabase
```

## Restore
```sh
postgres@primary:~$ pg_restore -U postgres -d test_db -v /vagrant/test_db_replica_backup.dump 
```
### (SQL Format)
```sh
psql -U postgres -h 127.0.0.1 -p 5432 -d mydatabase -f /path/to/backup.sql
```


## Perks

```sh
postgres=# select pg_is_in_recovery();
```

```sh
postgres=# select pg_promote();
```