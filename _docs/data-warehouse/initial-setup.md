# Initial setup

1. Create `warehouse` database in PostgreSQL:

    ```sql
    CREATE DATABASE warehouse;
    ```

2. Create `readonly` warehouse role:

    ```sql
    CREATE ROLE readonly;

    -- Grant access to existing tables
    GRANT USAGE ON SCHEMA public TO readonly;
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

    -- Grant access to future tables
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;
    ```

3. Set up user for `metabase`:

    ```sql
    CREATE USER metabase WITH ENCRYPTED PASSWORD 'GENERATE_A_PASSWORD_AND_SAVE_IN_VAULT';
    GRANT readonly TO metabase;
    ```
