# System Integration Manual: Host-Controller Communications for the K-Means Cluster Engine

## 1. Architectural Foundation: Dynamic LUT Abstraction
In high-performance hardware acceleration, the primary bottleneck often resides in the rigidity of data structures. Traditional image processing systems frequently rely on static RGB coordinate systems, where hardware must compare every incoming pixel against fixed color values. To overcome this, the K-Means Cluster Engine utilizes dynamic data abstraction. By decoupling physical color channels from their intensity values, the system achieves software-defined flexibility within a deterministic hardware fabric, maintaining the sub-millisecond latency of a hardware pipeline.

The engine employs a "Max, Mid, Min" methodology. Rather than storing explicit red, green, or blue coordinates, the Look-Up Tables (LUTs) store generic intensity hierarchies. During the hardware's internal process, the engine identifies the instantaneous hierarchy of an incoming pixel’s color channels. Crucially, the internal signals `rgb_max`, `rgb_mid`, and `rgb_min` are defined with the synthesis `keep` attribute to ensure they are preserved for precise timing analysis and debugging within the FPGA matrix. By mapping these generic LUT boundaries to physical channels on the fly, a single compact LUT array can represent an expansive color spectrum.

The following table illustrates the internal mapping logic based on instantaneous pixel intensity:

| Scenario | Hierarchy (Intensity) | Physical Channel Assignment | Hardware Mapping |
| :--- | :--- | :--- | :--- |
| **R > G > B** | R(Max), G(Mid), B(Min) | R $\to$ Max, G $\to$ Mid, B $\to$ Min | LUT.max, LUT.mid, LUT.min |
| **G > B > R** | G(Max), B(Mid), R(Min) | G $\to$ Max, B $\to$ Mid, R $\to$ Min | LUT.max, LUT.mid, LUT.min |
| **B > R > G** | B(Max), R(Mid), G(Min) | B $\to$ Max, R $\to$ Mid, G $\to$ Min | LUT.max, LUT.mid, LUT.min |

---

## 2. Communication Port Interface Specification
The host-to-FPGA interface is the strategic gateway that enables the engine’s reconfigurability without requiring a full bitstream reprogramming. The primary integration ports are defined as follows:

*   **`k_ind_w` (Write Index):** A 10-bit natural signal for command and memory addressing.
*   **`k_ind_r` (Read Index):** A natural signal used to target specific internal LUT indices for verification.
*   **`centroid_lut_in`:** A 24-bit input signal deliverable as three 8-bit generic values ([23:16] Max, [15:8] Mid, [7:0] Min).
*   **`centroid_lut_out`:** A 31-bit output signal (standardized as a 32-bit word with a `x"00"` prefix) used for reading back LUT values.
*   **`centroid_lut_select`:** A selection signal for the 9 distinct classification strategies.

---

## 3. Protocol I: Runtime Centroid Boundary Overwrites (Indices 0–120)
Protocol I enables immediate adaptation to environmental lighting shifts by targeting the engine’s Active/Live memory registers (the `ll` memory variants). These registers are reconfigurable at runtime to provide instantaneous updates to the clustering logic.

**Address Logic Map:**
*   **Indices 0–30:** Modification of `k_rgb_lut_4ll`
*   **Indices 31–60:** Modification of `k_rgb_lut_2ll`
*   **Indices 61–90:** Modification of `k_rgb_lut_3ll`
*   **Indices 91–120:** Modification of `k_rgb_lut_6ll`

These overwrites are broadcast to 30 parallel `rgb_cluster_core` instances simultaneously. Because the update is synchronized with the 25-stage shift register pipeline, manual centroid updates do not cause pipeline stalls or dropped frames.

---

## 4. Protocol II: Predefined Palette Swapping (Indices 201–221)
Protocol II allows the host to trigger swaps between active base LUTs and a library of Constant memory palettes (`k_rgb_lut_41l` through `59l`), which are stored in read-only memory.

*   **Targeted Tracking:** Sending a write index of `220` triggers the `k_rgb_lut_58l` palette, specifically optimized for isolating "Required Brown Colors."
*   **Hardware-Triggered Edge Cases:**
    *   **`k_rgb_lut_0_l` (Grayscale):** Invoked automatically when $|R-G| \le 6$ and $|G-B| \le 6$.
    *   **`k_rgb_lut_1_l` (High-Contrast):** Invoked automatically when pixel spread (Max - Min) $\ge 100$.

---

## 5. Systemic Steering: Classification Strategy Management
The `centroid_lut_select` port provides macro-level systemic control by altering the comparator logic flow within the cluster cores. Engineers can pivot between 9 distinct strategies, enabling the hardware to transition from multi-tiered intensity segmentation to dedicated grayscale tracking or high-contrast spread evaluation.

### Integration Workflow Summary
1.  **Pre-flight:** Verify bitstream integrity and port mapping.
2.  **Initialization:** Define the primary classification strategy via `centroid_lut_select`.
3.  **Adaptation:** Use **Protocol I** to overwrite `ll` memory registers for lighting compensation.
4.  **Profiling:** Use **Protocol II** to trigger constant-memory palettes for specialized tracking.
5.  **Audit:** Utilize `k_ind_r` and `centroid_lut_out` to verify internal register states.

> **Note:** The K-Means Cluster Engine maintains a continuous pixel stream across its 25-stage pipeline, balancing deterministic hardware speed with the adaptability required for sophisticated computer vision.