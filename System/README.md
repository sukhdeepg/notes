## WEB
⚙️ Working of PUT `/users/[id]/followers`
![image](https://github.com/user-attachments/assets/da6d3c4a-3464-47cc-8a49-7ace2f548441)

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

⚙️ DNS record types  
| Record Type | Core Idea | Example | Notes |
|-------------|-----------|---------|-------|
| A | Maps a domain name to an IPv4 address. | google.com → 142.250.190.46 | • For IPv4 only. If a site also supports IPv6, it will have an accompanying AAAA record.<br>• When paired with AAAA, browsers choose IPv6 if supported, then IPv4 as fallback. |
| AAAA | Maps a domain name to an IPv6 address. | ipv6.google.com → 2001:4860:4860::8888 | • For IPv6 only. If a site also supports IPv4, it will have an accompanying A record.<br>• When paired with A, IPv6-capable clients use AAAA, falling back to A if necessary. |
| CNAME | Creates an alias (nickname) from one hostname to another. | www.example.com → example.com *(points to the same IP as example.com)* | • Useful when multiple subdomains (e.g., www., blog., shop.) should all point to the same target—avoids duplicating A/AAAA records.<br>• Cannot create any other record type (A/AAAA, MX, etc.) for the exact same name as a CNAME.<br>• CNAME must point to a hostname, not directly to an IP address. |
| MX | Lists which mail servers accept email for the domain (with priority). | example.com MX → mail.example.com (pref: 10) *(when we send mail to user@example.com, our client looks up this record)* | • Lower "preference" numbers indicate higher priority. If multiple MX records exist, mail delivery tries the lowest-numbered server first.<br>• If the first (lowest-numbered) mail server is unreachable, the next-lowest is used automatically.<br>• Each MX record must refer to a valid A or AAAA record for the mail server hostname. |
| TXT | Holds arbitrary text for verification or policy information. | • SPF: v=spf1 include:_spf.google.com ~all *(prevents email spoofing)*<br>• Domain verification (e.g., Google/Microsoft asks us to add a specific TXT to prove ownership) | • Beyond SPF verification, TXT can store DKIM public keys, DMARC policies, or any custom text-based data.<br>• Examples include DKIM (_default._domainkey.example.com), DMARC (_dmarc.example.com), and other service verification strings.<br>• No special format enforced—any UTF-8 text is allowed. |
| NS | Specifies which name servers are authoritative for the domain. | example.com NS → ns1.nameserverprovider.com, ns2.nameserverprovider.com *(tells resolvers where to find the real DNS data)* | • At our registrar, we set NS records to delegate DNS hosting to providers (e.g., Route 53, Cloudflare).<br>• The authoritative NS set is published by the parent zone (e.g., the .com registry), so ensure our registrar's NS entries match what the parent zone has.<br>• All other DNS records (A, MX, TXT, etc.) are then managed under these authoritative servers. |
| SRV (Service record) | Defines the hostname and port for a specific service. | _sip._tcp.example.com → sip.example.com:5060 *(VoIP clients use this to locate the SIP server)* | • Commonly used for protocols like SIP (_sip._tcp), XMPP (_xmpp-server._tcp), LDAP (_ldap._tcp), etc.<br>• Specifies the target host and port number for the service.<br>• Clients query _service._proto.name (e.g., _sip._tcp.example.com) to discover and connect without hardcoding ports. |
| PTR (Pointer record) | Performs reverse lookup: maps an IP address back to a hostname. | 142.250.190.46 → google.com *(used by mail servers and logging systems to verify that an IP matches a domain)* | • Reverse DNS (PTR) records are managed by whoever controls the IP block (often our hosting provider or ISP).<br>• To set up or change a PTR record, we must request it from the IP block owner.<br>• Crucial for mail servers—many mail systems reject incoming messages if the sending IP's PTR doesn't match its HELO/EHLO hostname. |

⚙️ [How to properly use CloudFront to Cache an API with Cache-Control and HTTP 304 to provide cache revalidation functionality.](https://readmedium.com/how-to-properly-use-cloudfront-to-cache-an-api-with-cache-control-and-http-304-to-provide-cache-b893f6822475)

⚙️ Different types of latencies  
- **Processing (service) latency**
  - Time spent doing the work.
  - *Example:* A photo-filter API takes 15 ms of CPU time to apply effects.

- **Queuing latency**
  - Time a request waits before being processed.
  - *Example:* Under heavy load, our HTTP request sits in the server’s backlog for 50 ms.

- **Propagation delay**
  - Time for a signal to travel through the medium.
  - *Example:* A one-way trip to a geostationary satellite adds ≈240 ms.

- **Transmission (serialization) delay**
  - Time to push all bits onto the wire.
  - *Example:* Sending a 1 MB file over a 10 Mbps link takes ≈0.8 s.

- **Round-trip latency**
  - End-to-end there-and-back time.
  - *Example:* A typical transcontinental ping is \~100 ms RTT.

- **Tail (percentile) latency**
  - The slowest X% of requests—captures worst cases.
  - *Example:* p99 web-page loads might spike to 500 ms even if average is 100 ms.

- **Jitter**
  - Variation in packet delay.
  - *Example:* VoIP calls suffer choppy audio when jitter exceeds 30 ms.

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

⚙️ Docker and Kubernetes  
Docker:
- Containerization platform that packages our app + dependencies into a lightweight, portable container
- Handles building, running, and managing individual containers on a single machine
- Think: "How do I package and run my app consistently anywhere?"

Kubernetes:  
- Container orchestration system that manages clusters of machines running containers
- Handles deployment, scaling, load balancing, service discovery, and self-healing across multiple nodes
- Think: "How do I manage hundreds of containers across dozens of servers in production?"

Key difference:  
Docker gets our app running in a container. Kubernetes gets that container running reliably at scale across infrastructure.

Docker is the engine, Kubernetes is the fleet management system.

⚙️ Change Data Capture  
Real-time technique that captures database changes from transaction logs and streams them to downstream systems without impacting source database performance.

How CDC Works  
When data changes in our primary database (INSERT, UPDATE, DELETE), CDC captures these changes from the database's transaction log and streams them to downstream systems. This happens asynchronously, so our main application isn't affected.

Core Architecture
```
[Application] → [Database] → [CDC Agent] → [Message Broker] → [Consumers]
```

Key Components
- CDC Agent/Service:
    - Standalone process/container separate from application
    - Reads database transaction logs (not tables)
    - Maintains persistent connections to DB and message broker
    - Examples: Debezium connectors, AWS DMS, Maxwell

- Database Transaction Logs:
    - PostgreSQL: WAL + logical replication slots
    - MySQL: Binary logs (binlog)
    - MongoDB: Oplog
    - Pulls data (not pushed by DB)

- Message Broker:
    - Kafka (most common), Kinesis, Event Hubs, Pub/Sub

How It Works
- App writes to database → transaction log updated
- CDC agent reads log → converts to structured events
- Events published to message broker
- Multiple consumers process independently

Deployment Models
- **Self-Managed:** Kafka Connect cluster with Debezium connectors
- **Cloud-Managed:** AWS DMS replication instances, Google Datastream
- **Enterprise:** Confluent Platform, Striim

Real-World Example  
E-commerce: User profile update → PostgreSQL WAL → Debezium → Kafka → Multiple services (Search/Elasticsearch, Analytics, Notifications)

Industry Tools
- **Open Source:** Debezium, Maxwell, Kafka Connect
- **Cloud:** AWS DMS, Google Datastream, Azure Data Factory
- **Enterprise:** Confluent, Striim

Key Benefits
- Real-time data sync across microservices
- No application code changes needed
- Minimal database performance impact
- Event-driven architecture enablement

Message Format
```json
{
  "before": {...}, "after": {...},
  "op": "u", "ts_ms": 1634567890123,
  "source": {"db": "users", "table": "profiles"}
}
```

Database Transaction Log:
- PostgreSQL: Write-Ahead Log (WAL) with logical replication slots
- MySQL: Binary logs (binlog)
- MongoDB: Oplog (operations log)
- SQL Server: Transaction log with Change Tracking

Message Broker:
- Apache Kafka (most common)
- AWS Kinesis
- Azure Event Hubs
- Google Pub/Sub

How the CDC Agent Actually Works:
- Database Connection: CDC agent establishes a dedicated connection to read transaction logs
- Log Parsing: Continuously parses log entries and converts to structured events
- State Management: Maintains checkpoint/offset of last processed log position
- Event Publishing: Publishes change events to message broker topics
- Error Handling: Handles connection failures, restarts from last checkpoint

Real-World Scenario: E-commerce Platform  
Consider an e-commerce platform with microservices architecture:

Infrastructure Setup:
- User Service: Node.js app + PostgreSQL (primary database)
- CDC Layer: Debezium Connect cluster (3 nodes) + Kafka cluster
- Search Service: Updates Elasticsearch index
- Analytics Service: Writes to data warehouse

Flow:
- Customer updates profile → PostgreSQL transaction log updated
- Debezium connector reads WAL → converts to JSON event
- Event published to Kafka topic `user.profile.changes`
- Multiple consumers process the event independently

Technical Implementation Deep Dive

Database-Specific Mechanisms:  

PostgreSQL:
- Creates logical replication slot: `SELECT pg_create_logical_replication_slot('debezium', 'pgoutput')`
- CDC agent connects as replica and consumes WAL stream
- No impact on primary database performance

MySQL:
- Enables binlog: `log-bin=mysql-bin`, `binlog-format=ROW`
- CDC agent connects as replica server using MySQL replication protocol
- Reads binary log events in real-time

Message Format:
```json
{
  "before": {"id": 123, "name": "John", "email": "john@old.com"},
  "after": {"id": 123, "name": "John", "email": "john@new.com"},
  "op": "u",
  "ts_ms": 1634567890123,
  "source": {"db": "users", "table": "profiles"}
}
```

Industry Example - Uber: Uber uses CDC to sync rider/driver data across 50+ microservices. Their CDC infrastructure processes millions of database changes per second, updating real-time location services, pricing engines, and analytics systems without affecting their core ride-booking application performance.

Complete Architecture:
```
[User Service] → [PostgreSQL] → [Debezium Connect Cluster] → [Kafka] → [Multiple Consumers]
       ↓                ↓                    ↓
[App Containers]  [Dedicated Nodes]  [Elasticsearch, Warehouse, etc.]
```

This architecture ensures data consistency across microservices while maintaining system performance and enabling real-time analytics and search capabilities through purpose-built infrastructure components.

⚙️ App to Kafka vs DB to Kafka  
Event Sourcing Approaches: Manual vs CDC

Approach Comparison
| Approach | Pros | Cons |
|----------|------|------|
| **App→Kafka (manual emit)** | • Simple to understand and implement<br>• Event schema under app control | • Risk of "write DB but fail to publish" (or vice versa)<br>• We must build retry/idempotency logic<br>• Harder to guarantee exactly-once ordering between DB commit and event |
| **DB→Kafka (CDC)** | • Guaranteed 1:1 mapping of committed rows to events<br>• Exactly-once (or "read-committed") delivery semantics<br>• Fully decouples our app from event plumbing | • Requires CDC tooling (Debezium, Maxwell, native connector)<br>• Slightly more infrastructure to operate<br>• Can introduce a small replication lag |

Which to choose?
- **For small-to-medium scale or prototypes → App→Kafka**
    - We get up and running quickly, with minimal infra.

- **For mission-critical, high-scale systems → CDC**
    - We gain reliability, true event sourcing, and don't have to re-implement exactly-once guarantees in our service layer.

Often teams start with app-level emits and migrate to CDC once they hit reliability or consistency pain points. 

⚙️ ClickHouse  
ClickHouse is an open-source, column-oriented database designed specifically for high-performance analytical queries (OLAP – Online Analytical Processing).  

**Core Concepts & How ClickHouse Works:**
- **Column-Oriented Storage**
    - Data is stored column-wise instead of row-wise.
    - Efficient for aggregations and analytics because only necessary columns are read from storage.
    - **Example:**
        Given a table:

        | UserID | Age | Country | Purchase |
        | ------ | --- | ------- | -------- |
        | 1      | 25  | USA     | 200      |
        | 2      | 32  | UK      | 500      |

    - Column-wise storage:

        ```
        UserID:   [1, 2]
        Age:      [25, 32]
        Country:  ["USA", "UK"]
        Purchase: [200, 500]
        ```

    - **Advantage:** Queries like `AVG(Purchase)` only read the Purchase column, drastically reducing query time.

- **Compression & Indexing**  
    - Columnar storage allows for better compression because similar data is grouped together.
    - ClickHouse applies various efficient compression methods like LZ4, ZSTD.
        - **LZ4:** ClickHouse’s default codec, optimized for very fast compression and decompression with moderate space savings—use it when raw speed matters.
        - **Zstandard (ZSTD):** Offers higher compression ratios while keeping decode speed close to LZ4—choose it when storage efficiency outweighs absolute speed.
    - Uses sparse indexing for rapid query filtering.
        - **What it is** – ClickHouse sorts each data part by the primary-key columns and records *one pointer (“mark”) for the first row of every granule of ≈ 8 192 rows*; this list stays in RAM because it is thousands-of-times smaller than the table.
        - **How a row is found**
            - **Binary-search the marks** to locate the granule whose first key is just before (or equal to) the key we asked for.
            - **Jump straight to that granule on disk** (≤ 8 k rows) and load it. All other granules are skipped.
            - **Scan inside the granule** to return the exact row(s).

            - **Tiny example** – A table with **1 billion rows** sorted by `event_date` has \~122 000 marks (a few MB). A query
        `WHERE event_date = '2025-05-01' AND user_id = 42`
        touches only the two granules for that date (\~16 k rows) and skips the other 99.998 % of the data.

        - Picture it like sticky-notes every 8 000 pages: flip to the nearest note, read a handful of pages, ignore the rest.
    - **Mental Model:**
        - Think of indexing like bookmarks in a book. Sparse indexing is like placing bookmarks every few pages to quickly jump to the relevant section without having bookmarks on every single page.

- **Distributed & Parallel Processing**
    - ClickHouse can distribute data across multiple nodes for scalability.
    - Queries are executed concurrently on multiple nodes and aggregated for results.

    - **Mental Model:**
        - Imagine distributing workload evenly across multiple workers; each worker handles part of the task simultaneously, speeding up overall execution time.

- **Vectorized Execution Engine**
    - Processes data in batches (vectors), taking advantage of CPU cache and SIMD instructions.
    - Accelerates query performance by optimizing CPU utilization.

- **Real-Time Data Ingestion**
    - Designed to handle high rates of real-time data insertion efficiently.
    - Supports batch inserts and continuous streaming ingestion (via Kafka integration, for instance).

- **Real Industry Example:**
    - **Scenario:** Real-Time Analytics for E-commerce Platform
        - An e-commerce platform needs real-time analytics on user behavior, transactions, and product trends to optimize marketing campaigns.

    - **Data:**
        - User clickstream logs (millions of events per minute).
        - Transaction data (purchases, refunds).
        - User metadata (location, device, preferences).

    - **Requirements:**
        - Real-time reporting and dashboards.
        - Fast aggregation queries like "average purchase per country", "top-selling products", or "daily active users".

- **Solution with ClickHouse:**
    - **Columnar storage** helps quickly analyze large volumes of clickstream data.
    - **Compression** significantly reduces storage cost and I/O overhead.
    - **Distributed nodes** handle massive data loads by scaling horizontally.
    - **Real-time ingestion** via Kafka streams ensures instant data availability for immediate analytics.

    - **Outcome:**
        - Queries that previously took minutes now execute in seconds or milliseconds.
        - The marketing team gains actionable insights in near real-time.

- **Summary Mental Model:**
    - ClickHouse efficiently processes analytical queries on large-scale datasets by storing data in compressed, indexed columns, distributing workloads across clusters, and executing queries with optimized, parallelized computations, ideal for high-speed real-time analytics.

Comparison - ClickHouse vs Cassandra

**In one sentence:** Use **ClickHouse** when we mostly *append data and ask big summary questions* (analytics), and use **Cassandra** when we need *constant, quick updates and look-ups* (operational storage).


| What we care about       | ClickHouse                                                                                                                                                                    | Cassandra                                                                                                                                                                                 |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **How data sits on disk** | Packs each **column** together, so reading one column is very fast — perfect for crunching numbers across billions of rows.                             | Stores complete **rows** in immutable files called **SSTables (Sorted String Tables)**; each row lives “wide” and is easy to fetch by key. |
| **Typical workload**      | **OLAP (Online Analytical Processing)** → many records in, few big reports out.                                                                                | **OLTP (Online Transactional Processing)** → lots of quick inserts, updates, reads.                                                                                        |
| **Writing data**          | Best at *append-only* streams (logs, events). Updates rewrite whole data parts, so keep them rare.                                                     | Handles steady inserts **and** field updates; each row/column can have a **TTL (Time To Live)** to expire old data.                                                   |
| **Reading data**          | Reads columns in batches and uses CPU features that treat many numbers at once (**SIMD – Single Instruction Multiple Data**) for speed.               | Fetches whole rows or short ranges by key; we can tune read/write consistency for each request.                                                              |
| **Compression**           | Uses fast codecs such as **LZ4 (very quick)** and **ZSTD / Zstandard (better compression, still fast)** to shrink each column. | Compresses entire SSTables; no special column tricks.                                                                                                                 |
| **Scaling out**           | Shard for more write speed and replica for high availability; ideal for huge analytic clusters.                                                        | Adds nodes to grow capacity; replicates to multiple **data centres (DCs)** for global uptime.                                                                 |

Mental picture
- **ClickHouse**
    - Columns are lined up together, squished with LZ4/ZSTD, and the CPU chews through them a few thousand values at a time.
    - Great when the question is “How many, how fast, top-N?” across massive logs.

- **Cassandra**
    - Every user or device has a row; we flip to the right page (partition key) and read or update it in milliseconds, no matter the region.

When to pick which (real-life sketches)

| Situation                                                       | Choose ClickHouse                                                     | Choose Cassandra                                                                 |
| --------------------------------------------------------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **Ad-tech dashboard** tracking billions of impressions per hour | Needs lightning-quick counts and averages over fresh event data.      | —                                                                                |
| **Shopping-cart / session store** for a global e-commerce site  | —                                                                     | Needs constant row updates and fast look-ups close to users in multiple regions. |
| **IoT sensor flood** (temperatures every second)                | Append-only time series; roll-ups like “average per minute” run fast. | Could store raw data, but large scans would be slower.                           |

Key take-away
- Ask **big summaries over mostly immutable data? → ClickHouse**
- Need **instant row reads/updates everywhere? → Cassandra**

⚙️ Apache Flink  
Apache Flink is easiest to picture as a **fault-tolerant assembly line**: events roll off an input belt (sources), pass through machines that can filter, group or count while remembering a little state (operators on TaskManagers, orchestrated by a JobManager), get photographed every few seconds so nothing is lost (checkpoints), and finally drop into output bins (sinks).

Visual overview  
```
┌────────────┐   events   ┌────────────┐   tasks   ┌────────────────┐   results   ┌──────────────┐
│  SOURCES   │ ─────────▶ │ JobManager │ ────────▶ │ TaskManagers   │ ───────────▶│    SINKS     │
│ (Kafka …)  │            │  (planner) │           │ (workers)      │             │ (DB, Kafka…) │
└────────────┘            └────────────┘           └────────────────┘             └──────────────┘
        ▲ periodic snapshots                    ▲ exactly-once recovery
        └─────────── checkpoints ───────────────┘
```

- Where data comes from
    - Flink **doesn’t store data**; it plugs into Kafka, Kinesis, files, JDBC tables and dozens more via ready-made connectors that turn each feed into a stream as soon as the job starts.

- What happens inside **Flink engine**
    - Cluster roles
        - **JobManager** (traffic cop) builds the flow graph and coordinates checkpoints.
        - **TaskManager** is just **one Java (JVM) worker-process**; we chop its CPU + memory into **task-slots**, and **each slot runs one sub-task in its own thread**, so the more slots we configure, the more work that single JVM can do in parallel.

        - The **Java Virtual Machine (JVM)** is a small “software computer” that sits on top of our real hardware: it reads platform-neutral *.class* byte-code, turns each instruction into the native ones our CPU understands, and quietly handles chores like memory clean-up, so the *same* compiled file can run unchanged on Windows, Linux, or macOS.

            - **Example**
                1. Write `HelloWorld.java` and compile once to `HelloWorld.class` byte-code.
                2. Copy that single *.class* file to three different machines (Windows, Linux, Mac).
                3. On each machine just type `java HelloWorld`; the local JVM translates and runs the code, and every screen prints **Hello World**—no recompilation needed.

            - **How does the above relate to Flink?**  
                - Each Flink job is compiled into .class byte-code files that are bundled into a JAR.
                - **What’s a JAR, and why does Flink use it?**
                    - **JAR (Java Archive)** = one ZIP-like file that bundles **all** the compiled `.class` files, third-party libraries, resources (SQL, images, config) and a small *manifest* that tells Java which class has the `main()` entry point.
                    - **Why bundle?** A real Flink job often needs hundreds of classes plus jars from Kafka, Jackson, etc. Sending a single JAR keeps every byte in one package, cuts transfer time, preserves directory layout, and lets us sign or version the whole lot at once.
                    - **How Flink uses it:** when we run `flink run myJob.jar`, the JobManager copies that JAR to every TaskManager JVM; each TaskManager’s class-loader opens the archive, loads the contained `.class` files into memory, and starts the operators.
                    - **Why not ship raw `.class` files?** We’d have to send dozens of files and their resource folders separately, recreate the exact paths on every worker, and still tell Java which one to run. A single JAR eliminates that bookkeeping and guarantees every TaskManager sees the identical code and dependencies.

    - **Concrete operator example**
        *Scenario – counting product clicks every minute*

        | Cluster piece                | What happens                                                                                                                                                                                                                                                        |
        | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
        | **TaskManager with 4 slots** | The job’s pipeline is `map` (parse JSON) → `keyBy(productId)` → `tumblingWindow(1 min)` → `sum(1)` (count). Flink deploys **four identical copies** of that operator chain—one per slot—so each slot handles a different slice of the Kafka topic at the same time. |
        | **Parallelism = 4**          | All four slots are busy; every click is parsed, grouped, windowed, and counted exactly once, but spreads across the four threads for speed.                                                                                                                         |
        | **Need parallelism = 8**     | Start a second TaskManager with 4 more slots (or raise slots on the first). Flink automatically places the extra four sub-tasks there, doubling throughput without spinning up extra JVMs per task.                                                                 |

        The tumbling-window operator in the chain closes its 60-second bucket once the watermark moves past the minute mark, emitting stable per-product counts downstream.

    - Data-flow & operators
        - Our code becomes a directed graph of simple operators (`map`, `filter`, `keyBy`, `window`, …) that the JobManager chains into pipelines and ships to workers for continuous execution.

    - State & fault-tolerance (preview)
        - Operators can keep counters, joins or ML features in memory or RocksDB; periodic checkpoints snapshot that state so a crash rewinds to the last photo with **exactly-once** guarantees.

- Where data goes — **Sinks**
    - The processed stream is pushed out through connectors just like the inputs—to Kafka, Elasticsearch, JDBC DBs, S3, Redis and more. Transaction-aware sinks join the checkpoint to extend exactly-once all the way out.
        - **Buffer location** – When a checkpoint starts, each sink-operator thread (running inside a TaskManager JVM) opens an **un-committed transaction**:
            - for Kafka, it’s the producer’s in-memory buffer + a Kafka *transaction* that is still invisible to readers
            - for JDBC/MySQL, it’s the driver’s send-buffer plus the database’s open SQL transaction
            All outgoing records stay in that transaction until the checkpoint verdict arrives.

        - **Commit point** – If the JobManager says “checkpoint succeeded”, the sink **commits** the open transaction (Kafka `commitTransaction()`, SQL `COMMIT`) right away; otherwise it **aborts/ROLLBACKs** and those records are replayed later

        - **“Visible exactly once”** means downstream readers can see the batch **only after the single successful commit**, and because any earlier, crash-time writes were rolled back, the same record will never appear twice nor disappear

        - *One-liner mental model:* the sink keeps new rows in a sealed box during the snapshot; a green light commits the box, a red light dumps it, so the world outside Flink sees each record **once and only once**.

                                    ┌───────────────── Flink Checkpoint ────────────────┐
                TaskManager JVM │  (operator thread)                                 │       External System
                ────────────────┼────────────────────────────────────────────────────┼───────────────────────────
                                │                                                    │
                                │ 1. Begin TX & BUFFER records                       │
                Kafka Example   │  ┌────────────┐            Kafka Producer          │  ┌─────────────┐
                                │  │  TX-101    │══════════▶│  TX Buffer  │═════════▶│    Kafka Topic │
                                │  │(in memory) │           │ (uncommitted) │        │    (visible)   │
                                │  └────────────┘                                    │  └─────────────┘
                                │                ▲                                   │
                                │                │ abort on failure                  │
                                │ 2. COMMIT TX-101 on checkpoint success             │
                ────────────────┼────────────────────────────────────────────────────┼───────────────────────────
                                │                                                    │
                                │ 1. Begin TX & BUFFER rows                          │
                MySQL Example   │  ┌────────────┐            JDBC Driver             │  ┌──────────────┐
                                │  │  TX-42     │══════════▶│  Pending rows │═══════▶│     MySQL Table │
                                │  │(in memory) │           │ (uncommitted) │        │    (visible)    │
                                │  └────────────┘                                    │  └──────────────┘
                                │                ▲                                   │
                                │                │ rollback on failure               │
                                │ 2. COMMIT TX-42 on checkpoint success              │
                                └────────────────────────────────────────────────────┘

- Mini industry example — live product-view counts
    1. Nginx logs flow into **Kafka** (source).
    2. Flink reads, parses JSON, groups by `productId` (`map`, `keyBy`).
    3. A **tumbling 1-minute window** counts views.
    4. Counts are written to **Redis** (sink).
    5. Grafana graphs the numbers.
    The same pattern scales to millions of events per second for Uber’s surge-pricing features.

Time & Reliability essentials

| Piece                                        | What happens                                                                                                                                                                                                                                                                                                                                                                                                                                                            | 3-line example                                                                                                                                                                                                                                                                  |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Checkpoint = safety photo**                | Every *N* seconds the JobManager inserts a **barrier**; when that marker reaches each TaskManager, the task quickly writes its in-memory state **and** the last Kafka/MySQL offsets to durable storage, then keeps working. If a node dies, Flink restarts from that photo, rewinds the sources to the saved offsets, and replays the missing events — so nothing is lost or processed twice. | Counter is **42** at snapshot → crash at **47** → recovery loads 42, replays five events, ends again at **47** (no gaps, no double counts).                                                                                                                                     |
| **Event-time + Watermark = on-time windows** | Each event carries its “happened-at” timestamp. A **watermark** (e.g., “latest-seen minus 2 s”) moves with the stream; when it passes the end of a window, Flink knows “everything earlier is probably in—emit now.” This lets it cope with out-of-order events without waiting forever.                                                                                                    | 1-min tumbling window 12:00-12:00:59. Last event arrives at 12:01:01; watermark = 12:00:59.01 → window closes, count is emitted. A stray click stamped 12:00:45 that arrives later is flagged as “late”; our job can drop it or merge it, but it never opens the window again. |

**Mental picture:** the *watermark* is the belt’s timer telling machines a time bucket is “done”; the *checkpoint* is the safety photo that lets the factory rewind after a fault and still ship each finished bucket **exactly once**.

A Flink **barrier** is just a tiny control record—internally an object like
`CheckpointBarrier(id=42, timestamp=..., options=...)`—that is injected into the stream and travels between normal data records, so in the byte flow we’d see something like

```
…  eventA  eventB  [Barrier#42]  eventC  …
```

where `[Barrier#42]` marks “everything before me is in snapshot 42; everything after goes to the next one.

- Why do events arrive out of order if “Kafka is append-only”?
    - **Order is only per partition.** Kafka guarantees the order **inside** a single partition, not across all of them; when producers use several partitions for throughput, consumers see an interleaving that looks out-of-order globally.
    - **Network and retries add more shuffle.** Retries after a brief broker hiccup or slow network path can make an older event show up after a newer one in the same partition.

- What does Flink do with a late (out-of-order) event?  
    - **If a record arrives after the window closed** (i.e., its time ≤ closed window end) Flink routes it according to the job’s *late-data policy*: drop, side-output, or (with “allowed lateness”) reopen and fix the aggregate.  
    - **Scenario** 1-minute tumbling window counts product clicks ( 12:00:00-12:00:59 ).  
    At 12:01:03 the window closes and emits **42** clicks. A straggler stamped 12:00:45 arrives at 12:01:07—later than the watermark.

        | Late-data policy               | What Flink does                                                                                                              | Micro-example outcome                                                                                        |
        | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
        | **Drop** (default)             | Late record is ignored.                                                                                                      | Final count stays **42**; straggler vanishes.                                                                |
        | **Side-output**                | Late record is routed to a separate “late-events” stream we defined.                                                        | Main result stays **42**; the `(productId, 1)` tuple appears in the side stream for custom handling/logging. |
        | **Allowed lateness + re-emit** | We set `allowedLateness = 30 s`; Flink reopens the closed window, adds the straggler, and re-emits an **updated count 43**. | Downstream sees first result 42, later an update 43 (exactly once for each).                                 |

- Exactly-once after a failure—how the snapshot prevents double counts

    1. **Barrier sweep** – A tiny marker (barrier) flows with every stream. When all tasks see the same barrier they asynchronously persist their state *and* the input offsets; that’s a **checkpoint**.
    2. **Two-phase sinks** – Connectors that care about duplicates (e.g., JDBC, Kafka) open a transaction for each checkpoint, write the outgoing records, but *commit* only after the JobManager confirms the snapshot succeeded.
    3. **Crash & replay** – If a node dies after emitting counts 100–110 but before the commit, Flink rolls back to the last checkpoint (say, count = 99), rewinds Kafka to the saved offsets, replays events 100–110, recomputes the same result, then commits once. No gap, no duplicate.

- Key takeaway
    - **Data enrichment** means **attaching extra context to each raw event before we aggregate it**.
        - *Example:* a stream of `orderId, productId, qty` events is **lookup-joined** with a MySQL product table (or a broadcast catalog stream) so every record now carries `productName, price` as well; Flink offers broadcast state joins and temporal/lookup joins to do this in-flight.

    - **Re-framed pipeline**
        - *Events arrive → operators **enrich** (add product info) & **aggregate** (sum qty) while holding state → watermarks close the right time-windows → checkpoints snapshot state + input positions → on any failure Flink rewinds to that snapshot → transactional sinks commit once → clean results flow out.*

- **Watermark vs Barrier**

    - **Watermark**
    *What it is* – a special timestamp record (`Watermark{time= T}`) that a source or operator inserts every few events. It moves with the data and tells Flink “all events with `event-time ≤ T` have probably arrived.” When a window’s end time is ≤ the latest watermark, the window can safely close even if later events are still in flight.

    - **Barrier**
    *What it is* – a control record (`CheckpointBarrier{id= 77}`) that the JobManager injects at the start of a checkpoint. It never overtakes data: everything before the barrier belongs to snapshot 77; everything after belongs to the next snapshot. Operators snapshot state when the barrier reaches them, enabling exactly-once recovery.

    - **How they appear in the byte stream**

        ```
        …  eventA(t=12:00:01)  eventB(t=12:00:02)
            Watermark(12:00:02)
            eventC(t=12:00:03)
            CheckpointBarrier#77
            eventD(t=12:00:04)  …
        ```

    - **Tiny example**
    *Watermark use* – A 10-second tumbling window (12:00:00-12:00:09) closes when it sees `Watermark(12:00:09)`, emits its count, and starts the next window.
    *Barrier use* – The same stream hits `Barrier#77`; every task snapshots state (e.g., current count = 42) and source offsets. If a node fails later, Flink reloads snapshot 77 and replays from those offsets, guaranteeing no lost or duplicate events.

Sources:  
[Flink architecture](https://nightlies.apache.org/flink/flink-docs-master/docs/concepts/flink-architecture)  
[Uber streaming pipelines](https://www.uber.com/blog/building-scalable-streaming-pipelines)

⚙️ Reliability in distributed locking  
Mental model first  
A **distributed lock** is simply *“a note in a shared place saying who owns the resource and until when.”*
Failures that can erase or duplicate that note come from three directions:

| Failure class            | What happens                                                       | Consequence                                    |
| ------------------------ | ------------------------------------------------------------------ | ---------------------------------------------- |
| **Process / node crash** | Redis or the host dies                                             | Lock key disappears → anyone can grab the item |
| **Replication gap**      | Master dies, a replica takes over but hasn’t received the lock yet | New master doesn’t remember the lock           |
| **Network partition**    | App ↔ Redis link breaks                                            | Holder can’t renew; others think lock expired  |

Design is about containing those three.

Terms to know:  
* **AZ (Availability Zone)** – one data-centre building; losing an AZ = that whole building goes dark.
* **Fencing token** – every successful lock returns an ever-increasing number (1, 2, 3…). Down-stream writes must carry that number; if a stale owner with token 3 tries to write after token 4 exists, the DB rejects it.
* **CAS (Compare-And-Swap)** – “update row **only if** version =X”; a single SQL statement that succeeds for one caller and returns 0 rows for the rest.
* **Advisory lock** – a lock managed by the DB engine (e.g. `pg_try_advisory_lock`) instead of Redis; we must code our app to respect it.
* **Timeout tuning** – choosing how long the lease lasts (e.g. 5 s) and how often the owner renews it (e.g. every 2 s). Shorter leases shrink the danger window if the owner dies.

Main approaches (ascending reliability)  
| # | Technique                     | How it works                                                    | Survives                 | When it’s enough                         | Common tech                                |
| - | ----------------------------- | --------------------------------------------------------------- | ------------------------ | ---------------------------------------- | ------------------------------------------ |
| 1 | **Single Redis key with TTL** | `SET key NX PX 5000`; app renews                                | *No* Redis crash         | Very small app, best-effort stock limit  | Redis                                      |
| 2 | **Redis Sentinel / Cluster**  | 3+ nodes, one leader, async replicas                            | 1 node crash             | Medium traffic; brief oversell tolerated | Managed-Redis, ElastiCache                 |
| 3 | **Redlock**                   | Same key written to 5 independent Redis nodes; need quorum (≥3) | 2 node crashes, AZ loss  | High read/write rate; sub-ms latency     | Redisson, Redlock-Ruby                     |
| 4 | **Consensus KV store**        | Lock stored through Raft/Zab                                    | Any single-AZ loss       | Strong consistency beats latency         | etcd, ZooKeeper, Consul                    |
| 5 | **DB row + CAS (fencing)**    | `UPDATE stock SET qty = qty-1 WHERE id=… AND version=42`        | All app / cache failures | Ultimate correctness; fewer writes/sec   | Postgres advisory locks, MySQL row-version |

Putting it together for a flash-sale checkout  
1. **Acquire short lease**

   ```redis
   SET item:123 owner=svc-42 NX PX 5000
   INCR item:123:fence   --> returns 58
   ```
2. **Return fencing token 58 to checkout service.**
3. **Service renews key every 2 s**; if renewal fails, abort payment and show “Please try again”.
4. **Final stock decrement** uses the token:

   ```sql
   UPDATE items
   SET stock = stock-1 , version = 58
   WHERE id = 123 AND version = 57;
   ```

   Exactly one row updates; if 0, another buyer already won.
5. **DEL item:123** after success.

Even if Redis dies mid-sale, the service notices within 2 s (failed renew) and no longer trusts the lock; the DB write with an old token will be rejected, preventing double-sell.

Industry level example  
- **Shopify flash-sale engine (“Launchpad”)** – moved from Redis locks to MySQL row-versioning + advisory locks once volume grew.
- **Grab ride-allocation** – uses etcd locks with fencing tokens across three AZs for <100 ms median latency.
- **Netflix Conductor** – relies on Dynomite (Redis-compatible) quorum locks for orchestration steps.

⚙️ Fencing token  
Picture the problem — “the double-charge”  
*Flash-sale site, 1 item left.*  
- **Alice clicks BUY**
  - Checkout service asks Redis **“May I?”**
  - Redis replies **“Yes, you’re buyer #42”** (fencing token 42) and starts a 5 -second lease.
- **While Alice enters card details** the service **keeps pinging Redis every 2 s** to renew the lease.
- **Redis crashes** → lease cannot be renewed → after 5 s the lock is gone.
- **Bob clicks BUY**
  - New Redis leader answers **“Yes, you’re buyer #43.”**

**Both services now try to write “sold” to the database.**  
| Buyer | SQL attempted                                              | Database sees              | Result               |
| ----- | ---------------------------------------------------------- | -------------------------- | -------------------- |
| Alice | `UPDATE item SET stock = stock-1 WHERE id=1 AND token<42;` | current `token = 43`       | **0 rows → fails**   |
| Bob   | `UPDATE item SET stock = stock-1 WHERE id=1 AND token<43;` | token = 43 ➜ `<43` is true | **1 row → succeeds** |

Because the table keeps **the highest token ever written**, any late or “zombie” owner is fenced out automatically. No double-charge.

Who updates the DB?  
The *same checkout service that grabbed the lock* is the one that runs the SQL UPDATE.
Redis (or ZooKeeper/etcd) only hands out the fencing number; the database itself enforces “newest-number wins”.

Minimal, real-world sequence
- **Service A** gets lock
  - `SETNX item:1 …` → Redis returns **token 42**.
- **Service A** does the business work (talk to payment API, etc.).
- **Service A** commits the result:
   ```sql
   UPDATE items
      SET stock = stock-1,
          token = 42
    WHERE id = 1       -- the item
      AND token < 42;  -- only if my token is newer
   ```
- DB updates one row ⇒ success.

If another instance (**Service B**, token 43) reaches this step first, the row now stores `token = 43`.
When the older Service A finally tries its UPDATE, the `token < 42` clause is false, so **0 rows change** → Service A knows it lost and rolls back.

**Who touches the DB?**
Only the application instances (checkout services) do.
**Who guarantees order?**
The SQL `WHERE token < ?` guard inside the database.

Mental model  
**Token = time-stamp that only moves forward; the database accepts the newest stamp and silently blocks all older ones.**  
That’s it, no matter who crashes, wakes up, or lags, an outdated stamp can’t hurt inventory.

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

⚙️ Distributed tracing pattern  
The Distributed Tracing pattern is a crucial observability technique for understanding the end-to-end flow of requests through complex, distributed systems, especially those built with microservices. It works by assigning a **unique identifier** to a request at its inception and then **propagating this identifier** across all services and components involved in processing that request. This allows engineers to visualize the entire journey of a request, identify bottlenecks, troubleshoot errors, and understand dependencies across service boundaries.

Points  
- **Complex Distributed Systems:** This pattern is essential for environments where a single user action might trigger interactions across numerous microservices, databases, queues, and external APIs.
- **End-to-End Visibility:** Distributed tracing provides a complete picture of a request's path from its origin (e.g., a user's browser) through every service it touches, until a response is returned.
- **Trace:** A trace represents the complete, end-to-end journey of a single request or transaction through the distributed system. It's the entire story of a unit of work.
- **Span:** A span is a named, timed operation that represents a logical unit of work within a trace. Each service call, database query, or internal function execution can be a span. Spans have a start time, end time, duration, and other metadata.
- **Trace ID:** This is a unique identifier assigned to the initial request that gets propagated across all services. Every span within a single trace shares the same Trace ID.
- **Span ID:** A unique identifier for each individual span.
- **Improved Collaboration:** Provides a shared understanding across different teams (e.g., frontend, backend, database) responsible for different services.
- **Proactive Issue Detection:** Can help identify issues before they impact users by monitoring trace anomalies.

Contrast with Logs and Metrics
- **Logs:** These record discrete events at a specific point in time within a single service. They tell us *what* happened.
- **Metrics:** These aggregate numerical data over time (e.g., CPU usage, error rates). They tell us *how much* or *how often*.
- **Traces:** These provide an end-to-end view of a single request's journey, showing *why* an event occurred by connecting related operations across services. They connect the "what" and "how much" across the entire system.

Sampling  
In high-volume systems, not every trace may be collected due to performance overhead. **Sampling strategies** (e.g., head-based, tail-based) are used to select a representative subset of traces for analysis.

⚙️ Centralized logging pattern  
The Centralized Logging pattern is an architectural approach where logs generated by all applications, services, and infrastructure components across a distributed system are collected, aggregated, and stored in a single, unified location. This allows for efficient searching, analysis, monitoring, and correlation of log data, which is critical for troubleshooting, auditing, security, and understanding system behavior in complex environments.

Important Points:
- **Distributed Systems:** Especially critical in microservices or cloud-native architectures where applications are broken into numerous, often ephemeral, services running on different machines.
- **Log Generation:** Every application, service, and infrastructure component (e.g., servers, containers, databases, load balancers) generates log messages.
- **Log Aggregation:** The process of collecting logs from various sources. This typically involves agents or forwarders installed on hosts or integrated into applications that stream logs to a central system.
- **Central Storage:** A single, unified repository for all log data. This could be a specialized log management system (e.g., Elasticsearch with Kibana, Splunk, Sumo Logic), a data lake, or a distributed file system.
- **Indexing and Searchability:** Log data is usually indexed in the central system to enable fast and powerful searching, filtering, and querying based on various fields (e.g., timestamp, service name, log level, message content).
- **Visualization and Dashboards:** Centralized logging platforms often provide tools to visualize log data through dashboards, graphs, and charts, helping to identify trends, spikes, and anomalies.
- **Alerting:** The ability to configure alerts based on specific log patterns, errors, or thresholds, notifying operations teams of critical issues in real-time.
- **Correlation:** A key benefit is the ability to correlate log entries across different services that are part of the same transaction or request, even if they occurred on different machines. This is often enhanced by using unique request IDs (similar to Distributed Tracing).

**Benefits:**  
- **Simplified Troubleshooting:** Engineers can easily search and analyze logs from all parts of the system in one place, significantly speeding up the identification of root causes for issues.
- **Enhanced Observability:** Provides a comprehensive view of system behavior, aiding in understanding how different components interact.
- **Improved Security Posture:** Facilitates security monitoring, incident response, and forensic analysis by providing an auditable trail of events.
- **Compliance and Auditing:** Helps meet regulatory compliance requirements by maintaining a complete and immutable record of system activities.
- **Performance Analysis:** Allows for analysis of performance trends and identification of bottlenecks by examining log patterns.
- **Log Formats:** While not strictly part of the pattern, it's highly beneficial to adopt a consistent, structured log format (e.g., JSON) across all services. This makes parsing, indexing, and querying significantly more effective.
- **Log Levels:** Utilizing standardized log levels (e.g., DEBUG, INFO, WARN, ERROR, FATAL) helps in filtering and prioritizing log messages for analysis.
- **Log Retention:** Policies for how long logs are stored, balancing storage costs with compliance and debugging needs.
- **Common Tools/Stacks:** Popular implementations include the ELK/ECK Stack (Elasticsearch, Logstash, Kibana/Elastic Cloud on Kubernetes), Grafana Loki, Splunk, Sumo Logic, Datadog.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ┌─────────────┐                                                                │
│  │App Service 1│                                                                │
│  └─────────────┘                                                                │
│                  \                                                              │
│  ┌─────────────┐   \                                                            │
│  │App Service 2│────┼──►┌─────────────┐──►┌─────────────┐──►┌─────────────┐    │
│  └─────────────┘   /   │   Collect,  │   │   Store,    │   │  Analyse,   │    │
│                  /     │   Parse,    │   │   Index,    │   │ Visualise,  │    │
│  ┌─────────────┐       │ Transform   │   │   Search    │   │  Monitor    │    │
│  │App Service 3│       │             │   │             │   │             │    │
│  └─────────────┘       │ [Logstash]  │   │[Elasticsearch]│   │  [Kibana]   │    │
│                        └─────────────┘   └─────────────┘   └─────────────┘    │
│  ┌─────────────┐                                                               │
│  │App Service 4│                                                               │
│  └─────────────┘                                                               │
│                                                                                 │
│                              Centralized Logging                               │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**ephemeral meaning in the above context:** In this context, "ephemeral" means services are **temporary, short-lived, and can be easily created and destroyed** as needed, rather than being permanent.

⚙️ Backend For Frontend (BFF)  
**Mental Model:** Create a dedicated backend service for each frontend type (web, mobile, TV) that acts as a smart intermediary, aggregating and transforming data from multiple services into exactly what that specific frontend needs.

Key Points:
- **One BFF per frontend type** - Mobile BFF, Web BFF, TV BFF, etc.
- **Data aggregation** - Makes multiple backend calls, combines responses into single optimized payload
- **Frontend-specific optimization** - Returns data in exact format/structure needed by that client
- **Reduces network calls** - Frontend makes 1 request instead of many
- **Decouples frontend from backend evolution** - Changes in downstream services don't break frontends

Example: 
Netflix's mobile app needs user profile + recommendations + viewing history. Instead of the mobile app making 3 separate API calls, it calls its dedicated Mobile BFF once. The Mobile BFF fetches from UserService, RecommendationService, and HistoryService, then combines and formats the data specifically for mobile UI requirements before sending one optimized response back.

vs Generic API:  
Generic APIs serve all clients the same way, leading to over-fetching or under-fetching. BFFs solve this by tailoring responses per client type.
