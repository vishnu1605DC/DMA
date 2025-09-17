# DMA Controller (3-Channel, AHB-Lite, Burst Mode)

## 📖 Overview

This project implements a 3-channel Direct Memory Access (DMA) Controller with AHB-Lite master interface. The DMA allows high-speed data transfers between memory and peripherals without CPU intervention, supporting burst-only mode for efficiency.

### The design includes:
- Source/Destination address registers
- Transfer count registers
- Control and Status registers
- Mask and Request registers for channel management
- Round-robin arbiter for fair channel allocation

## 🏗️ Block Diagram
```
                +-------------------+
                |                   |
   CPU <------> |   AHB-Lite Slave  |
                | (Register Access) |
                +-------------------+
                         |
                         v
                +-------------------+
                | DMA Core          |
                |                   |
                |  +-------------+  |
 DRQ0 --------> |  |             |  | ---> DACK0
 DRQ1 --------> |  | Arbiter     |  | ---> DACK1
 DRQ2 --------> |  | (Round-     |  | ---> DACK2
                |  |  Robin)     |  |
                |  +-------------+  |
                |                   |
                | +----------------+|
                | | Address Logic  ||
                | | & Incrementor  ||
                | +----------------+|
                |                   |
                | +----------------+|
                | | Transfer Engine||
                | +----------------+|
                +-------------------+
                         |
                         v
                +-------------------+
                |   AHB-Lite Master |
                | (Memory Access)   |
                +-------------------+
```

## ⚙️ Features
- 3 independent DMA channels (Channel 0–2)
- Burst mode transfers only (address auto-increment by 4 bytes)
- Round-robin arbitration for fair channel access
- AHB-Lite interface:
  - Slave: register programming (CPU access)
  - Master: memory read/write operations
- Memory-mapped registers for configuration and status
- Transfer completion flag in status register
- Simple simulation memory included (`memory[]`)

## 📑 Register Map
| Address       | Register             | Width | Scope   | Description                      |
|-------------- |--------------------- |-------|---------|----------------------------------|
| 0x4000_0000   | Source Address       | 32    | Channel | Holds the starting source address |
| 0x4000_0004   | Destination Addr     | 32    | Channel | Holds the starting destination addr |
| 0x4000_0008   | Transfer Size        | 32    | Channel | Number of words to transfer       |
| 0x4000_000C   | Control              | 8     | Channel | Bit[0] = Enable transfer          |
| 0x4000_0010   | Status               | 8     | Channel | Bit[0] = Transfer Complete        |
| 0x4000_0014   | Request Register     | 4     | Global  | Pending requests from channels    |
| 0x4000_0018   | Mask Register        | 4     | Global  | Masks out specific channel reqs   |
| 0x4000_001C   | Command Register     | 8     | Global  | Optional global DMA commands      |

## 🛠️ How It Works

### CPU Configuration
- CPU writes `src_addr`, `dst_addr`, and `trans_size`.
- CPU enables DMA by writing 1 to `ctrl[0]`.

### Request Phase
- A device raises `dmac_req[i]` when it needs service.
- Arbiter grants channel in round-robin order, asserts `dmac_ack[i]`.

### Transfer Phase
- DMA reads from `src_addr` and writes to `dst_addr`.
- Both addresses increment by 4 bytes per transfer.
- `trans_size` decrements after each word.

### Completion
- When `trans_size == 0`, DMA clears `ctrl[0]` and sets `status[0]`.
- CPU can poll or check interrupt (if extended).

## 🔄 Round-Robin Arbiter
- Ensures fair channel access among `dmac_req[0..2]`.
- Maintains a 2-bit pointer (`arb_ptr`):
  - After granting one channel, moves to next.
  - Wraps around after channel 2 → channel 0.
- Guarantees no channel starvation.

## 📡 AHB-Lite Usage
- **Slave side** (`CS`, `HWRITE`, `HADDR`, `HWDATA`, `HRDATA`):
  - Used by CPU to configure registers (memory-mapped).
- **Master side** (internal FSM):
  - Generates `HADDR`, `HWRITE`, `HSIZE`, `HBURST`, `HTRANS` signals.
  - Performs incrementing bursts.
  - Uses internal burst counter to track progress.

## 🚀 Example Usage
```
CPU configures:
  src_addr = 0x2000_0000
  dst_addr = 0x2000_1000
  trans_size = 16
  ctrl[0] = 1
Peripheral requests DMA (sets dmac_req[0]).
Arbiter grants channel 0 → DMA copies 16 words.
After last transfer:
  status[0] = 1
  ctrl[0] = 0 (DMA auto-disabled)
```
