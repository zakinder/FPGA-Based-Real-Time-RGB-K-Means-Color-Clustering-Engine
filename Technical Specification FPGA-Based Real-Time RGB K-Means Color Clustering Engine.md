# Technical Specification: FPGA-Based Real-Time RGB K-Means Color Clustering Engine

## 1. Executive Overview and Strategic Logic
The FPGA-based RGB K-Means engine is a high-performance preprocessing subsystem designed to solve the computational bottleneck of real-time semantic video analysis. By moving clustering from software to dedicated hardware, it enables deterministic classification of up to 1080p60 video streams.

**Core Design Pillars:**
* **Pipelining:** Synchronous staging allows the system to accept a new pixel every clock cycle.
* **Massive Parallelism:** Concurrent evaluation of 30 centroids eliminates iterative loops.
* **Memory Abstraction:** Uses "Max/Mid/Min" storage to compress color space, shifting resources from BRAM to logic (LUTs).

---

## 2. High-Throughput Streaming Datapath Architecture
The engine features a 35-stage synchronous datapath capable of processing one pixel per clock cycle without frame buffering.

| Phase | Cycles | Function |
| :--- | :--- | :--- |
| **Input Capture** | 0 | Registration of 24-bit iRgb; bit-slicing (9:2). |
| **Feature Extraction** | 1-2 | Comparator tree identifies Max/Mid/Min hierarchy. |
| **Dynamic Selection** | 3 | Selects active LUT bank via control signals. |
| **Data Alignment** | 4-28 | 25-stage shift register for metadata/pixel alignment. |
| **Parallel Compute** | 29-31 | Broadcasts pixel to 30 clustering modules. |
| **Min-Reduction Tree** | 32-34 | Hierarchical comparison to find global minimum. |
| **Output Mapping** | 35 | Final arbitration and output formatting. |

---

## 3. Memory Optimization: Max/Mid/Min Color Abstraction
To maximize efficiency, the engine stores generic intensity roles rather than absolute RGB values.

| Bit Field | Name | Role |
| :--- | :--- | :--- |
| 23:16 | Max | Highest generic intensity boundary |
| 15:8 | Mid | Middle generic intensity boundary |
| 7:0 | Min | Lowest generic intensity boundary |

* **Dynamic Mapping:** A channel mapper assigns these boundaries to physical R, G, and B channels based on the dominant intensity of the incoming pixel, significantly reducing BRAM usage.

---

## 4. Multi-Palette Hierarchy and Dynamic Selection
The system supports tiered palettes to ensure classification stability under varying conditions:
* **Base Intensity:** Standard brightness grouping (`k_rgb_lut_2_l`).
* **Operational:** High-contrast/grayscale stability (`k_rgb_lut_0_l`).
* **Profiling:** Application-specific targets, such as earth-tone segmentation (`k_rgb_lut_41l-59l`).

**Decision Logic:** Uses neutrality checks (diff ≤ 6) and intensity spread thresholds to automatically pivot between palettes.

---

## 5. Runtime Reconfiguration and Shadow Activation
The engine supports zero-downtime updates via AXI-Lite, ensuring no frame-tearing during palette swaps.

* **Shadow LUT Double-Buffering:** Updates are staged in an inactive buffer and switched atomically at the vertical blanking interval, guaranteeing visual continuity.

---

## 6. Parallel Compute Core: The K-Means Clustering Array
The core employs 30 parallel sub-modules using **Squared Euclidean Distance** ($D^2 = (R - C_R)^2 + (G - C_G)^2 + (B - C_B)^2$).
* **Reduction Tree:** A 3-stage network consolidates distances:
    1.  **Local:** 30 inputs → 6 local minimums.
    2.  **Intermediate:** 6 inputs → 2 intermediate candidates.
    3.  **Global:** 2 inputs → 1 winning Cluster ID.

---

## 7. System Validation and Engineering Specifications
Validated as a production-grade subsystem with bit-accurate compliance against Python/MATLAB models.

### Performance Metrics (at 200 MHz)
| Metric | Specification |
| :--- | :--- |
| **Peak Throughput** | 200 Million Pixels/Sec |
| **Fixed Latency** | 35 Cycles (175 ns) |
| **Tie Policy** | Lower cluster index selected |
| **Integration** | AXI4-Stream / AXI-Lite |

This architecture provides a robust, deterministic solution for industrial inspection, robotics, and agricultural imaging, where high-speed color segmentation is a prerequisite for reliable AI inference.