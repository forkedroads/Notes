# Summary: Why Stacked 3D DRAM is the Right Future for AI Inference Accelerators

**Source:**  *Chip Log* — ["Why stacked 3D-DRAM is the right future for AI inference accelerators"](https://www.chiplog.io/p/why-stacked-3d-dram-is-the-right) (Sept 22, 2026)

## Overview
The article argues that stacking DRAM directly on compute dies (3D DRAM) is becoming the dominant architectural direction for AI inference accelerators — citing recent moves from Qualcomm (HBC), Cerebras (CS-6), Samsung (zHBM), Groq (rumored LP40), and especially **d-Matrix's Raptor** accelerator, which the piece analyzes in depth. The case rests on three pillars:

### 1. Performance — 3D DRAM enables the best architectures
- Keeping data physically close to compute (like SRAM) is the most energy-efficient design, per NVIDIA's Bill Dally and Jeff Dean (GTC 2026 fireside chat).
- SRAM-based accelerators (Groq, Cerebras, d-Matrix's first-gen Corsair) get this right but run out of capacity.
- 3D DRAM preserves SRAM-like dedicated bandwidth per tensor core while scaling capacity by an order of magnitude.
- Raptor example: 256 tensor engines per chiplet × 3 dedicated DRAM banks × 256-bit bus = ~196K data lanes per chiplet (~800K per package) — vastly more than Blackwell (8K lanes) or Rubin (16K lanes).

### 2. Power — Sub-0.1 pJ/bit
- In HBM, most energy loss comes from *transporting* data across the interposer (~2.5 pJ/bit), not from the DRAM array itself.
- Stacking DRAM directly on compute eliminates the interposer, dropping energy to ~0.35–0.4 pJ/bit.
- Hybrid bonding (micron-scale pitch vs. 36-micron bump pitch) shrinks this further to ~0.1 pJ/bit.
- At 100 TB/s, this is the difference between ~2000W (HBM) and ~296W (3D DRAM) — freeing up power budget for compute/networking.

### 3. Determinism — 3D DRAM access can behave like SRAM
DRAM is normally non-deterministic due to three factors; 3D DRAM mitigates or eliminates each:

| Source of Non-Determinism (HBM/LPDDR) | Why It Happens | How 3D DRAM Solves It |
|---|---|---|
| **Voltage/temperature drift requiring recalibration** | PHY must periodically retrain read/write "eye centering" to compensate for drift over long trace lengths | Direct bonding makes distances so short that no traditional PHY, training, or calibration is needed ("PHY-less" design) — timing is closed via static timing analysis |
| **Refresh overhead** | Refresh is temperature-controlled and takes memory offline periodically, directly costing performance | Banks in 3D DRAM are shallow, allowing a high refresh rate without a significant performance penalty |
| **Opportunistic transaction reordering** | HBM is a shared/unified memory pool — the controller reorders requests from multiple tensor engines to maximize page hits and bank rotation, making wait time dependent on contention from other engines | **Each tensor engine gets a dedicated path and dedicated pins to its own banks, shared with no other engine** — no contention to arbitrate, so the controller is simple and reordering isn't needed, making latency deterministic |

**Caveat:** Even 3D DRAM still needs smart data layout (e.g., for weights/KV cache) to maximize page hits — DRAM access patterns matter in a way SRAM's don't.

---
