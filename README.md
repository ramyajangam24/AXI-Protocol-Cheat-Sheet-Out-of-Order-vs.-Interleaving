# AXI Protocol Cheat Sheet: Out-of-Order vs. Interleaving

A quick-reference guide to the architectural differences between **Out-of-Order (OoO) transactions** and **Data Interleaving** in the ARM AMBA AXI protocol. Useful for RTL design, verification engineering, and ASIC/FPGA interview preparation.

## Table of Contents

- [Core Concepts](#-core-concepts-at-a-glance)
- [Visualizing the Difference](#-visualizing-the-difference)
- [Summary Comparison](#-summary-comparison)
- [AXI3 vs. AXI4: Write Interleaving](#-axi3-vs-axi4-write-interleaving)
- [Why Read Interleaving Is Still Allowed](#-why-read-interleaving-is-still-allowed)
- [Interview Quick Answers](#-interview-quick-answers)

---

## 💡 Core Concepts At a Glance

Both features prevent a slow peripheral from blocking a fast one (Head-of-Line blocking), but they operate at different granularities:

| Feature | Granularity | Requirement |
| :--- | :--- | :--- |
| **Out-of-Order** | Entire transactions (full burst completions) | Transactions must use **different IDs** |
| **Interleaving** | Individual data beats within active bursts | Transactions must use **different IDs** |

> **Note:** Transactions with the **same ID** must always complete in the order they were issued.

---

## 🛠️ Visualizing the Difference

Consider a master issuing **two read bursts** (T1 and T2), each with **4 data beats**.

### 1. In-Order Execution (Baseline)

Everything completes in request order. T1 must fully finish before T2 begins.

```text
Time  ------------->
Data:  [T1 B0] [T1 B1] [T1 B2] [T1 B3] | [T2 B0] [T2 B1] [T2 B2] [T2 B3]
```

### 2. Out-of-Order (OoO) Execution

Transactions complete in a different sequence than requested. Here **T2 finishes entirely before T1** (e.g., T2 hit a fast cache while T1 suffered a slow DRAM page miss).

> ⚠️ **Crucial Rule:** Beats of T2 are *not* mixed with T1. T2 simply jumps ahead in line.

```text
Time  ------------->
Data:  [T2 B0] [T2 B1] [T2 B2] [T2 B3] | [T1 B0] [T1 B1] [T1 B2] [T1 B3]
```

### 3. Interleaved Execution

Individual beats of T1 and T2 are **mixed together** on the same data bus.

> ⚠️ **Crucial Rule:** This requires different IDs. The receiver uses the ID attached to each beat to route data to the correct buffer.

```text
Time  ------------->
Data:  [T1 B0] [T2 B0] [T2 B1] [T1 B1] | [T1 B2] [T2 B2] [T2 B3] [T1 B3]
```

---

## 📊 Summary Comparison

| Execution Type | Data Bus Pattern | Why a Designer Uses It |
| :--- | :--- | :--- |
| **In-Order** | `T1 T1 T1 T1` → `T2 T2 T2 T2` | Simplest logic; default behavior. |
| **Out-of-Order** | `T2 T2 T2 T2` → `T1 T1 T1 T1` | Prevents a slow peripheral from blocking a fast one. |
| **Interleaved** | `T1 T2 T2 T1 T1 ...` | Maximizes bus utilization when multiple sources share one data pipeline. |

---

## ⚠️ AXI3 vs. AXI4: Write Interleaving

Interviewers and system architects often probe the protocol differences around write interleaving.

### AXI3: Why `WID` Existed

In AXI3, the write address channel (`AW`) and write data channel (`W`) operate independently. A master could send the address for Transaction A, then B, and then interleave the write data beats. Because beats were mixed, AXI3 required a **`WID` (Write ID)** signal alongside every data beat so the slave could match each beat to its address.

### The Problem with Write Interleaving

Although efficient on paper, it created significant hardware challenges:

- **Complex masters:** Required reordering buffers and dependency-checking logic.
- **Routing congestion:** `WID` added extra wires to the data bus, increasing physical routing congestion in large SoCs.
- **Low practical use:** Most masters (CPUs, DMA controllers) generate write data sequentially anyway.

### AXI4: `WID` Removed

To simplify hardware and save silicon area, ARM **removed the `WID` signal** in AXI4.

Without a `WID` tag, the slave cannot tell interleaved write data apart. AXI4 therefore enforces a strict rule:

> **Write data must be sent in the same order as the write addresses were issued.**
> If Address A precedes Address B, all data beats for A must complete before the first beat of B starts.

---

## ✅ Why Read Interleaving Is Still Allowed

Unlike writes, **read interleaving is still supported in AXI4**.

The slave (for example, a multi-channel DDR memory controller) controls the read data channel (`R`). A smart controller can return data from a fast, open page for Read B while Read A waits on a slow page miss. Since the read data channel already carries a built-in **`RID`** tag to route data back to the correct master, interleaving reads improves memory bandwidth with no extra wire overhead.

---

## 🎯 Interview Quick Answers

**Q: What's the difference between out-of-order and interleaving?**
Out-of-order reorders whole transactions; interleaving mixes individual beats of different transactions on the bus.

**Q: What does each require?**
Both need different transaction IDs. Same-ID transactions stay in order.

**Q: Why was `WID` removed in AXI4?**
To cut routing congestion and master complexity, since write interleaving was rarely used.

**Q: Is read interleaving supported in AXI4?**
Yes. `RID` is already present on every read beat, so it costs nothing extra.

---

## 📚 References

- ARM AMBA AXI and ACE Protocol Specification (IHI 0022)

## 🤝 Contributing

Corrections and additions are welcome. Open an issue or submit a pull request.
