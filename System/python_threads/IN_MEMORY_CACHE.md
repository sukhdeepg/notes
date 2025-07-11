### Thread-Safe In-Memory Cache
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
