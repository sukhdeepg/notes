## WEB
⚙️ Working of PUT `/users/[id]/followers`

### Method
PUT (typically used for creating/updating relationships)

### Path
`/users/[id]/followers` where `[id]` is the user ID we want to follow

### Body
Empty JSON object `{}`

### How it works
When we make this request, it creates a "follower" relationship between the authenticated user (us) and the target user specified by `[id]`. The PUT method suggests this is idempotent - calling it multiple times has the same effect as calling it once.

### Typical flow
1. User clicks "Follow" button on another user's profile
2. Frontend sends PUT request to this endpoint with target user's ID
3. Backend creates follower relationship in database
4. User is now following the target user

The empty body suggests no additional data is needed - the authentication token identifies who is following, and the URL parameter identifies who is being followed.

---

### Database Schema

Users table
```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Followers relationship table
```sql
CREATE TABLE followers (
    follower_id INT,
    following_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### Key points
- `followers` table stores the many-to-many relationship
- `follower_id` = user who is following
- `following_id` = user being followed
- Composite primary key prevents duplicate follows
- Foreign keys ensure data integrity

### For the API endpoint
- `PUT /users/123/followers` would insert `(current_user_id, 123)` into the followers table

⚙️ [What happens when we type google.com in the browser?](https://bytebytego.com/guides/what-happens-when-you-type-google/)

---

## Architecture
⚙️ What is the meaning of Serverless?  
**Serverless** is a cloud execution model where we deploy discrete, event-driven functions (FaaS - Function as a Service) and the provider transparently handles all server provisioning, scaling, patching and availability. We write short-lived functions that run in response to events—HTTP requests, message queues, file uploads, etc and we pay only for the actual compute time consumed.

**Why "serverless" if servers are still used?** Because we, the developer, never see or manage those servers. All infrastructure details, capacity planning, OS updates, fault-tolerance are abstracted away. From our perspective it's truly "without servers" to worry about.

### Example
Capital One employs AWS Lambda functions triggered by Kinesis Data Streams to perform real-time fraud detection on credit-card transactions. Each transaction event invokes a Lambda that runs its ML inference logic, flags anomalies, and writes results to DynamoDB. Under peak load (e.g. holiday shopping), Lambda automatically scales to thousands of concurrent executions. No servers to provision or idle while the bank pays only for the milliseconds each function runs.

⚙️ User search string to Elasticsearch query

### Frontend
```javascript
async function search(userInput) {
    const response = await fetch('/search', {
        method: 'POST',
        body: JSON.stringify({query: userInput.trim()})
    });
    return await response.json();
}
```

### Backend
```python
from fastapi import FastAPI
from elasticsearch import Elasticsearch
import re

app = FastAPI()
es = Elasticsearch(['localhost:9200'])

def parse_query(query: str):
    # Extract parts from user input
    location = re.search(r'in\s+(\w+)', query)
    price = re.search(r'\$(\d+)-\$(\d+)', query)
    
    return {
        'text': query,
        'location': location.group(1) if location else None,
        'price_min': int(price.group(1)) if price else None,
        'price_max': int(price.group(2)) if price else None
    }

def build_elasticsearch_query(entities):
    # Convert to Elasticsearch JSON format
    query = {
        "query": {
            "bool": {  # Boolean query combines multiple conditions
                "must": [],    # These conditions MUST match
                "filter": []   # These conditions filter results
            }
        }
    }
    
    # Add text search
    query["query"]["bool"]["must"].append({
        "multi_match": {
            "query": entities['text'],
            "fields": ["title^2", "description"]  # ^2 means title is 2x more important
        }
    })
    
    # Add location filter if found
    if entities['location']:
        query["query"]["bool"]["filter"].append({
            "term": {"location": entities['location']}
        })
    
    return query

@app.post("/search")
async def search(request: dict):
    # Step 1: Parse user input
    entities = parse_query(request['query'])
    
    # Step 2: Build Elasticsearch query
    es_query = build_elasticsearch_query(entities)
    
    # Step 3: Send to Elasticsearch
    return es.search(index="products", body=es_query)
```

---

### What Happens Inside Elasticsearch
1. Query Processing  
    - Receives JSON query (the structured format above)
    - Validates the query structure
    - Optimizes the query for faster execution

2. Find Relevant Documents  
    ```
    "pizza restaurant" becomes:
    ├── "pizza" → finds documents 1, 5, 12, 89...
    ├── "restaurant" → finds documents 2, 5, 15, 23...
    └── Combined: documents that have both words
    ```

3. Scoring Each Document
Elasticsearch uses **BM25** algorithm to score relevance:
    ```
    Higher score if:
    ├── Word appears more times in document (term frequency)
    ├── Word is rare across all documents (less common = more important)
    └── Document is shorter (concentrated relevance)
    ```

4. Machine Learning in Elasticsearch
    - **Vector Search**: Converts text to numbers for semantic similarity
    - **Learning to Rank**: ML models reorder results based on user behavior
    - **Custom Models**: We can upload trained models for classification

5. Return Results
    - Sorts by relevance score (highest first)
    - Applies pagination (first 10 results)
    - Returns JSON with documents and their scores

### Example Flow
```
User: "pizza restaurant in NYC under $20"
           ↓
Backend extracts: {text: "pizza restaurant", location: "NYC", price_max: 20}
           ↓
Elasticsearch finds: 1000 pizza restaurants in NYC
           ↓
Filters by: price under $20 → 200 results
           ↓
Scores by: relevance to "pizza restaurant" → sorted list
           ↓
Returns: Top 10 results with scores
```

Key Point
Elasticsearch doesn't understand meaning like humans do. It matches words, counts frequencies, and uses math to rank results. ML adds semantic understanding on top of this basic matching.

⚙️ Split brain problem in Redis  
**The Core Issue:** When the Redis cluster gets network partitioned, isolated nodes can't tell if the master is dead or just unreachable. Multiple nodes might promote themselves to master, creating chaos with conflicting data writes.

- **What Goes Wrong:**
    - Network partition splits our cluster into isolated islands
    - Each island thinks the master is down and elects a new one
    - Now we have multiple masters accepting different writes
    - Data becomes inconsistent and irreconcilable

### What is Redis Sentinel?
Redis Sentinel is a **separate lightweight process** that runs on its own servers (not on our Redis data nodes). Think of it as an external referee that watches our Redis cluster and makes failover decisions.

- **Key Points:**
    - Sentinel runs as independent processes on dedicated servers
    - We typically deploy 3 or 5 Sentinel instances across different availability zones
    - Each Sentinel continuously monitors our Redis masters and replicas
    - They communicate with each other to make collective decisions

### The Quorum Magic
- **How it works:**
    - Sentinel requires majority agreement (quorum) before promoting a new master
    - With 3 Sentinels: need 2 votes minimum
    - With 5 Sentinels: need 3 votes minimum
    - Network partitions can only leave majority on ONE side, never both

- **Why this prevents split brain:**
    - **Partition Example with 3 Sentinels:**
        - Side A: 1 Sentinel (minority) → Cannot promote master
        - Side B: 2 Sentinels (majority) → Can promote master
    - **Result: Only ONE master exists at any time**

- Quick Summary
    - **Problem:** Network splits create multiple masters with conflicting data
    - **Solution:** Sentinel acts as external referee requiring majority vote
    - **Key:** Majority can only exist on one side of any partition
    - **Deployment:** Run Sentinel on separate servers from our Redis data nodes

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Sentinel-1  │    │ Sentinel-2  │    │ Sentinel-3  │
│ (Zone A)    │    │ (Zone B)    │    │ (Zone C)    │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                           ▼
                ┌─────────────────┐
                │   Redis Master  │
                │   (Zone A)      │
                └─────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Replica   │    │   Replica   │    │   Replica   │
│  (Zone A)   │    │  (Zone B)   │    │  (Zone C)   │
└─────────────┘    └─────────────┘    └─────────────┘
```

## Network Partition Scenario:

```
┌─────────────┐    │    ┌─────────────┐    ┌─────────────┐
│ Sentinel-1  │    │    │ Sentinel-2  │    │ Sentinel-3  │
│ (Zone A)    │    │    │ (Zone B)    │    │ (Zone C)    │
└─────────────┘    │    └─────────────┘    └─────────────┘
       │           │           │                   │
       ▼           │           └───────────────────┘
┌─────────────┐    │                       │
│   Master    │    │                       │
│  (Dead?)    │    │           Only 2 Sentinels
└─────────────┘    │           Majority = 2/3 ✓
                   │           Can elect new master
                   │                       │
            PARTITION                      ▼
                   │                ┌─────────────┐
                   │                │   Replica   │ → Promoted to Master
                   │                │  (Zone B)   │
                   │                └─────────────┘
```

⚙️ Service Registry & Discovery
In microservices, services need to find and communicate with each other. Hard-coding IP addresses doesn't work because:
- Service instances start/stop dynamically
- IP addresses change with scaling/deployments
- Load balancing requires knowing all available instances

### Solution
**Service Registry:** Central database of all service instances and their locations  
**Service Discovery:** Mechanism for services to find each other

### Three Approaches
- Client-Side Discovery
    - Client queries registry directly and chooses instance

- Server-Side Discovery
    - Load balancer queries registry and routes requests

- DNS-Based Discovery
    - Services register with DNS server, clients resolve service names to IP addresses using standard DNS lookups. The DNS server returns multiple A records for load balancing or SRV records for port information.

### Real-World Examples
- Netflix Eureka
    - Services register themselves on startup
    - Clients cache service locations locally
    - Handles thousands of service instances
- Consul (HashiCorp)
    - DNS-based service discovery using standard DNS queries
    - Health checking built-in
    - Used by companies like Uber, Airbnb
- Kubernetes Service Discovery
    - Built-in DNS for service discovery (CoreDNS)
    - Services accessible via service names like `user-service.default.svc.cluster.local`
    - Automatic load balancing

- Key Concepts
    - **Registration:** Services announce their location when they start
    - **Discovery:** Clients find services by name, get list of healthy instances
    - **Health Checking:** Registry monitors service health via heartbeats/health endpoints
    - **Load Balancing:** Client chooses which instance to call (client-side) or load balancer does it (server-side)

- When to Use
    - Multiple instances of same service
    - Dynamic scaling (containers/cloud)
    - Services need to communicate
    - High availability requirements

The example shows a basic in-memory registry. Production systems use distributed registries like Consul, etcd, or cloud-native solutions.
```python
# Service Registry
import threading
from typing import Dict, List
import time

class ServiceRegistry:
    def __init__(self):
        self.services: Dict[str, List[dict]] = {}
        self.services_lock = threading.Lock()
        self.service_heartbeat_thread = threading.Thread(target=self._service_heartbeat)
        self.service_heartbeat_thread.daemon = True
        self.service_heartbeat_thread.start()

    def register_service(self, service_name: str, host: str, port: int, health_check_url: str = None):
        """Register a service instance"""
        with self.services_lock:
            if service_name not in self.services:
                self.services[service_name] = []
            
            service_info = {
                'host': host,
                'port': port,
                'health_check_url': health_check_url,
                'last_heartbeat': time.time(),
                'healthy': True
            }
            
            # Remove existing instance if it exists
            self.services[service_name] = [s for s in self.services[service_name] 
                                         if not (s['host'] == host and s['port'] == port)]
            
            self.services[service_name].append(service_info)
            
        print(f"Registered service {service_name} at {host}:{port}")

    def deregister_service(self, service_name: str, host: str, port: int):
        """Deregister a service instance"""
        with self.services_lock:
            if service_name in self.services:
                self.services[service_name] = [s for s in self.services[service_name] 
                                             if not (s['host'] == host and s['port'] == port)]
                if not self.services[service_name]:
                    del self.services[service_name]
                    
        print(f"Deregistered service {service_name} at {host}:{port}")

    def get_service_instances(self, service_name: str) -> List[dict]:
        """Get all healthy instances of a service"""
        with self.services_lock:
            if service_name in self.services:
                return [s for s in self.services[service_name] if s['healthy']]
            return []

    def heartbeat(self, service_name: str, host: str, port: int):
        """Update heartbeat for a service instance"""
        with self.services_lock:
            if service_name in self.services:
                for service in self.services[service_name]:
                    if service['host'] == host and service['port'] == port:
                        service['last_heartbeat'] = time.time()
                        service['healthy'] = True
                        break

    def _service_heartbeat(self):
        """Background thread to check service heartbeats"""
        while True:
            current_time = time.time()
            with self.services_lock:
                for service_name, instances in self.services.items():
                    for instance in instances:
                        # Mark as unhealthy if no heartbeat for 30 seconds
                        if current_time - instance['last_heartbeat'] > 30:
                            instance['healthy'] = False
            
            time.sleep(10)  # Check every 10 seconds

    def health_check(self):
        """Perform health checks on all registered services"""
        import requests
        
        with self.services_lock:
            for service_name, instances in self.services.items():
                for instance in instances:
                    if instance['health_check_url']:
                        try:
                            response = requests.get(instance['health_check_url'], timeout=5)
                            instance['healthy'] = response.status_code == 200
                        except Exception:
                            instance['healthy'] = False

# Service Discovery Client
class ServiceDiscoveryClient:
    def __init__(self, registry: ServiceRegistry):
        self.registry = registry

    def discover_service(self, service_name: str) -> dict:
        """Discover a service instance (simple round-robin)"""
        instances = self.registry.get_service_instances(service_name)
        if not instances:
            return None
        
        # Simple round-robin selection
        return instances[0]

# Example usage
def main():
    # Create registry
    registry = ServiceRegistry()
    
    # Register services
    registry.register_service("user-service", "192.168.1.100", 8080, "http://192.168.1.100:8080/health")
    registry.register_service("user-service", "192.168.1.101", 8080, "http://192.168.1.101:8080/health")
    registry.register_service("order-service", "192.168.1.102", 8080, "http://192.168.1.102:8080/health")
    
    # Discover service
    client = ServiceDiscoveryClient(registry)
    user_service = client.discover_service("user-service")
    
    if user_service:
        print(f"Found user service at {user_service['host']}:{user_service['port']}")
    else:
        print("User service not found")

if __name__ == "__main__":
    main()
```

### AWS Technologies for Service Registry and Discovery
- AWS Cloud Map
    - **What:** Managed service discovery service
    - **Best for:** Microservices architectures

**Example:**

```python
# Service Registration
import boto3

servicediscovery = boto3.client('servicediscovery')

# Register service instance
servicediscovery.register_instance(
    ServiceId='srv-xyz123',
    InstanceId='user-service-1',
    Attributes={
        'AWS_INSTANCE_IPV4': '10.0.1.100',
        'AWS_INSTANCE_PORT': '8080'
    }
)

# Client Discovery
instances = servicediscovery.discover_instances(
    NamespaceName='myapp.local',
    ServiceName='user-service'
)

service_url = f"http://{instances['Instances'][0]['Attributes']['AWS_INSTANCE_IPV4']}:{instances['Instances'][0]['Attributes']['AWS_INSTANCE_PORT']}"
```

- Application Load Balancer (ALB) + Target Groups
    - **What:** Load balancer with built-in service discovery
    - **Best for:** Container workloads (ECS/EKS)

**Example:**

```python
# Client just calls the ALB endpoint
import requests

# ALB routes to healthy instances
response = requests.get('https://api.myapp.com/users')
```

- ECS Service Discovery
    - **What:** Built into ECS, uses Cloud Map under the hood
    - **Best for:** ECS tasks

**Example:**

```yaml
# ECS Task Definition
services:
  user-service:
    image: user-service:latest
    # Automatically registered as user-service.myapp.local
```

```python
# Client discovery
response = requests.get('http://user-service.myapp.local:8080/api/users')
```

- Route 53 (DNS-based)
    - **What:** DNS service for simple service discovery
    - **Best for:** Simple setups, health checks

**Recommendation:** Use **AWS Cloud Map** for most microservices scenarios - it's purpose-built for this and integrates well with ECS/EKS.

⚙️ Redlock in Redis
What Is Redlock?  
Redlock is a distributed locking algorithm that provides exclusive access to shared resources across multiple Redis nodes. It ensures that only one client can hold a lock at any given time by requiring consensus from a majority of Redis masters.

Core Concepts
- **Distributed Lock**: Obtains time-bounded leases (locks) across multiple independent Redis nodes
- **Core Guarantee**: Only one client can hold the lock at any moment, as long as the lock's TTL hasn't expired
- **Key Strategy**: Acquire locks on a majority ([N/2] + 1) of N Redis masters within a short time window

### Algorithm Deep Dive

**1. Preparation Phase**: Before attempting to acquire a lock, we need to configure the following parameters:

- **Essential Parameters:**
- **N**: Total number of Redis masters (recommended: 5)
- **Quorum**: [N/2] + 1 (e.g., 3 out of 5 nodes)
- **Lock TTL**: Maximum time a lock can be held (e.g., 30 seconds)
- **Timeout**: Maximum time to wait for lock acquisition process

**2. Lock Acquisition Phase**: The acquisition process follows these critical steps:

**Step 1: Record Start Time**
```
start_time = current_time_ms()
```

**Step 2: Sequential Lock Attempts** on each Redis master execute:
```
SET resource_name token PX 30000 NX
```

**Step 3: Evaluate Success** Current timestamp - start_time < lock_validity_period (time
```
elapsed_time = current_time_ms() - start_time
valid_time = lock_ttl - elapsed_time - clock_drift
```

**Step 4: Determine Lock Status** Lock is successfully acquired if **both conditions are met:**
• **Majority rule**: >= quorum
• **Timing rule**: valid_time > 0

If successful, the lock is valid for (TTL - elapsed_time - clock_drift) milliseconds.

**3. Handle Failure If either condition fails, immediately release all acquired locks.**

**Lock Release Phase**: Send release command to all Redis masters to prevent accidental cleanup of other clients' locks:

**Script for Safe Release:**
```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

This ensures atomicity and one-side — No lock token can release from some lock.

### Safety vs Liveness Trade-offs

**TTL Configuration Strategy**  
- **Best Practices (30 seconds):**
    - Allow enough time for business operations
    - Include network latency and processing time

- **Long TTL (e.g., 30 seconds):**
    - **Advantage**: Safe for lengthy operations
    - **Disadvantage**: Slow recovery during failures

- **Recommended Configuration:**
    - **Short lock operations**: 5-10 seconds
    - **Long operations**: 30-60 seconds
    - **Design management operations**: 2-5 minutes

Let's run through a concrete example with N=5, quorum=3, TTL=30 seconds:
```
Time: 0ms    t0  = start_time
Node 1: SET resource_name 12345 PX 30000 NX  ✓
Node 2: SET resource_name 12345 PX 30000 NX  ✓
Node 3: SET resource_name 12345 PX 30000 NX  ✓
Node 4: SET resource_name 12345 PX 30000 NX  ✗
Node 5: SET resource_name 12345 PX 30000 NX  ✗

Time: 25ms
Successfully acquired: 3/5 (quorum met)       ✓
Elapsed time: 25ms (within TTL)               ✓
Valid time: 30000 - 25 - 50 = 29925ms        ✓
```

- Key Observations:
    - Redis's old lock doesn't prevent new lock acquisition once quorum is met
    - The old lock will expire naturally on its own schedule
    - If the new lock's TTL expires before work completes, other clients can acquire the lock using the same process

- Best Practices
    - **Choose odd numbers of Redis masters** (3, 5, 7) to avoid split-brain scenarios
    - **Monitor lock acquisition latency** to detect network issues
    - **Implement proper error handling** for partial lock acquisition failures
    - **Use independent Redis instances** (not master-slave pairs) for true fault tolerance
    - **Test failure scenarios** including network partitions and Redis node failures
---

## Database
⚙️ `UNION` and `OR` in MySQL  
`UNION` processes separate result sets by executing each `SELECT` statement independently (but typically on the same thread), then merging the results. This allows MySQL to optimize each query separately and use different execution plans.

**Example scenario: Finding employees earning \>50K OR working in Sales department.**

**Using UNION:**

```sql
SELECT * FROM employees WHERE salary > 50000
UNION
SELECT * FROM employees WHERE department = 'Sales';
```

**How UNION works:**

1.  Executes first query using salary index → Result Set A
2.  Executes second query using department index → Result Set B
3.  Combines and deduplicates → Final Result

**Using OR:**

```sql
SELECT * FROM employees
WHERE salary > 50000 OR department = 'Sales';
```

Why `UNION` is often better:
    - Each subquery can use its optimal index (`salary_idx`, `department_idx`)
    - MySQL optimizes each `SELECT` independently
    - Avoids complex condition evaluation across multiple columns

**When to use OR instead of UNION:** Use OR when querying the same column with multiple values:

```sql
-- Good use of OR
SELECT * FROM employees
WHERE department = 'Sales' OR department = 'Marketing' OR department = 'IT';

-- Don't use UNION here - it's overkill and less efficient
```

**Threading note:** `UNION` typically executes on a single thread sequentially, not parallel threads. The performance benefit comes from better index usage and query optimization, not parallelization.

⚙️ Transaction in SQL following ACID  
What is a Transaction?  
A transaction is a group of SQL operations that must all succeed or all fail together. Think of it like transferring money between bank accounts, we can't have the money leave one account without arriving in the other.

**Basic Transaction Syntax in MySQL**

```sql
START TRANSACTION;

-- Our SQL operations here
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT; -- Makes changes permanent
-- OR
ROLLBACK; -- Undoes all changes if something goes wrong
```

### ACID Properties Explained
- **Atomicity - All or Nothing**
    - Either all operations in the transaction complete successfully, or none do
    - If any operation fails, the entire transaction is rolled back
    - **Example:** If the second `UPDATE` fails, the first `UPDATE` is automatically undone

- **Consistency - Data Rules Are Maintained**
    - Database remains in a valid state before and after the transaction
    - All constraints, triggers, and rules are enforced
    - **Example:** Total money in the system stays the same after the transfer

- **Isolation - Transactions Don't Interfere**
    - Concurrent transactions don't see each other's uncommitted changes
    - MySQL uses different isolation levels (READ COMMITTED, REPEATABLE READ, etc.)
    - **Example:** Another user checking balances during the transfer won't see partial results

- **Durability - Changes Are Permanent**
    - Once `COMMIT` is executed, changes survive system crashes
    - Data is written to disk, not just kept in memory
    - **Example:** Even if the server crashes after `COMMIT`, the money transfer is saved

- Key Points
    - Transactions solve the problem of partial failures in multi-step operations
    - ROLLBACK happens automatically if any error occurs (or manually with ROLLBACK command)
    - MySQL's default isolation level is REPEATABLE READ
    - Use transactions for any operation that involves multiple related changes
    - Always handle exceptions in application code to ensure proper ROLLBACK

This ensures data integrity in scenarios like financial transfers, inventory updates, or any multi-table operations.

---

## Microservice patterns
⚙️ SAGA pattern  
The **Saga pattern** is a way to manage distributed transactions across multiple microservices without using traditional two-phase commits, which don't scale well in microservices architectures.

Core Concept: Instead of one big transaction, we break it into a series of smaller, independent transactions. If something fails partway through, we run compensating transactions to undo the previous steps.

How It Works  
There are two main approaches:
- **Choreography** - Services communicate directly through events
- **Orchestration** - A central coordinator manages the workflow

E-commerce Order Processing  
Let's say we're placing an order on Amazon:

Services Involved:
- Order Service
- Payment Service
- Inventory Service
- Shipping Service

**Orchestration Approach**  
Technology Stack:
- **Orchestrator:** Separate microservice (often called "Order Workflow Service")
- **Technology:** Temporal, Netflix Conductor, or custom service using Spring Boot
- **Communication:** Synchronous REST calls or asynchronous message queues
- **State Management:** Database to track saga state and progress

Flow:
```
Order Orchestrator receives "Place Order" request
↓
1. Order Service: Create order (PENDING) → REST call
2. Payment Service: Charge credit card → REST call
3. Inventory Service: Reserve items → REST call
4. Shipping Service: Schedule delivery → REST call
5. Order Service: Mark order as CONFIRMED → REST call
```

If Step 3 Fails (Out of Stock):
```
4. [Skip - never executed]
3. [Failed - nothing to compensate]
2. Payment Service: Refund charge → compensating REST call
1. Order Service: Cancel order → compensating REST call
```

**Choreography Approach**  
Technology Stack:
- **Event Bus:** Apache Kafka, AWS EventBridge, or RabbitMQ
- **Communication:** Asynchronous event publishing/subscribing
- **No central orchestrator** - each service knows what to do next

Flow:
```
1. Order Service: Creates order → publishes "OrderCreated" event to Kafka
2. Payment Service: Subscribes to "OrderCreated" → charges card → publishes "PaymentProcessed"
3. Inventory Service: Subscribes to "PaymentProcessed" → reserves items → publishes "ItemsReserved"
4. Shipping Service: Subscribes to "ItemsReserved" → schedules delivery → publishes "ShippingScheduled"
5. Order Service: Subscribes to "ShippingScheduled" → marks order CONFIRMED
```

If Step 3 Fails:
```
3. Inventory Service: publishes "ItemReservationFailed" event
2. Payment Service: Subscribes to failure event → refunds charge → publishes "PaymentRefunded"
1. Order Service: Subscribes to "PaymentRefunded" → cancels order
```
- Orchestrator Technologies:
    - **Temporal:** Workflow-as-code platform, handles retries and state
    - **Netflix Conductor:** Microservices orchestration engine
    - **Custom:** Spring Boot service with state machine (Spring State Machine)
    - **Cloud:** AWS Step Functions, Azure Logic Apps

- Event Technologies:
    - **Message Brokers:** Apache Kafka, RabbitMQ, AWS SQS/SNS
    - **Event Sourcing:** Store events in event store (EventStore, Apache Kafka)
    - **Database:** Outbox pattern with database triggers or CDC (Change Data Capture)

- State Management:
    - **Orchestration:** Saga state stored in database (PostgreSQL/MongoDB)
    - **Choreography:** Each service tracks its own state, events provide coordination

- Key Benefits:
    - **No distributed locks** - each service manages its own data
    - **Fault tolerance** - system can recover from failures
    - **Scalability** - services remain independent

- Points to Remember:
    - **Orchestration** = centralized control, easier debugging, single point of failure
    - **Choreography** = decentralized, more resilient, harder to track overall flow
    - Saga ensures **eventual consistency**, not immediate consistency
    - Compensating transactions must be **idempotent** (safe to retry)
    - **Outbox pattern** often used with choreography to ensure reliable event publishing
    - Choose orchestration for complex workflows, choreography for simpler ones

The pattern trades immediate consistency for availability and partition tolerance - classic CAP theorem.

⚙️ Timeout pattern  
Core Concept: Set a maximum wait time for any service call - if no response comes within that limit, automatically terminate and handle gracefully instead of waiting forever.

Key Mental Model Points
- Defense Against Hanging
    - Prevents the service from getting stuck waiting indefinitely for slow/dead dependencies
    - Think of it as a "panic button" that automatically activates
- Resource Protection
    - Frees up threads, memory, and connections that would otherwise be locked up
    - Stops one slow service from bringing down the entire system
- Cascading Failure Prevention
    - A slow service can cause other services to pile up waiting for it
    - Timeouts break this chain reaction before it spreads
- Graceful Degradation
    - When timeout hits, return cached data, default values, or user-friendly error messages
    - Better to give users something than nothing
- Different Timeout Types
    - **Connection:** How long to wait to establish connection
    - **Read/Response:** How long to wait for data after connecting
    - **Global:** Overall time limit for complex multi-service operations
- Complement with Other Patterns
    - **Retry:** Try again after timeout (with backoff)
    - **Circuit Breaker:** Stop calling failing services temporarily

The mental model: Think of timeouts as automatic "escape hatches" that keep our system responsive and prevent resource starvation when things go wrong.

⚙️ Strangler Fig pattern  
The Strangler Fig Pattern is a strategy for incrementally refactoring a monolithic application into microservices by diverting traffic from the monolith to new services, piece by piece, until the monolith can be decommissioned. This approach minimizes risk during migration by allowing new functionalities to be built and tested independently, while the existing system remains operational.

- Points:
    - **Monolith:** The large, tightly coupled, legacy application being refactored.
    - **New System/Microservices:** Smaller, independent services built to replace parts of the monolith's functionality.
    - **Strangler Application/Facade:** An intermediary layer (e.g., a proxy, API gateway, or a new application) that intercepts requests intended for the monolith and routes them to either the monolith or the new services.
    - **Incremental Migration:** The core principle of the pattern, where functionality is moved from the monolith to new services in small, manageable steps.
    - **Traffic Diversion:** The process of routing specific requests away from the monolith and towards the new services.
    - **Functionality Extraction:** Identifying a distinct business capability within the monolith to be extracted into a new service.
    - **Decommissioning:** The final step where the original monolith is retired once all its functionality has been successfully migrated.
    - **Reduced Risk:** By migrating incrementally, the impact of potential issues is localized to smaller parts of the system, rather than affecting the entire monolith.
    - **Coexistence:** The monolith and the new services operate simultaneously during the migration period.
    - **Non-Invasive:** The pattern allows for refactoring without requiring a complete rewrite or taking the entire system offline.

⚙️ Sidecar pattern 
Core Concept
- **Co-located Helper:** A sidecar is a separate container that runs alongside our main application in the same pod/host, handling cross-cutting concerns so our app can focus purely on business logic

Key Mental Models
- **"Personal Assistant" Model:** Think of the sidecar as a dedicated assistant that handles all the "administrative tasks" (logging, security, networking) while our main application focuses on its core job
- **Shared Resources, Separate Responsibilities:** Both containers share the same CPU, memory, and network (communicate via `localhost`), but each has distinct roles - like roommates sharing an apartment but having different jobs
- **Language-Agnostic Infrastructure:** Our Java app can use a C++ Envoy proxy sidecar for networking without any code changes - the sidecar handles the complexity

Real-World Examples
- **Service Mesh:** Envoy proxy sidecar in Istio/Linkerd handles all network traffic, load balancing, and security between microservices
- **Logging:** Fluent Bit sidecar collects logs from the app and ships them to centralized logging (ELK stack)
- **Configuration:** Consul Connect sidecar pulls config updates and injects them into our app

Key Trade-off
- **Benefit:** Clean separation + reusability across different apps and languages
- **Cost:** Higher resource usage (now running 2+ containers per application instance)

The mental model is essentially "modular infrastructure" - instead of baking cross-cutting concerns into every application, we compose them as separate, reusable components.

### Sidecar Demo Example

In this example, the **"GIT Sync Container"** is acting as the sidecar to the **"NGINX Server"**.

Here's how it works simply:

- **GIT Sync Container (Sidecar):** This container's job is to "Fetch" updates (like `index.html` or other website files) from the "Git Repository".
- **Shared Filesystem:** The Git Sync Container then "Updates" these fetched files onto a "Shared Filesystem". Both the Git Sync Container and the NGINX Server can access this same shared storage.
- **NGINX Server (Main Application):** The NGINX server, which is a web server, "Reads" the `index.html` (and other files) directly from this Shared Filesystem to serve them.
- **In essence:** The NGINX server doesn't need to know anything about Git or how to fetch files. Its only job is to serve files. The "GIT Sync Container" sidecar handles the entire process of getting the latest website content and placing it where NGINX can find it, keeping the NGINX server focused on its primary task.

<img src="https://github.com/user-attachments/assets/a9c724a6-b3a5-405d-b452-97a1aed40ad5" width="400" height="300">
<br>
<a href="https://images.app.goo.gl/5Ku1To2TkrhvUdFHA">Ref</a>

---

<img src="https://github.com/user-attachments/assets/0ba01175-dc4a-4af3-a4a9-b2b0487ffa4e" width="400" height="300">
<br>
<a href="https://images.app.goo.gl/eER2ottwo831KbKN9">Ref</a>

⚙️ Retry pattern 
1. Reattempt failed operation after a delay.
2. Addresses transient faults (temporary network issues, service unavailability, transient load spikes) that are expected to resolve themselves.
3. Max retry parameter is used to define max number of retries that can be done.
4. Exponential backoff: the delay increases exponentially with each subsequent retry (1s, 2s, 4s, 8s). This is to avoid overwhelming a struggling service.
5. Jitter: Randomness added to the delay.
6. Operations being retried should ideally be idempotent.
7. E.g. Tenacity in Python

```python
from tenacity import retry, wait_fixed, stop_after_attempt, after_log
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@retry(wait=wait_fixed(2), stop=stop_after_attempt(3), after=after_log(logger, logging.WARNING))
def call_external_service():
    """Simulates a call to an external service that might fail."""
    import random
    if random.random() < 0.7:  # 70% chance of failure
        logger.warning("External service call failed.")
        raise ConnectionError("Simulated network error")
    logger.info("External service call successful!")
    return "Data from service"

if __name__ == "__main__":
    try:
        result = call_external_service()
        print(f"Operation completed: {result}")
    except Exception as e:
        print(f"Operation failed after retries: {e}")
```

⚙️ Outbox pattern   
Problem: Reliably publish events when updating database without losing events if message broker fails.
Solution: Store events in same database transaction as business data, publish separately.

Example:
```sql
CREATE TABLE orders (id, customer_id, amount, status);
CREATE TABLE outbox_events (id, event_type, payload, published);
```

Service Code
```python
def create_order(order_data):
    with db.transaction():
        # Step 1: Save business data
        order = db.execute(
            "INSERT INTO orders (customer_id, amount, status) VALUES (?, ?, 'PENDING')",
            [order_data['customer_id'], order_data['amount']]
        )
        
        # Step 2: Save event (same transaction)
        db.execute(
            "INSERT INTO outbox_events (event_type, payload, published) VALUES (?, ?, false)",
            ["OrderCreated", json.dumps({'order_id': order.id, 'amount': order.amount})]
        )
        
        # Both succeed or both fail atomically
```

Background Publisher
```python
def event_publisher():
    while True:
        # Get unpublished events
        events = db.execute("SELECT * FROM outbox_events WHERE published = false")
        
        for event in unpublished:
            kafka.send('order-events', event.payload)  # Publish to message broker
            db.execute("UPDATE outbox_events SET published = true WHERE id = ?", [event.id])
        
        time.sleep(1)
```

Flow:
    - **Write**: Business data + event stored atomically in DB
    - **Publish**: Background process publishes events from outbox
    - **Mark**: Published events marked as complete
    - **Retry**: Failed events stay unpublished, retry automatically

Key Benefit:
    - **No lost events** - even if Kafka/RabbitMQ is down, events are safely stored in database and published when broker recovers.

Solves "dual write problem" - guarantees both database update AND event publishing happen together.

⚙️ [Command Query Responsibility Segregation (CQRS)](https://microservices.io/patterns/data/cqrs.html)

⚙️ Circuit breaker  
- Prevents cascading failures.
- When service experiences repeated failures, the “circuit” to that service “trips”.
- Circuit breaker monitors calls to a service, tracking success and failure rates.
- 3 states
    - Closed: Requests are allowed through to the service. Failures are counted.
    - Open: If the failure rate exceeds a configurable threshold within a defined time window, the circuit “trips” open. Subsequent requests are rejected.
    - Half-Open: After a configured timeout in the “Open” state, the circuit transitions to “Half-Open”. A limited number of test requests are allowed through the service.
- Recovery probe: In the “Half-Open” state, if the test request succeed, the circuit returns to “Closed”. If they “fail”, it returns to “Open” for another timeout period.
- Fallback mechanism: Often combined with fallback strategy (e.g. returning cached data, default values or an error) when the circuit is open, providing a graceful degradation of service.
- Technologies: Netflix Hystrix, Resilience4j, Istio/Linkerd.

⚙️ Canary deployment pattern
- Deploying new version (“the canary”) to a small percentage of live traffic or users first, rather than switching all traffic at once.
- Allows real world testing and monitoring of the new version with minimal impact.
- Switch using load balancers based on rules (e.g. 5% to canary, 95% to old).
- This can also be based on specific user segments (e.g. internal employees, beta testers).
- Can be combined with A/B testing, where different versions are shown to specific user segments to evaluate the business impact.
- Similar to Blue-Green, managing database changes can be complex, especially if Canary interacts with the same DB as the old version. Backward compatibility is the key.

⚙️ Bulkhead pattern  
Core Concept: Isolates resources to prevent cascading failures - if one component fails, others remain unaffected.

- Key Applications
    - **Thread Pools**: Separate pools for different external calls/services
    - **Connection Pools**: Isolated database/API connections per dependency
    - **Circuit Breaker Integration**: Bulkhead isolates, Circuit Breaker detects/trips on failures

Mental Model: **Ship with watertight compartments** - if one compartment floods (dependency fails), water stays contained and other compartments remain dry and functional.

Example
- E-commerce service uses:
    - Dedicated thread pool for payment API calls
    - Separate thread pool for inventory service calls
    - Isolated connection pool for user database

If payment API becomes slow/unresponsive, it only exhausts its dedicated resources - inventory and user operations continue normally.

## Python Example

```python
from concurrent.futures import ThreadPoolExecutor

# Separate thread pools (bulkheads)
payment_pool = ThreadPoolExecutor(max_workers=5)
inventory_pool = ThreadPoolExecutor(max_workers=3)

# If payment API hangs, only uses its 5 threads
payment_pool.submit(call_payment_api, order_data)

# Inventory calls remain unaffected
inventory_pool.submit(check_inventory, product_id)
```

Prevents resource exhaustion in one area from bringing down the entire system.

⚙️ Blue green deployment pattern  
- Near zero downtime and rapid rollback.
- Two production envs: blue and green. Only one is active at a given time. Other remains idle or is used for staging and testing the new version.
- Blue: represents the current stable version of the application this is actively serving live user traffic.
- Green: represents the new version of the application that is being prepared for deployment. It’s an exact replica of the blue environment, but runs the updated code.
- Traffic is switched using load balancer (most common), DNS (less common, DNS propagation delays).
- If green environment proves stable for a given period of time, blue environment can be decommissioned, or it can become the green environment for the next deployment.
- Resource intensive: maintaining 2 full production grade environments.
- Database schema change and data synchronization between blue and green can be complex especially with backward incompatible changes. Strategies like dual writing, schema evolution, or feature flags might be needed.
- In canary, it’s gradual roll out to a subset of users. Here, it’s instantaneous switch.
