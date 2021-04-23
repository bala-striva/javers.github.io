---
layout: page
title: Spring integration
category: Documentation
submenu: spring-integration
sidebar-url: docs-sidebar.html
---

We made JaVers easy to use for applications based on Spring Framework.
There are two modules for Spring integration.

**`javers-spring`** module provides two auto-audit aspects:

* [@JaversAuditable](#at-javers-auditable) for any kind of repositories,
* [@JaversSpringDataAuditable](#at-javers-spring-data-auditable) for Spring Data `CrudRepositories`.
   
**`javers-spring-jpa`** module &mdash; a superset of `javers-spring` &mdash; provides:

* [JPA & Hibernate integration](#jpa-entity-manager-integration) for SQL databases,
* support for Spring Data `JpaRepository.saveAndFlush()`.

### Dependencies ###

If you are using [Spring Boot](http://projects.spring.io/spring-boot/),
simplify your [JaVers setup with our Spring Boot starters](/documentation/spring-boot-integration/). This is
**the recommended way** of using JaVers in Spring-based applications.
Our starters give you the right configuration out of the box.


#### Non Spring Boot applications

Take `javers-spring-jpa` if you are using JaVers with SQL database and JPA & Hibernate:

```groovy
compile 'org.javers:javers-spring-jpa:{{site.javers_current_version}}'
```

Take `javers-spring` if you are using JaVers with MongoDB (or any persistence stack other than SQL with JPA & Hibernate):
 
```groovy
compile 'org.javers:javers-spring:{{site.javers_current_version}}'
```

Check
[Maven Central](https://search.maven.org/#artifactdetails|org.javers|javers-spring|{{site.javers_current_version}}|jar)
for other build tools snippets.

<h2 id="auto-audit-aspect">Auto-audit aspect</h2>
The auto-audit aspect is based on Spring AOP. It can automatically call
proper `javers.commit(...)` methods whenever your domain object are saved or deleted.

There are two variants of the auto-audit aspect:

* **`JaversSpringDataAuditableRepositoryAspect`** and 
  **`JaversSpringDataJpaAuditableRepositoryAspect`** for Spring Data `CrudRepositories`
  annotated with [@JaversSpringDataAuditable](#at-javers-spring-data-auditable).
  
* **`JaversAuditableAspect`** does not depend on Spring Data and can be used for any kind of repository.
  Use [@JaversAuditable](#at-javers-auditable) to mark all data-changing methods.

JaVers can audit your data changes automatically — AWESOME!

Below you can see which beans you need to register to use the auto-audit feature.

<h3 id="javers-instance-as-a-bean">JaVers instance as a Spring bean</h3>

You need to register exactly one JaVers instance in your Application Context.
For example, if you’re using MongoDB, setup JaVers as follows:

```java
@Bean
public Javers javers() {
    MongoRepository javersMongoRepository =
            new MongoRepository(mongo().getDatabase("mydatabase"));

    return JaversBuilder.javers()
            .registerJaversRepository(javersMongoRepository)
            .build();
}

@Bean
public MongoClient mongo() {
    return new MongoClient();
}
```

<h3 id="javers-auto-audit-aspects">Aspect beans</h3>

There are three beans that you can add to Context to enable the auto-audit feature:

**`JaversSpringDataAuditableRepositoryAspect`**
  for Spring Data `CrudRepositories`.
  <br/>
  It defines the pointcut on `save(..)` and `delete(..)` methods
  within `CrudRepositories` annotated with the class-level
  [@JaversSpringDataAuditable](#at-javers-spring-data-auditable) annotation.
  Choose it if you **are not using** JPA.

```java
@Bean
public JaversSpringDataAuditableRepositoryAspect javersSpringDataAuditableAspect() {
    return new JaversSpringDataAuditableRepositoryAspect(
            javers(), authorProvider(), commitPropertiesProvider());
}
```

**`JaversSpringDataJpaAuditableRepositoryAspect`** 
  for Spring Data `JpaRepositories`.
  <br/>
  It extends the first aspect with the pointcut on `JpaRepository.saveAndFlush()`
  and it is triggered by the same annotation &mdash; [@JaversSpringDataAuditable](#at-javers-spring-data-auditable).
  Choose it if you **are using** JPA.
  
```java
@Bean
public JaversSpringDataJpaAuditableRepositoryAspect javersSpringDataAuditableAspect(Javers javers) {
    return new JaversSpringDataJpaAuditableRepositoryAspect(javers, authorProvider(),
        commitPropertiesProvider());
}
```  
 
**`JaversAuditableAspect`**
  for any kind of repository.
  <br/>
  It defines the pointcut on any method annotated with the method-level
  [@JaversAuditable](#at-javers-auditable) annotation.
  Choose it if you have repositories that are not managed by Spring Data.

```java
@Bean
public JaversAuditableAspect javersAuditableAspect() {
    return new JaversAuditableAspect(javers(), authorProvider(), commitPropertiesProvider());
}
```

### How auto-audit aspects work?

After an advised method is executed, all of its **arguments**
are automatically saved to JaversRepository. <br/>
In the case where an argument is an `Iterable` instance,
JaVers iterates over it and saves each element separately.

Aspects require one more bean &mdash; [`AuthorProvider`](#author-provider-bean)
and optionally [`CommitPropertiesProvider`](#commit-properties-provider-bean) bean.

Note that these aspects are based on Spring `@AspectJ`.<br/>
**Remember to enable** `@AspectJ` support by putting the `@EnableAspectJAutoProxy`
annotation in your Spring configuration.

For more info refer to 
[Spring @AspectJ documentation](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html#aop-ataspectj).

<h3 id="author-provider-bean">AuthorProvider bean</h3>

Every JaVers commit (a data change) should be connected to its author, i.e. the
user who made the change.
Please don’t confuse JaVers commit (a bunch of data changes)
with the SQL commit command (finalizing an SQL transaction).

You need to register an implementation of the
[`AuthorProvider`](https://github.com/javers/javers/blob/master/javers-spring/src/main/java/org/javers/spring/auditable/AuthorProvider.java) interface,
which should return a current user name. It’s required by auto-audit aspects. 
For example:

```java
    @Bean
    public AuthorProvider authorProvider() {
        return new SpringSecurityAuthorProvider();
    }
```

JaVers comes with [`SpringSecurityAuthorProvider`](https://github.com/javers/javers/blob/master/javers-spring/src/main/java/org/javers/spring/auditable/SpringSecurityAuthorProvider.java)
&mdash;
suitable implementation when using Spring Security.
If you don’t care about commit author, use
[`MockAuthorProvider`](https://github.com/javers/javers/blob/master/javers-spring/src/main/java/org/javers/spring/auditable/MockAuthorProvider.java).

<h3 id="commit-properties-provider-bean">CommitPropertiesProvider bean</h3>
Every JaVers commit may have one or more commit properties, useful for querying
(see [CommitProperty filter example](/documentation/jql-examples/#commit-property-filter)).

In auto-audit aspect, commit properties are supplied by an implementation of the
[`CommitPropertiesProvider`](https://github.com/javers/javers/blob/master/javers-spring/src/main/java/org/javers/spring/auditable/CommitPropertiesProvider.java) interface,
for example:

```java
    @Bean
    public CommitPropertiesProvider commitPropertiesProvider() {
        return new CommitPropertiesProvider() {
            @Override
            public Map<String, String> provideForCommittedObject(Object domainObject) {
                if (domainObject instanceof DummyEntity) {
                    return Maps.of("dummyEntityId", ((DummyEntity)domainObject).getId() + "");
                }
                return Collections.emptyMap();
            }
        };
    }
```

If you don’t use commit properties, simply skip `commitPropertiesProvider` argument
in the aspect constructor or pass `new EmptyPropertiesProvider()`.

That’s the last bean in your Application Context required to run auto-audit aspects.
See the full Spring configuration examples for [MongoDB](#auto-audit-example-mongo) and
for [JPA & Hibernate](#spring-jpa-example)

<h3 id="at-javers-spring-data-auditable">@JaversSpringDataAuditable for Spring Data Repositories</h3>

If you’re using Spring Data, just annotate Repositories you want to audit
with the class-level `@JaversSpringDataAuditable`.<br/>

For example:

```java
import org.javers.spring.data.JaversSpringDataAuditable
import org.springframework.data.repository.CrudRepository
import org.springframework.stereotype.Repository

@Repository
@JaversSpringDataAuditable
interface UserRepository extends CrudRepository<User, String> {
}
```

From now, all objects passed to `save()` and `delete()` methods will be automatically versioned by JaVers.

<h3 id="at-javers-auditable">@JaversAuditable for ordinary Repositories</h3>

If you're using ordinary Repositories (non Spring Data),
annotate all data-changing methods you want to audit with the method-level `@JaversAuditable`.

For example:

```java
@Repository
class UserRepository {
    @JaversAuditable
    public void save(User user) {
        ...//
    }

    public User find(String login) {
        ...//
    }
}
```

In fact, you can use this method-level annotation for advising any bean in your application.
It could be a Service, Repository or anything which modifies domain objects.

From now, all objects passed to the annotated methods will be automatically versioned by JaVers.

<h3 id="at-javers-auditable-conditional-delete">@JaversAuditableConditionalDelete for any kind of repositories</h3>

If you’re using any kind of reposity, and have delete methods with conditions, you can put `@JaversAuditableConditionalDelete` annotation on the methods to audit these events.

For example:

```java
@Repository
class UserRepository {
    @JaversAuditableConditionalDelete
    public User deleteByLogin(String login) {
        ...//
    }

    @JaversAuditableConditionalDelete
    public List<User> deleteAllByLogin(String login) {
        ...//
    }
}
```

These delete methods must return either an Entity or a collection of Entity.

<h2 id="jpa-entity-manager-integration">JPA EntityManager integration</h2>
Transaction management is the important issue for applications backed by SQL databases.
Generally, all SQL statements executed by `JaversSQLRepository`
should be executed in the context of the current application's transaction
(called Persistence Context in JPA terminology).

Read more about [ConnectionProvider](/documentation/repository-configuration/#connection-provider)
and JaVers’ approach to transaction management.

<h3 id="spring-configuration-for-transactional-javers">Spring configuration for SQL</h3>
**First**, you need to register exactly one **transactional** JaVers instance in your Application Context.
Simply use `TransactionalJaversBuilder` instead of standard JaversBuilder.

**Second**, you need to register a transactional ConnectionProvider.
If you’re using JPA with **Hibernate**, choose `JpaHibernateConnectionProvider` implementation
which is Persistence Context aware and plays along with Spring JpaTransactionManager.

**Third**, if you are using Hibernate, you need to deal with lazy-loading proxies.
Hibernate silently wraps them around your Entities loaded from database.
We strongly encourage to get rid of lazy-loading proxies before committing Entities to JaversRepository.
It can be easily obtained with [HibernateUnproxyObjectAccessHook](#hibernate-unproxy-hook). 

<h3 id="hibernate-unproxy-hook">Hibernate unproxy hook</h3>

JaVers provides `HibernateUnproxyObjectAccessHook` which is a way to unproxy
and initialize your Hibernate Entities just before processing them by JaVers diff & commit algorithms. 

To use HibernateUnproxyObjectAccessHook simply bind it to your JaVers instance using `JaversBuilder.withObjectAccessHook()` method:

```java
TransactionalJaversBuilder
    .javers()
    .withTxManager(txManager)
    .withObjectAccessHook(new HibernateUnproxyObjectAccessHook()).build()
```

Feel free to provide your own implementation of `object-access` hook if you need better control over
the unproxing process.

See below for the full Spring configuration example [for JPA & Hibernate](#spring-jpa-example).

<h2 id="spring-jpa-example">Spring configuration example for JPA & Hibernate</h2>

Here is a working example of Spring Application Context
with all JaVers beans, JPA, Hibernate, Spring Data and Spring TransactionManager.

```java
package org.javers.spring.example;

import ...

@Configuration
@ComponentScan(basePackages = "org.javers.spring.repository")
@EnableTransactionManagement
@EnableAspectJAutoProxy
@EnableJpaRepositories({"org.javers.spring.repository"})
public class JaversSpringJpaApplicationConfig {

    //.. JaVers setup ..

    /**
     * Creates JaVers instance with {@link JaversSqlRepository}
     */
    @Bean
    public Javers javers(PlatformTransactionManager txManager) {
        JaversSqlRepository sqlRepository = SqlRepositoryBuilder
                .sqlRepository()
                .withConnectionProvider(jpaConnectionProvider())
                .withDialect(DialectName.H2)
                .build();

        return TransactionalJaversBuilder
                .javers()
                .withTxManager(txManager)
                .withObjectAccessHook(new HibernateUnproxyObjectAccessHook())
                .registerJaversRepository(sqlRepository)
                .build();
    }

    /**
     * Enables auto-audit aspect for ordinary repositories.<br/>
     *
     * Use {@link org.javers.spring.annotation.JaversAuditable}
     * to mark data writing methods that you want to audit.
     */
    @Bean
    public JaversAuditableAspect javersAuditable(Javers javers) {
        return new JaversAuditableAspect(javers, authorProvider(), commitPropertiesProvider());
    }

    /**
     * Enables auto-audit aspect for Spring Data repositories. <br/>
     *
     * Use {@link org.javers.spring.annotation.JaversSpringDataAuditable}
     * to annotate CrudRepository, PagingAndSortingRepository or JpaRepository
     * you want to audit.
     */
    @Bean
    public JaversSpringDataJpaAuditableRepositoryAspect javersSpringDataAspect(Javers javers) {
        return new JaversSpringDataJpaAuditableRepositoryAspect(javers, authorProvider(),
            commitPropertiesProvider());
    }

    /**
     * Required by auto-audit aspect. <br/><br/>
     *
     * Creates {@link SpringSecurityAuthorProvider} instance,
     * suitable when using Spring Security
     */
    @Bean
    public AuthorProvider authorProvider() {
        return new SpringSecurityAuthorProvider();
    }

    /**
     * Optional for auto-audit aspect. <br/>
     * @see CommitPropertiesProvider
     */
    @Bean
    public CommitPropertiesProvider commitPropertiesProvider() {
        return new CommitPropertiesProvider() {
            @Override
            public Map<String, String> provideForCommittedObject(Object domainObject) {
                if (domainObject instanceof DummyObject) {
                    return Maps.of("dummyObject.name", ((DummyObject)domainObject).getName());
                }
                return Collections.emptyMap();
            }
        };
    }

    /**
     * Integrates {@link JaversSqlRepository} with Spring {@link JpaTransactionManager}
     */
    @Bean
    public ConnectionProvider jpaConnectionProvider() {
        return new JpaHibernateConnectionProvider();
    }
    //.. EOF JaVers setup ..


    //.. Spring-JPA-Hibernate setup ..
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource());
        em.setPackagesToScan("org.javers.spring.model");

        JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        em.setJpaProperties(additionalProperties());

        return em;
    }

    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory emf){
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(emf);

        return transactionManager;
    }

    @Bean
    public PersistenceExceptionTranslationPostProcessor exceptionTranslation(){
        return new PersistenceExceptionTranslationPostProcessor();
    }

    @Bean
    public DataSource dataSource(){
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl("jdbc:h2:mem:test;DB_CLOSE_DELAY=-1");
        return dataSource;
    }

    Properties additionalProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.hbm2ddl.auto", "create");
        properties.setProperty("hibernate.connection.autocommit", "false");
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.H2Dialect");
        return properties;
    }
    //.. EOF Spring-JPA-Hibernate setup ..
}
```

<h2 id="auto-audit-example-mongo">Spring configuration example for MongoDB</h2>

Here is a working example of Spring Application Context
with JaVers instance, JaVers auto-audit aspect and Spring Data MongoDB.

```java
package org.javers.spring.example;

import ...

@Configuration
@ComponentScan(basePackages = "org.javers.spring.repository")
@EnableMongoRepositories({"org.javers.spring.repository"})
@EnableAspectJAutoProxy
public class JaversSpringMongoApplicationConfig {
    private static final String DATABASE_NAME = "mydatabase";

    /**
     * Creates JaVers instance backed by {@link MongoRepository}
     */
    @Bean
    public Javers javers() {
        MongoRepository javersMongoRepository =
                new MongoRepository(mongo().getDatabase(DATABASE_NAME));

        return JaversBuilder.javers()
                .registerJaversRepository(javersMongoRepository)
                .build();
    }

    /**
     * MongoDB setup
     */
    @Bean(name="realMongoClient")
    @ConditionalOnMissingBean
    public MongoClient mongo() {
        return new MongoClient();
    }

    /**
     * required by Spring Data MongoDB
     */
    @Bean
    public MongoTemplate mongoTemplate() throws Exception {
        return new MongoTemplate(mongo(), DATABASE_NAME);
    }

    /**
     * Enables auto-audit aspect for ordinary repositories.<br/>
     *
     * Use {@link org.javers.spring.annotation.JaversAuditable}
     * to mark data writing methods that you want to audit.
     */
    @Bean
    public JaversAuditableAspect javersAuditableAspect() {
        return new JaversAuditableAspect(javers(), authorProvider(), commitPropertiesProvider());
    }

    /**
     * Enables auto-audit aspect for Spring Data repositories. <br/>
     *
     * Use {@link org.javers.spring.annotation.JaversSpringDataAuditable}
     * to annotate CrudRepositories you want to audit.
     */
    @Bean
    public JaversSpringDataAuditableRepositoryAspect javersSpringDataAuditableAspect() {
        return new JaversSpringDataAuditableRepositoryAspect(javers(), authorProvider(),
                commitPropertiesProvider());
    }

    /**
     * <b>INCUBATING - Javers Async API has incubating status.</b>
     * <br/><br/>
     *
     * Enables asynchronous auto-audit aspect for ordinary repositories.<br/>
     *
     * Use {@link JaversAuditableAsync}
     * to mark repository methods that you want to audit.
     */
    @Bean
    public JaversAuditableAspectAsync javersAuditableAspectAsync() {
        return new JaversAuditableAspectAsync(javers(), authorProvider(), commitPropertiesProvider(), javersAsyncAuditExecutor());
    }

    /**
     * <b>INCUBATING - Javers Async API has incubating status.</b>
     * <br/><br/>
     */
    @Bean
    public ExecutorService javersAsyncAuditExecutor() {
        ThreadFactory threadFactory = new ThreadFactoryBuilder()
                .setNameFormat("JaversAuditableAsync-%d")
                .build();
        return Executors.newFixedThreadPool(2, threadFactory);
    }

    /**
     * Required by auto-audit aspect. <br/><br/>
     *
     * Creates {@link SpringSecurityAuthorProvider} instance,
     * suitable when using Spring Security
     */
    @Bean
    public AuthorProvider authorProvider() {
        return new SpringSecurityAuthorProvider();
    }

    /**
     * Optional for auto-audit aspect. <br/>
     * @see CommitPropertiesProvider
     */
    @Bean
    public CommitPropertiesProvider commitPropertiesProvider() {
        return new CommitPropertiesProvider() {
            @Override
            public Map<String, String> provideForCommittedObject(Object domainObject) {
                if (domainObject instanceof DummyObject) {
                    return Maps.of("dummyObject.name", ((DummyObject)domainObject).getName());
                }
                return Collections.emptyMap();
            }
        };
    }
}
```