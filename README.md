# 📘 SQL Common Commands

This guide provides a quick reference for **PostgreSQL** and **MySQL** database management, including user/role setup, permissions, and common administrative tasks.

---

## 🐘 A. PostgreSQL Commands

### 0. Connecting to Database
Install postgres client:

```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg

sudo install -d /usr/share/postgresql-common/pgdg
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc \
  | sudo gpg --dearmor -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg

  echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
| sudo tee /etc/apt/sources.list.d/pgdg.list

sudo apt update
sudo apt install postgresql-client-17
```
Connect:
```
psql -h <host> -U <username> -d <database>
```

---

### 👤 User Management

#### 1. Check Current Database and User
```sql
SELECT current_database(), current_user;
```

---

### 🔐 Role & Permission Setup

#### Create Roles
```sql
CREATE ROLE readonly NOLOGIN;
CREATE ROLE writer   NOLOGIN;
CREATE ROLE admin    NOLOGIN;
```

- **readonly** → Query-only access  
- **writer** → Read + write access (insert, update, delete)  
- **admin** → Full privileges including database management  

---

#### Grant Permissions

**Readonly Role**
```sql
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO readonly;
```

**Writer Role**
```sql
GRANT readonly TO writer;
GRANT USAGE, CREATE ON SCHEMA public TO writer;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO writer;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA public TO writer;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO writer;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO writer;
```

**Admin Role**
```sql
GRANT ALL PRIVILEGES ON DATABASE demo_db TO admin;
GRANT ALL PRIVILEGES ON SCHEMA public TO admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT ALL PRIVILEGES ON TABLES TO admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT ALL PRIVILEGES ON SEQUENCES TO admin;

ALTER ROLE admin CREATEDB;
```

---

#### Create Users
```sql
CREATE USER writer_user WITH PASSWORD 'password';
CREATE USER read_user   WITH PASSWORD 'password';
CREATE USER admin_user  WITH PASSWORD 'password';
```

---

#### Assign Roles to Users
```sql
GRANT readonly TO read_user;
GRANT writer   TO writer_user;
GRANT admin    TO admin_user;
```

---

### ⚙️ Database Management

**List All Tables in Current Schema**
```sql
SELECT tablename 
FROM pg_tables 
WHERE schemaname = current_schema();
```

**Dump & Restore Database**
```bash
pg_dump -h <db_endpoint> -U user -t <schema>.<table> <database> > table_backup.sql

psql -h <db_endpoint> -U user -d <database> -f table_backup.sql
```

**Lock Relation break**

Find the query:

```
SELECT
  pid,
  usename,
  state,
  wait_event_type,
  wait_event,
  now() - xact_start AS xact_age,
  query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_age DESC;
```

Terminate it:

```
SELECT pg_terminate_backend(<pid>);
```


---

## 🐬 B. MySQL Commands

### 0. Connecting to Database
```bash
sudo apt install mysql-client -y
mysql -h <host> -u <username> -p <database>
```

---

### 👤 User Management

#### 1. Check Current Database and User
```sql
SELECT DATABASE(), USER();
```

#### 2. Create User
```sql
CREATE USER 'app_user'@'%' IDENTIFIED BY 'strong_password_here';
```
- `'%'` → Allows connection from any host  
- Use `'localhost'` for local-only connections  

#### 3. Grant Database Access
```sql
GRANT ALL PRIVILEGES ON db_name.* TO 'app_user'@'%';
```

#### 4. Grant Schema Access & Table Creation
Already covered by `GRANT ALL PRIVILEGES`.  
For **creation-only rights**:
```sql
GRANT CREATE ON db_name.* TO 'app_user'@'%';
```

#### 5. Grant Full Access to All Existing Objects
```sql
GRANT ALL PRIVILEGES ON db_name.* TO 'app_user'@'%';
```

#### 6. Default Privileges for Future Objects
- MySQL does **not** support `ALTER DEFAULT PRIVILEGES`.  
- Granting `ALL PRIVILEGES ON db_name.*` ensures new objects are automatically accessible.

#### 7. Change User Password
```sql
ALTER USER 'app_user'@'%' IDENTIFIED BY 'new_strong_password';
```
---

### 🔐 Role Management

1. Create roles

```
CREATE ROLE 'readonly';
CREATE ROLE 'writer';
CREATE ROLE 'admin';
```

2. Grant permission to the roles
   
   A. readonly
```
GRANT SELECT ON db.* TO 'readonly';
```
  B. writer
```
GRANT 'readonly' TO 'writer';

GRANT SELECT, INSERT, UPDATE, DELETE
ON db.*
TO 'writer';
```
C. admin

```
GRANT ALL PRIVILEGES ON db.* to admin
```

3. Assign roles to users:

```
CREATE USER 'app_user'@'%' IDENTIFIED BY 'password';

GRANT 'writer' TO 'app_user'@'%';
SET DEFAULT ROLE 'writer' TO 'app_user'@'%';

```

---

### ⚙️ Database Management

**List Tables**
```sql
SHOW TABLES;
```

**Dump & Restore Database**
```bash
mysqldump -h <host> -u <username> -p <database> > backup_file.sql
mysql -u app_user -p my_database < my_database.sql
```

---

✨ **Best Practices**
- Always use **strong passwords** and avoid hardcoding them in scripts.  
- Use **environment variables** or secret managers for credentials.  
- Apply the **principle of least privilege**: grant only the permissions necessary.  
- Regularly back up databases and test restores to ensure reliability.  

---
