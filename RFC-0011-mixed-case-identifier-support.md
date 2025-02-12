# **RFC-0010 for Presto**

## Mixed case identifiers

Proposers

* Reetika Agrawal

## Summary

Improve Presto's identifier (schema, table & column names) handling to align with SQL standards, ensuring better interoperability with case-sensitive
and case-normalizing databases while minimizing SPI-breaking changes.

## Background

Presto treats all identifiers as case-insensitive, normalizing them to lowercase. This creates issues when
querying databases that are case-sensitive (e.g., MySQL, PostgreSQL) or case-normalizing to uppercase (e.g., Oracle,
DB2). Without a standard approach, identifiers might not match the actual names in the underlying data sources, leading
to unexpected query failures or incorrect results. Additionally, inconsistent handling of delimited and non-delimited
identifiers across different connectors further complicates cross-engine compatibility.

The goal here is to improve interoperability with storage engines by aligning identifier handling with SQL standards
while ensuring a seamless user experience. Ideally, the change should be implemented in a way that minimizes
breaking changes to the SPI, i.e. allowing connectors to adopt the new approach without significant.

### Goals

- Align Presto’s identifier handling with SQL standards to improve interoperability with case-sensitive and
  case-normalizing databases.
- Ensure consistent handling of delimited ("identifier") and non-delimited (identifier) identifiers across different connectors.
- Minimize SPI-breaking changes to maintain backward compatibility for existing connectors.
- Introduce a mechanism for connectors to define their own identifier normalization behavior.
- Allow identifiers to retain their original case where necessary, preventing unexpected query failures.
- Ensure Access Control SPI can correctly normalize identifiers.
- Preserve a seamless user experience while making these changes.

### Proposed Plan

Connectors handle identifiers in three ways:

1. SQL-Compliant (e.g., Oracle, DB2)
  - Delimited identifiers (e.g., "MyTable") keep their original case.
  - Non-delimited identifiers (e.g., mytable) are converted to uppercase.

2. Case-Sensitive (e.g., PostgreSQL)
  - Delimited identifiers keep their original case.
  - Non-delimited identifiers may be converted to lowercase or another case.

3. Case-Insensitive (e.g., Hive)
  - Delimited and non-delimited identifiers are treated the same.
  - Identifiers may be automatically converted to a specific case.

Presto's behavior includes:

- Delimited identifiers ("Identifier") and non-delimited identifiers (identifier) are converted to lowercase by default
  unless a connector enforces a specific behavior.
- Identifiers are normalized when:
  - Resolving schemas, tables, columns, views.
  - Retrieving metadata from connectors.
  - Displaying entity names in metadata introspection commands like SHOW TABLES and DESCRIBE.

Presto uses identifiers in several ways:

  - Matching identifiers to locate entities such as catalogs, schemas, tables, views.
  - Resolving column names based on table metadata provided by connectors.
  - Passing identifiers to connectors when creating new entities.
  - Processing and displaying entity names retrieved from connectors, including column resolution and metadata introspection commands like SHOW and DESCRIBE.

## Proposed Implementation

#### Core Changes

* In the presto-spi, add new API to pass original identifier (Schema, table and column names)
* Introduce new API in Metadata for preserving lower case identifier by default to preserve backward compatibility
* Introduce new Connector specific API in ConnectorMetadata

Metadata.java

```java
    String normalizeIdentifier(Session session, String catalogName, String identifier, boolean delimited);
```

MetadataManager.java

```java
    @Override
    public String normalizeIdentifier(Session session, String catalogName, String identifier, boolean delimited)
    {
        Optional<CatalogMetadata> catalogMetadata = getOptionalCatalogMetadata(session, transactionManager, catalogName);
        if (catalogMetadata.isPresent()) {
            ConnectorId connectorId = catalogMetadata.get().getConnectorId();
            ConnectorMetadata metadata = catalogMetadata.get().getMetadataFor(connectorId);
            return metadata.normalizeIdentifier(session.toConnectorSession(connectorId), identifier, delimited);
        }
        return identifier.toLowerCase(ENGLISH);
    }
```

ConnectorMetadata.java
```java

    /**
     * Normalize the provided SQL identifier according to connector-specific rules
    */
    default String normalizeIdentifier(ConnectorSession session, String identifier, boolean delimited)
    {
        return identifier.toLowerCase(ENGLISH);
    }
```

JDBC Connector specific implementation

JdbcMetadata.java

```java
    @Override
    public String normalizeIdentifier(ConnectorSession session, String identifier, boolean delimited)
    {
        return jdbcClient.normalizeIdentifier(session, identifier, delimited);
    }
```

JdbcClient.java
```java
    String normalizeIdentifier(ConnectorSession session, String identifier, boolean delimited);
```

BaseJdbcClient.java
```java
    @Override
    public String normalizeIdentifier(ConnectorSession session, String identifier, boolean delimited)
    {
        return identifier.toLowerCase(ENGLISH);
    }
```

Example - Connector specific implementation -
MySqlClient.java

```java
    @Override
    public String normalizeIdentifier(ConnectorSession session, String identifier, boolean delimited)
    {
        return identifier;
    }
```

#### Example Queries

#### MySQL Table Handling

```
presto> show schemas from mysql;
       Schema       
--------------------
 Test               
 TestDb             
 information_schema 
 performance_schema 
 sys                
 testdb             
(6 rows)

presto> show tables from mysql.TestDb;
   Table   
-----------
 Test      
 TestTable 
 testtable 
(3 rows)

presto> SHOW CREATE TABLE mysql.TestDb.Test;
             Create Table             
--------------------------------------
 CREATE TABLE mysql."TestDb"."Test" ( 
    "id" integer,                     
    "Name" char(10)                   
 )                                    
(1 row)

presto> select * from mysql.TestDb.Test;
 id |    Name    
----+------------
  2 | Tom        
(1 row)
```

#### PostgreSQL Behavior
Unlike MySQL, which uses backticks (`), Presto adheres to the SQL standard and does not support backticks for escaping identifiers.

```
presto> CREATE TABLE "MixedCaseTable" ("ID" INT, "UserName" TEXT);

presto> SELECT "ID", "UserName" FROM "MixedCaseTable";
```

Without double quotes, PostgreSQL would normalize unquoted identifiers to lowercase:
```
presto> SELECT ID, UserName FROM MixedCaseTable; // Error: table does not exist
```

## Backward Compatibility Considerations

* Existing connectors that do not implement normalizeIdentifier will default to lowercase normalization.
* Any connectors requiring case preservation can override the default behavior.
* A configuration flag could be introduced to allow backward-compatible identifier handling at the catalog level.

## Test Plan

* Ensure that existing CI tests pass for connectors where no specific implementation is added.
* Add unit tests for testing mixed-case identifiers support in a JDBC connector (e.g., MySQL, PostgreSQL).
* Cover cases such as:
  - Queries with mixed-case identifiers.
  - Queries with delimited and non-delimited identifiers.
  - Metadata retrieval commands (SHOW SCHEMAS, SHOW TABLES, DESCRIBE).
  - Joins, subqueries, and alias usage with mixed-case identifiers.

To ensure backward-compatibility current connectors where connector specific implementation is not added, existing CI tests should pass.
Add support for mixed case for a JDBC connector (ex. mysql, postgresql etc) and add relevant Unit tests for same.

## Modules involved
- `presto-main`
- `presto-common`
- `presto-spi`
- `presto-parser`
- `presto-base-jdbc`

## Final Thoughts

This RFC enhances Presto's identifier handling for improved cross-engine compatibility. The proposed changes ensure
better adherence to SQL standards while maintaining backward compatibility. Implementing connector-specific identifier
normalization will help prevent unexpected query failures and improve user experience when working with different
databases.
Would appreciate feedback on any additional cases or edge scenarios that should be covered!
