# M6x09-to-10-bit memory bridge analysis

Date: 2026-10-10  
Scope: the new bridge between an 8-bit M6x09 bus and a logical 16K × 10-bit memory store. This is a recast design note. It does not establish the original Keyboard Component's 6502 protocol, gameplay role, or player relationship.

## Problem statement

The inferred historic store is 16K × 10 bits. A 6809 or 6309 has an 8-bit external data bus (`D[7:0]`) and a 16-bit address bus (`A[15:0]`). Its additional 6309 registers help firmware hold values but do not widen physical bus transfers.

The bridge therefore has to turn several M6x09 byte accesses into one explicitly committed **logical 10-bit operation**. It must also serialize that operation with any CP1610-compatible request, if that side is present.

```text
 M6x09 bus                                      logical memory service
 A[15:0], D[7:0]                                        16K x 10
        |                                                   |
        v                                                   v
 +----------------+     complete request       +----------------------+
 | register        | ------------------------> | arbitration / word   |
 | interface       | <------------------------ | lock / SRAM access   |
 +----------------+      result + status       +----------+-----------+
                                                       |        |
                                                       v        v
                                                 62256 LOW  62256 HIGH
                                                  bits 0-7   bits 8-9
```

## The non-negotiable distinction

There are two different interfaces worth keeping separate:

| Interface | Purpose | Status |
| --- | --- | --- |
| Historic-compatible aperture | Mimic the eventual, evidenced 6502-visible low-byte/high-two-bit behavior | Deferred; MAME is a hypothesis, not sufficient proof |
| M6x09 transaction interface | Give new M6x09 firmware an unambiguous 10-bit read/write operation | Recommended starting interface |

The M6x09 should use the transaction interface. Do not force all new firmware through a guessed 6502 aperture merely because both CPUs are 8-bit.

## Minimal M6x09 register contract

The addresses are deliberately not chosen here; allocate one decoded I/O window in the actual M6x09 memory map.

| Register | Direction | Bits | Meaning |
| --- | --- | --- | --- |
| `ADDR_LO` | write | 7:0 | Logical word address bits 7:0 |
| `ADDR_HI` | write | 5:0 | Logical word address bits 13:8; upper bits must read/write as zero |
| `WRITE_LO` | write | 7:0 | Logical word bits 7:0, staged only |
| `WRITE_HI` | write | 1:0 | Logical word bits 9:8, staged only |
| `COMMIT_WRITE` | write | 0 | Submit one staged 10-bit write |
| `START_READ` | write | 0 | Request one 10-bit read at the staged address |
| `READ_LO` | read | 7:0 | Latched result bits 7:0 |
| `READ_HI` | read | 1:0 | Latched result bits 9:8; bits 7:2 read zero |
| `STATUS` | read | — | Busy, result-valid, arbitration wait, fault, and sequence state |
| `ACK` | write | — | Acknowledge/retire the current read result or fault |

`COMMIT_WRITE` is the important boundary. Writes to `ADDR_*` and `WRITE_*` merely prepare a request. They must have no effect on the logical memory until the bridge accepts `COMMIT_WRITE`.

## Required bridge behavior

### Write a logical word

```text
M6x09 stages address, low byte, and high two bits
M6x09 writes COMMIT_WRITE
bridge assigns a request sequence number
bridge waits for arbitration grant
bridge locks logical address N
bridge writes LOW[N] = bits 0-7 and HIGH[N] = bits 8-9
bridge releases N and marks the sequence complete
```

For the two-62256 implementation, the low SRAM receives all eight low bits. The high SRAM receives only two meaningful data bits. The other six physical bits are reserved/ignored and must not become untracked state.

### Read a logical word

```text
M6x09 stages address N
M6x09 writes START_READ
bridge assigns a request sequence number
bridge waits for arbitration grant
bridge locks logical address N
bridge reads LOW[N] and HIGH[N]
bridge releases N, latches both portions, and raises result-valid
M6x09 reads READ_LO and READ_HI
M6x09 writes ACK
```

The result latch prevents a later operation from changing one portion while the M6x09 reads the other.

## Consistency contract

The bridge should promise no more and no less than this:

| Scope | Guaranteed behavior |
| --- | --- |
| One submitted M6x09 read/write | Ordered by bridge sequence number |
| One logical 10-bit word | Read or written as one bridge transaction |
| Simultaneous CP-compatible request | Serialized by an explicit arbitration policy |
| Larger records | No automatic atomicity; use a sequence/commit protocol above the bridge |
| Reset/fault | Incomplete staged requests do not modify logical memory |

This is stronger and clearer than exposing independent low/high byte windows to new firmware, while still allowing a compatibility aperture to be added later.

## CP1610-compatible side

If CP1610 compatibility is included, its multiplexed address/data phases are reconstructed into a complete request before entering the same arbitration queue. A CP request can be represented as:

```text
{ requester = CP, operation = read|write, address[13:0], data[9:0] }
```

The M6x09 side produces the equivalent request only after `COMMIT_WRITE` or `START_READ`. Neither side directly drives the 62256s.

The historic arbitration policy is unknown. For the new bridge it must be named, versioned, and traced—for example `CP_PRIORITY`, `ROUND_ROBIN`, or `FIXED_SLOT`—rather than silently assumed to reproduce the original.

## 6309 use

The 6309 can make the M6x09-side firmware more pleasant and faster:

* retain a packed 10-bit value in a 16-bit register;
* retain addresses, sequence values, and masks in its additional registers; and
* move trace data more efficiently.

It does not change the bridge contract or the physical `D[7:0]` bus. A 6809 implementation uses the same registers and transaction protocol; it may simply take more instructions.

## Questions intentionally left open

* Whether historical software requires a low/high aperture at all, and its exact semantics.
* Whether a CP1610-compatible interface is required in the first new prototype.
* The address range and interrupt wiring of the M6x09 I/O window.
* The arbitration policy for a future two-requester bridge.
* Any interpretation of the 10-bit store above transaction mechanics.

The recommended first prototype is M6x09-only: implement the register contract and two 62256 banks, trace every submitted/completed request, and validate 10-bit integrity. Add the CP-compatible side only when the historically required behavior is known.
