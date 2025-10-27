# Setup MySQL in LINUX

### 1. Install MySQL Server

```bash
sudo apt install mysql-server -y
```

### 2. Start, Restart, and Check MySQL Service

* **Start MySQL server**

```bash
sudo systemctl start mysql
```

Purpose: Start the MySQL server if it is not running.

* **Restart MySQL server**

```bash
sudo systemctl restart mysql
```

Purpose: Restart the MySQL server to apply configuration changes or recover from errors.

* **Check MySQL server status**

```bash
sudo systemctl status mysql
```

Purpose: Verify if the MySQL server is active and running.

````

### 3. Login to MySQL
```bash
sudo mysql -u root -p
````

### 4. Set Root Password

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'java';
FLUSH PRIVILEGES;
EXIT;
```

### 5. Create Database

```sql
-- Create the database if not already exists
CREATE DATABASE IF NOT EXISTS crud_app;

-- Switch to the database
USE crud_app;
```

### 6. Create Table

```sql
-- Drop table if needed (optional safety cleanup)
-- DROP TABLE IF EXISTS users;

-- Create the `users` table with proper structure
CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  role ENUM('admin', 'viewer') NOT NULL DEFAULT 'viewer',
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
