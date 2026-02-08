---
layout: post
title: "DeltaSpike Data - alternative for Spring Data JPA"
featuredImage: "/images/payara-deltaspike.jpg"
author: "Korneliusz Rabczak"
date: 2016-10-27 20:06:30 +0200
comments: true
description: DeltaSpike Data - alternative for Spring Data JPA
keywords: payara, spring, spring data, deltaspike, jpa, persistence
categories: [payara, spring, spring data, deltaspike, jpa, persistence]
---




Sometimes while attending a Java conference or reading some article on the internet, you can find out that many people think that Java EE is dead. It comes mostly from people strongly addicted to Spring Framework and its many cool features. They believe that Java EE is so 90' and Spring is the only right solution. I don't deny that Spring Framework is a great tool. Java EE is just the same tool which resolves the same problems, so we should have an open mind. One of these cool features of Spring is the ease to use a high level database abstraction called Spring Data JPA.
 
<!-- more -->
 
On the last Java event in Poland, which was JDD, I came across Apache DeltaSpike Data Module. This library offers us similar to Spring Data JPA functionality, like writing queries in @Query annotation or extending interfaces instead of implementing own DAO/Repository class. It's a great alternative for using Spring Data JPA and you can easily use it with Java EE.
 
 
Payara micro
---------------------
 
We will run our example on a lightweight, derived from GlassFish application server called Payara. Payara has a really nice feature called Payara-micro, which is similar thing to [WildFly-Swarm](http://thecookiezen.com/blog/2016/01/10/wildfly-swarm-new-player-in-microservices-world/). It gives us the possibility to embed Payara into our application jar archive and run application the same way as in Spring Boot. It's very helpful in the microservices environment. 
 
 
First, we need to add dependency for Payara module.
 
```xml
<dependency>
    <groupId>fish.payara.extras</groupId>
    <artifactId>payara-micro</artifactId>
    <version>4.1.1.162</version>
</dependency>
```
 
 
 
An additional configuration is needed for maven-dependency-plugin. You can find pom.xml file at [example repository](https://github.com/kornelrabczak/payara-delta-spike-data/blob/master/payara-micro/pom.xml).
 
Next, we should create bootstrap class with deployment description. [More details on Payara website.](http://payara.fish)
 
```java
 
public class ApplicationRunner {
 
    public static void main(String args[]) throws BootstrapException {
        PayaraMicroRuntime runtime = PayaraMicro.getInstance()
                .setHttpAutoBind(true)
                .bootstrap();
 
        runtime.deploy("delta-spike","/delta-spike", ApplicationRunner.class.getClassLoader().getResourceAsStream("war/delta-spike-example-1.0-SNAPSHOT.war"));
    }
}
 
```
 
To build and run our jar file
 
```bash
 
mvn clean install 
 
java -jar payara-micro/target/payara-micro-1.0-SNAPSHOT.jar
 
```
 
 
Apache DeltaSpike Data Module
---------------------
 
The Data Module is a simplification of data access layer, whereby we can write less boilerplate code. Things that normally create a lot of code duplications, now can be replaced by extending EntityRepository interface. It’s also the higher level of abstraction for querying database. EntityRepository interface provides common important methods to fetch and modify data inside our data store:

```java
    E save(E entity);
 
    void remove(E entity);
 
    void refresh(E entity);
 
    void flush();
 
    E findBy(PK primaryKey);
 
    List<E> findAll();
 
    List<E> findBy(E example, SingularAttribute<E, ?>... attributes);
 
    List<E> findByLike(E example, SingularAttribute<E, ?>... attributes);
 
    Long count();
 
    Long count(E example, SingularAttribute<E, ?>... attributes);
 
    Long countLike(E example, SingularAttribute<E, ?>... attributes);
```


Configuration
---------------------
 
At the beginning we set up Maven dependencies in our pom.xml
 
```xml
<dependency>
    <groupId>org.apache.deltaspike.modules</groupId>
    <artifactId>deltaspike-data-module-api</artifactId>
    <version>${deltaspike.version}</version>
    <scope>compile</scope>
</dependency>
<dependency>
    <groupId>org.apache.deltaspike.modules</groupId>
    <artifactId>deltaspike-data-module-impl</artifactId>
    <version>${deltaspike.version}</version>
    <scope>runtime</scope>
</dependency>
```

We need to configure DeltaSpike Data Module to work with Java EE before we start coding our example. Exposing EntityManager as a CDI bean using @Produces feature is the key to bind everything together.
 
```java
public class EntityManagerProducer {
    @PersistenceUnit
    private EntityManagerFactory emf;
 
    @Produces
    public EntityManager create() {
        return emf.createEntityManager();
    }
 
    public void close(@Disposes EntityManager em) {
        if (em.isOpen()) {
            em.close();
        }
    }
}
```
 
To finalize the configuration definition of transaction strategy for JTA, DataSource is needed.
 
```java
globalAlternatives.org.apache.deltaspike.jpa.spi.transaction.TransactionStrategy=org.apache.deltaspike.jpa.impl.transaction.BeanManagedUserTransactionStrategy
```
 
 
Example
---------------------
 
We start developing the domain model, which in our case is a simple Book entity with id, title, isbn, author and created attributes. It's a standard JPA entity class. @Id and @GeneratedValue annotations are used for marking, which attribute will be entity ID.
 
```java
@Entity
public class Book {
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
 
    private String title;
 
    private String isbn;
 
    private String author;
 
    private Date created = new Date();
 
    // getters & setters
}
```
 
Now it's time to introduce our BookRepository that persists a book to the database. BookRepository is a simple interface and it extends already mentioned EntityRepository interface. The id and entity type are specified as the generic parameters of EntityRepository. All the methods names from BookRepository interface will be parsed by DeltaSpike Data Module and processed into equivalent queries. There are many prefixes like 'findBy', 'findOptionalBy', 'findAll' that you can add to your method name. Entity attribute name is used to create a query condition. We can concatenate multiple conditions using 'And' or 'Or'. The return type must either be an entity, list of entities or Optional of entity.
 
Query Method Expression is good for simple queries, but if we have some more sophisticated case, there is a Query Annotation. Using @Query we can declare named queries for entities outside the domain model. @Query annotation works on methods, so we can tie them together. With this solution we are able to separate named queries from entity class.
 
```java
@Repository
public interface BooksRepository extends EntityRepository<Book, Long> {
 
    List<Book> findByTitle(String title);
 
    List<Book> findByTitleAndAuthor(String title, String author);
 
    List<Book> findByTitleAndAuthorOrderByCreatedDesc(String title, String author);
 
    @Query("select b from Book b where b.isbn = ?1")
    Optional<Book> findByISBNNumber(String ISBN);
}
```
 
 
Below, there are presented methods of the repository interface and generated from them queries.
 
```java
    List<Book> findByTitle(String title);
 
      SELECT ID, AUTHOR, CREATED, ISBN, TITLE FROM BOOK WHERE (TITLE = ?)
        bind => [1 parameter bound]]]
 
```
 
```java
    List<Book> findByTitleAndAuthor(String title, String author);
 
      SELECT ID, AUTHOR, CREATED, ISBN, TITLE FROM BOOK WHERE ((TITLE = ?) AND (AUTHOR = ?))
        bind => [2 parameters bound]]]
 
```
 
```java
    List<Book> findByTitleAndAuthorOrderByCreatedDesc(String title, String author);
 
      SELECT ID, AUTHOR, CREATED, ISBN, TITLE FROM BOOK WHERE ((TITLE = ?) AND (AUTHOR = ?)) ORDER BY CREATED DESC
        bind => [2 parameters bound]]]
 
```
 
```java
    @Query("select b from Book b where b.isbn = ?1")
    Optional<Book> findByISBNNumber(String ISBN);
 
      SELECT ID, AUTHOR, CREATED, ISBN, TITLE FROM BOOK WHERE (ISBN = ?)
        bind => [1 parameter bound]]]
 
```
 
 
There are many more features of DeltaSpike Data Module. You can read about it [here](https://deltaspike.apache.org/documentation/data.html).
 
 
Conclusion
---------------------
 
DeltaSpike Data Module is a great extension for simple CRUD applications. You should still use Java EE, because it's a great and evolving technology. Even if it doesn't provide all the cool and fancy features, you can always find something for yourself. Don't rush to migrate your project from Java EE to Spring Framework. Switching to another technology depends on many factors, like: "Do you feel comfortable with your current technology?" or "Do you (your team) want to learn something new and experiment?" In my opinion, in commercial cases where you don't have time and space for mistakes, you should always use the technology that you and your team know best. In other cases: experiment and explore new solutions. It’s worth to learn new things!