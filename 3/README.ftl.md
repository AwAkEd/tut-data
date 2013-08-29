 
# Storing Order Data Using Java Persistence API (JPA)

Your next task for the Yummy Noodle Bar persistence project is to store Order data.  Yummy Noodle Bar has decided to use PostgreSQL to store this data, a freely available, robust, relational database.

Continuing from the last section, we will work within the Persistence domain.

In that domain we have a representation of an Order, `com.yummynoodle.persistence.domain.Order`, that we can optimise for persistence without affecting the rest of the application.

There is an event handler `OrderPersistenceEventHandler`, exchanging events with the application core,
and the repository `OrdersRepository` whose responsibility is to persist and retrieve Order data for the rest of the application.

You will implement `OrdersRepository` using Spring Data JPA and integrate this with the `OrderPersistenceEventHandler`

JPA is the Java standard for accessing relational databases, including Postgres.  It provides an API to map Java Objects to relational concepts and gives aq rich vocabulary for querying across these objects, transparently translating to SQL (most of the time).

Since JPA is a standard, there are many implementations, referred to as JPA *Providers*.

In addition to Spring Data JPA, Hibernate has been chosen as the JPA provider.

## Using H2 as an in memory database for testing

PostgreSQL is a fully functional database that is suitable for production. However for testing, a lighter, embedded database is suitable. You will use H2 for development purposes during this tutorial.

This allows the easy creation and destruction of database instances in a lifecycle controlled by the test.

The JPA standard and provider will provide enough of an abstraction that we may use different databases in production and development. It is recommended to test application integration with the production database in addition to the interaction tests run against H2. This would be part of the minimal set of acceptance or 'smoke' tests.
    
## Import Spring Data JPA

Import Spring Data JPA and the Hibernate JPA Provider into your project, adding it to the build.gradle

```groovy
dependencies {
   ...
  compile 'org.springframework.data:spring-data-mongodb:1.2.3.RELEASE'
  compile 'org.springframework.data:spring-data-jpa:1.3.4.RELEASE'
  compile 'org.hibernate.javax.persistence:hibernate-jpa-2.0-api:1.0.1.Final'
  compile 'org.hibernate:hibernate-entitymanager:4.0.1.Final'
  runtime 'com.h2database:h2:1.3.173'
  ...
}
```

Also here is the JDBC Driver for H2.

## Start with a (failing) test, introducing JPA

Following the pattern from the the previous section, you will first create a test to drive your development, first checking that the persistence mapping class correctly (de)serializes.

Create a new test class `com.yummynoodlebar.persistence.integration.OrderMappingIntegrationTests`.

This class will check that the class `com.yummynoodle.persistence.domain.Order` correctly maps to the JPA wrapped database (H2).  It is important to understand how your object is being mapped against the database, and test that it meets your expectations.  If you know how and why the mapping is occurring, you can create indexes and other optimisations safe in the knowledge that the data is where you expect it to be.  You may also access the data outside of the JPA provider, directly querying the database.

Next, create an empty class `com.yummynoodlebar.config.JPAConfiguration`.  This will contain setup necessary to initialise the necessary JPA infrastructure.

Once JPAConfiguration is present, update `OrderMappingIntegrationTests` to read

```java
package com.yummynoodlebar.persistence.integration;

import com.yummynoodlebar.config.JPAConfiguration;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.context.transaction.TransactionConfiguration;
import org.springframework.transaction.annotation.Transactional;

import javax.persistence.EntityManager;

import static com.yummynoodlebar.persistence.domain.fixture.JPAAssertions.assertTableExists;
import static com.yummynoodlebar.persistence.domain.fixture.JPAAssertions.assertTableHasColumn;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {JPAConfiguration.class})
@Transactional
@TransactionConfiguration(defaultRollback = true)
public class OrderMappingIntegrationTests {

  @Autowired
  EntityManager manager;

  @Test
  public void thatItemCustomMappingWorks() throws Exception {
    assertTableExists(manager, "NOODLE_ORDERS");
    assertTableExists(manager, "ORDER_ORDER_ITEMS");

    assertTableHasColumn(manager, "NOODLE_ORDERS", "ID");
    assertTableHasColumn(manager, "NOODLE_ORDERS", "SUBMISSION_DATETIME");
  }
}
```

This test is making use of an existing helper class `JPAAssertions` that makes use of the Hibernate JPA Provider and direct JDBC to inspect what has been done to the database schema.

```java
assertTableExists(manager, "NOODLE_ORDERS");
```

This line states the expectation that the table NOODLE_ORDERS exists. 
This is followed by the checks that assert the column structure.
If required, these tests could be extended to further check the schema definition to ensure that the data is being mapped as expected.

This test will not pass, however, as the JPA infrastructure needed to connect `Order` with the database has not been initialised.

Update `JPAConfiguration` to read

```java
package com.yummynoodlebar.config;

import com.yummynoodlebar.persistence.repository.OrdersRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.FilterType;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseBuilder;
import org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType;
import org.springframework.orm.hibernate4.HibernateExceptionTranslator;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;

import javax.persistence.EntityManager;
import javax.persistence.EntityManagerFactory;
import javax.sql.DataSource;
import java.sql.SQLException;

@Configuration
@EnableTransactionManagement
public class JPAConfiguration {

  @Bean
  public DataSource dataSource() throws SQLException {

    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
    return builder.setType(EmbeddedDatabaseType.H2).build();
  }

  @Bean
  public EntityManagerFactory entityManagerFactory() throws SQLException {

    HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    vendorAdapter.setGenerateDdl(true);

    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setJpaVendorAdapter(vendorAdapter);
    factory.setPackagesToScan("com.yummynoodlebar.persistence.domain");
    factory.setDataSource(dataSource());
    factory.afterPropertiesSet();

    return factory.getObject();
  }

  @Bean
  public EntityManager entityManager(EntityManagerFactory entityManagerFactory) {
    return entityManagerFactory.createEntityManager();
  }

  @Bean
  public PlatformTransactionManager transactionManager() throws SQLException {

    JpaTransactionManager txManager = new JpaTransactionManager();
    txManager.setEntityManagerFactory(entityManagerFactory());
    return txManager;
  }

  @Bean
  public HibernateExceptionTranslator hibernateExceptionTranslator() {
    return new HibernateExceptionTranslator();
  }
}
```

The method `DataSource dataSource()` creates the embedded H2 database.  This creates a new H2 instance within the same ApplicationContext and provides a DataSource interface to it, usable by JPA.

The method `EntityManagerFactory entityManagerFactory()` creates the `EntityManagerFactory`.  This class is responsible for creating the `EntityManager`, and is *JPA Provider specific*. In this case, this shows the creation and setup of a Hibernate JPA Provider, including the provision of the datasource `dataSource()`.
Note that the EntityManagerFactory is responsible for identifying the JPA Entities to be made available, the classes to be treated as database mapping/ persistence beans.

The method `EntityManager entityManager()` creates the core class of JPA.  `EntityManager` is the public interface of JPA, providing methods to persist, delete, update and query, and is used in the tests below for this purpose.

`transactionManager()` initialises the JPA transaction manager. This integrates with the declarative Transaction Management features of Spring, permitting the use of @Transactional and associated classes and configuration, for more information, [click here](http://static.springsource.org/spring/docs/3.2.4.RELEASE/spring-framework-reference/html/transaction.html)

Spring provides a exception translation framework to translate exceptions from many different sources into a consistent set that your application can use. In this case, the JPA Configuration setup expects a bean that provides these translations, which is provided by `hibernateExceptionTranslator()`

The test will now run without compilation or runtime errors, but will fail as the JPA entity is not set up.

Update `com.yummynoodle.persistence.domain.Order` to read

```java
package com.yummynoodlebar.persistence.domain;

import com.yummynoodlebar.events.orders.OrderDetails;

import javax.persistence.*;
import java.util.Collections;
import java.util.Date;
import java.util.Map;
import java.util.UUID;

@Entity(name = "NOODLE_ORDERS")
public class Order {

  @Column(name = "SUBMISSION_DATETIME")
  private Date dateTimeOfSubmission;

  @ElementCollection(fetch = FetchType.EAGER, targetClass = java.lang.Integer.class)
  @JoinTable(name="ORDER_ORDER_ITEMS", joinColumns=@JoinColumn(name="ID"))
  @MapKeyColumn(name="MENU_ID")
  @Column(name="VALUE")
  private Map<String, Integer> orderItems;

  @Transient
  private OrderStatus orderStatus;

  @Id
  @Column(name = "ORDER_ID")
  private String id;

  public void setId(String id) {
    this.id = id;
  }

  public void setDateTimeOfSubmission(Date dateTimeOfSubmission) {
    this.dateTimeOfSubmission = dateTimeOfSubmission;
  }

  public OrderStatus getStatus() {
    return orderStatus;
  }

  public void setStatus(OrderStatus orderStatus) {
    this.orderStatus = orderStatus;
  }

  public Date getDateTimeOfSubmission() {
    return dateTimeOfSubmission;
  }

  public String getId() {
    return id;
  }

  public void setOrderItems(Map<String, Integer> orderItems) {
    if (orderItems == null) {
      this.orderItems = Collections.emptyMap();
    } else {
      this.orderItems = Collections.unmodifiableMap(orderItems);
    }
  }

  public Map<String, Integer> getOrderItems() {
    return orderItems;
  }

  public OrderDetails toOrderDetails() {
    OrderDetails details = new OrderDetails();

    details.setKey(UUID.fromString(this.id));
    details.setDateTimeOfSubmission(this.dateTimeOfSubmission);
    details.setOrderItems(this.getOrderItems());

    return details;
  }

  public static Order fromOrderDetails(OrderDetails orderDetails) {
    Order order = new Order();

    order.id = orderDetails.getKey().toString();
    order.dateTimeOfSubmission = orderDetails.getDateTimeOfSubmission();
    order.orderItems = orderDetails.getOrderItems();

    return order;
  }
}
```

`@Entity(name = "NOODLE_ORDERS")` declares this class as a JPA *Entity*. This is a class that is mapped to a database and able to be consumed by `EntityManager`.

`@Column(name = "SUBMISSION_DATETIME")` is a JPA customisation that alters the name of the column this field will be mapped to. The default is the name of the field converted from lower camel (aFieldName) to uppsercase underscore case (A_FIELD_NAME)

```java
@ElementCollection(fetch = FetchType.EAGER, targetClass = java.lang.Integer.class)
  @JoinTable(name="ORDER_ORDER_ITEMS", joinColumns=@JoinColumn(name="ID"))
  @MapKeyColumn(name="MENU_ID")
  @Column(name="VALUE")
```

This complex mapping is used to create a joined table, ORDER_ORDER_ITEMS, that contains the data stored in the java.util.Map.

`@Transient` informs JPA that the given field should not be stored in the database, and is analogous to the Java *transient* keyword 

Finally `@Id` indicates to both JPA and Spring Data that the given field(s) is the Primary Key, and should be used to both index and provide the default access method for the Entity.

The tests will now pass, indicating that the mapping is all working as expected.


## Implement a CRUD repository

Now that the JPA Entity works, the Repository can be implemented.
In the same way as for MongoDB, Spring Data provides a way to automatically create JPA backed Repositories, given only an interface.

Create a new test class to check the Repository.

```java
package com.yummynoodlebar.persistence.integration;


import com.yummynoodlebar.config.JPAConfiguration;
import com.yummynoodlebar.persistence.domain.Order;
import com.yummynoodlebar.persistence.repository.OrdersRepository;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.context.transaction.TransactionConfiguration;
import org.springframework.transaction.annotation.Transactional;

import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

import static junit.framework.TestCase.assertEquals;
import static junit.framework.TestCase.assertNotNull;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {JPAConfiguration.class})
@Transactional
@TransactionConfiguration(defaultRollback = true)
public class OrdersRepositoryIntegrationTests {

  @Autowired
  OrdersRepository ordersRepository;

  @Test
  public void thatItemIsInsertedIntoRepoWorks() throws Exception {
    String key = UUID.randomUUID().toString();

    Order order = new Order();
    order.setDateTimeOfSubmission(new Date());
    order.setId(key);

    Map<String, Integer> items = new HashMap<String, Integer>();

    items.put("yummy1", 15);
    items.put("yummy3", 12);
    items.put("yummy5", 7);

    order.setOrderItems(items);

    ordersRepository.save(order);

    Order retrievedOrder = ordersRepository.findById(key);

    assertNotNull(retrievedOrder);
    assertEquals(key, retrievedOrder.getId());
    assertEquals(items.get("yummy1"), retrievedOrder.getOrderItems().get("yummy1"));
  }

}

```

The section 

```java
@Transactional
@TransactionConfiguration(defaultRollback = true)
```

Is new.  These annotations integrate with the Spring declarative transaction management mentioned above.  These state that every method on this class requires a transaction to be started and stopped around it, and that the transaction should be, by default, rolled back on method completion.  

This gives a natural way to construct tests against a database, as you may update the database through the test, reading and writing at will, and at the end of the test, the transaction will be rolled back and the data discarded, leaving a fresh environment for the next test to exectute within.

This test requires an instance of `OrdersRepository` and saves several Order instances to it before calling a findById method

This will fail, as the implementation of `OrdersRepository` does not yet exist.

To solve this, update `JPAConfiguration` to include enable the JPA Repository system.

Again, the desired repository is selected explicitly as multiple data sources/ Spring Data configurations are being used.

```java

@Configuration
@EnableJpaRepositories(basePackages = "com.yummynoodlebar.persistence.repository",
    includeFilters = @ComponentScan.Filter(value = {OrdersRepository.class}, type = FilterType.ASSIGNABLE_TYPE))
@EnableTransactionManagement
public class JPAConfiguration {

```

And replace the contents of `OrdersRepository` to extend the Spring Data `CrudRepository`

```java
package com.yummynoodlebar.persistence.repository;

import com.yummynoodlebar.persistence.domain.Order;
import org.springframework.data.repository.CrudRepository;

public interface OrdersRepository extends CrudRepository<Order, UUID> {

}
```
The test will now pass correctly, indicating that an implementation of `OrderRepository` is being created at runtime and works as expected.

## Extend the Repository with a Custom Finder

A requirement that affects the Persistence domain is that users need to be able to find Orders that contain certain menu items, by Menu Item ID.

As you should be comfortable with now, create a new test `OrdersRepositoryFindOrdersContainingTests` that will ensure this functionality is implemented correctly.

```java
package com.yummynoodlebar.persistence.integration;

import com.yummynoodlebar.config.JPAConfiguration;
import com.yummynoodlebar.persistence.domain.Order;
import com.yummynoodlebar.persistence.domain.fixture.PersistenceFixture;
import com.yummynoodlebar.persistence.repository.OrdersRepository;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.context.transaction.TransactionConfiguration;
import org.springframework.transaction.annotation.Transactional;

import javax.persistence.EntityManager;
import java.util.List;

import static junit.framework.TestCase.assertEquals;
import static junit.framework.TestCase.assertNotNull;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {JPAConfiguration.class})
@Transactional
@TransactionConfiguration(defaultRollback = true)
public class OrdersRepositoryFindOrdersContainingTests {

  @Autowired
  OrdersRepository ordersRepository;

  @Autowired
  EntityManager entityManager;

  @Test
  public void thatSearchingForOrdesContainingWorks() throws Exception {

    ordersRepository.save(PersistenceFixture.standardOrder());
    ordersRepository.save(PersistenceFixture.standardOrder());
    ordersRepository.save(PersistenceFixture.yummy16Order());
    ordersRepository.save(PersistenceFixture.yummy16Order());

    List<Order> retrievedOrders = ordersRepository.findOrdersContaining("yummy16");

    assertNotNull(retrievedOrders);
    assertEquals(2, retrievedOrders.size());
    assertEquals(22, (int) retrievedOrders.get(0).getOrderItems().get("yummy16"));

    retrievedOrders = ordersRepository.findOrdersContaining("yummy3");

    assertNotNull(retrievedOrders);
    assertEquals(2, retrievedOrders.size());
  }

}
```

Again, this test is a transactional database test, expecting database rollback on test conclusion.  
It saves a set of orders and performs two queries using a method `findOrdersContaining` that will accept a menu item ID.

Now, to implement the method, open the repository and update it with the new method

```java
@Query(value = "select no.* from NOODLE_ORDERS no where no.ORDER_ID in (select ID from ORDER_ORDER_ITEMS where MENU_ID = :menuId)", nativeQuery = true)
  List<Order> findOrdersContaining(@Param("menuId") String menuId);
```

This class uses a custom @Query. It passes in a SQL query, for which you have to set nativeQuery=true.  Without `nativeQuery=true', the string in @Query is assumed to be a JPA Query Language query instead.

Based on our knowledge of the mapping, using this SQL statement is safe, and we can rely on the structure it assumes.

The full listing is.

```java
package com.yummynoodlebar.persistence.repository;

import com.yummynoodlebar.persistence.domain.Order;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface OrdersRepository extends CrudRepository<Order, String> {
  Order findById(String key);

  @Query(value = "select no.* from NOODLE_ORDERS no where no.ORDER_ID in (select ID from ORDER_ORDER_ITEMS where MENU_ID = :menuId)", nativeQuery = true)
  List<Order> findOrdersContaining(@Param("menuId") String menuId);
}
```

The test will now pass, and the custom query is completed.

### Summary

Congratulations, Order data is safely stored in a JPA managed relational database.

Next, you can see how to store OrderStatus data in the Gemfire distributed data grid.

[Next…  Storing the Order Status in Gemfire using Spring Data Gemfire](../4/)