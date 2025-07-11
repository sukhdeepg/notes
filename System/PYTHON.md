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

<hr width="100%" size="2" color="#007acc" noshade>

⚙️ `OrderedDict`  
Maintains insertion order of keys (like regular dict in Python 3.7+) but adds extra methods for reordering.

**Core Operations & Time Complexity:**  

**Access/Lookup:**
- `dict[key]` → **O(1)** average
- `key in dict` → **O(1)** average

**Modification:**
- `dict[key] = value` (add/update) → **O(1)** average
- `del dict[key]` (remove) → **O(1)** average
- `dict.pop(key)` → **O(1)** average

**Ordering Operations:**
- `dict.move_to_end(key)` → **O(1)** 
- `dict.move_to_end(key, last=False)` (move to beginning) → **O(1)**
- `dict.popitem()` (remove last) → **O(1)**
- `dict.popitem(last=False)` (remove first) → **O(1)**

**Iteration:**
- Iterate through keys/values/items → **O(n)**

**Example:**
```python
from collections import OrderedDict

od = OrderedDict([('a', 1), ('b', 2), ('c', 3)])
print(od)  # OrderedDict([('a', 1), ('b', 2), ('c', 3)])

od.move_to_end('a')  # Move 'a' to end
print(od)  # OrderedDict([('b', 2), ('c', 3), ('a', 1)])

od.move_to_end('c', last=False)  # Move 'c' to beginning  
print(od)  # OrderedDict([('c', 3), ('b', 2), ('a', 1)])
```

**Key advantage:** Fast reordering operations that would be expensive with regular dicts or lists.