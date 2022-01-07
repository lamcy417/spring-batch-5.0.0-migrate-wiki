This document is meant to help you migrate your application to Spring Batch 5.0.

**This document is a work in progress.**

# Major changes

## JDK 17 baseline

Spring Batch 5 is based on Spring Framework 6 which requires Java 17 as a minimum version. Therefore, you need to use Java 17+ to run Spring Batch 5 applications.

## Dependencies upgrade

Spring Batch 5 is updating its Spring dependencies across the board to the following versions:

* Spring Framework 6
* Spring Integration 6
* Spring Data 3
* Spring AMQP 3
* Spring for Apache Kafka 3
* Micrometer 2

Moreover, this version marks the migration to Jakarta EE 9. Please make sure to update your import statements from `javax.*` to `jakarta.*` for all EE APIs you use.

## Database schema updates

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

#### All platforms

The column `JOB_CONFIGURATION_LOCATION` in the `BATCH_JOB_EXECUTION` table is not used anymore and can be marked as unused or dropped if needed:

```
ALTER TABLE BATCH_JOB_EXECUTION DROP COLUMN JOB_CONFIGURATION_LOCATION;
```

The syntax to drop the column might differ depending on the version of your database server, so please check the syntax of column deletion. This change might require a table reorganisation on some platforms.

## Job repository/explorer configuration updates

The Map-based job repository/explorer implementation were deprecated in v4 and completely removed in v5. You should use the Jdbc-based implementation instead. Unless you are using a custom Job repository/explorer implementation, the `@EnableBatchProcessing` annotation will configure a Jdbc-based `JobRepository` which requires a `DataSource` bean in the application context. The `DataSource` bean could refer to an embedded database like H2, HSQL, etc to work with an in-memory job repository.

## Transaction manager bean exposure

Up until version 4.3, the `@EnableBatchProcessing` annotation exposed a tranasaction manager bean in the application
context. While this was convenient in many cases, the unconditional exposure of a tranasaction manager could
interfere with a user-defined transaction manager. In this release, `@EnableBatchProcessing` does not expose a
transaction manager bean in the application context anymore.

## Data types updates

* Metric counters (`readCount`, `writeCount`, etc) in `org.springframework.batch.core.StepExecution` and `org.springframework.batch.core.StepContribution` have been changed from `int` to `long`. All getters and setters have been updated accordingly.
* The `skipCount` parameter in `org.springframework.batch.core.step.skip.SkipPolicy#shouldSkip` has been changed from `int` to `long`. This is related to the previous point.

# Deprecated APIs

The following APIs have been deprecated in version 5.0:

* `org.springframework.batch.core.listener.ChunkListenerSupport`
* `org.springframework.batch.core.listener.StepExecutionListenerSupport`
* `org.springframework.batch.repeat.listener.RepeatListenerSupport`
* `org.springframework.batch.core.listener.JobExecutionListenerSupport`
* `org.springframework.batch.core.listener.SkipListenerSupport`
* `org.springframework.batch.item.data.Neo4jItemReader`
* `org.springframework.batch.item.data.builder.Neo4jItemWriterBuilder`
* `org.springframework.batch.item.data.Neo4jItemWriter`
* `org.springframework.batch.item.data.builder.Neo4jItemWriterBuilder`
* `org.springframework.batch.item.database.support.SqlPagingQueryUtils#generateLimitGroupedSqlQuery(org.springframework.batch.item.database.support.AbstractSqlPagingQueryProvider, boolean, java.lang.String)`
* `org.springframework.batch.core.repository.dao.JdbcJobInstanceDao#setJobIncrementer(DataFieldMaxValueIncrementer jobIncrementer)`

Please refer to the Javadoc of each API for more details about the suggested replacement.

# Removed APIs

The following APIs were deprecated in previous versions and have been removed in this release:

* Class `org.springframework.batch.core.repository.support.MapJobRepositoryFactoryBean`
* Class `org.springframework.batch.core.repository.dao.MapExecutionContextDao`
* Class `org.springframework.batch.core.repository.dao.MapJobExecutionDao`
* Class `org.springframework.batch.core.repository.dao.MapJobInstanceDao`
* Class `org.springframework.batch.core.repository.dao.MapStepExecutionDao`
* Class `org.springframework.batch.core.explore.support.MapJobExplorerFactoryBean`
* Class `org.springframework.batch.core.repository.dao.XStreamExecutionContextStringSerializer`
* Class `org.springframework.batch.core.configuration.support.ClassPathXmlJobRegistry`
* Class `org.springframework.batch.core.configuration.support.ClassPathXmlApplicationContextFactory`
* Class `org.springframework.batch.core.launch.support.ScheduledJobParametersFactory`
* Class `org.springframework.batch.item.data.AbstractNeo4jItemReader`
* Class `org.springframework.batch.item.database.support.ListPreparedStatementSetter`
* Class `org.springframework.batch.integration.chunk.RemoteChunkingMasterStepBuilder`
* Class `org.springframework.batch.integration.chunk.RemoteChunkingMasterStepBuilderFactory`
* Class `org.springframework.batch.integration.partition.RemotePartitioningMasterStepBuilder`
* Class `org.springframework.batch.integration.partition.RemotePartitioningMasterStepBuilderFactory`
* Class `org.springframework.batch.test.AbstractJobTests`
* Class `org.springframework.batch.item.xml.StaxUtils`
* Enum `org.springframework.batch.item.file.transform.Alignment`
* Method `org.springframework.batch.core.JobExecution#stop()`
* Method `org.springframework.batch.core.JobParameters#getDouble(String key, double defaultValue)`
* Method `org.springframework.batch.core.JobParameters#getLong(String key, long defaultValue)`
* Method `org.springframework.batch.core.partition.support.SimpleStepExecutionSplitter(JobRepository jobRepository, Step step, Partitioner partitioner)`
* Method `org.springframework.batch.core.partition.support.SimpleStepExecutionSplitter#getStartable(StepExecution stepExecution, ExecutionContext context)`
* Method `org.springframework.batch.core.repository.support.AbstractJobRepositoryFactoryBean#getJobRepository()`
* Method `org.springframework.batch.item.database.AbstractCursorItemReader#cleanupOnClose()`
* Method `org.springframework.batch.item.database.HibernateItemWriter#doWrite(HibernateOperations hibernateTemplate, List<? extends T> items)`
* Method `org.springframework.batch.item.database.JdbcCursorItemReader#cleanupOnClose()`
* Method `org.springframework.batch.item.database.StoredProcedureItemReader#cleanupOnClose()`
* Method `org.springframework.batch.item.database.builder.HibernatePagingItemReaderBuilder#useSatelessSession(boolean useStatelessSession)`
* Method `org.springframework.batch.item.file.MultiResourceItemReader#getCurrentResource()`
* Method `org.springframework.batch.integration.config.annotation.BatchIntegrationConfiguration#remoteChunkingMasterStepBuilderFactory()`
* Method `org.springframework.batch.integration.config.annotation.BatchIntegrationConfiguration#remotePartitioningMasterStepBuilderFactory()`
* Method `org.springframework.batch.item.util.FileUtils#setUpOutputFile(File file, boolean restarted, boolean overwriteOutputFile)`
