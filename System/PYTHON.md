⚙️ Python multi-threading and multi-processing  
**GIL (Global Interpreter Lock)** - Python's fundamental constraint. Only one thread can execute Python bytecode at a time.

![image](https://github.com/user-attachments/assets/d25bec96-cb94-407e-aa2a-8dcd42cb7348)

- Multi-threading
    - Multiple threads, but GIL allows only one to run Python code simultaneously
    - **Good for:** I/O-bound tasks (file reading, network requests, database calls)
    - **Bad for:** CPU-intensive tasks (calculations, data processing)
    - Threads can release GIL during I/O operations, allowing others to work
    - Why threading helps with I/O:** When Thread A waits for disk/network, it releases GIL, Thread B can work.

- Multi-processing
    - Separate Python processes, each with its own GIL
    - **Good for:** CPU-intensive tasks
    - **Cost:** Higher memory usage, slower inter-process communication

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ Event loop working
![event_loop](https://github.com/user-attachments/assets/8caff9f6-7c6a-4ccd-a801-bba1c8fcb3f8)
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

⚙️ Python daemon threads    
Core Concept: Daemon threads in Python are background threads that automatically terminate when the main program exits, unlike regular threads which keep the program running until they complete.

**Key Points**
- **Definition** - Daemon threads are threads marked with `daemon=True` that run in the background and don't prevent program termination
- **Lifecycle behavior** - When the main thread finishes, Python immediately kills all daemon threads without waiting for them to complete their work
- **Setting daemon status** - Use `thread.daemon = True` before calling `thread.start()`, or pass `daemon=True` to the Thread constructor
- **Use cases** - Background tasks like logging, monitoring, periodic cleanup, or services that should stop when the main application stops
- **Limitations** - Daemon threads cannot perform critical cleanup operations since they may be terminated abruptly mid-execution
- **Thread safety** - Daemon status doesn't affect thread synchronization requirements - still need locks, queues, etc. for shared resources
- **Checking status** - Use `thread.daemon` property to check if a thread is marked as daemon
- **Cannot change after start** - Daemon status must be set before calling `thread.start()`, attempting to change it afterward raises RuntimeError

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ `select`, `epoll`, and `kqueue`  
Imagine we’re writing a chat server.  
We might have **10 000 client connections** open at once, but at any given moment only a few of them send data.  
Our program has to sit there asking:

> “Did **any** of my connections get a new message? If yes, which ones?”

Doing that one connection at a time would be painfully slow, so operating systems give us helper calls that watch **many** connections simultaneously.  

---

| Name                           | How you use it                                                                                                                                           | What’s happening under the hood                                                                                                               | Good for                                          |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| **`select`** (old, everywhere) | Every time we ask, we hand the OS a **list of sockets** (e.g. `[sock1, sock2, …]`). The OS scans the whole list and tells us which sockets are ready. | The scanning work is repeated on every call, so big lists (thousands of sockets) get slower and slower.                                       | Small programs with only a few dozen connections. |
| **`epoll`** (Linux)            | We **register** each socket once (`epoll.register(sock)`). After that we just call `epoll.poll()`, which returns any sockets that became ready.        | The OS keeps an internal “ready” set. No need to rescan our full list each time, so performance stays almost flat even with 100 000 sockets. | High-traffic servers or scrapers on Linux.        |
| **`kqueue`** (macOS / FreeBSD) | Same idea as `epoll`, different name. We register once, then poll for ready events.                                                                     | Kernel-side ready list, so it scales like `epoll`.                                                                                            | High-traffic servers on macOS or BSD systems.     |

```python
import selectors, socket

sel = selectors.DefaultSelector()      # Picks epoll on Linux, kqueue on macOS

def watch(sock):
    sock.listen()
    sel.register(sock, selectors.EVENT_READ)  # Tell OS: “Wake me if someone connects.”

# make two listening sockets just for demo
for _ in range(2):
    s = socket.socket(); s.bind(("localhost", 0))
    watch(s)

while True:
    ready = sel.select(timeout=1)      # Under the hood: epoll/kqueue/select
    for key, _ in ready:               # key.fileobj is the socket that’s ready
        conn, _ = key.fileobj.accept()
        print("New client!")
```

| Step                              | `select` (old way)                           | `epoll` / `kqueue` (modern way)                                                                                 |
| --------------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **1. Setup**                      | —                                            | We say once: “Watch socket A, B, C, … Z.”                                                                      |
| **2. Data arrives on socket K**   | Nothing yet.                                 | Kernel immediately drops **K** into its internal “ready bucket”.                                                |
| **3. We ask, “Anything ready?”** | Kernel loops over **A-Z** checking each one. | Kernel looks at the bucket and hands back **\[K]** right away.                                                  |
| **4. Loop repeats**               | The loop above happens *every* time.         | Only sockets added to the bucket since last call are returned—cost stays low no matter how many you registered. |

<hr width="100%" size="2" color="#007acc" noshade>

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

<hr width="100%" size="2" color="#007acc" noshade>

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
