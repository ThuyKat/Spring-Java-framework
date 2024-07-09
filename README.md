# Spring-Java-framework

Spring FW is used for building enterprise Java apps. Core features: 
- Dependency injection : allows objects to be configured and wired together
- Aspect-Oriented Programming: allows the separation of cross-cutting concerns from business logic. 
- Inversion of Control: when spring is doing dependency injection on our behalf. There are two options : via bean.xml and via annotations

# bean.xml
Spring maintains a container ( also called ApplicationContext/BeanFactory). When you run the program, Spring would check for bean.xml for which classes you specify to create a bean and create that bean inside the container. At anytime, when you use .getBean(bean_name) method, Spring will fetch the bean created in its container and provides to you
```java
Student st = (Student).getBean(Student);
```
For example, inside resource folder, we have "factoryBean.xml" file: 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://www.springframework.org/schema/beans
							http://www.springframework.org/schema/beans/spring-beans.xsd">
	<bean id = "address" class ="Address">
		<property name ="city" value ="Hanoi" />
		<property name ="postcode" value ="10000"/>
		</bean>
		
		<bean id = "student" class="Student">
		<property name ="rollNo" value="1"/>
		<property name = "name" value ="katie"/>
		<property name = "address" ref = "address"/>
	</bean>
</beans>
```
Also, in pom.xml, we add spring-context dependency : 
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.spring</groupId>
  <artifactId>Demo</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <dependencies>
  	<dependency>
  		<groupId>org.springframework</groupId>
  		<artifactId>spring-context</artifactId>
  		<version>6.1.10</version>
  	</dependency>
  </dependencies>
</project>

```
The dependency is the container that we mentioned above to manage the beans. 

Assume that we have class Student and class Address with all getter and setters

In main() method, we can call .getBean() method: 

```java
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class demo {
public static void main(String[] args) {
	ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("factoryBean.xml");
	
	//we need to downcast from Object to Student type
	Student student = (Student)applicationContext.getBean("student");
	
	System.out.println(student);
}
}
```
In default, bean in application context is Single Bean scope. That is, if we create another instance using .getBean() method, we will get the same hashcode as the previous one. 

To change this single bean scope and allow for multiple beans, we change scope="prototype" in factoryBean.xml in student bean tag

## Difference between Singleton and single bean scope
- Singleton: is a design pattern which doesnt allow 2 instance of the class to exist. It requires a private constructor, private attributes and a static method acting as the global point of access to the instance
- Single bean scope: default bean scope in Spring which allows the spring container to have only one instance of bean per Spring container. The same instance is shared and injected wherever needed. However, this bean is only confined to the container, outside the container, I can create an instance of the class, e.g. student2

## Early initialisation

By default, beeans in Spring are instantiated  early when the application context is called. This can lead to increased memory usage, not suitable for components that are rarely used. 

We can change this to lazy initialisation by define bean scope as prototypes and lazy-init="true". This means bean only be created when it is requested from the application context

In above code, we print out message when Spring initialises either Student or Address via their default constructors. As a result, the Address bean is initialised before Student bean: 

```
inside Address constructor
inside Student constructor
Student [name=katie, address=Address@17c386de, rollNo=1]
```
Now, if we redefine scope and lazy-init in factoryBean.xml, we get different result:
```
inside Student constructor
inside Address constructor
Student [name=katie, address=Address@12d3a4e9, rollNo=1]
```
## init-method 

In factoryBean.xml, we can add another argument called init-method="methodName" with methodName is the name of the method to be executed after the bean's properties have been set. 
e.g.,Student class:
```java
public void onceBeanCreatedRunThis() {
	System.out.println("Student bean is created");
}
```
```xml
<bean id = "student" class="Student" scope="prototype" lazy-init="true" init-method="onceBeanCreatedRunThis">
		<property name ="rollNo" value="1"/>
		<property name = "name" value ="katie"/>
		<property name = "address" ref = "address"/>
	</bean>
```
This is useful for performing any custom initialization logic that the bean requires. The init-method can be any method, it doesnt need to have any specific params or return type

Result: 

![alt text](SS/image.png)
# Annotations: 

@Component, @Autowired to manages the beans. When we create object, we create an instance of a class with "new" keyword. Bean is actually an instance of a class but created by Spring ( internally Spring also uses "new" keyword)

To use annotation we are going through these steps: 
1. Create beans and annotated with @Component
```java
package notification;

import org.springframework.stereotype.Component;

@Component
public class SMSGateway {

	public void sendMessage(String mes) {
		System.out.println("smsGW printing" + mes);
		
	}

}
```
2. When we need to use the bean, declare the beans and annotate them with @Autowired 

```java
package notification;

import org.springframework.beans.factory.annotation.Autowired;
@Component
public class NotificationService {

	@Autowired
	EmailGateway emailGW;
	@Autowired
	SMSGateway smsGW;
	
	public void sendSMS(String mes) {
		smsGW.sendMessage(mes);
	}
}
```
3. In main method, call an instance of AnnotationConfigApplicationContext and pass the name of the package that we need to scan annotations

```java
AnnotationConfigApplicationContext appContext = new AnnotationConfigApplicationContext("notification");
```
4. Just like how we call an instance of Student class above, we can use .getBean(instance_name) to get the instance of the class with default instance_name is the lowercase of the Class name( e.g., NotificationService will have instance name of notificationService). If you want to use your own identifier, specify the custom name as an argument of @Component("customName")
```java
NotificationService service = (NotificationService) appContext.getBean("notificationService");
	service.sendSMS("this is my message");
```

When we run above code, the constructor of NotificationService is evoked leads to SMSGW constructor evoked and @Autowired annotation is used to attach these two bean ( just like how ref argument do in factoryBean.xml). @Autowired annotation is used when we want to store component of 1 class in an other class. In this case, NotificationService requires reference of SMSGateway.

**How spring knows which ref that notificationService needs?**

Lets say apart from SMSGateway, we also have EmailGateway autowired to NotificationService. 
In this case, spring checks the type of attributes then use the bean for that particular type
Above code we use Field injection, howevery constructor injection is prefered because 
- it ensures dependencies available only when needed. 
- Dependencies can be 'final' -set for once and never change
- In this case, we can see smsGW is used for NotificationService, makeing the code more understandable
- Reduce the risk of 'NullPointerException' since constructor ensures all required dependencies are provided

**Another way to inject dependency: Setter injection**

```java
@Autowired
public void setSMSGateWay(SMSGateway smsGw){
    this.smsGateway = smsGw;
}
```
NotificationService is also created first in this case like field injection

**applicationContext.close()**: 

whatever bean you have inside the container will be destroyed together with the container itself

Before the container is destroyed, we can specify a method to be executed before it happens by adding **destroy-method** = "methodName". This is useful for performing any cleanup operations that the bean requires.

# Tomcat
Tomcat is a servlet container. It is configured in web.xml or later in WebAppInitializer class. 

 When we have backend server separated form front-end, backend server only provides response in Json format and its up to Front-end how it should look like. Previously Tomcat is used for both web server and servlet container, nowsaday it is only used for servlet container.

 Tomcat is still required in Spring boot as an embedded container because major purpose of Tomcat is hosting our servlets as a servlet container. 

 # pom.xml
 - starter dependencies: starter-web -> all dependencies required are provided. 
 - Springboot is an advancement of Spring MVC as it places importance on annotations rather xml code. @SpringBootApplication inclues 3 annotations: @Configuration , @EnableAutoConfiguration - tell SpringBoot to start adding beans based on classpath settings, auto config apps based on  dependencies added in the project, @ComponentScan to instruct SpringBoot to scan the current package and sub-packages for Spring Components to initiate beans.
 - Some dependencies like Loombok are <optional>true</optional> which means they will not be auto addd in the external library so other projects depending on this project will not auto inherit the optional dependencies. This helps to avoid pulling uneccessary dependencies in downstream project.

 # Dispatcher servlet
 Responsible for determining which of java code has to be executed given a path. Dispatcher servlet will only search beans allocated in classes annotated with @Controller. @Service and @Repository annotated classes are for internal uses

# Notes about Annotations
 - @RestController is equivalent to @ResponseBody + @Controller because, for example, if we return a String object, it is only understood internally in Java, but for client side we need to use html/text format.  This is handled by @ResponseBoddy annotation or @RestController. Using @ResponseBody, when we return a custom object, by default it returns json/xml format to the screen. 
 - @RequiredArgsConstructor: this allows us to only include properties which are mandatory and ignore unchangeable attributes like one with "final" keyword

 # HTTP methods
 ## Idempotency
 This means we get the same behaviours even on multiple identical calls. For example, DELETE method or GET method is idempotent.
It is recommended that except for POST, all other methods should be idempotent. 

- GET: return some data
- PATCH: update the resource, only replace certain parameters and not whole object
- PUT: either create new object if its not exist or replace the whole object if exists. In case we dont provide all parameters, the new objects will have those params as NULL. The ID of the object will not change after a PUT request because it is specified in the request path unless the ID is explicitly modified in the request body or in the update logic. Using PUT will results in replacing the entire resource so it is typically expects a full representation of the resource. It is recommended that for every layer, we should have certain validation for that using @Valid -> @NotBlank, etc.
- POST: new record will be auto generated and the client will not know about the object's id. We use POST method instead of PUT when id is unknown
# Safety vs Idempotency
It is not safe if we use any HTTP methods that able to modify the resource such as PUT, PATCH, DELETE,POST
# Static vs Templates folder
- Static folder: to store photos, html files that have no interactions with java codes
- Template folder: where view should be when your java code search for the view
# Maven scope
When adding any dependency into pom.xml we need to specify Maven scope
- Default scope: compile - this means dependency is only required at compile time as well as at runtime to run your project
- RUNTIME: e.g., mysql-connector-java is an intro driver used by JDBC, at compile time Spring doesnt know what implementation but when we run our program, it will find JDBC
- TEST: only need dependency for running our tests. E.g., JUnit dependency, mockito, assertion, verifying dependencies
- PROVIDED: required at compile time also at runtime. At compile time, dependency will be provided by the framework. 
- SYSTEM: At runtime, we have to specify the exact location path of the dependency.

# Logging
In spring, we use Logging instead of printing using System.out.print() to console because logging offers serveral advantages:
- Different Log levels: Logging frameworks(SLF4J, Logback,Log4j) provides various log levels that allow us to control the output. Staging requires different levels of logging, for example, level of gamma is DEBUG and level of producution is INFO. Not all logs required in staging are required in production which makes this logging configuration becomes handy. Level of logging based on their importance: 
**TRACE**: used for tracing program execution and understanding the flow of applicaiton
**DEBUG**: detailed information on the flow, more than TRACE. For example, we set the logging level to DEBUG to see detailed logs of each request, db query and service interaction:
```java
logger.debug("Query executed : SELECT * FROM persons WHERE id ={}",id);
```
**INFO**: general info shows the application's process. This is the default logging level
**WARN**: Indications of potential problems that do not current affect the applicaiton's normal operations
e.g., logger.warn("Disk space is running low");
**ERROR**: Serious issues requiring attention but do not require the apps to stop
e.g., logger.error("Failed to load configuration file",e);
**FATAL**: critical condition requires immediate attention and could lead to apps shutdown
e.g.,logger.fatal("system is out of memory,shutting down");

During development, DEBUG or TRACE can get detailed info for troubleshooting.

In production, WARN and ERROR reduce noise and focus on significant issues. Tools can be configured to trigger actions based on specific log levels

If default level is INFO, only loggers with importance level = INFO and above is printed (WARN, ERROR,FATAL). This is specified in application.properties : e.g., logging.level.root = debug. We can also specify for a particular file or package instead of root: e.g., logging.level.org.gfg.LogDemo = error

However, logging too many details can impace apps performance. 
- A filename can be specified in application.properties to records all logs when we run our application: 
```
logging.file.name = test.log
```
A rolling strategy can be applied where log files are managed and rotated over time. This ensures that log files do not grow indefinetely, consuming disk space. 
**time-based rolling**: e.g., log files made >24hrs before will be compressed to make an archive file
**size-based rolling**: e.g.,rotate logs when it reaches 10MB
**time-size based**
- Logging framework allows separation between business logic and logging. Some logging supports asynchronous which block the execution while writing log messages
- We can integrate with log aggregation tool (e.g., ELK stack- Elasticsearch,LogStash, Kibana) for centralised logging and analysis

# Spring project demo

Now we are going to build a simple Spring project in which database is connected via Connetion class and manually defined DataSource. 
To avoid manual configuration via web.xml or Java config, we are going to use SpringBoot web-starter dependency. 
Other dependencies we have: 
- Loombok
- HikariCP
- Spring-jdbc
- mysql-connector-j

Notice that Spring JDBC does not provide a way to manage connection pool, it uses external dependency which is Hikari to manage. This is to avoid multiple requests dependent on one single connection. Combination of HikariCP and Spring-jdbc is equivalent to spring-boot-starter-jdbc in Spring Boot

1. We create simple model named Person and use Loombok annotations to reduce boiler codes: 

```java
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;

@AllArgsConstructor
@Getter
@Setter
public class Person {
	String name;
	Integer id;

}
```
2. Create database in mysql named JDBC_demo
3. Now we need to connect java code to the database we just created. This will be handled by PersonRepository class. Here it is annotated with @Repository and includes methods to extract data from JDBC_demo database, or create table if not exist. Each method in PersonRepository requires an instance of Connection class and we dont want to call it everytime we write a method in Repository. Hence, we create a separate class to define the bean of Connection class. An instance of connection will further require an instance of DataSource class for it to function. 
```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

import javax.sql.DataSource;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

@Configuration
public class DBBean {
	@Bean
	public Connection connection() {
		Connection connection = null;
		
		try {
			connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/JDBC_demo","root","root");
		} catch (SQLException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return connection;
	}
	
	@Bean
	public DataSource dataSource() {
		DriverManagerDataSource dataSource = new DriverManagerDataSource();
		dataSource.setUrl("jdbc:mysql://localhost:3306/JDBC_demo");
		dataSource.setUsername("root");
		dataSource.setPassword("root");
		return dataSource;
	}	
	
}
```
@Bean is method level annotation, different to class annotations such as @Controller, @Service, @Repository or @Component. Hence, we need @Configuration annotation to mark the class and notify Spring to scan this bean at runtime. 

```java
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Repository;

import com.demo.SpringJDBC.Model.Person;

@Repository
public class PersonRepository {

	Connection connection;

	@Autowired
	PersonRepository(Connection connection) {
		this.connection = connection;
		// DB connection
		createTable();
	}

	public List<Person> getPersons() {
		List<Person> personList = new ArrayList<Person>();
		try {

			// get result from DB
			ResultSet resultSet = connection.createStatement().executeQuery("select * from Person");

			// turn into Java object
			while (resultSet.next()) {
				Person person = new Person(resultSet.getString("name"), resultSet.getInt("id"));
				personList.add(person);
			}
			return personList;

		} catch (SQLException e) {
			throw new RuntimeException(e);

		}

	}

	private void createTable() {
		try {
			connection.createStatement().executeUpdate("create table if not exists person(name varchar(20),id int)");
		} catch (SQLException e) {
			// TODO Auto-generated catch block
			throw new RuntimeException(e);

		}
	}
}
```
Running above code, we can see table person created on our database JDBC_demo which means database connection is successful. 

4. Now we have PersonService and PersonController classes to receive requests from client. Here we want to add a new person record. In PersonRepository, we create another createPerson method: 

```java
public Integer createPerson(Person person) {
		// TODO Auto-generated method stub
		int rowsAffected = 0;
		try {
			rowsAffected =  connection.createStatement().executeUpdate(
					"insert into person(name,id) values (' "+person.getName()+" ','"+ person.getId()+ ")");
		} catch (SQLException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return rowsAffected;
	}
```
Methods of connection.createStatement() include: 
- Execute: returns true if some ResultSet is returned, otherwise false
- ExecuteUpdate: returns an integer which is the number of rows impacted due to the query
- ExecuteQuery: returns data which we need to assign it to ResultSet which is a class to store data. 
Another way to simplify above query is to use PreparedStatement class to construct query. This also allows us to add certain dynamic params in our query. For example, the ExecuteUpdate() above will be re-written as follows: 

```java
PreparedStatement ps = connection.ps("insert into person(name,id) values (?,?)");
ps.setString(1,person.getName());
ps.setInt(2,person.getId());
return ps.executeUpdate();
```
However, we still need to follow the correct order of insertion and index it accordingly. 

**JDBC TEMPLATE**

- Using Connection class requires manually handle DriverManager. Spring framework provide JDBC templates to simplify DB operations. Hence, we no longer need a bean of Connection class. In repository, the code is re-written: 

```java
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

import com.demo.SpringJDBC.Model.Person;

import lombok.extern.slf4j.Slf4j;

@Repository
@Slf4j
public class PersonRepositorySpringJDBC implements IPersonRepository {

	JdbcTemplate jdbcTemplate;

	@Autowired
	PersonRepositorySpringJDBC(JdbcTemplate jdbcTemplate) {
		this.jdbcTemplate = jdbcTemplate;
	}

	@Override
	public List<Person> getPersons() {

		List<Person> personList = jdbcTemplate.query("select * from person", new RowMapper<Person>() {

			@Override
			public Person mapRow(ResultSet rs, int rowNum) throws SQLException {
				return new Person(rs.getString("name"), rs.getInt("id"));
			}
		});
		return personList;

	}
```
- Previously, we need to initiate an empty personList, then extract data from DB and assign it to a ResultSet, then iterate through each line in ResultSet using while loop to create a new Person object. JDBC Templates returns type is a list, hence we dont need to intiate an empty list. It takes 2 parameters: the sql query, and instance of RowMapper functional interface where you can override mapRow() method to return Person object. 

Shorter version with lambda expression: 
```java
@Override
	public List<Person> getPersons() {

		List<Person> personList = jdbcTemplate.query("select * from person", (rs,rowNum)-> new Person(rs.getString("name"), rs.getInt("id")

		));
		return personList;

	}
```
Note: we have to make sure that mapping order is correct which is the same issue we met with PreparedStatement. In order to remove that particular issue, we use a child class of JDBC Template: NamedParameterJdbcTemplate

```java
@Override
	public Integer createPerson(Person person) {
		MapSqlParameterSource parameterSource = new MapSqlParameterSource();
		parameterSource.addValue("name", person.getName());
		parameterSource.addValue("id", person.getId());
		
		return jdbcTemplate.update("insert into person(name,id) values(:name,:id)",parameterSource );

	}
```
# @Qualifier annotation

Lets say we have two implementations of PersonRepository, one using Connection and the other using JDBC Template, both implements IPersonRepository interface. In order to let Spring know which version to use, we use @Qualifier annotation. This annotation is used whenever there is a conflict of which bean to be used. We can add name to @Bean("dataS") and use it in @Qualifier as well. 

```java
@Service
public class PersonService {
	
	IPersonRepository ipersonRepository;
	
	@Autowired
	PersonService(@Qualifier("personRepositoryJDBC")IPersonRepository iPersonRepository){
		this.ipersonRepository = iPersonRepository;
	}
	
	public List<Person> getPersons() throws SQLException{
		return ipersonRepository.getPersons();
	}
```
Note about default naming convention as argument of @Qualifier: 
- If first 2 letters are consecutively uppercase, the class name remains the same 
- If only first letter is uppercase, we will write it as lowercase

# Spring Boot

If we use Spring Boot, there are few differences: 
1. Spring boot dependency: spring-boot-starter-jdbc 
2. Instead of specifying DataSource bean in DBBean class, we now use application.properties configuration. Spring Boot will auto create a bean of the DataSource
3. Update and execute Query becomes much simpler: previously we write lots of boilerplate code for queries to map the data returned from SQL to JAVA POJOs. This now will be handled by Object Relational Mapping (ORM). Object means Java POJOs, Relational means Relational model. 
