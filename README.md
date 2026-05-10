# The Pythonic Code Field Guide
### Every mistake ranked by damage — with Big O proof

Sorted from **most dangerous** to **benign**. Each entry includes:
- The bad pattern with an inline comment explaining the flaw
- The Pythonic fix
- Primary impact category: **Safety**, **Correctness**, **Speed**, or **Cleanliness**
- Time and space complexity for both versions
- Edge cases where the "bad" version is actually correct

---

## Severity Legend

| Level | Label | Meaning |
|---|---|---|
| 🔴 | **CRITICAL** | Can cause security vulnerabilities, data corruption, or silent data loss |
| 🟠 | **HIGH** | Correctness bugs, race conditions, resource leaks |
| 🟡 | **MEDIUM** | Meaningful performance degradation at scale |
| 🟢 | **LOW** | Readability and style; no runtime impact |

---

---

# 🔴 CRITICAL — Must Never Appear in Production

---

## 1. Bare `except` / Silencing All Exceptions

**Primary impact: Safety — hides 100% of unexpected failures including KeyboardInterrupt, SystemExit, MemoryError**

```python
# ❌ BAD — catches EVERYTHING including ctrl+c, OOM, interpreter exit
try:
    result = process_packet(data)
except:          # no type = catches BaseException; program can never be killed cleanly
    pass         # silent pass = bugs disappear forever
```

```python
# ✅ GOOD — catch only what you expect; log everything else
import logging

try:
    result = process_packet(data)
except (ValueError, KeyError) as e:
    logging.warning("Packet processing failed: %s", e)
    result = None
```

| Metric | Bad | Good |
|---|---|---|
| Time complexity | O(1) | O(1) |
| Space complexity | O(1) | O(1) |
| Risk | Swallows `SystemExit`, `KeyboardInterrupt`, `MemoryError` | Only catches declared exceptions |

**When bare `except` is acceptable:**
- Top-level crash reporters: `except BaseException as exc: log_and_reraise(exc)` — only to log before re-raising, never to suppress.

---

## 2. Mutable Default Arguments

**Primary impact: Safety — silent state mutation shared across ALL callers; produces impossible-to-debug bugs**

```python
# ❌ BAD — default list created ONCE at function definition, shared forever
def append_to_log(entry, log=[]):     # this [] lives for the lifetime of the process
    log.append(entry)
    return log

append_to_log("connect")   # ["connect"]
append_to_log("disconnect") # ["connect", "disconnect"] ← contaminated!
```

```python
# ✅ GOOD — sentinel pattern; fresh list created on every call
def append_to_log(entry, log=None):
    if log is None:
        log = []
    log.append(entry)
    return log
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) | O(1) |
| Space | O(n) accumulated across ALL calls | O(n) per call, isolated |
| Risk | Cross-call state pollution | No shared state |

**When mutable defaults are acceptable:**
- Intentional caching / memoization: `def cached(key, _cache={})` — a known Python idiom for function-level caches, but should be replaced with `functools.lru_cache` in modern code.

---

## 3. `eval()` / `exec()` on External Input

**Primary impact: Safety — arbitrary code execution; complete system compromise**

```python
# ❌ BAD — user controls what runs on your machine
user_input = input("Enter filter expression: ")
result = eval(user_input)     # "import os; os.system('rm -rf /')" is valid input

config = json.loads(raw)
exec(config["setup_code"])    # remote code execution via config file
```

```python
# ✅ GOOD — parse structure, never execute strings
import ast
import operator

# For math expressions: use ast.literal_eval (only literals, no calls)
safe_value = ast.literal_eval("{'key': [1, 2, 3]}")

# For filter logic: use an allowlist of operations
ALLOWED_OPS = {"+": operator.add, "-": operator.sub}
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) parse + O(∞) execution | O(n) parse only |
| Space | O(∞) | O(n) |
| Risk | Arbitrary code execution | Input never executed |

**When `eval` is acceptable:**
- Trusted, sandboxed REPL tools where the caller is the developer (e.g., a local debug shell). Still prefer `ast.literal_eval` for data.

---

## 4. Wildcard Imports

**Primary impact: Safety — namespace collisions silently override built-ins and your own functions**

```python
# ❌ BAD — imports hundreds of names; two modules may define the same name
from os.path import *     # imports: join, split, exists, ...
from posixpath import *   # silently overwrites join, split, exists with different impls

join("a", "b")            # which join? undefined without reading both modules
```

```python
# ✅ GOOD — explicit is better than implicit (PEP 20)
from os.path import join, exists, dirname
from pydivert import Packet, WinDivert
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) names imported | O(k) where k = what you need |
| Space (RAM) | O(n) objects loaded into namespace | O(k) |
| Risk | Silent name collisions, breaks `grep`, breaks IDEs | No collisions |

**When wildcard imports are acceptable:**
- `from typing import *` in type stub files (`.pyi`). Never in runtime code.

---

---

# 🟠 HIGH — Correctness and Resource Safety

---

## 5. Modifying a Collection While Iterating Over It

**Primary impact: Correctness — silently skips elements or raises `RuntimeError`; behaviour is undefined**

```python
# ❌ BAD — iterator advances while list shrinks; every other match is silently skipped
connections = ["192.168.1.1", "10.0.0.1", "172.16.0.1"]
for conn in connections:
    if conn.startswith("192"):
        connections.remove(conn)    # shifts indices; next element skipped

# ❌ BAD (dict) — raises RuntimeError: dictionary changed size during iteration
for key in my_dict:
    if key.startswith("tmp_"):
        del my_dict[key]
```

```python
# ✅ GOOD — filter to a new list; original untouched during iteration
connections = [c for c in connections if not c.startswith("192")]

# ✅ GOOD (dict) — collect keys first, delete after
stale = [k for k in my_dict if k.startswith("tmp_")]
for key in stale:
    del my_dict[key]

# ✅ BEST (dict, Python 3.9+)
my_dict = {k: v for k, v in my_dict.items() if not k.startswith("tmp_")}
```

| Metric | Bad | Good (comprehension) |
|---|---|---|
| Time | O(n²) — each `remove` is O(n) scan | O(n) |
| Space | O(1) extra (in-place, wrong) | O(n) for new collection |
| Correctness | Silently wrong | Correct |

**When modifying during iteration is acceptable:**
- Never. Use a copy, a comprehension, or `itertools.filterfalse`.

---

## 6. `is` Instead of `==` for Value Comparison

**Primary impact: Correctness — works by accident for small integers and interned strings; silently fails for anything else**

```python
# ❌ BAD — `is` checks object identity (same memory address), not value equality
status_code = 200
if status_code is 200:      # True in CPython for -5..256 (cached integers) — accidental
    ...

packet_type = "SYN"
if packet_type is "SYN":    # may be False for dynamically built strings — undefined
    ...
```

```python
# ✅ GOOD — `==` checks value equality, always correct
if status_code == 200:
    ...
if packet_type == "SYN":
    ...

# ✅ `is` is correct ONLY for None, True, False singletons
if result is None:
    ...
if flag is True:
    ...
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) — pointer comparison | O(n) for strings (character comparison), O(1) for int |
| Space | O(1) | O(1) |
| Correctness | Undefined outside CPython internals | Always correct |

**When `is` is the right choice:**
- Comparing to `None`, `True`, `False` — PEP 8 mandates `is None`, never `== None`.

---

## 7. Not Using Context Managers for Resources

**Primary impact: Safety — file handles, sockets, and locks leak on exceptions; causes OS-level resource exhaustion**

```python
# ❌ BAD — if readline() raises, file handle leaks forever
f = open("packets.log", "r")
data = f.readline()
f.close()     # never reached if exception occurs above

# ❌ BAD — lock never released if body raises; deadlock on next acquire
lock.acquire()
shared_state["count"] += 1
lock.release()    # skipped on exception
```

```python
# ✅ GOOD — __exit__ called even on exception; resource always released
with open("packets.log", "r") as f:
    data = f.readline()

# ✅ GOOD — lock always released
with lock:
    shared_state["count"] += 1
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) | O(1) |
| Space (OS handles) | O(n) leaking, limited by OS ulimit | O(1) per block |
| Risk | Resource exhaustion crash | Guaranteed cleanup |

**When manual open/close is acceptable:**
- Never in Python 3. `contextlib.contextmanager` lets you write custom context managers for any resource.

---

## 8. Using `type()` Instead of `isinstance()` for Type Checks

**Primary impact: Correctness — breaks polymorphism, inheritance, duck typing, and ABCs**

```python
# ❌ BAD — rejects subclasses; breaks every OOP pattern
def send(payload):
    if type(payload) == bytes:    # rejects bytearray, memoryview, subclasses of bytes
        socket.send(payload)

class MyBytes(bytes): ...
send(MyBytes(b"data"))    # silently rejected — type is MyBytes, not bytes
```

```python
# ✅ GOOD — accepts subclasses and ABCs; plays well with duck typing
from collections.abc import Buffer

def send(payload):
    if isinstance(payload, (bytes, bytearray, memoryview)):
        socket.send(payload)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) — exact match | O(d) — d = depth of MRO, practically O(1) |
| Space | O(1) | O(1) |
| Correctness | Breaks inheritance | Respects full type hierarchy |

**When `type() ==` is acceptable:**
- When you explicitly need to reject subclasses — e.g., a serializer that must produce exactly `dict`, not `OrderedDict` or `defaultdict`. Rare. Document it clearly.

---

---

# 🟡 MEDIUM — Performance at Scale

---

## 9. String Concatenation in a Loop (`+`)

**Primary impact: Speed — O(n²) time and O(n²) RAM due to string immutability; kills performance at scale**

```python
# ❌ BAD — creates a new string object on every iteration
# n=10,000 lines → ~50 million characters copied across all iterations combined
log_output = ""
for entry in log_entries:              # n iterations
    log_output += entry + "\n"         # new str object each time: O(1+2+3+...+n) = O(n²)
```

```python
# ✅ GOOD — collect parts, join once at the end
log_output = "\n".join(log_entries)    # O(n) time, O(n) space

# ✅ GOOD for complex assembly
parts = []
for entry in log_entries:
    parts.append(entry)
log_output = "\n".join(parts)          # join is O(n), single allocation
```

| Metric | Bad (`+` loop) | Good (`join`) |
|---|---|---|
| Time | **O(n²)** | **O(n)** |
| Space (RAM) | **O(n²)** — n intermediate strings | **O(n)** — one final string |
| n=100,000 lines | ~seconds | ~milliseconds |

**When `+` concatenation is acceptable:**
- Combining 2–3 known strings outside a loop: `path = dir + "/" + filename`. At small, fixed counts there's no loop overhead to compound.

---

## 10. `list` for Membership Testing

**Primary impact: Speed — O(n) per lookup vs O(1); 1,000× slower at scale**

```python
# ❌ BAD — linear scan every time
BLOCKED_IPS = ["192.168.1.5", "10.0.0.2", "172.16.0.99"]  # list

if src_ip in BLOCKED_IPS:     # O(n) scan on every packet — n=10k IPs = 10k comparisons
    drop(packet)
```

```python
# ✅ GOOD — hash lookup
BLOCKED_IPS = {"192.168.1.5", "10.0.0.2", "172.16.0.99"}  # set

if src_ip in BLOCKED_IPS:     # O(1) average — hash table lookup
    drop(packet)
```

| Metric | Bad (`list`) | Good (`set`) |
|---|---|---|
| Time per lookup | **O(n)** | **O(1)** average |
| Build time | O(n) | O(n) |
| Space | O(n) | O(n) (slightly higher constant) |
| n=10,000 IPs, 1M packets | ~10 billion comparisons | ~1 million lookups |

**When a `list` is correct for `in`:**
- When you need to preserve insertion order AND test membership on very small collections (n < ~10). For large n or repeated lookups: always `set` or `dict`.

---

## 11. Not Using Generators for Large Data

**Primary impact: Speed + RAM — loads entire dataset into memory when only one item is needed at a time**

```python
# ❌ BAD — reads all 10GB of packets into RAM before processing the first one
def load_all_packets(filepath):
    packets = []
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(1500), b""):
            packets.append(chunk)
    return packets    # O(n) RAM — entire file in memory

for pkt in load_all_packets("capture.pcap"):
    process(pkt)
```

```python
# ✅ GOOD — generator: one packet in memory at a time
def stream_packets(filepath):
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(1500), b""):
            yield chunk    # O(1) RAM — only current packet in memory

for pkt in stream_packets("capture.pcap"):
    process(pkt)
```

| Metric | Bad (list) | Good (generator) |
|---|---|---|
| Time | O(n) | O(n) |
| Space (RAM) | **O(n)** — full dataset | **O(1)** — single item |
| 10GB file | ~10GB RAM | ~1.5KB RAM |

**When loading everything is acceptable:**
- Random access patterns: if you need `packets[4500]` or multiple passes. Generators are forward-only. Use `list()` deliberately, not accidentally.

---

## 12. `map(lambda ...)` Instead of List Comprehension

**Primary impact: Speed + Cleanliness — function call overhead per element; less readable**

```python
# ❌ BAD — lambda + map: two function calls per element (map + lambda)
seq_numbers = [100, 200, 300, 400]
wrapped = list(map(lambda seq: (seq + 1) & 0xFFFFFFFF, seq_numbers))
# Also: list() forces eager evaluation, losing any lazy benefit of map
```

```python
# ✅ GOOD — list comprehension: no lambda overhead, compiled to bytecode directly
wrapped = [(seq + 1) & 0xFFFFFFFF for seq in seq_numbers]

# ✅ GOOD — if you only need to iterate once, keep it lazy with a generator expression
wrapped_iter = ((seq + 1) & 0xFFFFFFFF for seq in seq_numbers)
```

| Metric | Bad (`map+lambda`) | Good (comprehension) |
|---|---|---|
| Time | O(n) + n × function call overhead | O(n) — no call overhead |
| Space | O(n) (after `list()`) | O(n) list / O(1) generator |
| Benchmark (n=1M) | ~2× slower than comprehension | Baseline |

**When `map()` is acceptable:**
- When passing a named function (not a lambda): `list(map(str.upper, words))` — no lambda overhead, often faster than a comprehension. The anti-pattern is specifically `map(lambda ...)`.

---

## 13. Repeated Attribute Lookup in a Hot Loop

**Primary impact: Speed — attribute resolution runs on every iteration; O(n) extra dict lookups**

```python
# ❌ BAD — `self.connections` and `self.w.send` looked up on every packet
class FakeTcpInjector:
    def flush_all(self, packets):
        for pkt in packets:
            conn = self.connections.get(pkt.id)   # 2 attr lookups per iter
            self.w.send(pkt, False)               # 2 attr lookups per iter
            # total: 4 LOAD_ATTR bytecodes × n iterations
```

```python
# ✅ GOOD — hoist lookups out of the loop
class FakeTcpInjector:
    def flush_all(self, packets):
        connections = self.connections   # looked up once
        send = self.w.send               # looked up once
        for pkt in packets:
            conn = connections.get(pkt.id)
            send(pkt, False)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n × k) — k attrs per iter | O(k) + O(n) — hoisted |
| Space | O(1) | O(k) local vars |
| n=1M packets, k=4 | ~4M extra dict lookups | 4 lookups total |

**When hoisting is unnecessary:**
- Loop body runs < ~1,000 times. The overhead is measurable but irrelevant. Prioritise clarity over micro-optimisation at small n.

---

## 14. LBYL Dict Access (Double Lookup)

**Primary impact: Speed + Correctness (race condition in concurrent code) — pays cost twice**

```python
# ❌ BAD — two hash lookups; race window between check and access
if c_id in self.connections:               # lookup #1 — O(1)
    connection = self.connections[c_id]    # lookup #2 — O(1) again
    # another thread could delete c_id here; connection retrieval may still fail
```

```python
# ✅ GOOD — single lookup with sentinel default
connection = self.connections.get(c_id)    # O(1) — one lookup, thread-safer
if connection is None:
    self._send(packet)
    return

# ✅ GOOD — EAFP style for exceptional-case absences
try:
    connection = self.connections[c_id]
except KeyError:
    self._send(packet)
    return
```

| Metric | Bad (LBYL) | Good (`.get()` / EAFP) |
|---|---|---|
| Time (key present) | O(2) → O(1) | O(1) |
| Time (key absent) | O(1) | O(1) |
| Thread safety | ❌ TOCTOU window | ✅ Atomic |

**When LBYL double-lookup is acceptable:**
- Read-only code with no concurrency and n < 1,000 where you want to branch on presence without a default. Purely a style trade-off at that point.

---

---

# 🟢 LOW — Readability (No Runtime Impact)

---

## 15. `range(len(x))` for Iteration

**Primary impact: Cleanliness — obscures intent; forces manual indexing when indices aren't needed**

```python
# ❌ BAD — Java-style index-based loop
packets = [pkt1, pkt2, pkt3]
for i in range(len(packets)):
    process(packets[i])     # [i] access is redundant noise

# ❌ BAD — need index AND value
for i in range(len(packets)):
    print(i, packets[i])    # same info available from enumerate
```

```python
# ✅ GOOD — iterate directly
for packet in packets:
    process(packet)

# ✅ GOOD — need index AND value
for i, packet in enumerate(packets):
    print(i, packet)

# ✅ GOOD — need index AND value starting from 1
for i, packet in enumerate(packets, start=1):
    print(i, packet)
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(1) range object | O(1) iterator |
| Difference | None at runtime | Readability only |

**When `range(len(x))` is correct:**
- When you genuinely need to modify the list in-place by index: `for i in range(len(buf)): buf[i] ^= 0xFF` — direct index assignment requires `i`.

---

## 16. `not x == y` Instead of `x != y`

**Primary impact: Cleanliness — double negation forces the reader to decode intent**

```python
# ❌ BAD — reader must negate mentally
if not packet.tcp.seq_num == expected_seq:
    handle_mismatch()

if not connection.monitor == True:
    return
```

```python
# ✅ GOOD — direct, unambiguous
if packet.tcp.seq_num != expected_seq:
    handle_mismatch()

if not connection.monitor:    # even better — boolean flag needs no == True
    return
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(1) == then O(1) not | O(1) != |
| Space | O(1) | O(1) |
| CPython bytecode | 2 operations | 1 operation |

**When `not x == y` is acceptable:**
- When `__ne__` is not defined on a custom object but `__eq__` is — `not ==` falls back correctly while `!=` may raise `TypeError`. Extremely rare in Python 3 (default `__ne__` is auto-derived from `__eq__`).

---

## 17. Manual Counter Instead of `enumerate`

**Primary impact: Cleanliness — extra variable, extra mutation, extra line**

```python
# ❌ BAD — manually managing a counter variable
i = 0
for connection in active_connections:
    print(f"Connection #{i}: {connection}")
    i += 1
```

```python
# ✅ GOOD — enumerate is built for this
for i, connection in enumerate(active_connections, start=1):
    print(f"Connection #{i}: {connection}")
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(1) + counter variable | O(1) |
| Lines | 3 | 1 |

**When a manual counter is acceptable:**
- When counter increments are conditional: `if condition: i += 1`. `enumerate` always increments.

---

## 18. `zip` With `range(len(...))` for Parallel Iteration

**Primary impact: Cleanliness — unnecessary indexing when `zip` exists**

```python
# ❌ BAD — index used only to access both lists
src_ips = ["192.168.1.1", "10.0.0.1"]
dst_ips = ["8.8.8.8", "1.1.1.1"]

for i in range(len(src_ips)):
    print(f"{src_ips[i]} → {dst_ips[i]}")
```

```python
# ✅ GOOD — zip iterates both in parallel
for src, dst in zip(src_ips, dst_ips):
    print(f"{src} → {dst}")

# ✅ GOOD — if lists may differ in length and you need all items
from itertools import zip_longest
for src, dst in zip_longest(src_ips, dst_ips, fillvalue="N/A"):
    print(f"{src} → {dst}")
```

| Metric | Bad | Good |
|---|---|---|
| Time | O(n) | O(n) |
| Space | O(1) | O(1) |
| Difference | Readability only | — |

**When index-based parallel iteration is acceptable:**
- NumPy arrays: indexing is vectorised and often faster than Python-level iteration. Use indices for slicing.

---

## Summary Table

| # | Pattern | Severity | Primary Impact | Time Bad | Time Good | Space Bad | Space Good |
|---|---|---|---|---|---|---|---|
| 1 | Bare `except` | 🔴 CRITICAL | Safety | O(1) | O(1) | O(1) | O(1) |
| 2 | Mutable default args | 🔴 CRITICAL | Safety | O(1) | O(1) | **O(n) global** | O(n) local |
| 3 | `eval()` on user input | 🔴 CRITICAL | Safety | O(∞) | O(n) | O(∞) | O(n) |
| 4 | Wildcard imports | 🔴 CRITICAL | Safety | O(n) load | O(k) load | **O(n)** | O(k) |
| 5 | Modify while iterating | 🟠 HIGH | Correctness | **O(n²)** | O(n) | O(1) | O(n) |
| 6 | `is` for values | 🟠 HIGH | Correctness | O(1) | O(1) | O(1) | O(1) |
| 7 | No context managers | 🟠 HIGH | Safety | O(1) | O(1) | **O(n) leaking** | O(1) |
| 8 | `type()` vs `isinstance()` | 🟠 HIGH | Correctness | O(1) | O(d)≈O(1) | O(1) | O(1) |
| 9 | String `+` in loop | 🟡 MEDIUM | Speed | **O(n²)** | O(n) | **O(n²)** | O(n) |
| 10 | `list` for membership | 🟡 MEDIUM | Speed | **O(n)** per lookup | O(1) | O(n) | O(n) |
| 11 | No generators | 🟡 MEDIUM | Speed+RAM | O(n) | O(n) | **O(n)** | O(1) |
| 12 | `map(lambda ...)` | 🟡 MEDIUM | Speed+Clean | O(n)×call | O(n) | O(n) | O(n)/O(1) |
| 13 | Attr lookup in loop | 🟡 MEDIUM | Speed | O(n×k) | O(k)+O(n) | O(1) | O(k) |
| 14 | LBYL dict double lookup | 🟡 MEDIUM | Speed+Safety | O(2) | O(1) | O(1) | O(1) |
| 15 | `range(len(x))` | 🟢 LOW | Cleanliness | O(n) | O(n) | O(1) | O(1) |
| 16 | `not x == y` | 🟢 LOW | Cleanliness | O(1) | O(1) | O(1) | O(1) |
| 17 | Manual counter | 🟢 LOW | Cleanliness | O(n) | O(n) | O(1) | O(1) |
| 18 | `zip(range(len(...)))` | 🟢 LOW | Cleanliness | O(n) | O(n) | O(1) | O(1) |

---

*The most important column is not Big O — it's the severity level. An O(1) safety hole (bare `except`, mutable default) causes more damage in production than an O(n²) string concat that only fires on 10-element lists.*
