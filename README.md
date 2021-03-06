# eureka-server

In this tutorial, I will be using Spring Cloud Eureka technology to implement service registry and service discovery design patterns of microservice architecture.

Articles referred:
- https://sivalabs.in/2018/03/microservices-springcloud-eureka/
- Official Netflix Eureka documentation - https://github.com/Netflix/eureka/wiki
- https://spring.io/guides/gs/service-registration-and-discovery/
- https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-eureka-server.html


We need to create Eureka Server first and then add eureka client dependency in all the microservices which we want to register to Eureka Server.

By default, eureka server application registers itself to eureka server.

We can see Eureka Dashboard. For me it's on localhost so the endpoint is http://localhost:8761/

### What is Eureka Technology?
Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers.

At Netflix, Eureka is used for the following purposes apart from playing a critical part in mid-tier load balancing.

For aiding Netflix Asgard - an open source service which makes cloud deployments easier, in

Fast rollback of versions in case of problems avoiding the re-launch of 100's of instances which could take a long time.
In rolling pushes, for avoiding propagation of a new version to all instances in case of problems.
For our cassandra deployments to take instances out of traffic for maintenance.

For our memcached caching services to identify the list of nodes in the ring.

For carrying other additional application specific metadata about services for various other reasons.

#### Renew
Eureka client needs to renew the lease by sending heartbeats every 30 seconds. The renewal informs the Eureka server that the instance is still alive. If the server hasn't seen a renewal for 90 seconds, it removes the instance out of its registry. It is advisable not to change the renewal interval since the server uses that information to determine if there is a wide spread problem with the client to server communication.

#### Fetch Registry
Eureka clients fetches the registry information from the server and caches it locally. After that, the clients use that information to find other services. This information is updated periodically (every 30 seconds) by getting the delta updates between the last fetch cycle and the current one. The delta information is held longer (for about 3 mins) in the server, hence the delta fetches may return the same instances again. The Eureka client automatically handles the duplicate information.
After getting the deltas, Eureka client reconciles the information with the server by comparing the instance counts returned by the server and if the information does not match for some reason, the whole registry information is fetched again. Eureka server caches the compressed payload of the deltas, whole registry and also per application as well as the uncompressed information of the same. The payload also supports both JSON/XML formats. Eureka client gets the information in compressed JSON format using jersey apache client.

#### Time Lag
All operations from Eureka client may take some time to reflect in the Eureka servers and subsequently in other Eureka clients. This is because of the caching of the payload on the eureka server which is refreshed periodically to reflect new information. Eureka clients also fetch deltas periodically. Hence, it may take up to 2 mins for changes to propagate to all Eureka clients.

#### Communication mechanism
By default, Eureka clients use Jersey and Jackson along with JSON payload to communicate with Eureka Server. You can always use a mechanism of your choice by overriding the default one. Note that XStream is also part of the dependency graph for some legacy use cases.

### How to configure Eureka Server?
Create a new service with spring-cloud-starter-netflix-eureka-server in classpath and use @EnableEurekaServer
```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

Add following properties in bootstrap.yml. The combination of the two caches (client and server) and the heartbeats make a standalone Eureka server fairly resilient to failure, as long as there is some sort of monitor or elastic runtime (such as Cloud Foundry) keeping it alive. In standalone mode, you might prefer to switch off the client side behavior so that it does not keep trying and failing to reach its peers.
```
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```
Here we’re configuring an application port – 8761 is the default one for Eureka servers. We are telling the built-in Eureka Client not to register with ‘itself’ because our application should be acting as a server.

Bring up the application and we can see dashboard at http://localhost:8761/

### How to configure Eureka Clients?
Create client services with spring-cloud-starter-netflix-eureka-client in classpath. We can also annotate our application class with @EnableDiscoveryClient or @EnableEurekaClient but these annotations are optional because we already have spring-cloud-starter-netflix-eureka-client in classpath.

Bootstrap.yml looks like below:-
- We need to define spring.application.name as this same name is used as service id for registration with Eureka server.
- We need to define eureka.client.service-url.defaultZone property with the location of running eureka server.

```
spring:
  cloud:
    config:
     uri: http://localhost:8001
     failFast: true
     name: storefront-service
  application:
    name: storefront-service
---
spring:
  profiles: dev
---
spring:
  profiles:
    active: dev
management:
  endpoint:
    env:
      enabled: true 
    info:
      enabled: true
    threaddump:
      enabled: true
    refresh:
      enabled: true
  endpoints:
    web:
      exposure:
        include: env, bus-refresh
---
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```
#### Exploring @EnableDiscoveryClient
I added following operation in of the service and hitting this endpoint, I am able to see which all instances are running for the given service-id. All this is done with co-ordination of eureka server. Also, annotate main application class with @EnableDiscoveryClient
```
    @Autowired
    private DiscoveryClient discoveryClient;
    
    @RequestMapping("/service-instances/{applicationName}")
    public List<ServiceInstance> serviceInstancesByApplicationName(@PathVariable String applicationName) {
        return this.discoveryClient.getInstances(applicationName);
    }
```

#### High Availability, Zones and Regions
The Eureka server does not have a back end store, but the service instances in the registry all have to send heartbeats to keep their registrations up to date (so this can be done in memory). Clients also have an in-memory cache of Eureka registrations (so they do not have to go to the registry for every request to a service).

By default, every Eureka server is also a Eureka client and requires (at least one) service URL to locate a peer. If you do not provide it, the service runs and works, but it fills your logs with a lot of noise about not being able to register with the peer.

#### Running Eureka Server over more than one server
Eureka server can be single point of fialure if it is configured only on one server. To Avoid this we can run multiple instances of eureka server and make them aware of each other. They themselves will act as eureka client for other eureka server. Refer these configurations - https://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-eureka-server.html#spring-cloud-eureka-server-peer-awareness




