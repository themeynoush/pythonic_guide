# The Pythonic Code Field Guide
### Mistakes to avoid in production Python — ranked by damage

This guide is a practical checklist for writing Pythonic code: code that is secure, correct, resource-aware, fast enough, and easy to read. Each entry shows a bad pattern, a better pattern, the primary impact, complexity/resource notes, and the rare cases where the “bad” pattern is intentional.

The goal is not clever code. The goal is code that is hard to misuse, cheap to run, clear to maintain, and honest about edge cases.

---

## Ground Rules

- **Correctness beats cleverness.** A fast bug is still a bug.
- **Security beats Big O.** An O(1) security hole is worse than an O(n²) loop over ten items.
- **Big-O notes are about growth, not benchmark guarantees.** Constants, Python version, interpreter, CPU, and data shape matter.
- **CPython details are not always Python-language guarantees.** When this guide relies on CPython behavior, it says so.
- **“Avoid” does not mean “impossible to use.”** It means “use only with a documented reason.”

---

## Severity Legend

| Level | Label | Meaning |
|---|---|---|
| 🔴 | **CRITICAL** | Can create security vulnerabilities, arbitrary code execution, data loss, or invalid authorization |
| 🟠 | **HIGH** | Common correctness bugs, resource leaks, race-prone code, or hard-to-debug state corruption |
| 🟡 | **MEDIUM** | Meaningful resource or speed cost at scale |
| 🟢 | **LOW** | Readability, maintainability, or style; usually little runtime impact |

---

## Impact Categories

| Category | Meaning |
|---|---|
| **Security** | Prevents code injection, command injection, SQL injection, predictable secrets, unsafe deserialization, or authorization bypass |
| **Correctness** | Prevents wrong results, hidden errors, shared state bugs, race-prone logic, or broken polymorphism |
| **Resource Cost** | Reduces unnecessary memory, file handles, sockets, locks, subprocesses, or temporary objects |
| **Speed** | Reduces avoidable CPU time or algorithmic growth |
| **Cleanliness** | Improves readability and maintainability |

---

# 🔴 CRITICAL — Must Not Reach Production Without a Documented Reason

---

## 1. `eval()` / `exec()` on External Input

**Primary impact: Security — arbitrary Python execution or unbounded side effects**

```python
# ❌ BAD — user input becomes Python code
expr = request.args["filter"]
result = eval(expr)

# Dangerous input can be an expression that executes code:
# __import__("os").system("rm -rf /tmp/app-data")

config = json.loads(raw_config)
exec(config["setup_code"])
```

```python
# ✅ GOOD — parse data, do not execute strings
import json

data = json.loads(raw_config)  # still limit input size for untrusted data

# ✅ GOOD — allowlist behavior instead of evaluating code
import operator

OPS = {
    "eq": operator.eq,
    "lt": operator.lt,
    "gt": operator.gt,
}

op_name = data["op"]
if op_name not in OPS:
    raise ValueError(f"Unsupported operation: {op_name}")

result = OPS[op_name](data["left"], data["right"])
```

```python
# ✅ OK for trusted literals only — not a general safe parser
import ast

value = ast.literal_eval("{'ports': [80, 443]}")
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) parse + unbounded execution | O(n) parse/validation |
| Space | Input-dependent; can allocate unbounded memory | O(n), with size limits |
| Risk | Arbitrary code execution | Data is parsed, not executed |

**When `eval` / `exec` is acceptable:**
- Trusted developer tooling, local debug consoles, or intentionally programmable systems.
- Never for user input, config files from untrusted locations, request data, database content, or messages.

**Official sources:** [`eval()`](https://docs.python.org/3/library/functions.html#eval), [`exec()`](https://docs.python.org/3/library/functions.html#exec), [`ast.literal_eval()` warning](https://docs.python.org/3/library/ast.html#ast.literal_eval), [`json` warning](https://docs.python.org/3/library/json.html)

---

## 2. Unpickling Untrusted Data

**Primary impact: Security — arbitrary code execution during deserialization**

```python
# ❌ BAD — request body controls what gets instantiated/executed
import pickle

obj = pickle.loads(request.body)
```

```python
# ✅ GOOD — use a data format, then validate expected structure
import json

payload = json.loads(request.body)
if not isinstance(payload, dict):
    raise ValueError("Expected a JSON object")
```

```python
# ✅ GOOD — if you must receive binary data, define your own schema
from dataclasses import dataclass

@dataclass(frozen=True)
class Packet:
    src: str
    dst: str
    port: int

def parse_packet(data: dict) -> Packet:
    return Packet(src=str(data["src"]), dst=str(data["dst"]), port=int(data["port"]))
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) deserialize + arbitrary behavior | O(n) parse + validation |
| Space | O(n) + arbitrary object graph | O(n), bounded by schema/input size |
| Risk | Code execution during unpickling | Data-only parsing |

**When `pickle` is acceptable:**
- Trusted, internal-only data where both producer and consumer are controlled.
- Signed data can prove integrity, but it does not make an untrusted producer safe.

**Official source:** [`pickle` warning](https://docs.python.org/3/library/pickle.html)

---

## 3. Building SQL With String Formatting

**Primary impact: Security — SQL injection**

```python
# ❌ BAD — user input becomes SQL syntax
symbol = request.args["symbol"]
cur.execute(f"SELECT * FROM stocks WHERE symbol = '{symbol}'")
```

```python
# ✅ GOOD — bind values with placeholders
symbol = request.args["symbol"]
cur.execute("SELECT * FROM stocks WHERE symbol = ?", (symbol,))
```

```python
# ✅ GOOD — for identifiers, use an allowlist because placeholders bind values, not SQL syntax
ALLOWED_SORTS = {"symbol", "price", "created_at"}
sort_by = request.args.get("sort", "symbol")
if sort_by not in ALLOWED_SORTS:
    raise ValueError("Invalid sort column")

cur.execute(f"SELECT * FROM stocks ORDER BY {sort_by}")
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) string construction | O(n) binding/parsing |
| Space | O(n) | O(n) |
| Risk | SQL injection | Values are bound as values |

**When string-built SQL is acceptable:**
- Static SQL with no external values.
- Dynamic identifiers only after strict allowlisting.

**Official source:** [`sqlite3` placeholders](https://docs.python.org/3/library/sqlite3.html#how-to-use-placeholders-to-bind-values-in-sql-queries)

---

## 4. `subprocess(..., shell=True)` With External Input

**Primary impact: Security — shell command injection**

```python
# ❌ BAD — shell parses user-controlled text
import subprocess

pattern = request.args["pattern"]
filename = request.args["file"]
subprocess.run(f"grep {pattern} {filename}", shell=True, check=True)
```

```python
# ✅ GOOD — pass argv as a list; no shell parsing
import subprocess

pattern = request.args["pattern"]
filename = request.args["file"]
subprocess.run(["grep", pattern, filename], check=True)
```

```python
# ✅ GOOD — use Python APIs when you do not need an external process
from pathlib import Path

needle = request.args["pattern"]
text = Path(filename).read_text(encoding="utf-8")
matched = [line for line in text.splitlines() if needle in line]
```

| Metric | Bad | Good |
|---|---|---|
| Time | Process startup + shell parsing + command | Process startup + command |
| Space | OS/process-dependent | OS/process-dependent |
| Risk | Command injection | Arguments are passed as arguments |

**When `shell=True` is acceptable:**
- When you truly need shell features such as pipes, globbing, or shell built-ins, and every part of the command is trusted or correctly quoted.

**Official source:** [`subprocess` security considerations](https://docs.python.org/3/library/subprocess.html#security-considerations)

---

## 5. `random` for Passwords, Tokens, or Security Decisions

**Primary impact: Security — predictable pseudo-random values**

```python
# ❌ BAD — random is not for security
import random
import string

token = "".join(random.choice(string.ascii_letters + string.digits) for _ in range(32))
```

```python
# ✅ GOOD — secrets is for security-sensitive randomness
import secrets

token = secrets.token_urlsafe(32)
```

```python
# ✅ GOOD — choose from an alphabet securely
import secrets
import string

alphabet = string.ascii_letters + string.digits
password = "".join(secrets.choice(alphabet) for _ in range(32))
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(n) | O(n) |
| Risk | Predictable tokens | Cryptographically strong randomness |

**When `random` is acceptable:**
- Simulations, tests, games, randomized algorithms, sampling, and non-security use cases.

**Official sources:** [`random` warning](https://docs.python.org/3/library/random.html), [`secrets`](https://docs.python.org/3/library/secrets.html)

---

## 6. `assert` for Runtime Validation, Authorization, or Data Safety

**Primary impact: Security + Correctness — assertions can be removed with optimization**

```python
# ❌ BAD — this check can disappear under python -O
assert user.is_admin
transfer_funds(src, dst, amount)

# ❌ BAD — input validation can disappear
assert amount > 0
```

```python
# ✅ GOOD — explicit checks remain active
if not user.is_admin:
    raise PermissionError("Admin privileges required")

if amount <= 0:
    raise ValueError("amount must be positive")

transfer_funds(src, dst, amount)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1), or removed under optimization | O(1) |
| Space | O(1) | O(1) |
| Risk | Check may not run | Check always runs |

**When `assert` is acceptable:**
- Internal invariants and debugging assumptions, not user-facing validation or security decisions.

**Official sources:** [`assert` statement](https://docs.python.org/3/reference/simple_stmts.html#the-assert-statement), [`-O` removes assert statements](https://docs.python.org/3/using/cmdline.html#cmdoption-O)

---

# 🟠 HIGH — Correctness, Resource Safety, and Hidden State Bugs

---

## 7. Bare `except` / Silencing Exceptions

**Primary impact: Correctness + Resource Safety — hides unexpected failures and catches too much**

```python
# ❌ BAD — catches BaseException, including KeyboardInterrupt and SystemExit
try:
    result = process_packet(data)
except:
    pass
```

```python
# ✅ GOOD — catch only the failures you are prepared to handle
import logging

try:
    result = process_packet(data)
except (ValueError, KeyError) as exc:
    logging.warning("Packet processing failed: %s", exc)
    result = None
```

```python
# ✅ GOOD — top-level logging may catch broadly, then re-raise
try:
    main()
except Exception:
    logging.exception("Application error")
    raise
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) | O(1) |
| Space | O(1) | O(1) |
| Risk | Hidden failures, broken shutdown, corrupted state | Expected failures handled; unexpected failures propagate |

**When bare `except` is acceptable:**
- Rare top-level cleanup/logging where the exception is re-raised.
- Prefer `except Exception` for “almost everything” in application errors; let `KeyboardInterrupt` and `SystemExit` escape unless you have a top-level reason.

**Official sources:** [Errors and exceptions](https://docs.python.org/3/tutorial/errors.html), [PEP 8 programming recommendations](https://peps.python.org/pep-0008/#programming-recommendations)

---

## 8. Mutable Default Arguments

**Primary impact: Correctness — state is shared across calls**

```python
# ❌ BAD — [] is created once, when the function is defined
def append_to_log(entry, log=[]):
    log.append(entry)
    return log

append_to_log("connect")     # ['connect']
append_to_log("disconnect")  # ['connect', 'disconnect']
```

```python
# ✅ GOOD — create a new list for calls that do not pass one
def append_to_log(entry, log=None):
    if log is None:
        log = []
    log.append(entry)
    return log
```

```python
# ✅ GOOD — use a private sentinel if None is a valid argument
_MISSING = object()

def append_to_log(entry, log=_MISSING):
    if log is _MISSING:
        log = []
    log.append(entry)
    return log
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) append | O(1) check + append |
| Space | Shared state grows across calls | Per-call state unless explicitly passed |
| Risk | Cross-call contamination | Isolated default state |

**When mutable defaults are acceptable:**
- Intentional function-local caches, but prefer `functools.cache` or `functools.lru_cache` when possible.

**Official sources:** [Default argument values](https://docs.python.org/3/tutorial/controlflow.html#default-argument-values), [`functools.cache`](https://docs.python.org/3/library/functools.html#functools.cache)

---

## 9. Not Using Context Managers or `try/finally` for Resources

**Primary impact: Resource Cost + Correctness — leaked handles, sockets, locks, or transactions**

```python
# ❌ BAD — close() is skipped if readline() raises
f = open("packets.log", "r", encoding="utf-8")
data = f.readline()
f.close()

# ❌ BAD — release() is skipped if the body raises
lock.acquire()
shared_state["count"] += 1
lock.release()
```

```python
# ✅ GOOD — __exit__ runs when leaving the block, including on exceptions
with open("packets.log", "r", encoding="utf-8") as f:
    data = f.readline()

# ✅ GOOD — lock is released when the block exits
with lock:
    shared_state["count"] += 1
```

```python
# ✅ ALSO CORRECT — explicit try/finally when no context manager exists
resource = acquire_resource()
try:
    use(resource)
finally:
    resource.close()
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) | O(1) |
| Space/resource cost | Can leak OS resources | Resource lifetime is bounded |
| Risk | Exhausted file descriptors, deadlocks, unflushed data | Deterministic cleanup |

**When manual close/release is acceptable:**
- When protected by `try/finally`.
- When object lifetime intentionally spans a larger scope and ownership is explicit.

**Official sources:** [`with` statement](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement), [`contextlib`](https://docs.python.org/3/library/contextlib.html), [`threading.Lock`](https://docs.python.org/3/library/threading.html#lock-objects)

---

## 10. Changing a Collection’s Size While Iterating Over It

**Primary impact: Correctness — skipped items, incomplete iteration, or `RuntimeError`**

```python
# ❌ BAD — removing shifts later elements; matches can be skipped
connections = ["192.168.1.1", "192.168.1.2", "10.0.0.1"]
for conn in connections:
    if conn.startswith("192"):
        connections.remove(conn)

# ❌ BAD — adding/deleting dict entries during view iteration may raise or miss entries
for key in my_dict:
    if key.startswith("tmp_"):
        del my_dict[key]
```

```python
# ✅ GOOD — create a filtered list
connections = [c for c in connections if not c.startswith("192")]

# ✅ GOOD — mutate the same list object by assigning to a slice
connections[:] = [c for c in connections if not c.startswith("192")]

# ✅ GOOD — collect keys first
for key in [k for k in my_dict if k.startswith("tmp_")]:
    del my_dict[key]

# ✅ GOOD — rebuild the dict
my_dict = {k: v for k, v in my_dict.items() if not k.startswith("tmp_")}
```

| Metric | Bad | Good |
|---|---|---|
| Time | Often O(n²) for repeated `list.remove` | O(n) filtering |
| Space | O(1) extra but wrong | O(n) for new collection |
| Risk | Skipped elements or runtime error | Stable iteration source |

**When mutation during iteration is acceptable:**
- Replacing existing list elements by index without changing length: `for i in range(len(buf)): buf[i] ^= 0xFF`.
- Mutating objects contained in a collection is different from changing the collection’s size/keys.

**Official sources:** [Looping over copies/new collections](https://docs.python.org/3/tutorial/controlflow.html#for-statements), [Dictionary view iteration warning](https://docs.python.org/3/library/stdtypes.html#dictionary-view-objects)

---

## 11. `is` Instead of `==` for Value Comparison

**Primary impact: Correctness — identity is not equality**

```python
# ❌ BAD — is checks object identity, not value equality
status_code = int("200")
if status_code is 200:
    allow()

packet_type = "".join(["S", "Y", "N"])
if packet_type is "SYN":
    handle_syn()
```

```python
# ✅ GOOD — == checks equality
if status_code == 200:
    allow()

if packet_type == "SYN":
    handle_syn()

# ✅ GOOD — is is correct for singletons
if result is None:
    return
```

```python
# ✅ GOOD — boolean flags usually need truthiness, not comparison to True/False
if connection.monitor:
    start_monitoring()

if not connection.monitor:
    return
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) identity check | Type-dependent equality; often O(1), strings O(n) worst case |
| Space | O(1) | O(1) |
| Risk | Accidentally works for some cached/interned objects | Correct semantic comparison |

**When `is` is correct:**
- Singletons such as `None` and `NotImplemented`.
- Identity checks where object identity is the actual question.

**Official sources:** [Comparisons](https://docs.python.org/3/reference/expressions.html#comparisons), [PEP 8 singleton comparisons](https://peps.python.org/pep-0008/#programming-recommendations)

---

## 12. `type(x) == T` Instead of `isinstance(x, T)`

**Primary impact: Correctness — rejects subclasses and virtual subclasses**

```python
# ❌ BAD — exact type only
class MyBytes(bytes):
    pass

def send(payload):
    if type(payload) == bytes:
        socket.send(payload)

send(MyBytes(b"data"))  # rejected
```

```python
# ✅ GOOD — accepts subclasses
class MyBytes(bytes):
    pass

def send(payload):
    if isinstance(payload, (bytes, bytearray, memoryview)):
        socket.send(payload)
```

```python
# ✅ GOOD — often better: rely on behavior, not type
from pathlib import Path

def read_text(path):
    return Path(path).read_text(encoding="utf-8")
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) exact type check | Usually O(1); depends on MRO/ABC machinery |
| Space | O(1) | O(1) |
| Risk | Breaks polymorphism | Respects inheritance and interfaces |

**When exact `type()` checks are acceptable:**
- When subclasses must be rejected on purpose, for example strict serialization rules. Document that reason.

**Official sources:** [`isinstance()`](https://docs.python.org/3/library/functions.html#isinstance), [`collections.abc`](https://docs.python.org/3/library/collections.abc.html)

---

## 13. Late-Binding Closures in Loops

**Primary impact: Correctness — every closure sees the final loop variable value**

```python
# ❌ BAD — all lambdas close over the same variable x
funcs = []
for x in range(5):
    funcs.append(lambda: x * x)

print(funcs[0]())  # 16, not 0
print(funcs[2]())  # 16, not 4
```

```python
# ✅ GOOD — bind the current value as a default argument
funcs = []
for x in range(5):
    funcs.append(lambda x=x: x * x)
```

```python
# ✅ GOOD — or use a function factory
def make_square(x):
    def square():
        return x * x
    return square

funcs = [make_square(x) for x in range(5)]
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) closure creation | O(n) closure creation |
| Space | O(n) closures | O(n) closures |
| Risk | All callbacks use the final value | Each callback keeps the intended value |

**When late binding is acceptable:**
- When the closure intentionally reads the current/live value at call time.

**Official source:** [Python FAQ: lambdas in loops](https://docs.python.org/3/faq/programming.html#why-do-lambdas-defined-in-a-loop-with-different-values-all-return-the-same-result)

---

## 14. Shared Mutable State Without a Lock

**Primary impact: Correctness — compound operations are not transactions**

```python
# ❌ BAD — read/modify/write on shared state without synchronization
counter = 0

def worker():
    global counter
    counter += 1
```

```python
# ✅ GOOD — protect the invariant with a lock
import threading

counter = 0
counter_lock = threading.Lock()

def worker():
    global counter
    with counter_lock:
        counter += 1
```

```python
# ✅ GOOD — use synchronized queues for producer/consumer handoff
from queue import Queue

jobs = Queue()
jobs.put("packet-1")
job = jobs.get()
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) per operation, but race-prone | O(1) plus lock contention |
| Space | O(1) | O(1) for lock/queue object |
| Risk | Lost updates or inconsistent invariants | One thread updates protected state at a time |

**Important nuance:**
- Some individual built-in operations have documented thread-safety guarantees in current Python builds, but a multi-step check/update sequence is still not a transaction.
- If correctness depends on a shared invariant, protect the whole invariant with a lock or use a synchronized data structure.

**Official sources:** [`threading.Lock`](https://docs.python.org/3/library/threading.html#lock-objects), [`queue.Queue`](https://docs.python.org/3/library/queue.html), [thread-safety guarantees](https://docs.python.org/3/library/threadsafety.html)

---

## 15. LBYL Dict Access and Ambiguous `.get()` Defaults

**Primary impact: Correctness + Speed — duplicate lookups and ambiguous missing values**

```python
# ❌ BAD — two lookups when the key exists
if c_id in connections:
    connection = connections[c_id]
    send(connection)
else:
    create_connection(c_id)

# ❌ BAD — ambiguous if None is a real stored value
connection = connections.get(c_id)
if connection is None:
    create_connection(c_id)
```

```python
# ✅ GOOD — one lookup with an explicit sentinel
_MISSING = object()

connection = connections.get(c_id, _MISSING)
if connection is _MISSING:
    create_connection(c_id)
else:
    send(connection)
```

```python
# ✅ GOOD — EAFP when absence is exceptional
try:
    connection = connections[c_id]
except KeyError:
    create_connection(c_id)
else:
    send(connection)
```

| Metric | Bad | Good |
|---|---|---|
| Time | Two average O(1) lookups when present | One average O(1) lookup |
| Space | O(1) | O(1) |
| Risk | Ambiguous `None`; race-prone if shared without a lock | Clear missing-value handling |

**When LBYL is acceptable:**
- Small, non-shared code where readability is better and `None` ambiguity is not present.
- If the dict is shared across threads and the operation must be atomic as a whole, use a lock around the whole check/update.

**Official sources:** [`dict.get()`](https://docs.python.org/3/library/stdtypes.html#dict.get), [thread-safety guarantees](https://docs.python.org/3/library/threadsafety.html)

---

# 🟡 MEDIUM — Performance and Resource Cost at Scale

---

## 16. String Concatenation in a Loop

**Primary impact: Speed + Resource Cost — fragile performance and many temporary strings**

```python
# ❌ BAD — repeatedly creates larger strings
log_output = ""
for entry in log_entries:
    log_output += entry + "\n"
```

```python
# ✅ GOOD — collect parts and join once
log_output = "\n".join(log_entries)

# ✅ GOOD — preserve final newline if needed
log_output = "".join(f"{entry}\n" for entry in log_entries)
```

```python
# ✅ GOOD — streaming-style assembly
from io import StringIO

buf = StringIO()
for entry in log_entries:
    buf.write(entry)
    buf.write("\n")
log_output = buf.getvalue()
```

| Metric | Bad | Good |
|---|---|---|
| Time | Can become O(total_chars²); CPython has fragile optimizations | O(total_chars) across implementations |
| Space | Many transient strings; peak depends on implementation | Final string plus parts/buffer |
| Risk | Slow at scale or across implementations | Portable linear assembly |

**When `+` is acceptable:**
- A small fixed number of strings: `full_name = first + " " + last`.
- Non-hot code where readability is better and size is bounded.

**Official sources:** [PEP 8 string concatenation note](https://peps.python.org/pep-0008/#programming-recommendations), [`str.join()`](https://docs.python.org/3/library/stdtypes.html#str.join), [`io.StringIO`](https://docs.python.org/3/library/io.html#io.StringIO)

---

## 17. `list` for Repeated Membership Testing

**Primary impact: Speed — linear scans repeated many times**

```python
# ❌ BAD — list membership scans until a match is found or the list ends
BLOCKED_IPS = ["192.168.1.5", "10.0.0.2", "172.16.0.99"]

if src_ip in BLOCKED_IPS:
    drop(packet)
```

```python
# ✅ GOOD — set membership uses hashing
BLOCKED_IPS = {"192.168.1.5", "10.0.0.2", "172.16.0.99"}

if src_ip in BLOCKED_IPS:
    drop(packet)
```

| Metric | Bad (`list`) | Good (`set`) |
|---|---|---|
| Time per lookup | O(n) | Average O(1), worst-case collision behavior can degrade |
| Build time | O(n) | O(n) |
| Space | O(n) | O(n), usually higher constant factor |

**When a list is correct for `in`:**
- Tiny collections where order matters and the code is not hot.
- When duplicates matter. Sets remove duplicates.

**Official sources:** [Dictionary hashing design FAQ](https://docs.python.org/3/faq/design.html#how-are-dictionaries-implemented-in-cpython), [sets](https://docs.python.org/3/library/stdtypes.html#set-types-set-frozenset)

---

## 18. Using `list.pop(0)` as a Queue

**Primary impact: Speed — removing from the front shifts all later elements**

```python
# ❌ BAD — every pop(0) shifts the rest of the list
queue = []
queue.append("job-1")
queue.append("job-2")

while queue:
    job = queue.pop(0)
    process(job)
```

```python
# ✅ GOOD — deque is designed for fast appends/pops at both ends
from collections import deque

queue = deque()
queue.append("job-1")
queue.append("job-2")

while queue:
    job = queue.popleft()
    process(job)
```

| Metric | Bad (`list.pop(0)`) | Good (`deque.popleft`) |
|---|---|---|
| Time per dequeue | O(n) shift | O(1) typical |
| Space | O(n) | O(n) |
| Risk | Slow queues at scale | Queue operation matches data structure |

**When `list.pop(0)` is acceptable:**
- Tiny queues where simplicity matters more than scaling.

**Official source:** [Using lists as queues](https://docs.python.org/3/tutorial/datastructures.html#using-lists-as-queues)

---

## 19. Loading All Data When Streaming Works

**Primary impact: Resource Cost — O(n) memory when one item at a time is enough**

```python
# ❌ BAD — reads every line into memory before processing
with open("packets.log", "r", encoding="utf-8") as f:
    lines = f.readlines()

for line in lines:
    process(line)
```

```python
# ✅ GOOD — file objects are iterable; process one line at a time
with open("packets.log", "r", encoding="utf-8") as f:
    for line in f:
        process(line)
```

```python
# ✅ GOOD — generator for custom streaming

def stream_packets(path):
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(1500), b""):
            yield chunk

for packet in stream_packets("capture.pcap"):
    process(packet)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(n) | O(1) or bounded chunk size |
| Risk | Memory spikes, delayed first item | Low memory, immediate processing |

**When loading everything is acceptable:**
- You need random access, multiple passes, sorting, or the dataset is known to be small.

**Official sources:** [File iteration](https://docs.python.org/3/tutorial/inputoutput.html#methods-of-file-objects), [Generator expressions](https://docs.python.org/3/howto/functional.html#generator-expressions-and-list-comprehensions)

---

## 20. Building Temporary Lists for `any`, `all`, `sum`, `min`, or `max`

**Primary impact: Resource Cost + Speed — unnecessary materialization**

```python
# ❌ BAD — builds a full list before any() can answer
if any([packet.is_blocked() for packet in packets]):
    drop_batch()

# ❌ BAD — materializes all sizes
largest = max([len(packet.payload) for packet in packets])
```

```python
# ✅ GOOD — generator expression computes values as needed
if any(packet.is_blocked() for packet in packets):
    drop_batch()

largest = max(len(packet.payload) for packet in packets)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n), but no short-circuit until list is built | O(n), with short-circuit for `any`/`all` |
| Space | O(n) temporary list | O(1) iterator state |
| Risk | Memory waste | Streams values |

**When a list comprehension is acceptable:**
- You need the list later.
- The result is small and list readability is better.

**Official sources:** [`any()` / `all()`](https://docs.python.org/3/library/functions.html#any), [Generator expressions](https://docs.python.org/3/howto/functional.html#generator-expressions-and-list-comprehensions)

---

## 21. `map(lambda ...)` Instead of a Comprehension

**Primary impact: Cleanliness + sometimes Speed — less readable inline transformation**

```python
# ❌ BAD — lambda hides a simple expression
seq_numbers = [100, 200, 300, 400]
wrapped = list(map(lambda seq: (seq + 1) & 0xFFFFFFFF, seq_numbers))
```

```python
# ✅ GOOD — direct expression
wrapped = [(seq + 1) & 0xFFFFFFFF for seq in seq_numbers]

# ✅ GOOD — keep it lazy if you only iterate once
wrapped_iter = ((seq + 1) & 0xFFFFFFFF for seq in seq_numbers)
```

```python
# ✅ GOOD — map is fine with an existing named function
words = ["syn", "ack", "fin"]
upper = list(map(str.upper, words))
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n); lambda call per item | O(n); usually clearer for inline expressions |
| Space | O(n) after `list()`; lazy before `list()` | O(n) list or O(1) generator |
| Risk | Readability cost; performance varies | Clear expression of transformation |

**When `map()` is acceptable:**
- Existing named functions, especially C-level methods such as `str.upper`.
- Lazy pipelines where `map` improves clarity.

**Official source:** [`map()`](https://docs.python.org/3/library/functions.html#map)

---

## 22. Sorting Everything to Get One Item or a Small Top-K

**Primary impact: Speed + Resource Cost — full sort does more work than needed**

```python
# ❌ BAD — sorts the entire dataset to get one item
youngest = sorted(users, key=lambda u: u.age)[0]

# ❌ BAD — sorts everything to get top 10
top_10 = sorted(scores, reverse=True)[:10]
```

```python
# ✅ GOOD — one item
youngest = min(users, key=lambda u: u.age)

# ✅ GOOD — small top-k
import heapq

top_10 = heapq.nlargest(10, scores)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n log n) full sort | O(n) for `min`/`max`; top-k avoids full sort for small k |
| Space | O(n) sorted list | O(1) for `min`/`max`; O(k) typical heap storage |
| Risk | Extra CPU/memory | Tool matches the question |

**When sorting everything is acceptable:**
- You need the full sorted order.
- `k` is large; the `heapq` docs say `nlargest`/`nsmallest` perform best for smaller `n`, and sorting may be better for larger values.

**Official sources:** [`min()`](https://docs.python.org/3/library/functions.html#min), [`max()`](https://docs.python.org/3/library/functions.html#max), [`heapq.nlargest()` / `nsmallest()`](https://docs.python.org/3/library/heapq.html#heapq.nlargest)

---

## 23. Repeated Attribute or Method Lookup in a Very Hot Loop

**Primary impact: Speed — constant-factor overhead in tight loops**

```python
# ❌ BAD in genuinely hot loops — repeated lookup every iteration
class FakeTcpInjector:
    def flush_all(self, packets):
        for pkt in packets:
            conn = self.connections.get(pkt.id)
            self.w.send(pkt, False)
```

```python
# ✅ GOOD in genuinely hot loops — hoist stable lookups
class FakeTcpInjector:
    def flush_all(self, packets):
        connections = self.connections
        send = self.w.send
        for pkt in packets:
            conn = connections.get(pkt.id)
            send(pkt, False)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n), higher constant factor | O(n), lower constant factor if attributes are stable |
| Space | O(1) | O(1) local bindings |
| Risk | Slower in hot loops | Faster but can hide dynamic attribute changes |

**When hoisting is unnecessary or wrong:**
- Small loops.
- When attributes are properties/descriptors with intentional dynamic behavior.
- When another part of the program may change `self.w` or `self.connections` during the loop.
- Modern CPython specializes many operations, so profile before applying this.

**Official source:** [PEP 659: specializing adaptive interpreter](https://peps.python.org/pep-0659/)

---

# 🟢 LOW — Readability and Maintainability

---

## 24. Wildcard Imports

**Primary impact: Cleanliness + Correctness — unclear namespace and possible name collisions**

```python
# ❌ BAD — unclear where names come from
from os.path import *
from pathlib import *

path = join("a", "b")
```

```python
# ✅ GOOD — explicit imports
from os.path import join, exists
from pathlib import Path

path = join("a", "b")
home = Path.home()
```

```python
# ✅ GOOD — module qualification is often clearest
import os.path
from pathlib import Path

path = os.path.join("a", "b")
home = Path.home()
```

| Metric | Bad | Good |
|---|---|---|
| Time | Import-time namespace work; usually not the real issue | Usually similar |
| Space | More names in local namespace | Fewer names in local namespace |
| Risk | Reader/tool confusion, accidental shadowing | Clear source of each name |

**When wildcard imports are acceptable:**
- A module deliberately re-exporting a public API, especially when `__all__` controls what is public.
- Type stub files or generated API surfaces where this is intentional.

**Official sources:** [PEP 8 wildcard imports](https://peps.python.org/pep-0008/#imports), [`__all__` and import `*`](https://docs.python.org/3/reference/simple_stmts.html#the-import-statement)

---

## 25. `range(len(x))` When You Do Not Need the Index

**Primary impact: Cleanliness — manual indexing hides the data being processed**

```python
# ❌ BAD — index is only used to fetch the item
packets = [pkt1, pkt2, pkt3]
for i in range(len(packets)):
    process(packets[i])
```

```python
# ✅ GOOD — iterate over values directly
for packet in packets:
    process(packet)
```

```python
# ✅ GOOD — use enumerate when you need index and value
for i, packet in enumerate(packets):
    print(i, packet)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(1) | O(1) |
| Risk | Noise and indexing mistakes | Intent is direct |

**When `range(len(x))` is correct:**
- In-place assignment by index: `for i in range(len(buf)): buf[i] ^= 0xFF`.
- Algorithms that genuinely need numeric positions.

**Official sources:** [`range()`](https://docs.python.org/3/tutorial/controlflow.html#the-range-function), [`enumerate()`](https://docs.python.org/3/library/functions.html#enumerate)

---

## 26. Manual Counter Instead of `enumerate`

**Primary impact: Cleanliness — extra mutable state**

```python
# ❌ BAD — manual counter
index = 0
for connection in active_connections:
    print(f"Connection #{index}: {connection}")
    index += 1
```

```python
# ✅ GOOD — equivalent behavior; enumerate starts at 0 by default
for index, connection in enumerate(active_connections):
    print(f"Connection #{index}: {connection}")
```

```python
# ✅ GOOD — use start=1 only when human numbering should start at 1
for number, connection in enumerate(active_connections, start=1):
    print(f"Connection #{number}: {connection}")
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(1) | O(1) |
| Risk | Off-by-one and forgotten increment | Built-in counter/value pairing |

**When a manual counter is acceptable:**
- Conditional counters: `if condition: count += 1`.
- Multiple counters with different increment rules.

**Official source:** [`enumerate()`](https://docs.python.org/3/library/functions.html#enumerate)

---

## 27. Index-Based Parallel Iteration Instead of `zip`

**Primary impact: Cleanliness + Correctness — unnecessary indexing and hidden length assumptions**

```python
# ❌ BAD — index used only to access both lists
src_ips = ["192.168.1.1", "10.0.0.1"]
dst_ips = ["8.8.8.8", "1.1.1.1"]

for i in range(len(src_ips)):
    print(f"{src_ips[i]} -> {dst_ips[i]}")
```

```python
# ✅ GOOD — pair values directly
for src, dst in zip(src_ips, dst_ips):
    print(f"{src} -> {dst}")
```

```python
# ✅ GOOD — Python 3.10+: fail fast if lengths must match
for src, dst in zip(src_ips, dst_ips, strict=True):
    print(f"{src} -> {dst}")
```

```python
# ✅ GOOD — keep all items when missing values are valid
from itertools import zip_longest

for src, dst in zip_longest(src_ips, dst_ips, fillvalue="N/A"):
    print(f"{src} -> {dst}")
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(1) | O(1) |
| Risk | Index errors or hidden truncation assumptions | Direct pairing; `strict=True` can validate length |

**When index-based parallel iteration is acceptable:**
- You need positions for slicing, assignment, or index-specific logic.
- Array libraries where vectorized/indexed operations are the intended API.

**Official sources:** [`zip()`](https://docs.python.org/3/library/functions.html#zip), [`itertools.zip_longest()`](https://docs.python.org/3/library/itertools.html#itertools.zip_longest), [Python 3.10 `zip(strict=True)`](https://docs.python.org/3/whatsnew/3.10.html#pep-618-add-optional-length-checking-to-zip)

---

## 28. `not x == y` Instead of `x != y`

**Primary impact: Cleanliness — indirect expression of inequality**

```python
# ❌ BAD — reader has to mentally negate equality
if not packet.tcp.seq_num == expected_seq:
    handle_mismatch()
```

```python
# ✅ GOOD — direct inequality
if packet.tcp.seq_num != expected_seq:
    handle_mismatch()
```

| Metric | Bad | Good |
|---|---|---|
| Time | Usually equivalent in normal code | Usually equivalent in normal code |
| Space | O(1) | O(1) |
| Risk | Less readable; may differ for exotic custom comparisons | Direct intent |

**Important nuance:**
- `not x == y` is parsed as `not (x == y)`.
- `x != y` calls inequality behavior. Custom classes can make `__eq__` and `__ne__` differ, so these are not guaranteed identical for every object.

**When `not x == y` is acceptable:**
- Rare code intentionally relying on `__eq__` and boolean negation instead of `__ne__`.

**Official sources:** [Operator precedence and `not`](https://docs.python.org/3/library/stdtypes.html#boolean-operations-and-or-not), [Rich comparison methods](https://docs.python.org/3/reference/datamodel.html#object.__eq__)

---

## 29. Comparing Boolean Values to `True` or `False`

**Primary impact: Cleanliness — noisy and sometimes subtly wrong**

```python
# ❌ BAD — boolean flags do not need equality checks
if connection.monitor == True:
    start_monitoring()

if connection.monitor == False:
    return
```

```python
# ✅ GOOD — use truthiness for boolean flags
if connection.monitor:
    start_monitoring()

if not connection.monitor:
    return
```

```python
# ✅ GOOD — if exact bool type really matters, say so explicitly
if isinstance(connection.monitor, bool) and connection.monitor:
    start_monitoring()
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) | O(1) |
| Space | O(1) | O(1) |
| Risk | Noise; may hide non-bool truthy/falsy values | Clear boolean intent |

**When comparison is acceptable:**
- Almost never for normal boolean flags.
- Exact identity checks like `value is True` are only for cases where you truly require the singleton `True`, not just truthiness.

**Official source:** [PEP 8 boolean comparisons](https://peps.python.org/pep-0008/#programming-recommendations)

---

# Summary Table

| # | Pattern | Severity | Primary Impact | Better Rule |
|---:|---|---|---|---|
| 1 | `eval()` / `exec()` on external input | 🔴 CRITICAL | Security | Parse data; allowlist operations |
| 2 | Unpickling untrusted data | 🔴 CRITICAL | Security | Use data formats and schemas |
| 3 | SQL string formatting | 🔴 CRITICAL | Security | Bind values with placeholders |
| 4 | `subprocess(shell=True)` with external input | 🔴 CRITICAL | Security | Pass argv lists; avoid shell parsing |
| 5 | `random` for secrets | 🔴 CRITICAL | Security | Use `secrets` |
| 6 | `assert` for validation/security | 🔴 CRITICAL | Security + Correctness | Use explicit `if` + exception |
| 7 | Bare `except` / silent pass | 🟠 HIGH | Correctness | Catch specific exceptions; re-raise unexpected failures |
| 8 | Mutable default arguments | 🟠 HIGH | Correctness | Use `None` or a sentinel |
| 9 | No context manager / no `try/finally` | 🟠 HIGH | Resource Cost | Use `with` or `try/finally` |
| 10 | Changing collection size while iterating | 🟠 HIGH | Correctness | Iterate over copy or build a new collection |
| 11 | `is` for value equality | 🟠 HIGH | Correctness | Use `==`; reserve `is` for identity/singletons |
| 12 | `type(x) == T` | 🟠 HIGH | Correctness | Use `isinstance()` or duck typing |
| 13 | Late-binding closures in loops | 🟠 HIGH | Correctness | Bind current value or use a factory |
| 14 | Shared mutable state without lock | 🟠 HIGH | Correctness | Protect invariants with locks/queues |
| 15 | LBYL dict double lookup / ambiguous `.get()` | 🟠 HIGH | Correctness + Speed | Use sentinel, EAFP, or a lock for shared state |
| 16 | String `+` in loop | 🟡 MEDIUM | Speed + Resource Cost | Use `join()` or `StringIO` |
| 17 | `list` for repeated membership | 🟡 MEDIUM | Speed | Use `set` or `dict` |
| 18 | `list.pop(0)` as queue | 🟡 MEDIUM | Speed | Use `collections.deque` |
| 19 | Loading all data when streaming works | 🟡 MEDIUM | Resource Cost | Iterate/generate one item at a time |
| 20 | Temporary lists for `any`/`all`/aggregates | 🟡 MEDIUM | Resource Cost | Use generator expressions |
| 21 | `map(lambda ...)` for simple expressions | 🟡 MEDIUM | Cleanliness | Use comprehensions; keep `map` for named functions |
| 22 | Sorting everything for one item/top-k | 🟡 MEDIUM | Speed + Resource Cost | Use `min`, `max`, or `heapq` |
| 23 | Attribute lookup in hot loop | 🟡 MEDIUM | Speed | Hoist stable lookups only after profiling |
| 24 | Wildcard imports | 🟢 LOW | Cleanliness | Use explicit imports or module qualification |
| 25 | `range(len(x))` without needing index | 🟢 LOW | Cleanliness | Iterate directly |
| 26 | Manual counter | 🟢 LOW | Cleanliness | Use `enumerate()` |
| 27 | Index-based parallel iteration | 🟢 LOW | Cleanliness + Correctness | Use `zip`, `strict=True`, or `zip_longest` |
| 28 | `not x == y` | 🟢 LOW | Cleanliness | Use `x != y` |
| 29 | Comparing booleans to `True`/`False` | 🟢 LOW | Cleanliness | Use `if flag:` / `if not flag:` |

---

# Official Reference Set

- [Python built-in functions](https://docs.python.org/3/library/functions.html)
- [Python language reference: simple statements](https://docs.python.org/3/reference/simple_stmts.html)
- [Python language reference: compound statements](https://docs.python.org/3/reference/compound_stmts.html)
- [Python language reference: expressions](https://docs.python.org/3/reference/expressions.html)
- [Python data model](https://docs.python.org/3/reference/datamodel.html)
- [Python tutorial: control flow](https://docs.python.org/3/tutorial/controlflow.html)
- [Python tutorial: data structures](https://docs.python.org/3/tutorial/datastructures.html)
- [Python tutorial: input and output](https://docs.python.org/3/tutorial/inputoutput.html)
- [Python errors and exceptions](https://docs.python.org/3/tutorial/errors.html)
- [Python standard types](https://docs.python.org/3/library/stdtypes.html)
- [PEP 8 — Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [PEP 20 — The Zen of Python](https://peps.python.org/pep-0020/)
- [`ast`](https://docs.python.org/3/library/ast.html)
- [`json`](https://docs.python.org/3/library/json.html)
- [`pickle`](https://docs.python.org/3/library/pickle.html)
- [`sqlite3`](https://docs.python.org/3/library/sqlite3.html)
- [`subprocess`](https://docs.python.org/3/library/subprocess.html)
- [`random`](https://docs.python.org/3/library/random.html)
- [`secrets`](https://docs.python.org/3/library/secrets.html)
- [`contextlib`](https://docs.python.org/3/library/contextlib.html)
- [`threading`](https://docs.python.org/3/library/threading.html)
- [`queue`](https://docs.python.org/3/library/queue.html)
- [`collections.deque`](https://docs.python.org/3/library/collections.html#collections.deque)
- [`heapq`](https://docs.python.org/3/library/heapq.html)
- [`itertools`](https://docs.python.org/3/library/itertools.html)
- [`io`](https://docs.python.org/3/library/io.html)
- [`functools`](https://docs.python.org/3/library/functools.html)
