# Formal Feature Blueprint: FPGA LUT Management Module for Color Clustering

## 1. Executive Summary and Strategic Context
The LUT Management Module serves as the critical intelligence layer within the broader RGB K-Means Cluster Engine, acting as a sophisticated bridge between incoming raw pixel streams and high-speed parallel classification hardware. In real-time vision environments—where systems must process over 124 million pixels per second for standard 1080p60 feeds—this module eliminates the need for computationally expensive iterative software loops and floating-point math. 

By providing a hardware-mapped memory structure that manages centroid boundaries, the module enables the system to classify each pixel within a deterministic window.

### Primary Design Requirements
| Requirement | Hardware Implementation Goal |
| :--- | :--- |
| **Throughput** | Process one RGB pixel per clock cycle after initial pipeline fill (up to 200 MHz). |
| **Latency** | Maintain a deterministic 35-cycle full-pipeline delay for predictable synchronization. |
| **Memory Efficiency** | Utilize Max/Mid/Min abstraction to represent the 16.7M color space in compact LUTs. |
| **Adaptability** | Support zero-downtime runtime LUT overwrites and palette swapping via external host. |
| **Parallelism** | Drive 30 simultaneous distance engines for high-compute nearest-centroid selection. |
| **Implementation** | Portable VHDL RTL optimized for FPGA DSP inference. |

---

## 2. Memory Abstraction: The Max/Mid/Min Storage Model
Traditional hardware color classification often struggles with the storage requirements of the 24-bit RGB color space. To overcome this, the module utilizes a "Max/Mid/Min" storage abstraction.

### The 24-bit Entry Format
Each 24-bit LUT entry is formatted into three distinct 8-bit generic intensity fields:
* **Bits 23:16 (Max):** Highest generic centroid boundary.
* **Bits 15:8 (Mid):** Middle generic centroid boundary.
* **Bits 7:0 (Min):** Lowest generic centroid boundary.

### Dynamic Channel Mapping Logic
The hardware utilizes a dynamic channel mapper to bridge these generic values with physical pixel data:
1.  **Hierarchy Identification:** A comparator tree analyzes R, G, and B channels to determine their relative strengths.
2.  **Permutation Mapping:** The system identifies which of the six possible RGB permutations (e.g., R>G>B, B>R>G) the pixel fits into.
3.  **Assignment:** The generic LUT values (Max/Mid/Min) are assigned back to physical channels based on the observed hierarchy.

---

## 3. Multi-Palette Subsystem and Categorization
The module manages a multi-palette subsystem, acting as a "Mercy Filter" for downstream classification. Categories include:

* **Base Intensity Palettes:** (e.g., `k_rgb_lut_2_l` through `6_l`) Default segmentation for general-purpose brightness and tone grouping.
* **Operational Palettes:** (e.g., `k_rgb_lut_0_l, 1_l`) Specialized logic for grayscale detection or high-contrast handling.
* **Profiling Palettes:** (e.g., `k_rgb_lut_41l` through `59l`) Application-specific targets for precision tracking (e.g., industrial or agricultural materials).

---

## 4. Pipelined Data Flow and Synchronization Architecture
The module implements a 7-stage pipeline (total functional depth of 35 cycles) to ensure single-cycle pixel processing.

**Stages:**
1.  **Input Capture/Quantization:** 10-bit to 8-bit downsampling.
2.  **Feature Extraction:** Comparator tree identifies hierarchy.
3.  **Channel Ordering:** Pipeline sorting into Max/Mid/Min.
4.  **LUT Dispatching:** Retrieval of active values.
5.  **Parallel Assignment:** Broadcast to 30 Euclidean Distance (D^2) engines.
6.  **Min-Reduction Tree:** Multi-stage aggregation for nearest-cluster identification.
7.  **Output Arbitration:** Index matching and formatting.

---

## 5. Runtime Reconfiguration and External Control Interface
For zero-downtime adaptability, the module exposes control pins (`k_ind_w`, `centroid_lut_in`).

* **Direct Overwrite:** Indices 0–120 for field calibration.
* **Index-Controlled Swapping:** Indices 201–221 for palette replacement.
* **Shadow LUT Double-Buffering:** Updates are staged in inactive memory and switched atomically at frame boundaries to prevent visual tearing.

---

## 6. System Integrity, Validation, and Fallback Logic
* **Safety Protocols:** Includes automated index validation, readback support for verification, and pipeline flushing.
* **Deterministic Tie-Breaking:** Hardcoded selection of the lower cluster index for equidistant results.
* **Synchronous Fallback:** If the comparator tree fails, the system triggers an "All-Ones" state (white pixel output) to signal a classification fault, preventing the propagation of erroneous data.