---
layout: post
title: "Data synchronization - part 1 : Service Discovery and Leader election"
date: 2016-12-12 11:00:00 +0100
comments: true
description: Data synchronization - part 1 - Service Discovery and Leader election
keywords: zookeeper, nginx, curator, high availibility, data synchronization
categories: [zookeeper, nginx, curator, data synchronization]
---
                                                                                                                                                                                                                                  
-> {% img center-displayed /images/ha-1.jpg 'image' 'images' %} <-
                                                                                                                                                                                                                                  
 This blog post will be the first part of the bigger proof of concept related to the data replication in distributed environment, which I'm working on. At the beginning, I will show how to separate GET requests from POST having multiple instances of a single application. The goal is to redirect all of the POST requests to the master instance, all GET requests should be distributed across all nodes. Today's showcase will be focused on three subjects: single master - multiple slaves communication model, leader election between multiple application nodes, and using Service Discovery for automatic load balancer configuration. In this part we're finding out how to simply use [ZooKeeper](https://zookeeper.apache.org/) for Service Discovery and Leader Election,  [NGINX](https://nginx.org/) as a load balancer with dynamically changing configuration by [confd](https://github.com/kelseyhightower/confd). NGINX is a web server, which can act as reverse proxy, load balancer or HTTP cache. In our case NGINX will be responsible for traffic load balancing to the appropriate nodes.
                                                                                                                                                                                                                                  
<!-- more -->

-> {% img center-displayed /images/ha-2.jpg 'image' 'images' %} <-

When we are deploying our application at the production environment, we should think about how to avoid the single point of failure and how to scale our system. When one of the nodes fails, the rest of the system should work properly without any downtime. This characteristic of the system is called high availability. To achieve high availability we can run multiple instances of our application and distribute high load traffic across our nodes. Load balancing server needs to know where all of the application instances are located. The best option is to use Service Discovery solution. Service Discovery detects changes in application's availibility, taking place over a computer network. 

##ZooKeeper

Apache ZooKeeper is a centralized key-value store tool for maintaining configuration, naming and distributed synchronization. ZooKeeper also provides Service Discovery, which helps to build applications in distributed environment. Multiple nodes rely there on communication using load balancing and Service Discovery for finding and talking to one another. This solution takes away a large and complex work overhead from the developers. 


###Curator

 ZooKeeper provides a fairly low level of primitive features, which require complex recipes and a lot of code. Because of the complexity, a higher level of abstraction library - Curator was created, for simplifying developer's life.

-> {% img center-displayed /images/ph-quote.png 'quote' 'quote' %} <-

Curator is a set of Java libraries that makes use of Zookeeper even easier. It provides multiple implementations of the important ZooKeeper recipes used in distributed environment like: leader election, distributed locking, caching or barriers. Curator handles ZooKeeper complexity issues through: supporting retry mechanism, connection state monitoring and ZooKeeper instance management.

####Leader Election

In distributed environment, leader election process is a coordinated action of workers collection by electing one instance as the leader. This pattern resolves possible conflicts that may arise between the instances. At the beginning all application nodes are unaware which node is the leader. After election, information about the leader is distributed across the network. In our case the leader will be responsible for  accepting all of the POST request type messages.

{% codeblock lang:java ClusterStatus %}

@Singleton
public class ClusterStatus implements LeaderLatchListener {

	// zookeeper configuration fields 
   
    private CuratorFramework client;
    
    private LeaderLatch leaderLatch;

    @PostConstruct
    public void init(@Observes @Initialized(ApplicationScoped.class) Object ignore) {
		client = CuratorFrameworkFactory.newClient(zookeeperConnection, new ExponentialBackoffRetry(1000, 3));
		client.start();

		try {
		    client.blockUntilConnected();

		    leaderLatch = new LeaderLatch(client, latchPath, nodeId);
		    leaderLatch.addListener(this);
		    leaderLatch.start();
		} catch (Exception e) {
		    log.error("Error when starting leaderLatch", e);
		}
    }

    public boolean hasLeadership() {
		return leaderLatch.hasLeadership();
    }

    @Override
    public void isLeader() {
		log.info("Node : " + nodeId + " is a leader");
    }

    @Override
    public void notLeader() {
		log.info("Node : " + nodeId + " is not a leader");
   	}

    @PreDestroy
    public void close() {
		try {
		    leaderLatch.close();
		} catch (IOException e) {
		    log.error("Error when closing leaderLatch.", e);
		}
		client.close();
    }
}

{% endcodeblock %}


- Lines 10-20: we are creating and starting CuratorFramework client instance, next we need to block current thread until a connection to ZooKeeper is available; after connection is established, we can create and start LeaderLatch instance. Once LeaderLach has started, negotiation process begins with others LeaderLatch instances that share same ZooKeeper path. Election process ends, when one of the Leaderlatch instances is randomly chosen to be a leader.
- Line 27: return true if leadership is currently held by this instance
- Line 31-38: we are implementing LeaderLatchListener methods for asynchronously notification about when the state of the LeaderLatch has changed.
- Line 43: we are releasing leadership, now another LeaderLatch instance can be chosen for a leader.


{% codeblock lang:javascript console %}
{% raw %}

./bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: ~/tools/zookeeper-3.4.9/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-1 -Dport=8070 -jar target/high-availability-active-passive-microservices
-1.0-SNAPSHOT.jar

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-2 -Dport=8080 -jar target/high-availability-active-passive-microservices
-1.0-SNAPSHOT.jar

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-3 -Dport=8090 -jar target/high-availability-active-passive-microservices
-1.0-SNAPSHOT.jar

INFO  com.thecookiezen.microservices.infrastructure.status.ClusterStatus  Node : node-1 is a leader [main-EventThread]

~/tools/zookeeper-3.4.9/bin/zkCli.sh -server 127.0.0.1:2181

[zk: 127.0.0.1:2181(CONNECTED) 3] ls /example/leader
[_c_99b5c4b4-2940-4b9d-b2a0-a1ac972eaa67-latch-0000000001, _c_8978e7f0-2079-4e10-a8d3-a645a4b7f681-latch-0000000000, _c_076cddfe-e575-4e0b-ac5a-bf4dfe03be42-latch-0000000002]

curl http://localhost:8070/microservice/test
true

curl http://localhost:8080/microservice/test
false

curl http://localhost:8090/microservice/test
false

{% endraw %}
{% endcodeblock %}

Now we need to check if everything works as we wanted. After starting ZooKeeper server and 3 instances of our application on different ports, we can see in logs that one of the instances has become a leader. We can also check using the ZooKeeper CLI which instances were registered under the path /example/leader. Additionally, the application exposes information on whether is the leader or not through REST interface. 

####Service Discovery

In such a distributed environment multiple different applications that are constantly changing need a way to find each other. With Service Discovery you can find up services, which are running in the topology of your network. Service Discovery is a mechanism for applications to register their actual state(health, availability), locating a single instance of a service and notifying on any services change. 

{% codeblock lang:java ClusterStatus %}
@Singleton
public class ClusterStatus implements LeaderLatchListener {

    // zookeeper configuration
    
    private CuratorFramework client;
    private LeaderLatch leaderLatch;
    private ServiceDiscovery<InstanceDetails> discovery;

    private final JsonInstanceSerializer<InstanceDetails> serializer = new JsonInstanceSerializer<>(InstanceDetails.class);

    private final InstanceDetails payload = new InstanceDetails(false);
    private ServiceInstance<InstanceDetails> serviceInstance;

    @PostConstruct
    public void init(@Observes @Initialized(ApplicationScoped.class) Object ignore) {
		client = CuratorFrameworkFactory.newClient(zookeeperConnection, new ExponentialBackoffRetry(1000, 3));
		client.start();

		try {
		    client.blockUntilConnected();

		    serviceInstance = ServiceInstance.<InstanceDetails>builder()
		            .uriSpec(new UriSpec("{scheme}://{address}:{port}"))
		            .address("localhost")
		            .port(Integer.parseInt(port))
		            .name("bookService")
		            .payload(payload)
		            .build();

		    discovery = ServiceDiscoveryBuilder.builder(InstanceDetails.class)
		            .basePath("service-doscovery")
		            .client(client)
		            .thisInstance(serviceInstance)
		            .watchInstances(true)
		            .serializer(serializer)
		            .build();

		    discovery.start();

		    leaderLatch = new LeaderLatch(client, latchPath, nodeId);
		    leaderLatch.addListener(this);
		    leaderLatch.start();
		} catch (Exception e) {
		    log.error("Error when starting leaderLatch", e);
		}
    }

    public boolean hasLeadership() {
		return leaderLatch.hasLeadership();
    }

    @Override
    public void isLeader() {
		log.info("Node : " + nodeId + " is a leader");
		payload.setLeader(true);
		try {
		    discovery.updateService(serviceInstance);
		} catch (Exception e) {
		    log.error("Error when updating service discovery.", e);
		}
    }

    @Override
    public void notLeader() {
		log.info("Node : " + nodeId + " is not a leader");
		payload.setLeader(false);
		try {
		    discovery.updateService(serviceInstance);
		} catch (Exception e) {
			log.error("Error when closing leaderLatch.", e);
		}
    }
}
{% endcodeblock %}

- Lines 17-18: same as before: we are creating and starting CuratorFramework client instance 
- Lines 23-29: we need to define representation of our service as ServiceInstance which have a name, id, address, port and an optional payload. Payload is a user defined class with additional instance information, in our case it will be simple InstanceDetails instance with single boolean property: leader. ServiceInstances are serialized and stored as JSON in ZooKeeper
- Lines 31-37: register ServiceInstance instance in a Zookeeper through ServiceDiscovery object
- Lines 56-58: when our instance receive notification about leader state changes, we need to update our ServiceInstance in a ZooKeeper 

{% codeblock lang:javascript console %}
{% raw %}

./bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: ~/tools/zookeeper-3.4.9/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-1 -Dport=8070 -jar target/high-availability-active-passive-microservices
-1.0-SNAPSHOT.jar

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-2 -Dport=8080 -jar target/high-availability-active-passive-microservices
-1.0-SNAPSHOT.jar

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-3 -Dport=8090 -jar target/high-availability-active-passive-microservices
-1.0-SNAPSHOT.jar

~/tools/zookeeper-3.4.9/bin/zkCli.sh -server 127.0.0.1:2181

[[zk: 127.0.0.1:2181(CONNECTED) 2] ls /service-discovery/bookService
[a1bc97ba-9926-41be-92a4-2fe6b02597b4, e3d83250-20df-4a4b-bc72-a89e847cfacb, d4ee622c-20e8-4e3d-b053-82677270fac6] 

[zk: 127.0.0.1:2181(CONNECTED) 3] get /service-discovery/bookService/a1bc97ba-9926-41be-92a4-2fe6b02597b4
{"name":"bookService","id":"a1bc97ba-9926-41be-92a4-2fe6b02597b4","address":"localhost","port":8070,"sslPort":null,"payload":{"@class":"com.thecookiezen.microservices.infrastructure.status.InstanceDetails","leader":true},"registrationTimeUTC":1480963577325,"serviceType":"DYNAMIC","uriSpec":{"parts":[{"value":"scheme","variable":true},{"value":"://","variable":false},{"value":"address","variable":true},{"value":":","variable":false},{"value":"port","variable":true}]}}

[zk: 127.0.0.1:2181(CONNECTED) 5] get /service-discovery/bookService/e3d83250-20df-4a4b-bc72-a89e847cfacb
{"name":"bookService","id":"e3d83250-20df-4a4b-bc72-a89e847cfacb","address":"localhost","port":8080,"sslPort":null,"payload":{"@class":"com.thecookiezen.microservices.infrastructure.status.InstanceDetails","leader":false},"registrationTimeUTC":1480963591367,"serviceType":"DYNAMIC","uriSpec":{"parts":[{"value":"scheme","variable":true},{"value":"://","variable":false},{"value":"address","variable":true},{"value":":","variable":false},{"value":"port","variable":true}]}}

[zk: 127.0.0.1:2181(CONNECTED) 6] get /service-discovery/bookService/d4ee622c-20e8-4e3d-b053-82677270fac6
{"name":"bookService","id":"d4ee622c-20e8-4e3d-b053-82677270fac6","address":"localhost","port":8090,"sslPort":null,"payload":{"@class":"com.thecookiezen.microservices.infrastructure.status.InstanceDetails","leader":false},"registrationTimeUTC":1480963600005,"serviceType":"DYNAMIC","uriSpec":{"parts":[{"value":"scheme","variable":true},{"value":"://","variable":false},{"value":"address","variable":true},{"value":":","variable":false},{"value":"port","variable":true}]}}

{% endraw %}	
{% endcodeblock %}

We will check if our instances are properly registered in Services Discovery. Under the path /service-discovery/bookService in a ZooKeeper we can find all the three services and get details as a JSON object. Node with port 8070 (node-1) is the leader.

##confd

Confd is a tool for dynamic managing application's configuration. Using templates and information stored in Service Discovery systems like etcd, Consul or ZooKeeper, confd can keep track changes of specific registry. When data under the specified registry changes, confd process generates new configuration based on the template and provided information from Service Discovery system. At the end it will invoke user defined action, like reloading web server, to pick new configuration up. In our example, all details about our running services are kept in ZooKeeper as JSON objects. Confd is watching for changes like appearance of a new node or crash of one of the present. Then it generates new configuration for our load balancer NGINX, and perform the server reload. Thanks to this solution, we don't need to do manually update of the NGINX configuration every time when something changes - everything happens automatically.

* Last confd release doesn't have ZooKeeper watch option, so we need to download the newest source code from [github repo](https://github.com/kelseyhightower/confd) and compile by ourselves. 

{% codeblock lang:javascript nginx.conf.toml %}
{% raw %}

[template]
src = "services.conf.tmpl"
dest = "/etc/nginx/nginx.conf"
keys = [
  "/service-discovery/bookService/"
]
check_cmd = "/usr/sbin/nginx -t -c {{.src}}"
reload_cmd = "systemctl reload nginx"
{% endraw %}
{% endcodeblock %}

First we need to create a template resource configuration which is defined in TOML format. It is required to define:

- src - path of a configuration template
- dest - target file (NGINX configuration location)
- keys - keys in Service Discovery 


{% codeblock lang:javascript services.conf.tmpl %}
{% raw %}
events {
	worker_connections 768;
	# multi_accept on;
}

http {

	upstream read_nodes {
	{{range gets "/service-discovery/bookService/*"}}
	{{$data := json .Value}}
	 	 	server {{$data.address}}:{{$data.port}};
	{{end}}
	}

	upstream write_nodes {
	{{range gets "/serivce-discovery/bookService/*"}}
	{{$data := json .Value}}
		{{$payload := $data.payload}}
		{{if $payload.leader}}
	 	 	server {{$data.address}}:{{$data.port}};
	 	{{end}}
	{{end}}
	}

	server {
		listen 80;
	   	server_name  thecookiezen.com;

	   	location / {
		proxy_pass        http://read_nodes/;
	        proxy_redirect    off;
	        proxy_set_header  Host             $host;
	        proxy_set_header  X-Real-IP        $remote_addr;
	        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;

	    	limit_except GET {
	      		proxy_pass http://write_nodes;
	    	}
	  	}
	}
}
{% endraw %}
{% endcodeblock %}

Then we define our template in Go's text/template for NGINX configuration. Configuration will have two separate upstreams: one for reads (only GET requests), and one for writes. Upstream for writes will be composed only of those nodes, that are leaders. It will be single node for writes. 

To start confd process for watching ZooKeeper, we need to invoke the following command:

{% codeblock lang:javascript console %}
{% raw %}

./confd -backend zookeeper -node 127.0.0.1:2181 -watch
./confd[8092]: INFO Backend set to zookeeper
./confd[8092]: INFO Starting confd
./confd[8092]: INFO Backend nodes set to 127.0.0.1:2181
Connected to 127.0.0.1:2181
Authenticated: id=97062410017046535, timeout=4000

{% endraw %}
{% endcodeblock %}

When a change is detected, the information about new configuration appears in the log.

{% codeblock lang:javascript console %}
{% raw %}

./confd[8092]: INFO /etc/nginx/nginx.conf has md5sum a275673fe190cf1f201e83321ffe9b5d should be ae2adf87ffdc5139e14fa779bdf98b8d
./confd[8092]: INFO Target config /etc/nginx/nginx.conf out of sync
./confd[8092]: INFO Target config /etc/nginx/nginx.conf has been updated

{% endraw %}
{% endcodeblock %}


##Summary

It's time to run everything together. First we check the NGINX upstreams configuration.

{% codeblock lang:javascript /etc/nginx/nginx.conf %}
{% raw %}

upstream read_nodes {
}

upstream write_nodes {
}

{% endraw %}
{% endcodeblock %}

Upstreams are empty because everything is down, lets start: ZooKeeper, confd, NGINX and single application's instance 

{% codeblock lang:javascript console %}
{% raw %}

./bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: tools/zookeeper-3.4.9/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

sudo ./confd -backend zookeeper -node 127.0.0.1:2181 -watch
2016-12-08T20:26:05+01:00 ./confd[9947]: INFO Backend set to zookeeper
2016-12-08T20:26:05+01:00 ./confd[9947]: INFO Starting confd
2016-12-08T20:26:05+01:00 ./confd[9947]: INFO Backend nodes set to 127.0.0.1:2181
2016/12/08 20:26:05 Connected to 127.0.0.1:2181
2016/12/08 20:26:05 Authenticated: id=97073568427278339, timeout=4000

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-1 -Dport=8070 -jar target/high-availability-active-passive-microservices-1.0-SNAPSHOT.jar 

sudo systemctl start nginx

{% endraw %}
{% endcodeblock %}

{% codeblock lang:javascript /etc/nginx/nginx.conf %}
{% raw %}
upstream read_nodes {
    server localhost:8070;
}

upstream write_nodes {
    server localhost:8070;
}

{% endraw %}
{% endcodeblock %}

First instance was elected as a leader and is located in both upstreams. Lets run 2 more instances.

{% codeblock lang:javascript console %}
{% raw %}

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-2 -Dport=8080 -jar target/high-availability-active-passive-microservices-1.0-SNAPSHOT.jar 

java -DzookeeperConnection=127.0.0.1:2181 -DlatchPath=/example/leader -DnodeId=node-3 -Dport=8090 -jar target/high-availability-active-passive-microservices-1.0-SNAPSHOT.jar 

{% endraw %}
{% endcodeblock %}

{% codeblock lang:javascript /etc/nginx/nginx.conf %}
{% raw %}

upstream read_nodes {
    server localhost:8070;
    server localhost:8080;
    server localhost:8090;
}

upstream write_nodes {
    server localhost:8070;
}

{% endraw %}
{% endcodeblock %}

Both new instances are not a leader and went to read_nodes upstream. Now we can send some requests to test our solution. The response from the endpoint is the node name.

{% codeblock lang:javascript console %}
{% raw %}

curl -X POST http://localhost/microservice/test/hello
node-1

curl -X POST http://localhost/microservice/test/hello
node-1

curl -X POST http://localhost/microservice/test/hello
node-1

curl -X POST http://localhost/microservice/test/hello
node-1

curl -X DELETE http://localhost/microservice/test/hello
node-1

curl -X DELETE http://localhost/microservice/test/hello
node-1

curl -X GET http://localhost/microservice/test/hello
node-1

curl -X GET http://localhost/microservice/test/hello
node-2

curl -X GET http://localhost/microservice/test/hello
node-3

curl -X GET http://localhost/microservice/test/hello
node-1

curl -X GET http://localhost/microservice/test/hello
node-2

curl -X GET http://localhost/microservice/test/hello
node-3

{% endraw %}
{% endcodeblock %}

As we can see all GET requests are distributed across all the nodes and requests responsible for the resource modification goes only to node-1 which is a leader. In the second part (that is scheduled to be ready by the end of this year), we will focus on replication messages through Apache Kafka.

[Full source code](https://github.com/nikom1337/ha-active-passive-microservices)