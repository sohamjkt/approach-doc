# Approach Document: Global Neo4j Driver Management

## Background

The application previously created Neo4j driver instances in multiple places, often calling `neo4j_runner.close()` in `finally` blocks after queries. This led to several inconcistent issues under concurrent usage, including:

- `BufferError`
- `AttributeError`
- `IncompleteCommit`

The root cause was the lifecycle mismatch of driver instances: closing drivers while other parts of the application were still using them.

## Problem Analysis

### Major Errors
- **Defunct Connection**: This happens because the connection is no longer active but code still attempts to use it.
- **BufferError / IncompleteCommit**: Transactions in-flight were disrupted by premature driver closure.  
- **AttributeError**: Accessing attributes of a closed or non-existent driver instance.  

### Root Cause

1. **Multiple Driver Instances**  
   Every call to `GraphDatabaseFactory.get_database()` created a new `Neo4jDb` instance, each with its own driver. This caused a proliferation of connections under concurrent usage.

2. **Driver Lifecycle Mismatch**  
   Closing drivers in `finally` blocks created race conditions. Other modules still referencing the driver would encounter errors.

3. **Rapid Creation & Destruction**  
   Creating and closing drivers per query or per module is unsafe:
   - Neo4j uses internal connection pools.
   - Closing a driver while transactions are in-flight triggers buffer and commit errors.
   - Concurrent usage exacerbates race conditions.

## Solution: Global / Singleton Driver

### Implementation Changes

- **GraphDatabaseFactory Refactor**
  - Store the driver instance in a private class variable `_instance`.
  - Provide `get_database()` to return the global instance.
  - Provide `clear_instance()` or `.disconnect()` to close the driver only during application shutdown.
  
- **Application Lifecycle**
  - Initialize the global driver once at application startup.
  - Remove all other `.close()` calls from `finally` blocks.
  - Close the driver only in the shutdown event (`FastAPI @app.on_event("shutdown")`).

## Conclusion

- The global/singleton driver approach solves lifecycle-related errors and race conditions.
- Key principles:
  - Initialize once at startup.
  - Reuse across all modules.
  - Disconnect only at application shutdown.
