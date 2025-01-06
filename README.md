# MySQL Primary/Replica Setup with Docker

This guide documents the setup of a MySQL Primary/Replica configuration using Docker containers, including PHPMyAdmin for database management.

## 1. Docker Compose Configuration

Create a `docker-compose.yml` file with the following content:

```yaml
version: '3.8'

services:
  mysql-primary:
    image: mysql:latest
    container_name: mysql-primary
    environment:
      MYSQL_ROOT_PASSWORD: your-root-password
    ports:
      - "3306:3306"
    command: >
      --server-id=1
      --log-bin=mysql-bin
      --binlog-format=ROW
      --sync-binlog=1
      --gtid-mode=ON
      --enforce-gtid-consistency=ON
      --log-replica-updates=ON
    volumes:
      - primary_data:/var/lib/mysql
    networks:
      - mysql

  mysql-replica:
    image: mysql:latest
    container_name: mysql-replica
    environment:
      MYSQL_ROOT_PASSWORD: your-root-password
    ports:
      - "3307:3306"
    command: >
      --server-id=2
      --log-bin=mysql-bin
      --binlog-format=ROW
      --sync-binlog=1
      --gtid-mode=ON
      --enforce-gtid-consistency=ON
      --log-replica-updates=ON
    volumes:
      - replica_data:/var/lib/mysql
    networks:
      - mysql

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    environment:
      PMA_HOSTS: mysql-primary,mysql-replica
      PMA_USER: root
      PMA_PASSWORD: your-root-password
      MYSQL_ROOT_PASSWORD: your-root-password
      PMA_ABSOLUTE_URI: http://localhost:8080/
      BLOWFISH_SECRET: your-blowfish-secret
    ports:
      - "8080:80"
    networks:
      - mysql

networks:
  mysql:
    name: mysql
    driver: bridge

volumes:
  primary_data:
  replica_data:
```

## 2. Initial Setup

Start the containers:
```bash
docker-compose down -v  # Clean up any existing setup
docker-compose up -d   # Start the containers
```

Wait about 30 seconds for the containers to fully initialize.

## 3. Replication Setup

Execute these commands in order:

1. Create replication user on primary:
```bash
docker exec -it mysql-primary mysql -uroot -p'your-root-password' -e "
CREATE USER 'replica_user'@'%' IDENTIFIED WITH caching_sha2_password BY 'your-root-password';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'%';
FLUSH PRIVILEGES;"
```

2. Create a backup of the primary:
```bash
docker exec mysql-primary mysqldump -uroot -p'your-root-password' --all-databases --master-data=1 --single-transaction > backup.sql
```

3. Import the backup to replica:
```bash
docker exec -i mysql-replica mysql -uroot -p'your-root-password' < backup.sql
```

4. Set read-only mode on replica:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;"
```

5. Configure and start replication:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='mysql-primary',
    SOURCE_USER='replica_user',
    SOURCE_PASSWORD='your-root-password',
    SOURCE_AUTO_POSITION=1,
    GET_SOURCE_PUBLIC_KEY=1;

START REPLICA;"
```

6. Check replica status:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "SHOW REPLICA STATUS\G"
```

## 4. Testing the Setup

1. Test writing to primary:
```bash
docker exec -it mysql-primary mysql -uroot -p'your-root-password' -e "
CREATE DATABASE test_replication;
USE test_replication;
CREATE TABLE test (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO test VALUES (1, 'test replication');"
```

2. Verify replication to replica:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "
USE test_replication;
SELECT * FROM test;"
```

3. Verify replica is read-only:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "
USE test_replication;
INSERT INTO test VALUES (2, 'this should fail');"
```
This should fail with an error about super-read-only mode.

## 5. Access Details

- Primary MySQL: localhost:3306
- Replica MySQL: localhost:3307
- PHPMyAdmin: http://localhost:8080
- Login credentials:
  - Username: root
  - Password: your-root-password

## 6. Network Information

All containers are connected to a Docker network named 'mysql', which allows them to communicate with each other using their service names as hostnames.

## 7. Maintenance Notes

- All data is persisted in Docker volumes: `primary_data` and `replica_data`
- The replica is configured as read-only and will reject direct write operations
- All write operations should be performed on the primary server
- Both servers can be managed through PHPMyAdmin
- The replica can be used for read operations to reduce load on the primary

## 8. Success Criteria

The setup is working correctly when:
- The primary accepts write operations
- Changes on the primary are replicated to the replica
- The replica rejects direct write operations
- PHPMyAdmin can connect to both servers
- SHOW REPLICA STATUS shows no errors

## 9. Container Restart Behavior

### Automatic Replication Recovery
The setup uses GTID (Global Transaction Identifier) based replication, which ensures automatic recovery after container restarts. Here's what happens in different restart scenarios:

1. Primary (Source) MySQL Restart:
- All configurations are preserved in the Docker volume
- When it comes back online, it's immediately ready to continue as the source
- The replica will automatically reconnect and continue replication
- Any transactions that occurred before shutdown will be tracked by GTIDs

2. Replica MySQL Restart:
- Replication configuration is preserved in the Docker volume
- GTID tracking ensures it knows exactly where it left off
- Replication automatically resumes without manual intervention
- The read-only settings are preserved

3. Both Containers Restart:
- The same GTID-based recovery applies
- Replication will resume automatically
- The replica will catch up with any missed transactions

### Verifying Replication After Restart
You can verify the replication status after any restart using:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "SHOW REPLICA STATUS\G"
```

Key status indicators:
- `Replica_IO_Running: Yes`
- `Replica_SQL_Running: Yes`
- No errors in `Last_IO_Error` or `Last_SQL_Error`

### Manual Intervention
Manual intervention is typically only needed if:
- There was a replication error (conflict or failed transaction)
- Database corruption occurred
- Network partition lasted longer than timeout values

## 10. Large Data Import Configuration
The primary MySQL server is configured to handle larger data imports with these settings:

```yaml
command: >
  [previous settings...]
  --max_allowed_packet=20M
  --net_buffer_length=1M
  --innodb_buffer_pool_size=1G
  --interactive_timeout=600
  --wait_timeout=600
```

PHPMyAdmin is also configured to handle larger uploads:
```yaml
environment:
  [previous settings...]
  UPLOAD_LIMIT: 20M
```

These settings allow:
- Upload of files up to 20MB in size
- Better performance for large data operations
- Longer timeouts for big import operations
- Improved memory allocation for large datasets

The replica server maintains the default settings since it doesn't handle direct imports.

## 11. Refreshing Primary Database from Production Dump

When developing locally, you may need to refresh your primary database with a new dump from production. Here's the safe procedure that preserves the replication setup:

### Step-by-Step Process

1. Stop the replica to prevent replication during the update:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "STOP REPLICA;"
```

2. Drop and recreate the database on primary (replace 'your_database' with your actual database name):
```bash
docker exec -it mysql-primary mysql -uroot -p'your-root-password' -e "
DROP DATABASE your_database;
CREATE DATABASE your_database;"
```

3. Load the production dump to primary:
```bash
# If your dump file is on your local machine
docker exec -i mysql-primary mysql -uroot -p'your-root-password' your_database < /path/to/your/production_dump.sql
```

4. Create a new backup of the primary that includes GTID information:
```bash
docker exec mysql-primary mysqldump -uroot -p'your-root-password' --all-databases --master-data=1 --single-transaction > backup.sql
```

5. Drop and reload the replica:
```bash
# Drop and recreate database on replica
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "
DROP DATABASE your_database;
CREATE DATABASE your_database;"

# Load the new backup to replica
docker exec -i mysql-replica mysql -uroot -p'your-root-password' < backup.sql
```

6. Reset and restart replication:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "
STOP REPLICA;
RESET REPLICA ALL;
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='mysql-primary',
    SOURCE_USER='replica_user',
    SOURCE_PASSWORD='your-root-password',
    SOURCE_AUTO_POSITION=1,
    GET_SOURCE_PUBLIC_KEY=1;
START REPLICA;"
```

7. Verify replication is working:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "SHOW REPLICA STATUS\G"
```

### Important Notes
- Always verify replication is working after the refresh
- Make sure you have enough disk space for both the dump file and the database
- The process will cause a brief interruption in replication
- The replica will be temporarily out of sync during the process
- GTID-based replication ensures proper synchronization after the refresh

## 12. Removing Databases from the Setup

While DROP DATABASE commands are replicated from primary to replica, it's important to verify the removal is successful on both servers. Here's the safe procedure:

### Verifying and Removing Databases

1. First check replication status:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "SHOW REPLICA STATUS\G"
```

2. Drop the database from the primary:
```bash
docker exec -it mysql-primary mysql -uroot -p'your-root-password' -e "DROP DATABASE database_name;"
```

3. Verify the database was dropped from the replica:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "SHOW DATABASES;"
```

4. If the database still exists on the replica (which might happen if replication had issues), manually drop it:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "DROP DATABASE database_name;"
```

### Important Notes
- Always verify replication status before dropping databases
- The DROP DATABASE command will replicate if replication is working properly
- Manual cleanup on the replica might be needed if replication was not active
- Always verify the database is removed from both servers
- Consider backing up important data before dropping databases

## 13. Adding New Databases to the Replication Setup

When adding a new database to your replication setup, you only need to create it on the primary. The creation and all subsequent changes will automatically replicate to the replica server.

### Creating a New Database

1. First verify replication is working:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "SHOW REPLICA STATUS\G"
```

2. Create the new database on the primary:
```bash
docker exec -it mysql-primary mysql -uroot -p'your-root-password' -e "CREATE DATABASE new_database_name;"
```

3. Verify the database was created on the primary:
```bash
docker exec -it mysql-primary mysql -uroot -p'your-root-password' -e "SHOW DATABASES;"
```

4. Verify the database was replicated to the replica:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "SHOW DATABASES;"
```

### Testing the New Database

1. Create a test table on the primary:
```bash
docker exec -it mysql-primary mysql -uroot -p'your-root-password' -e "
USE new_database_name;
CREATE TABLE test (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO test VALUES (1, 'replication test');"
```

2. Verify the table and data appear on the replica:
```bash
docker exec -it mysql-replica mysql -uroot -p'your-root-password' -e "
USE new_database_name;
SELECT * FROM test;"
```

### Important Notes
- Only create databases on the primary server
- All CREATE, ALTER, and DROP operations will automatically replicate
- The replica should remain read-only; all changes must come through replication
- If the database doesn't appear on the replica, check replication status
- Remember that the replica is read-only, so you cannot create databases directly on it

## 14. PHPMyAdmin Configuration Storage

When using PHPMyAdmin with a primary/replica setup, you may encounter a warning message on the replica interface about not being able to save configuration due to the super-read-only setting. This is normal behavior but can be resolved.

### The Issue
When accessing the replica server through PHPMyAdmin, you might see:
```
Could not save configuration
#1290 - the mysql server is running with the super-read-only option so it cannot execute this statement
```

### Solution
Configure PHPMyAdmin to store its configuration only on the primary server by updating the PHPMyAdmin service in your docker-compose.yml:

```yaml
  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    environment:
      PMA_HOSTS: mysql-primary,mysql-replica
      PMA_USER: root
      PMA_PASSWORD: your-root-password
      MYSQL_ROOT_PASSWORD: your-root-password
      PMA_ABSOLUTE_URI: http://localhost:8080/
      BLOWFISH_SECRET: your-blowfish-secret
      UPLOAD_LIMIT: 20M
      PMA_PMADB: phpmyadmin
      PMA_CONTROLHOST: mysql-primary
      PMA_CONTROLUSER: root
      PMA_CONTROLPASS: your-root-password
    ports:
      - "8080:80"
    networks:
      - mysql
```

Key configuration additions:
- `PMA_PMADB`: Specifies the configuration database name
- `PMA_CONTROLHOST`: Directs PHPMyAdmin to store configuration only on the primary
- `PMA_CONTROLUSER` and `PMA_CONTROLPASS`: Credentials for configuration storage

### Applying the Changes
```bash
docker-compose down
docker-compose up -d
```

### Notes
- This configuration ensures PHPMyAdmin only attempts to store its settings on the primary server
- The warning message should no longer appear when accessing the replica
- The replica remains read-only; this only affects PHPMyAdmin's internal configuration storage
- If you prefer, you can also simply ignore the warning as it doesn't affect the functionality of the replica or the replication setup