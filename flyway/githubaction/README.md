# Pre-requisites

1. Create flyway user in postgres
   
   ```
   CREATE USER flyway WITH PASSWORD 'abc123';
   ```
   
3. Change schema owner to flyway
   
```
ALTER SCHEMA myschema OWNER TO flyway;
```

3. change all existing tables ownership to flyway:
   
```
DO $$
DECLARE
    r RECORD;
BEGIN
    -- Tables
    FOR r IN
        SELECT schemaname, tablename
        FROM pg_tables
        WHERE schemaname = 'myschema'
    LOOP
        EXECUTE format(
            'ALTER TABLE %I.%I OWNER TO flyway',
            r.schemaname, r.tablename
        );
    END LOOP;

    -- Sequences
    FOR r IN
        SELECT sequence_schema, sequence_name
        FROM information_schema.sequences
        WHERE sequence_schema = 'myschema'
    LOOP
        EXECUTE format(
            'ALTER SEQUENCE %I.%I OWNER TO flyway',
            r.sequence_schema, r.sequence_name
        );
    END LOOP;

    -- Views
    FOR r IN
        SELECT table_schema, table_name
        FROM information_schema.views
        WHERE table_schema = 'myschema'
    LOOP
        EXECUTE format(
            'ALTER VIEW %I.%I OWNER TO flyway',
            r.table_schema, r.table_name
        );
    END LOOP;

    -- Functions
    FOR r IN
        SELECT n.nspname, p.proname, pg_get_function_identity_arguments(p.oid) args
        FROM pg_proc p
        JOIN pg_namespace n ON n.oid = p.pronamespace
        WHERE n.nspname = 'myschema'
    LOOP
        EXECUTE format(
            'ALTER FUNCTION %I.%I(%s) OWNER TO flyway',
            r.nspname, r.proname, r.args
        );
    END LOOP;
END$$;

```

4. change default privileges for future tables, (should be run by flyway user)

```
ALTER DEFAULT PRIVILEGES IN SCHEMA myschema
	GRANT SELECT ON TABLES TO readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA myschema
	GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO writer;

ALTER DEFAULT PRIVILEGES IN SCHEMA myschema
	GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO writer;

ALTER DEFAULT PRIVILEGES IN SCHEMA myschema
	GRANT ALL PRIVILEGES ON TABLES TO admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA myschema
	GRANT ALL PRIVILEGES ON SEQUENCES TO admin;

```

5. Create 3 environments (dev,qa,prod) in GitHub Repo and in each environment

create below 3 secrets:

```
DB_HOST
DB_PASSWORD
DB_USER
```

NOTE: This pipeline uses self-hosted runners as RDS are hosted in private subnets in VPC, and pipeline expects 2 runners one with `dev` label which will process `dev` and `qa` environment migrations, and one with `prod` label which will process production RDS migrations.
