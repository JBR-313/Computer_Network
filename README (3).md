# IoT Cryptographic Energy Analysis — Visualization & Data Analysis

> **Individual Portfolio** | Computer Networks 2026 | Group 6  
> 🔗 [Group Repository](https://github.com/ryprelude/IoT-lowpower-efficiency-project) · [Demo Video](https://youtu.be/BpkBcQrJ3DM)

---

## My Role: Data Visualization & Insight Extraction

In this project, I was responsible for **merging experimental data, designing the visualization pipeline, and drawing analytical insights from the results**. While other members focused on porting cryptographic algorithms and running simulations, my job was to turn raw measurement numbers into graphs that could tell a clear story — and to ask the right questions about what those graphs actually meant.

This page documents what I built, what I learned, and what I found surprising along the way.

---

## 1. Why This Problem Matters

IoT devices are everywhere — smart sensors, healthcare monitors, wearable devices. Most of them run on small batteries for years at a time. Security is non-negotiable (these devices are prime targets for interception and unauthorized access), but the conventional go-to algorithm, **AES-GCM**, was designed for powerful servers, not 16-bit microcontrollers.

The question our group set out to answer:

> *In a real IEEE 802.15.4 + 6LoWPAN IoT environment, does the NIST-standardized lightweight algorithm ASCON actually outperform AES-GCM in energy consumption — and at what payload size does the advantage kick in?*

---

## 2. What the Data Looked Like Before Visualization

After Members 2 and 3 completed their measurements in the Cooja simulator (Sky mote, MSP430), I received two sets of raw values:

| Algorithm | Data Size | CPU Ticks | Energy (mJ) |
|-----------|-----------|-----------|-------------|
| ASCON     | 16 bytes  | 3.271     | 0.00431     |
| ASCON     | 64 bytes  | 4.451     | 0.00587     |
| AES-GCM   | 16 bytes  | 15.22     | 0.00250     |
| AES-GCM   | 64 bytes  | 59.14     | 0.00974     |

At first glance, the numbers are telling two different stories depending on which row you look at. My job was to make that contrast visible — and to figure out *why* it exists.

---

## 3. Visualization Design Decisions

I produced three graphs. Each one was a deliberate choice, not just an aesthetic preference.

### 3-1. Bar Chart — `energy_bar.png`

**Purpose:** Show the absolute energy values side by side at each payload size.

The bar chart makes the reversal immediately obvious: AES-GCM's bar is shorter at 16 bytes, then dramatically taller at 64 bytes. This is the most intuitive entry point for someone seeing the results for the first time.

### 3-2. Line Chart — `comparison.png`

**Purpose:** Show the *trajectory* of energy growth as payload size increases.

This is the most important graph. The line chart revealed what the bar chart couldn't: the **slopes are fundamentally different**. AES-GCM's line climbs steeply; ASCON's line is nearly flat. This is not a coincidence — it reflects the structural difference between block cipher processing (AES-GCM) and streaming sponge construction (ASCON).

- **AES-GCM**: Each additional 16-byte block requires a full new encryption round + GHASH authentication update. CPU ticks scale almost linearly with block count: 15.22 → 59.14 (×3.9 for ×4 data).
- **ASCON**: One-time initialization cost, then data is absorbed incrementally. CPU ticks increase gradually: 3.271 → 4.451 (×1.4 for ×4 data).

The line chart is also where the **crossover point** becomes visible — somewhere around 32 bytes, ASCON becomes the more energy-efficient choice.

### 3-3. Average + Standard Deviation — `avg_std.png`

**Purpose:** Show measurement stability, not just central tendency.

Reporting only averages hides important information. If the 1,000-run average has high variance, the comparison might not be reliable. This graph was added after I realized that our group's Q&A follow-up document acknowledged this gap — we reported averages but didn't fully address spread.

The error bar graph makes it possible to see whether the gap between ASCON and AES-GCM is consistent or whether it's driven by a few extreme runs.

---

## 4. The 32-Byte Problem — What I Learned from Post-Presentation Feedback

The original line chart connected only two directly measured points: 16 bytes and 64 bytes. The 32-byte point appeared on the chart as a smooth interpolation between them.

During the Q&A session, Groups 10 and 12 asked:

> *"Was the 32-byte value actually measured, or just estimated? If estimated, how were nonlinear factors accounted for?"*

This was a fair challenge. Our group's follow-up answer acknowledged:

- The original 32-byte point was **not directly measured** — it was visually implied by the connecting line.
- After the presentation, we ran additional Cooja host-timing measurements to obtain calibrated 32-byte values.

**ASCON:**
```
16 bytes host elapsed: 12,249 μs (×10,000)
32 bytes host elapsed: 15,317 μs (×10,000)
64 bytes host elapsed: 25,111 μs (×10,000)
```
Linear estimate: ~0.00483 mJ → Calibrated value: **0.00468 mJ** (3.2% difference)

**AES-GCM:**
```
16 bytes host elapsed: 61,561 μs (×10,000)
32 bytes host elapsed: 102,058 μs (×10,000)
64 bytes host elapsed: 190,564 μs (×10,000)
```
Linear estimate: ~0.00491 mJ → Calibrated value: **0.00477 mJ** (2.9% difference)

The overall trend did not change — the crossover still occurs near 32 bytes. But this experience taught me something important about data visualization: **a smooth line implies continuous measurement, even when it doesn't exist**. Choosing a line chart without explicitly marking which points are measured versus interpolated is a form of misleading presentation, even if unintentional.

In the revised visualization, the 32-byte point is marked as a calibrated estimate, not a direct measurement.

> **Crossover summary:**
> - 16 bytes: AES-GCM saves 42% energy
> - 32 bytes: ASCON saves ~2% (near-equal, crossover zone)
> - 64 bytes: ASCON saves 40% energy

---

## 5. Why the Crossover Happens — The Structural Explanation

Understanding the graph requires understanding the algorithms.

**AES-GCM** is a block cipher. Data must be padded to 16-byte block boundaries, and each block requires a full encryption pass + GHASH update. In the MSP430 environment (no hardware AES accelerator), this is pure software overhead — every byte of extra data costs proportionally more CPU time.

**ASCON** uses a sponge construction. There is a fixed initialization cost (320-bit state setup, 12-round permutation), but after that, data is absorbed 64 bits at a time in a streaming fashion. The per-byte cost after initialization is much lower.

This is why the lines cross: at small sizes, ASCON's initialization overhead dominates. At larger sizes, AES-GCM's block-count scaling catches up and surpasses it.

There is also a **network-layer amplification effect** that the energy numbers alone don't capture:

```
AES-GCM padding overhead increases
        ↓
Exceeds 102-byte 6LoWPAN payload limit
        ↓
6LoWPAN fragmentation occurs
        ↓
IEEE 802.15.4 retransmission probability increases
        ↓
Additional radio energy consumed (Tx: ~17.4mA, Rx: ~18.8mA)
```

The measured energy values only reflect CPU computation. The real-world gap between the two algorithms is likely larger than our numbers show, because AES-GCM's larger overhead triggers fragmentation that ASCON avoids.

---

## 6. Trade-offs

| Metric | ASCON | AES-GCM |
|--------|-------|---------|
| Energy at 16 bytes | ❌ Higher (0.00431 mJ) | ✅ Lower (0.00250 mJ) |
| Energy at 64 bytes | ✅ Lower (0.00587 mJ) | ❌ Higher (0.00974 mJ) |
| Energy growth rate | ✅ Gradual (+36%) | ❌ Steep (+290%) |
| 6LoWPAN fragmentation risk | ✅ Lower | ❌ Higher |
| Hardware compatibility (16-bit MCU) | ❌ Porting failed on MSP430 | ✅ Works |
| NIST LWC Standard | ✅ Adopted (2023) | ❌ N/A |
| Cryptanalysis maturity | ❌ Newer (less field-tested) | ✅ 20+ years |

**Bottom line:** For real IoT sensor payloads (32 bytes and above), ASCON is the better fit on energy-constrained hardware — *provided* the hardware supports 64-bit operations. On legacy 16-bit MCUs like MSP430, actual porting fails.

---

## 7. Limitations I Observed Through the Visualization Work

Working with the data directly gave me a clear view of where our experiment's boundaries are:

1. **CPU-only measurement.** `RTIMER_NOW()` measures computation time, not radio energy. The Tx/Rx costs — which dominate real IoT energy budgets — are not included. The Energest framework would be needed for full-system measurement.

2. **The 32-byte point is calibrated, not directly energy-metered.** Host-timing ratios were used as a proxy. This is a reasonable approximation, but block boundary effects, padding overhead, and timer resolution can cause non-linear behavior that a simple ratio doesn't capture.

3. **Simulation-only results.** Cooja emulates the MSP430 instruction set well, but does not model RF interference, obstacles, or channel conditions. Physical validation on TelosB or Z1 hardware would be needed before making deployment decisions.

4. **ASCON was simplified on MSP430.** Because GCC 4.7.4 doesn't support 64-bit operations, the actual ASCON permutation could not run. The comparison is between full AES-GCM and a structurally similar but not identical ASCON substitute. Results are conservative estimates — real ASCON on a capable MCU would likely perform better.

---

## 8. Future Directions

- Re-run on ARM Cortex-M4 (64-bit capable) with Energest for full-system energy including radio
- Measure full DTLS handshake energy, not just record-layer cipher cost
- Test payloads beyond 64 bytes to confirm the trend holds past the 6LoWPAN fragmentation boundary
- Add box plots and confidence intervals to the visualization to properly characterize measurement spread

---

## 9. How to Run the Visualization

```bash
# Merge CSV data
python visualization/merge.py

# Generate all three charts
python visualization/visualize.py
```

Output files: `results/energy_bar.png`, `results/comparison.png`, `results/avg_std.png`

### Generated Charts

![Energy Bar Chart](https://raw.githubusercontent.com/ryprelude/IoT-lowpower-efficiency-project/main/results/energy_bar.png)
![Energy Line Chart](https://raw.githubusercontent.com/ryprelude/IoT-lowpower-efficiency-project/main/results/comparison.png)
![Average with Std Dev](https://raw.githubusercontent.com/ryprelude/IoT-lowpower-efficiency-project/main/results/avg_std.png)

---

## 10. References

- [1] Group 6 Repository: https://github.com/ryprelude/IoT-lowpower-efficiency-project
- [2] NIST, "Lightweight Cryptography (LWC) Project," 2023. https://csrc.nist.gov/projects/lightweight-cryptography
- [3] V. A. Thakor et al., "Lightweight Cryptography Algorithms for Resource-Constrained IoT Devices," *IEEE Access*, 2021.
- [4] M. G. Spina et al., "An IoE-Powered Framework for Adaptive Energy-Security Trade-Off in IoT," *IEEE Network*, 2026.
- [5] Contiki-NG Documentation. https://docs.contiki-ng.org
