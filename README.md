# APB Slave – SystemVerilog with Layered Class-Based Verification

An AMBA APB (Advanced Peripheral Bus) Slave controller implemented in
synthesizable SystemVerilog, verified using a complete layered class-based
testbench with constrained-random stimulus, self-checking scoreboard,
and slave error detection.

---

## 📌 Project Overview

This project implements an APB Slave (`apb_s`) conforming to the ARM AMBA
APB protocol. The slave supports read and write transactions to a 16-byte
internal memory, with built-in address range checking and slave error
(`pslverr`) reporting. The verification environment uses OOP-based
SystemVerilog classes modeled after UVM methodology.

---

## ✨ Design Features

- ✅ AMBA APB Slave compliant design
- ✅ 16-byte internal memory (`mem[16]`)
- ✅ Write and Read transactions with proper SETUP → ACCESS phase sequencing
- ✅ `pready` handshake signal support
- ✅ `pslverr` (slave error) generation for out-of-range addresses
- ✅ Active-low asynchronous reset (`presetn`)
- ✅ FSM-based control: `idle → write / read → idle`
- ✅ Address constraint: valid range 0–15 (`paddr[31:0]`)
- ✅ SystemVerilog interface for clean DUT–testbench connectivity

---

## 🗂️ File Structure

| File | Description |
|------|-------------|
| `design.sv` | APB Slave RTL + SystemVerilog interface (`abp_if`) |
| `testbench.sv` | Full layered verification environment (Gen → Drv → Mon → Sco) |

---

## 🔧 Module Interface
```verilog
module apb_s (
  input         pclk,       // APB clock
  input         presetn,    // Active-low asynchronous reset
  input  [31:0] paddr,      // 32-bit address bus
  input         psel,       // Slave select
  input         penable,    // Enable (ACCESS phase)
  input  [7:0]  pwdata,     // Write data
  input         pwrite,     // 1 = Write, 0 = Read
  output reg [7:0] prdata,  // Read data
  output reg    pready,     // Slave ready
  output        pslverr     // Slave error flag
);
```

---

## 🔄 APB Protocol – Transfer Phases
```
         SETUP Phase           ACCESS Phase
            │                      │
PCLK  ──────┼──────────────────────┼──────────
PSEL  ──────┤ HIGH ────────────────┤ HIGH ────
PENABLE─────┤ LOW  ────────────────┤ HIGH ────
PWRITE──────┤ set  ────────────────┤ held ────
PREADY──────┤  ─   ────────────────┤ HIGH ────  ← DUT responds
```

### FSM States
```
IDLE ──(psel=1, pwrite=1)──▶ WRITE ──(penable=1)──▶ IDLE
     ──(psel=1, pwrite=0)──▶ READ  ──(penable=1)──▶ IDLE
```

---

## 🧪 Verification Environment

Full layered class-based testbench with 4 components:
```
┌──────────────────────────────────────────────────┐
│                  ENVIRONMENT                     │
│                                                  │
│  ┌───────────┐  mbx   ┌─────────────────────┐   │
│  │ GENERATOR │───────▶│      DRIVER          │   │
│  │           │        │  Write: SETUP+ACCESS │   │
│  │ randc     │        │  Read:  SETUP+ACCESS │   │
│  │ pwrite    │        └─────────────────────┘   │
│  └─────┬─────┘                                  │
│        │ events (nextdrv / nextsco)              │
│        ▼                                        │
│  ┌───────────┐  mbx   ┌─────────────────────┐   │
│  │  MONITOR  │───────▶│    SCOREBOARD        │   │
│  │ pready=1  │        │  Write: store ref    │   │
│  │ capture   │        │  Read:  compare      │   │
│  └───────────┘        │  pslverr: log error  │   │
│                       └─────────────────────┘   │
└──────────────────────────────────────────────────┘
```

### Verification Components

| Component | Description |
|-----------|-------------|
| `transaction` | Randomized APB signals; `randc pwrite` ensures alternating R/W; address constrained to 0–15 |
| `generator` | Sends `count` randomized transactions; syncs with driver and scoreboard via events |
| `driver` | Drives proper APB SETUP → ACCESS phases for both write and read operations |
| `monitor` | Captures DUT outputs when `pready=1`; samples `prdata`, `pslverr` |
| `scoreboard` | Maintains a reference memory model; checks read data vs expected; tracks `pslverr`; reports mismatch count |
| `environment` | Wires all components; runs `pre_test()` (reset) → `test()` (fork-join_any) → `post_test()` (finish) |

### Constraints
```systemverilog
constraint addr_c  { paddr  >= 0; paddr  <= 15;  }   // Valid address range
constraint data_c  { pwdata >= 0; pwdata <= 255; }   // Full 8-bit data range
randc bit pwrite;                                     // Alternating R/W coverage
```

---

## ▶️ How to Simulate

Simulate on [EDA Playground](https://www.edaplayground.com/) or ModelSim/QuestaSim.

1. Clone the repository:
```bash
   git clone https://github.com/BinoyBabu10/APB.git
```
2. Load `design.sv` and `testbench.sv` into your simulator.
3. Set the transaction count (default: 20):
```systemverilog
   env.gen.count = 20;
```
4. Run and check logs for PASS/FAIL, slave errors, and final mismatch count.

### Sample Log Output
```
[DRV]: RESET DONE
-----------------------------
[GEN]: paddr:5  pwdata:172 pwrite:1 prdata:0 pslverr:0 @ 0
[DRV]: paddr:5  pwdata:172 pwrite:1 prdata:0 pslverr:0 @ 30000
[MON]: paddr:5  pwdata:172 pwrite:1 prdata:0 pslverr:0 @ 50000
[SCO]: DATA STORED DATA:172 ADDR:5
-------------------
[GEN]: paddr:5  pwdata:0   pwrite:0 prdata:0 pslverr:0 @ 60000
[MON]: paddr:5  pwdata:0   pwrite:0 prdata:172 pslverr:0 @ 90000
[SCO]: Data Matched
-------------------
-----Total number of Mismatch : 0----
```

---

## 🧠 Concepts Demonstrated

- ARM AMBA APB protocol (SETUP and ACCESS phase sequencing)
- FSM-based slave design in SystemVerilog
- `pslverr` slave error generation and detection
- Layered OOP verification: transaction, generator, driver, monitor, scoreboard
- `randc` for cycling through write/read alternation
- Scoreboard reference memory model for self-checking
- `pre_test / test / post_test` structured test execution
- VCD waveform dumping for post-simulation debug

---

## 🛠️ Tools & Technologies

- **Language:** SystemVerilog
- **Protocol:** AMBA APB (Advanced Peripheral Bus)
- **Verification Style:** Layered Class-Based Testbench (UVM-inspired)
- **Simulator:** EDA Playground / ModelSim / QuestaSim
- **Waveform:** VCD (`dump.vcd`)

---

## 👤 Author

**Binoy Babu**  
[GitHub Profile](https://github.com/BinoyBabu10)
