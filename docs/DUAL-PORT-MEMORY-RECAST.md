# The Keyboard Component dual-port memory: circuit hypothesis and recast

Status: first-pass circuit hypothesis, 2026-10-10. It is intentionally separated from established facts. One important premise is established for this project: **the CP1610 handles its external data through multiplexed bus phases**. There is no available physical Keyboard Component computer to probe or test. The one unit visible in the supplied video is observational evidence only; this document must not prescribe tests on it.

## The important deduction

MAME's Keyboard Component inventory names **ten MM5290J DRAMs** and describes the CP1610 memory as “16K×10.” That is highly suggestive of ten one-bit dynamic RAMs operated in parallel:

```text
  10 × (16,384 locations × 1 bit)  =  16,384 locations × 10 bits
```

Those are ordinary DRAMs, not a single dual-port-memory part. Therefore “dual-port” most likely describes the **service offered by surrounding logic**: either processor can reach the same storage, but only one is granted a DRAM access slot at a time. This also explains why the board inventory includes two 32×8 timing PROMs.

The DRAM fact is independently corroborated by the local [MM5290 datasheet](MM5290.pdf): it is explicitly a 16,384 × 1 DRAM with multiplexed row/column addresses, RAS, CAS, and refresh requirements. The board's exact arbitration remains a hypothesis; without a schematic or non-invasive source material, it cannot be promoted to fact.

## There are three different multiplexing problems

They must not be folded into one vague “memory mux” concept:

| Layer | What is multiplexed | Consequence for the board |
| --- | --- | --- |
| CP1610 interface — established premise | CP1610 external bus phases carry address and data at different times | The board needs an address-phase latch and a data-phase transceiver/driver before it can make a memory request. |
| MM5290 DRAM — datasheet fact | Seven address pins carry row address, then column address | The DRAM controller needs a row/column mux and correctly sequenced RAS/CAS. |
| CP1610/6502 sharing — board-level hypothesis | Two bus masters may request the same DRAM bank | Arbitration or time-slot logic must decide whose *completed* memory request reaches the DRAM next. |

The first two exist even if the 6502 were absent. The third is the genuinely unresolved dual-port mechanism.

## Likely historical circuit

```text
 CP1610 multiplexed bus       request arbiter / DRAM controller            6502 bus
 ┌───────────────────┐       ┌────────────────────────────────┐         ┌───────────────┐
 │ addr phase / data │─>     │ request / grant                │    <────┤ A[15:0],D[7:0]│
 │ phase, control    │  latch│ timing PROMs                   │ latch/  │  RD, WR, CLK  │
 └───────────────────┘       │ row/column address muxes       │ buffer  └───────────────┘
                             │ RAS, CAS, WE, refresh          │
                             │ 10-bit data latch/transceivers │
                             └──────────────┬─────────────────┘
                                            │ 10 data lines
                                ┌───────────▼───────────┐
                                │ 10 × MM5290J DRAM     │
                                │       16K × 10        │
                                └───────────────────────┘
```

## Current hypothetical RAM replacement

```text
 CP1610 multiplexed bus         request arbiter / SRAM bridge          M6x09 8-bit bus
 ┌───────────────────┐       ┌────────────────────────────────┐         ┌───────────────┐
 │ addr phase / data │─>     │ request / grant                │    <────┤ A[15:0],D[7:0]│
 │ phase, control    │  latch│ SRAM address; /CE, /OE, /WE    │ latch/  │  RD, WR, CTRL │
 └───────────────────┘       │ no RAS, CAS, or refresh        │ buffer  └───────────────┘
                             │ 10-bit data latch/transceivers │
                             └──────────────┬─────────────────┘
                                            │ 10 logical bits
                                ┌───────────▼───────────┐
                                │ 2 × 62256 SRAM        │
                                │ logical 16K × 10      │
                                └───────────────────────┘
```

### 62256 SRAM physical-bank detail

```text
 CP1610 multiplexed bus         request arbiter / SRAM bridge         M6x09 8-bit bus
 ┌───────────────────┐      ┌────────────────────────────────┐     ┌────────────────┐
 │ addr phase / data │─>    │ request / grant                │ <───┤ A[15:0],D[7:0] │
 │ phase, control    │ latch│ address; /CE, /OE, /WE         │ latch/│ RD, WR, control│
 └───────────────────┘      │ 10-bit data latch/transceivers │ buffer└────────────────┘
                            │ no RAS/CAS; no refresh         │
                            └──────────────┬─────────────────┘
                                           │ 10 logical bits
                               ┌───────────┴───────────┐
                               │                       │
                   ┌───────────▼───────────┐ ┌─────────▼─────────────┐
                   │ 62256 LOW SRAM        │ │ 62256 HIGH SRAM       │
                   │ bits 0–7              │ │ bits 8–9 used         │
                   │ A[13:0]; 16K in use   │ │ A[13:0]; 16K in use   │
                   └───────────────────────┘ └───────────────────────┘
```

This is a recast hypothesis, not a claim about the historical circuit. The bridge still serializes requests; SRAM removes DRAM row/column timing and refresh only.

### What each part probably does

| Block | Likely circuitry | Reason for it |
| --- | --- | --- |
| CP1610 interface | Address-phase latch, 10-bit data latch/buffer, phase/control decode, chip-select decode | The CP1610 multiplexes address/data phases; its request must be reconstructed before the shared-memory controller can arbitrate it. |
| 6502 interface | Address decode, 8-bit data buffer, separate access path for bits 8–9 | A 6502 is byte-oriented while the shared words appear to be 10 bits. |
| Arbiter | Flip-flop/state machine plus request/grant or fixed time slots | Both CPUs cannot safely drive the same DRAM cycle. |
| Timing PROMs | Microsequenced control outputs for mux selection, RAS, CAS, write-enable, refresh, and possibly grant timing | Two 32×8 PROMs are a compact 1970s/early-1980s way to generate a repeatable DRAM/bus timing state machine. |
| DRAM controller | Row/column address multiplexers, refresh counter, RAS/CAS timing, write gating | 16K DRAMs require multiplexed addressing and periodic refresh. |
| Bus isolation | 74xx/LS-series transceivers and tri-state buffers | Prevents either CPU's bus from electrically fighting the other or the DRAM data bus. |

## Why the clocks matter

The MAME configuration clocks both the CP1610 and 6502 at about 894.9 kHz (`3.579545 MHz / 4` and `7.159090 MHz / 8`). That could be coincidence in the emulator, but it is exactly the sort of relationship that simplifies a shared-DRAM arbiter: after a CP1610 address phase has been latched, the controller can assign fixed, non-overlapping DRAM subcycles to each processor and still reserve cycles for DRAM refresh.

A plausible repeating schedule would be:

```text
phase 0: capture CP1610 address phase or capture a 6502 request
phase 1: complete the selected CP1610 or 6502 DRAM cycle
phase 2: capture/select the other request
phase 3: complete its DRAM cycle
phase 4: refresh row n
... repeat
```

The actual scheme may instead grant one side until it is idle, or force wait states during video-sensitive periods. The timing PROMs are therefore a key uncertainty, not an available reverse-engineering target: we do not have a physical Keyboard Component to dump or probe.

## The 10-bit / 8-bit impedance mismatch

The CP1610-side window is 16K words. The 6502-side MAME model exposes:

* a 16K window for bits 0–7; and
* a second window represented as the upper two bits of the same logical words.

This is what one would expect if ten DRAM bit planes are shared by a 10-bit word machine and an 8-bit byte machine. It is not proof of the real register-level protocol: MAME's upper-plane mapping contains holes around the I/O and CRTC regions and may be a practical approximation. Do not assume that a physical 6502 reads “the top two bits” by the exact addresses used in the driver without validation.

The circuit could implement that mapping in several ways:

1. **Two aperture reads:** a low-byte aperture drives DRAM D0–D7; a second aperture packs DRAM D8–D9 into 6502 D0–D1.
2. **Latch-assisted access:** a first read fills a 10-bit latch, then separate registers return low and high portions.
3. **Address-bit banking:** one or more 6502 address bits select the low versus high data plane through multiplexers.

The first is the simplest match to the emulator, so it is the leading hypothesis.

## Collision semantics: the missing contract

The physical circuitry must choose a behavior when both processors request the same RAM cycle. Possibilities include:

| Policy | Effect | How to recognize it |
| --- | --- | --- |
| Fixed slot interleave | Predictable maximum delay for both CPUs | Plausible timing-PROM implementation; unverified |
| CP1610 priority | Game/console retains responsiveness; 6502 waits | Plausible endpoint-first implementation; unverified |
| 6502 priority | Text/OS service wins; console waits | Possible; unverified |
| Round-robin/last-grant | Better average fairness | Possible; unverified |

At the data level, a write must be atomic only at a single 10-bit word. Any structure larger than one word can still be torn if both processors update it without a software protocol. This is precisely the friction point: bare dual-port memory is shared storage, not a shared-language or ownership model.

## Recast: remove shared writable state

For machine intelligence, do not recreate raw dual-port memory as the normal internal interface. Preserve the valuable architectural separation while replacing the ambiguous bus with two explicit channels:

```text
  real-time endpoint                                  supervisory / agent host
  ┌─────────────────┐     observation FIFO      ┌──────────────────────────┐
  │ video capture   │ ─────────────────────────> │ estimator, policy, trace │
  │ action gate     │ <───────────────────────── │ experiment console       │
  │ clock + failsafe│        action FIFO          │ storage / replay         │
  └─────────────────┘                             └──────────────────────────┘
```

Each FIFO has one writer, one reader, sequence numbers, bounded capacity, and a defined overflow policy. The endpoint accepts actions only at a declared frame/tick boundary and falls back to a safe control state when the host is absent, late, or malformed. The host receives the endpoint tick and captured-frame ID with every observation, so a decision can be causally connected to its physical effect.

Use a third, immutable snapshot/trace ring for bulk state and training data. A producer writes the payload, checksum, and metadata, then publishes the sequence number last; readers only consume published sequences. This gives the useful part of the old shared workspace without exposing either side to torn mutable state.

## Chosen recast: M6x09 supervisory host

The new computer-side domain will be a homegrown **M6x09 board**, not a recreated 6502 subsystem. This is the right substitution because it preserves the historical split—an independent computer supervising an embodied endpoint—without inheriting the 10-bit shared-RAM problem.

```text
                    +--------------------------+
                    | endpoint interface board |
                    | timing / safety / bridge |
                    +--------------------------+
                       ^                    ^
                       |                    |
       endpoint-side signals            M6x09 memory-mapped bus
                       |                    |
              [console endpoint]      [M6x09 board]
```

The M6x09 board is a **peer host**, not a bus master on the original console's memory. It gets a clean, 8-bit, memory-mapped peripheral interface to the endpoint interface board. That board alone touches physical controller signals and, if compatibility demands it later, the CP1610 bus.

### Minimal M6x09-facing register contract

Exact addresses belong to the M6x09 memory map, but the interface should expose only these functions:

| Register group | Direction | Purpose |
| --- | --- | --- |
| `STATUS` | interface → M6x09 | FIFO fill levels, frame/tick, capture sync, reset state, fault bits |
| `OBS_DATA` / `OBS_POP` | interface → M6x09 | Ordered observation records, one consumer |
| `ACT_DATA` / `ACT_COMMIT` | M6x09 → interface | Staged action record; commit makes it eligible at its target tick |
| `CONTROL` | M6x09 → interface | Arm/disarm, clear fault, request safe neutral state |
| `ID / VERSION` | interface → M6x09 | Hardware/firmware compatibility discovery |

`ACT_COMMIT` is essential: writing bytes is not an action. Only a complete record with a valid length, sequence, target tick, and checksum may cross into the endpoint scheduler. A reset, underrun, overflow, invalid record, or host silence leaves the controller in its defined neutral/failsafe state.

### What this retains and what it intentionally discards

| Retained from the Keyboard Component | Deliberately discarded |
| --- | --- |
| Two independent compute domains | Random shared writable words |
| Deterministic endpoint timing | 10-bit/8-bit software-visible width mismatch |
| A local, inspectable host machine | Electrical bus contention as a normal operating mode |
| A bridge that can be instrumented | Unspecified collision semantics |

If running recovered Keyboard Component firmware becomes a preservation goal, add a separate FPGA/CPLD compatibility adapter. It may implement CP1610 bus demultiplexing and the measured 16K×10 behavior internally, but it remains outside the M6x09 experiment interface. That keeps firmware archaeology from contaminating the safety and trace contracts of the machine-intelligence system.

## Two build targets

| Target | What to build | Why |
| --- | --- | --- |
| Preservation-compatible expansion | FPGA/CPLD bridge with two bus front ends, internal 16K×10 memory, measured arbitration, optional original aperture mapping | Run or study recovered CP1610/6502 firmware. |
| New machine-intelligence system | Microcontroller/FPGA endpoint plus dual-clock FIFOs and trace storage | Deterministic safety, replayable experiments, and a clean agent interface. |

The first target may internally emulate a 16K×10 store, but it should keep that format behind a compatibility boundary. The second should not expose random shared writes at all.

## Evidence boundary and next sources

There is no test plan involving a real Keyboard Component. No unit is available to instrument, and the machine visible in the video must not be treated as accessible hardware. The correct next evidence sources are non-invasive:

1. additional archival schematics, service material, board photographs, or ROM/PROM dumps released by their owners;
2. comparison of independent emulator and preservation sources, while retaining provenance for every claim; and
3. a new, self-contained M6x09/62256 prototype that tests only the *new* bridge contract, never a claim about original electrical behavior.

The prototype can validate its own phase adapter, 10-bit representation, arbitration policy, trace counters, and reset rules. It cannot validate which historic policy the Keyboard Component used.

Until then, the correct engineering position is: **the 16K×10 DRAM bank is likely; literal dual-port behavior and its arbitration protocol are not yet established.**
