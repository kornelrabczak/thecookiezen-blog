---
layout: post
title: "Exposing WildFly JMS queue statistics through REST/JSON for monitoring"
date: 2015-10-20 20:39:21 +0200
comments: true
description: Exposing WildFly JMS queue statistics through REST/JSON for monitoring
keywords: javaee, java, jms, monitoring, metrics, dropwizard. wildfly, jboss
categories: [java, jboss, jms, metrics]
---

-> {% img center-displayed /images/queue.jpeg 'image' 'images' %} <-

 Today I want to show you how easily you can monitor your JMS queues on WildFly 8 application server. JMS provides some statistics data like total added messages, messages pending in queue or number of consumers connected to queue. We can gather and analyze those data in our monitoring system. 

<!-- more -->

We could also create triggers for the purpose of alerting us when there are no consumers for the queue. This could easily end up with queue swelling thousands of messages. We will use Dropwizard Metrics library which is very nice and easy for gathering and measuring data in our application. We will expose this data through REST as JSON. We won't rely on JMX protocol because protocol used for providing data for monitoring should be technology agnostic. While providing data for monitoring system, we should use standard protocol for every technology, in our case, it will be HTTP.

Introducing  Metrics
---------------------

{% blockquote http://metrics.dropwizard.io %}
"Metrics is a Java library which gives you unparalleled insight into what your code does in production.
{% endblockquote %}
     
[Dropwizard Metrics](http://metrics.dropwizard.io) library is a part of Dropwizard Java framework. Metrics is very useful for code instrumentation. It provides us many measuring tools like meters, gauges, counters, histograms. Thanks to that we can easily monitor the behavior of our application. With some additional modules we have ready to use set of metrics for Jetty, Logback, Log4j, Apache HttpClient, Ehcache, JDBI, Jersey and many more. We can also send data gathered by Metrics to JMX, console, CSV or more advance reporting backends like Ganglia and Graphite. Main part of Metrics library is MetricRegistry instance with stores a collection of all the metrics from our application. We need mostly only one instance per JVM.  

Defining dependencies
---------------------

Firstly, we need to define Metrics dependencies in our pom.xml:

*   metrics-core provides basic functionality like metrics registry, main metrics types (gauges, counters, histograms, meters and timers) and possibility of reporting to many sources like JMX, console, CSV files or SLF4J logger.

{% codeblock lang:xml %}
<dependency>
    <groupId>io.dropwizard.metrics</groupId>
    <artifactId>metrics-core</artifactId>
    <version>3.1.0</version>
</dependency>
{% endcodeblock %}

*   metrics-servlets provides set of ready to use servlets like PingServlet for checking if application does return OK responses, HealthCheckServlet for checking registered health checks and, of course, MetricsServlet which is most important for us in this tutorial because it exposes the state of all metrics

{% codeblock lang:xml %}
<dependency>
    <groupId>io.dropwizard.metrics</groupId>
    <artifactId>metrics-servlets</artifactId>
    <version>3.1.0</version>
</dependency>
{% endcodeblock %}

ContextListener
---------------------

Next we will need to extend ContextListener from MetricsServlet and inject MetricRegistry instance to the newly created class into field and return it in overriden method. It provides MetricRegistry instance to the MetricsServlet.  

{% codeblock lang:java MetricsContextListener %}
package com.thecookiezen.monitoring.boundary;

import com.codahale.metrics.MetricRegistry;
import com.codahale.metrics.servlets.MetricsServlet;

import javax.inject.Inject;

public class MetricsContextListener extends MetricsServlet.ContextListener {

    @Inject
    private MetricRegistry metricRegistry;

    @Override
    protected MetricRegistry getMetricRegistry() {
        return metricRegistry;
    }
}
{% endcodeblock %}

web.xml configuration
---------------------

Now we should create web.xml descriptor and register our servlet context listener. We also need to register MetricsServlet for providing JSON. For this example we will use "monitoring" path.

{% codeblock lang:xml web.xml %}
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
         http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">

    <servlet>
        <servlet-name>metrics-admin</servlet-name>
        <servlet-class>com.codahale.metrics.servlets.MetricsServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>metrics-admin</servlet-name>
        <url-pattern>/monitoring/*</url-pattern>
    </servlet-mapping>

    <listener>
        <listener-class>com.thecookiezen.monitoring.boundary.MetricsContextListener</listener-class>
    </listener>


</web-app>
{% endcodeblock %}

Creating own metrics
---------------------

It's time for the most important part. Implementation of MetricSet interface that will return Map<String, Metric> with all our metrics. HornetQ is JMS implementation build in WildFly and all information and statistics are stored inside MBeanServer. We are going to access those MBeans to get necessary attributes.

{% codeblock lang:java JmsMetricsSet %}
package com.thecookiezen.monitoring.boundary;

import com.codahale.metrics.JmxAttributeGauge;
import com.codahale.metrics.Metric;
import com.codahale.metrics.MetricSet;

import javax.management.MBeanServer;
import javax.management.MalformedObjectNameException;
import javax.management.ObjectName;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

import static com.codahale.metrics.MetricRegistry.name;
import static java.util.Collections.EMPTY_MAP;

public class JmsMetricsSet implements MetricSet {

    private static final String[] ATTRIBUTES = {"messageCount", "messagesAdded"};
    private static final String JBOSS_MESSAGING_RUNTIME_QUEUE_PATTERN = "jboss.as:subsystem=messaging,hornetq-server=default,runtime-queue=jms.queue.*";

    private final MBeanServer mBeanServer;

    public JmsMetricsSet(MBeanServer mBeanServer) {
        this.mBeanServer = mBeanServer;
    }

    @Override
    public Map<String, Metric> getMetrics() {
        final Set<ObjectName> queues;
        try {
            queues = findQueues();
        } catch (MalformedObjectNameException e) {
            return EMPTY_MAP;
        }

        Map<String, Metric> gauges = new HashMap<>();
        for (ObjectName queue : queues) {
            String queueName = queue.getKeyProperty("runtime-queue");
            for (final String attribute : ATTRIBUTES) {
                gauges.put(name(queueName, attribute), new JmxAttributeGauge(mBeanServer, queue, attribute));
            }
        }
        return Collections.unmodifiableMap(gauges);
    }

    private Set<ObjectName> findQueues() throws MalformedObjectNameException {
        return mBeanServer.queryNames(new ObjectName(JBOSS_MESSAGING_RUNTIME_QUEUE_PATTERN), null);
    }
}
{% endcodeblock %}

We will take apart JmsMetricsSet class and analyze most important code fragments :

{% codeblock lang:java %}
private static final String JBOSS_MESSAGING_RUNTIME_QUEUE_PATTERN = "jboss.as:subsystem=messaging,hornetq-server=default,runtime-queue=jms.queue.*";
{% endcodeblock %}

This is pattern for names of queues in WildFly under which are stored information in JMX that we want to pull from MBeanServer.
 
{% codeblock lang:java %}
private static final String[] ATTRIBUTES = {"messageCount", "messagesAdded"};
{% endcodeblock %}

Queue attributes that we want to expose as JSON for monitoring.
 
{% codeblock lang:java %}
private final MBeanServer mBeanServer;
{% endcodeblock %}

MBeanServer stores registered MBeans, which are managed Java objects. MBeanServer instance will help us finding necessary registered MBeans related with JMS statistics.

{% codeblock lang:java %}
mBeanServer.queryNames(new ObjectName(JBOSS_MESSAGING_RUNTIME_QUEUE_PATTERN), null);
{% endcodeblock %}

Now we need to query MBean server using previously defined pattern to get a set of ObjectName objects. Method "queryNames" called on MBeanServer returns actual names of MBeans specified by pattern matching on the ObjectName.

{% codeblock lang:java %}
Map<String, Metric> gauges = new HashMap<>();
for (ObjectName queue : queues) {
    String queueName = queue.getKeyProperty("runtime-queue");
    for (final String attribute : ATTRIBUTES) {
        gauges.put(name(queueName, attribute), new JmxAttributeGauge(mBeanServer, queue, attribute));
    }
}
return Collections.unmodifiableMap(gauges);
{% endcodeblock %}

We are creating a map where the key is a concatenation of queue name and attribute, due to use of MetricRegistry.name() method which is a static helper method from MetricRegistry for generating unique names. The value is  JmxAttributeGauge instance which takes as a constructor parameters : reference to MBeanServer, ObjectName instance and attribute name.
Briefly: JmxAttributeGauge is Gauge implementation which queries an MBean server for an attribute of an object. 


Glue everything
---------------------

In the end we are creating factory method that will return singleton instance of MetricRegistry and bridge MetricRegistry with our ContextListener. For this solution we will use CDI annotation @Produces on method which acts as a source of objects to be injected when @Inject annotations occure. Also we need @ApplicationScoped for object which is created once for the duration of the application lifetime. In MetricRegistry instance we must register our JmsMetricsSet with platform MBean server. 

{% codeblock lang:java MetricsRegistryFactory %}
package com.thecookiezen.monitoring.boundary;

import com.codahale.metrics.MetricRegistry;

import javax.enterprise.context.ApplicationScoped;
import javax.enterprise.inject.Produces;
import java.lang.management.ManagementFactory;

public class MetricsRegistryFactory {

    @Produces
    @ApplicationScoped
    public MetricRegistry createMetricRegistry() {
        final MetricRegistry metricRegistry = new MetricRegistry();
        metricRegistry.registerAll(new JmsMetricsSet(ManagementFactory.getPlatformMBeanServer()));
        return metricRegistry;
    }
}
{% endcodeblock %}

In our beans.xml  we need also to add scanning control attribute bean-discovery-mode with value "all"  because default value "annotated" recognizes  only annotated CDI managed beans. Beans without any annotation will be ignored. 

{% codeblock lang:xml beans.xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/beans_1_1.xsd"
       bean-discovery-mode="all">
</beans>
{% endcodeblock %}

The final result
---------------------

Fire up WildFly server. Compile code. Package into war and deploy. Now we can test our metrics endpoint in a browser that we defined in web.xml. After that we receive JSON with our statistics of queues. This JSON can be consumed by various monitoring systems like Zabbix or Nagios.

[~] curl -XGET http://localhost:8080/appname.war/monitoring?pretty=true

{% codeblock lang:json %}
{
    "version": "3.0.0",
    "gauges": {
		"jms.queue.exampleQueue1.messageCount": {
		    "value": 0
		},
		"jms.queue.exampleQueue1.messagesAdded": {
		    "value": 0
		},
		"jms.queue.exampleQueue2.messageCount": {
		    "value": 0
		},
		"jms.queue.exampleQueue2.messagesAdded": {
		    "value": 0
	    }
    },
    "counters": { },
    "histograms": { },
    "meters": { },
    "timers": { }

}
{% endcodeblock %}