# Refactoring an ERP Monolith for Microservice Deployment in the Cloud with MongoDB

In my capacity as the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most formidable undertakings in my career was the conversion of a legacy monolithic ERP system into a contemporary microservices architecture. The antiquated monolith had grown burdensome to maintain and was impeding my client's ability to drive innovation.

Fortunately, this challenge was not foreign to us, given our extensive experience in providing tailored [Software Development Services in Buffalo](https://hybridwebagency.com/buffalo-ny/best-software-development-company/). We are intimately acquainted with the hurdles organizations face when grappling with inflexible and resistant systems. That's why I am eager to divulge the systematic approach we adopted to restructure the data model and disassemble the monolith, both at the code and database levels.

Throughout the migration process, my team and I unearthed some less-traveled best practices for molding domain entities within a document database like MongoDB. These practices are designed to seamlessly support fully autonomous microservices. If you have ever pondered the art of future-proofing your database architecture for the cloud while preserving historical data, you are in for a wealth of enlightening strategies.

By the conclusion of this article, you will have in your possession a comprehensive blueprint for the migration of your legacy systems to a contemporary architecture. I will be sharing a treasure trove of practical insights to help you steer clear of common pitfalls and expedite the delivery of new features to your clientele. Let's embark on this journey of discovery!

## The Perks of Embracing a Microservices Architecture

The adoption of a microservices architecture brings forth a multitude of advantages over the conventional monolithic approach. Microservices enable autonomous deployments, facilitating rapid development and feature releases without causing upheaval across the entire application.

Moreover, individual services can be crafted using diverse programming languages and frameworks. This flexibility empowers organizations to select the most fitting technologies for each specific domain. For example, a recommendation engine might harness Python's machine learning libraries, while the user interface is crafted with React.

This decoupling of concerns streamlines the work of specialized teams, each focusing on separate services. It fosters a culture of experimentation, as ideas can be swiftly validated through prototypes prior to committing to a full-scale overhaul of the monolithic system. Additionally, new team members can make substantial contributions to a single service aligning with their expertise.

In a landscape where technology trends wax and wane, microservices act as a safeguard against these shifts. Replacements only necessitate modifications to small, isolated portions of the system, rather than an extensive rewrite. Clearly defined interfaces ensure a smooth transition when migrating components to new implementations.

#### Independent Scalability

Efficient resource management materializes when services scale autonomously in response to demand. The frontend API gateway, for instance, efficiently directs traffic based on URLs and can securely deploy behind a load balancer to handle surges in traffic. During peak seasons, such as holidays, only the order processing service requires additional servers, sparing the entire system from unnecessary scaling.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service level results in substantial cost savings through precise right-sizing of resources. Idle support microservices, like user profiles, no longer necessitate costly overprovisioning to manage traffic that has no impact on them.

## Scrutinizing the Data Model

### Grasping Entity Relationships

The inaugural step in transitioning to microservices involves a meticulous examination of how entities interrelate within the monolithic data model. We embarked on a comprehensive exploration of each collection within the MongoDB database to pinpoint clusters of domains and transactional boundaries.

Entities such as Users, Products, and Orders emerged as the pivotal elements within bounded contexts. Relationships between these core entities emerged as prime candidates for service decomposition. For instance, we noticed that Orders contained foreign keys linking to Users for customer information and Products to represent purchased items.

To gain deeper insights into these interdependencies, we produced sample documents to visualize associated fields. An intriguing revelation was that legacy code redundantly stored data that now pertained to separate business capabilities. For example, shipping addresses needlessly duplicated user profiles rather than referencing a lightweight counterpart.

This examination of relationships unearthed problematic tight couplings between modules, resulting in cascading updates. The normalization of redundant data eliminated impediments to the independent development of user profiles and shipping namespaces.

We harnessed database tools to explore the intricate network of connections. Utilizing MongoDB Compass, we meticulously charted relationships through $lookup pipelines and executed aggregate queries to tally references between entities. This unveiled pivotal breakpoints for dividing logic into coherent services.

These relationships guided the delineation of domain boundaries and ensured services presented clean, precisely defined interfaces. These well-defined contracts empowered autonomous teams to develop and deploy modules incrementally as micro frontends without impeding one another.

### Identifying Transactional Boundaries

Beyond relationships, we delved into the transactions entrenched in the existing codebase to comprehend the flow of business processes. This endeavor aided us in pinpointing areas where data modifications needed to be confined within individual services to uphold data consistency and integrity.

For instance, in the realm of order processing, we discerned that any updates related to the order itself, associated payments, inventory levels, and shipment notifications had to occur within a single service, in a transactional manner. This realization was instrumental in delineating the boundaries of our Order Management service.

A meticulous examination of both relationships and transactions yielded invaluable insights that shaped the refactoring of the data model and logic into independently deployable microservices, each characterized by clearly defined interfaces.

## Restructuring for Microservices

### Normalizing Data Schemas

To accommodate independent services that may deploy to different data stores if necessary, we embarked on the normalization of schemas. The aim was to eliminate redundancy and include only the essential data required by each service.

For instance, the original Orders schema encompassed the entire User object. We streamlined this by introducing a lightweight reference:

```
// Before
Orders: {
  user: {
   

 name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

Similarly, we extracted product details from Orders and placed them into their own collections. This approach allowed these entities to evolve independently over time.

### Leveraging Domain-Driven Design

We harnessed the concept of bounded contexts from Domain-Driven Design to logically segregate services, such as Order Fulfillment versus User Profiles. Interfaces assumed a pivotal role as they abstracted data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

Queries and commands underwent substantial refactoring to align with the new architectural paradigm. In the past, services directly accessed data using calls like `db.collection.find()`. We introduced abstraction through data access libraries:

```
// order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this.ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This ensured flexibility, allowing us to migrate databases without necessitating changes to consumer code.

## Implementing Microservices

### Independent Scaling

In the realm of microservices, autoscaling is carried out at the individual service level, diverging from the application-wide approach. We implemented scaling logic using Docker Swarm and Kubernetes.

Deployment manifests delineated scaling policies based on CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

Scaling tiers provided reserve capacity and elastic overflow handling. The orders service autonomously monitored its performance and spawned or terminated containers as needed:

```js
// orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers such as Nginx and Traefik adeptly routed traffic to scaled replica sets. This optimization of resource utilization bolstered throughput and concurrently reduced costs.

### Enforcing Resilience

Our arsenal of resiliency techniques encompassed retry policies, timeouts, and circuit breakers. Rate limiting and throttling acted as shields against cascading failures, while the Platform service took charge of transient error policies for dependent services.

Homemade and open-source solutions, including Polly, Hystrix, and Resilience4j, stood as guardians against potential mishaps. Centralized logging via Elasticsearch played a pivotal role in tracing errors across the distributed applications.

### Ensuring Reliability

Incorporating reliability into microservices necessitated the implementation of various techniques aimed at averting single points of failure. We prioritized automated responses to transient errors and designed measures to tackle overload scenarios.

Leveraging the Resilience4J library, we implemented circuit breakers to gracefully handle faults:

```java
// OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was employed to prevent service flooding during times of stress:

```java 
// RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were introduced to terminate lengthy calls:

```java
// ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

We introduced retry logic through policies defined at the client level:

```java
// RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy = new SimpleRetryPolicy();
    policy.setMaxAttempts(3);
    return a RetryTemplate(policy);
  }

} 
```

These techniques guaranteed consistent responses and thwarted cascading failures across the spectrum of services.

## In Conclusion

The migration of our monolithic ERP system into microservices has been an invaluable learning experience. It transcends the realm of a mere technical migration, signifying an organizational metamorphosis that equips our client to better serve their ever-evolving customer base.

By dismantling tightly coupled layers and instituting clear domain boundaries, our development team has unlocked a new level of agility. Features can now be developed and deployed independently, driven by business priorities rather than architectural constraints. This capacity for rapid experimentation and refinement positions the application to remain responsive to dynamic market demands.

Simultaneously, our operations team now enjoys full visibility and control over each system component. Anomalous behavior can be promptly detected through improved monitoring of discrete services. Scaling and failover mechanisms have transitioned from manual interventions to automated processes, fostering enhanced resilience that will serve as a reliable foundation for our client's continued growth.

While the benefits of microservices are unequivocal, embarking on such a migration is not without its challenges. By dedicating time to meticulous relationship analysis, interface definition, and abstraction introduction—rather than resorting to a crude 'rip and replace' approach—we have

 cultivated a flexible architecture capable of evolving in tandem with customer needs.

Above all, I am profoundly grateful for the opportunity this project has afforded us, allowing us to collaborate closely with our client on their digital transformation journey. Sharing our experiences and insights in this article is my way of paying it forward, with the hope that it empowers more businesses to embrace modernization. The rewards, both for customers and businesses, are unquestionably worth the endeavor.

## References 

- MongoDB Documentation - Official documentation on data modeling, queries, deployment, and more. [https://docs.mongodb.com/](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles. [https://martinfowler.com/articles/microservices.html](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains. [https://domainlanguage.com/ddd/](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps that are readily deployable to the cloud. [https://12factor.net/](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, and cloud platforms. [https://containerjournal.com/](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps. [https://docs.docker.com/](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture. [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)
