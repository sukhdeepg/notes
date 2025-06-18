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

### Key Point
Elasticsearch doesn't understand meaning like humans do. It matches words, counts frequencies, and uses math to rank results. ML adds semantic understanding on top of this basic matching.

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

### Key Points
  * Transactions solve the problem of partial failures in multi-step operations
  * ROLLBACK happens automatically if any error occurs (or manually with ROLLBACK command)
  * MySQL's default isolation level is REPEATABLE READ
  * Use transactions for any operation that involves multiple related changes
  * Always handle exceptions in application code to ensure proper ROLLBACK

This ensures data integrity in scenarios like financial transfers, inventory updates, or any multi-table operations.