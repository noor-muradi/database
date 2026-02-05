
# change schema owner to flyway

ALTER SCHEMA schema_v2 OWNER TO flyway;

# change all existing tables ownership to flyway:

DO $$
DECLARE
    r RECORD;
BEGIN
    -- Tables
    FOR r IN
        SELECT schemaname, tablename
        FROM pg_tables
        WHERE schemaname = 'schema_v2'
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
        WHERE sequence_schema = 'schema_v2'
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
        WHERE table_schema = 'schema_v2'
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
        WHERE n.nspname = 'schema_v2'
    LOOP
        EXECUTE format(
            'ALTER FUNCTION %I.%I(%s) OWNER TO flyway',
            r.nspname, r.proname, r.args
        );
    END LOOP;
END$$;



# change default privileges for future tables, should be run by flyway

ALTER DEFAULT PRIVILEGES IN SCHEMA schema_v2
	GRANT SELECT ON TABLES TO readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA schema_v2
	GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO writer;

ALTER DEFAULT PRIVILEGES IN SCHEMA schema_v2
	GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO writer;


ALTER DEFAULT PRIVILEGES IN SCHEMA schema_v2
	GRANT ALL PRIVILEGES ON TABLES TO admin;

ALTER DEFAULT PRIVILEGES IN SCHEMA schema_v2
	GRANT ALL PRIVILEGES ON SEQUENCES TO admin;
