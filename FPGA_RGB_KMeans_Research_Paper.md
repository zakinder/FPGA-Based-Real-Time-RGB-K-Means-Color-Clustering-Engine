# High-Performance FPGA-Based RGB K-Means Clustering Engine for Real-Time Machine Vision

**Author:** Sakinder Ali  
**Document Type:** Research Paper Format  
**Domain:** FPGA Acceleration, Real-Time Video Processing, Machine Vision, Embedded Vision Systems  

---

## Abstract

Real-time pixel-level classification is a major computational challenge in industrial machine vision, autonomous robotics, and embedded artificial intelligence systems. A 1080p60 video stream produces approximately 124.4 million pixels per second, and when each pixel must be evaluated against a centroid codebook, the computational workload rapidly scales into billions of color-distance evaluations per second. Conventional CPU-based and GPU-based implementations may provide high aggregate throughput, but they often struggle to guarantee deterministic sub-millisecond latency in continuous streaming pipelines because of memory contention, scheduling variability, and frame-buffer dependencies.

This paper presents a high-performance FPGA-based RGB K-Means clustering engine designed for deterministic, line-rate video processing. The architecture transforms the iterative K-Means algorithm into a streaming assignment engine capable of processing one pixel per clock cycle. The design integrates thirty parallel squared-Euclidean distance engines, a pipelined comparator-reduction tree, dynamic LUT-based centroid management, and a Max/Mid/Min abstraction layer that reduces Block RAM utilization. The system supports runtime palette reconfiguration through a frame-safe shadow activation mechanism, making it suitable for adaptive machine vision applications under changing illumination or scene conditions. The proposed design targets AMD/Xilinx FPGA platforms such as the Kria KV260 and is optimized for AXI4-Stream video pipelines.

---

## Keywords

FPGA, RGB clustering, K-Means, machine vision, real-time video processing, AXI4-Stream, DSP48, BRAM optimization, embedded vision, Kria KV260, hardware acceleration, color segmentation.

---

## 1. Introduction

Real-time vision systems increasingly require deterministic, high-throughput processing at the pixel level. Applications such as industrial inspection, robotic perception, autonomous navigation, object segmentation, and AI preprocessing often require every incoming pixel to be classified or transformed without interrupting the video stream.

A standard 1080p60 stream contains approximately 124.4 million pixels per second. If each pixel is compared against thirty RGB cluster centroids, the system must perform a large number of color-space distance calculations every second. Although CPUs and GPUs are capable of high arithmetic throughput, their execution models are not always ideal for strict streaming latency. CPU instruction scheduling, cache behavior, memory contention, and GPU batch-processing overhead may introduce variability that is undesirable in deterministic control systems.

This research paper presents an FPGA-based RGB K-Means clustering engine optimized for 1.0 Cycles Per Pixel (CPP) operation. The engine is placed directly in the AXI4-Stream video datapath, enabling line-rate classification without full-frame buffering. Once the pipeline is filled, the architecture produces one clustered pixel output per clock cycle.

The main contributions of this work are:

1. A streaming FPGA K-Means assignment architecture for RGB video.
2. A thirty-engine parallel squared-Euclidean distance datapath.
3. A comparator-tree reduction network for global minimum centroid selection.
4. A Max/Mid/Min abstraction layer for LUT and BRAM reduction.
5. A frame-safe shadow-register mechanism for runtime palette updates.
6. A PS/PL integration model suitable for embedded FPGA platforms.

---

## 2. Background and Motivation

### 2.1 K-Means Clustering in Image Processing

K-Means clustering is commonly used in image processing to group pixels into color classes. In conventional software form, the algorithm alternates between two stages:

1. **Assignment:** Each data point is assigned to the nearest centroid.
2. **Centroid Update:** Each centroid is recomputed based on assigned samples.

For real-time video processing, the full iterative algorithm is difficult to execute inline because centroid updates require global frame statistics and repeated passes over image data. Therefore, this design separates centroid training or selection from the streaming assignment stage. Training or palette selection may occur asynchronously through software or frame-boundary control, while the FPGA continuously assigns each incoming pixel to the nearest active centroid.

### 2.2 Need for FPGA Acceleration

FPGAs are well suited for deterministic video processing because they support deep pipelining, spatial parallelism, and direct integration into streaming interfaces. Unlike sequential software processors, an FPGA can instantiate multiple arithmetic engines that operate concurrently. This allows all centroid distances to be evaluated in parallel for every pixel.

The proposed architecture is motivated by the need for:

- deterministic latency,
- one-pixel-per-clock throughput,
- low frame-buffer dependency,
- efficient use of DSP and BRAM resources,
- runtime reconfiguration of centroid palettes,
- direct compatibility with AXI4-Stream video systems.

---

## 3. System Overview

The proposed RGB K-Means clustering engine accepts incoming 10-bit RGB pixel data, internally quantizes the values to 8-bit precision, compares the pixel against a centroid table, identifies the nearest centroid, and outputs a clustered 10-bit RGB value.

At a high level, the system performs the following operation:

```text
Input Pixel: P(R, G, B)

For each centroid C[k]:
    D[k] = (R - C[k].R)^2 + (G - C[k].G)^2 + (B - C[k].B)^2

Winning_Index = argmin(D[0], D[1], ..., D[29])

Output Pixel = Centroid_RGB[Winning_Index]
```

The design targets 1.0 CPP throughput. This means that after pipeline initialization, the system accepts one pixel and produces one clustered result every clock cycle.

---

## 4. Distance Metric Selection

### 4.1 Manhattan Distance

The Manhattan distance metric is defined as:

```text
D_L1 = |R - C_r| + |G - C_g| + |B - C_b|
```

This metric is attractive for FPGA implementation because it does not require multipliers. However, it provides less accurate cluster geometry for applications requiring spherical or radial distance behavior in RGB color space.

### 4.2 Squared Euclidean Distance

The proposed design uses squared Euclidean distance:

```text
D^2 = (R - C_r)^2 + (G - C_g)^2 + (B - C_b)^2
```

The square-root operation normally associated with Euclidean distance is unnecessary because the square-root function is monotonic. The nearest centroid can be determined by comparing squared distances directly.

This choice provides two important advantages:

1. **Accuracy:** It better approximates radial distance in RGB space.
2. **Hardware Efficiency:** Squared terms map efficiently to DSP48 slices in AMD/Xilinx FPGA devices.

---

## 5. Proposed FPGA Microarchitecture

### 5.1 Pipeline Organization

The engine is implemented as a synchronous multi-stage pipeline. Its purpose is to maintain one-pixel-per-clock throughput while preserving timing closure at high clock frequencies.

| Pipeline Stage | Operation | Functional Role |
|---|---|---|
| S0 | Input capture | Receives 10-bit RGB pixel data and AXI4-Stream metadata |
| S1–S2 | Channel analysis | Finds Max/Mid/Min channels and quantizes 10-bit values to 8-bit |
| S3 | LUT mapping | Selects the active LUT bank or mode-dependent palette |
| S4–S28 | LUT pipeline | Provides stable centroid and boundary retrieval from BRAM |
| S29–S31 | Distance calculation | Computes parallel squared-Euclidean distances |
| S32–S34 | Minimum reduction | Finds the nearest centroid through a comparator tree |
| S35 | Output mapping | Maps the winning index to output RGB and restores 10-bit format |

### 5.2 Parallel Distance Engines

The architecture instantiates thirty concurrent distance engines. Each distance engine receives the same input pixel and one centroid value. The engines operate simultaneously, producing thirty squared-distance values.

For centroid index `k`:

```text
Dist_k = (P.R - Centroid[k].R)^2
       + (P.G - Centroid[k].G)^2
       + (P.B - Centroid[k].B)^2
```

The use of parallel distance engines avoids sequential centroid scanning and enables constant throughput independent of centroid count, provided sufficient DSP resources are available.

### 5.3 Comparator Reduction Tree

After all distances are computed, a pipelined comparator tree identifies the global minimum value. The tree compares the thirty distance outputs and propagates both the minimum distance and the associated centroid index.

This staged reduction approach supports timing closure by avoiding a single large combinational comparison network.

### 5.4 Output Mapping

The winning centroid index is used to retrieve the corresponding RGB output vector. Since the internal datapath uses 8-bit precision, the output value is rescaled to match the 10-bit video interface by appending two least-significant zero bits:

```text
oRgb = Clustered_Vector & "00"
```

---

## 6. Max/Mid/Min Abstraction Layer

### 6.1 Motivation

A direct RGB lookup-table approach is expensive because the RGB color cube contains 16.7 million possible 8-bit combinations. Storing classification data for all RGB combinations would consume excessive BRAM and would not scale well on resource-constrained FPGA platforms.

### 6.2 Channel Hierarchy Normalization

The proposed design introduces a Max/Mid/Min abstraction layer. For each incoming pixel, the hardware identifies the ordering of the RGB channel values. There are six possible channel permutations:

1. R > G > B  
2. R > B > G  
3. G > R > B  
4. G > B > R  
5. B > R > G  
6. B > G > R  

Instead of storing separate classification data for every physical RGB permutation, the system normalizes the pixel into an abstract channel hierarchy:

```text
Max = highest channel value
Mid = middle channel value
Min = lowest channel value
```

The classification logic then uses the normalized representation to access shared boundary patterns.

### 6.3 BRAM Reduction

This abstraction allows a single set of stored boundary patterns to support all six physical RGB orderings. As a result, the architecture can achieve an approximate 6:1 reduction in LUT memory footprint.

This reduction is significant for embedded FPGA platforms where BRAM must be shared among video buffers, ISP blocks, control logic, and additional accelerators.

---

## 7. Dynamic LUT Management

### 7.1 Runtime Palette Reconfiguration

Real-world machine vision systems often operate under changing environmental conditions. Lighting, object color distribution, camera exposure, and application context may require the active centroid palette to change at runtime.

The design supports runtime palette updates through a host-controlled LUT interface. New centroid values can be loaded while the engine continues processing video data.

### 7.2 Shadow Activation Strategy

To prevent visual tearing or inconsistent classification within a frame, the design uses a shadow-register or dual-bank activation strategy.

The mechanism operates as follows:

1. The active bank continues serving the current frame.
2. Host software writes new centroid values into the inactive bank.
3. A palette swap request is issued through the control interface.
4. The hardware defers the bank switch until a frame-safe boundary.
5. The new palette becomes active at the next valid vertical blanking or EOF/SOF transition.

This ensures that all pixels in a frame are classified using a consistent palette.

---

## 8. System Integration

### 8.1 Target Platform

The clustering engine is designed for integration into FPGA-based embedded vision platforms such as the AMD Kria KV260. The engine is encapsulated as part of a Video Frame Processing IP block and is connected directly to the streaming video pipeline.

### 8.2 PS/PL Control Flow

The end-to-end system uses both the Processing System (PS) and Programmable Logic (PL):

1. A host-side Python script serializes RGB centroid data.
2. The payload is transmitted over TCP.
3. An lwIP-based server running on the PS receives the update.
4. The PS writes centroid values into the FPGA IP through AXI4-Lite registers.
5. The PL clustering engine applies the new centroid codebook to the video stream.

### 8.3 Cache Coherence Considerations

When video buffers are shared between VDMA and software running on the PS, cache coherency must be handled carefully. The PS application should invalidate stale cache lines before reading DDR regions written by the VDMA engine. On Xilinx platforms, this may require calls such as:

```text
Xil_DCacheInvalidateRange(...)
```

The software frame-buffer copy logic must also use correctly typed 32-bit or platform-appropriate pointers to avoid address truncation on 64-bit Zynq UltraScale+ architectures.

---

## 9. Verification Methodology

### 9.1 Golden Reference Model

Verification is performed using a software Golden Reference Model. The reference model computes the expected centroid assignment for each input pixel using the same quantization and squared-distance rules as the RTL implementation.

The RTL output is compared against the software model to verify:

- RGB quantization correctness,
- centroid lookup correctness,
- squared-distance computation,
- minimum-distance selection,
- output RGB mapping,
- AXI4-Stream metadata alignment.

### 9.2 Pipeline and Metadata Alignment

Because the engine uses deep pipelining, pixel data and video metadata must be delayed by matching amounts. AXI4-Stream sideband signals such as `TUSER` and `TLAST` are passed through delay registers so they remain synchronized with the transformed pixel data.

This prevents frame corruption, misplaced start-of-frame markers, and incorrect end-of-line alignment in downstream video subsystems.

---

## 10. Performance Analysis

The proposed architecture is optimized for deterministic streaming throughput.

| Metric | Value |
|---|---|
| Throughput efficiency | 1.0 Cycles Per Pixel |
| Deterministic latency | Approximately 25–35 clock cycles |
| Target frequency | 200 MHz |
| Input bit depth | 10-bit RGB |
| Internal bit depth | 8-bit RGB |
| Output bit depth | 10-bit RGB |
| Distance engines | 30 parallel engines |
| Distance metric | Squared Euclidean distance |
| DSP usage | DSP48-oriented arithmetic mapping |
| Memory optimization | Max/Mid/Min LUT abstraction |
| Palette update method | Frame-safe shadow activation |

At a 200 MHz operating frequency, the architecture can theoretically process up to 200 million pixels per second. This exceeds the approximate pixel rate required for 1080p60 video and provides margin for additional control or video-processing stages.

---

## 11. Applications

The proposed FPGA RGB K-Means clustering engine is suitable for several real-time embedded vision domains.

### 11.1 Industrial Inspection

The engine can perform color-based defect detection, object sorting, surface classification, and inline quality control. Deterministic latency is valuable for conveyor systems and robotic actuators that require fixed timing.

### 11.2 Robotics Perception

Robotic systems can use the engine for high-speed object segmentation, environmental mapping, target tracking, and scene simplification before higher-level decision logic.

### 11.3 AI Preprocessing

The engine can reduce image complexity before neural-network inference by quantizing raw RGB data into a smaller set of representative color classes. This can reduce downstream memory bandwidth and model input complexity.

### 11.4 Embedded Vision Systems

The architecture is appropriate for edge devices where low power, low latency, and predictable operation are required.

---

## 12. Discussion

The design demonstrates how an algorithm traditionally expressed as an iterative software routine can be refactored into a deterministic hardware datapath. The key architectural decision is to separate centroid learning from centroid assignment. This allows the FPGA to focus on the high-throughput assignment stage while software or host-side control handles palette generation and updates.

The Max/Mid/Min abstraction layer is especially important because it addresses memory scalability. Rather than storing exhaustive RGB mappings, the system stores compact normalized boundary data. This enables more efficient use of BRAM while retaining classification flexibility.

The use of squared Euclidean distance provides a favorable balance between segmentation quality and FPGA implementation efficiency. By avoiding square-root computation and mapping arithmetic to DSP48 slices, the design maintains both mathematical relevance and timing feasibility.

---

## 13. Limitations and Future Work

Although the proposed design achieves high throughput and deterministic latency, several areas remain available for future improvement:

1. **Adaptive centroid learning in hardware:** Future versions may include partial or full centroid update logic inside the FPGA.
2. **Support for additional color spaces:** HSV, YCbCr, CIELAB, or normalized RGB could improve perceptual clustering accuracy.
3. **Configurable centroid count:** Runtime-selectable centroid counts could improve flexibility across applications.
4. **Multi-pixel-per-clock scaling:** Wider AXI4-Stream datapaths could support 4K or higher frame rates.
5. **Formal verification:** Property-based verification could be added for AXI protocol behavior and pipeline alignment.
6. **Resource benchmarking:** Detailed post-synthesis utilization and timing data should be collected across multiple FPGA families.

---

## 14. Conclusion

This paper presented a high-performance FPGA-based RGB K-Means clustering engine for deterministic real-time video processing. The architecture converts K-Means into a streaming assignment engine capable of one-pixel-per-clock throughput. Through thirty parallel squared-Euclidean distance engines, a pipelined minimum-reduction tree, Max/Mid/Min LUT abstraction, and frame-safe runtime palette updates, the system provides a practical hardware solution for embedded machine vision.

The design is suitable for industrial inspection, robotics perception, AI preprocessing, and other edge-vision workloads requiring predictable latency and high throughput. By combining algorithmic restructuring with FPGA-specific architectural optimization, the proposed engine demonstrates a scalable path toward real-time color clustering in resource-constrained embedded systems.

---

## References

[1] J. MacQueen, “Some Methods for Classification and Analysis of Multivariate Observations,” *Proceedings of the Fifth Berkeley Symposium on Mathematical Statistics and Probability*, 1967.

[2] R. C. Gonzalez and R. E. Woods, *Digital Image Processing*, Pearson.

[3] AMD/Xilinx, *AXI4-Stream Video IP and Interface Documentation*.

[4] AMD/Xilinx, *DSP48 Slice User Guide*.

[5] AMD/Xilinx, *Kria KV260 Vision AI Starter Kit Documentation*.

[6] S. Ali, “Technical Design Specification: High-Performance FPGA RGB K-Means Clustering Engine,” uploaded source specification, 2026.

---

## Appendix A: Simplified RTL-Oriented Operation

```text
for each incoming pixel:
    capture RGB and AXI sideband metadata
    quantize RGB from 10-bit to 8-bit
    determine Max/Mid/Min channel hierarchy
    select active LUT bank
    retrieve centroid values
    compute 30 parallel squared distances
    reduce distances through comparator tree
    select winning centroid index
    map winning index to output RGB
    delay metadata to match pixel latency
    emit clustered RGB and aligned AXI sideband signals
```

---

## Appendix B: Research Paper Summary Table

| Category | Description |
|---|---|
| Problem | Real-time RGB pixel classification at video rate |
| Proposed Solution | FPGA streaming K-Means assignment engine |
| Main Optimization | Parallel squared-distance computation |
| Memory Innovation | Max/Mid/Min abstraction layer |
| Runtime Feature | Frame-safe LUT/palette reconfiguration |
| Target Interface | AXI4-Stream video |
| Target Platform | AMD/Xilinx FPGA, Kria-class SOM |
| Key Result | 1.0 CPP deterministic clustering pipeline |
