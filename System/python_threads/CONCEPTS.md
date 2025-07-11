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
