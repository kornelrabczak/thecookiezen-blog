---
layout: post
title: "WildFly-Swarm - New player in microservices world"
date: 2016-01-10 17:43:04 +0100
comments: true
description: WildFly-Swarm - New player in microservices world
keywords: wildfly, jboss, microservices, swarm, java, javaee
categories: [java, wildfly, microservices, javaee]
---

-> {% img center-displayed /images/swarm_logo.png 'image' 'images' %} <-

Earlier in 2015 Red Hat made initial release of their new project called WildFly Swarm. This solution allows to run JavaEE based applications as microservices. It's a competitor for many frameworks like Spring-boot or Dropwizard which are used for creating small application based on microservices architecture very fast. The application server now can be embedded into your application jar with all his dependencies or you can deploy your applications war into standalone WildFly application server instance as before. It's a big step for JavaEE world and I hope that it will silence all people who were complaining about JavaEE fat application servers.

<!-- more -->

WildFly-Swarm is decomposed into small modules with a good separation of concerns. With those modules we can fit server dependencies to our own requirements. Besides standard modules, there are a few modules which can help you maintaining your applications and integrate them into your environment. For example Jolokia used to exposing JMX data as JSON over HTTP for monitoring purpose or Netflix Ribbon which is for software load balancing. With WildFly-Swarm we have all JavaEE toolkit setup and standardization, which has a big impact on the application maintenance. Additionally, we can test our application using Arquillian adapter, which helps us test code from outside and inside of the running application.

Full modules list :

*   Logging
*   Logstash
*   JAX-RS
*   Weld (CDI)
*   Management
*   JSF
*   Messaging
*   Clustering
*   Infinispan
*   Keycloak
*   Netflix Ribbon
*   Hawkular
*   Jolokia

Code example
---------------------

First we need wildfly-swarm-plugin definition in our pom.xml

{% codeblock lang:xml pom.xml %}
<plugins>
  <plugin>
    <groupId>org.wildfly.swarm</groupId>
    <artifactId>wildfly-swarm-plugin</artifactId>
    <executions>
      <execution>
        <goals>
          <goal>package</goal>
        </goals>
      </execution>
    </executions>
  </plugin>
</plugins>
{% endcodeblock %}

Our example application will be simple REST resource. That's why we need JAX-RS dependency only

{% codeblock lang:xml pom.xml %}
<dependency>
  <groupId>org.wildfly.swarm</groupId>
  <artifactId>wildfly-swarm-jaxrs</artifactId>
</dependency>
{% endcodeblock %}

also standard JAX-RS configuration class

{% codeblock lang:java JAXRSConfiguration %}
import javax.ws.rs.ApplicationPath;
import javax.ws.rs.core.Application;

@ApplicationPath("resources")
public class JAXRSConfiguration extends Application {
}
{% endcodeblock %}

and sample resource for testing

{% codeblock lang:java ExampleResource %}
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;

@Path("hello")
public class ExampleResource {

    @GET
    @Path("/{name}")
    public String hello(@PathParam("name") String name) {
        return "Hello, " + name + "!";
    }
}
{% endcodeblock %}

Running as WAR
---------------------

For running application we can package our compiled code into WAR and deploy on server or simpy use widly-swarm-plugin

{% codeblock lang:ruby %}
mvn wildfly-swarm:run
{% endcodeblock %}

a minimal version of WildFly will be started with only the modules that are needed. Our application is running, now we can test resource url localhost:8080/resources/hello/John

Running as JAR
---------------------

We have to define main class where we will configure the container and our deployment archives. We are building deployment using ShrinkWrap tool which is open source project for easy creating, importing, exporting and manipulating archives.  

{% codeblock lang:java Application %}
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.wildfly.swarm.container.Container;
import org.wildfly.swarm.jaxrs.JAXRSArchive;

public class Application {

    public static void main(String[] args) throws Exception {
        Container container = new Container();

        container.start();

        JAXRSArchive deployment = ShrinkWrap.create(JAXRSArchive.class);
        deployment.addPackage(Application.class.getPackage());
        deployment.addAllDependencies();

        container.deploy(deployment);
    }

}
{% endcodeblock %}

Also we need to change packaging type for jar

{% codeblock lang:xml %}
<packaging>jar</packaging>
{% endcodeblock %}

and additional wildfly-swarm-plugin configuration where we point to our main class

{% codeblock lang:xml %}
<configuration>
    <mainClass>com.thecookiezen.example.Application</mainClass>
</configuration>
{% endcodeblock %}

{% codeblock lang:ruby %}
mvn package
{% endcodeblock %}

then we get fat jar with all dependencies, which we now run using java -jar command

{% codeblock lang:ruby %}
java -jar target/wildfly-swarm-example-1.0-SNAPSHOT-swarm.jar
{% endcodeblock %}

and check test url localhost:8080/resources/hello/John

[Full example source code](https://github.com/kornelrabczak/wildfly-swarm-example)

Summary
---------------------

For a long time there was no real competitor for Spring Boot. There were a lot of frameworks but none could compete with them in terms like monitoring, maintenance and productivity. I think WildFly-Swarm is future of JavaEE and microservices. Now it’s going to be much simpler to build JavaEE applications for running in the microservices environment.