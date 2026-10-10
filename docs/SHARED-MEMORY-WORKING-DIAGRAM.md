# Shared-memory working diagram

Use this as a paper starting point. Solid blocks are required by the memory problem; the `?` marks are unresolved historical behavior. Nothing here assigns gameplay, player, or machine-intelligence meaning to the memory.

## 1. Historical hardware hypothesis

```text
 CP1610                                                  6502
 multiplexed bus                                          8-bit bus
     |                                                        |
     v                                                        v
+-------------------+                              +-------------------+
| CP bus-phase      |                              | 6502 memory        |
| capture           |                              | adapter            |
| address + control |                              | address + R/W      |
| data[9:0]         |                              | data[7:0]          |
+---------+---------+                              +---------+---------+
          |                                              |        |
          | complete CP request                          |        | ? upper bits
          |                                              |        |
          +----------------------+  +--------------------+        |
                                 v  v                             |
                       +--------------------+                     |
                       | REQUEST ARBITER    |<--------------------+
                       |                    |
                       | ? priority         |
                       | ? fixed slots      |
                       | ? wait behaviour   |
                       +---------+----------+
                                 |
                         exactly one grant
                                 |
                                 v
                       +--------------------+
                       | DRAM CONTROLLER    |
                       | row / column mux   |
                       | RAS / CAS / WE     |
                       | refresh            |
                       +---------+----------+
                                 |
                              10 bits
                                 |
                                 v
                       +--------------------+
                       | 10 x MM5290        |
                       | 16K x 1 DRAM       |
                       | = logical 16K x 10 |
                       +--------------------+
```

The central rule to work from is:

```text
One granted request -> one completed physical memory cycle -> next request.
```

That rule describes serialized access, not coherent shared state. The original board's arbitration details remain unknown.

## 2. 62256 recast

```text
 CP1610-compatible side                              M6x09 side
 (only if required)                                  8-bit bus
        |                                                |
        v                                                v
+--------------------+                         +--------------------+
| CP phase adapter   |                         | M6x09 adapter      |
| complete 10-bit    |                         | LOW[address]       |
| request            |                         | HIGH[address]      |
+---------+----------+                         +---------+----------+
          |                                              |
          +----------------------+  +--------------------+
                                 v  v
                       +--------------------+
                       | BRIDGE             |
                       | grant / completion |
                       | transaction trace  |
                       | word lock for RMW  |
                       +---------+----------+
                                 |
                  +--------------+--------------+
                  |                             |
                  v                             v
        +--------------------+        +--------------------+
        | 62256 LOW          |        | 62256 HIGH         |
        | A[13:0]            |        | A[13:0]            |
        | D[7:0] = bits 0-7  |        | D[1:0] = bits 8-9  |
        +--------------------+        +--------------------+
                  \                             /
                   \                           /
                    +-------------------------+
                    | logical store: 16K x 10 |
                    +-------------------------+
```

`LOW[address]` and `HIGH[address]` are a compatibility representation only. A M6x09-side update of both portions is a bridge transaction:

```text
lock word address
read unchanged portion
merge new portion
write logical word
release word address
record completion sequence
```

The lock prevents the bridge itself from tearing a 10-bit word during that operation. It does not create multiword atomicity; records larger than one logical word require an explicit commit/sequence protocol above this layer.

## 3. Questions to annotate on paper

1. Which historic signals establish “CP request complete” after its multiplexed bus phases?
2. Does the historic 6502 access one 10-bit word in one operation, or low and upper portions separately?
3. What decides the next grant: fixed timing, request priority, or a PROM-driven sequence?
4. What is the smallest evidence-supported contract to preserve in the M6x09/62256 bridge?
5. Which state must be visible in the bridge trace without assigning it higher-level meaning?

Do not answer questions 2–3 from this drawing; they are deliberately marked as unknowns.
