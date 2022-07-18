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
* Micrometer 1.10

Moreover, this version marks the migration to:

* Jakarta EE 9: Please make sure to update your import statements from `javax.*` to `jakarta.*` for all EE APIs you use.
* Hibernate 6: Hibernate (cursor/paging) item readers and writer have been updated to use Hibernate 6.1 APIs (previously using Hibernate 5.6 APIs)

In addition to that:

* `org.springframework:spring-jdbc` is now a required dependency in `spring-batch-core`
* `junit:junit` (junit 4) is now an optional dependency in `spring-batch-test`. If you use `org.springframework.batch.test.AssertFile`, you will need to manually add the `junit:junit` dependency to your test classpath.

## Database schema updates

#### Oracle

In this version, Oracle sequences are now ordered. The sequences creation script has been updated for new applications. Existing applications can use the migration script in `org/springframework/batch/core/migration/5.0/migration-oracle.sql` to alter the existing sequences.

Moreover, the DDL scripts for Oracle have been renamed as follows:
* `org/springframework/batch/core/schema-drop-oracle10g.sql` has been renamed to `org/springframework/batch/core/schema-drop-oracle.sql`
* `org/springframework/batch/core/schema-oracle10g.sql` has been renamed to `org/springframework/batch/core/schema-oracle.sql`

#### MS SQLServer

Up until v4, the DDL script for MS SQLServer used tables to emulate sequences. In this version, this usage has been updated with real sequences:

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

:exclamation: Important note :exclamation: This change is mainly related to the removal of the JSR-352 implementation which was the only part of the framework using this column. As a consequence, the field `JobExecution#jobConfigurationName` has been removed as well as all APIs using it (constructors and getter in the domain object `JobExecution`, method `JobRepository#createJobExecution(JobInstance, JobParameters, String);` in `JobRepository`).

## Infrastructure beans configuration with `@EnableBatchBatchProcessing`

### Job repository/explorer configuration updates

The Map-based job repository/explorer implementation were deprecated in v4 and completely removed in v5. You should use the Jdbc-based implementation instead. Unless you are using a custom Job repository/explorer implementation, the `@EnableBatchProcessing` annotation will configure a Jdbc-based `JobRepository` which requires a `DataSource` bean in the application context. The `DataSource` bean could refer to an embedded database like H2, HSQL, etc to work with an in-memory job repository.

### Transaction manager bean exposure

Up until version 4.3, the `@EnableBatchProcessing` annotation exposed a tranasaction manager bean in the application
context. While this was convenient in many cases, the unconditional exposure of a tranasaction manager could
interfere with a user-defined transaction manager. In this release, `@EnableBatchProcessing` does not expose a
transaction manager bean in the application context anymore.

### Default transaction manager type

When no transaction manager is specified, `@EnableBatchProcessing` used (up to version 4.3) to register a default transaction manager of type `org.springframework.jdbc.datasource.DataSourceTransactionManager` in the proxy around `JobRepository` when a `DataSource` bean is registered in the application context. In this release, the type of the default transaction manager has changed to `org.springframework.jdbc.support.JdbcTransactionManage`.

## Data types updates

* Metric counters (`readCount`, `writeCount`, etc) in `org.springframework.batch.core.StepExecution` and `org.springframework.batch.core.StepContribution` have been changed from `int` to `long`. All getters and setters have been updated accordingly.
* The `skipCount` parameter in `org.springframework.batch.core.step.skip.SkipPolicy#shouldSkip` has been changed from `int` to `long`. This is related to the previous point.

## Observability updates

* Micrometer has been updated to version 1.10
* All tags are now prefixed with the meter name. For example, the tags of the timer `spring.batch.job` are named `name` and `status` in version 4.x. In version 5, those tags are now named `spring.batch.job.name` and `spring.batch.job.status` respectively.
* The `BatchMetrics` class (which is intended for internal use only) has been moved from `org.springframework.batch.core.metrics` to the `org.springframework.batch.core.observability` package.

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

# Moved APIs

* The `BatchMetrics` class (which is intended for internal use only) has been moved from `org.springframework.batch.core.metrics` to the `org.springframework.batch.core.observability` package.

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

Moreover, the following APIs have been removed/updated without deprecation:

* The default constructor in `org.springframework.batch.core.configuration.annotation.DefaultBatchConfigurer` has been removed. Use the constructor that accepts a `DataSource` instead, see https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-5.0-Migration-Guide#job-repositoryexplorer-configuration-updates.
* The method `org.springframework.batch.core.step.builder.SimpleStepBuilder#processor(Function)` has been removed to allow lambdas to be passed as item processors. See https://github.com/spring-projects/spring-batch/issues/4061 for more details about the rationale behind this removal. If you use an actual `Function`  as argument for `SimpleStepBuilder::processor`, then you can change that to `.processor(function::apply)` to migrate to v5.
* `org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder#resource`, `org.springframework.batch.item.file.ResourceAwareItemWriterItemStream#setResource`, `org.springframework.batch.item.json.builder.JsonFileItemWriterBuilder#resource`, `org.springframework.batch.item.json.JsonFileItemWriter#JsonFileItemWriter`, `org.springframework.batch.item.support.AbstractFileItemWriter#setResource`, `org.springframework.batch.item.xml.builder.StaxEventItemWriterBuilder#resource` and `org.springframework.batch.item.xml.StaxEventItemWriter#setResource` have been updated to accept a `org.springframework.core.io.WritableResource` instead of a `org.springframework.core.io.Resource`. For more details about this change, please check https://github.com/spring-projects/spring-batch/issues/756
* The static type `org.springframework.batch.item.data.builder.RepositoryItemReaderBuilder.RepositoryMethodReference` along with the method `org.springframework.batch.item.data.builder.RepositoryItemReaderBuilder#repository(RepositoryMethodReference<?>)` have been removed.

# Pruning

## SQLFire support removal

SqlFire has been announced to be EOL as of November 1st, 2014. The support of SQLFire as a job repository
was deprecated in version 4.3 and removed in version 5.0.

## JSR-352 implementation removal

Due to a lack of adoption, the implementation of the JSR-352 has been discontinued in this release.
