---
layout:     post
title:      JPA and Hibernate Tutorial using Spring Boot Data JPA
date:       2022-07-20 12:31:19
summary:    Complete journey starting from JDBC to JPA to Spring Data JPA using an example with Spring Boot Data JPA starter project. We use Hibernate as the JPA Implementation.
categories:  SpringBootJPA
permalink:  /introduction-to-jpa-with-spring-boot-data-jpa
---

This guide will help you understand what JPA is and how to setup a simple JPA example using Spring Boot.
 
## You will learn
- What is JPA?
- What is the problem solved by JPA-Object Relational Impedance?
- What are the alternatives to JPA? 
- What is Hibernate and how does it relate to JPA?
- What is Spring Data JPA?
- How to create a simple JPA project using Spring Boot Data JPA Starter?



## You will require the following tools:
- Maven 3.0+ is your build tool
- Your favorite IDE. We use Eclipse.
- JDK 17
- In memory database H2

## What is Object Relational Impedance Mismatch?

![Image](/images/JPA_01_Introduction.png "JPA Introduction")

Java is an object-oriented programming language. In Java, all data is stored in objects.

Typically, relational databases are used to store data (these days, a number of other NoSQL data stores are also becoming popular; we will stay away from them, for now). Relational databases store data in tables.

The way we design objects is different from the way relational databases are designed. This results in an impedance mismatch.
 - Object-oriented programming consists of concepts like encapsulation, inheritance, interfaces, and polymorphism.
 - Relational databases are made up of tables with concepts like normalization.

### Examples of Object Relational Impedance Mismatch

Let's consider a simple example: Employees and Tasks.

Each `Employee` can have multiple tasks. Each `Task` can be shared by multiple Employees. There is a many-to-many relationship between them. Let's consider a few examples of impedance mismatch.

#### Example 1: The task table below is mapped to the task table. However, there are mismatches in column names.

```java
public class Task {
	private int id;

	private String desc;

	private Date targetDate;

	private boolean isDone;

	private List<Employee> employees;
}
```

```sql
 CREATE TABLE task 
  ( 
     id          INTEGER GENERATED BY DEFAULT AS IDENTITY, 
     description VARCHAR(255), 
     is_done     BOOLEAN, 
     target_date TIMESTAMP, 
     PRIMARY KEY (id) 
  ) 
```

#### Example 2: Relationships between objects are expressed in a different way compared with those between tables.

Each `Employee` can have multiple `Tasks`. Each `Task` can be shared by multiple employees. There is a many-to-many relationship between them.

```java
public class Employee {
   
     //Some other code
	
	private List<Task> tasks;
}

public class Task {

     //Some other code

	private List<Employee> employees;
}
```

```sql
CREATE TABLE employee 
  ( 
     id            BIGINT NOT NULL, 
     OTHER_COLUMNS
  ) 


  CREATE TABLE employee_tasks 
  ( 
     employees_id BIGINT NOT NULL, 
     tasks_id     INTEGER NOT NULL 
  ) 

  CREATE TABLE task 
  ( 
     id          INTEGER GENERATED BY DEFAULT AS IDENTITY, 
     OTHER_COLUMNS
  ) 

```

#### Example 3: Sometimes multiple classes are mapped to a single table and vice-versa.

Objects

```java
public  class Employee {	
    //Other Employee Attributes
}

public class FullTimeEmployee extends Employee {
	protected Integer salary;
}

public class PartTimeEmployee extends Employee {
	protected Float hourlyWage;
}
```

Tables
```sql
CREATE TABLE employee 
  ( 
     employee_type VARCHAR(31) NOT NULL, 
     id            BIGINT NOT NULL, 
     city          VARCHAR(255), 
     state         VARCHAR(255), 
     street        VARCHAR(255), 
     zip           VARCHAR(255), 

     hourly_wage   FLOAT,  --PartTimeEmployee

     salary        INTEGER, --FullTimeEmployee

     PRIMARY KEY (id) 
  ) 

```

## Other approaches before JPA-JDBC, Spring JDBC & myBatis

Other approaches before JPA focused on queries and how to translate results from queries to objects.

Any approach using a query typically does two things.
- Setting parameters to the query requires us to read values from objects and set them as parameters to the query.
- Liquidation of results from the query: the results from the query need to be mapped to the beans.

### JDBC
- JDBC stands for Java Database Connectivity.
- It used concepts like Statement, PreparedStatement, and ResultSet.
- In the example below, the query used is ```Update todo set user=?, desc=?, target_date=?, is_done=? where id=?```
- The values needed to execute the query are set into the query using different set methods on the PreparedStatement.
- Results from the query are populated into the ResultSet. We had to write code to liquidate the result set into objects.


#### Update Todo
```java
Connection connection = datasource.getConnection();

PreparedStatement st = connection.prepareStatement(
		"Update todo set user=?, desc=?, target_date=?, is_done=? where id=?");

st.setString(1, todo.getUser());
st.setString(2, todo.getDesc());
st.setTimestamp(3, new Timestamp(todo.getTargetDate().getTime()));
st.setBoolean(4, todo.isDone());
st.setInt(5, todo.getId());

st.execute();

st.close();

connection.close();
```

#### Retrieved by Todo
```java
Connection connection = datasource.getConnection();

PreparedStatement st = connection.prepareStatement(
		"SELECT * FROM TODO where id=?");

st.setInt(1, id);

ResultSet resultSet = st.executeQuery();


if (resultSet.next()) {

    Todo todo = new Todo();
	todo.setId(resultSet.getInt("id"));
	todo.setUser(resultSet.getString("user"));
	todo.setDesc(resultSet.getString("desc"));
	todo.setTargetDate(resultSet.getTimestamp("target_date"));
	return todo;
}

st.close();

connection.close();

return null;
```

### Spring JDBC

- Spring JDBC provides a layer on top of JDBC
- It used concepts like JDBCTemplate.
- typically needs a lesser number of lines compared to JDBC as the following are simplified.
   - mapping parameters to queries.
   - liquidating resultsets to beans


#### Update Todo
```java
jdbcTemplate
.update("Update todo set user=?, desc=?, target_date=?, is_done=? where id=?",
	todo.getUser(), 
	todo.getDesc(),
	new Timestamp(todo.getTargetDate().getTime()),
	todo.isDone(), 
	todo.getId());
```

#### Retrieve a Todo

```java
@Override
public Todo retrieveTodo(int id) {

	return jdbcTemplate.queryForObject(
			"SELECT * FROM TODO where id=?",
			new Object[] { id }, new TodoMapper());

}
```

Reusable Row Mapper

```java
// new BeanPropertyRowMapper(TodoMapper.class)
class TodoMapper implements RowMapper<Todo> {
	@Override
	public Todo mapRow(ResultSet rs, int rowNum)
			throws SQLException {
		Todo todo = new Todo();

		todo.setId(rs.getInt("id"));
		todo.setUser(rs.getString("user"));
		todo.setDesc(rs.getString("desc"));
		todo.setTargetDate(
				rs.getTimestamp("target_date"));
		todo.setDone(rs.getBoolean("is_done"));
		return todo;
	}
}	
```

### myBatis

MyBatis removes the need for manually writing code to set parameters and retrieve results. It provides a simple XML or annotation based configuration to map Java POJOs to the database.

Below, we compare the approaches used to write queries.

- JDBC or Spring JDBC - ```Update todo set user=?, desc=?, target_date=?, is_done=? where id=?```
- myBatis - ```Update todo set user=#{user}, desc=#{desc}, target_date=#{targetDate}, is_done=#{isDone} where id=#{id}```

#### Todo Update and Todo Retrieve

```java
@Mapper
public interface TodoMybatisService
		extends TodoDataService {

	@Override
	@Update("Update todo set user=#{user}, desc=#{desc}, target_date=#{targetDate}, is_done=#{isDone} where id=#{id}")
	public void updateTodo(Todo todo) throws SQLException;

	@Override
	@Select("SELECT * FROM TODO WHERE id = #{id}")
	public Todo retrieveTodo(int id) throws SQLException;
}

public class Todo {

	private int id;

	private String user;

	private String desc;

	private Date targetDate;

	private boolean isDone;
}

```

### Common features of JDBC, Spring JDBC, and myBatis
- JDBC, Spring JDBC, and myBatis involve writing queries.
- Queries in a large application can become complex, especially when retrieving data from multiple tables.
- This creates a problem whenever there are changes in the structure of the database.

## How does JPA work?

JPA evolved as a result of a different thought process. How about mapping the objects directly to tables?
 - Entities
 - Attributes
 - Relationships

This mapping is also called ORM-Object Relational Mapping. Before JPA, ORM was the term more commonly used to refer to these frameworks. That's one of the reasons Hibernate is called an ORM framework.

## Important Concepts in JPA

![Image](/images/JPA_02_Architecture.png "JPA Architecuture")

JPA allows you to map application classes to tables in a database.
- Once the mappings are defined, the entity manager can manage your entities. The Entity Manager handles all interactions with the database.
- JPQL (Java Persistence Query Language) -Provides ways to write queries to execute searches against entities. The important thing to understand is that these are different from SQL queries. JPQL queries already understand the mappings that are defined between entities. We can add additional conditions as needed.
- Criteria API defines a Java based API to execute searches against databases.

## JPA vs Hibernate

Hibernate is one of the most popular ORM frameworks. 

JPA defines the specification. It is an API. 
 - How do you define entities?
 - How do you map attributes?
 - How do you map relationships between entities?
 - Who manages the entities?

Hibernate is one of the popular implementations of JPA.
 - Hibernate understands the mappings that we add between objects and tables. It ensures that data is stored and retrieved from the database based on the mappings.
 - Hibernate also provides additional features on top of JPA. But depending on them would mean a lock-in to Hibernate. You can not move to other JPA implementations like Toplink.

## Examples of JPA Mappings

Let's look at a few examples to understand how JPA can be used to map objects to tables.

#### Example 1

The Task class below is mapped to the Task table. However, there are mismatches in column names. We use a few JPA annotations to do the mapping.
 - @Table(name = "Task")
 - @Id
 - @GeneratedValue
 - @Column(name = "description")
 
```java
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.ManyToMany;
import jakarta.persistence.Table;

@Entity
@Table(name = "Task")
public class Task {
	@Id
	@GeneratedValue
	private int id;

	@Column(name = "description")
	private String desc;

	@Column(name = "target_date")
	private Date targetDate;

	@Column(name = "is_done")
	private boolean isDone;

}
```

```sql
 CREATE TABLE task 
  ( 
     id          INTEGER GENERATED BY DEFAULT AS IDENTITY, 
     description VARCHAR(255), 
     is_done     BOOLEAN, 
     target_date TIMESTAMP, 
     PRIMARY KEY (id) 
  ) 
```

#### Example 2

Relationships between objects are expressed in a different way compared to those between tables.

Each `Employee` can have multiple `Tasks`. Each `Task` can be shared by multiple Employees. There is a many-to-many relationship between them. We use @ManyToMany annotation to establish the relationship.

```java
public class Employee {
   
     //Some other code
	
	@ManyToMany
	private List<Task> tasks;
}

public class Task {

     //Some other code

	@ManyToMany(mappedBy = "tasks")
	private List<Employee> employees;
}
```

```sql
CREATE TABLE employee 
  ( 
     id            BIGINT NOT NULL, 
     OTHER_COLUMNS
  ) 


  CREATE TABLE employee_tasks 
  ( 
     employees_id BIGINT NOT NULL, 
     tasks_id     INTEGER NOT NULL 
  ) 

  CREATE TABLE task 
  ( 
     id          INTEGER GENERATED BY DEFAULT AS IDENTITY, 
     OTHER_COLUMNS
  ) 

```

#### Example 3

Sometimes multiple classes are mapped to a single table and vice-versa. In these situations, we define a inheritance strategy. In this example, we use a strategy of InheritanceType.SINGLE_TABLE.

Objects

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "EMPLOYEE_TYPE")
public  class Employee {	
    //Other Employee Attributes
}

public class FullTimeEmployee extends Employee {
	protected Integer salary;
}

public class PartTimeEmployee extends Employee {
	protected Float hourlyWage;
}
```

Tables

```sql
CREATE TABLE employee 
  ( 
     employee_type VARCHAR(31) NOT NULL, 
     id            BIGINT NOT NULL, 
     city          VARCHAR(255), 
     state         VARCHAR(255), 
     street        VARCHAR(255), 
     zip           VARCHAR(255), 

     hourly_wage   FLOAT,  --PartTimeEmployee

     salary        INTEGER, --FullTimeEmployee

     PRIMARY KEY (id) 
  ) 

```

## Step by Step Code Example

### Bootstrapping a Web application with Spring Initializr

Creating a JPA application with Spring Initializr is very simple.

![Image](/images/Spring-Initializr-Web.png "Web, Actuator and Developer Tools")   

As shown in the image above, the following steps have to be taken.

- Launch Spring Initializr [http://start.spring.io/](http://start.spring.io/){:target="_blank"} and choose the following
  - Choose `com.in28minutes.springboot` as Group
  - Choose `H2InMemoryDbDemo` as Artifact
  - Choose following dependencies
    - Web
    - JPA
    - H2 - We use H2 as in memory database
- Click the "Generate" button at the bottom of the page.
- Import the project into Eclipse.

#### The structure of the project created

- H2InMemoryDbDemoApplication.java-Spring Boot Launcher. It initialises Spring Boot Auto Configuration and Spring Application Context.
- application.properties - Application Configuration file.
- H2InMemoryDbDemoApplicationTests.java - A straightforward launcher for use in unit tests.
- pom.xml - I included dependencies for Spring Boot Starter Web and Data JPA. It uses Spring Boot Starter Parent as the parent pom.

Important dependencies are shown below.

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
	<groupId>com.h2database</groupId>
	<artifactId>h2</artifactId>
	<scope>runtime</scope>
</dependency>
```

### User Entity

Let's define a bean user and add the appropriate JPA annotations.

```java
package com.example.h2.user;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;

@Entity
@NamedQuery(query = "select u from User u", name = "query_find_all_users")
public class User {

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	private Long id;

	private String name;// Not perfect!! Should be a proper object!

	private String role;// Not perfect!! An enum should be a better choice!

	protected User() {
	}

	public User(String name, String role) {
		super();
		this.name = name;
		this.role = role;
	}

}
```
Important things to note:
 - ```@Entity```: Specifies that the class is an entity. This annotation is applied to the entity class.
 - ```@NamedQuery```: In the Java Persistence query language, specifies a static, named query.
 - ```@Id```: Specifies an entity's primary key.
 - ```@GeneratedValue```: Provides for the specification of generation strategies for the values of primary keys.
 - ```protected User()```: Default constructor to make JPA Happy.


## User Service to talk to Entity Manager

Typically, with JPA, we need to create a service to talk to the entity manager. In this example, we create a UserService to manage the persistence of a user entity.

```java
@Repository
@Transactional
public class UserService {
	
	@PersistenceContext
	private EntityManager entityManager;
	
	public long insert(User user) {
		entityManager.persist(user);
		return user.getId();
	}

	public User find(long id) {
		return entityManager.find(User.class, id);
	}
	
	public List<User> findAll() {
		Query query = entityManager.createNamedQuery(
				"query_find_all_users", User.class);
		return query.getResultList();
	}
}
```
Important things to note
 - ```@Repository```: This is a Spring Annotation to indicate that this component handles storing data to a data store.
 - ```@Transactional```: Spring annotations are used to simplify transaction management.
 - ```@PersistenceContext```: A persistence context handles a set of entities which hold data to be persisted in some persistence store (e.g. a database). In particular, the context is aware of the different states an entity can have (e.g. managed, detached) in relation to both the context and the underlying persistence store.
 - ```EntityManager``` : The interface is used to interact with the persistence context.
 - ```entityManager.persist(user)```: Make user entity instances managed and persistent, i.e., saved to the database.
 - ```entityManager.createNamedQuery```: It creates an instance of TypedQuery for executing a Java Persistence query language named query. The second parameter indicates the type of result.

Notes from http://docs.oracle.com/javaee/6/api/javax/persistence/EntityManager.html#createNamedQuery(java.lang.String)

> An EntityManager instance is associated with a persistence context. A persistence context is a set of entity instances in which for any persistent entity identity there is a unique entity instance. Within the persistence context, the entity instances and their lifecycle are managed. The EntityManager API is used to create and remove persistent entity instances, to find entities by their primary key, and to query over entities.

> The set of entities that can be managed by a given EntityManager instance is defined by a persistence unit. A persistence unit defines the set of all classes that are related or grouped by the application, and which must be colocated in their mapping to a single database.

## Command Line Runner for User Entity Manager

The CommandLineRunner interface is used to indicate that this bean has to be run as soon as the Spring application context is initialized.

We are executing a few simple methods on the UserService.

```java
@Component
public class UserEntityManagerCommandLineRunner implements CommandLineRunner {

	private static final Logger log = LoggerFactory.getLogger(UserEntityManagerCommandLineRunner.class);
	
	@Autowired
	private UserService userService;

	@Override
	public void run(String... args) {

		log.info("-------------------------------");
		log.info("Adding Tom as Admin");
		log.info("-------------------------------");
		User tom = new User("Tom", "Admin");
		userService.insert(tom);
		log.info("Inserted Tom" + tom);

		log.info("-------------------------------");
		log.info("Finding user with id 1");
		log.info("-------------------------------");
		User user = userService.find(1L);
		log.info(user.toString());

		log.info("-------------------------------");
		log.info("Finding all users");
		log.info("-------------------------------");
		log.info(userService.findAll().toString());
	}
}
```

Important things to note: 
 - `@Autowired` private UserService userService: The user service is autowired.
 - The rest of the stuff is straight-forward.

## Spring Data JPA

Spring Data aims to provide a consistent model for accessing data from different kinds of data stores.

UserService (which we created earlier) contains a lot of redundant code which can be easily generalized. Spring Data aims to simplify the code below.

```java
@Repository
@Transactional
public class UserService {
	
	@PersistenceContext
	private EntityManager entityManager;
	
	public long insert(User user) {
		entityManager.persist(user);
		return user.getId();
	}

	public User find(long id) {
		return entityManager.find(User.class, id);
	}
	
	public List<User> findAll() {
		Query query = entityManager.createNamedQuery(
				"query_find_all_users", User.class);
		return query.getResultList();
	}
}
```

As far as JPA is concerned, there are two Spring Data modules that you would need to know about.
 - Spring Data Commons defines the concepts that are shared by all Spring Data Modules. 
 - Spring Data JPA - Provides easy integration with JPA repositories.


### CrudRepository

CrudRepository is the Spring Data Commons pre-defined core repository class that enables the basic CRUD functions on a repository. Important methods are shown below.

```java
public interface CrudRepository<T, ID extends Serializable>
    extends Repository<T, ID> {

    <S extends T> S save(S entity);

    T findOne(ID primaryKey);       

    Iterable<T> findAll();          

    Long count();                   

    void delete(T entity);          

    boolean exists(ID primaryKey);  

    // … more functionality omitted.
}
```

### JpaRepository

JpaRepository (defined in Spring Data JPA) is the JPA-specific repository interface.

```java
public interface JpaRepository<T, ID extends Serializable>
		extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
```

We will now use the JpaRepository to manage the user entity. The below snippet shows the important details. We would want the UserRepository to manage the User entity which has a primary key of type Long.

```java
package com.example.h2.user;

import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, Long> {
}
```

### User Repository: CommandLineRunner

The code below is very simple. The CommandLineRunner interface is used to indicate that this bean has to be run as soon as the Spring application context is initialized. We are executing a few simple methods on the UserRepository.


```java
package com.example.h2;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import com.example.h2.user.User;
import com.example.h2.user.UserRepository;

@Component
public class UserRepositoryCommandLineRunner implements CommandLineRunner {

	private static final Logger log = LoggerFactory.getLogger(UserRepositoryCommandLineRunner.class);

	@Autowired
	private UserRepository userRepository;

	@Override
	public void run(String... args) {
		User harry = new User("Harry", "Admin");
		userRepository.save(harry);
		log.info("-------------------------------");
		log.info("Finding all users");
		log.info("-------------------------------");
		for (User user : userRepository.findAll()) {
			log.info(user.toString());
		}
	}

}
```
Important things to note: 
 - @Autowired private UserRepository userRepository: The user repository is autowired.
 - The rest of the stuff is straight-forward.

### H2 Console
We will enable h2 console in /src/main/resources/application.properties

```
spring.h2.console.enabled=true
```

You can start the application by running H2InMemoryDbDemoApplication as a Java application.

The H2-Console can also be accessed via the web browser.
- http://localhost:8080/h2-console
- Use db url jdbc:h2:mem:testdb

### Questions
- Where was the database created?
  - In Memory - Using H2
- What schema is used to create the tables?
  - created based on the entities defined
- Where are the tables created?
  - Created based on the entities defined
  - In Memory - Using H2
- Can I see the data in the database?
   - http://localhost:8080/h2-console
   - Use db url jdbc:h2:mem:testdb
- Where is Hibernate coming from?
  - Through the Spring Data JPA Starter
- How is a data source created?
  - Through Spring Boot Auto Configuration 

### Spring Boot's magic and in-memory databases
- Zero project setup or infrastructure
- Zero Configuration
- Zero Maintainance
- Easy to use for Learning and Unit Tests
- Simple Configuration to switch to a real database

### Restrictions on using in-memory databases
- Data is not persisted between restarts.



## Complete Code Example


### /pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.example</groupId>
	<artifactId>jpa-in-10-steps</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>jpa-with-in-memory-db-in-10-steps</name>
	<description>Demo project for in memory database H2</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.0.0-M3</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<java.version>17</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>

		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

	<repositories>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>

	<pluginRepositories>
		<pluginRepository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</pluginRepository>
	</pluginRepositories>

</project>
```
---

### /src/main/java/com/example/h2/H2InMemoryDbDemoApplication.java

```java
package com.example.h2;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class H2InMemoryDbDemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(H2InMemoryDbDemoApplication.class, args);
	}
}
```
---

### /src/main/java/com/example/h2/user/User.java

```java
package com.example.h2.user;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.NamedQuery;

@Entity
@NamedQuery(query = "select u from User u", name = "query_find_all_users")
public class User {

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	private Long id;

	private String name;// Not perfect!! Should be a proper object!
	private String role;// Not perfect!! An enum should be a better choice!

	protected User() {
	}

	public User(String name, String role) {
		super();
		this.name = name;
		this.role = role;
	}

	public Long getId() {
		return id;
	}

	public String getName() {
		return name;
	}

	public String getRole() {
		return role;
	}

	@Override
	public String toString() {
		return String.format("User [id=%s, name=%s, role=%s]", id, name, role);
	}

}
```
---

### /src/main/java/com/example/h2/user/UserRepository.java

```java
package com.example.h2.user;

import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, Long> {
}
```
---

### /src/main/java/com/example/h2/user/UserService.java

```java
package com.example.h2.user;

import java.util.List;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.persistence.Query;
import jakarta.transaction.Transactional;

import org.springframework.stereotype.Repository;

@Repository
@Transactional
public class UserService {
	
	@PersistenceContext
	private EntityManager entityManager;
	
	public long insert(User user) {
		entityManager.persist(user);
		return user.getId();
	}

	public User find(long id) {
		return entityManager.find(User.class, id);
	}
	
	public List<User> findAll() {
		Query query = entityManager.createNamedQuery(
				"query_find_all_users", User.class);
		return query.getResultList();
	}
}
```
---

### /src/main/java/com/example/h2/UserEntityManagerCommandLineRunner.java

```java
package com.example.h2;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import com.example.h2.user.User;
import com.example.h2.user.UserService;

@Component
public class UserEntityManagerCommandLineRunner implements CommandLineRunner {

	private static final Logger log = LoggerFactory.getLogger(UserEntityManagerCommandLineRunner.class);
	
	@Autowired
	private UserService userService;

	@Override
	public void run(String... args) {

		log.info("-------------------------------");
		log.info("Adding Tom as Admin");
		log.info("-------------------------------");
		User tom = new User("Tom", "Admin");
		userService.insert(tom);
		log.info("Inserted Tom" + tom);

		log.info("-------------------------------");
		log.info("Finding user with id 1");
		log.info("-------------------------------");
		User user = userService.find(1L);
		log.info(user.toString());

		log.info("-------------------------------");
		log.info("Finding all users");
		log.info("-------------------------------");
		log.info(userService.findAll().toString());
	}
}
```
---

### /src/main/java/com/example/h2/UserRepositoryCommandLineRunner.java

```java
package com.example.h2;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import com.example.h2.user.User;
import com.example.h2.user.UserRepository;

@Component
public class UserRepositoryCommandLineRunner implements CommandLineRunner {

	private static final Logger log = LoggerFactory.getLogger(UserRepositoryCommandLineRunner.class);

	@Autowired
	private UserRepository userRepository;

	@Override
	public void run(String... args) {
		User harry = new User("Harry", "Admin");
		userRepository.save(harry);
		log.info("-------------------------------");
		log.info("Finding all users");
		log.info("-------------------------------");
		for (User user : userRepository.findAll()) {
			log.info(user.toString());
		}
	}

}
```
---

### /src/main/resources/application.properties

```
spring.h2.console.enabled=true
#logging.level.org.hibernate=debug
spring.jpa.show-sql=true
spring.datasource.url=jdbc:h2:mem:testdb;NON_KEYWORDS=USER
spring.data.jpa.repositories.bootstrap-mode=default
spring.jpa.defer-datasource-initialization=true
```
---

### /src/main/resources/data.sql

```
insert into user (id, name, role) values (101, 'Ranga', 'Admin');
insert into user (id, name, role) values (102, 'Ravi', 'User');
insert into user (id, name, role) values (103, 'Satish', 'Admin');
```
---

### /src/test/java/com/example/h2/H2InMemoryDbDemoApplicationTests.java

```java
package com.example.h2;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit.jupiter.SpringExtension;

@ExtendWith(SpringExtension.class)
@SpringBootTest
public class H2InMemoryDbDemoApplicationTests {

	@Test
	public void contextLoads() {
	}

}
```
---

### /src/test/java/com/example/h2/user/UserRepositoryTest.java

```java
package com.example.h2.user;

import static org.junit.Assert.assertEquals;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import com.example.h2.user.UserRepository;

@DataJpaTest
@ExtendWith(SpringExtension.class)
public class UserRepositoryTest {

	@Autowired
	UserRepository userRepository;

	@Autowired
	TestEntityManager entityManager;

	@Test
	public void check_todo_count() {
		assertEquals(3, userRepository.count());
	}
}
```
---
