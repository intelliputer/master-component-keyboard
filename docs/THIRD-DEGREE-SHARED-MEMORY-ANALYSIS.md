# Third-degree analysis: recasting the shared memory with 62256 SRAM

Date: 2026-10-10  
Scope: only the low-level storage and bus-interface layer. This document makes **no claim** about what the Keyboard Component means to a game, how it provides a game object to a player, or how a human/autonomous player should be controlled. Those are higher-degree questions.

## Starting facts

The MAME board inventory records ten MM5290J DRAMs for the Keyboard Component memory. The local [MM5290 datasheet](MM5290.pdf) identifies each part as a 16,384 × 1 dynamic RAM with:

* seven multiplexed address inputs (`A0–A6`), used first for row and then for column;
* separate RAS, CAS, and write-enable timing;
* a single data input and a single data output; and
* refresh of all 128 row addresses within 2 ms.

Ten such devices in parallel naturally form a **16,384 × 10-bit** store. This explains the MAME representation of a 16K CP1610-visible word store with a low 8-bit view and an upper 2-bit view for the 6502. It does not establish the original board's exact arbitration or software protocol.

## Direct SRAM substitution

A 62256 is a 32K × 8 asynchronous static RAM. Two devices are sufficient to replace the capacity and width of the inferred MM5290 bank:

```text
                     logical 16K × 10 shared store

 address A[13:0] ────────────────────────┬───────────────────────┐
 control /OE /WE ─────────────────────────┼───────────────────────┤
                                         ▼                       ▼
                                  ┌─────────────┐         ┌─────────────┐
                                  │ 62256 LOW   │         │ 62256 HIGH  │
                                  │ D[7:0]      │         │ D[1:0] used │
                                  │ 16K of 32K  │         │ 16K of 32K  │
                                  └─────────────┘         └─────────────┘
                                        │                       │
                                 word bits 0–7          word bits 8–9
```

Wire `A0–A13` to both SRAMs. Tie each `A14` to a defined level so only 16K locations are selected; leave the unused upper half unavailable. The low SRAM's eight data pins represent bits 0–7. Only two pins of the high SRAM represent bits 8–9; its remaining six data pins must be left unused or reserved behind an explicit design decision. Do not treat them as silently available compatibility bits.

This substitution removes row/column address multiplexing, RAS/CAS generation, and refresh. It does **not** remove the need to:

1. capture and interpret the CP1610's multiplexed bus phases;
2. translate the CP1610 word view and M6x09 byte view; and
3. arbitrate if both sides can request the storage independently.

## Behavioral difference: DRAM versus 62256 SRAM

At the software level either store can provide the same basic contract—read a location, obtain a value; write a location, later obtain the written value. The difference is the physical behavior needed to provide that contract.

| Behavior | MM5290 DRAM (historic bank) | 62256 SRAM (recast bank) |
| --- | --- | --- |
| Storage cell | Capacitor charge | Static latch |
| Retention while powered | Requires periodic refresh | Retains state continuously while powered |
| Addressing | Seven pins carry row address, then column address | A complete address remains present for the memory cycle |
| Control | RAS, CAS, write-enable, and a refresh schedule | Chip-enable, output-enable, and write-enable |
| Data pins | Separate data input and data output | Bidirectional data pins |
| Access behavior | A controller sequences several phases | An asynchronous memory cycle can be direct |
| Supply burden | +12 V, +5 V, and −5 V on the MM5290 | Normally a single +5 V rail |
| Power-off retention | Lost | Lost |
| Simultaneous independent access | Not provided | Not provided |

The original DRAM controller must repeatedly perform the equivalent of:

```text
present row address → assert RAS
present column address → assert CAS
read or write data
periodically refresh every row
```

The 62256 reduces this to an SRAM access:

```text
present complete address → assert chip/output enable → read data
present address + data → assert write enable → write data
```

This is a major electrical simplification, not a license to simplify the machine's observable behavior. A 62256 removes DRAM refresh and row/column sequencing; it does not make the two CPU interfaces coherent or dual-ported, and it does not resolve higher-level races over multiword state.

## Preservation constraints for the new design

The storage replacement must be a subordinate implementation choice. It must not silently remove machine-intelligence features or prematurely impose an interpretation on them.

| Preserve | Why it matters | Bridge requirement |
| --- | --- | --- |
| Independent domains | The historical design separates the console endpoint from the computer-side domain | Keep separately clocked/requested interfaces; do not turn the M6x09 into an unbounded master of the endpoint. |
| Deterministic ordering | An experiment needs to know what observation preceded what action/state change | Sequence every completed bridge transaction and expose a monotonic bridge tick. |
| Timing observability | Historic timing may matter; future intelligence experiments need causal traces | Timestamp requests, grants, completions, stalls, reset events, and faults. |
| Explicit ownership | Shared words alone do not say who is allowed to update a structure | Define ownership per aperture/register/queue, including handoff and reset rules. |
| Full logical 10-bit state | Losing bits 8–9 can alter compatibility behavior and hides information | Store and test all ten bits; preserve unused physical SRAM bits as unavailable by default. |
| Safe failure | Host or bridge failure must not create undefined endpoint behavior | Default external-bus outputs to high impedance; report faults and enter a defined reset/safe state. |
| Reproducibility | Model training and historical investigation both need replayable evidence | Retain an append-only trace export; never use SRAM contents as the sole historical record. |
| Architectural openness | The memory layer must not decide gameplay/player semantics | Keep only physical transactions and neutral metadata in this layer. |

The critical design rule is therefore: **emulate only the historic memory service that has evidence, and add observability around it; do not bake higher-degree meanings into the bridge.**

## Recommended physical boundary

Place the two 62256s behind a small bridge implemented in a CPLD or FPGA. The bridge is the owner of SRAM address, data, and control pins. Neither CPU bus should directly drive the SRAM.

```text
 CP1610 multiplexed bus ──> CP phase latch / adapter ─┐
                                                       │
 M6x09 8-bit bus ────────> M6x09 register adapter ────┼─> CPLD/FPGA ─> 2 × 62256
                                                       │       │
 reset / diagnostic ──────────────────────────────────┘       └─> status/interrupt
```

This is deliberately a **bridge**, not literal electrical dual-port SRAM. A 62256 has one address/data port. The CPLD/FPGA serializes accesses internally and makes the chosen contention rule observable.

### Minimal bridge responsibilities

| Function | Why it belongs in the bridge |
| --- | --- |
| CP1610 address/data phase capture | The CP1610's multiplexed bus must be reconstructed into a complete request before accessing SRAM. |
| Width conversion | CP1610-facing data is logically 10 bits; M6x09-side data is 8 bits. |
| SRAM cycle generation | Address, `/CE`, `/OE`, and `/WE` must remain stable for the chosen 62256's datasheet timing. |
| Arbitration | Defines what happens if requests overlap; a single-port SRAM cannot decide this itself. |
| Read-data hold/drive timing | Ensures neither CPU sees invalid data or bus contention. |
| Reset and diagnostics | Makes initial contents, bus faults, and test access explicit. |

## M6x09 representation

For a compatibility-oriented view, expose the logical 10-bit word as two M6x09-visible apertures:

| Aperture | Read/write data | Meaning |
| --- | --- | --- |
| `LOW[address]` | 8 bits | Logical word bits 0–7 |
| `HIGH[address]` | bits 0–1 significant | Logical word bits 8–9; read returns zeros in bits 2–7 |

The bridge must perform a read-modify-write when an M6x09 write changes only the high two bits, preserving the low eight bits. Conversely, the high bits must be preserved when a low-byte write arrives. This is an implementation transaction, not an atomic operation visible to a competing requester; the bridge must lock the logical word for the duration.

For a non-compatibility experiment interface, do not expose either aperture. Keep the 16K×10 representation private and expose queues, snapshots, or named registers. The two choices can coexist: compatibility apertures for archaeology, message interfaces for new work.

## Arbitration: make the policy a parameter

The original policy is unknown. The new bridge should make it selectable in build-time configuration and report it in its ID/version register.

| Policy | Rule | Suitable use |
| --- | --- | --- |
| `CP_PRIORITY` | Complete CP1610 requests before M6x09 requests | Safest first setting while validating a live console endpoint. |
| `ROUND_ROBIN` | Alternate after each completed access while both request | Fair stress testing and measurement. |
| `FIXED_SLOT` | Allocate predetermined bridge cycles to each side | Faithful-timing experiments once original behavior is measured. |
| `M6X09_ONLY` | Disable CP1610 aperture | Standalone development of the storage/register interface. |

Each completed access should increment separate counters: CP reads/writes, M6x09 reads/writes, arbitration waits, and rejected/faulted transactions. Such counters make a third-degree decision measurable instead of inferred from software behavior.

## Timing and electrical notes

* Choose a 5 V-compatible 62256 with access time comfortably below the bridge cycle budget. At the roughly 895 kHz CPU-rate indicated by the emulator configuration, even a conservative SRAM is vastly faster than the historic DRAM service requirement; bridge sequencing and bus turnaround dominate.
* The original MM5290 uses +12 V, +5 V, and −5 V supplies. A 62256-based bank generally removes the +12 V/−5 V DRAM power burden, but the surrounding original board may still require those rails elsewhere. Do not remove supply rails until the actual board is traced.
* Add series resistors or registered/tri-stated bridge outputs at any connection to an original CPU bus. Default reset state must be high impedance toward that bus.
* A logic analyzer header for CP phase/control, M6x09 request/acknowledge, SRAM address/control, and bridge grant is worth more than speculative firmware complexity.

## Bring-up sequence

1. Validate the CPLD/FPGA plus two 62256s from the M6x09 side only: walking bits, address uniqueness, low/high aperture preservation, and reset contents.
2. Enable the CP1610 adapter with its outputs disabled; capture the multiplexed phases and confirm the reconstructed address/control requests.
3. Enable CP reads, then writes, with `CP_PRIORITY` selected and M6x09 idle.
4. Run a non-destructive cross-side pattern test with bridge counters and capture signals enabled.
5. Only then test alternate arbitration policies or recovered-firmware compatibility.

## Decisions deferred on purpose

* The original board's true grant/wait-state behavior.
* The original 6502 upper-two-bit aperture semantics.
* Any role the shared store had in gameplay, player interaction, or autonomous agency.
* Whether a final system should retain CP1610 compatibility at all.

The 62256 solution is a low-risk storage replacement. It lets us investigate the historic interface while keeping all architectural meaning above this third-degree layer open.
