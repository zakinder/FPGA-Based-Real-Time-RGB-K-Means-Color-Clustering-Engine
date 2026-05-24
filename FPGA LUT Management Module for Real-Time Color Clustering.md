# Formal Blueprint: FPGA LUT Management Module for Real-Time Color Clustering

## 1. Executive Architectural Overview
The Lookup Table (LUT) Management Module is a core functional unit within the "RGB K-Means Cluster Engine," serving as a register-balanced preprocessing stage for embedded vision AI. It executes hardware-level image quantization and color segmentation, transforming raw high-bandwidth video into a quantized stream of cluster indices.

* **Strategic Offloading:** By processing data in-line, the module reduces the computational burden on downstream AI processors, which are better utilized for high-level decision-making.
* **Deterministic Throughput:** The module maintains a constant throughput of one pixel per clock cycle with fixed latency, ensuring the predictable performance required for real-time robotic perception.

---

## 2. Module Entity and Port Specification
The `color_k5_clustering` entity utilizes a record-based interface for efficient top-level routing and modular consistency.

| Signal Name | Direction | Type | Functional Description |
| :--- | :--- | :--- | :--- |
| `clk` | In | `std_logic` | Global system clock. |
| `rst_l` | In | `std_logic` | Active-low asynchronous reset. |
| `iRgb` | In | Channel (Rec) | Input RGB stream with sync metadata. |
| `k_lut_selected` | In | `natural` | 31-bit selector for active centroid dataset. |
| `k_lut_in` | In | `std_logic_vector` | Data bus for writing new 24-bit centroids. |
| `k_lut_out` | Out | `std_logic_vector` | Readback bus for LUT verification. |
| `k_ind_w` / `r` | In | `natural` | Multiplexed write/read indices for updates. |
| `oRgb` | Out | Channel (Rec) | Pipelined output with quantized cluster data. |

* **Scalability:** The `i_data_width` generic (default 8 bits) allows for adaptation across various FPGA families, with internal bit-slicing (9 downto 2) to optimize timing.

---

## 3. LUT Data Structures and Programmable Centroid Storage
The module organizes centroids using the `rgb_k_range(0 to 30)` structure, allowing for 31 parallel distance computations per clock cycle.

* **Predefined Profiles:**
    * `k_rgb_lut_0_l`: Linear greyscale profile for neutral-tone processing.
    * `k_rgb_lut_41l`: Specialized RGB mappings for soil and target detection.
* **Flexibility:** While fixed constants provide reliability, internal signal arrays act as buffers for dynamic runtime updates via the control interface.

---

## 4. Dynamic LUT Selection and Switching Logic
The module manages LUT updates through the `k_ind_w` signal, enabling real-time environmental adaptation without bitstream reconfiguration.

* **Greyscale Tolerance:** Checks `abs(rgb_sync2.red - rgb_sync2.green) <= 6` (and blue equivalent). If satisfied, the module defaults to greyscale, preventing neutral-tone misclassification.
* **Intensity Evaluation:** Analyzes the delta between max and min color components (`rgb_max1 - rgb_min1 >= 100`) to confirm color dominance before initiating intensive distance calculations.

---

## 5. Metadata Preservation and Pipeline Timing Alignment
To prevent coordinate drift in deep pipelines, synchronization signals (`valid`, `eol`, `sof`) are managed through a precision-timed shift register system.

* **Distance Calculation:** Employs the **Manhattan RGB Distance metric** for hardware efficiency.
* **Comparator Tree:** Uses a staged hierarchical reduction (minimizing 5, then 3, then 2 candidates) to determine the winning cluster index.
* **Deterministic Latency:** The output stage is strictly tapped at index 25 (`oRgb.valid <= rgbSyncValid(25)`). This spreading of logic across 25 cycles avoids routing congestion and high fan-out risks.

---

## 6. Implementation Use Cases in Embedded Vision AI
This module acts as a performance multiplier by reducing data complexity at the edge.

* **Robotics & Navigation:** Real-time segmentation for obstacle avoidance and pathfinding.
* **Environmental Detection:** Rapid isolation of specific terrain features in agricultural/exploratory robotics.
* **Edge Devices:** High-bandwidth vision processing on platforms with limited power budgets (e.g., drones and mobile platforms).