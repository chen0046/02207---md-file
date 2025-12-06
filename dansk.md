# Memory Arbiter (MARB) Testbench - Kildekodeguide

## 📁 Mappestruktur Oversigt

```
marb/src/
├── rtl/                          # RTL Design (Hardware)
│   ├── mem_arb.sv               # Top-level arbiter modul
│   ├── apb_config_if.sv         # APB kontrol interface
│   ├── priority_sel.sv          # Prioritetsvelger
│   ├── single_sort.sv           # Single sorter modul
│   └── pkg_mem_arb_types.sv     # Type definitioner
│
└── tb/                           # Testbench (Verifikation)
    ├── cl_marb_tb_base_test.py  # Base test klasse (A3)
    ├── cl_marb_tb_config.py     # Konfigurationsobjekt (A3)
    ├── cl_marb_tb_env.py        # Miljø (A2)
    ├── cl_marb_tb_virtual_sequencer.py  # Virtual sekvens (A2)
    │
    ├── cl_marb_ref_model.py     # Reference model (A5)
    ├── cl_marb_scoreboard.py    # Scoreboard (A6)
    ├── cl_marb_coverage.py      # Coverage indsamler (A8)
    ├── cl_marb_ack_checker.py   # ACK checker (A9)
    │
    ├── tests/                    # Test tilfælde (A4)
    │   ├── cl_marb_basic_test.py
    │   ├── cl_marb_static_test.py
    │   └── cl_marb_dynamic_test.py
    │
    ├── vseqs/                    # Virtual sekvenser (A4)
    │   ├── cl_marb_basic_seq.py
    │   ├── cl_marb_static_seq.py
    │   └── cl_marb_dynamic_seq.py
    │
    ├── uvc/                      # Genanvendelig UVC (Universal Verification Components)
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
    ├── ref_model/               # Reference model filer
    ├── sim_build/               # Simuleringsopbygning output
    └── Makefile                 # Opbygnings- og simuleringskommandoer
```

---

## 🏗️ Core Filer Forklaring (A1-A9)

### **A1: Verificeringsplan**
Verificeringsplanen er dokumenteret i rapporten og dækker:
- Design krav (DR01-DR11) kortlægning
- Coverage mål og målinger
- Verificerings strategier (dirigeret + tilfældig)
- Test scenarios matrix

---

### **A2: UVC Integration**

#### **`cl_marb_tb_env.py`** - Miljøbeholder
**Formål**: Centralt knudepunkt som opretter og forbinder alle testbench komponenter

**Nøglekomponenter Oprettet**:
```python
class cl_marb_tb_env(uvm_env):
    def build_phase(self):
        # 1. Hent konfiguration fra ConfigDB
        self.cfg = ConfigDB().get(self, "", "cfg")
        
        # 2. Opret 3 CIF agenter (SDT Producer tilstand)
        self.sdt_cif_agents = []
        for i in range(3):
            agent = cl_sdt_agent(f"cif{i}_agent", self)
            # Konfigurér som producer
        
        # 3. Opret 1 MIF agent (SDT Consumer tilstand)
        self.sdt_mif_agent = cl_sdt_agent("mif_agent", self)
        # Konfigurér som consumer
        
        # 4. Opret kontrolkomponenter
        self.ref_model = cl_marb_ref_model(...)
        self.scoreboard = cl_marb_scoreboard(...)
        self.coverage = cl_marb_coverage_collector(...)
        
    def connect_phase(self):
        # Forbind agenter til verificeringskomponenter
        # Forbind monitorer til analysis ports
        # Opsætning af TLM forbindelser for dataflow
```

**Dataflow i Miljøet**:
```
Sekvenser → Agent Driver → DUT Signaler
                             ↓
Agent Monitor → Kontrolkomponenter
                ├── Reference Model (forudsiger adfærd)
                ├── Scoreboard (sammenligner forudsigelser mod faktisk)
                └── Coverage Indsamler (måler fremskridt)
```

---

#### **`uvc/sdt/src/cl_sdt_agent.py`** - SDT Protocol Agent
**Formål**: Håndterer alle SDT (SyoSil Data Transfer) protokol transaktioner

**Konfiguration Tilstande**:
- **PRODUCER**: Driver rd/wr anmodninger, venter på ack (brugt til CIF0-2)
- **CONSUMER**: Reagerer med ack når anmodninger kommer (brugt til MIF)

**Nøglekomponenter**:
- `cl_sdt_driver.py` - Driver protokol signaler
- `cl_sdt_monitor.py` - Observerer og registrerer transaktioner
- `cl_sdt_sequencer.py` - Koordinerer sekvenser

**SDT Protokol Signaler**:
```
Signaler:     rd, wr, addr, wr_data, rd_data, ack
Retning:      Forskellig for producer vs consumer
Håndshake:    rd/wr + ack = en transaktion
```

---

#### **`uvc/apb/src/cl_apb_agent.py`** - APB Konfigurationsagent
**Formål**: Læser/skriver MARB konfigurationsregistre

**Registre**:
- `0x00 - Control Register`: enable + mode (statisk/dynamisk)
- `0x04 - Priority Register`: CIF0/1/2 prioritetsværdier (8-bit hver)

**Brug i Testbench**:
- Konfigurér arbitrerings mode (statisk eller dynamisk)
- Sæt dynamiske prioriteter
- Vent på prioritetssortering at fuldføres (6 klokke cyklusser)

---

### **A3: Konfigurationsobjekter**

#### **`cl_marb_tb_config.py`** - Top-Level Konfiguration
**Formål**: Beholder for alle UVC konfigurationer

```python
class cl_marb_tb_config(uvm_object):
    def __init__(self):
        self.apb_cfg = cl_apb_config()          # APB konfiguration
        self.sdt_cif_cfgs = []                  # 3x CIF konfigurationer
        for i in range(3):
            self.sdt_cif_cfgs.append(cl_sdt_config())
        self.sdt_mif_cfg = cl_sdt_config()      # MIF konfiguration
```

**Hvad hver konfiguration indeholder**:
- `cl_apb_config`: ADDR_WIDTH, DATA_WIDTH, interface handler
- `cl_sdt_config`: ADDR_WIDTH, DATA_WIDTH, driver type (PRODUCER/CONSUMER), interface handler

**Konfigurationsflow**:
```
Base Test opretter cl_marb_tb_config
         ↓
Sætter APB/SDT parametre
         ↓
Gemmer i ConfigDB
         ↓
Miljø henter fra ConfigDB
         ↓
Agenter henter deres respektive konfigurationer
```

---

### **A4: Test Tilfælde & Sekvenser**

#### **`cl_marb_tb_base_test.py`** - Base Test Klasse
**Formål**: Giver fælles opsætning for alle tests

```python
class cl_marb_tb_base_test(uvm_test):
    def build_phase(self):
        # 1. Opret konfigurationsobjekt
        self.cfg = cl_marb_tb_config()
        
        # 2. Konfigurér APB (32-bit adresse/data)
        self.cfg.apb_cfg.ADDR_WIDTH = 32
        self.cfg.apb_cfg.DATA_WIDTH = 32
        
        # 3. Konfigurér SDT (8-bit adresse/data)
        for cif_cfg in self.cfg.sdt_cif_cfgs:
            cif_cfg.ADDR_WIDTH = 8
            cif_cfg.DATA_WIDTH = 8
            cif_cfg.driver = PRODUCER
        
        self.cfg.sdt_mif_cfg.driver = CONSUMER
        
        # 4. Opret miljø
        self.marb_tb_env = cl_marb_tb_env("marb_tb_env", self)
        
    def connect_phase(self):
        # Forbind alle signaler fra DUT til virtuelle interfaces
        cif0 = self.cfg.sdt_cif_cfgs[0].vif
        cif0.rd = self.dut.c0_rd
        cif0.wr = self.dut.c0_wr
        cif0.addr = self.dut.c0_addr
        # ... osv for alle signaler
        
    async def run_phase(self):
        # 1. Start ACK checker (A9)
        ack_checker = MarbAckChecker(...)
        cocotb.start_soon(ack_checker.start())
        
        # 2. Start SDT protokol checkere (A7)
        for i, vif in enumerate([cif0, cif1, cif2, mif]):
            checker = SDTProtocolChecker(f"CIF{i}", vif)
            cocotb.start_soon(checker.start())
        
        # 3. Start klokke og reset
        await self.start_clock()
        await self.trigger_reset()
        
        # 4. Vent på simulering (test afledt klasse kører sekvenser)
        await Timer(2000, "ns")
```

**Klokke/Reset Generering**:
```python
async def start_clock(self):
    clk_period = randint(2, 5)  # Tilfældiggør mellem 2-5 ns
    cocotb.start_soon(Clock(self.dut.clk, clk_period, "ns").start())

async def trigger_reset(self):
    await ClockCycles(self.dut.clk, randint(1, 3))  # Vent 1-3 cyklusser
    self.dut.rst.value = 1
    await ClockCycles(self.dut.clk, randint(5, 10))  # Hold reset 5-10 cyklusser
    self.dut.rst.value = 0
```

---

#### **Test Tilfælde Eksempler**

**`tests/cl_marb_static_test.py`** - Statisk Prioritet Test
```python
class cl_marb_static_test(cl_marb_tb_base_test):
    """Test med fast prioritet: CIF0 > CIF1 > CIF2"""
    
    async def run_phase(self):
        await super().run_phase()
        
        # 1. Kør konfigurationssekvens (sætter mode=0, enable=1)
        conf_seq = cl_marb_static_apb_cfg_seq()
        cocotb.start_soon(conf_seq.start(self.marb_tb_env.virtual_sequencer))
        
        # 2. Kør virtual sekvens med 3 samtidige klient sekvenser
        vseq = cl_marb_static_vseq()
        await vseq.start(self.marb_tb_env.virtual_sequencer)
        
        # 3. Resultater kontrolleret automatisk af scoreboard
```

**`tests/cl_marb_dynamic_test.py`** - Dynamisk Prioritet Test
```python
class cl_marb_dynamic_test(cl_marb_tb_base_test):
    """Test med programmerede prioriteter via registre"""
    
    async def run_phase(self):
        await super().run_phase()
        
        # Konfigurationssekvens:
        # 1. Deaktivér arbiter (mode=1, enable=0)
        # 2. Skriv prioritetsværdier (0x04: CIF0=0x57, CIF1=0x24, CIF2=0x70)
        # 3. Vent 50ns for sortering at fuldføres (≥6 cyklusser)
        # 4. Aktivér arbiter (mode=1, enable=1)
        
        conf_seq = cl_marb_dynamic_apb_cfg_seq()
        cocotb.start_soon(conf_seq.start(self.marb_tb_env.virtual_sequencer))
        
        vseq = cl_marb_dynamic_vseq()
        await vseq.start(self.marb_tb_env.virtual_sequencer)
```

---

#### **Sekvenser & Virtual Sekvenser**

**`vseqs/cl_marb_basic_seq.py`** - Virtual Sekvens
```python
class cl_marb_basic_seq(uvm_sequence):
    """Koordinerer 3 CIF sekvenser parallelt"""
    
    async def body(self):
        # Få referencer til alle sekvencere
        cif0_seqr = self.sequencer.cif_seqrs[0]
        cif1_seqr = self.sequencer.cif_seqrs[1]
        cif2_seqr = self.sequencer.cif_seqrs[2]
        
        # Start 3 sekvenser parallelt med cocotb
        cocotb.start_soon(self.cif_seq(cif0_seqr, base_addr=0x10))
        cocotb.start_soon(self.cif_seq(cif1_seqr, base_addr=0x20))
        cocotb.start_soon(self.cif_seq(cif2_seqr, base_addr=0x30))
        
        # Vent på at alle fuldføres
        await ClockCycles(self.sequencer.vif.clk, 100)
    
    async def cif_seq(self, seqr, base_addr):
        # Generer tilfældige transaktioner
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

#### **`cl_marb_ref_model.py`** - Gyldent Model
**Formål**: Forudsiger forventet arbitrerings adfærd

```python
class cl_marb_ref_model(uvm_subscriber):
    """Observerer samme stimulus som DUT og forudsiger output"""
    
    def __init__(self):
        self.enable = 0              # Er arbitrering aktiveret?
        self.mode = 0                # 0=statisk, 1=dynamisk
        self.dprio_vals = [0, 0, 0]  # Prioritetsværdier
        self.pending = [deque() for _ in range(3)]  # Anmodnings køer
        
    def write(self, item):
        if isinstance(item, apb_item):
            # Opdatér intern tilstand fra APB skrivninger
            if item.addr == 0x00:
                self.enable = item.data & 0x01
                self.mode = (item.data >> 1) & 0x03
            elif item.addr == 0x04:
                self.dprio_vals[0] = item.data & 0xFF
                self.dprio_vals[1] = (item.data >> 8) & 0xFF
                self.dprio_vals[2] = (item.data >> 16) & 0xFF
        
        elif isinstance(item, sdt_item):
            # Sæt ventende anmodninger i kø
            self.pending[item.client_id].append(item)
            
            # Forudsig hvilken klient der får service
            winning_client = self.arbitrate()
            
            # Send forudsigelse til scoreboard
            self.ref_ap.write(prediction)
    
    def arbitrate(self):
        """Returner klient med højeste prioritet"""
        if self.mode == 0:  # Statisk
            order = [0, 1, 2]  # CIF0 > CIF1 > CIF2
        else:  # Dynamisk
            order = sorted([0,1,2], 
                          key=lambda i: -self.dprio_vals[i])
        
        for cif_id in order:
            if self.pending[cif_id]:
                return cif_id
        return None
```

**Dataflow**:
```
CIF Monitorer → ref_model.write()
                        ↓
APB Monitor → Opdaterer prioritets tilstand
                        ↓
Arbitrerings beslutning
                        ↓
Forudsigelse → scoreboard (via ref_ap)
```

---

### **A6: Scoreboard**

#### **`cl_marb_scoreboard.py`** - Transaktions Sammenligning
**Formål**: Sammenligner reference model forudsigelser vs DUT output

```python
class cl_marb_scoreboard(uvm_component):
    """Sammenligner DUT adfærd mod reference model"""
    
    def __init__(self):
        self.ref_queue = []
        self.dut_queue = []
        self.mismatch_count = 0
    
    def write(self, item, from_ref_model=False):
        if from_ref_model:
            self.ref_queue.append(item)
        else:
            self.dut_queue.append(item)
        
        # Når begge køer har elementer, sammenlign
        if self.ref_queue and self.dut_queue:
            ref_txn = self.ref_queue.pop(0)
            dut_txn = self.dut_queue.pop(0)
            
            if ref_txn.addr != dut_txn.addr:
                self.logger.error(f"Adresse mismatch: "
                    f"ref={ref_txn.addr} vs dut={dut_txn.addr}")
                self.mismatch_count += 1
            
            if ref_txn.data != dut_txn.data and ref_txn.is_write:
                self.logger.error(f"Data mismatch: "
                    f"ref={ref_txn.data} vs dut={dut_txn.data}")
                self.mismatch_count += 1
```

**Rapport Fase**:
```python
def report_phase(self):
    if self.mismatch_count == 0:
        self.logger.info("SCOREBOARD PASS: Alle transaktioner matchede")
    else:
        self.logger.error(f"SCOREBOARD FAIL: {self.mismatch_count} mismatch")
```

---

### **A7: Protokol Checkere**

#### **`uvc/sdt/src/sdt_if_assertions.py`** - SDT Protokol Håndhævelse
**Formål**: Validerer SDT protokol overholdelse under simulering

```python
class SDTProtocolChecker:
    """Overvåger SDT signaler for protokol brud"""
    
    async def start(self):
        while True:
            await RisingEdge(self.vif.clk)
            
            # Check 1: rd og wr er gensidigt udelukkende
            if self.vif.rd.value and self.vif.wr.value:
                raise AssertionError("rd og wr begge høj (ulovligt)")
            
            # Check 2: ack kræver forudgående anmodning
            if self.vif.ack.value:
                if not self.last_had_request:
                    raise AssertionError("ack uden anmodning")
            
            # Check 3: Når rd/wr er aktiv, addr skal være gyldig
            if (self.vif.rd.value or self.vif.wr.value):
                if self.vif.addr.value == 'X':
                    raise AssertionError("X på addr under anmodning")
            
            # Spor til næste cykel
            self.last_had_request = (self.vif.rd.value or 
                                    self.vif.wr.value)
```

**Protokol Invarianter Kontrolleret**:
1. ✓ rd ∧ wr = 0 (gensidigt udelukkelse)
2. ✓ ack → previous(rd ∨ wr) (ack kræver anmodning)
3. ✓ addr ikke X når anmodning aktiv
4. ✓ wr_data ikke X når skrivning aktiv

---

### **A8: Coverage Indsamler**

#### **`cl_marb_coverage.py`** - Funktionel Coverage
**Formål**: Måler verificerings fuldstændighed

```python
class cl_marb_coverage_collector(uvm_subscriber):
    """Indsamler funktionel coverage fra MIF transaktioner"""
    
    def __init__(self):
        # Coverage grupper
        self.write_read_same_addr_bg = CoverageGroup()
        self.burst_detection_cg = CoverageGroup()
        
        # Tilstands sporing
        self.last_addr = None
        self.last_is_write = False
        self.burst_active = False
        self.burst_start_addr = 0
        self.burst_len = 0
    
    def write(self, item):  # Kaldt for hver MIF transaktion
        # Coverage 1: Skrivning efterfulgt af læsning til samme adresse
        if self.last_is_write and item.is_read and \
           self.last_addr == item.addr:
            self.write_read_same_addr_bg.sample(item.addr)
        
        # Coverage 2: Burst detektering
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
        
        # Opdatér tilstand
        self.last_addr = item.addr
        self.last_is_write = item.is_write
    
    def final_phase(self):
        # Eksportér coverage til XML
        self.coverage_model.export_coverage("sim_build/marb_cov.xml")
```

**Coverage Målinger**:
- Skriv-læs samme adresse (back-to-back): 0.78%
- Burst mønstre: 1.57%
- Fuld adresseplads (0-255): Varierer efter test
- APB data stier: 100%

---

### **A9: ACK Checker**

#### **`cl_marb_ack_checker.py`** - Design Krav DR08
**Formål**: Verificerer kun en CIF modtager ACK pr. cykel

```python
class MarbAckChecker:
    """Håndhæver: kun 1 CIF ACK'et pr. cykel (DR08)"""
    
    def __init__(self, name, cif0_vif, cif1_vif, cif2_vif, mif_vif):
        self.vifs = [cif0_vif, cif1_vif, cif2_vif, mif_vif]
    
    async def start(self):
        while True:
            await RisingEdge(self.vifs[0].clk)
            
            # Sum alle ACK signaler
            ack_sum = (self.vifs[0].ack.value + 
                      self.vifs[1].ack.value + 
                      self.vifs[2].ack.value)
            
            # Verificer begrænsning
            if ack_sum > 1:
                raise AssertionError(
                    f"Flere CIF'er ACK'et i samme cykel: {ack_sum}")
```

**Verificering**:
- ✓ ack₀ + ack₁ + ack₂ ≤ 1 ved hver klok kant
- ✓ Ingen overlappende ACK'er detekteret i 115 transaktioner

---

## 📊 Dataflow & Komponent Interaktions Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              TEST FLOW                                           │
│  ┌─ tests/cl_marb_static_test.py [Statisk Prioritet Test]                      │
│  └─ tests/cl_marb_dynamic_test.py [Dynamisk Prioritet Test]                    │
│  └─ tests/cl_marb_basic_test.py [Basis Test Skabelon]                          │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      KONFIGURATION & STIMULUS GENERERING                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  ┌─ cl_marb_tb_config.py [Konfigurationsobjekt]                                │
│  │  ├─ Holder: APB config, SDT CIF/MIF configs                                 │
│  │  └─ Sendt via ConfigDB til miljø                                            │
│  │                                                                               │
│  ┌─ cl_marb_tb_base_test.py [Basis Test]                                       │
│  │  ├─ build_phase: Opretter config, miljø                                     │
│  │  ├─ connect_phase: Kortlægger DUT signaler til virtuelle interfaces         │
│  │  └─ run_phase: Starter klokke, reset, checkere                              │
│  │                                                                               │
│  ┌─ cl_marb_tb_virtual_sequencer.py [Virtual Sekvens]                          │
│  │  ├─ cif0_seqr, cif1_seqr, cif2_seqr [CIF Sekvencere]                       │
│  │  └─ apb_seqr [APB Sekvencer]                                                │
│  │                                                                               │
│  ┌─ vseqs/cl_marb_*_seq.py [Virtual Sekvenser]                                 │
│  │  ├─ cl_marb_static_seq.py [Statisk Prioritet Sekvens]                      │
│  │  ├─ cl_marb_dynamic_seq.py [Dynamisk Prioritet Sekvens]                    │
│  │  ├─ cl_marb_basic_seq.py [Basis Sekvens]                                   │
│  │  └─ cl_reg_simple_seq.py [Registrer Konfig Sekvens]                        │
│  │                                                                               │
│  └─ Koordinerer stimulus over alle agenter                                      │
│                                                                                   │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │ APB Agent    │      │ CIF Agenter  │      │ MIF Agent    │
    │ (PRODUCER)   │      │ (PRODUCER)   │      │ (CONSUMER)   │
    │ uvc/apb/     │      │ uvc/sdt/     │      │ uvc/sdt/     │
    │              │      │              │      │              │
    │ ├─ Driver    │      │ ├─ Drivers   │      │ ├─ Monitor   │
    │ ├─ Monitor   │      │ ├─ Monitorer │      │ └─ Sekvencer │
    │ └─ Sekvencer │      │ ├─ Sekvencere│      │              │
    │              │      │ └─ (3x CIF)  │      │              │
    └──────┬───────┘      └──────┬───────┘      └──────┬───────┘
           │                     │                     │
           │                     │                     │
           └──────────┬──────────┴──────────┬──────────┘
                      │                     │
                      ▼                     ▼
            [DUT RTL: mem_arb.sv]
            ├─ APB Konfig Interface
            ├─ CIF0, CIF1, CIF2 Anmodnings Interfaces
            └─ MIF Output Interface
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    ┌────────────────────────────────────────────────────────────────┐
    │        KONTROLERING & VERIFICERINGS KOMPONENTER (Parallelt)     │
    ├────────────────────────────────────────────────────────────────┤
    │                                                                  │
    │  ┌─ cl_marb_ack_checker.py [ACK Begrænsning Checker - A9]      │
    │  │  Funktion: Sikrer kun 1 CIF ACK pr. cykel (DR08)            │
    │  │  Input: Alle CIF ack signaler                               │
    │  │  Output: Pass/Fail assertion                                │
    │  │                                                              │
    │  ┌─ uvc/sdt/src/sdt_if_assertions.py [Protokol Checker - A7]   │
    │  │  Funktion: Validerer SDT protokol overholdelse              │
    │  │  Input: Alle SDT interface signaler                         │
    │  │  Output: Pass/Fail assertions                               │
    │  │  Kontroller: rd∧wr=0, ack timing, signal validitet          │
    │  │                                                              │
    │  ┌─ cl_marb_ref_model.py [Reference Model - A5]               │
    │  │  Funktion: Forudsiger korrekt arbitrerings resultat         │
    │  │  Input 1: APB Monitor → Konfigurationsændringer             │
    │  │  Input 2: CIF0/1/2 Monitorer → Klient anmodninger          │
    │  │  Algoritme: Statisk prioritet [CIF0>CIF1>CIF2] eller        │
    │  │             Dynamisk prioritet (sorteret efter registerværdi)│
    │  │  Output: Forudsigelser via ref_ap analysis port             │
    │  │                                                              │
    │  ┌─ cl_marb_scoreboard.py [Scoreboard - A6]                   │
    │  │  Funktion: Sammenligner forudsigelser vs faktisk adfærd     │
    │  │  Input 1: ref_model forudsigelser (ref_ap)                  │
    │  │  Input 2: MIF Monitor faktiske txns (dut_ap)                │
    │  │  Sammenligning: Adresse match, access type match, data      │
    │  │  Output: Mismatch count (0 = PASS)                          │
    │  │                                                              │
    │  └─ cl_marb_coverage.py [Coverage Indsamler - A8]             │
    │     Funktion: Måler verificerings fuldstændighed               │
    │     Input: MIF Monitor transaktioner                           │
    │     Målinger: Skriv-læs mønstre, bursts, adresseplads          │
    │     Output: Coverage rapporter (XML)                           │
    │                                                                  │
    └────────────────────────────────────────────────────────────────┘
```

---

## 📈 Dataflow Mellem Komponenter

```
STIMULUS STI:
vseqs/ sekvenser  →  Sekvencere (virtual_sequencer)
                       ├─ cif0_seqr  →  CIF0 Agent driver  →  DUT
                       ├─ cif1_seqr  →  CIF1 Agent driver  →  DUT
                       ├─ cif2_seqr  →  CIF2 Agent driver  →  DUT
                       └─ apb_seqr   →  APB Agent driver   →  DUT APB

OBSERVERINGS STI:
DUT signaler  →  Monitorer (indenfor agenter)
                 ├─ APB Monitor  →  Analysis port  →  ref_model
                 ├─ CIF0 Monitor →  Analysis port  →  ref_model
                 ├─ CIF1 Monitor →  Analysis port  →  ref_model
                 ├─ CIF2 Monitor →  Analysis port  →  ref_model
                 └─ MIF Monitor  →  Analysis port  ─┬─→ scoreboard (dut_subscriber)
                                                      └─→ coverage

FORUDSIGELSENS STI:
ref_model  →  Analysis port (ref_ap)  →  scoreboard (ref_subscriber)
               (giver forudsigelser)

KONTROL STI:
DUT signaler  →  Checkere (kører parallelt med simulering)
                 ├─ Protokol Checkere  (sdt_if_assertions.py)
                 └─ ACK Checker        (cl_marb_ack_checker.py)

RAPPORT STI:
scoreboard  →  Mismatch count  →  report_phase  →  Pass/Fail
coverage    →  Coverage målinger →  final_phase  →  sim_build/marb_cov.xml
```

---

## 🔄 Simulerings Flow

```
1. UDARBEJDELSE
   └─ Indlæs alle Python klasser
   └─ Indlæs RTL modul
   └─ Opret DUT håndtag

2. BUILD FASE
   ├─ cl_marb_tb_base_test.build_phase()
   │  ├─ Opret cl_marb_tb_config
   │  ├─ Opret cl_marb_tb_env
   │  └─ Gem i ConfigDB
   └─ cl_marb_tb_env.build_phase()
      ├─ Opret 3x CIF agenter (Producer)
      ├─ Opret 1x MIF agent (Consumer)
      ├─ Opret APB agent
      ├─ Opret ref_model, scoreboard, coverage

3. FORBIND FASE
   ├─ cl_marb_tb_base_test.connect_phase()
   │  └─ Kortlæg DUT signaler til virtuelle interfaces
   └─ cl_marb_tb_env.connect_phase()
      └─ Forbind monitor analysis ports
      └─ Forbind ref_model til analysis ports
      └─ Forbind scoreboard inputs

4. SIMULERINGS START FASE
   └─ Start protokol checkere (A7)

5. RUN FASE (Faktisk Test)
   ├─ Start ACK checker (A9)
   ├─ Start klokke generering
   ├─ Anvend reset
   ├─ Kør konfigurationssekvens (APB)
   ├─ Kør virtual sekvens (klienter)
   │  └─ Parallelle CIF sekvenser genererer trafik
   │     └─ Transaktioner flyder gennem DUT
   │        └─ MIF transaktioner observeret
   │           └─ Sammenlignet i scoreboard
   │           └─ Samplet af coverage
   └─ Vent på simulering at fuldføres

6. RAPPORT FASE
   └─ Scoreboard rapporterer mismatch
   └─ Coverage rapporterer målinger
   └─ Eksportér UCIS XML til visning
```

---

## 🔑 Nøglebegreber

### **UVM Faser**
- `build_phase`: Opret komponenter
- `connect_phase`: Forbind komponenter
- `run_phase`: Udfør test (async)
- `report_phase`: Udskriv resultater

### **Analysis Ports**
- `request_ap`: CIF monitorer udsender anmodninger øjeblikkeligt (før arbitrering)
- `ap`: MIF monitor udsender serverede transaktioner
- `ref_ap`: Reference model udsender forudsigelser

### **Konfiguration Database (ConfigDB)**
- Hierarkisk parameter overførsel
- Eksempel: `ConfigDB().set(self, "path", "param_name", value)`
- Eksempel: `param = ConfigDB().get(self, "path", "param_name")`

### **Virtuelle Interfaces (VIFs)**
- Python objekter der omvikler DUT signaler
- Oprettet i basis test
- Sendt gennem config til agenter
- Brugt af drivers/monitorer for at få adgang til DUT

---

## 📝 Test Eksempel Gennemgang

### Statisk Prioritet Test Flow:

```python
1. cl_marb_static_test arver fra cl_marb_tb_base_test
   ✓ Arver al build/connect/run opsætning
   ✓ Får fuldt miljø med agenter

2. run_phase() udførelse:
   a) await super().run_phase()
      - Starter ACK checker
      - Starter protokol checkere
      - Starter klokke
      - Anvender reset
      - Venter ~2000 ns
   
   b) conf_seq = cl_marb_static_apb_cfg_seq()
      - Opret sekvens: sæt mode=0 (statisk), enable=1
      - Kør på APB sekvencer
      - Skriver til Control register
   
   c) top_seq = cl_marb_static_vseq()
      - Virtual sekvens koordinering 3 klienter
      - cocotb.start_soon() starter dem parallelt
      - Hver klient genererer 5-15 tilfændige transaktioner
      - Alle bruger samme adresse vindue (0x10-0x2F)
      - Skaber konkurrence for arbiter
   
   d) MIF transaktioner er:
      - Overvåget af MIF monitor
      - Sendt til reference model (kontrollerer forudsigelser)
      - Sendt til scoreboard (sammenligner med ref_model)
      - Samplet af coverage (målinger indsamling)

3. Forventede resultater:
   ✓ Scoreboard: 0 mismatch
   ✓ Coverage: Adresseplads coverage, mønster coverage
   ✓ Protokol checkere: 0 brud
   ✓ ACK checker: Kun 1 CIF ACK'et pr. cykel
   ✓ Bølgeformer: CIF0 serviceret før CIF1, CIF1 før CIF2
```

---

## 🛠️ Hvordan Man Udvider

### Tilføj en ny testcase:
```python
# tests/cl_marb_new_test.py
@pyuvm.test(timeout_time=1000, timeout_unit='ns')
class cl_marb_new_test(cl_marb_tb_base_test):
    async def run_phase(self):
        await super().run_phase()  # Får fuldt miljø
        
        # Din custom test logik her
        conf_seq = ...
        vseq = ...
        await vseq.start(self.marb_tb_env.virtual_sequencer)
```

### Tilføj Coverage:
```python
# I cl_marb_coverage.py, tilføj ny covergroup:
self.new_cg = CoverageGroup("new_coverage")

def write(self, item):
    # ... eksisterende coverage ...
    
    # Nyt coverage punkt
    self.new_cg.sample(item.field_of_interest)
```

### Tilføj Protokol Checker:
```python
# I sdt_if_assertions.py
class SDTProtocolChecker:
    async def start(self):
        while True:
            await RisingEdge(self.vif.clk)
            # Dine nye kontroller her
```

---

## 📚 Fil Afhængigheder Sammenfattelse

```
cl_marb_tb_base_test.py (cl_marb_tb_base_test.py)
├─ imports: cl_marb_tb_config, cl_marb_tb_env
├─ imports: uvc.sdt, uvc.apb
├─ imports: MarbAckChecker, SDTProtocolChecker
└─ brugt af: alle test tilfælde

cl_marb_tb_env.py (cl_marb_tb_env.py)
├─ imports: cl_marb_ref_model, cl_marb_scoreboard, cl_marb_coverage
├─ imports: uvc.sdt.cl_sdt_agent, uvc.apb.cl_apb_agent
└─ brugt af: cl_marb_tb_base_test

tests/cl_marb_*_test.py (tests/*)
├─ imports: cl_marb_tb_base_test
├─ imports: vseqs.cl_reg_simple_seq, vseqs.cl_marb_*_seq
└─ parent: cl_marb_tb_base_test (arver al opsætning)

vseqs/cl_marb_*_seq.py (vseqs/*)
├─ imports: uvc.sdt, uvc.apb
└─ brugt af: test tilfælde til stimulus kørsel
```

---

## ✅ Verificerings Status

Din Gruppe 7 har successfuldt implementeret:
- ✅ A1: Verificeringsplan (dokumenteret i rapport)
- ✅ A2: UVC Integration (SDT & APB agenter forbundet)
- ✅ A3: Konfigurationsobjekter (cl_marb_tb_config)
- ✅ A4: Test tilfælde & Sekvenser (statiske & dynamiske tests)
- ✅ A5: Reference Model (forudsiger arbitrering)
- ✅ A6: Scoreboard (0 mismatch i testning)
- ✅ A7: SDT Protokol Checkere (ingen brud)
- ✅ A8: Coverage Indsamler (0.78-100% afhængig af metrik)
- ✅ A9: ACK Checker (DR08 verificeret)

**Alle tests bestået!** 🎉
