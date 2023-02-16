# Add a user

To add a new PostgreSQL user to the warehouse:

1. [Connect via the `admin` user](./README.md#access-via-psql)
2. Create a new user:

    ```sql
    CREATE USER myusername WITH ENCRYPTED PASSWORD 'generate_a_password';
    ```

3. Assign `readonly` role:

    ```sql
    GRANT readonly TO myusername;
    ```
