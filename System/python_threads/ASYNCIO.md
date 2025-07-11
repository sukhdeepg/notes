⚙️ Event loop working
![event_loop](https://github.com/user-attachments/assets/8caff9f6-7c6a-4ccd-a801-bba1c8fcb3f8)

Basic asyncio template  
```python
import asyncio

async def main():
    """Main async function - your entry point"""
    print("Starting async program...")
    
    # Your async code goes here
    await asyncio.sleep(1)  # Simulate async work
    
    print("Async program completed!")

# Run the async program
if __name__ == "__main__":
    asyncio.run(main())
```
- **`import asyncio`** - The core async library
- **`async def`** - Creates an async function (coroutine) that can be paused and resumed
- **`await`** - Pauses the current coroutine until the awaited operation completes
- **`asyncio.run()`** - Runs the async function in the event loop (only use this once at the top level)
- **`asyncio.sleep()`** - Non-blocking sleep (unlike `time.sleep()` which would block)

The magic of asyncio is in the `await` keyword. It lets other code run while waiting for something to complete (like network requests, file I/O, or delays). This is cooperative multitasking where functions voluntarily yield control.

```python
import asyncio

async def task_a():
    print("Task A: starting I/O operation")
    # simulate e.g. file or network read
    await asyncio.sleep(1)
    print("Task A: I/O complete")

async def task_b():
    print("Task B: starting I/O operation")
    # simulate another I/O
    await asyncio.sleep(2)
    print("Task B: I/O complete")

async def main():
    # schedule both tasks concurrently
    t1 = asyncio.create_task(task_a())
    t2 = asyncio.create_task(task_b())
    # wait for both to finish
    await t1
    await t2
    print("All tasks done.")

if __name__ == "__main__":
    # starts the single-threaded event loop
    asyncio.run(main())
```
```text
Task A: starting I/O operation
Task B: starting I/O operation
Task A: I/O complete
Task B: I/O complete
All tasks done.
```

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ When to use `gather` and `create_task`?

Code using both
```python
import asyncio
import time

async def fetch_data(url, delay):
    """Simulate fetching data from a URL"""
    print(f"🔄 Fetching {url}...")
    await asyncio.sleep(delay)  # Simulate network delay
    print(f"✅ Got data from {url}")
    return f"Data from {url}"

async def main():
    print("=== LEVEL 2: Multiple Tasks ===\n")
    
    # Method 1: await one by one (SEQUENTIAL - slow)
    print("1. Sequential execution:")
    start = time.time()
    result1 = await fetch_data("api.com/users", 1)
    result2 = await fetch_data("api.com/posts", 2)
    print(f"Sequential time: {time.time() - start:.1f}s\n")
    
    # Method 2: gather() - run concurrently (PARALLEL - fast)
    print("2. Concurrent with gather():")
    start = time.time()
    results = await asyncio.gather(
        fetch_data("api.com/users", 1),
        fetch_data("api.com/posts", 2),
        fetch_data("api.com/comments", 1.5)
    )
    print(f"Concurrent time: {time.time() - start:.1f}s")
    print(f"Results: {results}\n")
    
    # Method 3: create_task() - more control
    print("3. Using create_task():")
    start = time.time()
    task1 = asyncio.create_task(fetch_data("api.com/profile", 1))
    task2 = asyncio.create_task(fetch_data("api.com/settings", 2))
    
    # Can do other work here while tasks run in background
    print("Tasks started, doing other work...")
    await asyncio.sleep(0.5)
    print("Other work done, now waiting for tasks...")
    
    # Wait for tasks to complete
    result1 = await task1
    result2 = await task2
    print(f"Task time: {time.time() - start:.1f}s")

if __name__ == "__main__":
    asyncio.run(main())
```
<img width="656" alt="image" src="https://github.com/user-attachments/assets/bf6da6c4-bd11-456d-aa41-a010510c6035" />

**Use `asyncio.gather()` when:**
- We want to run multiple tasks and wait for **ALL** of them to complete
- We need the results in the **same order** as we passed them
- Simple "fire and forget all, then collect all results" scenario

```python
# All tasks, wait for all, get ordered results
results = await asyncio.gather(task1, task2, task3)
```

**Use `asyncio.create_task()` when:**
- We want **more control** over individual tasks
- We need to do **other work** while tasks run in background
- We want to wait for tasks at **different times**
- We might need to **cancel** specific tasks
- We want to handle **each task's errors** separately

```python
# Start tasks immediately, control when to wait
task1 = asyncio.create_task(fetch_data())
task2 = asyncio.create_task(fetch_data())

# Do other work...
await asyncio.sleep(1)

# Wait for specific task when we need it
result1 = await task1  # Wait for this one now
# Maybe cancel task2 if needed
result2 = await task2  # Wait for this one later
```

**Quick rule**: Use `gather()` for simple parallel execution, use `create_task()` when we need more control over timing and individual task management.

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ The key difference in **blocking vs non-blocking** sleep:  
**`time.sleep(1)`** - **BLOCKING**
- Completely freezes our entire program for 1 second
- Nothing else can run during this time
- Like putting our whole program to sleep

**`await asyncio.sleep(1)`** - **NON-BLOCKING**
- Pauses only the current async function for 1 second
- Other async functions can run during this time
- Like saying "I'll wait here, but others can keep working"

```python
import asyncio
import time

async def worker(name):
    print(f"{name} starting work...")
    await asyncio.sleep(2)  # NON-BLOCKING - others can work
    print(f"{name} finished work!")

async def main():
    print("With asyncio.sleep (NON-BLOCKING)")
    start = time.time()
    
    # Both workers run concurrently
    await asyncio.gather(
        worker("Worker 1"),
        worker("Worker 2")
    )
    
    print(f"Total time: {time.time() - start:.1f} seconds")
    print("Notice: Both workers ran at the same time!")

# Run the async program
if __name__ == "__main__":
    asyncio.run(main())
```

Here's a quick example to see the difference:  
If we used `time.sleep(2)` instead of `await asyncio.sleep(2)`, both workers would run one after the other, taking 4 seconds total. With `asyncio.sleep()`, they run concurrently and finish in just 2 seconds.

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ Exception handling with `asyncio`
```python
import asyncio
import random

async def unreliable_api(name, delay, fail_chance=0.3):
    """Simulate an unreliable API that might fail or be slow"""
    print(f"🔄 Calling {name}...")
    await asyncio.sleep(delay)
    
    if random.random() < fail_chance:
        raise Exception(f"❌ {name} failed!")
    
    print(f"✅ {name} succeeded!")
    return f"Data from {name}"

async def slow_api(name, delay):
    """Simulate a slow API"""
    print(f"🐌 Calling slow {name}...")
    await asyncio.sleep(delay)
    print(f"✅ Slow {name} completed!")
    return f"Data from {name}"

async def main():    
    # 1. Try/except with single task
    print("1. Basic error handling:")
    try:
        result = await unreliable_api("API-1", 1)
        print(f"Success: {result}")
    except Exception as e:
        print(f"Error: {e}")
    
    print()
    
    # 2. Error handling with gather() - one failure stops all
    print("2. Error handling with gather():")
    try:
        results = await asyncio.gather(
            unreliable_api("API-1", 1),
            unreliable_api("API-2", 1.5),
            unreliable_api("API-3", 0.5)
        )
        print(f"All succeeded: {results}")
    except Exception as e:
        print(f"One failed, all stopped: {e}")
    
    print()
    
    # 3. Error handling with gather() - continue on failure
    print("3. Continue on failure with return_exceptions=True:")
    results = await asyncio.gather(
        unreliable_api("API-1", 1),
        unreliable_api("API-2", 1.5),
        unreliable_api("API-3", 0.5),
        return_exceptions=True  # Returns exceptions instead of raising
    )
    
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Task {i+1} failed: {result}")
        else:
            print(f"Task {i+1} succeeded: {result}")
    
    print()
    
    # 4. Timeout handling
    print("4. Timeout handling:")
    try:
        result = await asyncio.wait_for(
            slow_api("slow-api", 3),  # Takes 3 seconds
            timeout=2.0  # But we only wait 2 seconds
        )
        print(f"Success: {result}")
    except asyncio.TimeoutError:
        print("❌ Operation timed out after 2 seconds!")
    
    print()
    
    # 5. Task cancellation
    print("5. Task cancellation:")
    task = asyncio.create_task(slow_api("cancellable-api", 5))
    
    # Wait a bit then cancel
    await asyncio.sleep(1)
    task.cancel()
    
    try:
        result = await task
        print(f"Success: {result}")
    except asyncio.CancelledError:
        print("❌ Task was cancelled!")
    
    print()
    
    # 6. Timeout with multiple tasks
    print("6. Timeout with multiple tasks:")
    try:
        results = await asyncio.wait_for(
            asyncio.gather(
                slow_api("slow-1", 1),
                slow_api("slow-2", 4),  # This one is too slow
                slow_api("slow-3", 2)
            ),
            timeout=3.0
        )
        print(f"All completed: {results}")
    except asyncio.TimeoutError:
        print("❌ Some tasks took too long!")

if __name__ == "__main__":
    asyncio.run(main())
```
<img width="364" alt="image" src="https://github.com/user-attachments/assets/6d69c90d-ed33-4441-a74e-3b57a3147087" />

1. **`try/except`**: Handle errors in async functions just like regular functions
2. **`return_exceptions=True`**: In `gather()`, return exceptions instead of stopping everything
3. **`asyncio.wait_for()`**: Set timeouts for operations
4. **`asyncio.TimeoutError`**: Raised when operations take too long
5. **`task.cancel()`**: Cancel running tasks
6. **`asyncio.CancelledError`**: Raised when a task is cancelled

- Set timeouts for external APIs
- Use `return_exceptions=True` when you want partial results
- Cancel tasks to free up resources when we don't need them anymore

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ Use of `asyncio.open_connection`  
Used when we need to ingest a never-ending stream of data (logs, metrics, sensor feeds, chat messages, etc.) without ever loading it all into memory. Example: an async log collector that connects over TCP to many servers sending us lines of log text, processes each line as it arrives, and never buffers more than 1 KB at a time.
```python
import asyncio

async def process_log_line(line: str):
    # your real processing: parse, enrich, send to storage, etc.
    print(f"[LOG] {line}")

async def stream_logs(host: str, port: int):
    # reader: async stream to receive bytes
    # writer: async stream to send bytes
    reader, writer = await asyncio.open_connection(host, port)
    buffer = b""
    print(f"Connected to {host}:{port}, streaming logs…")

    while True:
        chunk = await reader.read(1024)    # up to 1024 bytes when any is ready
        if not chunk:                      # empty ⇒ remote closed
            break

        buffer += chunk
        # split off each complete line
        while b"\n" in buffer:
            # the 1 means “split at most once,” so we get just the part before the
            # first newline and the rest as two separate values.
            line, buffer = buffer.split(b"\n", 1)
            # tells Python to substitute any bytes that can’t be decoded with the Unicode
            # replacement character (�) instead of raising an exception.
            text = line.decode("utf-8", errors="replace")
            await process_log_line(text)

            # send an ACK back to the server for this line
            writer.write(b"ACK\n")        # buffer "ACK\n" to send
            await writer.drain()          # wait until it's actually sent

    print("Connection closed by server")
    writer.close()                       # politely close our side of the connection
    await writer.wait_closed()           # wait for the close handshake

async def main():
    await stream_logs("logs.internal.example.com", 514)

if __name__ == "__main__":
    asyncio.run(main())
```

- **Fixed-sized chunks** (`1024` bytes) bound memory usage. No risk of piling up arbitrarily large messages.
- `await reader.read(1024)` returns *as soon as any data* (up to 1024 B) is available. If only 200 B arrived, we get 200 B immediately. We don’t wait for a full kilobyte.
- If *more* than 1024 B arrives in one shot, the extra stays hidden in the OS buffer; our next `read()` call picks up the remainder.
- We assemble those chunks into complete “lines” (delimited by `\n`), process each as soon as it’s ready, and keep looping. Perfect for long-lived streams.
- `writer.write(data)`: queues `data` (here `"ACK\n"`) to be sent over the same TCP connection.
- `await writer.drain()`: waits until the internal write buffer is flushed to the OS (i.e. actually sent).

This exact approach scales to any protocol or service that streams data over TCP: real-time chat servers, sensor networks (IoT), video/audio streaming (we’d pick a larger chunk size), proxy servers, even HTTP downloads where we want to display a progress bar without loading the whole file at once.

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ Why `asyncio` is faster than `threading` module  
> `asyncio` juggles 10,000+ waits with one thread

- **One worker, many jobs**
    - We run a **single** operating-system thread (usually the main one).
    - Inside that thread lives the **event-loop**. A forever-running `while True` that decides *which* coroutine gets CPU time next.
- **Jobs hand back control themselves**
    - A coroutine is just a Python function that can **pause** at an `await` and later **resume** right where it left off.
    - When it reaches an `await` that would block (e.g., “wait for data from the network”), it *voluntarily* pauses and the loop immediately moves on to another coroutine that is ready.
- **Tiny pause-and-resume record**
    - To come back later, Python stores a **small “bookmark”**, roughly 1–2 KB, holding local variables and the next line to run.
    - Compare that with a real thread: the OS reserves roughly **0.5–2 MB** of stack memory even if the thread is mostly idle.
    - Result: we can keep **tens of thousands** of coroutine bookmarks in the same space that just a few hundred threads would need.
- **The OS gives a ready-made short list**
    - The event-loop asks the kernel, “Which of my sockets just got data?” using `epoll` (Linux) or `kqueue` (macOS/BSD).
    - The kernel **already knows** which sockets woke up and hands back only that short list, maybe 5 out of 10 000. No need to scan them all again.
- **No expensive “kernel jump” on every switch**
    - All coroutine switches happen in **user space** (your program’s normal area).
    - A real thread switch forces the CPU to jump into the **kernel** (the privileged core of the OS) and back out, which costs a few micro-seconds.
        - When the computer swaps threads, it pauses our program, runs some operating-system code, then returns. This detour takes a few microseconds.
        - "some operating-system code" - the OS scheduler must save the current thread’s registers, pick the next thread in its queue, and load that thread’s registers before letting our program continue.
    - Coroutine switches stay in user space and take just a few hundred nano-seconds.
        - User space = ring-3 (least-privileged CPU mode for regular programs), non-privileged CPU mode where each process runs inside its own virtual-memory page table; only on a system call does the CPU switch to ring-0 (most-privileged CPU mode where the operating system runs) kernel space (privileged mode) to execute scheduler, drivers, or other core OS code. 
- **What if you need real parallel CPU work?**
    - That single thread can still only execute one Python instruction at a time (because of the GIL).
    - When we hit heavy number-crunching, hand those tasks off to a `ProcessPoolExecutor` (separate processes, separate CPUs) and let the event-loop keep handling the I/O waits.

> “`asyncio` keeps one thread asking the OS which sockets are ready and only resumes the coroutines that need attention. Each coroutine’s pause-state is just a 2 KB bookmark, and switching between them never leaves user space, so one thread can juggle tens of thousands of I/O waits where hundreds of real threads would run out of memory and slow down.”

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ Common design questions
> Concurrent Log Ingestion and Analysis Pipeline  

This design simulates a system where multiple "producers" generate log data and multiple "consumers" process it concurrently. We use an `asyncio.Queue` as the central, thread-safe buffer between them.

```python
import asyncio
import random
import time

# asyncio.Queue is our thread-safe buffer. It handles the complexities of
# locking and waiting, ensuring that producers can safely add items and
# consumers can safely remove them without conflicts, even when many
# are running at once.
log_queue = asyncio.Queue(maxsize=100) # Max size helps manage memory

# Shared Resource for Processed Data
# This is a simple list to store the results of the analysis.
# We'll need a lock to make sure that when consumers add items to it,
# they don't corrupt the data.
processed_logs = []
processed_logs_lock = asyncio.Lock() # A mutex lock for our results list

# The Log Producer
# Simulates an agent or endpoint that generates log data.
async def log_producer(producer_id: int):
    """
    Generates logs and puts them into the shared queue.
    """
    for i in range(5): # Each producer will generate 5 logs
        # Simulate I/O wait (return a random floating-point number between the given boundary)
        await asyncio.sleep(random.uniform(0.1, 0.5))
        log_entry = f"Log entry {i} from producer {producer_id}"
        
        # 'put' will wait if the queue is full, preventing system overload.
        await log_queue.put(log_entry)
        print(f"🌲 Producer {producer_id}: Added log to queue.")

# The Log Consumer
# Simulates a worker that processes log data from the queue.
async def log_consumer(consumer_id: int):
    """
    Fetches logs from the queue and processes them.
    """
    while True:
        try:
            # 'get' will wait if the queue is empty.
            # The timeout prevents it from waiting forever if no more logs are produced.
            log_entry = await asyncio.wait_for(log_queue.get(), timeout=3.0)
            
            # Log Analysis Simulation
            # Here, we would do the actual work: parse the log, check for
            # patterns, query a database, etc. We'll just simulate it.
            print(f"🔬 Consumer {consumer_id}: Processing '{log_entry}'...")
            await asyncio.sleep(random.uniform(0.5, 1.0)) # Simulate processing time
            processed_result = f"Analyzed: {log_entry.upper()}"
            
            # Safely Writing to Shared Results
            # To prevent a race condition (where two consumers might try to
            # write to the list at the exact same time and cause an error),
            # we acquire the lock. Only one consumer can hold the lock at a time.
            async with processed_logs_lock:
                processed_logs.append(processed_result)
            
            # Mark the task as done. This is important for queue management.
            log_queue.task_done()
            print(f"✅ Consumer {consumer_id}: Finished processing.")

        except asyncio.TimeoutError:
            # If the queue is empty for too long, we assume there's no more work.
            print(f"🌙 Consumer {consumer_id}: No more logs to process. Shutting down.")
            break # Exit the loop to stop the consumer

async def main():
    """
    Main function to set up and run the producers and consumers.
    """
    start_time = time.time()
    
    # The "Thread Pool" (TaskGroup)
    # asyncio.TaskGroup is the modern way to manage a group of concurrent tasks.
    # It's like a thread pool: we create tasks within it, and it ensures
    # that all tasks are completed before the program moves on. This is
    # how we run multiple producers and consumers in parallel.
    async with asyncio.TaskGroup() as tg:
        # Create producer tasks
        for i in range(3): # 3 producers
            tg.create_task(log_producer(i + 1))
            
        # Create consumer tasks
        for i in range(4): # 4 consumers to handle the load
            tg.create_task(log_consumer(i + 1))
            
    # The 'async with tg' block will wait here until all tasks are done.

    print("\n--- All logs processed! ---")
    print(f"Total logs analyzed: {len(processed_logs)}")
    print(f"Processed logs: {processed_logs}")
    print(f"Total time taken: {time.time() - start_time:.2f} seconds")

if __name__ == "__main__":
    asyncio.run(main())
```

> Thread-Safe In-Memory Cache  

This design creates a cache for threat data. It needs to handle many "reader" tasks looking up data and occasional "writer" tasks updating it, all without conflict. The key challenge is to allow multiple readers at the same time for performance while ensuring writers have exclusive access.  
For this, a Read-Write Lock is ideal, but `asyncio` doesn't have a built-in one. We can simulate it for clarity or use a simple `asyncio.Lock` for demonstration. Let's use `asyncio.Lock` for simplicity and comment on where a Read-Write lock would be better.

```python
import asyncio
import random
from collections import OrderedDict

class ThreatCache:
    """
    A thread-safe, in-memory cache with an LRU (Least Recently Used) eviction policy.
    """
    def __init__(self, capacity: int = 5):
        # The cache itself. OrderedDict remembers the insertion order,
        # which is perfect for an LRU eviction strategy.
        self._cache = OrderedDict()
        self.capacity = capacity
        
        # The Lock
        # This single lock protects the cache. Any time a task wants to read
        # or write to self._cache, it must first acquire this lock.
        #
        # **IDEAL SCENARIO**: A Read-Write lock would be better. It would allow
        # multiple readers to access the cache simultaneously, only locking
        # exclusively when a writer needs access. This improves performance
        # on a read-heavy system. We use a simple lock here for clarity.
        self._lock = asyncio.Lock()

    async def get(self, key: str) -> str | None:
        """
        Gets a value from the cache. If found, it's marked as recently used.
        """
        async with self._lock: # Acquire lock to safely access the cache
            if key not in self._cache:
                return None
            
            # LRU Policy: Mark as Recently Used
            # Move the accessed item to the end of the ordered dict to show
            # it was just used. The item at the beginning will be the least
            # recently used.
            self._cache.move_to_end(key)
            return self._cache[key]

    async def set(self, key: str, value: str):
        """
        Sets a value in the cache, evicting the least recently used item if full.
        """
        async with self._lock: # Acquire lock for exclusive write access
            if key in self._cache:
                # If the key already exists, just update it and move to the end.
                self._cache.move_to_end(key)
            
            self._cache[key] = value
            
            # LRU Policy: Eviction
            # If the cache has exceeded its capacity, remove the first item,
            # which is the least recently used.
            if len(self._cache) > self.capacity:
                evicted_key, _ = self._cache.popitem(last=False)
                print(f"🗑️ Cache full. Evicted '{evicted_key}'.")

    async def display(self):
        """Helper to display the current state of the cache."""
        async with self._lock:
            print(f"📦 Current Cache: {list(self._cache.items())}")

# Simulation Tasks
async def reader_task(cache: ThreatCache, reader_id: int):
    """Simulates a task that reads from the cache."""
    keys_to_read = ["1.1.1.1", "bad-hash.exe", "8.8.8.8"]
    for key in keys_to_read:
        await asyncio.sleep(random.uniform(0.1, 0.3))
        value = await cache.get(key)
        if value:
            print(f"🧐 Reader {reader_id}: Found '{key}' -> '{value}'")
        else:
            print(f"❓ Reader {reader_id}: Did not find '{key}'")
            
async def writer_task(cache: ThreatCache, writer_id: int):
    """Simulates a task that writes to the cache."""
    threats = [
        ("1.1.1.1", "Cloudflare DNS"),
        ("bad-hash.exe", "Ransomware"),
        ("evil-domain.com", "Phishing Site"),
        ("another-hash.dll", "Spyware"),
        ("8.8.8.8", "Google DNS"),
        ("final-threat.sh", "Trojan")
    ]
    for key, value in threats:
        print(f"✍️ Writer {writer_id}: Adding '{key}'...")
        await cache.set(key, value)
        await cache.display()
        await asyncio.sleep(random.uniform(0.5, 1.0))

async def main_cache():
    """
    Main function to run the cache simulation.
    """
    cache = ThreatCache(capacity=4) # Cache can only hold 4 items
    
    # We run readers and a writer concurrently to test for race conditions.
    # The lock inside the cache class should prevent any issues.
    async with asyncio.TaskGroup() as tg:
        tg.create_task(writer_task(cache, 1)) # One writer task
        
        # Multiple reader tasks trying to access the cache at the same time
        for i in range(3):
            tg.create_task(reader_task(cache, i + 1))
            
    print("\n Cache simulation finished!")
    await cache.display()

if __name__ == "__main__":
    asyncio.run(main_cache())
```
