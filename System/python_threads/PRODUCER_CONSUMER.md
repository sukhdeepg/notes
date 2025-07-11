### Concurrent Log Ingestion and Analysis Pipeline
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
