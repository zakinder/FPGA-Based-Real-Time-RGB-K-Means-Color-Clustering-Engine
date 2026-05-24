# Feature Deck: FPGA Parallel Distance Engine (PDE) for Color Clustering

## 1. Executive Summary: The Geometry of Vision
The Parallel Distance Engine (PDE) is the primary computational core of the RGB K-Means Cluster Engine. While the LUT Management Module defines the *what* (centroid boundaries), the PDE solves the *how* (the spatial relationship between pixel and cluster). In real-time industrial vision, pixel-level categorization must happen at line rate; the PDE achieves this by replacing iterative, non-deterministic software algorithms with a massively parallel hardware structure.

### Core Value Proposition
*   **Geometric Determinism:** Performs over 3.7 billion distance evaluations per second with a fixed, cycle-accurate 35-clock latency.
*   **Resource Efficiency:** Uses a squared-distance metric to eliminate costly square-root hardware (CORDIC or divider logic).
*   **Scalable Throughput:** The "Tournament Reduction" architecture ensures that adding more centroids does not increase the processing time per pixel.

---

## 2. The Computational Challenge: Why Distance is "Hard"
In a standard 1080p60 stream, the system faces a "clock-cycle crunch." With pixels arriving every 5ns (at 200MHz), there is zero margin for instruction-based loops or conditional branching.

### PDE vs. Traditional Compute
| Feature | CPU/GPU (Iterative) | PDE (Streaming Hardware) |
| :--- | :--- | :--- |
| **Logic Model** | Sequential loops/SIMD | Parallel combinational logic |
| **Accuracy** | Floating point (high latency) | Fixed-point integer (deterministic) |
| **Throughput** | Bottlenecked by memory bus | 1 pixel per clock cycle |
| **Scaling** | Diminishing returns per core | Linear scaling via DSP slices |

To classify a single pixel against 30 centroids, an iterative processor would spend hundreds of cycles. The PDE completes this in a single, pipelined clock cycle after the initial pipeline fill.

---

## 3. Mathematical Optimization: The Squared Metric
To maintain a 200 MHz clock speed, the PDE ignores the standard Euclidean distance formula, $\sqrt{(R-C_r)^2 + (G-C_g)^2 + (B-C_b)^2}$, because the square-root function is non-linear and timing-prohibitive.

**The PDE "So What?":** The engine calculates $D^2 = \Delta R^2 + \Delta G^2 + \Delta B^2$. Because the square root is a monotonic function, comparing $D^2$ values yields the same winning centroid as comparing $D$ values. By calculating squared distances in dedicated DSP48/DSP58 slices, the PDE achieves high-precision math with zero impact on the overall pixel-processing frequency.

---

## 4. The 3-Stage "Tournament Reduction" Tree
The PDE manages 30 concurrent Distance Evaluation Units (DEUs). To find the nearest centroid without a massive, slow multiplexer, the engine employs a hierarchical "Tournament Tree."

*   **Stage 1:** Pairwise comparison of 30 distances, resulting in 15 winners.
*   **Stage 2:** Comparison of 15 winners, resulting in 7 winners.
*   **Stage 3:** Final aggregation to identify the global minimum (nearest cluster index).

This logarithmic reduction (log2(30)) allows the engine to isolate the global minimum in only three pipeline stages, ensuring the output is perfectly synchronized with the video stream.

---

## 5. Implementation Strategy: DSP-Centric Architecture
The PDE is engineered for modern AMD/Xilinx Zynq and Kria fabric. It utilizes the pre-adder/multiplier capabilities of the DSP48 architecture:
1.  **Subtraction:** Pixel channels $(P_n)$ are subtracted from Centroid boundaries $(C_n)$ using the internal pre-adder.
2.  **Squaring:** The result is squared within the multiplier block.
3.  **Accumulation:** The three squared channels are summed in the 48-bit accumulator register.

This integration ensures that the PDE consumes almost zero general-purpose logic (LUTs), leaving maximum fabric available for networking, filtering, and system-on-chip control logic.

---

## 6. System Integration & Reliability
Reliability is as important as speed in autonomous navigation and robotics.

### Safety & Integrity Features
*   **Tie-Breaking:** Hardcoded logic forces a selection of the lower cluster index if two centroids are equidistant, preventing jitter in classification outcomes.
*   **"All-Ones" Fallback:** If the engine detects a logic fault (e.g., all distance metrics remain at max value), it asserts an error flag and drives the output to a "White" pixel state, notifying downstream AI of a classification failure.
*   **Sideband Integrity:** While the PDE computes distances, a 31-stage shift register holds the frame coordinates (SOF/EOF/EOL). This ensures the "Cluster ID" output is physically linked to the correct pixel location, regardless of the pipeline depth.

---

## 7. Performance Datasheet
| Metric | Specification |
| :--- | :--- |
| **Throughput** | 1 Pixel per clock (Deterministic) |
| **Latency** | 35 clock cycles (Full-Pipe) |
| **Parallelism** | 30 Concurrent DSP-based MACs |
| **Math Precision** | 18-bit internal sum (8-bit inputs) |
| **Interface** | AXI4-Stream |
| **Resource Inference** | DSP48E2/DSP58 |

---

## 8. Conclusion: Strategic Impact
The Parallel Distance Engine turns high-resolution video into "Semantic Intelligence." By moving from raw RGB values to discrete "Cluster IDs" in hardware, we enable:
*   **Bandwidth Reduction:** Converting 24-bit color data into a 5-bit Cluster ID, slashing downstream processing requirements.
*   **Edge AI Acceleration:** Simplified data inputs allow lightweight neural networks to perform inference at 60fps on low-power platforms.
*   **Deterministic Navigation:** Fixed, ultra-low latency reaction times for high-speed robotics.

The PDE is the definitive solution for edge devices requiring industrial-grade vision without the overhead of traditional software processing.