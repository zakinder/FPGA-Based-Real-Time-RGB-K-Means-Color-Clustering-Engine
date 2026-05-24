# Technical Design Specification: High-Performance FPGA RGB K-Means Clustering Engine

## 1. Executive Design Overview and Strategic Motivation
In industrial machine vision and autonomous robotics, pixel-level classification is a primary computational bottleneck. A 1080p60 stream generates ~124.4 million pixels per second, requiring billions of color-space distance evaluations to categorize each pixel against a codebook of 30 centroids. Sequential architectures, including high-end CPUs, fail to meet the deterministic, sub-millisecond latency requirements of streaming environments due to instruction jitter and bus contention.

This specification details a hardware-accelerated K-Means engine optimized for **1.0 Cycles Per Pixel (CPP)** efficiency. By embedding classification logic directly into the AXI4-Stream data path, we achieve line-rate processing with zero frame-buffer overhead.

### Design Goals vs. Strategic Impact
| Design Goal | Strategic Impact |
| :--- | :--- |
| **Deterministic Latency** | Fixed 25–35 cycle processing delay ensures synchronization for real-time control loops. |
| **Massive Parallelism** | 30 concurrent clustering engines leverage DSP resources to exceed 1080p60 throughput. |
| **1.0 CPP Efficiency** | Processes one pixel/clock (up to 200 MHz), eliminating MIPI/VDMA back-pressure. |
| **BRAM Efficiency** | Max/Mid/Min abstraction provides a 6:1 BRAM reduction for resource-constrained Edge AI. |

---

## 2. Hardware-Optimized K-Means Theory and Distance Metrics
The algorithm is refactored from an iterative software process into a streaming assignment engine. Training and centroid updates occur asynchronously at frame boundaries, while the FPGA executes continuous assignment at line rate.

### Distance Metric: Squared Euclidean Distance ($D^2$)
To balance accuracy and hardware constraints, the engine calculates $D^2$ as follows:
$$D^2 = (R - C_r)^2 + (G - C_g)^2 + (B - C_b)^2$$

*   **Monotonic Optimization:** Because the square root is a monotonic function, $D^2$ comparisons yield the same nearest-neighbor index as standard Euclidean distance, eliminating the non-deterministic, high-latency square-root datapath.
*   **DSP48 Inference:** The $D^2$ implementation uses dedicated hardware multipliers and accumulators, preserving fabric logic and maximizing timing closure margins.

---

## 3. Pipelined Microarchitecture and Datapath Synchronicity
The engine utilizes a synchronous, multi-stage pipeline ensuring that once primed, the engine retires one clustered result per clock cycle.

### Pipeline Stage Analysis
| Stage(s) | Operation | Functional Requirement |
| :--- | :--- | :--- |
| **S0** | Input Capture | Ingest 10-bit RGB and AXI4-Stream sideband signals. |
| **S1–S2** | Quantization | Hierarchy detection; 10-bit to 8-bit downsampling. |
| **S3** | LUT Mapping | Active bank selection based on heuristics. |
| **S4–S28** | LUT Pipeline | 25-stage dedicated pipeline for BRAM-based boundary retrieval. |
| **S29–S31** | Distance Calc | Parallel $D^2$ computation in DSP48 slices. |
| **S32–S34** | Min-Reduction | 3-stage comparator tree identifies global minimum. |
| **S35** | Output Mapper | Index-to-RGB formatting (re-scaled to 10-bit). |

**Sideband Management:** A 31-stage shift register preserves metadata (TUSER/TLAST) in parallel with pixels to maintain spatial integrity and prevent frame corruption.

---

## 4. The Max/Mid/Min Abstraction Layer
Direct-mapped tables for the 16.7 million colors in the RGB cube are memory-prohibitive. We implement a hierarchy-based abstraction to normalize data into six permutations (e.g., $R > G > B$, $B > R > G$, etc.). By storing only generic intensity boundaries (Max, Mid, Min), a single BRAM entry serves multiple physical color-space permutations, achieving a **6:1 footprint reduction**.

---

## 5. Dynamic LUT Management and Frame-Safe Activation
To support adaptive vision (e.g., lighting shifts), the engine features **Runtime Palette Reconfiguration**. 
*   **Shadow-Register Mechanism:** Host software populates inactive memory banks via `k_lut_in`. 
*   **Atomic Updates:** The engine defers palette swaps until the Vertical Blanking Interval (VBI), signaled by the EOF/SOF transition, preventing mid-frame visual tearing.

---

## 6. System Integration and Host Control
Encapsulated within the Video Frame Processing (VFP) IP for platforms like the AMD Kria KV260:
1.  **Host Control:** Python serializes RGB triplets.
2.  **Transport:** Binary payload transmitted via TCP to an lwIP server on the Processor System (PS).
3.  **Orchestration:** PS executes AXI4-Lite register writes to the PL (Programmable Logic) engine.
4.  **Acceleration:** PL applies the codebook to the MIPI CSI-2 RX stream.

> **Technical Note:** Developers must ensure `Xil_DCacheInvalidateRange` is called in PS software to prevent stale cache lines from leading to discrepancies between the host-commanded palette and the hardware output.

---

## 7. Performance Summary
*   **Throughput:** 1.0 Cycle Per Pixel.
*   **Latency:** 25–35 clock cycles.
*   **Target Frequency:** 200 MHz.
*   **Pipeline Path:** 10-bit Input $\to$ 8-bit Internal $\to$ 10-bit Output.
*   **Applications:** High-speed defect detection, object segmentation, and AI dimensionality reduction.