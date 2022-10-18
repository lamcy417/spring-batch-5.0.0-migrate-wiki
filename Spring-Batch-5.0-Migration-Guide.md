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
* `junit:junit` is no more a required dependency in `spring-batch-test`.
* `com.fasterxml.jackson.core:jackson-core` is now optional in `spring-batch-core`

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

##### Removal of `BATCH_JOB_EXECUTION#JOB_CONFIGURATION_LOCATION` column

The column `JOB_CONFIGURATION_LOCATION` in the `BATCH_JOB_EXECUTION` table is not used anymore and can be marked as unused or dropped if needed:

```
ALTER TABLE BATCH_JOB_EXECUTION DROP COLUMN JOB_CONFIGURATION_LOCATION;
```

The syntax to drop the column might differ depending on the version of your database server, so please check the syntax of column deletion. This change might require a table reorganisation on some platforms.

:exclamation: Important note :exclamation: This change is mainly related to the removal of the JSR-352 implementation which was the only part of the framework using this column. As a consequence, the field `JobExecution#jobConfigurationName` has been removed as well as all APIs using it (constructors and getter in the domain object `JobExecution`, method `JobRepository#createJobExecution(JobInstance, JobParameters, String);` in `JobRepository`).

##### Column change in `BATCH_JOB_EXECUTION_PARAMS`

The `BATCH_JOB_EXECUTION_PARAMS` has been updated as follows:

```diff
CREATE TABLE BATCH_JOB_EXECUTION_PARAMS  (
	JOB_EXECUTION_ID BIGINT NOT NULL ,
---	TYPE_CD VARCHAR(6) NOT NULL ,
---	KEY_NAME VARCHAR(100) NOT NULL ,
---	STRING_VAL VARCHAR(250) ,
---	DATE_VAL DATETIME(6) DEFAULT NULL ,
---	LONG_VAL BIGINT ,
---	DOUBLE_VAL DOUBLE PRECISION ,
+++	NAME VARCHAR(100) NOT NULL ,
+++	TYPE VARCHAR(100) NOT NULL ,
+++	VALUE VARCHAR(2500) ,
	IDENTIFYING CHAR(1) NOT NULL ,
	constraint JOB_EXEC_PARAMS_FK foreign key (JOB_EXECUTION_ID)
	references BATCH_JOB_EXECUTION(JOB_EXECUTION_ID)
);
```

This is related to the way job parameters are persisted as revisited in https://github.com/spring-projects/spring-batch/issues/3960. Migration scripts can be found in `org/springframework/batch/core/migration/5.0`.

## Infrastructure beans configuration with `@EnableBatchBatchProcessing`

### Job repository/explorer configuration updates

The Map-based job repository/explorer implementation were deprecated in v4 and completely removed in v5. You should use the Jdbc-based implementation instead. Unless you are using a custom Job repository/explorer implementation, the `@EnableBatchProcessing` annotation will configure a Jdbc-based `JobRepository` which requires a `DataSource` bean in the application context. The `DataSource` bean could refer to an embedded database like H2, HSQL, etc to work with an in-memory job repository.

### Transaction manager bean exposure/configuration

Up until version 4.3, the `@EnableBatchProcessing` annotation exposed a tranasaction manager bean in the application
context. While this was convenient in many cases, the unconditional exposure of a tranasaction manager could
interfere with a user-defined transaction manager. In this release, `@EnableBatchProcessing` does not expose a
transaction manager bean in the application context anymore. This change is related to issue https://github.com/spring-projects/spring-batch/issues/816.

As a result of the aforementioned issue, and in combination with the inconsistency between the XML and Java config styles regarding the transaction manager that was fixed in https://github.com/spring-projects/spring-batch/issues/4130, it is now required to manually configure the transaction manager on any tasklet step definition. The method `StepBuilderHelper#transactionManager(PlatformTransactionManager)` was moved down a level, to `AbstractTaskletStepBuilder`.

The typical migration path from v4 to v5 in that regard is as follows:

```
// Sample with v4
@Configuration
@EnableBatchProcessing
public class MyStepConfig {

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    public Step myStep() {
        return this.stepBuilderFactory.get("myStep")
                .tasklet(..) // or .chunk()
                .build();
    }

}
```

```
// Sample with v5
@Configuration
@EnableBatchProcessing
public class MyStepConfig {

    @Bean
    public Step myStep(JobRepository jobRepository, Tasklet myTasklet, PlatformTransactionManager transactionManager) {
        return new StepBuilder("myStep", jobRepository)
                .tasklet(myTasklet, transactionManager)
                .build();
    }

}
```

This is only required for tasklet steps, other step types do not require a transaction manager by design.

Moreover, the transaction manager was configurable by implementing `BatchConfigurer#getTransactionManager`. The transaction manager being an implementation detail of the `JobRepository`, it should not be configurable at the same level as the `JobRepository` (ie in the same interface). In this release, the `BatchConfigurer` interface was removed. If needed, a custom transaction manager could be supplied either declaratively as an attribute of `@EnableBatchProcessing`, or programmatically by overriding `DefaultBatchConfiguration#getTransactionManager()`. For more details about this change, please check https://github.com/spring-projects/spring-batch/issues/3942.

### JobBuilderFactory and StepBuilderFactory bean exposure/configuration

`JobBuilderFactory` and `StepBuilderFactory` are not exposed as beans in the application context anymore, and are now deprecated for removal in v5.2 in favor of using the respective builders they create.

The typical migration path from v4 to v5 in that regard is as follows:

```
// Sample with v4
@Configuration
@EnableBatchProcessing
public class MyJobConfig {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Bean
    public Job myJob(Step step) {
        return this.jobBuilderFactory.get("myJob")
                .start(step)
                .build();
    }

}
```

```
// Sample with v5
@Configuration
@EnableBatchProcessing
public class MyJobConfig {

    @Bean
    public Job myJob(JobRepository jobRepository, Step step) {
        return new JobBuilder("myJob", jobRepository)
                .start(step)
                .build();
    }

}
```

The same pattern can be used to remove the usage of the deprecated `StepBuilderFactory`. For more details about this change, please check https://github.com/spring-projects/spring-batch/issues/4188.

## Data types updates

* Metric counters (`readCount`, `writeCount`, etc) in `org.springframework.batch.core.StepExecution` and `org.springframework.batch.core.StepContribution` have been changed from `int` to `long`. All getters and setters have been updated accordingly.
* The `skipCount` parameter in `org.springframework.batch.core.step.skip.SkipPolicy#shouldSkip` has been changed from `int` to `long`. This is related to the previous point.
* The type of fields `startTime`, `endTime`, `createTime` and `lastUpdated` in `JobExecution` and `StepExecution` was changed from  `java.util.Date` to `java.time.LocalDateTime`.

## Observability updates

* Micrometer has been updated to version 1.10
* All tags are now prefixed with the meter name. For example, the tags of the timer `spring.batch.job` are named `name` and `status` in version 4.x. In version 5, those tags are now named `spring.batch.job.name` and `spring.batch.job.status` respectively.
* The `BatchMetrics` class (which is intended for internal use only) has been moved from `org.springframework.batch.core.metrics` to the `org.springframework.batch.core.observability` package.

## Execution context serialization updates

Starting from v5, the default `ExecutionContextSerializer` was changed from `JacksonExecutionContextStringSerializer` to `DefaultExecutionContextSerializer`. The default execution context serializer was updated to serialize/deserialize the context to/from Base64.

The dependency to Jackson was made optional. In order to use the `JacksonExecutionContextStringSerializer`, `jackson-core` should be added to the classpath.

## SystemCommandTasklet updates

The `SystemCommandTasklet` has been revisited in this release and was changed as follows:

* A new strategy interface named `CommandRunner` was introduced in order to decouple the command execution from the tasklet execution. The default implementation is the `JvmCommandRunner` which uses the `java.lang.Runtime#exec` API to run system commands. This interface can be implemented to use any other API to run system commands.

* The method that runs the command now accepts an array of `String`s representing the command and its arguments. There is no need anymore to tokenize the command or do any pre-processing. This change makes the API more intuitive, and less prone to errors.

## Job parameters handling updates

### Support for any type as a job parameter

This change adds support to use any type as a job parameter, and not only the 4 pre-defined types (long, double, string, date) as in v4. The main change in the following:

```diff
---public class JobParameter implements Serializable {
+++public class JobParameter<T> implements Serializable {

---   private Object parameter;
+++   private T value;

---   private ParameterType parameterType;
+++   private Class<T> type;

}
```

With this revision, the `JobParameter` can be of any type. This change required many API changes like changing the return type of `getType()`, or the removal of the `ParameterType` enum. All changes related to this update can be found in the "[Deprecated|Moved|Removed] APIs" sections.

This change had an impact on how job parameters are persisted in the database (There are no more 4 distinct columns for each predefined type). Please check [Column change in BATCH_JOB_EXECUTION_PARAMS](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-5.0-Migration-Guide#column-change-in-batch_job_execution_params) for DDL changes. The fully qualified name of the type of the parameter is now persisted as a `String`, as well as the parameter value. String literals are converted to the parameter type with the standard Spring conversion service. The standard conversion service can be enriched with any required converter to convert user specific types to and from String literals.

### Default job parameter conversion

The default notation of job parameters in v4 was specified as follows:

```
[+|-]parameterName(parameterType)=value
```

where `parameterType` is one of [string,long,double,date]. In addition to be limited and constraining, this notation caused several issues as described in https://github.com/spring-projects/spring-batch/issues/3960.

In v5, there are two way to specify job parameters:

#### Default notation

The default notation is specified as follows:

```
parameterName=parameterValue,parameterType,identificationFlag
```

where `parameterType` is the fully qualified name of the type of the parameter. Spring Batch provides the `DefaultJobParametersConverter` to support this notation.

#### Extended notation

While the default notation is well suited for the majority of use cases, it might be inconvenient when the value contains a comma for example. In this case, the extended notation can be used, which is inspired by Spring Boot's [Json Application Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.application-json) and specified as follows:

```
parameterName='{"value": "parameterValue", "type":"parameterType", "identifying": "booleanValue"}'
```

where `parameterType` is the fully qualified name of the type of the parameter. Spring Batch provides the `JsonJobParametersConverter` to support this notation.

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
* `org.springframework.batch.core.launch.support.SimpleJobLauncher`
* `org.springframework.batch.core.configuration.annotation.JobBuilderFactory`
* `org.springframework.batch.core.configuration.annotation.StepBuilderFactory`
* The constructor `org.springframework.batch.core.job.builder.JobBuilder#JobBuilder(java.lang.String)`
* The constructor `org.springframework.batch.core.step.builder.StepBuilder#StepBuilder(java.lang.String)`
* The method `org.springframework.batch.core.step.builder.StepBuilder#tasklet(org.springframework.batch.core.step.tasklet.Tasklet)`
* The method `org.springframework.batch.core.step.builder.StepBuilder#chunk(int)`
* The method `org.springframework.batch.core.step.builder.StepBuilder#chunk(org.springframework.batch.repeat.CompletionPolicy)`
* The method `org.springframework.batch.core.step.builder.TaskletStepBuilder#tasklet(org.springframework.batch.core.step.tasklet.Tasklet)`
* The constructor `org.springframework.batch.integration.chunk.RemoteChunkingManagerStepBuilder#RemoteChunkingManagerStepBuilder(java.lang.String)`
* The constructor `org.springframework.batch.integration.partition.RemotePartitioningManagerStepBuilder#RemotePartitioningManagerStepBuilder(java.lang.String)`
* The constructor `org.springframework.batch.integration.partition.RemotePartitioningWorkerStepBuilder#RemotePartitioningWorkerStepBuilder(java.lang.String)`
* The method `org.springframework.batch.integration.partition.RemotePartitioningWorkerStepBuilder#tasklet(org.springframework.batch.core.step.tasklet.Tasklet)`
* The method `org.springframework.batch.integration.partition.RemotePartitioningWorkerStepBuilder#chunk(int)`
* The method `org.springframework.batch.integration.partition.RemotePartitioningWorkerStepBuilder#chunk(org.springframework.batch.repeat.CompletionPolicy)`
* The method `org.springframework.batch.core.JobParameters#toProperties`
* The method `org.springframework.batch.core.JobParametersBuilder#addParameter`
* The class `org.springframework.batch.core.launch.support.JobRegistryBackgroundJobRunner`
* The class `org.springframework.batch.test.DataSourceInitializer`
* The class `org.springframework.batch.integration.step.DelegateStep`
* The annotation `org.springframework.batch.support.annotation.Classifier`
* The class `org.springframework.batch.item.ItemStreamSupport`
* The method `org.springframework.batch.core.step.builder.AbstractTaskletStepBuilder#throttleLimit`
* The method `org.springframework.batch.repeat.support.TaskExecutorRepeatTemplate#setThrottleLimit`
* The interface `org.springframework.batch.repeat.support.ResultHolder`
* The interface `org.springframework.batch.repeat.support.ResultQueue`
* The class `org.springframework.batch.repeat.support.ResultHolderResultQueue`
* The class `org.springframework.batch.repeat.support.ThrottleLimitResultQueue` 

Please refer to the Javadoc of each API for more details about the suggested replacement.

# Moved APIs

* The `BatchMetrics` class (which is intended for internal use only) has been moved from `org.springframework.batch.core.metrics` to the `org.springframework.batch.core.observability` package.
* The `Chunk` class was moved from the `org.springframework.batch.core.step.item` package (`spring-batch-core` module) to the `org.springframework.batch.item` package (`spring-batch-infrastrucutre` module)
* The `ScopeConfiguration` class has moved from `org.springframework.batch.core.configuration.annotation` to `org.springframework.batch.core.configuration.support`

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
* Enum `org.springframework.batch.core.JobParameter#ParameterType`
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

* The interface `org.springframework.batch.core.configuration.annotation.BatchConfigurer` was removed. See https://github.com/spring-projects/spring-batch/issues/3942
* The class `org.springframework.batch.core.configuration.annotation.DefaultBatchConfigurer` was removed. See https://github.com/spring-projects/spring-batch/issues/3942
* The method `org.springframework.batch.core.step.builder.SimpleStepBuilder#processor(Function)` has been removed to allow lambdas to be passed as item processors. See https://github.com/spring-projects/spring-batch/issues/4061 for more details about the rationale behind this removal. If you use an actual `Function`  as argument for `SimpleStepBuilder::processor`, then you can change that to `.processor(function::apply)` to migrate to v5.
* `org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder#resource`, `org.springframework.batch.item.file.ResourceAwareItemWriterItemStream#setResource`, `org.springframework.batch.item.json.builder.JsonFileItemWriterBuilder#resource`, `org.springframework.batch.item.json.JsonFileItemWriter#JsonFileItemWriter`, `org.springframework.batch.item.support.AbstractFileItemWriter#setResource`, `org.springframework.batch.item.xml.builder.StaxEventItemWriterBuilder#resource` and `org.springframework.batch.item.xml.StaxEventItemWriter#setResource` have been updated to accept a `org.springframework.core.io.WritableResource` instead of a `org.springframework.core.io.Resource`. For more details about this change, please check https://github.com/spring-projects/spring-batch/issues/756
* The static type `org.springframework.batch.item.data.builder.RepositoryItemReaderBuilder.RepositoryMethodReference` along with the method `org.springframework.batch.item.data.builder.RepositoryItemReaderBuilder#repository(RepositoryMethodReference<?>)` have been removed.
* The signature of the method `ItemWriter#write(List)` was changed to `ItemWriter#write(Chunk)`
* All implementations of `ItemWriter` were updated to use the `Chunk` API instead of `List`
* All methods in the `ItemWriteListener` interface were updated to use the `Chunk` API instead of `List`
* All implementations of `ItemWriteListener` were updated to use the `Chunk` API instead of `List`
* The constructor of `ChunkRequest` was changed to accept a `Chunk` instead of a `Collection` of items
* The return type of `ChunkRequest#getItems()` was changed from `List` to `Chunk`
* The `JobRepositoryTestUtils` was changed to work against the `JobRepository` interface without depending on a datasource bean. For this change, the constructor that takes a `DataSource` as a parameter (`JobRepositoryTestUtils(JobRepository jobRepository, DataSource dataSource)`) as well as the public `DataSource` setter were removed. This is related to issue #4070.
* The method `StepBuilderHelper#transactionManager(PlatformTransactionManager)` was moved to `AbstractTaskletStepBuilder`. This is related to issue https://github.com/spring-projects/spring-batch/issues/4130.
* The methods `RemotePartitioningManagerStepBuilder#transactionManager(PlatformTransactionManager)` and `RemotePartitioningWorkerStepBuilder#transactionManager(PlatformTransactionManager)` were removed. A transaction manager is not required for those type of steps. This is related to issue https://github.com/spring-projects/spring-batch/issues/4130.
* The method `JobParameter#getType` now returns `T` instead of `Object`
* The constructors in `JobParameter` that took the 4 pre-defined job parameter types (date, string, long, double) where removed.
* The constructor `SkipWrapper(Throwable e)` was removed
* The setter `setIsolationLevelForCreate(Isolation)` in `AbstractJobRepositoryFactoryBean` was renamed to `setIsolationLevelForCreateEnum`

# Pruning

## SQLFire support removal

SqlFire has been announced to be EOL as of November 1st, 2014. The support of SQLFire as a job repository
was deprecated in version 4.3 and removed in version 5.0.

## JSR-352 implementation removal

Due to a lack of adoption, the implementation of the JSR-352 has been discontinued in this release.

## Gemfire support removal

Based on the [decision to discontinue](https://github.com/spring-projects/spring-data-geode#notice
) the support of Spring Data for Apache Geode, the support for Geode in Spring Batch was removed. The code was moved to the [spring-batch-extensions](https://github.com/spring-projects/spring-batch-extensions) repository as a community-driven effort.
