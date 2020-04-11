---
layout: post
title: "MemoryImage - Clean Architecture approach to storing data in your next application"
date: 2016-02-23 21:02:26 +0100
comments: true
description: MemoryImage - Clean Architecture approach to storing data in your next application
keywords: memory image, prevayler, clean architecture, airomem
categories: [prevayler, clean architecture, airomem]
---

-> {% img center-displayed /images/beczka.jpg 'image' 'images' %} <-

In this post I want to introduce simple solution that allows you to keep your application architecture clean. For those of you who haven't heard about Clean Architecture approach I recommend to read [Uncle Bobs article](https://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html). The main idea is to defer the decision of selecting storage engine we choose for our application, because the database is only a detail. It's not important if it will be SQL, NoSQL or graph database. A database in our application is only the detail which should be easy to change. Premature selection of a database can have negative influence for creating our domain model. For example if we use Hibernate which is JPA implementation and we start writing code from creating Hibernate entities, it will lead us then to high coupling between our core elements and ORM framework. Postponing decision about a database can change our point of view on domain model and stop future degradation of code base.

<!-- more -->

Problem
---------------------

I took part in multiple projects where choosing database was the first decision made just after the start of the project. No one knew the full specification of the project and what features needed to be written, but there was always a decision that our application would use Oracle or MySQL as a storage. Solutions such as Hibernate can lead to many problems like detached/attached entities, phantom reads, n+1 queries problem, transactions isolation problem, lazy initialization and also code smells that go beyond repository/entity layer. We, as responsible developers, can’t allow for some fancy framework to take control over our software. Maybe we don't need to store everything in a database. Maybe some part of our data can be loaded on application startup from file and stored in memory because it will never change. We need to slow down, wait for the proper moment and it may turn out that we don't need any database at all. 


Clean Architecture and Prevayler
---------------------

Now you already know about deferring decision of choosing a database and you wonder how we can achieve this goal. We can try to keep everything in files on disk or Maps in the memory. Those are easy things, but how we provide persistence of maps between application restarts ? There is an old pattern called Prevayler or MemoryImage. You can find more detailed description on [Martin Fowler's site](http://martinfowler.com/bliki/MemoryImage.html). The main idea is to keep everything in memory and make periodically snapshots of objects that need to be persisted. Snapshots are made at short intervals of time to prevent losing data. Between snapshots, there is the events log that keeps every action of our persisted object (in case of map will be put/remove/etc) logged in specified file. When we restart our application, first of all the snapshot is loaded. Then, all transactions from events log are replied to keep consistence. All operations are made on objects in memory.  Such solution leads to lack of need Session, EntityManager, complicated entities relationships, DB deadlocks. The problems related to ORMs are gone.

Airomem 
---------------------

Let me introduce you one of the solutions that can be used to achieve Clean Architecture. It's called [Airomem](https://github.com/airomem/airomem) and it's a Java persistence framework based on [Prevayler](http://prevayler.org/). Besides Prevayler engine, there are some extra features like Java 8 lambdas, Kryo based serialization and rich [CQEngine collections](https://github.com/npgall/cqengine). You need to remember that each object from your domain must be serializable and all operations that change domain must be enclosed in commands.
 
Let's code some example 
 
First we need to define a maven dependency in our pom.xml
 
{% codeblock lang:xml pom.xml %}
<dependency>
    <groupId>pl.setblack</groupId>
    <artifactId>airomem-core</artifactId>
    <version>1.0.2</version>
</dependency>
{% endcodeblock %}

 
than we create our simple domain model which is Book and Library. Book can has ISBN number, author and title. Of course our class must be serializable, if we want it to be persisted.
 
{% codeblock lang:java Book %}
public class Book implements Serializable {

    private static final long serialVersionUID = 1L;

    private final String ISBN;
    private final String author;
    private final String title;

    public Book(String isbn, String author, String title) {
        this.ISBN = isbn;
        this.author = author;
        this.title = title;
    }

    public String getISBN() {
        return ISBN;
    }

    public String getAuthor() {
        return author;
    }

    public String getTitle() {
        return title;
    }
}
{% endcodeblock %}

Next, we create our storage class, that will be Library. All books will be stored in a thread-safe variant of ArrayList - CopyOnWriteArrayList. Library has methods for creating and fetching books by ISBN, author or title.

{% codeblock lang:java Library %}
public class Library implements Serializable {

    private static final long serialVersionUID = 1L;

    private CopyOnWriteArrayList<Book> books = new CopyOnWriteArrayList<>();

    public void addNewBook(String title, String author) {
        assert WriteChecker.hasPrevalanceContext();
        final Book book = new Book(UUID.randomUUID().toString(), author, title);
        this.books.add(book);
    }

    public Collection<Book> getAll() {
        return this.books;
    }

    public Book getByISBN(String ISBN) {
        final Predicate<Book> bookPredicate = b -> b.getISBN().equals(ISBN);
        return this.books.stream().filter(bookPredicate).findAny().get();
    }

    public Collection<Book> getByAuthor(String author) {
        final Predicate<Book> bookPredicate = b -> b.getAuthor().equals(author);
        return findByPredicate(bookPredicate);
    }

    public Collection<Book> getByTitle(String title) {
        final Predicate<Book> bookPredicate = b -> b.getTitle().equals(title);
        return findByPredicate(bookPredicate);
    }

    private Collection<Book> findByPredicate(Predicate<Book> bookPredicate) {
        return this.books.stream().filter(bookPredicate).collect(Collectors.toList());
    }
}
{% endcodeblock %}

Last class is implementation of REST resource to expose our books collections to outside.
 
{% codeblock lang:java BookResource %}
@Path("/books")
public class BookResource {

    private PersistenceController<DataRoot<Library , Library>, Library> controller;

    @PostConstruct
    public void init() {
        final PersistenceFactory factory = new PersistenceFactory();
        controller = factory.initOptional("library", () -> new DataRoot<>(new Library()));
    }

    @GET
    public Collection<Book> getAll() {
        return controller.query(Library::getAll);
    }

    @GET
    @Path("/author/{author}")
    public Collection<Book> getByAuthor(@PathParam("author") String author) {
        return controller.query(view -> view.getByAuthor(author));
    }

    @POST
    public void create(@QueryParam("title") String title, @QueryParam("author") String author) {
        controller.execute(view -> view.getDataObject().addNewBook(title, author));
    }
}
{% endcodeblock %}
 
In BookResource class we keep instance of PersistenceController which is responsible for querying and executing command on our Library storage. In *init* method we initialize a controller, in *getByAuthor* we fetch books by specified author and in *create* method new book is stored.

[Full example source code](https://github.com/kornelrabczak/airomem-example)

Conclusion
---------------------

Pros :

*   simple solution (KISS)
*   performance
*   no database code overhead
*   clean architecture without jpa frameworks taking control over a project
*   fast development
*   airomem-direct for JavaEE applications is just 2 annotations and interceptor

Cons :

*   no migration mechanism after changes in domain model, need to delete all data
*   lack of serializer/deserializers for common classes like URL in Kryo library
*   lack of tools for viewing data
*   no clustering/replication posibility