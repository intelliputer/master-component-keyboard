# Keyboard Component: first-pass technical foundation

Date: 2026-10-10  
Scope: a reconstruction from the MAME driver in `mattel/`, the local assembly/burn-in document, and the supplied video. This is an architecture baseline, not yet a schematic-level reproduction.

## Executive reading

The Keyboard Component is best understood as a **two-computer expansion**, not as a keyboard bolted onto an Intellivision.

1. The original Intellivision remains a CP1610/STIC game computer with cartridge, controllers, sound, and its own display generator.
2. A second computer, built around a 6502, supplies a keyboard, 40-column text system, cassette and printer-facing I/O, and BASIC I/O cartridges.
3. The two processors cooperate through a large dual-port memory. The text system is composited over the game display, which makes the appliance simultaneously a game machine and a home computer.

That split is the unusually useful idea for machine-intelligence work: retain a deterministic, low-latency embodied/game endpoint while placing interaction, inspection, storage, and higher-level control in a separate host domain. The host need not steal cycles from the endpoint and the endpoint need not trust the host with its timing-critical state.

## Evidence and confidence

| Claim | Confidence | Basis |
| --- | --- | --- |
| The system contains CP1610 and 6502 processors. | High | MAME instantiates both in `mattel/intv.cpp`; the 6502 ROM and CPU are also listed in the driver notes. |
| The processors share dual-port memory. | High | The CP1610 map exposes `0x8000–0xbfff`; the 6502 map exposes corresponding low-byte and upper-bit windows. The handlers explicitly call it dual-port memory. |
| The shared store is logically 16K × 10 bits. | Medium-high | Its CP1610 window is 16K words; the 6502 sees an 8-bit low plane and only two upper bits. This is a strong inference from the emulator mapping, not a recovered memory schematic. |
| The component has a 6502 text plane, keyboard matrix, cassette, printer interface, and BASIC I/O cartridge slots. | High | Explicit 6502 address map and I/O handlers in the driver. The assembly/burn-in flow also names logic board, keyboard, tape-control board, preamp, tape deck, and power supply. |
| Text is 40 columns × 24 rows, using 8×8 monochrome glyphs. | High | The renderer reads 40 columns over 24 rows from a 2 KiB text RAM and decodes a one-bit 8×8 character set. |
| The observed video demonstrates the Keyboard Component itself. | Low | The video clearly shows an Intellivision and an external autonomous-controller setup. It does not show a Keyboard Component boot, text screen, cassette, or internal board closely enough to establish those details. |

MAME marks `intvkbd` as **not working**. Its code is therefore a very valuable executable hypothesis, but not an authority for electrical timing, tape behavior, or every register meaning.

## Reconstructed machine

```text
                         original Intellivision domain
  controllers ──> AY-3-8914 I/O ──> CP1610 ──> STIC ──> composite/video
                                      │          │
                                      │          └──── game image
                                      │
                         16K × 10-bit dual-port memory
                                      │
                                      ▼
  keyboard ──> matrix I/O ──> 6502 ──> CRTC + text RAM + character ROM
                                  │                  │
                                  ├── cassette control / preamp / deck
                                  ├── printer interface
                                  └── BASIC I/O cartridge slots

                       text plane overlays the game image
```

### Clocking and display path

The driver clocks the CP1610 at `3.579545 MHz / 4` and the 6502 and TMS9927-compatible CRTC at `7.159090 MHz / 8`: approximately 894.9 kHz in all three cases. The STIC uses the 3.579545 MHz colorburst crystal and the screen refresh is configured at 59.92 Hz. Treat these as a first timing target to validate against hardware, not a final measured specification.

The base graphics engine is the STIC: 20 × 12 cards, nominally 8 × 8 pixels, with an effective game image of 159 × 96 pixels (the last column is seven pixels) and vertical scaling in the emulator. The Keyboard Component doubles the horizontal output scale and overlays the 6502 text layer. The text layer is 40 × 24 characters, 8 × 8 pixels per character, white glyphs, with a cursor generated from the CRTC state. The visible text footprint is consequently 320 × 192 pixels before the surrounding video geometry is considered.

### Processor roles

| Domain | CPU and firmware | Responsibilities exposed by the source |
| --- | --- | --- |
| Console/endpoint | CP1610 plus Intellivision EXEC and Keyboard Component CP1610 ROM | Cartridge/game execution, STIC graphics, sound and controller-facing I/O; access to the shared store. |
| Computer/interaction host | 6502, a 4 KiB mask ROM and a 4 KiB EPROM | Keyboard scan, text display, cassette control/status, printer-facing port, and I/O-cartridge access. |

The ROM inventory preserved in the driver is useful for an eventual preservation plan: `0104.U20` is 6502 mask ROM, `CPU2D.U21` is 6502 EPROM, `024.U60` plus `4D72.U62`/`4D71.U63` form the Keyboard Component CP1610 firmware, and `4C52.U34` is the character ROM. Two small PROMs are labelled as timing PROMs, but their exact role needs schematic or board verification.

## Address-map contract

Addresses below are the emulator's CPU-visible addresses. They are the contract to reproduce in a behavioral model; they should not yet be used to derive physical pin connections.

### CP1610 side

| Range | Function |
| --- | --- |
| `0x0000–0x003f` | STIC registers |
| `0x0100–0x01ef` | 8-bit system RAM |
| `0x01f0–0x01ff` | AY-3-8914 sound and controller I/O |
| `0x0200–0x035f` | 16-bit system RAM |
| `0x1000–0x1fff` | Intellivision EXEC ROM |
| `0x3000–0x37ff` | STIC GROM; `0x3800–0x3bff` GRAM |
| `0x7000–0x7fff` | Keyboard Component CP1610 ROM |
| `0x8000–0xbfff` | Shared dual-port memory |
| cartridge regions | Remain available in the conventional cartridge windows |

### 6502 side

| Range | Function |
| --- | --- |
| `0x0000–0x3fff` | Low 8-bit plane of shared memory |
| `0x4000–0x40bf` | Component I/O: cassette status/control, text blanking, and keyboard selection |
| `0x40c0–0x40cf` | TMS9927-compatible CRTC registers |
| `0x4200–0x7fff` | Upper shared-memory bits, represented by the driver as a 2-bit plane |
| `0xb7f8–0xb7ff` | Peripheral interface; printer status/control/data use `0xb7fe–0xb7ff` |
| `0xb800–0xbfff` | 2 KiB text-display RAM |
| `0xc000–0xdfff` | 6502 firmware ROM |
| `0xe000–0xffff` | I/O cartridge ROM if installed, otherwise firmware/EPROM readback |

The awkward separation of the upper shared-memory plane is a warning sign: it may reflect genuine 10-bit memory organization, but the exact accessibility of its initial portion and bus arbitration need measurement or schematics.

## The dual-port-memory friction point

This is the key part to **recast**, rather than reproduce by default. A literal implementation requires two independently clocked buses to contend for a non-standard-width, shared random-access store. It also leaves the most important semantic questions implicit: who owns a location, what a simultaneous read/write means, whether either CPU is stalled, and how either side detects a partially written multiword structure. The driver preserves the address windows, but does not establish those electrical or concurrency rules.

The historical store served three distinct purposes which should be separated in a new design:

| Original apparent use | Modern replacement | Required property |
| --- | --- | --- |
| Fast control/status handoff | Two unidirectional asynchronous FIFOs | Ordered, bounded latency; no torn messages |
| Bulk data/program exchange | Chunked transfer buffer with producer/consumer ownership | Explicit completion and checksum |
| Shared inspection/work area | Versioned snapshot ring | Readers never observe a half-written snapshot |

The resulting bridge has no writable location owned by both domains. Each queue has exactly one writer and one reader; ownership transfers only by advancing a committed sequence number.

```text
 endpoint clock domain                                      host clock domain
 ┌────────────────────┐     observations / frames     ┌────────────────────┐
 │ capture + safety    │ ──> [EP→HOST async FIFO] ──> │ agent + recorder   │
 │ action scheduler    │ <── [HOST→EP async FIFO] <── │ planner / console  │
 └────────────────────┘                                └────────────────────┘
             │                                                       │
             └────── immutable, sequence-numbered trace/snapshots ──┘
```

For the current autonomous-player work, the endpoint should accept only a compact action record such as `{sequence, target_frame, controls, maximum_duration, safe_fallback, crc}`. It acknowledges the sequence when the action is actually applied, not when it is merely received. The host receives `{sequence, captured_frame, endpoint_tick, observations, fault_flags}`. This creates causal data for training and debugging: every decision can be joined to the sensed frame and the physical action that reached the controller.

There are two legitimate build modes, and they should not be conflated:

| Mode | Bridge implementation | Use |
| --- | --- | --- |
| Compatibility/preservation | FPGA or CPLD bus bridge presenting the original CP1610 and 6502 windows, backed by dual-clock RAM and a measured collision/wait-state policy | Run recovered firmware or validate original hardware behavior |
| Machine-intelligence rebuild | FIFO/message bridge plus trace store; no direct host writes into endpoint state | Reliable experiments, safe actuation, replay, and model training |

The compatibility bridge can emulate a 16K × 10-bit store internally, including the 8-bit low plane and 2-bit upper plane, but that representation must remain behind the bridge. It is an adapter for old software, not the internal contract of the new system.

## Human and peripheral interfaces

The keyboard is an active-low 10 × 8 scanned matrix. The 6502 selects the matrix line with four one-bit latches at I/O offsets `0x44–0x47`, then reads a line at offset `0x60`. The documented logical layout includes Shift, Control, Repeat, Lock, Clear Screen, Tab, Escape, Return, cursor keys, alphanumerics, and punctuation. This provides a compact, deterministic event surface well suited to recording and replay.

Cassette I/O is more than ordinary audio storage in the model. It exposes data, ready/watermark, leader/end-of-tape, cassette-present, inter-record-gap, dropout, clock interrupt, and tape interrupt status. Controls separately select drive enable, direction, fast mode, record/read, channel mute, data/audio mode, erase, and interrupt enable. The driver declares tape support incomplete, so the presence and broad interface are reliable, while semantics and timing are not.

The printer interface is only partially modelled: a status/control register at `0xb7fe` and output data at `0xb7ff`. Two generic I/O cartridge slots are configured for BASIC-related cartridges.

## What the local non-code material adds

`docs/CCF10232011_00020.pdf` is a 35-page Keyboard Component assembly and burn-in document, not a circuit schematic. In its system-test flow it explicitly enumerates the logic board, keyboard, power-supply board, tape-control board, preamp board, tape deck, and plastic housing. It also places separate 24-hour burn-in stages around system assembly. This corroborates the modular physical design and identifies the cassette chain as several physical subsystems rather than a single abstract peripheral.

`video/keyboard-component-video.mp4` is 16:48 long. It visibly shows an Intellivision boot/menu, a long computer-controlled football sequence, then a wired controller/console and laptop workbench setup. It is useful evidence for the present machine-intelligence direction: external actuation can coexist with unmodified game video. It is not evidence for the Keyboard Component's electrical or software behavior.

## Recast for machine intelligence

Preserve the architecture's separation of concerns, but replace fragile historical interfaces with explicit, inspectable contracts:

```text
             real-time endpoint                     supervisory host
       ┌───────────────────────────┐        ┌────────────────────────────┐
       │ console / game / actuator │        │ agent runtime               │
       │ video capture             │ events │ state estimator + policy    │
       │ controller safety gate    │───────>│ experiment recorder          │
       │ deterministic tick clock  │<───────│ bounded action mailbox       │
       └───────────────────────────┘ action └────────────────────────────┘
                         ▲                         ▲
                         └── versioned shared trace ┘
```

The modern analogue of the dual-port RAM should be a **pair of versioned, bounded mailboxes**, not casually shared mutable state:

* Endpoint → host: video-frame IDs, sampled controller state, emulated/hardware time, game observations, and fault flags.
* Host → endpoint: timestamped action intents with a maximum duration, an idempotency ID, and a required safe/default action.
* Both directions: sequence numbers, ownership rules, clock domain, and overflow policy captured in the trace format.

The design should maintain three planes:

| Plane | Job | Historical inspiration |
| --- | --- | --- |
| Embodied endpoint | Frame-accurate actuation and sensing, independent fail-safe behavior | CP1610/STIC console domain |
| Interaction/experiment plane | Console, diagnostics, program loading, experiment control, and data capture | 6502/text/keyboard/cassette domain |
| Bridge | Deterministic exchange with no hidden authority | dual-port memory |

This avoids a common failure mode in agent projects: allowing a high-level process to control timing-sensitive I/O directly. The host proposes bounded actions; the endpoint enforces the clock, electrical limits, and fallback behavior.

## First implementation slice

Build a behavioral “Keyboard Component bridge” before attempting a hardware clone:

1. Model the two domains, reset behavior, and address-map contracts. Use one-way queues and explicit ownership rather than a shared mutable memory model.
2. Implement the 40 × 24 text overlay and keyboard matrix as testable devices; preserve the original game image as a separate layer.
3. Replace cassette/printer internals with trace-backed adapters first. Define their timing and error states from captured observations instead of copying MAME's unfinished tape model.
4. Attach the current autonomous-controller project only through the bounded action mailbox. Log every observation, chosen action, gate decision, and resulting frame ID.
5. Use a hardware capture session to resolve the unknowns: shared-RAM width/arbitration, PROM function, CRTC timing, text-video mixing, reset sequencing, and cassette signalling.

## Open questions that must remain open

* Is shared RAM physically 16K × 10, and how exactly are its top two bits exposed to the 6502?
* Which timing and bus-arbitration behavior is implemented in the two small PROMs?
* Does the physical video path truly overlay text over STIC video as the emulator renderer does, or is the composition an emulator approximation?
* What are the exact cassette encoding, clocking, interrupt, and channel semantics?
* What hardware lives behind the driver’s “mystery” chip/CRT controller identification and how does it connect to the timing PROMs?
* Which parts of the 6502 firmware are necessary for a usable boot and which are product-specific BASIC support?

## Source map

* Machine configuration, ROM inventory, memory maps, and input matrix: `mattel/intv.cpp`.
* Shared-memory, cassette, keyboard, printer, reset, and interrupt handlers: `mattel/intv_m.cpp`.
* Text/game compositing: `mattel/intv_v.cpp`.
* Base STIC geometry and timing assumptions: `mattel/stic.h` and `mattel/stic.cpp`.
* Modular assembly/burn-in evidence: `docs/CCF10232011_00020.pdf`, notably PDF pages 9–10 (printed pages 5–6).
* Present autonomous-play and physical-harness observations: `video/keyboard-component-video.mp4`.
