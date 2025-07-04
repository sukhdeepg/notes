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

⚙️ Pagination offset and cursor based
```javascript
// api/users.js
app.get('/users', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = 10;
  const offset = (page - 1) * limit;

  const { rows } = await pool.query(
    'SELECT * FROM users ORDER BY id LIMIT $1 OFFSET $2',
    [limit, offset]
  );
  res.json(rows);
});
```

```sql
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 500000;
```

**Real-world impact**: If we have 1,000,000 pages of 10 items each, loading page 50,000 could scan and sort records for page 1 before it gets to page 50,000.

**Pros**: Simple, supports jumping to specific pages.
**Cons**: Performance degrades with large offsets, inconsistent results during data changes.

Cursor-Based Pagination
```javascript
// api/users.js
app.get('/users', async (req, res) => {
    const cursor = req.query.cursor || null;
    const limit = 10;

    let query;
    let params;

    if (cursor) {
        query = 'SELECT * FROM users WHERE id > $1 ORDER BY id LIMIT $2';
        params = [cursor, limit];
    } else {
        query = 'SELECT * FROM users ORDER BY id LIMIT $1';
        params = [limit];
    }

    const { rows } = await pool.query(query, params);
    const nextCursor = rows.length === limit ? rows[rows.length - 1].id : null;

    res.json({
        data: rows,
        nextCursor,
    });
});
```

**How Cursor-Based Pagination Works**

The key is using a unique, sequential column (like `id` or `created_at`) as a "cursor". Instead of "skip N rows", the query becomes "get rows after this specific value".

```sql
-- First page
SELECT * FROM users ORDER BY id LIMIT 10;

-- Let's say the last user ID from the first page was 10
-- Next page
SELECT * FROM users WHERE id > 10 ORDER BY id LIMIT 10;
```

**Implementation Steps:**
- **Initial Request**: Client requests the first page without a cursor.
- **Server Response**: Server fetches the first `N` items. It includes the data and a `nextCursor`, which is the value of the cursor column (e.g., `id`) of the last item.
- **Subsequent Requests**: For the next page, the client sends the `nextCursor` it received.
- **Server Query**: The server uses this cursor in the `WHERE` clause to fetch items that come after that cursor value.
- **End of Data**: If the number of items returned is less than the limit, or if no items are returned, the server sends a `null` or empty `nextCursor`, signaling the end.

```javascript
// api/posts.js

// GET /posts?cursor=...
// ... (imports and setup)

app.get('/posts', async (req, res) => {
  const limit = 10;
  const cursor = req.query.cursor ? parseInt(req.query.cursor, 10) : null;

  try {
    const query = {
      // Use a unique, sequential column for the cursor.
      // Here we use `id`, but a timestamp like `created_at` is also common.
      take: limit,
      orderBy: {
        id: 'asc', // Or 'desc', depending on our desired order
      },
      ...(cursor && {
        // The `cursor` object tells Prisma where to start fetching.
        cursor: {
          id: cursor,
        },
        // `skip: 1` is crucial to avoid refetching the cursor item itself.
        skip: 1,
      }),
    };

    const posts = await prisma.post.findMany(query);

    // The next cursor is the ID of the last item in the returned list.
    const nextCursor = posts.length === limit ? posts[posts.length - 1].id : null;

    res.json({
      posts,
      nextCursor,
    });
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: "Something went wrong" });
  }
});

// Pros: Efficient, scalable, provides stable pagination even with frequent writes.
// Cons: More complex to implement, doesn't allow jumping to a specific page.
```

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

⚙️ Image upload flow with object store (s3) and CDN (CloudFront)  
How browsers know about CDNs  
The browser doesn't automatically "know" about CDNs. Here's what actually happens:

**Our application tells the browser** - When our app serves HTML/CSS/JS, we specify the CDN URLs in our code. For example:
```html
<img src="https://cdn.example.com/images/profile.jpg">
```

**The browser just follows instructions** - It sees that URL and makes a request to wherever we told it to go (the CDN domain).

**We choose the URLs** - As a developer, we decide whether to use:
- Direct server URLs: `https://myserver.com/images/profile.jpg`
- CDN URLs: `https://cdn.example.com/images/profile.jpg`

**The flow is:**
- Our backend uploads to S3
- Our backend returns a CDN URL to our frontend (not the direct S3 URL)
- Our frontend uses that CDN URL in the HTML
- Browser requests from CDN
- If CDN doesn't have it (cache miss), CDN fetches from S3

**Key point:** The browser doesn't "discover" CDNs - we explicitly configure our application to use CDN URLs instead of direct server URLs. It's a conscious architectural choice we make when building our system.

Example: User Profile Image Upload/Access  
**Setup:**
- S3 Bucket: `myapp-images`
- CloudFront Distribution: `d1a2b3c4d5e6f7.cloudfront.net`
- CloudFront Origin: `myapp-images.s3.amazonaws.com`

Flow 1: Pre-signed URLs WITHOUT CDN  
**Upload Flow:**
- User clicks "Upload Profile Picture"
- Frontend: `POST /api/get-upload-url`
- Service generates pre-signed URL and returns:
```
https://myapp-images.s3.amazonaws.com/users/123/profile.jpg?AWSAccessKeyId=AKIAI...&Expires=1609459200&Signature=xyz...
```
- Frontend uploads directly to this S3 URL
- Image stored in S3: `users/123/profile.jpg`

**Access Flow:**
- User views profile page
- Service returns direct S3 URL:
```
https://myapp-images.s3.amazonaws.com/users/123/profile.jpg?AWSAccessKeyId=AKIAI...&Expires=1609459200&Signature=abc...
```
- Browser fetches directly from S3

Upload/Update Flow:  
- User -\> uploads image via our application
- Service -\> receives image, processes if needed (resize, validation, etc.)
- Service -\> uploads image to S3 with new/updated key
- Service -\> triggers CDN invalidation (via CDN API) for the specific image path
- Service -\> returns success response to user with new image URL

**How new image gets added to CDN:**
- User/Browser -\> requests the new image URL: `GET https://cdn.example.com/images/profile.jpg`
- CDN -\> cache miss, makes HTTP request to S3: `GET https://mybucket.s3.amazonaws.com/images/profile.jpg`
- S3 -\> checks if object exists at that exact key path (`images/profile.jpg`)
- S3 -\> if exists: returns image data + metadata; if not: returns 404 error
- CDN -\> caches the response (image or 404) and serves it to user
- Subsequent requests -\> served directly from CDN cache

**Cache-busting explained:** Problem: Image updated but CDN still serves old cached version

**Solution 1 - Timestamp in filename:**
- Old: `profile.jpg`
- New: `profile-1698765432.jpg` (different filename = different URL = no cache conflict)

**Solution 2 - Query parameter:**
- Old: `profile.jpg?v=1`
- New: `profile.jpg?v=2` (different URL = bypasses cache)

Cache-busting works because each update creates a completely new URL that the CDN has never seen before.

CloudFront doesn't automatically discover S3 - we configure it:

**We manually set up the connection:** When we create a CloudFront distribution, we explicitly tell it:
- "Hey CloudFront, our origin is this specific S3 bucket: `mybucket.s3.amazonaws.com`"
- We configure this in the CloudFront console/API

**CloudFront configuration example:**
```
Origin Domain: mybucket.s3.amazonaws.com
Origin Path: /images (optional)
```
**The flow becomes:**
- User requests: `https://d1234.cloudfront.net/profile.jpg`
- CloudFront checks its cache - cache miss\!
- CloudFront thinks: "My configured origin is `mybucket.s3.amazonaws.com`"
- CloudFront makes request: `https://mybucket.s3.amazonaws.com/profile.jpg`
- S3 returns the image
- CloudFront caches it and serves to user

**Key insight:** CloudFront is essentially a "smart proxy" that we configure to point to our S3 bucket. It's not automatic discovery - it's explicit configuration.

**We set up three things:**

1.  S3 bucket (our storage)
2.  CloudFront distribution (our CDN)
3.  The connection between them (origin configuration)

Think of it like telling a delivery service: "When someone asks us for something, go check warehouse \#5 on Main Street."

Flow 2: Pre-signed URLs WITH CDN
**Upload Flow:**
- User clicks "Upload Profile Picture"
- Frontend: `POST /api/get-upload-url`
- Service returns **same S3 pre-signed URL** (upload always goes to S3):
```
https://myapp-images.s3.amazonaws.com/users/123/profile.jpg?AWSAccessKeyId=AKIAI...&Expires=1609459200&Signature=xyz...
```
- Frontend uploads directly to S3
- Image stored in S3, **CDN cache is empty**

**Access Flow (First Request):**
- User views profile page
- Service returns **CDN URL with signed parameters:**
```
https://d1a2b3c4d5e6f7.cloudfront.net/users/123/profile.jpg?AWSAccessKeyId=AKIAI...&Expires=1609459200&Signature=abc...
```
- Browser requests from CloudFront
- **CloudFront cache miss** -\> CloudFront fetches from S3
- **Image now cached in CloudFront**
- CloudFront serves to browser

**Subsequent Requests:**  
Same CDN URL served from CloudFront cache (fast\!)

**Key:** Upload path identical, access path uses CDN domain instead of S3 domain.

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

### Network Partition Scenario:

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

<img src="https://github.com/user-attachments/assets/da24272b-93d5-425d-b4f2-ed946c269777" width="400" height="300">
<br>
<a href="https://images.app.goo.gl/huC8eWj9jsZfGF5Q8">Ref</a>

⚙️ Redis Streams  
Redis Streams provide **conditional at-least-once** delivery semantics, not at-most-once, but with important limitations.

- How Redis Streams Work:
    - When a consumer reads a message with `XREADGROUP`, the message is added to the **Pending Entries List (PEL)**
    - The message remains in the PEL until explicitly acknowledged with `XACK`
    - If the consumer crashes before sending `XACK`, the message stays in the PEL and can be reclaimed by another consumer using `XPENDING` and `XCLAIM`

- At-most-once vs At-least-once:
    - **At-most-once**: Message is delivered 0 or 1 times (risk of message loss, no duplicates)
    - **At-least-once**: Message is delivered 1 or more times (no message loss, but possible duplicates)

- Redis Streams Delivery Guarantees
    - Against consumer failures: True at-least-once
        - Consumer crashes after reading but before `XACK`
        - Message remains in PEL, can be reclaimed by another consumer
        - Same message may be processed multiple times

- Against Redis server failures: No guarantee
    - **PEL is stored in Redis memory**: If Redis crashes, PEL is lost
    - **All unacknowledged messages are lost**: no way to reclaim them
    - **Stream data itself can be lost** without proper persistence
    - **Consumer group state is lost**: positions, tracking, everything

- Persistence limitations:
    - **RDB snapshots**: Gap between snapshots = potential data loss
    - **AOF logging**: Still has fsync gaps, performance impact
    - Even with persistence, we lose data between last write and crash

- Industry Example - Fintech Trading Platform:
    - A fintech platform uses Redis Streams for trade events with three consumer groups (Risk Analysis, Settlement Engine, Audit Logger).
        - **What works:** If a risk analyst consumer crashes while processing a trade, another consumer can reclaim and reprocess it from the PEL.
        - **What fails:** If Redis crashes during market hours, all unacknowledged trades in PELs are lost forever - potentially millions of dollars in unprocessed transactions. This is why real fintech systems:

- Use Redis Streams for **non-critical, high-speed processing**
- Combine with persistent message brokers (Kafka) for **critical trade data**
- Implement **dual-write patterns** where critical events go to both systems
- Use Redis clustering/replication for high availability

**Bottom Line:** Redis Streams provide at-least-once delivery **only** for consumer-level failures. For true at-least-once guarantees across all failure scenarios, we need distributed message broker designed for durability.

⚙️ Redis NX  
- **Command:** `SET <key> <value> NX EX <ttl>`
- **When used:** to atomically create a key only if it doesn't already exist, and have it expire after `<ttl>` seconds.
- **NX (Only set if Not eXists):** prevents overwriting an existing key—returns `OK` only when the key is new; otherwise returns `nil`.
- **EX `<ttl>` (Expire):** sets a time-to-live in seconds so the key auto-deletes after `<ttl>` seconds.
- **Use case:** common for distributed locks or deduplication—e.g., `SET lock_id token NX EX 30` acquires a 30s lock only if nobody else holds it.

⚙️ Redis NX working in distributed architecture  
Idempotent Consumer (Dedup)
```
SET processed:<msgID> 1 NX EX 3600
```
- Only the first `SET` succeeds (others return `nil`) → first consumer processes; retries within 1h are dropped.
- Atomic Check-and-Set (NX)
    - **NX flag ensures `SET` only if key is absent, and is atomic because Redis processes commands sequentially in a single-threaded loop.**
- TTL: EX vs. PX
    - **EX `<seconds>`** → key expires after N seconds
    - **PX `<milliseconds>`** → key expires after N ms (millisecond precision)
- Distributed Sharding (Cluster)
    - Keys routed to shards by `CRC16(key) mod 16384`; each shard is a standalone Redis instance retaining per-command atomicity.

Quorum Lock (Redlock)
```
SET lock:<res> <token> NX PX 30000
```

Client attempts lock on multiple masters; lock is granted when a majority respond OK, preventing single-node failure issues.

⚙️ Redis Caching Strategy: Key-Value Basics to TTL, Invalidation & Tagging  
Redis at its core is simple - we store data with a key and retrieve it later:
```redis
# Store a value
SET "my-key" "my-value"

# Get a value
GET "my-key"
# Returns: "my-value"
```

That's it! Redis is just a giant dictionary/map where we store `"key"` → `"value"` pairs.

**Key Naming: Why Use Colons?**  
Since Redis only has one flat namespace (no folders), developers use colons `:` to organize keys:
```redis
# Instead of random names:
SET "abc123" "Taylor Swift Concert"
SET "xyz789" "Madison Square Garden"

# Use organized names:
SET "event:12345" "Taylor Swift Concert"
SET "venue:msg" "Madison Square Garden"
SET "search:taylor-swift:2024" "list of events"
```

The colons are just part of the key name - like naming files `photo_vacation_beach.jpg` instead of `photo1.jpg`. Redis treats `"event:12345"` as one single key name.

**TTL: Making Keys Expire**  
Sometimes we want data to automatically disappear after some time:
```redis
# Normal SET (stays forever)
SET "event:12345" "Taylor Swift Concert"

# SET with expiration (disappears after 3600 seconds = 1 hour)
SETEX "event:12345" 3600 "Taylor Swift Concert"

# After 1 hour, this key automatically gets deleted
GET "event:12345"  # Returns nothing
```

Real example:
```redis
# Cache search results for 1 hour
SETEX "search:taylor-swift:2024" 3600 "[event1, event2, event3]"

# After 1 hour, Redis automatically does: DEL "search:taylor-swift:2024"
```

Cache Invalidation: Manual Deletion  

Instead of waiting for TTL, we can manually delete keys:
```redis
# Store some data
SET "search:taylor-swift:2024" "[event1, event2, event3]"
SET "event:12345" "Taylor Swift MSG Concert"

# When Taylor Swift cancels a show, immediately remove related cache:
DEL "search:taylor-swift:2024"
DEL "event:12345"
```

**Redis Sets: Beyond Key-Value**  
Redis also supports **Sets** - collections of unique items:
```redis
# Create a set and add items to it
SADD "my-set" "item1" "item2" "item3"

# See what's in the set
SMEMBERS "my-set"
# Returns: ["item1", "item2", "item3"]

# Remove the entire set
DEL "my-set"
```

**Cache Tags: Using Sets to Group Keys**  
Here's the clever part - we can use Sets to keep track of related cache keys:
```redis
# First, store our actual cache data (normal key-value pairs)
SET "search:taylor-swift:2024" "[event1, event2, event3]"
SET "event:12345" "Taylor Swift Concert Details"
SET "event:12346" "Another Taylor Swift Concert"

# Then, create a "tag" that lists all Taylor Swift related keys
SADD "tag:performer:taylor-swift" "search:taylor-swift:2024" "event:12345" "event:12346"
```

Now we have:
- **Regular cache data:** `"search:taylor-swift:2024"` → `"[event1, event2, event3]"`
- **Tag tracking which keys are Taylor Swift related:** `"tag:performer:taylor-swift"` → `{"search:taylor-swift:2024", "event:12345", "event:12346"}`

**Putting It All Together: Bulk Invalidation**  
When Taylor Swift data changes, we can find and delete ALL related cache at once:
```redis
# 1. Find all keys tagged as Taylor Swift related
SMEMBERS "tag:performer:taylor-swift"
# Returns: ["search:taylor-swift:2024", "event:12345", "event:12346"]

# 2. Delete all those cache keys
DEL "search:taylor-swift:2024" "event:12345" "event:12346"

# 3. Delete the tag itself (cleanup)
DEL "tag:performer:taylor-swift"
```

**Mental Model Summary**
- **Main storage:** Key-value pairs for our actual cached data
- **Organization:** Use colons in key names for organization (like file naming)
- **Auto-cleanup:** TTL makes keys automatically expire
- **Manual cleanup:** DEL removes keys immediately
- **Grouping:** Sets act like "folders" that list which keys belong together
- **Bulk operations:** Use the "folders" to find and delete related keys all at once

The combination lets us say: "Delete everything related to Taylor Swift" without having to remember every single cache key - the tag set remembers them for us!

⚙️ Inter-process vs Inter-thread Communication  
**Inter-process communication (IPC)** means different programs/processes talking to each other. Processes are separate programs running independently with their own memory space.  

**Inter-thread communication** means threads within the same program sharing data and coordinating. Threads are lightweight execution units within a single process that share memory.

Real-world examples:  
Inter-process:  
- Web browser (one process) requests data from a database server (another process) over the network
- A video game (one process) communicates with Steam client (another process) for achievements and friends list
- Microservices architecture - payment service talks to inventory service via REST APIs

Inter-thread:  
- A web server handling multiple user requests simultaneously - each request runs on a different thread, but they might share a connection pool
- Video editing software - one thread handles the UI, another processes video effects, another handles audio, all sharing the same video file data
- Database systems - multiple threads handle different queries but coordinate access to shared data structures

Key difference:  
Inter-process is like different companies exchanging information through formal channels. Inter-thread is like different departments within the same company sharing resources and coordinating work.

⚙️ Exactly-once processing with a queue + Redis  
**At-Least-Once Delivery** from the Queue Guarantees we may see duplicates → consumer must handle retries.

Unique Message ID (Idempotency Key)  
Every message gets a UUID or hash; stored in Redis so duplicates are detected before reprocessing.

Atomic Check-and-Set in Redis  
Use `SETNX` or a Lua script to claim "first processor" and save result/state with a TTL.

Distributed Lock (Redlock)  
Acquire the same lock key across N Redis masters (majority wins) to prevent race-conditions under high concurrency.

Process → Record → Acknowledge  
- Fetch message (visibility timeout starts)
- Acquire lock + check idempotency key
- If new, do work and write back result in Redis
- Release lock and ACK the queue
- If duplicate or lock fails, skip or retry later.

TTL Cleanup  
Idempotency keys expire after our safe-replay window to bound Redis storage.

Architecture Flow
```
[Producer] -> [Queue] -> Consumer -> [Redis: SETNX id:<msgID> + Redlock] - Process & store result - Release lock + ACK
```

Marry at-least-once delivery with Redis-backed idempotency keys and distributed locks to achieve "effectively exactly-once" message processing.

⚙️ Exactly once message delivery  
Idempotent consumers
- Relational DB (PostgreSQL/MySQL PROCESSED_MESSAGES)
    - Durable ACID store with a UNIQUE constraint on `MessageID`; simply `INSERT` and, on duplicate-key violation, drop the message. Guarantees consistency but has higher latency and storage overhead.
- Redis
    - In-memory key–value store using atomic `SET <key> <value> NX EX <ttl>` (or Lua scripts) to "insert if not exists," plus optional persistence (AOF/RDB) and clustering for scale; extremely low latency but requires TTL management to bound memory and may evict old entries.

Message deduplication  
- Use a FIFO queue with built-in deduplication (e.g. AWS SQS FIFO + MessageDeduplicationId or content-based dedupe). Retries with the same ID are dropped within the dedupe window.

Atomic coordination
- Employ a transactional messaging system (e.g. Apache Kafka's transactional producer API: `initTransactions()`, `beginTransaction()`, `commitTransaction()`) so writes and offset commits occur as one atomic operation.

⚙️ Elastic search multi-layer caching   
Key Concepts
- **Document:** One record (e.g. a restaurant with fields like name, location, price, rating).
- **Index:** The full set of our documents (like a database table).
- **Shard:** A slice of that index. A shard holds only a subset of documents.
- **Lucene:** The search engine library under the hood, which builds inverted indexes, bitsets, and scores.

We'll use Shard 0 with 8 documents (IDs 1–8) as our running example:

| ID | Name | Location | Price | Rating |
|----|------|----------|-------|--------|
| 1 | Mario's Pizza | NYC | 15 | 4.5 |
| 2 | Luigi's Pizzeria | NYC | 22 | 4.2 |
| 3 | Bella's Burgers | LA | 12 | 4.0 |
| 4 | Tony's Tacos | NYC | 9 | 4.8 |
| 5 | Panda Express | LA | 11 | 3.9 |
| 6 | Sushi World | NYC | 30 | 4.6 |
| 7 | Curry House | NYC | 18 | 4.3 |
| 8 | Burger King | LA | 8 | 3.5 |

Cache Layers Overview  
| Layer | What It Stores | Where | When It's Hit |
|-------|----------------|-------|---------------|
| 1. Request Cache | Entire search response (hits + aggs) | Coordinating node's memory | If we `request_cache: true` and run the same query |
| 2. Query (Filter) Cache | Boolean filter bitsets | Each shard's JVM heap | On any filter clause we mark cacheable |
| 3. Scoring | — | Each shard (computed afresh) | On the filtered docs, every time |
| 4. Doc Values / Field Data | Field values for sorting/aggs (e.g. price, rating) | On-disk via OS file cache | When we sort or aggregate on a field |
| 5. OS File System Cache | Lucene's on-disk index files (postings, doc values) | Operating system RAM cache | Always: speeds up disk reads |

Step-by-Step Flow with Examples  
A. User Sends Search Request

```json
GET /restaurants/_search
{
  "query": {
    "bool": {
      "must": { "match": { "cuisine": "Italian" } },
      "filter": [
        { "term": { "location": "NYC" } },
        { "range": { "price": { "lt": 20 } } }
      ]
    }
  },
  "request_cache": true
}
```

B. Request Cache Check (Coordination Node)  
**Hit?** Has this exact JSON run before with caching enabled?
    - **Yes** → return stored response in ~1ms (skip shards).
    - **No** → forward to all shards.

*Example: First run ⇒ miss. Second run ⇒ hit, returns same JSON with top hits and aggs.*

C. Shard-Level Execution (Shard 0)  
Boolean Filters → Query (Filter) Cache  
- **Filter:** `location:NYC`
    - **Cache miss on first run** → scan docs1–8 → build bitset (1=match, 0=no):
  
    ```csharp
    [1,1,0,1,0,1,1,0]  // IDs 1,2,4,6,7 are in NYC
    ```
  - **Store this bitset under key** `term:location=NYC`.
- **Filter:** `price<20`
    - **Cache miss** → scan docs → bitset:
  
    ```csharp
    [1,0,1,1,1,0,1,1]  // IDs 1,3,4,5,7,8 are under $20
    ```
    - **Store under key** `range:price<20`.

- **Combine with bitwise AND:**
    ```csharp
    [1,1,0,1,0,1,1,0]
    AND
    [1,0,1,1,1,0,1,1]
    =
    [1,0,0,1,0,0,1,0]
    → Matching Doc IDs [1,4,7]
    ```

**Subsequent run (e.g. "Chinese in NYC under 20"):**
Both bitsets are **cache hits**, so ES skips scanning and ANDs instantly.

Scoring Clause → Fresh Calculation
- **On the filtered docs [1,4,7], run full-text scoring (e.g. BM25):**
| ID | Name | Score |
|----|------|-------|
| 1 | Mario's Pizza | 12.4 |
| 7 | Curry House | 11.9 |
| 4 | Tony's Tacos | 10.1 |

- **Never cached**, ensuring up-to-date relevance.

Sorting / Aggregations → Doc Values & OS Cache
- **Doc Values** store each field column (price, rating) on disk.
- **OS file cache** keeps recently accessed columns in RAM.

*Example: We request* `sort: rating desc` *and an aggregation* `avg(price)`.
- **"ES reads rating values [4.5, 4.8, 4.3] from doc values via OS cache."**
- **"Calculates average price:** `(15 + 9 + 18) / 3 = 14.0`**"

D. Merge Shard Results (Coordination Node)
- **Collect each shard's top-N scored docs + agg buckets.**
- **Merge, re-rank by score, apply pagination.**

*Example final top-3 (global) might still be docs 1,7,4 if other shards have lower scores.*

E. Store in Request Cache
- **Because** `"request_cache": true`, save the entire merged JSON under a key derived from the request body.
- **Next identical request bypasses everything except this quick lookup.**

F. Return Final Response
```json
{
  "hits": {
    "total": 3,
    "hits": [
      { "_id": 1, "_score": 12.4, "_source": { … } },
      { "_id": 7, "_score": 11.9, "_source": { … } },
      { "_id": 4, "_score": 10.1, "_source": { … } }
    ]
  },
  "aggregations": {
    "avg_price": { "value": 14.0 }
  }
}
```

Memory Layout Example (16 GB Server)
```pgsql
Total RAM: 16 GB
├─ JVM Heap (8 GB)
│  ├─ Query (Filter) Cache        0.8 GB  (10%)
│  ├─ Field Data / Doc Values     3.2 GB  (40%)
│  ├─ Request Cache               0.08 GB (1%)
│  └─ Other ES Operations         3.92 GB (49%)
└─ OS & File Cache (8 GB)
   ├─ File System Cache (Lucene)  7.2 GB  (90%)
   └─ Other OS Processes          0.8 GB  (10%)
```

Quick Recap of Hit Order
- **Request Cache** → full-response replay
- **Query (Filter) Cache** → boolean bitsets (AND to filter)
- **Scoring** → fresh relevance scores
- **Doc Values + OS Cache** → fast field lookups for sorting/aggs
- **Request Cache Store** → save full JSON

⚙️ Dual Write problem  
- **Definition:** Writing the same data to two different systems/databases simultaneously, risking inconsistency if one write fails.
- **Core Issue:** No atomic transaction across systems - partial success leaves data out of sync.
- **Example:** E-commerce order:
    - Write to Orders DB: ✅ Success
    - Write to Inventory DB: ❌ Fails
    - Result: Order exists but inventory not decremented
- **Solutions:**
    - Saga pattern (compensating transactions)
    - Event sourcing
    - Two-phase commit
    - Single source of truth with eventual consistency

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

⚙️ Race condition in DB  
A race condition occurs when the outcome of operations depends on the unpredictable timing of events. In databases, this happens when multiple transactions access shared data simultaneously, and the final result depends on which transaction "wins the race."

Classic Race Condition Example  
**Scenario:** Two people trying to buy the last item in stock
```sql
-- Both transactions run simultaneously
-- Transaction A (Person 1)
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;  -- Returns 1 (last item)
-- Context switch happens here - Transaction B runs
UPDATE products SET stock = stock - 1 WHERE id = 1;  -- Sets stock to 0
COMMIT;

-- Transaction B (Person 2)
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;  -- Also returns 1 (same item!)
UPDATE products SET stock = stock - 1 WHERE id = 1;  -- Sets stock to -1
COMMIT;
```

**Problem:** Stock becomes -1, meaning we sold 2 items when only 1 existed!  
Types of Race Conditions  

Lost Update Problem  
**What happens:** One transaction's changes get overwritten

```sql
-- Initial balance: $1000
-- Transaction A: Deposit $100
-- Transaction B: Withdraw $50
-- Both read $1000, calculate separately, last write wins
-- Result: Either $1100 or $950 (other update is lost)
```

Dirty Read Problem  
**What happens:** Reading uncommitted data that might be rolled back

```sql
-- Transaction A
UPDATE accounts SET balance = 2000 WHERE id = 1;
-- Transaction B reads balance as 2000
-- Transaction A gets ROLLBACK due to error
-- Transaction B made decisions based on invalid data
```

Non-Repeatable Read Problem  
**What happens:** Same query returns different results within one transaction

```sql
-- Transaction A
SELECT balance FROM accounts WHERE id = 1;  -- Returns $1000
-- Transaction B commits a transfer
SELECT balance FROM accounts WHERE id = 1;  -- Returns $500
-- Same transaction, different results!
```

Solutions to Race Conditions

Use Proper Isolation Levels
```sql
-- REPEATABLE READ prevents most race conditions
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT stock FROM products WHERE id = 1;
-- Stock value remains consistent throughout transaction
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

Pessimistic Locking
```sql
-- Lock the row immediately when reading
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;
-- No other transaction can modify this row now
IF stock > 0 THEN
    UPDATE products SET stock = stock - 1 WHERE id = 1;
END IF;
COMMIT;
```

Optimistic Locking
```sql
-- Use version numbers or timestamps
START TRANSACTION;
SELECT stock, version FROM products WHERE id = 1;
-- Application logic checks stock > 0
UPDATE products 
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = @original_version;
-- If affected rows = 0, someone else modified it
COMMIT;
```

Atomic Operations
```sql
-- Single statement that checks and updates
UPDATE products 
SET stock = stock - 1
WHERE id = 1 AND stock > 0;
-- Returns affected rows count
-- If 0 rows affected, no stock available
```

Real-World Race Condition Scenarios  
E-commerce Inventory
```sql
-- WRONG: Check then act
SELECT stock FROM products WHERE id = 1;
-- Another user might buy here
UPDATE products SET stock = stock - 1 WHERE id = 1;

-- RIGHT: Atomic check and update
UPDATE products SET stock = stock - 1
WHERE id = 1 AND stock > 0;
```

Banking Transfers
```sql
-- WRONG: Separate operations
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- RIGHT: Single transaction
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1 WHERE balance >= 100;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Counter/Sequence Generation
```sql
-- WRONG: Read then increment
SELECT counter FROM sequences WHERE name = 'order_id';
UPDATE sequences SET counter = counter + 1 WHERE name = 'order_id';

-- RIGHT: Atomic increment
UPDATE sequences SET counter = counter + 1 WHERE name = 'order_id';
SELECT counter FROM sequences WHERE name = 'order_id';
```

Prevention Best Practices  
- Application Level
    - **Keep transactions short** - less time for race conditions to occur
    - **Use database constraints** - let the database enforce rules
    - **Handle retry logic** - gracefully handle failed operations due to conflicts
    - **Validate assumptions** - check if conditions still hold before acting
- Database Level
    - **Choose appropriate isolation levels** - balance consistency vs performance
    - **Use SELECT FOR UPDATE** when we plan to modify data we just read
    - **Implement proper error handling** - handle deadlocks and constraint violations
    - **Design schema with constraints** - prevent invalid states at database level

- Key Points
    - **Race conditions happen when timing matters** - multiple transactions accessing shared data
    - **ACID properties help prevent race conditions** - especially Isolation and Consistency
    - **"Check-then-act" patterns are dangerous** - state can change between check and action
    - **Atomic operations are safer** - combine check and action in single statement
    - **Higher isolation levels reduce race conditions** but hurt performance
    - **Application code must handle retries** - database might reject conflicting transactions
    - **Prevention is better than detection** - design to avoid race conditions

**Key Principle:** In concurrent systems, never assume data remains unchanged between separate operations. Always use atomic operations or proper locking mechanisms.

⚙️ One DB per service vs one DB for many service  
- **One database per service** when:
    - Services have distinct, isolated data needs.
    - We require independent scalability, technology choices (e.g., NoSQL for one, relational for another), and deployment for each service's data.
    - Data ownership is clear and strictly encapsulated within a single service.
    - This aligns with a **microservices architecture**.
    - **Industry Example:** An e-commerce platform where the `Order Service` has its own database (e.g., PostgreSQL) for order details, and the `Product Catalog Service` has its own database (e.g., MongoDB) for product information. Each service can evolve and scale its data store independently.

 - **One database for many services** when:
    - Services share a highly cohesive and interdependent data model.
    - Data consistency across services is paramount and difficult to achieve with distributed transactions.
    - Simplicity of initial setup and management is a higher priority than extreme independent scalability.
    - This is common in a **monolithic or tightly coupled service-oriented architecture**.
    - **Industry Example:** A traditional enterprise resource planning (ERP) system where `Inventory`, `Purchasing`, and `Sales` modules all share a single relational database. Changes in one module directly impact the others through shared tables and relationships within that central database.

⚙️ MySQL Isolation Levels  
Isolation levels control what a transaction can see when other transactions are running simultaneously. Think of it as setting privacy settings for the data a transaction can access from other, unfinished transactions.

### The Four Isolation Levels (Weakest to Strongest)

#### 1. READ UNCOMMITTED
**What it allows:** See uncommitted changes from other transactions.
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- Transaction 1 sees data from Transaction 2 even before Transaction 2 commits.
```
**Problem:** Dirty reads - we might get data that gets rolled back.
**Use case:** Rarely used, only for approximate counts where accuracy isn't critical.

#### 2. READ COMMITTED
**What it allows:** Only see committed changes from other transactions.
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Transaction 1 only sees data after Transaction 2 commits.
```
**Problem:** Non-repeatable reads - same query can return different results within one transaction.
**Example:** We read a balance twice in our transaction, but another transaction commits a change between our reads.

#### 3. REPEATABLE READ (MySQL Default)
**What it allows:** Same query always returns same results within a transaction.
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- If we read data at start of transaction, we'll see same data throughout.
```

**Problem:** Phantom reads - new rows might appear in range queries.
**Example:** `COUNT(*)` might change if another transaction inserts new rows.

#### 4. SERIALIZABLE (Strongest)
**What it allows:** Complete isolation - transactions run as if they're sequential.
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- No concurrency issues, but slowest performance.
```
**Problem:** Lowest performance due to heavy locking.
**Use case:** Critical operations where data consistency is more important than speed.

#### Real-World Example

**Scenario:** Two people checking and updating the same bank account.
```sql
-- Person A
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- shows $1000
-- Person B updates balance to $500 and commits
SELECT balance FROM accounts WHERE id = 1; -- what do we see?
COMMIT;
```

- Results by Isolation Level
    - **READ UNCOMMITTED:** Might see `$500` even before Person B commits.
    - **READ COMMITTED:** First read shows `$1000`, second shows `$500`.
    - **REPEATABLE READ:** Both reads show `$1000` (snapshot from start of transaction).
    - **SERIALIZABLE:** Person A waits until Person B finishes.

- Key Points
    - **SERIALIZABLE** is the safest but has low performance.
    - **MySQL default (REPEATABLE READ)** prevents most common problems.
    - **READ COMMITTED** is popular in high-concurrency applications.
    - **READ UNCOMMITTED** is rarely used due to data integrity risks.
    - Most applications stick with the default unless they have specific requirements.
    - The key tradeoff is always **consistency vs concurrency** - stronger isolation gives you cleaner data but allows fewer simultaneous operations.

⚙️ MVCC?  
MVCC creates snapshots for reads, but writes can still conflict
- Reads see consistent data from transaction start
- Writes need current data to avoid overwriting newer changes
- Problem: Read-then-write operations using stale snapshot data

**Lost Update Scenario**

Initial: Stock = 100

TxA: Reads 100, plans Stock = 90 (100-10)
TxB: Updates Stock = 95 (100-5), commits
TxA: Tries UPDATE Stock = 90 $\leftarrow$ OVERWRITES TxB's change!

**Solutions by Isolation Level**

READ COMMITTED: TxA re-reads current value (95), calculates 95-10=85 ✓
REPEATABLE READ: TxA blocks or gets current row for UPDATE ✓
SERIALIZABLE: TxA fails/retries due to phantom read detection ✓
Optimistic Locking: Version column prevents stale updates ✓

**Key Insight**

MVCC solves read consistency but **isolation levels + locking prevent lost updates** by ensuring writes use current data, not snapshot data.  

⚙️ Database locks  
Locks are mechanisms that prevent conflicts when multiple transactions try to access the same data simultaneously. Think of them like reservation systems - they ensure only authorized transactions can modify data at specific times.

Types of Locks  
- Shared Lock (S Lock) - Read Lock
    - **Purpose:** Multiple transactions can read, but none can write.
    ```sql
    SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;
    -- Multiple transactions can read this record
    -- but no one can modify it
    ```

    - **Analogy:** Like a library book: many people can read it, but no one can edit it.

- Exclusive Lock (X Lock) - Write Lock
    - **Purpose:** Only one transaction can read or write.
    ```sql
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
    -- Only this transaction can access this row
    -- All other transactions must wait
    ```

    - **Analogy:** Like editing a document: only one person can make changes at a time.

Lock Granularity (Scope)  
- Row-Level Locks
    - **What:** Locks individual rows.
    ```sql
    -- 
    UPDATE accounts SET balance = 500 WHERE id = 1;
    -- Only row with id=1 is locked
    ```

    - Advantages:
        - High concurrency - other transactions can work on different rows
        - Faster than table-level locks

- Table-Level Locks
    - **What:** Locks entire table.
    ```sql
    LOCK TABLES accounts WRITE;
    -- Entire table is locked
    -- No other transaction can access any row
    UNLOCK TABLES;
    ```
    - Advantages:
        - Simple, low overhead
    - Disadvantages:
        - Poor concurrency (Used by: MyISAM)

How Locks Work Automatically  
During SELECT (Read Operations)
```sql
-- Normal SELECT (no explicit lock)
SELECT balance FROM accounts WHERE id = 1;
-- InnoDB uses MVCC for consistent reads
```

During UPDATE/DELETE/INSERT (Write Operations)
```sql
-- UPDATE automatically gets exclusive lock
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Row is locked until transaction commits
-- Other transactions wait
```

Deadlock Example
**The Problem:** Two transactions waiting for each other.
```sql
-- Transaction A:
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
UPDATE accounts SET balance = balance + 50 WHERE id = 2;

-- Transaction B:
UPDATE accounts SET balance = balance - 30 WHERE id = 2;
UPDATE accounts SET balance = balance + 30 WHERE id = 1;
```

**MVCC's Solution:** Automatically detects deadlocks and rolls back transactions.

Lock Modes in Practice  
Optimistic Locking (Application Level)
```sql
-- Read with version
SELECT balance, version FROM accounts WHERE id = 1;

-- Update with version check
UPDATE accounts SET balance = 450, version = version + 1 
WHERE id = 1 AND version = 5;
```

Pessimistic Locking (Database Level)
```sql
-- Lock immediately
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Process business logic
UPDATE accounts SET balance = 450 WHERE id = 1;
```

Key Points  
- **Shared locks allow multiple** readers but NO writers (read-only LOCK statements)
- **Locks on biggest granularity:** row locks have fewer conflicts than table locks
- **Atomic transactions:** Every lock conflict causes deadlock
- **Deadlock resolution:** Database picks a victim to rollback and retry automatically
- **FOR UPDATE clause:** used when we need to read data we plan to modify
- **MVCC handles deadlock detection and resolution automatically**

Performance Tips
- Keep transactions short to minimize lock duration
- Access resources in consistent order across transactions
- Use appropriate isolation levels - don't use SERIALIZABLE unless necessary
- Consider optimistic locking for web applications with infrequent conflicts

The key principle: **Locks ensure data consistency but can hurt performance if held too long or acquired in conflicting patterns.**

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

Python Example
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

⚙️ Health check pattern  
- Dedicated endpoints (/health) that external services can periodically query.
- Crucial for micro services and containerized environments where services can fail or become degraded independently.
- Load balancers, API gateways, services meshes, orchestration platforms (Kubernetes, ECS), and dedicated monitoring tool (Prometheus, Nagios) are primary consumer of health checks.
- State checks
    - Are you alive?
    - Are you ready to serve traffic? A service might be alive but not ready to serve traffic e.g. still initializing, loading data, warming up caches, connecting to database)

⚙️ Event sourcing pattern    
Imagine we're only allowed to write down things that happen, never erase **Event Sourcing** means storing the history of all changes (events) to our data, rather than just its current state. To find out the current state, we simply "replay" this history.
- **Events are facts:** "Account Created," "Deposited $100," "Item Added to Cart."
- **State is derived:** The current account balance or cart content is found by applying all its past events.

Benefits:
- **Full Audit Trail:** Complete history of changes for debugging and auditing.
- **Temporal Queries:** Reconstruct state at any past point in time.
- **Decoupling:** Events can trigger actions in other system parts.

Drawbacks:
- **Learning Curve:** Different programming model.
- **Querying Complexity:** Directly querying events for current state can be slow; often needs CQRS (separate read models).

Aggregates  
An **Aggregate** is a cluster of domain objects (entities and value objects) that can be treated as a single unit, ensuring consistency for operations performed on it.  
**Example:** A `ShoppingCart` (the aggregate root) along with its `LineItem` objects forms an aggregate, and we always save or load the entire `ShoppingCart` with all its items to ensure its rules (like total price) are consistent.

Brief Industry Example: Banking Transaction  
A bank account's transaction history is a natural fit for Event Sourcing.
- **Events:** `AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn`, `FeeApplied`.
- **State:** The current balance is derived by replaying these transaction events.
- **Benefit:** Perfect auditability and ability to reconstruct balance at any point in time (e.g., for statements).

---

## Language specific
⚙️ Python threading and multi-processing  
**GIL (Global Interpreter Lock)** - Python's fundamental constraint. Only one thread can execute Python bytecode at a time. Think of it as a single bathroom key in an office building.

- Single Threading
    - Default Python execution
    - One task at a time, sequential processing
    - Simple, predictable, no concurrency issues

- Multi-threading (`threading` module)
    - Multiple threads, but GIL allows only one to run Python code simultaneously
    - **Good for:** I/O-bound tasks (file reading, network requests, database calls)
    - **Bad for:** CPU-intensive tasks (calculations, data processing)
    - Threads can release GIL during I/O operations, allowing others to work
    - Why threading helps with I/O:** When Thread A waits for disk/network, it releases GIL, Thread B can work.

- Multi-processing (`multiprocessing` module)
    - Separate Python processes, each with its own GIL
    - **Good for:** CPU-intensive tasks
    - **Cost:** Higher memory usage, slower inter-process communication
    - True parallelism for CPU-bound work

- Use Single Threading when:
    - Simple scripts
    - Sequential processing is sufficient
    - Avoiding complexity

- Key Points
    - **GIL limitation:** Python threading doesn't help CPU-bound tasks
    - **I/O releases GIL:** Why threading works for network/file operations
    - **Memory trade-off:** Multiprocessing uses more memory but gets true parallelism
    - **Language comparison:** Java and Go have better native concurrency models
    - **asyncio alternative:** Modern Python also offers async/await for I/O-bound tasks

The mental model: Python threading is like having multiple workers but only one work station - they take turns. Multiprocessing gives each worker their own station but costs more space.

⚙️ Python daemon threads    
Core Concept: Daemon threads in Python are background threads that automatically terminate when the main program exits, unlike regular threads which keep the program running until they complete.

**Key Points**
- **Definition** - Daemon threads are threads marked with `daemon=True` that run in the background and don't prevent program termination
- **Lifecycle behavior** - When the main thread finishes, Python immediately kills all daemon threads without waiting for them to complete their work
- **Setting daemon status** - Use `thread.daemon = True` before calling `thread.start()`, or pass `daemon=True` to the Thread constructor
- **Default inheritance** - Threads inherit daemon status from their parent thread (main thread is non-daemon by default)
- **Use cases** - Background tasks like logging, monitoring, periodic cleanup, or services that should stop when the main application stops
- **Limitations** - Daemon threads cannot perform critical cleanup operations since they may be terminated abruptly mid-execution
- **Program termination** - Python waits for all non-daemon threads to finish before exiting, but kills daemon threads immediately
- **Thread safety** - Daemon status doesn't affect thread synchronization requirements - still need locks, queues, etc. for shared resources
- **Checking status** - Use `thread.daemon` property to check if a thread is marked as daemon
- **Cannot change after start** - Daemon status must be set before calling `thread.start()`, attempting to change it afterward raises RuntimeError

⚙️ Normal, class and static methods in Python  
- **Normal (Instance) Methods**
    - Bound to specific object instances
    - First parameter is **`self`** (the instance)
    - Access/modify instance data
    - Most common method type

    ```python
    class BankAccount:
        def __init__(self, balance):
            self.balance = balance

        def withdraw(self, amount): # Normal Method
            self.balance -= amount
    ```

- **Class Methods**
    - Bound to the class, not the instance
    - First parameter is **`cls`** (the class)
    - Used for factory methods that create instances using alternative constructors
    - Decorated with **`@classmethod`**

    ```python
    class Person:
        def __init__(self, name, age):
            self.name = name
            self.age = age

        @classmethod
        def from_birth_year(cls, name, birth_year): # Alternative constructor
            age = 2023 - birth_year
            return cls(name, age) # Creates an Person instance

        # This method is not in the original image but added for context of a static method
        @staticmethod
        def is_adult(age):
            return age > 18
    ```

- **Static Methods**
    - Not bound to class or instance
    - No special first parameter
    - Utility functions related to the class
    - Decorated with **`@staticmethod`**

    ```python
    class MathUtils:
        @staticmethod
        def is_prime(n): # Utility function
            return n > 1 and all(n % i != 0 for i in range(2, int(n**0.5) + 1))
    ```

**Real World Examples**  
Django Model (Normal Methods)
```python
class Post(models.Model):
    #... fields
    email = models.EmailField()

    def send_confirmation_email(self): # Normal Method
        # Accesses instance data (self.email)
        send_mail('Subject', 'Message', from_email, [self.email])
```

datetime (Class Methods)
```python
from datetime import datetime

# fromtimestamp is a class method that acts as a factory
now = datetime.now() # Creates a datetime instance
parsed = datetime.fromtimestamp(1672531200) # Class Method
```

pathlib (Static Methods)
```python
from pathlib import Path

class Path:
    # ...
    @staticmethod
    def cwd(): # Static method -- doesn't need instance
        # Returns the current working directory as a Path object
        return os.getcwd()
```

- **Mental Model**
    - **Normal method:** "Do something WITH this specific object."
    - **Class method:** "Create or access something ABOUT the class itself."
    - **Static method:** "Utility functions that BELONG with this class conceptually."

Use normal methods for 90% of the time. Use class methods for alternative constructors or class-level operations. Use static methods for utilities that logically belong with the class but don't need class/instance data.

⚙️ generator and yield in Python  
A **generator** is a special type of function that returns values one at a time using the `yield` keyword instead of `return`. It pauses execution and resumes where it left off.

Key differences:
- Regular function uses `return` (exits completely)
- Generator uses `yield` (pauses and can continue)

Basic example:
```python
def simple_generator():
    yield 1
    yield 2
    yield 3

# Using the generator
gen = simple_generator()
print(next(gen))  # 1
print(next(gen))  # 2
print(next(gen))  # 3
```

Why use generators?  
- **Memory efficient:** Generate values on-demand instead of storing all in memory.
- **Lazy evaluation:** Only compute when needed.

```python
# Memory-efficient way to generate large sequences
def big_numbers():
    for i in range(1000000):
        yield i * i

# Only generates values as we iterate
for square in big_numbers():
    if square > 100:
        break
    print(square)
```

Generators are perfect for processing large datasets or infinite sequences without consuming excessive memory.

Practical example:
```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

# Usage
for num in countdown(5):
    print(num)  # Prints: 5, 4, 3, 2, 1
```