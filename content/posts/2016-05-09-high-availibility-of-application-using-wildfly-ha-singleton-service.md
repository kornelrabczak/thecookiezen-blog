---
layout: post
title: "High Availibility of application using Wildfly HA Singleton Service"
featuredImage: "/images/ha.jpg"
author: "Korneliusz Rabczak"
date: 2016-05-09 19:22:46 +0200
comments: true
description: High Availibility of application using Wildfly HA Singleton Service
keywords: high availibility, wildfly, singleton service
categories: [high availibility, wildfly, singleton service] 
---



Why should I provide HA for my application?
---------------------

If we are creating application for some client or for ourselves we need at some point to deploy our work on production servers and expose to the world. For most cases it's obvious that our application cannot work as one instance. We need multiple instances of our service in case of failure of one of our servers. Also the network can fail. Also the accidence of high load or some other problems which might suddenly surprise us. Therefore we have to think about High Availability much earlier just not to fail and to be awakened during the weekend.

<!-- more -->

High availability eliminates SPOF (single point of failure) in our system. Failure of a single component that belongs to our system will not lead to failure of the whole system. We can think about HA as of a characteristic to our system, which main goal is to ensure specified performance of the system during a large load or some failure independent from our application.

What should be highly available in my application?
---------------------

There are several configurations of High availability. The most common are active/active and active/passive.

- Active/Active - whole traffic from failed node is passed to some backup instance node or distributed into the remaining nodes. All nodes in the cluster are the same, so it requires software configuration to be homogeneous.
- Active/Passive - each node (mostly called master) must have redundant instance called slave, to which traffic is switched only in the event of failure of the primary node.

Today we will focus only on Active/Passive called also Active/Standby. We’re taking as an example the application that has some important worker, which all the time does some important job. Worker can only be in one instance at the time and our application must be high available under penalty of high fines. We should think about this worker as HA Singleton Service. And here it comes to us: a very useful solution based on JGroups/WildFly clustering - HA Singleton.


How to achieve High Availability in Active/Passive configuration using Wildfly module?
---------------------

First of all we need some maven dependencies in pom.xml

```xml
<dependency>
	<groupId>org.jboss.msc</groupId>
	<artifactId>jboss-msc</artifactId>
	<version>${jboss-msc.version}</version>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>org.wildfly</groupId>
	<artifactId>wildfly-clustering-common</artifactId>
	<version>${wildfly.version}</version>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>org.wildfly</groupId>
	<artifactId>wildfly-clustering-singleton</artifactId>
	<version>${wildfly.version}</version>
	<scope>provided</scope>
</dependency>
```

also manifest entries in maven-war-plugin configuration to tell JBoss Modules what modules of our application depend on making them available for our application

```xml
<manifestEntries>
	<Dependencies>
	  org.jboss.msc,
	  org.wildfly.clustering.singleton,
	  org.jboss.as.server
	</Dependencies>
</manifestEntries>
```

Next will be activator of our service. We need to pull ServiceRegister from the given context, create instance of our service that is configured as singleton service and install/register it. Default configuration value of quorum is 1, so only one node is needed to run our service as singleton. We also need to specify election policy, which in our case will be PreferredSingletonElectionPolicy based on the name of the node. Our preffered node name will be "node-1" and every time when new node joins into the cluster with this name it is marked as "active" and previous "active" node will be deactivated. Additionally, we create ServiceRegistryWrapper, which is utility class for keeping reference to ServiceRegistry instance for further purpose.

```java
public class HAServiceActivator implements ServiceActivator {

	private static final String CONTAINER_NAME = "server";
	private static final String CACHE_NAME = "default";
	private static final ServiceName FACTORY_NAME = SingletonServiceBuilderFactory.SERVICE_NAME.append(CONTAINER_NAME, CACHE_NAME);
	public static final String PREFERRED_NODE = "node-1";
	public static final ServiceName SINGLETON_SERVICE_NAME = ServiceName.JBOSS.append("ha", "singleton", "example");

	@Override
	public void activate(ServiceActivatorContext context) {
		InjectedValue<ServerEnvironment> env = new InjectedValue<>();
		HASingletonService service = new HASingletonService(env);
		final ServiceRegistry serviceRegistry = context.getServiceRegistry();
		ServiceRegistryWrapper.setServiceRegistry(serviceRegistry);
		ServiceController<?> factoryService = serviceRegistry.getRequiredService(FACTORY_NAME);
		SingletonServiceBuilderFactory factory = (SingletonServiceBuilderFactory) factoryService.getValue();
		factory.createSingletonServiceBuilder(SINGLETON_SERVICE_NAME, service)
			.electionPolicy(new PreferredSingletonElectionPolicy(new SimpleSingletonElectionPolicy(), new NamePreference(PREFERRED_NODE)))
			.build(context.getServiceTarget())
			.addDependency(ServerEnvironmentService.SERVICE_NAME, ServerEnvironment.class, env)
			.setInitialMode(ServiceController.Mode.ACTIVE)
			.install();
	}
}
```

HASingletonService is our implementation of service and there will be only one instance of this service in the cluster. We’re also keeping flag specifying status of our service.

```java
@Log4j
public class HASingletonService implements Service<String> {

	private final Value<ServerEnvironment> env;
	private final AtomicBoolean started = new AtomicBoolean(false);

	public HASingletonService(Value<ServerEnvironment> env) {
		this.env = env;
	}

	@Override
	public void start(StartContext startContext) throws StartException {
		if (!started.compareAndSet(false, true)) {
			log.warn("The service " + this.getClass().getName() + " is already active!");
		} else {
			// START YOUR IMPORTANT JOB
		}
	}

	@Override
	public void stop(StopContext stopContext) {
		if (!started.compareAndSet(true, false)) {
			log.warn("The service " + this.getClass().getName() + " is not active!");
		} else {
			// STOP YOUR IMPORTANT JOB
		}
	}

	@Override
	public String getValue() throws IllegalStateException, IllegalArgumentException {
		if (!this.started.get()) {
			throw new IllegalStateException();
		}
		return this.env.getValue().getNodeName();
	}
}
```

We still need a REST resource for exposing information if current node is master or a slave.It is done by comparing the current node name with the node name returned from singleton service, which can be active on different node.

```java

@Path("/healthcheck")
public class HealthCheckResource {

	private static final String UNDEFINED = "undefined";
	private String nodeName;

	@PostConstruct
	public void init() {
		nodeName = System.getProperty("jboss.node.name");
		if (nodeName == null) {
			nodeName = UNDEFINED;
		}
	}

	@GET
	public Response check() {
		final ServiceController<?> requiredService = ServiceRegistryWrapper.getServiceRegistry()
			.getRequiredService(HAServiceActivator.SINGLETON_SERVICE_NAME);
		final Service<?> service = requiredService.getService();
		final String masterNodeName = (String) service.getValue();
		return nodeName.equals(masterNodeName) ? Response.ok().build() : Response.status(Response.Status.GONE).build();
	}
}
```

Last thing that we want is to tell WildFly to invoke our custom activator. It's based on ServiceLoader API, where we need to provide a META-INF/services/org.jboss.msc.service.ServiceActivator file with fully qualified name of our activator class.

Now we can start multiple WildFly instances (we need to specify IP address for jgroups; 0.0.0.0 doesn’t work in this case)

```

./bin/standalone.sh -b `hostname -I` -Djboss.node.name=node-1
./bin/standalone.sh -b `hostname -I` -Djboss.node.name=node-2
./bin/standalone.sh -b `hostname -I` -Djboss.node.name=node-3

```

and shutdown/restart randomly nodes, that we can observe applications behavior in cluster. Additionally, we have HealthCheck endpoint, which tells its current node a master 200 or slave 410 HTTP response status. We can use this information to configure our load balancer order to direct the whole load traffic only to the master node.

[Full example source code](https://github.com/kornelrabczak/ha-singleton-service)