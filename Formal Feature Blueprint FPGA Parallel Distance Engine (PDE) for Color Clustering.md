# Formal Feature Blueprint: FPGA Parallel Distance Engine (PDE) for Color Clustering

## 1. Architectural Mission and Strategic Value
The Parallel Distance Engine (PDE) serves as the primary computational workhorse of the RGB K-Means Cluster Engine. Following the LUT Management Module's classification of centroid boundaries, the PDE performs the high-speed geometric analysis required to map pixels to the "Nearest Centroid." 

Strategically, the PDE replaces iterative software-based distance calculations with a deterministic, massively parallel hardware structure. By implementing the Squared Euclidean Distance (D²) formula in silicon, the module eliminates the requirement for power-intensive square root operations, enabling real-time classification at 1080p60 rates. The engine is architected to prioritize throughput, maintaining a fixed-latency pipeline that feeds processed classification indices to downstream AI decision-making units.

## 2. Mathematical Foundation: The Squared Euclidean Distance (D²)
To maintain the required 200 MHz throughput, the PDE utilizes a squared distance metric. This avoids the non-deterministic latency of square root hardware (CORDIC or iterative dividers).

For a pixel $P(R, G, B)$ and a centroid $C(R_c, G_c, B_c)$, the engine calculates:
$$D^2 = (R - R_c)^2 + (G - G_c)^2 + (B - B_c)^2$$

### Implementation Details:
* **Precision:** Inputs are 8-bit, but intermediate subtractions are performed using 9-bit signed arithmetic to handle negative differences.
* **Summation:** Squared values are mapped to 32-bit `mac_syn` registers to prevent overflow during the accumulation of $(dR^2 + dG^2 + dB^2)$.
* **Resource Optimization:** DSP48E slices are utilized for single-cycle multiplication, balancing area usage against the need for massive parallel execution.

## 3. Massively Parallel Broadcast Architecture
The PDE utilizes a "Broadcast-and-Compare" architecture to handle 30 simultaneous centroid comparisons per clock cycle.

* **Broadcasting:** A single incoming pixel stream is buffered and replicated across 30 parallel `rgb_cluster_core` instances.
* **Parallel MAC Arrays:** Each instance contains a dedicated distance engine that calculates the $D^2$ for its specific assigned centroid.
* **The Reduction Tournament:** The resulting 30 distance values are passed into a three-stage pipelined Comparator Tree. This structure functions as a tournament bracket, where pairwise comparisons identify the local minimum at each node until a single global minimum ($D^2_{min}$) is identified.



## 4. Pipeline Integration and Timing Closure
The PDE is integrated into the system’s 35-cycle global latency model. The pipeline is staged to ensure optimal routing and timing closure on high-density FPGAs like the Zynq UltraScale+ series.

| Stage | Operation |
| :--- | :--- |
| **Stage 1** | Pixel broadcast and register synchronization. |
| **Stage 2-3** | Parallel $D^2$ calculation (Sub-Square-Sum). |
| **Stage 4-6** | Multi-stage Comparator Tree (The Tournament). |
| **Stage 7** | Min-Index identification and output formatting. |

* **Floorplanning Note:** Due to the high fan-out of the pixel broadcast bus and the density of 30 parallel cores, the architecture requires localized floorplanning (Pblocks) to prevent routing congestion and timing violations at 200 MHz.

## 5. Operational Integrity and Safety Features
To ensure reliable operation in autonomous environments, the PDE includes integrated integrity checks:

* **Tie-Breaking Logic:** In instances where two centroids are equidistant, the PDE is hardcoded to select the lower index, ensuring consistent, repeatable output across different hardware runs.
* **"All-Ones" Fallback:** If the PDE detects an invalid or uninitialized centroid (e.g., all distance metrics remain at max value), it asserts a fault signal, forcing the output to a predetermined "White" state to alert downstream AI of a classification error.
* **Sideband Alignment:** The engine utilizes a dedicated 31-stage shift register to track `sof`, `eof`, and `valid` signals, ensuring that the spatial geometry of the frame is maintained even if specific pixel clusters are discarded.

## 6. Strategic Impact on Embedded Vision AI
The PDE acts as a "Feature Condenser." By reducing a continuous 16.7 million-color input space into 30 discrete semantic indices, the engine reduces the computational load on downstream neural networks by orders of magnitude.

### Critical Advantages for Edge AI:
1.  **Deterministic Latency:** Fixed 35-cycle delay ensures predictable reaction times for high-speed navigation loops.
2.  **Reduced AI Inference Overhead:** The classifier provides "Symbolic" RGB data, allowing lightweight networks to focus on structure and motion rather than color gradient analysis.
3.  **Adaptive Throughput:** The broadcast architecture is scalable; the number of parallel cores can be adjusted based on the specific hardware footprint of the target FPGA (e.g., 15 cores for low-power, 60+ for high-performance precision).

## 7. Architect’s Note on Implementation
The PDE’s reliance on DSP inference is fundamental to its performance. By forcing the mapping of $dR^2 + dG^2 + dB^2$ into DSP slices rather than logic gates (LUTs), the design ensures that timing constraints are met while leaving FPGA fabric free for image preprocessing and control logic. When porting this architecture, developers should prioritize the synchronization of the `mac_syn` pipeline to prevent skew between the 30 parallel distance engines.