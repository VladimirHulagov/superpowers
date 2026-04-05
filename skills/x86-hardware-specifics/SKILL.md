---
name: x86-hardware-specifics
description: Use when developing low-level x86 firmware, kernel drivers, or platform code that interacts directly with CPU registers, chipset interfaces, PCI/PCIe configuration, APIC, SMM, VT-x/VT-d, power management, or platform tables (ACPI, SMBIOS, MPTable)
---

# x86 Hardware Specifics

## Overview

Quick-reference tables for x86 platform programming: CPU registers, paging, interrupts, PCI/PCIe, virtualization, SMM, power management, and firmware tables. Values are Intel/AMD common unless noted.

## When to Use

- Writing UEFI DXE/SMM drivers or OS kernel code that touches MSRs, CR registers, or CPUID
- Configuring APIC, IOAPIC, MSI/MSI-X interrupt delivery
- Setting up paging structures, EPT, or VT-d remapping
- Programming PCI/PCIe config space, BARs, or capabilities
- Implementing SMI handlers, SMRAM layout, or measured boot
- Parsing or constructing ACPI, SMBIOS, or MPTable entries
- Power state transitions (C-states, P-states, MWAIT)
- Working with Intel GPIO pad registers or SPI flash descriptor

## When NOT to Use

- Application-level or userspace development
- High-level framework code (ACPI AML interpretation, UEFI protocol consumers)
- ARM or RISC-V platform work
- Documentation or comment writing without register-level detail

## 1. CPU Register Reference

### Key MSRs

| MSR Index | Name | Purpose |
|-----------|------|---------|
| 0x0000001B | IA32_APIC_BASE | Local APIC base address, BSP flag, global enable |
| 0x0000002A | MSR_EBL_CR_POWERON | Bus ratio, clock control |
| 0x0000003A | IA32_FEATURE_CONTROL | VMX/SMX lock, VMX outside SMX enable |
| 0x00000079 | IA32_BIOS_UPDT_TRIG | Microcode update trigger (write-only) |
| 0x0000008B | IA32_BIOS_SIGN_ID | Microcode revision (read after write to 0x79) |
| 0x000000C1 | IA32_PMC0 | Performance counter 0 |
| 0x000000CD | MSR_FSB_FREQ | Front-side bus frequency encoding |
| 0x000000E2 | MSR_PKG_CST_CONFIG_CONTROL | Package C-state limit, C-state demotion |
| 0x000000E7 | IA32_MPERF | Maximum performance frequency clock count |
| 0x000000E8 | IA32_APERF | Actual performance frequency clock count |
| 0x000000FE | MSR_MTRR_CAPABILITIES | MTRR count, fixed-range support, WC support |
| 0x0000011E | MSR_MISC_ENABLE | Fast strings, TM1/TM2, speedstep |
| 0x00000174 | IA32_SYSENTER_CS | SYSENTER CS segment selector |
| 0x00000175 | IA32_SYSENTER_ESP | SYSENTER stack pointer |
| 0x00000176 | IA32_SYSENTER_EIP | SYSENTER entry point |
| 0x00000199 | IA32_PERF_CTL | P-state control (target ratio, IDA enable) |
| 0x0000019A | IA32_CLOCK_MODULATION | Duty cycle, TM1 enable |
| 0x0000019B | IA32_THERM_INTERRUPT | Thermal interrupt thresholds, policy |
| 0x0000019C | IA32_THERM_STATUS | Thermal status flags, PROCHOT |
| 0x0000019D | IA32_THERM2_CTL | TM2 target ratio |
| 0x000001A0 | IA32_MISC_ENABLE | Enhanced SpeedStep, MONITOR/MWAIT, limit CPUID max |
| 0x000001A2 | MSR_TEMPERATURE_TARGET | Tjunction max, target offset |
| 0x00000200 | MSR_MTRR_PHYS_BASE0 | MTRR variable range 0 base |
| 0x000002FF | MSR_MTRR_DEF_TYPE | Default memory type, MTRR enable |
| 0x00000300 | MSR_BBL_CR_CTL3 | L2 cache control |
| 0x00000345 | IA32_PERF_CAPABILITIES | PEBS, LBR format, SMM freeze |
| 0x000003F1 | IA32_PEBS_ENABLE | PEBS counter enable |
| 0x000003F8 | MSR_PKG_POWER_SKU_UNIT | Power/energy/time units (multiplier) |
| 0x0000060A | MSR_IA32_CORE_CAPS | Core capability reporting |
| 0x00000610 | IA32_BNDCFGS | MPX bound configuration |
| 0x00000611 | IA32_XSS | Extended supervisor states |
| 0x0000064D | MSR_PP0_POWER_LIMIT | Power limit for package domain 0 |
| 0x0000066C | MSR_LASTBRANCH_FROM_IP | LBR stack from IP |
| 0x0000066D | MSR_LASTBRANCH_TO_IP | LBR stack to IP |
| 0x0000066E | MSR_LASTBRANCH_FROM_IP | LBR from (alternate bank) |
| 0x000006A0 | MSR_IA32_RING_PERF_LIMIT_REASONS | Ring performance limit reasons |
| 0x00000C80 | IA32_LBR_CTL | LBR enable, filtering, call stack mode |
| 0x00000C81 | IA32_LBR_DEPTH | LBR stack depth (configurable on some models) |
| 0x00000DA0 | MSR_IA32_CORE_THREAD_COUNT | Core and thread count |
| 0x00000FE0 | IA32_MC0_CTL | Machine check control for bank 0 |
| 0x00001000 | IA32_PAT | Page attribute table (8 entries × 3 bits) |
| 0x00002000 | IA32_MTRR_FIX64K_00000 | Fixed MTRR: 0x00000–0x7FFFF (8 × 64KB) |
| 0x0000208 | IA32_MTRR_FIX16K_80000 | Fixed MTRR: 0x80000–0x9FFFF |
| 0x0000209 | IA32_MTRR_FIX16K_A0000 | Fixed MTRR: 0xA0000–0xBFFFF |
| 0x0000400 | IA32_DS_AREA | Debug store area pointer |
| 0x0000480 | IA32_VMX_BASIC | VMCS revision ID, VMCS size, VMX controls |
| 0x0000481 | IA32_VMX_PINBASED_CTLS | Pin-based VM-execution controls |
| 0x0000482 | IA32_VMX_PROCBASED_CTLS | Primary processor-based VM-execution controls |
| 0x000048E | IA32_VMX_PROCBASED_CTLS2 | Secondary processor-based VM-execution controls |
| 0x0000600 | IA32_VMX_TRUE_PINBASED_CTLS | True pin-based controls (no default1 fixup) |
| 0x0000680 | IA32_DS_AREA | Debug store (DS) save area |
| 0x0000770 | MSR_IA32_CORE_CAPS | Core capabilities (MDS, TAA no) |
| 0x0000C80 | IA32_LBR_CTL | LBR enable and filter controls |
| 0xC0000080 | MSR_EFER | Extended feature enable (SCE, LME, LMA, NXE, SCE) |
| 0xC0000081 | MSR_STAR | SYSCALL target CS/SS selectors |
| 0xC0000082 | MSR_LSTAR | SYSCALL 64-bit entry point |
| 0xC0000083 | MSR_CSTAR | SYSCALL compat entry point |
| 0xC0000084 | MSR_SFMASK | SYSCALL flags mask |
| 0xC0000100 | MSR_FS_BASE | FS segment base |
| 0xC0000101 | MSR_GS_BASE | GS segment base |
| 0xC0000102 | MSR_KERNEL_GS_BASE | Shadow GS base (swapgs target) |
| 0xC0000103 | MSR_TSC_AUX | RDTSCP auxiliary value (written to ECX) |

### CR0 Bit-Field Map

| Bits | Field | Description |
|------|-------|-------------|
| 0 | PE | Protection enable (real vs protected mode) |
| 1 | MP | Monitor coprocessor (WAIT/FWAIT interaction with TS) |
| 2 | EM | Emulate coprocessor (x87 FPU not present) |
| 3 | TS | Task switched (lazy FPU save) |
| 4 | ET | Extension type (387DX present; hardcoded 1 on P5+) |
| 5 | NE | Numeric error (native x87 error reporting via IRQ13) |
| 16 | WP | Write protect (supervisor writes to read-only pages fault) |
| 18 | AM | Alignment mask (AC flag in EFLAGS is checked in CPL3) |
| 29 | NW | Not write-through (cache write-through policy) |
| 30 | CD | Cache disable (set to disable caching) |
| 31 | PG | Paging enable (requires PE=1) |

### CR2

Read-only after page fault; holds the faulting linear address.

### CR3 Bit-Field Map

| Bits | Field | Description |
|------|-------|-------------|
| 0 | PCID (if CR4.PCIDE=1) | Process-context identifier (bits 0–11) |
| 2:1 | Reserved | Must be 0 |
| 3 | PWT | Page-level write-through (for PML4 table) |
| 4 | PCD | Page-level cache disable (for PML4 table) |
| 11:5 | Reserved (if no PCID) / PCID high bits | Must be 0 if CR4.PCIDE=0 |
| 51:12 | Physical address | PML4 table base (4-KB aligned) |

### CR4 Bit-Field Map

| Bits | Field | Description |
|------|-------|-------------|
| 0 | VME | Virtual-8086 mode extensions |
| 1 | PVI | Protected-mode virtual interrupts |
| 2 | TSD | Time stamp disable (RDTSC CPL0 only) |
| 3 | DE | Debugging extensions (I/O breakpoints in DR registers) |
| 4 | PSE | Page size extension (4 MB pages in 32-bit paging) |
| 5 | PAE | Physical address extension (36-bit address) |
| 6 | MCE | Machine-check enable |
| 7 | PGE | Page global enable (global pages in TLB) |
| 8 | PCE | Performance-monitoring counter enable (CPL3 RDPMC) |
| 9 | OSFXSR | OS support for FXSAVE/FXRSTOR |
| 10 | OSXMMEXCPT | OS support for unmasked SIMD FP exceptions |
| 11 | UMIP | User-mode instruction prevention (SGDT/SIDT/SLDT/STR) |
| 13 | VMXE | VMX enable |
| 14 | SMXE | SMX enable |
| 16 | FSGSBASE | Enable RDFSBASE/WRFSBASE/RDGSBASE/WRGSBASE |
| 17 | PCIDE | PCID enable |
| 18 | OSXSAVE | OS enable for XGETBV/XSETBV and XSAVE |
| 20 | SMEP | Supervisor mode execution prevention |
| 21 | SMAP | Supervisor mode access prevention |
| 22 | PKE | Protection keys enable (user pages) |
| 23 | CET | Control-flow enforcement enable |
| 24 | PKS | Protection keys for supervisor pages |
| 25 | KL | Key locker enable |

### CR8 (Task Priority Register, 64-bit only)

| Bits | Field | Description |
|------|-------|-------------|
| 3:0 | TPR | Task priority (0–15), blocks interrupts with priority ≤ TPR |

### CPUID Leaves

| EAX Input | Output | Meaning |
|-----------|--------|---------|
| 01H | EAX=version, EBX=brand index/clflush/cores, EDX/ECX=feature flags | Processor info and feature bits |
| 02H | EAX/EBX/ECX/EDX=cache/tlb descriptors | Cache and TLB information (legacy) |
| 04H | EAX=cache type/level, EBB=ways/parts, ECX=threads, EDX=line size | Deterministic cache parameters (subleaf for each level) |
| 07H | EAX=max subleaf, EBX/ECX/EDX=extended features | Extended features (FSGSBASE, SGX, BMI1/2, AVX2, AVX512, etc.) |
| 07H subleaf 1 | EAX=features, EDX=features | AMX, AVX-VNNI, CMPCCXADD |
| 0BH | EAX=x2APIC shift, EBB=logicals at level, ECX=level type, EDX=x2APIC ID | Extended topology enumeration |
| 0DH | EAX=supported states, EBX=xsave size, ECX=extended size, EDX=supervisor states | XSAVE/XRSTOR features (subleaf per state component) |
| 0FH | RDT monitoring (subleaf 0: ebx=num RMID; subleaf 1: edx=l3 cos) | QoS monitoring |
| 10H | RDT allocation (subleaf 0: ebx=num closid; subleaf 1: L3 mask) | QoS enforcement |
| 12H | SGX capability enumeration | SGX leaf info and enclave characteristics |
| 14H | Processor trace | Intel PT capabilities |
| 15H | EAX=denominator, EBX=numerator, ECX=nominal TSC freq | TSC / nominal core crystal ratio |
| 16H | EAX=base freq MHz, EBX=max freq MHz, ECX=bus ref freq MHz | Processor frequency info |
| 1BH | PCONFIG information | Platform configuration (key locker, TDX) |
| 80000001H | EDX/ECX=extended features | SYSCALL/NX/LM/LAHF/CMOV, etc. |
| 80000002H–80000004H | EAX/EBX/ECX/EDX=16 bytes each | Processor brand string (48 bytes total) |
| 80000005H | L1 cache and TLB info (AMD) | L1 cache/TLB |
| 80000006H | L2/L3 cache info (AMD), L2 on Intel | L2 cache line/size/assoc |
| 80000007H | EDX=advanced power management | Invariant TSC (bit 8), C-states |
| 80000008H | EAX=phys/virt address bits, ECX=features | Max physical/virtual address sizes, CLZERO, RDCL_NO |

### EFLAGS Meaningful Bits

| Bit | Name | Description |
|-----|------|-------------|
| 0 | CF | Carry flag |
| 2 | PF | Parity flag |
| 4 | AF | Auxiliary carry flag |
| 6 | ZF | Zero flag |
| 7 | SF | Sign flag |
| 8 | TF | Trap flag (single-step) |
| 9 | IF | Interrupt enable flag |
| 10 | DF | Direction flag |
| 11 | OF | Overflow flag |
| 12–13 | IOPL | I/O privilege level (CPL required for IN/OUT) |
| 14 | NT | Nested task |
| 16 | RF | Resume flag (suppress debug exception) |
| 17 | VM | Virtual-8086 mode |
| 18 | AC | Alignment check (requires CR0.AM) |
| 19 | VIF | Virtual interrupt flag |
| 20 | VIP | Virtual interrupt pending |
| 21 | ID | CPUID instruction available (read-only test) |

### Debug Registers

| Register | Bits | Description |
|----------|------|-------------|
| DR0 | 63:0 | Breakpoint linear address 0 |
| DR1 | 63:0 | Breakpoint linear address 1 |
| DR2 | 63:0 | Breakpoint linear address 2 |
| DR3 | 63:0 | Breakpoint linear address 3 |
| DR6 | 0 | B0: breakpoint 0 detected |
| DR6 | 1 | B1: breakpoint 1 detected |
| DR6 | 2 | B2: breakpoint 2 detected |
| DR6 | 3 | B3: breakpoint 3 detected |
| DR6 | 13 | BD: debug register access detected |
| DR6 | 14 | BS: single step |
| DR6 | 15 | BT: task switch |
| DR7 | 0 | L0: local breakpoint 0 enable |
| DR7 | 1 | G0: global breakpoint 0 enable |
| DR7 | 2 | L1: local breakpoint 1 enable |
| DR7 | 3 | G1: global breakpoint 1 enable |
| DR7 | 4 | L2: local breakpoint 2 enable |
| DR7 | 5 | G2: global breakpoint 2 enable |
| DR7 | 6 | L3: local breakpoint 3 enable |
| DR7 | 7 | G3: global breakpoint 3 enable |
| DR7 | 8–9 | R/W0: 00=exec, 01=write, 10=I/O (if CR4.DE), 11=R/W |
| DR7 | 10–11 | LEN0: 00=1B, 01=2B, 11=4B, 10=8B |
| DR7 | 16–17, 24–25 | R/W1, R/W2 |
| DR7 | 18–19, 26–27 | LEN1, LEN2 |
| DR7 | 20–21 | R/W3 |
| DR7 | 22–23 | LEN3 |

### XCR0 (Extended Control Register 0)

| Bit | Feature | State Component |
|-----|---------|-----------------|
| 0 | x87 FPU | x87 FPU state |
| 1 | SSE | SSE state (XMM0–XMM15) |
| 2 | AVX | YMM upper 128 bits (Hi16_ZMM if AVX-512) |
| 3 | BNDREGS | MPX bound registers |
| 4 | BNDCSR | MPX bound configuration |
| 5 | OPMASK | AVX-512 opmask registers (k0–k7) |
| 6 | ZMM_Hi256 | AVX-512 upper 256 bits of ZMM0–ZMM15 |
| 7 | Hi16_ZMM | AVX-512 ZMM16–ZMM31 |
| 8 | PT | Processor trace (Intel PT) |
| 9 | PKRU | Protection keys register |
| 10 | PASID | Process address space ID |
| 11 | CET_U | CET user state |
| 12 | CET_S | CET supervisor state |
| 13 | HDC | Hardware duty cycling |
| 15 | LBR | Last branch records |

## 2. Memory & Paging

### 4-Level Paging Entry Format (PML4/PDP/PD)

| Bits | Field | Description |
|------|-------|-------------|
| 0 | P | Present |
| 1 | R/W | Read/Write |
| 2 | U/S | User/Supervisor |
| 3 | PWT | Page-level write-through |
| 4 | PCD | Page-level cache disable |
| 5 | A | Accessed |
| 6 | D (PD→PT only) | Dirty |
| 7 | PS | Page size (1=maps large page) |
| 8 | G | Global (if CR4.PGE) |
| 9–11 | Ignored/Available | OS usable |
| 12–M-1 | Physical address | Page frame address (M=MAXPHYADDR, typically 46 or 51) |
| M–51 | Reserved | Must be 0 |
| 52–58 | Available | OS usable (if not reserved by M) |
| 59–62 | Available | OS usable |
| 63 | XD/NX | Execute disable (if IA32_EFER.NXE=1) |

### 5-Level Paging

Adds PML5 table level; CR3 → PML5E → PML4E → PDPTE → PDE → PTE. Requires CR4.LA57=1. Physical address extends up to bit 51 (57-bit linear address).

### EPT Entry Format

| Bits | Field | Description |
|------|-------|-------------|
| 0 | Read | EPT read access |
| 1 | Write | EPT write access |
| 2 | Execute | EPT execute access |
| 3 | Reserved (if not PML4) / Leaf flags | Depends on EPT paging-structure level |
| 4 | Reserved | Must be 0 |
| 5 | A | EPT accessed |
| 6 | D | EPT dirty |
| 7 | User-executable | Execute-only for supervisor if mode-based |
| 8 | N | Ignore PAT (use EPT MT) |
| 9 | PAT | Page attribute table select |
| 10 | Suppressor | Suppress \#VE (EPT violation → \#VE) |
| 12–51 | Physical address | Host physical page frame |
| 52–58 | Available for VMM | Custom storage |
| 59–62 | Available for VMM | Custom storage |
| 63 | VMX control | Sub-page write permission / ve-info |

### PAT Memory Types

| PAT Index | Type | Typical Use |
|-----------|------|-------------|
| 0 | UC (Uncacheable) | Device MMIO |
| 1 | WC (Write Combining) | Frame buffer, PCIe |
| 2 | WT (Write Through) | Rarely used |
| 3 | WP (Write Protected) | SMM shadow memory |
| 4 | WB (Write Back) | Normal RAM |
| 5 | UC- (Uncacheable, weak) | May be overridden by MTRR |
| 6 | UC | Same as 0 |
| 7 | UC | Same as 0 |

Default PAT: 0x0007040600070406 (PA6=UC, PA5=WP, PA4=WT, PA3=UC-, PA2=WB, PA1=WC, PA0=UC).

### SMRAM Regions

| Region | Address Range | Size | Description |
|--------|---------------|------|-------------|
| ASEG | 0xA0000–0xBFFFF | 128 KB | Legacy SMM memory (SMRAM base = A-seg) |
| HSEG | 0xFEDA0000–0xFEDBFFFF | 128 KB | High SMRAM (T-seg compatible) |
| TSEG | Top of DRAM – N MB | Variable (BIOS config) | Extended SMRAM, size set by MRC/PCH |

SMRAMC register (PCH) bits: T_EN (bit 0), TSEG_SZ (bits 1:2: 0=1MB, 1=2MB, 2=8MB, 3=reserved), H_EN (bit 4), A_EN (bit 5), L_EN (bit 6 = lock).

### MTRR

**Fixed-range MTRRs:**

| Register | Address Range | Granularity |
|----------|---------------|-------------|
| IA32_MTRR_FIX64K_00000 | 0x00000–0x7FFFF | 8 × 64KB (8 types) |
| IA32_MTRR_FIX16K_80000 | 0x80000–0x9FFFF | 8 × 16KB (8 types) |
| IA32_MTRR_FIX16K_A0000 | 0xA0000–0xBFFFF | 8 × 16KB (8 types) |
| IA32_MTRR_FIX4K_C0000–C8000 | 0xC0000–0xCFFFF | 8 regs × 8 × 4KB |
| IA32_MTRR_FIX4K_D0000–D8000 | 0xD0000–0xDFFFF | 8 regs × 8 × 4KB |
| IA32_MTRR_FIX4K_E0000–E8000 | 0xE0000–0xEFFFF | 8 regs × 8 × 4KB |
| IA32_MTRR_FIX4K_F0000–F8000 | 0xF0000–0xFFFFF | 8 regs × 8 × 4KB |

**Variable MTRR pairs:** IA32_MTRR_PHYS_BASEn (base + type) / IA32_MTRR_PHYS_MASKn (mask + valid). Count from IA32_MTRR_CAPABILITIES[7:0] (typically 8–10). Mask uses powers of 2 alignment.

### VT-d Remapping Structures

**Root Entry (4KB table, 256 entries):**

| Bits | Field | Description |
|------|-------|-------------|
| 0 | P | Present |
| 1–11 | Reserved | |
| 12–63 | CTDP | Context Table pointer (4KB aligned) |

**Context Entry (4KB table, 256 entries per root entry):**

| Bits | Field | Description |
|------|-------|-------------|
| 0 | P | Present |
| 1 | FPD | Fault Processing Disable |
| 2–7 | Reserved | |
| 8 | TT (bits 8–9) | Translation Type: 00=legacy, 01=pass-through, 10,11=reserved |
| 12–63 | SLPTPTR | Second-Level Page Translation pointer (ASR) |
| 64–127 | Reserved / Extended | Domain identifier in bits 72:81 (domain ID 16 bits) |

## 3. Interrupt & APIC

### Local APIC MMIO Register Map

| Offset | Register | Description |
|--------|----------|-------------|
| 0x020 | LAPIC_ID | Local APIC ID (bits 24–31 on P6, all bits on x2APIC) |
| 0x030 | LAPIC_VERSION | Version (max LVT entry, EOI suppression support) |
| 0x080 | TPR | Task priority register (bits 7:0, priority class+sub) |
| 0x0B0 | EOI | End of interrupt (write any value) |
| 0x0D0 | LDR | Logical destination register |
| 0x0F0 | SPIV | Spurious interrupt vector (vector, APIC enable bit 8, focus bit 9) |
| 0x100–0x170 | ISR | In-service register (256 bits, 0x100–0x170) |
| 0x180–0x190 | TMR | Trigger mode register |
| 0x200–0x270 | IRR | Interrupt request register |
| 0x280 | ESR | Error status register (write before read to clear) |
| 0x290 | LVT_CMCI | LVT entry for corrected machine-check interrupt |
| 0x300 | ICR_LOW | Interrupt command register (low 32 bits) |
| 0x310 | ICR_HIGH | Interrupt command register (high 32 bits: dest field) |
| 0x320 | LVT_TIMER | LVT timer entry |
| 0x330 | LVT_THERMAL | LVT thermal sensor entry |
| 0x340 | LVT_PMC | LVT performance counter entry |
| 0x350 | LVT_LINT0 | LVT local interrupt 0 entry |
| 0x360 | LVT_LINT1 | LVT local interrupt 1 entry |
| 0x370 | LVT_ERROR | LVT error entry |
| 0x380 | TIMER_ICR | Timer initial count register |
| 0x390 | TIMER_CCR | Timer current count register |
| 0x3E0 | TIMER_DCR | Timer divide configuration |

### IPI ICR Layout (0x300)

| Bits | Field | Description |
|------|-------|-------------|
| 7:0 | Vector | Interrupt vector number |
| 10:8 | DM | Delivery mode: 000=fixed, 010=SMI, 100=NMI, 101=INIT, 110=SIPI |
| 11 | Reserved | Must be 0 |
| 12 | DS | Delivery status: 0=idle, 1=send pending |
| 13 | Reserved | |
| 14 | Level | 0=deassert, 1=assert (for INIT deassert) |
| 15 | TM | Trigger mode: 0=edge, 1=level |
| 17:16 | RSVD | Must be 0 |
| 19:18 | SD | Shorthand: 00=none (use dest), 01=self, 10=all incl self, 11=all excl self |
| 63:56 | Dest | Destination APIC ID (xAPIC: bits 63:56; x2APIC: entire 32-bit dest in ICR_HIGH+low) |

### LVT Entry Format

| Bits | Field | Description |
|------|-------|-------------|
| 7:0 | Vector | Interrupt vector |
| 9:8 | DM | Delivery mode: 000=fixed, 010=SMI, 100=NMI, 101=INIT, 111=ExtINT |
| 10 | DS | Delivery status (read-only): 0=idle, 1=send pending |
| 11 | Reserved | |
| 12 | Polar | Input pin polarity: 0=active high, 1=active low |
| 13 | TM | Trigger mode: 0=edge, 1=level |
| 14 | Mask | Interrupt mask: 0=enabled, 1=masked |
| 17:16 | Timer mode | 00=one-shot, 01=periodic, 10=TSC-deadline (timer only) |

### IOAPIC Registers

| Register | Offset/Value | Description |
|----------|-------------|-------------|
| IOREGSEL | 0x00 (index register) | Select IOAPIC register for access |
| IOWIN | 0x10 (data register) | Read/write selected register |
| IOAPICID | Index 0x00 | IOAPIC ID (bits 24–27) |
| IOAPICVER | Index 0x01 | Version (bits 23:16 = max redirection entry) |
| IOAPICARB | Index 0x02 | Arbitration ID |
| IOREDTBL0–23 | Index 0x10–0x3F | Redirection entries (2 × 32-bit per RTE) |

**RTE Format (64 bits):**

| Bits | Field | Description |
|------|-------|-------------|
| 7:0 | Vector | Interrupt vector |
| 10:8 | DM | Delivery mode (same as LVT) |
| 11 | DM | Destination mode: 0=physical, 1=logical |
| 12 | DS | Delivery status |
| 13 | Polar | Pin polarity |
| 14 | TM | Trigger mode: 0=edge, 1=level |
| 15 | Mask | Interrupt mask |
| 16 | Destination (bits 56–63) | Destination APIC ID |

### MSI Capability Structure

| Offset (from cap) | Register | Description |
|-------------------|----------|-------------|
| 0x00 | Cap ID + Next | Capability ID 0x05, next pointer |
| 0x02 | Message Control | MSI enable (bit 0), per-vector masking (bit 8), multi-message enable/count (bits 4:6, 1:3) |
| 0x04 | Message Address (low 32) | Bits 31:2 of address, bit 0=always 0 |
| 0x08 | Message Address (upper 32) | Upper 32 bits if 64-bit capable (bit 23 of msg ctrl) |
| 0x0C | Message Data | Vector (bits 7:0), delivery mode (bits 10:8), level (bit 14), trigger (bit 15) |
| 0x10 | Mask Bits | Per-vector mask (if per-vector masking capable) |
| 0x14 | Pending Bits | Per-vector pending |

### MSI-X Capability Structure

| Offset (from cap) | Register | Description |
|-------------------|----------|-------------|
| 0x00 | Cap ID + Next | Capability ID 0x11, next pointer |
| 0x02 | Message Control | MSI-X enable (bit 15), function mask (bit 14), table size (bits 10:0 = N-1) |
| 0x04 | Table Offset/BIR | Table offset (bits 31:3), BAR index (bits 2:0) |
| 0x08 | PBA Offset/BIR | PBA offset (bits 31:3), BAR index (bits 2:0) |

Each MSI-X table entry: 16 bytes — MsgAddr(32), MsgUpperAddr(32), MsgData(32), VectorCtrl(32, bit 0 = mask).

### IRTE (Interrupt Remapping Table Entry, VT-d)

| Bits | Field | Description |
|------|-------|-------------|
| 0 | P | Present |
| 1 | FPD | Fault processing disable |
| 2–4 | DM | Delivery mode |
| 5 | Pol | Coherent (side effect of upstream) / RH |
| 6 | TM | Trigger mode |
| 7 | RHD | Remap enable: 0=remapped, 1=posted |
| 8–15 | SVT/SID | Source ID qualification / source ID |
| 16–31 | Source ID | Bus/Dev/Func of source |
| 32–63 | Dest ID | Destination APIC ID (x2APIC compatible) |
| 64–71 | Vector | Interrupt vector |
| 79 | IM | Interrupt mode for posted |
| 87:80 | DLM | Delivery mode |

## 4. PCI/PCIe

### CF8/CFC Legacy Access Mechanism

CF8 (Configuration Address, 32-bit write):

| Bits | Field | Description |
|------|-------|-------------|
| 0 | Enable | Must be 1 to generate config cycle |
| 1–7 | Reserved | |
| 8–15 | Bus | Bus number |
| 16–20 | Device | Device number |
| 21–23 | Function | Function number |
| 24–31 | Register | DWORD-aligned register offset |

CFC: Read/write the selected DWORD. Byte/word accesses use CF8 + 0–3 offset mapping.

### ECAM (Enhanced Configuration Access Mechanism)

Address = Base + (Bus × 2²⁰) + (Device × 2¹⁵) + (Function × 2¹²) + (Register × 2⁰)

Base from MCFG ACPI table. Supports full 4KB config space per function.

### PCIe Configuration Space Layout

**Type 0 Header (Endpoint):**

| Offset | Register |
|--------|----------|
| 0x00–0x01 | Vendor ID |
| 0x02–0x03 | Device ID |
| 0x04–0x05 | Command |
| 0x06–0x07 | Status |
| 0x08 | Revision ID |
| 0x09–0x0B | Class Code (prog IF, subclass, class) |
| 0x0C | Cache Line Size |
| 0x0D | Latency Timer |
| 0x0E | Header Type (bit 7 = multi-function, bits 6:0 = 0) |
| 0x0F | BIST |
| 0x10–0x27 | BAR0–BAR5 (6 × 32-bit) |
| 0x28–0x2B | Cardbus CIS Pointer |
| 0x2C–0x2D | Subsystem Vendor ID |
| 0x2E–0x2F | Subsystem Device ID |
| 0x30–0x33 | Expansion ROM Base |
| 0x34 | Capabilities Pointer |
| 0x3C | Interrupt Line |
| 0x3D | Interrupt Pin |
| 0x3E–0x3F | Min Gnt / Max Lat |

**Type 1 Header (Bridge):**

| Offset | Register |
|--------|----------|
| 0x00–0x0F | Same as Type 0 |
| 0x10–0x13 | BAR0 |
| 0x14–0x17 | BAR1 |
| 0x18–0x19 | Primary Bus Number |
| 0x1A–0x1B | Secondary Bus Number |
| 0x1C–0x1D | Subordinate Bus Number |
| 0x1E | Secondary Latency Timer |
| 0x1F | I/O Base |
| 0x20 | I/O Limit |
| 0x21–0x22 | Secondary Status |
| 0x23 | Memory Base |
| 0x24 | Memory Limit |
| 0x25–0x28 | Prefetchable Memory Base |
| 0x29–0x2C | Prefetchable Memory Limit |
| 0x2D–0x30 | Prefetchable Base Upper 32 |
| 0x31–0x34 | Prefetchable Limit Upper 32 |
| 0x35 | I/O Base Upper 16 |
| 0x36 | I/O Limit Upper 16 |
| 0x37 | Capabilities Pointer |
| 0x38 | Expansion ROM Base |
| 0x3C | Interrupt Line |
| 0x3D | Interrupt Pin |
| 0x3E | Bridge Control |

### TLP Header Format

**3DW TLP (Memory Read/Write, 32-bit address):**

| Byte | Bits | Description |
|------|------|-------------|
| 0 | 23:22 | Fmt: 000=3DW no data, 010=3DW with data |
| 0 | 21:16 | Type: 000000=Mrd, 100000=Mwr |
| 0 | 15 | TC[2] |
| 0 | 14:13 | Attr[2:1] (relaxed ordering, no snoop) |
| 0 | 11:10 | TD, EP |
| 0 | 9:0 | Length[9:0] (DWORDs) |
| 4 | 31:12 | Requester ID (bits 31:16), Tag (bits 15:8), Last DW BE (bits 7:4), 1st DW BE (bits 3:0) |
| 8 | 31:2 | Address[31:2] |
| 12 | — | (data or next TLP) |

**4DW TLP (64-bit address):** Same as above but address is bytes 8–15 (64-bit).

### BAR Sizing Protocol

1. Save original BAR value
2. Write 0xFFFFFFFF to BAR
3. Read back: mask the lower bits per type (memory: bits 3:0, IO: bits 1:0)
4. Invert all bits, add 1 → size
5. Restore original BAR value

### Capability IDs

| ID | Name | Key Registers |
|----|------|---------------|
| 0x01 | Power Management (PM) | PM CSR (D0–D3 power state bits 0:1), PME enable/status |
| 0x05 | MSI | Message control, address, data |
| 0x07 | PCI-X | Command, status, bridge |
| 0x08 | HyperTransport | |
| 0x10 | PCIe Cap | Link status/control, device status, slot |
| 0x0D | SSID/SSVID | Subsystem vendor/device ID |
| 0x11 | MSI-X | Table/PBA offset + BIR |
| 0x0E | SATA | |
| 0x12 | Advanced Error Reporting (AER) | UE/CE status, TLP prefix log |
| 0x13 | Enhanced Allocation (EA) | |
| 0x14 | FLit | |
| 0x15 | FRS | |
| 0x16 | PCIe Cap (modified) | |
| 0x17 | TPH | |
| 0x19 | secondary PCI Express | |
| 0x1B | Data Link Feature | |
| 0x1C | Physical Layer 16.0 GT/s | |
| 0x1D | Physical Layer 32.0 GT/s | |
| 0x1E | Physical Layer 64.0 GT/s | |
| 0x1F | Lane Margining | |
| 0x23 | ASPM | L0s/L1 latency, ASPM disable |
| 0x25 | ARI | ARI capability (next function number) |
| 0x27 | ATS | Address translation services, invalidate queue depth |
| 0x28 | SR-IOV | Total/busy VFs, VF BARs, VF stride/offset |
| 0x2E | ACS | Source validation, translation blocking, P2P |
| 0x34 | DPC | Downstream port containment |

### ASPM

- **L0s**: Entry ~100ns, exit ~1us. Software transparent. Per-direction (Rx/Tx independent).
- **L1**: Entry ~1–3us, exit ~10–20us. Both directions enter. Saves more power.
- **L1.1/L1.2**: Additional sub-states (CLKREQ\# based). ~10us exit, ~100us exit respectively.

Link Control register ASPM field: 00=disabled, 10=L0s, 01=L1, 11=L0s+L1.

## 5. Virtualization

### VMCS Field Categories

| Encoding Range | Category |
|---------------|----------|
| 0x0000–0x0017 | Guest-state selectors (ES, CS, SS, DS, FS, GS, TR, LDTR) |
| 0x0400–0x0442 | Host-state selectors |
| 0x0800–0x0804 | Control fields (pin-based, processor-based, exec, exception bitmap) |
| 0x0C00–0x0C1E | Secondary controls, VM functions |
| 0x1000–0x1028 | VM-exit control fields |
| 0x1400–0x1404 | VM-entry control fields |
| 0x2000–0x2012 | VM-exit info (reason, qualification, guest linear, guest physical) |
| 0x2400–0x2404 | Guest register state (RIP, RSP, RFLAGS) |
| 0x2800–0x2808 | Host register state (RIP, RSP) |
| 0x2C00–0x2C04 | CR0/CR3/CR4 read/write shadows |
| 0x4000–0x401F | Guest MSR area (count, address) |
| 0x4400–0x4401 | Host MSR area |
| 0x4800–0x4810 | VMCS link pointer, IA32_DEBUGCTL, etc. |
| 0x6000–0x6008 | EPTP, VPID, posted interrupt |
| 0x6400–0x640E | EOI-exit bitmap, PML |
| 0x6800–0x6802 | Virtualization exception, XSS bitmap |

### VM-Exit Reasons (Key)

| Exit Reason | Name | Description |
|-------------|------|-------------|
| 0 | EXCEPTION_OR_NMI | Guest exception or NMI |
| 1 | EXTERNAL_INTERRUPT | Physical interrupt arrived |
| 2 | TRIPLE_FAULT | Triple fault in guest |
| 3 | INIT_SIGNAL | INIT signal received |
| 4 | SIPI | Startup IPI received |
| 5 | IO_SMI | SMI received during I/O |
| 6 | OTHER_SMI | Other SMI |
| 7 | INTERRUPT_WINDOW | Interrupt window open (IF=1) |
| 8 | NMI_WINDOW | NMI window open |
| 9 | TASK_SWITCH | Task switch |
| 10 | CPUID | CPUID instruction |
| 12 | HLT | HLT instruction |
| 13 | INVD | INVD instruction |
| 14 | INVLPG | INVLPG instruction |
| 15 | RDPMC | RDPMC instruction |
| 16 | RDTSC | RDTSC instruction |
| 17 | RSM | RSM instruction (not in VMX) |
| 18 | VMCALL | VMCALL instruction |
| 19 | VMCLEAR | VMCLEAR instruction |
| 20 | VMLAUNCH | VMLAUNCH instruction |
| 21 | VMPTRLD | VMPTRLD instruction |
| 22 | VMPTRST | VMPTRST instruction |
| 23 | VMREAD | VMREAD instruction |
| 24 | VMRESUME | VMRESUME instruction |
| 25 | VMWRITE | VMWRITE instruction |
| 26 | VMXOFF | VMXOFF instruction |
| 27 | VMXON | VMXON instruction |
| 28 | CR_ACCESS | Control register access (CR0/CR3/CR4/CR8) |
| 29 | DR_ACCESS | Debug register access |
| 30 | IO_INSTRUCTION | IN/OUT instruction |
| 31 | RDMSR | RDMSR instruction |
| 32 | WRMSR | WRMSR instruction |
| 33 | VM_ENTRY_FAILURE | VM-entry failure |
| 34 | MWAIT | MWAIT instruction |
| 36 | MONITOR | MONITOR instruction |
| 37 | PAUSE | PAUSE instruction |
| 39 | MWAIT_COND | MWAIT with condition met |
| 40 | SHUTDOWN | Guest shutdown |
| 43 | TPR_BELOW_THRESHOLD | TPR below virtual APIC threshold |
| 44 | APIC_ACCESS | APIC access (MMIO or virtual) |
| 45 | EOI_INDUCED | Virtualized EOI |
| 46 | GDTR_IDTR | GDTR/IDTR access |
| 47 | LDTR_TR | LDTR/TR access |
| 48 | EPT_VIOLATION | EPT violation |
| 49 | EPT_MISCONFIG | EPT misconfiguration |
| 50 | INVEPT | INVEPT instruction |
| 51 | RDTSCP | RDTSCP instruction |
| 52 | PREEMPTION_TIMER | VMX preemption timer expired |
| 53 | INVVPID | INVVPID instruction |
| 54 | WBINVD | WBINVD instruction |
| 55 | XSETBV | XSETBV instruction |
| 56 | APIC_WRITE | APIC write to virtual APIC |
| 57 | RDRAND | RDRAND instruction |
| 58 | INVPCID | INVPCID instruction |
| 61 | VMFUNC | VMFUNC instruction |
| 62 | ENCLS | ENCLS instruction (SGX) |
| 64 | XSAVES | XSAVES instruction |
| 65 | XRSTORS | XRSTORS instruction |
| 66 | SPP | Sub-page permission |
| 67 | UMWAIT | UMWAIT instruction |
| 68 | TPAUSE | TPAUSE instruction |
| 69 | LOADIWKEY | LOADIWKEY instruction |
| 70 | ENCLV | ENCLV instruction (SGX) |

### VT-d DMAR Structure Types

| Type | Name | Description |
|------|------|-------------|
| 0 | DRHD | DMA Remapping Hardware Unit Definition (base address, flags, device scope) |
| 1 | RMRR | Reserved Memory Region Reporting (BIOS-reserved DMA regions) |
| 2 | ATSR | Root Port ATS Capability (list of root ports supporting ATS) |
| 3 | RHSA | Remapping Hardware Static Affinity (NUMA proximity domain) |
| 4 | ANDD | ACPI Name-space Device Declaration |

### EPT Exit Qualification Bits

| Bit | Name | Description |
|-----|------|-------------|
| 0 | Read | Read access caused violation |
| 1 | Write | Write access caused violation |
| 2 | Execute | Execute access caused violation |
| 3 | Reserved | Page-table entry reserved bit set |
| 4 | Readable | Was the page readable in EPT? |
| 5 | Writable | Was the page writable in EPT? |
| 6 | Executable | Was the page executable in EPT? |
| 7 | Valid | Guest linear address valid |
| 8 | Linear address | Faulting guest linear address present |
| 9 | NMI unmasking | Violation during NMI unmasking |

### APICv Features

| VMX Control | Feature | Description |
|-------------|---------|-------------|
| Virtual APIC (proc-based bit 0) | Virtualize APIC accesses | APIC reads/writes are virtualized |
| TPR shadow (proc-based bit 1) | TPR virtualization | TPR writes cause VM exit only if below threshold |
| Virtual-interrupt delivery (2nd-based bit 9) | VID | Posted interrupts evaluated on VM entry |
| APIC-register virtualization (2nd-based bit 0) | APICv reg | All APIC regs virtualized (not just TPR) |
| Posted interrupts (pin-based bit 7) | PI | External interrupts → posted to VM without exit |
| Virtualize x2APIC (2nd-based bit 4) | x2APIC mode | MSR-based APIC access virtualized |

## 6. SMM & Security

### SMRAM Save State Area (Intel, offset from SMBASE + 0x8000 + offset)

| Offset | Register | Offset | Register |
|--------|----------|--------|----------|
| 0xFFD8 | Reserved | 0xFFC8 | DR7 |
| 0xFFD0 | DR6 | 0xFFC0 | RFLAGS |
| 0xFFB8 | RIP | 0xFFB0 | R15 |
| 0xFFA8 | R14 | 0xFFA0 | R13 |
| 0xFF98 | R12 | 0xFF90 | R11 |
| 0xFF88 | R10 | 0xFF80 | R9 |
| 0xFF78 | R8 | 0xFF70 | RDI |
| 0xFF68 | RSI | 0xFF60 | RBP |
| 0xFF58 | RSP | 0xFF50 | RBX |
| 0xFF48 | RDX | 0xFF40 | RCX |
| 0xFF38 | RAX | 0xFF34 | IOMEMBASE |
| 0xFF30 | CR4 | 0xFF28 | CR3 |
| 0xFF20 | CR0 | 0xFF1C | Auto HALT Restart |
| 0xFF18 | I/O Restart | 0xFF10 | SMBASE (writable to relocate) |
| 0xFF08 | SMM Revision ID | 0xFF00 | CPU Vendor ID (signature) |
| 0xFEF8 | ES | 0xFEF0 | CS |
| 0xFEE8 | SS | 0xFEE0 | DS |
| 0xFED8 | FS | 0xFED0 | GS |
| 0xFEC8 | LDT | 0xFEC0 | TR |
| 0xFEB8 | IDTR base | 0xFEB0 | IDTR limit |
| 0xFEA8 | GDTR base | 0xFEA0 | GDTR limit |
| 0xFE98 | IO restart RIP | 0xFE90 | IO restart RCX |
| 0xFE88 | IO restart RSI | 0xFE80 | IO restart RDI |
| 0xFE00 | I/O trap (4 DWORDs) | 0xFEF0 | GDTR/IDTR shadow |

### SMI Entry/Exit Flow

**Entry:**
1. CPU saves state to SMBASE + 0x8000 (save state area)
2. Sets RIP = SMBASE + 0x8000 (start of SMI handler)
3. CS base = SMBASE, CS limit = 0xFFFFFFFF
4. CR0: PG=0, PE=1, all other bits preserved
5. EFLAGS: IF=0, TF=0, RF=1
6. SMRAM opens (D_OPEN), DR7=0, all debug disabled
7. CPU is in SMM (real-mode-like, 64-bit on capable CPUs)

**Exit (RSM):**
1. Restore all registers from save state
2. If SMBASE was modified in save state, update SMBASE for next SMI
3. SMRAM closes
4. Resume execution at restored RIP

### STM (Dual Monitor Model)

- SMI handler can VMXON and enter STM (SMM Transfer Monitor)
- VMCS used: MSEG (STM VMCS) at SMBASE + 0x8000
- VM-exit from STM → SMM guest via VMRESUME
- BIOS enables STM via TXT measured launch or VMXON in SMM

### SGX Leaf Functions

**ENCLS (Supervisor, ring 0):**

| Leaf | Name | Description |
|------|------|-------------|
| ECREATE | ECREATE | Create enclave in EPC |
| EADD | EADD | Add page to enclave |
| EINIT | EINIT | Initialize enclave (verify signature) |
| EREMOVE | EREMOVE | Remove page from EPC |
| EDGRD | EDBGRD | Read enclave data (debug only) |
| EDGWRT | EDBGWR | Write enclave data (debug only) |
| EEXTEND | EEXTEND | Extend measurement (256-byte chunk) |
| ELDB/ELDU | ELDB/ELDU | Load blocked/unblocked page |
| EBLOCK | EBLOCK | Block page (prevent further access) |
| EPA | EPA | Add version array |
| EWB | EWB | Evict page (write back to main memory) |
| ETUU | ETUU | Track unblocked |
| EMODPE | EMODPE | Extend page permissions |
| EMODPR | EMODPR | Restrict page permissions |
| EMODT | EMODT | Modify page type |
| EACCEPT | EACCEPT | Accept (enclave confirms changes) |
| EAUG | EAUG | Add pages (elasticity) |
| EMODT | EMODT | Modify page type (accept needed) |

**ENCLU (User, ring 3):**

| Leaf | Name | Description |
|------|------|-------------|
| EREPORT | EREPORT | Generate report (local attestation) |
| EGETKEY | EGETKEY | Derive sealing/reporting key |
| EENTER | EENTER | Enter enclave |
| ERESUME | ERESUME | Resume enclave after AEX |
| EEXIT | EEXIT | Exit enclave |
| EACCEPT | EACCEPT | Accept enclave modification |
| EACCEPTCOPY | EACCEPTCOPY | Accept and copy page |
| EMODPE | EMODPE | Modify page permissions from within |
| EVERIFYREPORT | EVERIFYREPORT | Verify report (newer CPUs) |

### CET Shadow Stack

| Register/MSR | Description |
|-------------|-------------|
| IA32_U_CET (0x6A0) | User CET enable: SH_STK_EN (bit 0), WR_SHAD_EN (bit 1), ENDBR_EN (bit 2), LEG_IW (bit 3), NO_TRACK (bit 4), SUPPRESS_DIS (bit 5), INDIRECT_BRANCH_TRACKER (bits 11:10) |
| IA32_S_CET (0x6A2) | Supervisor CET enable (same bit layout) |
| IA32_PL3_SSP (0x6A7) | User shadow stack pointer (RPL3) |
| IA32_INTERRUPT_SSP_TABLE_ADDR (0x6A8) | Shadow stack interrupt table |
| SSP | Current shadow stack pointer (MSR or register) |

ENDBR64/ENDBR32: Mark valid indirect branch targets (NOP on non-CET hardware).

### TXT Measured Launch Sequence

1. CPU supports TXT: IA32_FEATURE_CONTROL bit 6 (SMX enable)
2. BIOS creates SINIT ACM (Authenticated Code Module), places in memory
3. GETSEC[SENTER]: Enter measured environment
   - Verifies SINIT ACM signature
   - Measures ACM + policy into PCRs (TPM)
   - Enables measured SMM (STM possible)
4. GETSEC[SEXIT]: Exit measured environment

Key MSRs: TXT_PUBLIC_KEY (0x372), TXT_HEAP_BASE (0x400), TXT_SINIT_BASE, TPM hash log.

## 7. Power Management

### C-States with MWAIT Hints

| State | MWAX Hint (EAX) | Description | Typical Latency |
|-------|------------------|-------------|-----------------|
| C0 | — | Active (executing) | 0 |
| C1 | 0x00 | Halt (MWAIT with hint 0) | ~1us |
| C1E | 0x01 | Enhanced halt (lower freq + voltage) | ~1us |
| C2 | 0x10 | Stop clock (P6 era) | ~1–10us |
| C3 | 0x20 | Deep sleep (clock stopped, cache flushed) | ~40–60us |
| C4 | 0x30 | Deeper sleep (clock stopped, reduced voltage) | ~100us |
| C4S | 0x31 | C4 with clock gating | ~100us |
| C5 | 0x40 | Reduced voltage further | ~150us |
| C6 | 0x50 | Reduced voltage to 0 (complete power gate) | ~200us |
| C6 | 0x51 | C6 with retention | ~200us |
| C7 | 0x60 | C6 + LLC ways power gated | ~200us |
| C7s | 0x64 | C7 sub-state | ~200us |
| C8 | 0x70 | Additional LLC and PLL gating | ~300us |
| C9 | 0x80 | Further PLL gating | ~400us |
| C10 | 0x90 | Maximum power savings | ~500us+ |

MWAIT hints above 0x0F: sub-states are encoded in bits 7:4, extensions in bits 3:0. Bit 0 of ECX = interrupt break event (break on IRQ even if masked).

### P-States (IA32_PERF_CTL 0x199)

| Bits | Field | Description |
|------|-------|-------------|
| 15:8 | Target Ratio | Target P-state bus ratio (e.g., 0x14 = 20x) |
| 14 | IDA Enable | Intel Dynamic Acceleration enable |
| 0 | Target State | 0=lowest non-idle P-state |

Transition: write target ratio → hardware transitions frequency/voltage. OS reads IA32_APERF/IA32_MPERF for actual vs nominal frequency.

### MWAIT/MONITOR Usage

```asm
; MONITOR: sets up address range for monitoring
; MONITOR EAX(LinearAddr), ECX(Extensions), EDX(Hints)
mov eax, monitoring_address
mov ecx, 0            ; extensions: 0
mov edx, 0            ; hints: 0
monitor

; MWAIT: wait for write to monitored range
; MWAIT EAX(Hints), ECX(Extensions)
mov eax, hint_value   ; e.g., 0x50 for C6
mov ecx, 0            ; 0=break on masked IRQs too? bit 0=break if masked
mwait
```

Monitor range size: CPUID.05H:EAX[bits 15:0] (minimum, typically 64 bytes). Line must be aligned.

### Intel HWP (Hardware P-Performance)

| MSR | Name | Description |
|-----|------|-------------|
| 0x770 | IA32_HWP_CAPABILITIES | Highest/Lowest/Guaranteed/Most Efficient performance |
| 0x771 | IA32_HWP_REQUEST | Min/Max performance, Desired, Energy Perf Preference |
| 0x772 | IA32_HWP_STATUS | Activity window, transition latency |
| 0x773 | IA32_HWP_INTERRUPT | Thermal/OC events |
| 0x774 | IA32_HWP_REQUEST_PKG | Package-level HWP request (if HWP_PKG_CTL) |

HWP enable: set bit 0 of MSR 0x770 after setting IA32_PM_ENABLE[0]=1.

### RTD3 (Runtime D3)

- Device enters D3cold when idle; power fully removed
- Firmware signals readiness via ACPI _PR3/_PR0 power resources
- Re-entry requires re-initialization (PCI config, BARs, firmware download)
- Common for NVMe, USB controllers, WiFi on mobile platforms

## 8. Platform Interfaces

### ACPI Table Signatures

| Signature | Name | Key Fields |
|-----------|------|------------|
| DSDT | Differentiated System Description Table | Full AML bytecode for system devices |
| SSDT | Secondary System Description Table | Additional AML (hot-plug, dynamic devices) |
| FACP (FADT) | Fixed ACPI Description Table | PM1a/b_EVT/CNT, RESET_REG, SCI_INT, DSDT pointer |
| APIC (MADT) | Multiple APIC Description Table | LAPIC addr, IOAPIC entries, overrides, NMI sources |
| DMAR | DMA Remapping | DRHD, RMRR entries for VT-d |
| BGRT | Boot Graphics Resource Table | Logo framebuffer address, format |
| MCFG | PCI Express MMCFG | Base address, bus range for ECAM |
| IORT | IO Remapping Table | SMMU/IOMMU, ITS groups for ARM (also used on x86 for VT-d) |
| CSRT | Core System Resource Table | DMA controllers, I2C, SPI resources |
| NFIT | NVDIMM Firmware Interface Table | Persistent memory regions, interleave |
| FPDT | Firmware Performance Data Table | Boot performance timestamps |
| HPET | High Precision Event Timer | Block ID, base address, min tick |
| SRAT | System Resource Affinity | NUMA proximity domains, APIC/UID affinity |
| SLIT | System Locality Information | NUMA latency matrix |
| TCPA | Trusted Computing Platform | TPM event log |
| TPM2 | TPM 2.0 | TPM2 event log location |
| WAET | Windows ACPI Emulated | Timer emulation hints |
| WSMT | Windows SMM Security | SMM communication protections |
| TPM2 | TPM2 Table | TPM2 interface type, log area |

**ACPI Table Header Format (first 36 bytes of every table):**

| Offset | Length | Field |
|--------|--------|-------|
| 0 | 4 | Signature |
| 4 | 4 | Length (total table) |
| 8 | 1 | Revision |
| 9 | 1 | Checksum |
| 10 | 6 | OEM ID |
| 16 | 8 | OEM Table ID |
| 24 | 4 | OEM Revision |
| 28 | 4 | Creator ID |
| 32 | 4 | Creator Revision |

### SMBIOS

**Entry Point (32-bit):**

| Offset | Length | Field |
|--------|--------|-------|
| 0 | 4 | Anchor String: "\_SM\_" |
| 4 | 1 | Entry Point Length |
| 5 | 1 | Major Version |
| 6 | 1 | Minor Version |
| 7 | 2 | Maximum Structure Size |
| 9 | 1 | Entry Point Revision |
| 10 | 5 | Formatted Area |
| 15 | 5 | Intermediate Anchor: "\_DMI\_" |
| 18 | 1 | Intermediate Length |
| 19 | 2 | Structure Table Length |
| 21 | 4 | Structure Table Address |
| 25 | 2 | Number of Structures |
| 27 | 1 | BCD Revision |

**64-bit (3.x) Entry Point:** Anchor "\_SM3\_", length 24, max size 16, address 8 bytes.

### SMBIOS Key Structure Types

| Type | Name | Key Fields |
|------|------|------------|
| 0 | BIOS Information | Vendor, Version, Start Segment, ROM Size |
| 1 | System Information | Manufacturer, Product, Serial, UUID, SKU |
| 2 | Baseboard | Manufacturer, Product, Serial, Asset Tag |
| 3 | Chassis | Manufacturer, Type, Lock, Serial, Asset Tag |
| 4 | Processor | Socket, Type, Family, Speed, Status, Core Count |
| 9 | System Slots | Slot Designation, Type, Bus/Dev/Func, Status |
| 11 | OEM Strings | Count + string set |
| 16 | Physical Memory Array | Location, Use, Error Correction, Max Capacity |
| 17 | Memory Device | Locator, Bank, Speed, Size, Manufacturer, Serial |
| 18 | 32-bit Memory Error | Error Type, Granularity, Address |
| 19 | Memory Array Mapped Address | Start/End Address, Array Handle |
| 38 | IPMI Device | Interface Type, Base Address, IRQ |
| 43 | TPM Device | Vendor, Major/Minor Version, Firmware |

### MPTable Floating Pointer

| Offset | Length | Field | Description |
|--------|--------|-------|-------------|
| 0 | 4 | Signature | "\_MP\_" |
| 4 | 4 | MP Table Address | Physical address of MP configuration table |
| 8 | 1 | Length | Floating pointer structure size / 16 |
| 9 | 1 | Spec Revision | 1 or 4 |
| 10 | 1 | Checksum | |
| 11 | 1 | MP Feature Byte 1 | IMCR (bit 7), configured (bits 0–6) |
| 12 | 4 | MP Feature Byte 2 | |
| 16 | 4 | MP Feature Byte 3 | |

Found by scanning: 0xF0000–0xFFFFF, EBDA first 1KB, or BIOS ROM.

### SPI Flash Descriptor Region Layout

| Region | Offset (from descriptor start) | Description |
|--------|-------------------------------|-------------|
| Descriptor | 0x0000–0x0FFF | Flash descriptor itself (BIOS maps this) |
| BIOS | Variable (in region map) | BIOS firmware region |
| ME (Intel ME) | Variable | Management engine firmware |
| GbE | Variable | Gigabit Ethernet firmware/config |
| Platform Data | Variable | Platform-specific data |

Region map in descriptor at offset 0x04 (Flash Region 0–4 offset/size registers). Flash master (CPU, ME, GbE) read/write permissions at offset 0x0C.

### LPC Decode Ranges

| Range | Name | Description |
|-------|------|-------------|
| 0x0000–0x0FFF | DMA | Legacy DMA registers |
| 0x0010–0x003F | DMA | DMA command/status |
| 0x0040–0x0043 | PIT | Programmable interval timer |
| 0x0060–0x0064 | Keyboard | Keyboard controller (8042) |
| 0x0070–0x0071 | RTC | Real-time clock / CMOS |
| 0x0080–0x008F | DMA Page | DMA page registers |
| 0x0092 | PS/2 | System control port A (fast A20 gate) |
| 0x00A0–0x00BF | PIC2 | Secondary interrupt controller |
| 0x00C0–0x00DF | DMA | DMA channel registers |
| 0x00F0 | FPU | Coprocessor error |
| 0x0170–0x0177 | IDE2 | Secondary IDE controller |
| 0x01F0–0x01F7 | IDE1 | Primary IDE controller |
| 0x0376 | IDE2 | Secondary IDE command |
| 0x03F6 | IDE1 | Primary IDE command |
| 0x03F8–0x03FF | COM1 | Serial port 1 |
| 0x02F8–0x02FF | COM2 | Serial port 2 |
| 0x03E8–0x03EF | COM3 | Serial port 3 |
| 0x02E8–0x02EF | COM4 | Serial port 4 |
| 0x0278–0x027F | LPT1 | Parallel port |
| 0x0378–0x037F | LPT2 | Parallel port |
| 0x04D0–0x04D1 | ELCR | Edge/level control registers |

### Intel PCH GPIO Pad Register Layout

**DW0 (PADCFG_DW0):**

| Bits | Field | Description |
|------|-------|-------------|
| 0 | GPIOTXSTATE | Pad output value |
| 1 | GPIOTXSTATEEN | Output state enable (tristate) |
| 2 | GPIORXSTATE | Pad input value (read-only) |
| 3 | GPIORXINV | Invert input |
| 4–6 | Pad Mode | 0=GPIO, 1–7=native functions |
| 8–10 | GPIROUTIOXAPIC | Route to IOxAPIC |
| 9 | GPIROUTSCI | Route SCI |
| 10 | GPIROUTSMI | Route SMI |
| 11 | GPIROUTNMI | Route NMI |
| 12 | RXEVCFG | Pad Rx event: 0=raw, 1=edge, 2=disabled, 3=both edges |
| 13 | RXINV1 | Additional invert |
| 14–16 | PREGFRXSEL | Filter select |
| 17 | RXFILTER1EN | Glitch filter 1 enable |
| 18–19 | RXFILTER1SEL | Glitch filter config |
| 20 | RXFILTER2EN | Glitch filter 2 enable |
| 21 | Pad Reset | Reset type: 0=power good, 1=deep, 2=global, 3=resume |
| 23 | PADRSTCFG | Reset config override |
| 24–25 | RXPADSTATESEL | Raw/internal signal select |
| 26 | Pad TOL | Voltage tolerance: 0=3.3V, 1=1.8V |
| 27 | Terminate | Termination: 0=none, 1=PD, 2=PU, 4=keepers |
| 28–30 | Term | Pull config: 0=none, 1=PD, 2=PU, 3=N/A, 4=keeper |
| 31 | DEBOUNCE | Debounce enable |

**DW1 (PADCFG_DW1):**

| Bits | Field | Description |
|------|-------|-------------|
| 0–9 | INTSEL | Interrupt line select (which PAD IRQ) |
| 10–12 | PADTOL | Additional pad tolerance control |
| 16–24 | DEBOUNCE_SCALE | Debounce period scale |
| 25 | DEBOUNCE_EN | Debounce enable per pad |
| 26–31 | IOSSTBY | I/O standby config |

### AMD GPIO Bank Register Layout

| Offset | Register | Description |
|--------|----------|-------------|
| 0x00 | PIN_MUX | Function select (bits 1:0 = pin function) |
| 0x04 | PIN_CFG | Pull up/down, drive strength, receiver enable |
| 0x08 | PIN_INT | Interrupt enable (bit 0), type (bits 1:2: 0=edge, 1=level, 2=both edges), polarity (bit 3) |
| 0x0C | PIN_WAKE | Wake control, filter select |
| 0x10 | PIN_STATUS | Pin state (read-only) |

Each bank: 4 GPIO pins per group, up to 4 groups per bank. Total GPIO count is SoC-specific.

## 9. Interconnects

### UPI (Ultra Path Interconnect)

- Intel's coherent interconnect for multi-socket and die-to-die
- Replaces QPI (QuickPath Interconnect)
- Up to 10.4 GT/s per link, up to 3 links per socket
- Each link: 20 lanes (full width) or 10 lanes (half width)
- Protocol layers: Home, Snoop, Data, Non-Coherent
- Address-based routing with directory-based snoop filter

### DMI (Direct Media Interface)

- Connects CPU to PCH (Platform Controller Hub)
- Gen3 DMI: x4 lanes, ~8 GT/s per lane (~4 GB/s total)
- Gen4 DMI: x4 lanes, ~16 GT/s per lane (~8 GB/s total)
- Gen5 DMI: x4 lanes, ~32 GT/s per lane
- Electrically identical to PCIe; appears as PCIe root complex to software
- DMI is PCH-attached devices' path to CPU memory controller

### PCIe Bifurcation

- Single x16 slot can split: x16 / x8x8 / x4x4x4x4
- Controlled by BIOS strap or runtime config
- Per-port configuration in PCIe Cap register or root port PCI config
- Requires same GEN speed for all split links
- Active-state power management per link

### CXL (Compute Express Link)

- Runs over PCIe 5.0/6.0 physical layer
- Three protocol types:
  - **CXL.io**: PCIe-compatible I/O (discovery, config, interrupts)
  - **CXL.cache**: Coherent cache protocol (device caches host memory)
  - **CXL.mem**: Memory protocol (host accesses device-attached memory)
- Three device types:
  - **Type 1**: CXL.io + CXL.cache (accelerators with coherent cache)
  - **Type 2**: CXL.io + CXL.cache + CXL.mem (accelerators with HBM)
  - **Type 3**: CXL.io + CXL.mem (memory expanders, no cache)
- CXL 2.0: Switching, fabric, DDR regions
- CXL 3.0: 64 GT/s, enhanced fabric, peer-to-peer

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing to CR3 without invalidating TLB | Use MOV to CR3 (flushes non-global) or INVLPG for specific entries; set CR4.PGE=1 and use global pages for kernel mappings |
| Enabling paging before setting up identity-mapped page tables | Always create identity mappings (VA=PA) for the code executing during the CR0.PG=1 transition |
| Forgetting to set CR0.WP=1 before writing to read-only pages from CPL0 | WP protects supervisor writes; many kernels set it early for copy-on-write and security |
| Using CPUID without checking max supported leaf | Always check EAX output of basic leaf 0 before querying higher leaves; check 0x80000000 for extended |
| Accessing APIC at wrong base address | Default base is 0xFEE00000 (from IA32_APIC_BASE MSR); must be mapped as UC/strong-uncacheable |
| Not handling EOI in interrupt handlers | Always write to LAPIC EOI (0xFEE000B0) before IRET; otherwise no further interrupts delivered |
| Programming MSI without enabling bus master in PCI Command register | Set PCI Command bit 2 (Bus Master Enable) or DMA/MSI transactions won't be issued |
| Using WRMSR to MSRs that require 64-bit writes | Always use `rdmsr; modify; wrmsr` pattern — never partially write a 64-bit MSR from 32-bit code |
| Ignoring MTRR/PAT conflict resolution | When MTRR and PAT conflict, MTRR takes precedence for write-back vs UC; use PAT UC- to let MTRR decide |
| Forgetting VMXOFF after VMXON | Always pair VMXON/VMXOFF; on shutdown path, VMXOFF before disabling paging or entering real mode |
| Using INT 15h/E820 in UEFI mode | Use GetMemoryMap() boot service instead; INT 15h is legacy BIOS only |
| Writing to BAR before sizing | Always save original BAR, write 0xFFFFFFFF, read size, restore original — wrong order corrupts the BAR |
| Not flushing caches before entering SMM or changing MTRRs | Use WBINVD before MTRR changes and SMRAM setup; stale cache lines can cause corruption |
| Enabling x2APIC without proper CPUID check | CPUID.01H:ECX[21]=1 required; must transition from xAPIC (EN=1, EXTD=0) to x2APIC (EN=1, EXTD=1) via MSR 0x1B |
