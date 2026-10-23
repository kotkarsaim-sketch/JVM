# Implementation Phases: JVM Bytecode Superoptimizer & Equivalence Obfuscator

This document specifies the technical requirements, component boundaries, algorithms, deliverables, and validation criteria for all eight implementation phases of the research prototype.

---

## Phase Overview

```
+-----------------------------------------------------------------------------------+
| Phase 1: Bytecode Analyzer & Control Flow Graph                                   |
| (ASM Tree API, Basic Block Partitioning, Opcode Classification, CFG Construction)  |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 2: JVM Integer Instruction Semantics                                        |
| (32-bit Bit-Vectors, Two's-Complement Overflow, Stack Height & Underflow Model)    |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 3: Bounded Superoptimizer Search Engine                                     |
| (STOKE-inspired Combinatorial Search, Search Pruning, Stack Delta Verification)  |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 4: Z3 SMT-Based Equivalence Verification                                    |
| (QF_BV Theory, Symbolic Execution, Solver Queries: UNSAT=Proof, SAT=Counterex)   |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 5: Bytecode Transformation & Classfile Reassembly                           |
| (ASM InsnList Splicing, FrameRecomputer, StackMapTable, CheckClassAdapter)        |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 6: Multiple Equivalent Variants & Diversity Objectives                      |
| (MIN_SIZE, STRUCTURAL_DIVERSITY, BALANCED, Deterministic Random Seed)             |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 7: Build-Time CLI & JAR Archive Pipeline                                    |
| (Reproducible Builds, Manifest/Resource Preservation, JSON Transformation Report)  |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 8: Empirical Validation, Differential Testing & Benchmarking                |
| (10,000+ Random Runs, Boundary Cases: MIN/MAX/0/1, Isolated ClassLoader, Metrics)|
+-----------------------------------------------------------------------------------+
```

---

## Phase 1: Bytecode Analyzer & Control Flow Graph

### 1.1 Objective
Parse compiled `.class` files into structured Control Flow Graphs (CFGs) and basic blocks without mutating bytecode. Detect straight-line integer instruction sequences eligible for superoptimization while safely identifying unsupported instructions and exception handler boundaries.

### 1.2 Target Components
* `com.superopt.analysis.ClassAnalyzer`: Reads `.class` bytes or files via ASM `ClassReader` and `ClassNode`.
* `com.superopt.analysis.MethodAnalyzer`: Inspects method descriptors, access flags, and instructions.
* `com.superopt.analysis.BasicBlockAnalyzer`: Partitions instruction streams into basic blocks.
* `com.superopt.analysis.ControlFlowGraph`: Graph representation with directed edges (fall-through, conditional branches, jumps).
* `com.superopt.analysis.InstructionInfo`: Opcode metadata catalog (stack push/pop effects, operand types, side effects).

### 1.3 Boundary Detection Rules
A basic block boundary is established at:
1. Target of any branch instruction (`IF_ICMPEQ`, `IFNE`, `GOTO`, etc.).
2. Target of any switch instruction (`TABLESWITCH`, `LOOKUPSWITCH`).
3. Instruction immediately following a jump or return instruction (`GOTO`, `IRETURN`, `RETURN`, `ATHROW`).
4. Beginning or end of a `try-catch` protected block (`TryCatchBlockNode`).

### 1.4 Deliverables & Verification
* CLI command: `java -jar jvm-superoptimizer.jar analyze <file.class>`.
* Unit tests in `BytecodeAnalysisTest` testing classes containing loops, nested branches, arithmetic, and exception handlers.
* Output: Human-readable terminal report summarizing classes, methods, CFG blocks, and eligible instruction sequences.

---

## Phase 2: JVM Integer Instruction Semantics

### 2.1 Objective
Construct a rigorous formal model of JVM integer arithmetic based on 32-bit two's complement bit-vectors, preserving exact JVM overflow and operand stack behavior.

### 2.2 Target Components
* `com.superopt.semantics.JvmInstructionSemantics`: Operational rules for each supported opcode.
* `com.superopt.semantics.StackEffectModel`: Stack height simulation, underflow detection, and type tracking.
* `com.superopt.semantics.IntegerSemantics`: Bit-level definitions for arithmetic, bitwise, and shift operations.

### 2.3 Supported Instruction Set (Initial Model)
* **Constants:** `ICONST_M1`, `ICONST_0` .. `ICONST_5`, `BIPUSH`, `SIPUSH`, `LDC` (integer constants).
* **Local Variables:** `ILOAD`, `ILOAD_0` .. `ILOAD_3`, `ISTORE`, `ISTORE_0` .. `ISTORE_3`.
* **Arithmetic:** `IADD`, `ISUB`, `IMUL`, `INEG`.
* **Bitwise & Shifts:** `IAND`, `IOR`, `IXOR`, `ISHL`, `ISHR`, `IUSHR`.
* **Stack Manipulation:** `DUP`, `POP`.
* **Method Return:** `IRETURN`.

### 2.4 Formal Semantic Properties
1. **Overflow Semantics:** $2^{31}-1 + 1 = -2^{31}$ (exact modular arithmetic $\mathbb{Z} / 2^{32}\mathbb{Z}$).
2. **Shift Masking:** The shift distance operand $s$ for `ISHL`, `ISHR`, `IUSHR` is masked to 5 bits ($s \ \& \ 0x1F \in [0, 31]$).
3. **Arithmetic vs Logical Shift:** `ISHR` performs sign extension; `IUSHR` shifts in zero bits.
4. **Stack Safety:** Candidate sequences that attempt to pop an empty stack or result in stack type discrepancies are immediately rejected.

---

## Phase 3: Bounded Superoptimizer Search Engine

### 3.1 Objective
Synthesize alternative instruction sequences using bounded combinatorial search and stochastic mutation (inspired by STOKE) rather than static lookup tables.

### 3.2 Target Components
* `com.superopt.optimizer.CandidateGenerator`: Generates candidate sequences of length $L \in [1, L_{max}]$.
* `com.superopt.optimizer.SearchStrategy`: Forward breadth-first search and stochastic mutator.
* `com.superopt.optimizer.CandidateEvaluator`: Filters candidates by net stack effect and type compatibility.
* `com.superopt.optimizer.Superoptimizer`: Orchestrates search loops, timeouts, and metrics collection.

### 3.3 Search & Pruning Constraints
1. **Configurable Depth:** Maximum sequence length $L_{max}$ (default: 5 instructions).
2. **Search Budget:** Node count limit (e.g. 10,000 nodes) and wall-clock timeout (e.g. 5 seconds per block).
3. **Stack Balance Filter:** Candidate must have the identical net stack delta ($\Delta_{stack}$) and identical final local variable footprint as the target sequence.
4. **Deduplication:** Sequences with identical opcode and operand tuples are tracked in a hash set to eliminate redundant verification queries.

---

## Phase 4: Z3 SMT-Based Equivalence Verification

### 4.1 Objective
Formally prove whether a candidate instruction sequence is semantically identical to the original sequence using the Z3 SMT solver over 32-bit Bit-Vector logic (`QF_BV`).

### 4.2 Target Components
* `com.superopt.verification.Z3EquivalenceChecker`: Z3 solver lifecycle and query dispatcher.
* `com.superopt.verification.SemanticEncoder`: Translates original and candidate sequences into symbolic Z3 `BitVecExpr` ASTs.
* `com.superopt.verification.EquivalenceResult`: Enum result: `EQUIVALENT`, `NOT_EQUIVALENT`, `UNKNOWN`, `UNSUPPORTED`.
* `com.superopt.verification.Counterexample`: Concrete 32-bit input vector exhibiting divergent behavior when `NOT_EQUIVALENT`.

### 4.3 Theorem Proving Formula
For input symbolic bitvectors $\vec{x}$ (initial stack elements and live local variables):
$$\text{Output}_{orig}(\vec{x}) \neq \text{Output}_{cand}(\vec{x}) \lor \bigvee_{v} \left( \text{Local}_{orig}[v](\vec{x}) \neq \text{Local}_{cand}[v](\vec{x}) \right)$$
* **UNSAT:** No input assignment exists where the outputs differ $\implies$ **Formally Equivalent**.
* **SAT:** Z3 extracts a concrete model satisfying the difference $\implies$ **Counterexample Identified**.
* **UNKNOWN / TIMEOUT:** Solver cannot decide within limits $\implies$ **Safely Rejected**.

---

## Phase 5: Bytecode Transformation & Classfile Reassembly

### 5.1 Objective
Safely splice verified candidate instruction sequences into the target ASM method node and recompute valid JVM StackMapTable frames.

### 5.2 Target Components
* `com.superopt.transformation.BytecodeTransformer`: Replaces instruction sub-lists in `MethodNode.instructions`.
* `com.superopt.transformation.FrameRecomputer`: Recomputes stack map frames and maximum stack/local values.
* Verification harness wrapping ASM's `CheckClassAdapter.verify()`.

### 5.3 Safety Invariants
1. **Preservation of Non-Transformed Code:** Method access flags, annotations, try-catch handlers, and surrounding bytecode remain untouched.
2. **Stack Frame Validity:** Recomputed `StackMapTable` attributes must satisfy Java 7+ strict verification (`-Xverify:all`).
3. **Safe Class Loading:** Frame computation handles multi-class hierarchies without triggering runtime `ClassNotFoundException`.

---

## Phase 6: Multiple Equivalent Variants & Diversity Objectives

### 6.1 Objective
Synthesize and collect a pool of distinct verified instruction sequences and select candidates based on optimization or obfuscation objectives.

### 6.2 Target Components
* `com.superopt.transformation.VariantGenerator`: Aggregates all proven equivalent variants for an eligible basic block.
* `com.superopt.transformation.VariantSelector`: Chooses the active variant based on a chosen strategy.
* `com.superopt.optimizer.OptimizationObjective`: Selection strategies:
  * `MIN_SIZE`: Selects sequence with fewest bytecode instructions / smallest byte length.
  * `STRUCTURAL_DIVERSITY`: Maximizes opcode divergence / edit distance from the original sequence.
  * `BALANCED`: Multi-criteria score balancing size, estimated instruction latency, and diversity.

### 6.3 Determinism
* Configurable build seed (`--seed <long>`) guarantees identical variant selection across repeated builds with the same inputs.

---

## Phase 7: Build-Time CLI & JAR Archive Pipeline

### 7.1 Objective
Deliver a production-ready command-line tool capable of processing standalone `.class` files and complete `.jar` archives.

### 7.2 Target Components
* `com.superopt.Main`: Application entry point.
* `com.superopt.cli.CommandParser`: Subcommand dispatcher (`analyze`, `optimize`, `variants`, `benchmark`).
* `com.superopt.cli.CliOptions`: Option definitions and validation.
* JAR transformer preserving manifest files (`META-INF/MANIFEST.MF`), XML configurations, and resources while stripping invalid signatures.
* JSON report generator emitting structured machine-readable statistics (`transformation-report.json`).

---

## Phase 8: Empirical Validation, Differential Testing & Benchmarking

### 8.1 Objective
Cross-validate formal verification results through empirical differential execution and measure performance impact.

### 8.2 Target Components
* `com.superopt.testing.DifferentialTester`: Dynamic invocation of original vs transformed methods in isolated `URLClassLoader` instances.
* `com.superopt.testing.RandomInputGenerator`: Generates 10,000 pseudo-random 32-bit integer arguments per eligible method.
* `com.superopt.testing.BoundaryValueTests`: Evaluates critical edge values:
  * `Integer.MIN_VALUE` (`-2147483648`)
  * `Integer.MAX_VALUE` (`2147483647`)
  * `-1`, `0`, `1`
  * Powers of two: $2^k, 2^k - 1, 2^k + 1$
* `com.superopt.benchmarking.BenchmarkRunner`: Measures transformation throughput, verification latency, and runtime overhead.
* `com.superopt.benchmarking.ReportGenerator`: Formats benchmark summaries in JSON and Markdown.

---

## Acceptance Matrix

| Phase | Core Deliverable | Acceptance Gate |
| :---: | :--- | :--- |
| **1** | CFG & BasicBlock Analyzer | Correctly identifies blocks on branch/loop/exception test classes. |
| **2** | 32-bit Integer Semantics | Matches JVM two's complement and shift mask behavior across all boundaries. |
| **3** | Candidate Generator | Discovers alternative sequences without hardcoded rewrite tables. |
| **4** | Z3 SMT Verifier | UNSAT on known equivalents; SAT with counterexamples on invalid candidates. |
| **5** | ASM Bytecode Rewriter | Passes `CheckClassAdapter` and executes cleanly in JVM. |
| **6** | Variant Selector | Generates distinct sequences; reproducible outputs under fixed seed. |
| **7** | CLI & JAR Pipeline | Transforms standalone classes and multi-entry JAR files with JSON reports. |
| **8** | Differential Validation | 10,000 random runs + boundary cases achieve 100% equivalence match. |
