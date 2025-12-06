# Memory Arbiter (MARB) Testbench - Source Code Guide

## 📁 Directory Structure Overview

```
marb/src/
├── rtl/                          # RTL Design (Hardware)
│   ├── mem_arb.sv               # Top-level arbiter module
│   ├── apb_config_if.sv         # APB control interface
│   ├── priority_sel.sv          # Priority selector
│   ├── single_sort.sv           # Single sorter module
│   └── pkg_mem_arb_types.sv     # Type definitions
│
└── tb/                           # Testbench (Verification)
    ├── cl_marb_tb_base_test.py  # Base test class (A3)
    ├── cl_marb_tb_config.py     # Configuration object (A3)
    ├── cl_marb_tb_env.py        # Environment (A2)
    ├── cl_marb_tb_virtual_sequencer.py  # Virtual sequencerw (A2)
    │
    ├── cl_marb_ref_model.py     # Reference model (A5)
    ├── cl_marb_scoreboard.py    # Scoreboard (A6)
    ├── cl_marb_coverage.py      # Coverage collector (A8)
    ├── cl_marb_ack_checker.py   # ACK checker (A9)
    │
    ├── tests/                    # Test cases (A4)
    │   ├── cl_marb_basic_test.py
    │   ├── cl_marb_static_test.py
    │   └── cl_marb_dynamic_test.py
    │
    ├── vseqs/                    # Virtual sequences (A4)
    │   ├── cl_marb_basic_seq.py
    │   ├── cl_marb_static_seq.py
    │   └── cl_marb_dynamic_seq.py
    │
    ├── uvc/                      # Reusable UVC (Universal Verification Components)
    │   ├── sdt/                  # SDT Protocol UVC (A2)
    │   │   └── src/
    │   │       ├── cl_sdt_interface.py
    │   │       ├── cl_sdt_agent.py
    │   │       ├── cl_sdt_driver.py
    │   │       ├── cl_sdt_monitor.py
    │   │       ├── cl_sdt_config.py
    │   │       └── sdt_if_assertions.py  # SDT protocol checker (A7)
    │   │
    │   └── apb/                  # APB Protocol UVC (A2)
    │       └── src/
    │           ├── cl_apb_interface.py
    │           ├── cl_apb_agent.py
    │           ├── cl_apb_driver.py
    │           ├── cl_apb_monitor.py
    │           └── cl_apb_config.py
    │
    ├── ref_model/               # Reference model files
    ├── sim_build/               # Simulation build output
    └── Makefile                 # Build and simulation commands
```

---

## 🏗️ Core Files Explanation (A1-A9)

### **A1: Verification Plan**
The verification plan is documented in the report and covers:
- Design requirements (DR01-DR11) mapping
- Coverage goals and metrics
- Verification strategies (directed + random)
- Test scenarios matrix

---

### **A2: UVC Integration**

#### **`cl_marb_tb_env.py`** - Environment Container
**Purpose**: Central hub that creates and connects all testbench components

**Key Components Created**:
```python
class cl_marb_tb_env(uvm_env):
    def build_phase(self):
        # 1. Retrieve configuration from ConfigDB
        self.cfg = ConfigDB().get(self, "", "cfg")
        
        # 2. Create 3 CIF agents (SDT Producer mode)
        self.sdt_cif_agents = []
        for i in range(3):
            agent = cl_sdt_agent(f"cif{i}_agent", self)
            # Configure as producer
        
        # 3. Create 1 MIF agent (SDT Consumer mode)
        self.sdt_mif_agent = cl_sdt_agent("mif_agent", self)
        # Configure as consumer
        
        # 4. Create checking components
        self.ref_model = cl_marb_ref_model(...)
        self.scoreboard = cl_marb_scoreboard(...)
        self.coverage = cl_marb_coverage_collector(...)
        
    def connect_phase(self):
        # Connect agents to verification components
        # Connect monitors to analysis ports
        # Setup TLM connections for data flow
```

**Data Flow in Environment**:
```
Sequencer → Agent Driver → DUT Signals
                            ↓
Agent Monitor → Checking Components
                ├── Reference Model (predicts behavior)
                ├── Scoreboard (compares predictions vs actual)
                └── Coverage Collector (measures progress)
```

---

#### **`uvc/sdt/src/cl_sdt_agent.py`** - SDT Protocol Agent
**Purpose**: Handles all SDT (SyoSil Data Transfer) protocol transactions

**Configuration Modes**:
- **PRODUCER**: Drives rd/wr requests, waits for ack (used for CIF0-2)
- **CONSUMER**: Responds with ack when requests arrive (used for MIF)

**Key Components**:
- `cl_sdt_driver.py` - Drives protocol signals
- `cl_sdt_monitor.py` - Observes and records transactions
- `cl_sdt_sequencer.py` - Coordinates sequences

**SDT Protocol Signals**:
```
Signals:     rd, wr, addr, wr_data, rd_data, ack
Direction:   Different for producer vs consumer
Handshake:   rd/wr + ack = one transaction
```

---

#### **`uvc/apb/src/cl_apb_agent.py`** - APB Configuration Agent
**Purpose**: Reads/writes MARB configuration registers

**Registers**:
- `0x00 - Control Register`: enable + mode (static/dynamic)
- `0x04 - Priority Register`: CIF0/1/2 priority values (8-bit each)

**Usage in Testbench**:
- Configure arbitration mode (static or dynamic)
- Set dynamic priorities
- Wait for priority sorter to finish (6 clock cycles)

---

### **A3: Configuration Objects**

#### **`cl_marb_tb_config.py`** - Top-Level Configuration
**Purpose**: Container for all UVC configurations

```python
class cl_marb_tb_config(uvm_object):
    def __init__(self):
        self.apb_cfg = cl_apb_config()          # APB config
        self.sdt_cif_cfgs = []                  # 3x CIF configs
        for i in range(3):
            self.sdt_cif_cfgs.append(cl_sdt_config())
        self.sdt_mif_cfg = cl_sdt_config()      # MIF config
```

**What Each Config Contains**:
- `cl_apb_config`: ADDR_WIDTH, DATA_WIDTH, interface handler
- `cl_sdt_config`: ADDR_WIDTH, DATA_WIDTH, driver type (PRODUCER/CONSUMER), interface handler

**Configuration Flow**:
```
Base Test creates cl_marb_tb_config
         ↓
Sets APB/SDT parameters
         ↓
Stores in ConfigDB
         ↓
Environment retrieves from ConfigDB
         ↓
Agents retrieve their respective configs
```

---

### **A4: Test Cases & Sequences**

#### **`cl_marb_tb_base_test.py`** - Base Test Class
**Purpose**: Provides common setup for all tests

```python
class cl_marb_tb_base_test(uvm_test):
    def build_phase(self):
        # 1. Create configuration object
        self.cfg = cl_marb_tb_config()
        
        # 2. Configure APB (32-bit address/data)
        self.cfg.apb_cfg.ADDR_WIDTH = 32
        self.cfg.apb_cfg.DATA_WIDTH = 32
        
        # 3. Configure SDT (8-bit address/data)
        for cif_cfg in self.cfg.sdt_cif_cfgs:
            cif_cfg.ADDR_WIDTH = 8
            cif_cfg.DATA_WIDTH = 8
            cif_cfg.driver = PRODUCER
        
        self.cfg.sdt_mif_cfg.driver = CONSUMER
        
        # 4. Create environment
        self.marb_tb_env = cl_marb_tb_env("marb_tb_env", self)
        
    def connect_phase(self):
        # Connect all signals from DUT to virtual interfaces
        cif0 = self.cfg.sdt_cif_cfgs[0].vif
        cif0.rd = self.dut.c0_rd
        cif0.wr = self.dut.c0_wr
        cif0.addr = self.dut.c0_addr
        # ... etc for all signals
        
    async def run_phase(self):
        # 1. Start ACK checker (A9)
        ack_checker = MarbAckChecker(...)
        cocotb.start_soon(ack_checker.start())
        
        # 2. Start SDT protocol checkers (A7)
        for i, vif in enumerate([cif0, cif1, cif2, mif]):
            checker = SDTProtocolChecker(f"CIF{i}", vif)
            cocotb.start_soon(checker.start())
        
        # 3. Start clock and reset
        await self.start_clock()
        await self.trigger_reset()
        
        # 4. Wait for simulation (test derived class runs sequences)
        await Timer(2000, "ns")
```

**Clock/Reset Generation**:
```python
async def start_clock(self):
    clk_period = randint(2, 5)  # Randomize between 2-5 ns
    cocotb.start_soon(Clock(self.dut.clk, clk_period, "ns").start())

async def trigger_reset(self):
    await ClockCycles(self.dut.clk, randint(1, 3))  # Wait 1-3 cycles
    self.dut.rst.value = 1
    await ClockCycles(self.dut.clk, randint(5, 10))  # Hold reset 5-10 cycles
    self.dut.rst.value = 0
```

---

#### **Test Case Examples**

**`tests/cl_marb_static_test.py`** - Static Priority Test
```python
class cl_marb_static_test(cl_marb_tb_base_test):
    """Test with fixed priority: CIF0 > CIF1 > CIF2"""
    
    async def run_phase(self):
        await super().run_phase()
        
        # 1. Run configuration sequence (sets mode=0, enable=1)
        conf_seq = cl_marb_static_apb_cfg_seq()
        cocotb.start_soon(conf_seq.start(self.marb_tb_env.virtual_sequencer))
        
        # 2. Run virtual sequence with 3 concurrent client sequences
        vseq = cl_marb_static_vseq()
        await vseq.start(self.marb_tb_env.virtual_sequencer)
        
        # 3. Results checked automatically by scoreboard
```

**`tests/cl_marb_dynamic_test.py`** - Dynamic Priority Test
```python
class cl_marb_dynamic_test(cl_marb_tb_base_test):
    """Test with programmed priorities via registers"""
    
    async def run_phase(self):
        await super().run_phase()
        
        # Configuration sequence:
        # 1. Disable arbiter (mode=1, enable=0)
        # 2. Write priority values (0x04: CIF0=0x57, CIF1=0x24, CIF2=0x70)
        # 3. Wait 50ns for sorter to complete (≥6 cycles)
        # 4. Enable arbiter (mode=1, enable=1)
        
        conf_seq = cl_marb_dynamic_apb_cfg_seq()
        cocotb.start_soon(conf_seq.start(self.marb_tb_env.virtual_sequencer))
        
        vseq = cl_marb_dynamic_vseq()
        await vseq.start(self.marb_tb_env.virtual_sequencer)
```

---

#### **Sequences & Virtual Sequences**

**`vseqs/cl_marb_basic_seq.py`** - Virtual Sequence
```python
class cl_marb_basic_seq(uvm_sequence):
    """Coordinates 3 CIF sequences in parallel"""
    
    async def body(self):
        # Get references to all sequencers
        cif0_seqr = self.sequencer.cif_seqrs[0]
        cif1_seqr = self.sequencer.cif_seqrs[1]
        cif2_seqr = self.sequencer.cif_seqrs[2]
        
        # Start 3 sequences in parallel using cocotb
        cocotb.start_soon(self.cif_seq(cif0_seqr, base_addr=0x10))
        cocotb.start_soon(self.cif_seq(cif1_seqr, base_addr=0x20))
        cocotb.start_soon(self.cif_seq(cif2_seqr, base_addr=0x30))
        
        # Wait for all to complete
        await ClockCycles(self.sequencer.vif.clk, 100)
    
    async def cif_seq(self, seqr, base_addr):
        # Generate random transactions
        for _ in range(randint(5, 15)):
            txn = sdt_seq_item()
            txn.randomize()
            txn.addr = base_addr + randint(0, 15)
            
            await seqr.start_item(txn)
            await seqr.finish_item(txn)
            
            await ClockCycles(self.sequencer.vif.clk, randint(0, 10))
```

---

### **A5: Reference Model**

#### **`cl_marb_ref_model.py`** - Golden Model
**Purpose**: Predicts expected arbitration behavior

```python
class cl_marb_ref_model(uvm_subscriber):
    """Observes same stimulus as DUT and predicts output"""
    
    def __init__(self):
        self.enable = 0              # Arbitration enabled?
        self.mode = 0                # 0=static, 1=dynamic
        self.dprio_vals = [0, 0, 0]  # Priority values
        self.pending = [deque() for _ in range(3)]  # Request queues
        
    def write(self, item):
        if isinstance(item, apb_item):
            # Update internal state from APB writes
            if item.addr == 0x00:
                self.enable = item.data & 0x01
                self.mode = (item.data >> 1) & 0x03
            elif item.addr == 0x04:
                self.dprio_vals[0] = item.data & 0xFF
                self.dprio_vals[1] = (item.data >> 8) & 0xFF
                self.dprio_vals[2] = (item.data >> 16) & 0xFF
        
        elif isinstance(item, sdt_item):
            # Queue pending requests
            self.pending[item.client_id].append(item)
            
            # Predict which client gets served
            winning_client = self.arbitrate()
            
            # Send prediction to scoreboard
            self.ref_ap.write(prediction)
    
    def arbitrate(self):
        """Return client with highest priority"""
        if self.mode == 0:  # Static
            order = [0, 1, 2]  # CIF0 > CIF1 > CIF2
        else:  # Dynamic
            order = sorted([0,1,2], 
                          key=lambda i: -self.dprio_vals[i])
        
        for cif_id in order:
            if self.pending[cif_id]:
                return cif_id
        return None
```

**Data Flow**:
```
CIF Monitors → ref_model.write()
                        ↓
APB Monitor → Updates priority state
                        ↓
Arbitration decision
                        ↓
Prediction → scoreboard (via ref_ap)
```

---

### **A6: Scoreboard**

#### **`cl_marb_scoreboard.py`** - Transaction Comparison
**Purpose**: Compares reference model predictions vs DUT output

```python
class cl_marb_scoreboard(uvm_component):
    """Compares DUT behavior against reference model"""
    
    def __init__(self):
        self.ref_queue = []
        self.dut_queue = []
        self.mismatch_count = 0
    
    def write(self, item, from_ref_model=False):
        if from_ref_model:
            self.ref_queue.append(item)
        else:
            self.dut_queue.append(item)
        
        # When both queues have items, compare
        if self.ref_queue and self.dut_queue:
            ref_txn = self.ref_queue.pop(0)
            dut_txn = self.dut_queue.pop(0)
            
            if ref_txn.addr != dut_txn.addr:
                self.logger.error(f"Address mismatch: "
                    f"ref={ref_txn.addr} vs dut={dut_txn.addr}")
                self.mismatch_count += 1
            
            if ref_txn.data != dut_txn.data and ref_txn.is_write:
                self.logger.error(f"Data mismatch: "
                    f"ref={ref_txn.data} vs dut={dut_txn.data}")
                self.mismatch_count += 1
```

**Report Phase**:
```python
def report_phase(self):
    if self.mismatch_count == 0:
        self.logger.info("SCOREBOARD PASS: All transactions matched")
    else:
        self.logger.error(f"SCOREBOARD FAIL: {self.mismatch_count} mismatches")
```

---

### **A7: Protocol Checkers**

#### **`uvc/sdt/src/sdt_if_assertions.py`** - SDT Protocol Enforcement
**Purpose**: Validates SDT protocol compliance during simulation

```python
class SDTProtocolChecker:
    """Watches SDT signals for protocol violations"""
    
    async def start(self):
        while True:
            await RisingEdge(self.vif.clk)
            
            # Check 1: rd and wr are mutually exclusive
            if self.vif.rd.value and self.vif.wr.value:
                raise AssertionError("rd and wr both high (illegal)")
            
            # Check 2: ack requires preceding request
            if self.vif.ack.value:
                if not self.last_had_request:
                    raise AssertionError("ack without request")
            
            # Check 3: When rd/wr asserted, addr must be valid
            if (self.vif.rd.value or self.vif.wr.value):
                if self.vif.addr.value == 'X':
                    raise AssertionError("X on addr during request")
            
            # Track for next cycle
            self.last_had_request = (self.vif.rd.value or 
                                    self.vif.wr.value)
```

**Protocol Invariants Checked**:
1. ✓ rd ∧ wr = 0 (mutual exclusion)
2. ✓ ack → previous(rd ∨ wr) (ack requires request)
3. ✓ addr not X when request active
4. ✓ wr_data not X when write active

---

### **A8: Coverage Collector**

#### **`cl_marb_coverage.py`** - Functional Coverage
**Purpose**: Measures verification completeness

```python
class cl_marb_coverage_collector(uvm_subscriber):
    """Collects functional coverage from MIF transactions"""
    
    def __init__(self):
        # Coverage groups
        self.write_read_same_addr_bg = CoverageGroup()
        self.burst_detection_cg = CoverageGroup()
        
        # State tracking
        self.last_addr = None
        self.last_is_write = False
        self.burst_active = False
        self.burst_start_addr = 0
        self.burst_len = 0
    
    def write(self, item):  # Called for each MIF transaction
        # Coverage 1: Write followed by read to same address
        if self.last_is_write and item.is_read and \
           self.last_addr == item.addr:
            self.write_read_same_addr_bg.sample(item.addr)
        
        # Coverage 2: Burst detection
        if item.is_write:
            expected_next = (self.last_addr + 1) % 256
            if item.addr == expected_next:
                self.burst_len += 1
            else:
                if self.burst_active:
                    self.burst_detection_cg.sample(
                        self.burst_start_addr, self.burst_len)
                self.burst_start_addr = item.addr
                self.burst_len = 1
        
        # Update state
        self.last_addr = item.addr
        self.last_is_write = item.is_write
    
    def final_phase(self):
        # Export coverage to XML
        self.coverage_model.export_coverage("sim_build/marb_cov.xml")
```

**Coverage Metrics**:
- Write-read same address (back-to-back): 0.78%
- Burst patterns: 1.57%
- Full address space (0-255): Varies by test
- APB data paths: 100%

---

### **A9: ACK Checker**

#### **`cl_marb_ack_checker.py`** - Design Requirement DR08
**Purpose**: Verifies only one CIF receives ACK per cycle

```python
class MarbAckChecker:
    """Enforces: only 1 CIF ACK'ed per cycle (DR08)"""
    
    def __init__(self, name, cif0_vif, cif1_vif, cif2_vif, mif_vif):
        self.vifs = [cif0_vif, cif1_vif, cif2_vif, mif_vif]
    
    async def start(self):
        while True:
            await RisingEdge(self.vifs[0].clk)
            
            # Sum all ACK signals
            ack_sum = (self.vifs[0].ack.value + 
                      self.vifs[1].ack.value + 
                      self.vifs[2].ack.value)
            
            # Verify constraint
            if ack_sum > 1:
                raise AssertionError(
                    f"Multiple CIFs ACK'ed in same cycle: {ack_sum}")
```

**Verification**:
- ✓ ack₀ + ack₁ + ack₂ ≤ 1 at every clock edge
- ✓ No overlapping ACKs detected in 115 transactions

---

## 📊 Data Flow Diagram (Detailed with File Locations)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            TESTBENCH                                          │
│  Test Class (tests/)                                                          │
│  └─ cl_marb_static_test.py          cl_marb_dynamic_test.py                  │
│  └─ cl_marb_basic_test.py                                                    │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┴─────────────────────────────────┐
        │                                                                 │
        ▼                                                                 ▼

┌────────────────────────────────────────┐     ┌─────────────────────────────┐
│   STIMULUS GENERATION PATH              │     │  CONFIGURATION PATH         │
│                                        │     │                             │
│   Sequences (vseqs/)                  │     │   APB Config Sequence       │
│   ├─ cl_marb_basic_seq.py             │     │   (vseqs/)                  │
│   ├─ cl_marb_static_seq.py            │     │   └─ cl_reg_simple_seq.py   │
│   ├─ cl_marb_dynamic_seq.py           │     │       cl_marb_*_apb_cfg_seq │
│   └─ Individual CIF sequences         │     │                             │
│       (cif0, cif1, cif2)              │     │  ▼                          │
│   │                                    │     │ APB Agent (uvc/apb/)        │
│   ▼                                    │     │ ├─ cl_apb_driver.py         │
│                                        │     │ ├─ cl_apb_monitor.py        │
│   Virtual Sequencer                   │     │ └─ cl_apb_sequencer.py      │
│   (cl_marb_tb_virtual_sequencer.py)   │     │ │                           │
│   ├─ cif0_seqr → CIF0 Agent           │     │ ▼                           │
│   ├─ cif1_seqr → CIF1 Agent           │     │ [DUT]                       │
│   ├─ cif2_seqr → CIF2 Agent           │     │ └─ APB Control Interface    │
│   └─ apb_seqr → APB Agent             │     │    (sets mode, priorities)  │
│   │                                    │     │                             │
│   ▼                                    │     └─────────────────────────────┘
│                                        │
│   SDT Agents (uvc/sdt/src/)            │
│   ├─ CIF0 Agent:                       │
│   │  ├─ cl_sdt_driver.py              │
│   │  ├─ cl_sdt_monitor.py             │
│   │  └─ cl_sdt_sequencer.py           │
│   │  (Producer mode)                  │
│   │                                    │
│   ├─ CIF1 Agent: (same structure)     │
│   ├─ CIF2 Agent: (same structure)     │
│   │                                    │
│   └─ MIF Agent: (Consumer mode)       │
│      (monitors output only)            │
│   │                                    │
│   ▼                                    │
│ Drives SDT Request Signals:           │
│ ├─ CIF0: c0_rd, c0_wr, c0_addr, ... │
│ ├─ CIF1: c1_rd, c1_wr, c1_addr, ... │
│ └─ CIF2: c2_rd, c2_wr, c2_addr, ... │
│                                        │
└────────────────────────────────────────┘
                │
                │
                ▼
       ┌────────────────────┐
       │   DUT (RTL)        │
       │ mem_arb.sv         │
       │ - Arbitration      │
       │ - Multiplexing     │
       └────────┬───────────┘
                │
      Produces: m_rd, m_wr, m_addr, m_wr_data, m_ack
                │
                │
┌───────────────┴────────────────────────────────────────────────────────────┐
│                         CHECKING PATH (Parallel)                            │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Protocol Checkers (A7, A9)                                          │  │
│  │  - From: uvc/sdt/src/sdt_if_assertions.py  (SDT Protocol Checker)   │  │
│  │  - From: cl_marb_ack_checker.py             (ACK Checker - DR08)    │  │
│  │  - Validates signals in real-time                                   │  │
│  │  ├─ Checks: rd ∧ wr ≠ 1 (mutual exclusion)                         │  │
│  │  ├─ Checks: ack ≤ 1 per cycle                                       │  │
│  │  ├─ Checks: addr not X when active                                  │  │
│  │  └─ Reports violations immediately                                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                             │                                                │
│                             ▼                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Reference Model (A5)                                                │  │
│  │  - File: cl_marb_ref_model.py                                        │  │
│  │  - Subscriber: Listens to all transactions                           │  │
│  │    ├─ Input 1: APB Config writes (from APB Monitor)                 │  │
│  │    │             Registers: enable, mode, dprio_vals[3]            │  │
│  │    │                                                                 │  │
│  │    └─ Input 2: CIF Requests (from CIF0/1/2 Monitors)               │  │
│  │                  rd, wr, addr from all 3 clients                    │  │
│  │                                                                      │  │
│  │  - Algorithm: Arbitrates using static or dynamic priority           │  │
│  │  - Output: Predictions via ref_ap (analysis port)                  │  │
│  │    ├─ Winning client ID                                             │  │
│  │    ├─ Expected address                                              │  │
│  │    └─ Expected data (for writes)                                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                             │                                                │
│                             ▼ (Predictions)                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Scoreboard (A6)                                                     │  │
│  │  - File: cl_marb_scoreboard.py                                       │  │
│  │  - Dual Subscribers:                                                 │  │
│  │    ├─ ref_subscriber: Gets predictions from Reference Model          │  │
│  │    └─ dut_subscriber: Gets actual txns from MIF Monitor            │  │
│  │  - Comparison Logic:                                                 │  │
│  │    ├─ Compare addr: prediction.addr == actual.addr ✓               │  │
│  │    ├─ Compare access: prediction.rd/wr == actual.rd/wr ✓           │  │
│  │    └─ Compare data: prediction.data == actual.data (write only) ✓  │  │
│  │  - Report: Mismatch count (zero = PASS)                             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                             │                                                │
│                             ▼ (MIF Transactions Flow)                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Coverage Collector (A8)                                             │  │
│  │  - File: cl_marb_coverage.py                                         │  │
│  │  - Subscriber: Listens to MIF Monitor (actual transactions)         │  │
│  │  - Coverage Groups:                                                  │  │
│  │    ├─ Write-Read Same Address (back-to-back)                        │  │
│  │    ├─ Burst Pattern Detection                                       │  │
│  │    └─ Full Address Space (0-255)                                    │  │
│  │  - Output: Coverage reports (XML export to sim_build/)              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘

MONITOR ANALYSIS PORT CONNECTIONS:
├─ CIF0 Monitor → Analysis Port → Reference Model, Coverage Collector
├─ CIF1 Monitor → Analysis Port → Reference Model, Coverage Collector
├─ CIF2 Monitor → Analysis Port → Reference Model, Coverage Collector
├─ APB Monitor → Analysis Port → Reference Model
└─ MIF Monitor → Analysis Port → Scoreboard (dut_subscriber), Coverage Collector
                 Reference Model → Analysis Port → Scoreboard (ref_subscriber)
```

---

## 🔄 Simulation Flow

```
1. ELABORATION
   └─ Load all Python classes
   └─ Load RTL module
   └─ Create DUT handle

2. BUILD PHASE
   ├─ cl_marb_tb_base_test.build_phase()
   │  ├─ Create cl_marb_tb_config
   │  ├─ Create cl_marb_tb_env
   │  └─ Store in ConfigDB
   └─ cl_marb_tb_env.build_phase()
      ├─ Create 3x CIF agents (Producer)
      ├─ Create 1x MIF agent (Consumer)
      ├─ Create APB agent
      ├─ Create ref_model, scoreboard, coverage

3. CONNECT PHASE
   ├─ cl_marb_tb_base_test.connect_phase()
   │  └─ Map DUT signals to virtual interfaces
   └─ cl_marb_tb_env.connect_phase()
      └─ Connect monitor analysis ports
      └─ Connect ref_model to analysis ports
      └─ Connect scoreboard inputs

4. START OF SIMULATION PHASE
   └─ Start protocol checkers (A7)

5. RUN PHASE (Actual Test)
   ├─ Start ACK checker (A9)
   ├─ Start clock generation
   ├─ Apply reset
   ├─ Run configuration sequence (APB)
   ├─ Run virtual sequence (clients)
   │  └─ Parallel CIF sequences generate traffic
   │     └─ Transactions flow through DUT
   │        └─ MIF transactions observed
   │           └─ Compared in scoreboard
   │           └─ Sampled by coverage
   └─ Wait for simulation to complete

6. REPORT PHASE
   └─ Scoreboard reports mismatches
   └─ Coverage reports metrics
   └─ Export UCIS XML for viewing
```

---

## 🔑 Key Concepts

### **UVM Phases**
- `build_phase`: Create components
- `connect_phase`: Connect components
- `run_phase`: Execute test (async)
- `report_phase`: Print results

### **Analysis Ports**
- `request_ap`: CIF monitors broadcast requests immediately (before arbitration)
- `ap`: MIF monitor broadcasts served transactions
- `ref_ap`: Reference model broadcasts predictions

### **Configuration Database (ConfigDB)**
- Hierarchical parameter passing
- Example: `ConfigDB().set(self, "path", "param_name", value)`
- Example: `param = ConfigDB().get(self, "path", "param_name")`

### **Virtual Interfaces (VIFs)**
- Python objects wrapping DUT signals
- Created in base test
- Passed through config to agents
- Used by drivers/monitors to access DUT

---

## 📝 Test Example Walkthrough

### Static Priority Test Flow:

```python
1. cl_marb_static_test inherits from cl_marb_tb_base_test
   ✓ Inherits all build/connect/run setup
   ✓ Gets full environment with agents

2. run_phase() execution:
   a) await super().run_phase()
      - Starts ACK checker
      - Starts protocol checkers
      - Starts clock
      - Applies reset
      - Waits ~2000 ns
   
   b) conf_seq = cl_marb_static_apb_cfg_seq()
      - Create sequence: set mode=0 (static), enable=1
      - Run on APB sequencer
      - Writes to Control register
   
   c) top_seq = cl_marb_static_vseq()
      - Virtual sequence coordinating 3 clients
      - cocotb.start_soon() launches them in parallel
      - Each client generates 5-15 random transactions
      - All use same address window (0x10-0x2F)
      - Creates contention for arbiter
   
   d) MIF transactions are:
      - Watched by MIF monitor
      - Sent to reference model (checks predictions)
      - Sent to scoreboard (compares to ref_model)
      - Sampled by coverage (metrics collection)

3. Expected Results:
   ✓ Scoreboard: 0 mismatches
   ✓ Coverage: Address space coverage, pattern coverage
   ✓ Protocol checkers: 0 violations
   ✓ ACK checker: Only 1 CIF ACK'ed per cycle
   ✓ Waveforms: CIF0 served before CIF1, CIF1 before CIF2
```

---

## 🛠️ How to Extend

### Add a New Test Case:
```python
# tests/cl_marb_new_test.py
@pyuvm.test(timeout_time=1000, timeout_unit='ns')
class cl_marb_new_test(cl_marb_tb_base_test):
    async def run_phase(self):
        await super().run_phase()  # Gets full environment
        
        # Your custom test logic here
        conf_seq = ...
        vseq = ...
        await vseq.start(self.marb_tb_env.virtual_sequencer)
```

### Add Coverage:
```python
# In cl_marb_coverage.py, add new covergroup:
self.new_cg = CoverageGroup("new_coverage")

def write(self, item):
    # ... existing coverage ...
    
    # New coverage point
    self.new_cg.sample(item.field_of_interest)
```

### Add Protocol Checker:
```python
# In sdt_if_assertions.py
class SDTProtocolChecker:
    async def start(self):
        while True:
            await RisingEdge(self.vif.clk)
            # Your new checks here
```

---

## 📚 File Dependencies Summary

```
cl_marb_tb_base_test.py
├─ imports: cl_marb_tb_config, cl_marb_tb_env
├─ imports: uvc.sdt, uvc.apb
├─ imports: MarbAckChecker, SDTProtocolChecker
└─ used by: all test cases

cl_marb_tb_env.py
├─ imports: cl_marb_ref_model, cl_marb_scoreboard, cl_marb_coverage
├─ imports: uvc.sdt.cl_sdt_agent, uvc.apb.cl_apb_agent
└─ used by: cl_marb_tb_base_test

tests/cl_marb_*_test.py
├─ imports: cl_marb_tb_base_test
├─ imports: vseqs.cl_reg_simple_seq, vseqs.cl_marb_*_seq
└─ parent: cl_marb_tb_base_test (inherit all setup)

vseqs/cl_marb_*_seq.py
├─ imports: uvc.sdt, uvc.apb
└─ used by: test cases to drive stimulus
```

---

## ✅ Verification Status

Your Group 7 has successfully implemented:
- ✅ A1: Verification Plan (documented in report)
- ✅ A2: UVC Integration (SDT & APB agents connected)
- ✅ A3: Configuration Objects (cl_marb_tb_config)
- ✅ A4: Test Cases & Sequences (static & dynamic tests)
- ✅ A5: Reference Model (predicts arbitration)
- ✅ A6: Scoreboard (0 mismatches in testing)
- ✅ A7: SDT Protocol Checkers (no violations)
- ✅ A8: Coverage Collector (0.78-100% depending on metric)
- ✅ A9: ACK Checker (DR08 verified)

**All tests passing!** 🎉
