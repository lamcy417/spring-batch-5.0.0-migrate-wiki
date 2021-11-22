This document is meant to help you migrate your application to Spring Batch 5.0.

**This document is a work in progress.**

# Major changes

## JDK 17 baseline

Spring Batch 5 is based on Spring Framework 6 which requires Java 17 as a minimum version.So you need to use Java 17+ to run Spring Batch 5 applications.

## Dependencies upgrade

TBD

## DDL scripts updates

#### Oracle

In this version, Oracle sequences are now ordered. The sequences creation script has been updated for new applications. Existing applications can use the migration script in `org/springframework/batch/core/migration/5.0/migration-oracle.sql` to alter the existing sequences.

#### MS SQLServer

Up until v4, the DDL script for MS SQLServer used tables to emulates sequences. In this version, this usage has been updated with real sequences:

```sql
CREATE SEQUENCE BATCH_STEP_EXECUTION_SEQ START WITH 0 MINVALUE 0 MAXVALUE 9223372036854775807 NO CACHE NO CYCLE;
CREATE SEQUENCE BATCH_JOB_EXECUTION_SEQ START WITH 0 MINVALUE 0 MAXVALUE 9223372036854775807 NO CACHE NO CYCLE;
CREATE SEQUENCE BATCH_JOB_SEQ START WITH 0 MINVALUE 0 MAXVALUE 9223372036854775807 NO CACHE NO CYCLE;
```

New applications can use the provided script with no modifications. Existing applications should consider modifying the snippet above to start sequences from the last value in sequence tables used with v4.

// TODO see if this can be automated in a script

## Job repository/explorer configuration updates

The Map-based job repository/explorer implementation were deprecated in v4 and completely removed in v5. You should use the Jdbc-based implementation instead. Unless you are using a custom Job repository/explorer implementation, the `@EnableBatchProcessing` annotation will configure a Jdbc-based `JobRepository` which requires a `DataSource` bean in the application context.

## Data types updates

* Metric counters (`readCount`, `writeCount`, etc) in `org.springframework.batch.core.StepExecution` and `org.springframework.batch.core.StepContribution` have been changed from `int` to `long`. All getters and setters have been updated accordingly.
* The `skipCount` parameter in `org.springframework.batch.core.step.skip.SkipPolicy#shouldSkip` has been changed from `int` to `long`. This is related to the previous point.

# Deprecated APIs

TBD

# Removed APIs

TBD