# Azure Synapse Analytic: _External Data Source_

## Database Scope Credential

A **Database Credential** is not mapped to a server login or database user. The
credential is used by the database to access to the external location anytime the
database is performing an operation that requires access.

### List Credentials

```sql
SELECT * FROM [sys].[database_scoped_credentials];
```

### Create Master Key

```sql
-- Optional: Create MASTER KEY if not exists in database:
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'P@ssW0rd'
GO
```

If the master key already exists on the database, you can use:

```
OPEN MASTER KEY DECRYPTION BY PASSWORD = 'P@ssW0rd';
...
CLOSE MASTER KEY;
```

### Create Credential

=== "Managed Identity"

    ```sql
    CREATE DATABASE SCOPED CREDENTIAL <credential-name>
    WITH IDENTITY = 'Managed Identity'
    GO

    CREATE EXTERNAL DATA SOURCE <external-data-source>
    WITH (
        LOCATION   = 'https://<storage_account>.dfs.core.windows.net/<container>/<path>',
        CREDENTIAL = <credential-name>
    )
    ```

=== "Service Principle"

    ```sql
    -- authority-url: `https://login.microsoftonline.com/<tenant-id>/oauth2/token`
    CREATE DATABASE SCOPED CREDENTIAL <credential-name>
    WITH IDENTITY = '<client-id>@<authority-url>',
        SECRET = '<client-secret>'
    GO

    CREATE EXTERNAL DATA SOURCE <external-data-source>
    WITH (
        LOCATION   = 'https://<storage_account>.dfs.core.windows.net/<container>/<path>',
        CREDENTIAL = <credential-name>
    )
    ```

=== "Shared Access Signature"

    ```sql
    -- The secret value must remove the leading '?'
    CREATE DATABASE SCOPED CREDENTIAL <credential-name>
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
        SECRET = 'sv=2018-03-28&ss=bfqt&...&sig=lQHczN...'
    GO

    CREATE EXTERNAL DATA SOURCE <external-data-source>
    WITH (
        LOCATION   = 'https://<storage_account>.dfs.core.windows.net/<container>/<path>',
        CREDENTIAL = <credential-name>
    )
    ```

And the permission of User, Managed Identity, or Service Principle that want to
access data on target external data source should be any role in `Storage Blob Data Owner/Contributor/Reader`
roles in order for the application to access the data.

!!! example

    ```sql
    CREATE LOGIN <username> WITH PASSWORD = 'P@ssW0rd';
    GO

    GRANT REFERENCES ON DATABASE SCOPED CREDENTIAL::<credential-name> TO <username>;
    GO
    ```

    ```sql
    IF NOT EXISTS (
        SELECT *
        FROM [sys].[external_data_sources]
        WHERE [name] = '<external-data-source-name>'
    )
        CREATE EXTERNAL DATA SOURCE <external-data-source-name>
        WITH (
            CREDENTIAL = <credential-name>,
            LOCATION = 'abfss://<container>@<storage-account>.dfs.core.windows.net'
        )
    GO
    ```

    ```sql
    CREATE OR ALTER VIEW [CURATED].[<view-name>]
    AS
        SELECT *
        FROM OPENROWSET(
            BULK '/delta_silver/<delta-table-name>',
            DATA_SOURCE = '<external-data-source-name>',
            FORMAT = 'DELTA'
    ) AS [r]
    GO

    GRANT SELECT ON OBJECT::[DEVDWHCURATED].[VW_DELTA_DIM_SALE_ADB] TO adbuser
    GO
    ```

## External Data Source

### List Data Source

```sql
SELECT * FROM [sys].[external_data_sources]
;
```

### Create Data Source

=== "Dedicate SQL Pool"

    ```sql
    CREATE EXTERNAL DATA SOURCE [<external-data-source>]
    WITH(
        LOCATION = 'abfss://<container>@<storage-account>.dfs.core.windows.net',
        CREDENTIAL = <credential-name>,
        TYPE = HADOOP
    );
    ```

    !!! note

        **PolyBase** data virtualization is used when the `EXTERNAL DATA SOURCE`
        is created with `TYPE=HADOOP`.

=== "Serverless SQL Pool"

    ```sql
    CREATE EXTERNAL DATA SOURCE [<external-data-source>]
    WITH(
        LOCATION = 'abfss://<container>@<storage-account>.dfs.core.windows.net',
        CREDENTIAL = <credential-name>
    );
    ```

!!! note

    If you want to use **Azure AD** for access an external data source you can use:

    ```sql
    -- The Permission from this solution will up to user that want to access
    -- target external data source.
    CREATE EXTERNAL DATA SOURCE [<external-data-source>]
    WITH (
        LOCATION  = 'https://<storage_account>.dfs.core.windows.net/<container>/<path>'
    )
    ```

## External File Format

### Create File Format

```sql
CREATE EXTERNAL FILE FORMAT <parquet_snappy>
WITH (
    FORMAT_TYPE = PARQUET,
    DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'
);
```

```sql
CREATE EXTERNAL FILE FORMAT <skip_header_csv>
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS(
        FIELD_TERMINATOR    = ',',
        STRING_DELIMITER    = '"',
        FIRST_ROW           = 2,
        USE_TYPE_DEFAULT    = True
    )
);
```

## Examples

### Copy data from Azure Blob to Table

* [Loading Data in Azure Synapse using Copy](https://www.sqlservercentral.com/articles/loading-data-in-azure-synapse-using-copy)

## References

* [Microsoft: Develop Storage Files Access Control](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql/develop-storage-files-storage-access-control?tabs=user-identity)
* [Microsoft: TSQL - Create External Data Source](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-data-source-transact-sql?view=azure-sqldw-latest&preserve-view=true&tabs=dedicated)
* [Microsoft: TSQL - Create External File Format](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-file-format-transact-sql)