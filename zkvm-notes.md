# ZK Whiteboard session RISC-V zkVMs (SP1/Succinct) - Notes

> Uma Roy’s [ZK Whiteboard session](https://www.youtube.com/watch?v=Y4kIgPm95WM)

## 
A **zkVM** lets you write ordinary programs (e.g., **Rust**), compile them to **RISC‑V**, and then prove **the correctness of execution** using **STARKs**. SP1 (Succinct) uses a **lookup‑centric multi‑table** STARK design, **offline memory checking** (no per‑access Merkle proofs), **precompiles** for heavy ops like **Keccak**, **sharding + recursion** for long traces, and then wraps the STARK into a **SNARK** for cheap **Ethereum** verification.

---

## 1) What is a zkVM? Why RISC‑V?
**zkVM** = *Zero‑Knowledge Virtual Machine.*  
Classic ZK: hand‑write a circuit for each computation.  
zkVM: write **normal code**, compile to an **ISA**, and **prove execution** of that ISA.

**Why RISC‑V (RV32IM)** in SP1 :
- Open, small, well‑documented **Reduced Instruction Set**.
- Standard target for **Rust** (no compiler forks).
- RV32 base + **M extension** for multiplication (trickier to constrain but supported).

```
High-Level Flow (conceptual)
============================
Rust program
   │ compile
   ▼
RISC-V ELF
   │ emulate (PC, branches, HALT, ECALL)
   ▼
STARK proof of correct execution
```

---

## 2) Build & Prove Pipeline (High-Level)
1) **Write** in Rust  
2) **Compile** to **RISC‑V -> ELF**  
3) **Execute** with a RISC‑V **emulator/runtime**  
   - **Program Counter (PC)** drives fetch/decode/execute
   - **JAL/branches** alter PC; **HALT/exit** stops
   - **Inputs/outputs via ECALL** (syscalls)
4) **STARK proving** (AIR + FRI) over a **small field** (BabyBear/M31)

```
Pipeline (ASCII)
----------------
[ Rust ] -> [ RISC-V ELF ] -> [ Emulator trace ] -> [ STARKs (AIR/FRI) ]
```

---

## 3) From ELF to Constraints: The CPU Table
A **CPU table** has **one row per instruction** with columns like: `opcode | operands | PC | witnesses (e.g., carries)`.

Per-row constraints enforce:
- **PC transition** correctness (sequential vs branch/jump)
- **Register & memory** reads/writes are consistent
- **Opcode semantics** (e.g., ADD: `dest = src1 + src2`)

```
CPU Table (schematic rows)
--------------------------
Row i:    opcode | ops | PC | ...witness...
Row i+1:  opcode | ops | PC | ...witness...
   ...

Constraints:
- PC(i+1) correct given Row i
- Reg/Mem after Row i match opcode semantics
```

---

## 4) Small Fields & 32‑bit Values (Limb Arithmetic)
- STARKs commonly use **small prime fields** (e.g., **BabyBear**, **M31**).
- RISC‑V words are **32 bits**; the field is **< 2^32**.  
  ⇒ Represent each 32‑bit word as **4 limbs** (field elements).  
- **Grade‑school arithmetic** with carries constrains adds/mults across limbs.

```
Limb Model (base 2^8 example)
-----------------------------
x = x0 + 2^8 x1 + 2^16 x2 + 2^24 x3
y = y0 + 2^8 y1 + 2^16 y2 + 2^24 y3
z = x + y
Constrain limbwise with carries c0..c3.
```

---

## 5) Lookup‑Centric, Multi‑Table Architecture (SP1)
**Goal:** Avoid paying for complex witnesses (e.g., MUL) on every instruction.

**Design:**
- **CPU table** does minimal work (fetch/PC/memory wiring).
- CPU then **LOOKUPs** into **opcode‑specific tables**: `ADD`, `AND`, `MUL`, `SHIFT`, …
- **Auxiliary tables** carry opcode‑specific constraints and witnesses.
- Tables/STARKs are linked via the **log‑derivative lookup argument**.

```
Lookup Architecture (ASCII)
---------------------------
                +-----------------+
                |   CPU (thin)    |
                |  - fetch/PC     |
                |  - read operands|
                |  - minimal chk  |
                +----+----+---+---+
                     |    |   |
   lookup (a,b,c)->  |    |   |  <- lookup (a,b,c,...)
                     |    |   |
                 +---v+ +--v-+ +--v---+
                 |ADD | |AND| |  MUL  |  ... (SHIFT, etc.)
                 +----+ +----+ +------+
                (narrow)(narrow)(wide, many witnesses)
```

**Effect:** Significant **trace‑area** savings vs a single, bloated CPU table.

---

## 6) Inputs & Outputs via ECALL (Syscalls)
SP1 uses **RISC‑V ECALL** for program I/O and precompiles.

```
ECALL Selectors (examples)
--------------------------
READ_INPUT    -> load input bytes to regs/mem
WRITE_OUTPUT  -> write output bytes
KECCAK(ptr)   -> precompile: hash data at memory pointer
```

---

## 7) Memory: Merkle (Old) vs Offline Memory Checking (SP1)
**Merkle‑ized memory (older):**
- Commit memory to a Merkle root; each access proves a Merkle path; writes update the root (**O(log n)** per access).

**Offline Memory Checking (SP1):**
- Keep a strictly increasing **timestamp** `ts` per instruction.
- For an access to `(addr, value_now, ts_now)`:
  - Add **+ (addr, value_now, ts_now)** to a global accumulator.
  - Add **− (addr, value_prev, ts_prev)** for the previous access to `addr`.
- **Reads:** `value_now == value_prev`. **Writes:** value changes.
- Enforce `ts_now > ts_prev` per address.
- **End:** all tuples **cancel** ⇒ accumulator **== 0** iff consistent & ordered.

```
Offline Memory Accumulator (ASCII)
----------------------------------
For each access:
  + (addr, value_now, ts_now)
  - (addr, value_prev, ts_prev)

Reads: value_now == value_prev
Writes: value_now may differ
At end: sum over all tuples == 0
```

**Mini Example (from talk):**
```
+ (addr=500, val=7,  ts=10)
- (addr=500, val=0,  ts=0 )   // init
+ (addr=500, val=7,  ts=11)
- (addr=500, val=7,  ts=10)
+ (addr=500, val=15, ts=20)
- (addr=500, val=7,  ts=11)
--------------------------------
All cancel -> accumulator = 0
```

---

## 8) Precompiles / Accelerators (Keccak example)
- Naively compiling **Keccak** to RISC‑V can take ~**10k cycles** (order‑of‑magnitude in talk).
- SP1 uses **ECALL** to trigger a **Keccak table**:
  - Pass a **pointer** to input.
  - A **wide** table performs the permutation in **dozens of rows**, talking directly to memory.
- Replaces “many CPU rows × ~200 cols” with “dozens rows × ~3000 cols” ⇒ **order‑of‑magnitude** fewer cells.
- **Dev UX:** use a **forked Rust crate** that swaps inner loops with ECALL; keep the same API (or `cargo patch`).

```
Keccak Precompile Flow (ASCII)
------------------------------
[ Program ] --ECALL:KECCAK(ptr)--> [ CPU ]
      CPU --lookup--> [ Keccak Table (wide, ~dozens rows) ]
 Keccak Table --write digest--> [ Memory ]
```

---

## 9) Long Programs: Sharding + STARK Recursion
- One proof fits ~hundreds of thousands to ~1M cycles.
- For **hundreds of millions/billions**:
  1) **Shard** into fixed-size chunks (e.g., ~1M instructions).  
  2) Prove each **shard STARK** (parallelizable).  
  3) **Recursively aggregate** pairs/groups into a **final STARK**.

```
Recursion Tree (ASCII)
----------------------
 [Shard 1]  [Shard 2]  [Shard 3]  [Shard 4]
     \         /           \          /
      [ Recursion ]        [ Recursion ]
              \                /
                \            /
                 [   Final STARK   ]
```

**Linking checks:**
- **PC continuity**: end PC(shard i) = start PC(shard i+1)  
- **Global memory**: sum shard accumulators ⇒ final sum == **0**

**Recursion engine:** specialized recursion VM/circuit (Poseidon, FRI fold, etc.).

---

## 10) STARK → SNARK (Ethereum)
- Direct STARK verification on Ethereum costs **millions of gas**.
- Wrap final STARK in a **pairing SNARK** on **BN254** (EVM precompiles).  
- Use **Plonkish KZG** (via **Gnark**, universal trusted setup).

**Ballpark from talk:**
- STARK: **hundreds of KB**
- SNARK: **hundreds of bytes** (few curve points)
- Verify: **~200k–300k gas**

```
On-chain Path (ASCII)
---------------------
[ Final STARK ] -> [ SNARK circuit verifies FRI/commitments ]
                         |
                         v
                   [ SNARK (BN254/KZG) ]
                         |
                         v
                   [ Ethereum verify ]
```

---

## 11) Where zkVMs Shine (Use Cases in Talk)
- **zk‑Rollups** (Ethereum scaling): batch txs, prove execution, post state root.  
- **Bridges, coprocessors, oracles**, etc.  
- **Maintenance/DX**: Reuse Rust code (e.g., parts of **Reth**); upgrades often become dependency updates rather than circuit redesigns.

---

## 12) Trade‑offs 
- **Theoretical** overhead vs bespoke circuits/zkEVMs.  
- **Practical**: shared engineering on the general VM + **precompiles** + **recursion** => competitive enough for many workloads; far easier to build/maintain.

---

## 13) Minimal Execution Pseudocode (with ECALL)
```rust
pc = entry_point;
ts = 0;
halted = false;

while !halted {
    let insn = ELF[pc];                 // fetch
    let ops  = read_operands(insn);     // logs READ (+ / −) to accumulator
    let (out, next_pc, writes) = exec_opcode(insn, ops);
    apply_writes(writes);               // logs WRITE (+ / −) to accumulator
    pc = next_pc;
    ts += 1;
    halted = is_halt(insn);
}

// ECALL selectors: READ_INPUT, WRITE_OUTPUT, KECCAK(ptr), ...
assert(memory_accumulator == 0);
```

---

## 14) Quick Glossary
- **ELF**: Executable format; compiled RISC‑V code & metadata.  
- **PC**: Program Counter; next instruction index.  
- **ECALL**: RISC‑V syscall opcode; used for I/O and precompiles in SP1.  
- **AIR**: Algebraic Intermediate Representation (constraint system for STARKs).  
- **FRI**: Fast Reed‑Solomon IOPP (low‑degree test).  
- **logUp lookup**: Log‑derivative lookup argument; links tables/STARKs.  
- **Offline memory checking**: time‑ordered `(addr, value, ts)` tuples cancel to 0 when consistent.  
- **Shard**: Fixed‑size chunk of the trace proven separately.  
- **STARK→SNARK**: Wrap final STARK inside a pairing SNARK (BN254/KZG) for cheap EVM verification.

---

