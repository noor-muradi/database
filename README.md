# SQL Common commands

## A. POSTGRES SQL Commands

0. Connecting to database:
```
sudo apt install postgresql-client -y
psql -h <host> -U <username> -d <database>
```

### User management:

1. Check current databse and user:

```
SELECT current_database(), current_user;
```

2. create user:

```
CREATE USER app_user WITH PASSWORD 'strong_password_here';
```

3. grant db access

```
GRANT ALL PRIVILEGES ON DATABASE db_name TO app_user;
```

4. grant scheme access and table creation:

```
GRANT USAGE, CREATE ON SCHEMA public TO app_user;
```

5. Grant full access to all existing objects:

```
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO app_user;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO app_user;
```

6. Set default privilege for future objects:

```
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT ALL PRIVILEGES ON TABLES TO app_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT ALL PRIVILEGES ON SEQUENCES TO app_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT ALL PRIVILEGES ON FUNCTIONS TO app_user;

```

7. Change user password:
   
```
ALTER USER app_user WITH PASSWORD 'new_strong_password';
```

### DB Management

1. List Tables
   
```
\dt
or
SELECT tablename FROM pg_tables WHERE schemaname = current_schema();
```
2. Dump and restore database:

```
pg_dump -U app_user -d my_database > my_database.sql
psql -U app_user -d my_database < my_database.sql

```   

## B. MySQL Commands

0. Connecting to database

```
sudo apt install mysql-client -y
mysql -h <host> -u <username> -p <database>
```
### User Management
1. Check current database and user

```
SELECT DATABASE(), USER();
```
2. Create user

```
CREATE USER 'app_user'@'%' IDENTIFIED BY 'strong_password_here';
```
'%' means the user can connect from any host. Replace with 'localhost' if only local connections are allowed.

3. Grant database access
```

GRANT ALL PRIVILEGES ON db_name.* TO 'app_user'@'%';
```


4. Grant schema access and table creation

Already covered by the GRANT ALL PRIVILEGES ON db_name.* above.

If you want only creation rights:
```
GRANT CREATE ON db_name.* TO 'app_user'@'%';
```

5. Grant full access to all existing objects

Same as step 3. MySQL doesn’t separate sequences/functions the way Postgres does.

To cover everything:

```
GRANT ALL PRIVILEGES ON db_name.* TO 'app_user'@'%';
```

6. Set default privileges for future objects
MySQL doesn’t have ALTER DEFAULT PRIVILEGES.
Instead, privileges apply automatically to new objects in the database if you grant ALL PRIVILEGES ON db_name.*.

7. Change user password:

```
ALTER USER 'app_user'@'%' IDENTIFIED BY 'new_strong_password';
```

### DB Management

1. List tables
```
SHOW TABLES;
```
2. Dump and restore database:

```
mysqldump -h <host> -u <username> -p <database> > backup_file.sql
mysql -u app_user -p my_database < my_database.sql
```


