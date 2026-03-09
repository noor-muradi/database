## **General Best Practices (All DB Engines):**
- **Avoid row-at-a-time/app-level inserts:** Loading through `INSERT ... VALUES (...), ...` per row is very slow.
- **Use native database bulk tools**: They’re optimized for bulk throughput.
- **Run from an EC2 in the same VPC** if possible (minimizes network latency/bandwidth issues).
- **Consider disabling indexes/constraints during load** (if possible, and you can rebuild/validate them after).
- **Monitor I/O credits** if using burstable storage.

---

## **PostgreSQL**

### **1. Use `pg_dump` + `pg_restore`**

- **Export**:  
  ```bash
  pg_dump -h source-rds.endpoint -U username -Fc -t your_table dbname > table.dump
  ```
  (Use `-t` for a specific table, or leave off for full DB.)

- **Import**:  
  ```bash
  pg_restore -h target-rds.endpoint -U username -d dbname --single-transaction --no-owner table.dump
  ```

- **Best for:** Structure + data, very fast for large sets, handles all data types.

### **2. Use `COPY` (Fastest for Table Data Only)**

- On **source**:
  ```sql
  \COPY your_table TO '/tmp/your_table.csv' CSV
  ```
  (If on psql client with access.)

- Upload the CSV file to **a client machine** or S3.

- On **target**:
  ```sql
  \COPY your_table FROM '/tmp/your_table.csv' CSV
  ```
  or
  ```sql
  COPY your_table FROM 's3://yourbucket/.../your_table.csv' IAM_ROLE '...';
  ```

- **Super fast.** You can use S3 integration in RDS (especially with recent versions).

### **3. AWS DMS (Database Migration Service)**
- Fast, reliable for ongoing/multi-table/multi-DB migrations.
- Handles schema/data/type mapping, and can do ongoing CDC.
- Best for: very large migrations, production, or repeatable pipelines.

---

## **MySQL/MariaDB**

### **1. Use `mysqldump` and `mysql`**

- **Export:**
  ```bash
  mysqldump -h source-rds.endpoint -u username -p dbname your_table > table.sql
  ```
- **Import:**
  ```bash
  mysql -h target-rds.endpoint -u username -p dbname < table.sql
  ```

### **2. Use `SELECT INTO OUTFILE` and `LOAD DATA INFILE`**

- **On source:**
  ```sql
  SELECT * INTO OUTFILE '/tmp/your_table.csv' FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n' FROM your_table;
  ```
- **Transfer file** to step to target RDS’s import location (may need a jumpbox/EC2 instance).

- **On target:**
  ```sql
  LOAD DATA INFILE '/tmp/your_table.csv' INTO TABLE your_table FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n';
  ```

### **3. AWS DMS**

Same as above—great if migrating across accounts, many tables, or require minimal downtime.

---

## **Summary Table**

| **Method**        | **Speed**    | **Ease**           | **Schema Transfer** | **Best For**                          |
|-------------------|--------------|--------------------|---------------------|----------------------------------------|
| pg_dump/restore   | Fast         | Medium             | Yes                 | PostgreSQL, all data                  |
| COPY/LOAD DATA    | Very Fast    | Medium             | No (data only)      | Large tables, one-to-one schema        |
| AWS DMS           | Fast-High    | High (GUI/CLI)     | Yes                 | Cross-account, ongoing sync/many tables|
| App-level scripts | Slow         | Easy               | No                  | Small, simple, or rare ad hoc jobs     |

---

## **Recommendation (TL;DR):**
- For **one-off/large table**:  
  - Use **pg_dump/pg_restore** (Postgres)  
  - Use **mysqldump/mysql** (MySQL)  
  - For fastest data-only: Use **COPY** (Postgres) or **LOAD DATA INFILE** (MySQL)
- For **complex or repeated migrations**: Use **AWS DMS**
- **Run migration from an EC2 in the same AWS region/VPC** for highest throughput
