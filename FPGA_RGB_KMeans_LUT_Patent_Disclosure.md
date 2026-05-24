# FPGA-Based Real-Time RGB K-Means Color Clustering Engine with Dynamic LUT Management, Max/Mid/Min Color Abstraction, Runtime Palette Reconfiguration, and Frame-Safe Shadow Activation

**Inventor / Author:** Sakinder Ali  
**Technology Area:** FPGA image processing, color clustering, edge AI preprocessing, real-time video pipelines, embedded vision systems, lookup table management, dynamic color abstraction.

---

## 1. Title of the Invention

**FPGA-Based Real-Time RGB K-Means Color Clustering Engine with Dynamic Max/Mid/Min LUT Abstraction, Runtime Palette Swapping, and Frame-Boundary Safe Shadow LUT Activation**

---

## 2. Field of the Invention

The present disclosure relates to **real-time image and video processing hardware**, and more particularly to an **FPGA-implemented RGB color clustering engine** configured to classify streaming pixels using parallel nearest-centroid computation, dynamic lookup table management, channel-hierarchy color abstraction, runtime reconfiguration, and deterministic video metadata alignment.

The disclosed invention is suitable for **embedded vision AI**, robotics, autonomous vehicles, industrial inspection, agricultural imaging, color-based object tracking, high-speed video segmentation, and edge-device preprocessing.

---

## 3. Background of the Invention

Modern image sensors generate extremely high data rates. A 1080p60 video stream requires processing more than 124 million pixels per second, and classifying each pixel against 30 color centroids may require billions of distance evaluations per second.

Conventional software-only RGB clustering systems suffer from several limitations:

| Limitation | Effect |
|---|---|
| Sequential or semi-parallel execution | Cannot reliably sustain high-pixel-rate streams |
| Cache misses and memory contention | Variable latency and frame delay |
| Operating system jitter | Non-deterministic perception timing |
| High floating-point overhead | Excessive power and thermal load |
| Large color-space memory requirements | Inefficient use of embedded memory |
| Runtime palette updates during active video | Risk of tearing, discontinuity, or unstable clustering |

In edge AI and autonomous systems, perception timing is safety-critical. A non-deterministic delay or metadata misalignment may cause incorrect downstream decisions. Therefore, there is a need for a **hardware-native, deterministic, runtime-adaptive color clustering engine** capable of operating directly inside the streaming video path.

---

## 4. Summary of the Invention

The invention provides an FPGA-based RGB K-Means color clustering engine that receives streaming RGB pixel data, analyzes channel hierarchy, selects and maps lookup table boundaries, computes distances to multiple centroids in parallel, identifies a nearest centroid using a hierarchical reduction tree, and outputs a clustered RGB signal with synchronized video metadata.

The invention further provides a specialized **LUT Management Module** that stores generic **Max/Mid/Min intensity boundary values** rather than exhaustive physical RGB coordinates. A dynamic channel mapper assigns those generic values to actual red, green, and blue channels according to each incoming pixel’s color hierarchy. This allows one abstract LUT entry to be reused across multiple RGB permutations, substantially reducing memory requirements.

In one embodiment, the engine includes:

1. A streaming RGB input interface.
2. A quantization stage.
3. A channel analyzer for Max/Mid/Min ordering and grayscale or high-contrast detection.
4. A dynamic LUT management subsystem.
5. Multiple LUT palette categories.
6. A runtime reconfiguration interface.
7. Direct overwrite support for LUT entries.
8. Command-index palette swapping.
9. Shadow LUT double-buffering.
10. A 25-stage or similar LUT distribution pipeline.
11. Thirty parallel clustering cores.
12. A hierarchical minimum-reduction tree.
13. A centroid output multiplexer.
14. A metadata alignment pipeline for Valid, SOF, EOF, and EOL.
15. A deterministic clustered RGB output stream.

---

## 5. Core Inventive Concept

The strongest inventive concept may be stated as follows:

> **A frame-safe, dynamically reconfigurable FPGA LUT management subsystem for real-time color clustering, wherein generic Max/Mid/Min color-boundary entries are stored instead of literal RGB coordinates, dynamically mapped to physical RGB channels based on incoming pixel hierarchy, updated at runtime through direct write indices or command-based palette swapping, staged in a shadow LUT, and atomically activated only at a safe video frame boundary to prevent visual tearing or clustering discontinuity.**

This feature combines **memory compression**, **runtime adaptability**, **deterministic hardware routing**, and **video-safe update control** into a single FPGA color clustering subsystem.

---

## 6. Technical Problems Solved

| Technical Problem | Inventive Solution |
|---|---|
| Exhaustive RGB LUT storage is impractical | Store generic Max/Mid/Min boundary triples |
| RGB permutations multiply storage demand | Dynamically map abstract boundaries to physical channels |
| Software clustering is too slow | Use FPGA streaming pipeline |
| Multiple centroid comparisons are expensive | Compute distances in parallel |
| Runtime palette changes may corrupt frames | Stage updates in shadow LUT |
| Mid-frame LUT update causes visual tearing | Activate shadow LUT only at frame boundary |
| Lighting conditions change during operation | Permit runtime direct overwrite and palette swap |
| Host software needs confirmation | Provide readback through LUT output path |
| Invalid update commands can destabilize hardware | Apply index validation and controlled dispatch |
| Pixel data can drift from metadata | Use matched sideband delay pipeline |

---

## 7. System Architecture

### 7.1 High-Level Block Diagram Description

A representative system includes the following functional blocks:

```text
Image Sensor / Video Stream
        ↓
RGB Input Capture
        ↓
Bit-Depth Quantization
        ↓
Channel Analyzer
(Max / Mid / Min, Grayscale, High-Contrast)
        ↓
Dynamic LUT Management Module
        ↓
Max/Mid/Min-to-RGB Channel Mapper
        ↓
LUT Pipeline Distribution
        ↓
30 Parallel RGB Cluster Cores
        ↓
Distance Computation
        ↓
Hierarchical Min-Reduction Tree
        ↓
Winning Centroid Selection
        ↓
Output RGB / Cluster ID Formatting
        ↓
Metadata-Aligned Clustered Video Stream
```

The system may be deployed in a Kria KV260-style vision pipeline using MIPI camera input, demosaic processing, the K-Means video-processing core, VDMA, DDR, processor-side control, and network streaming.

---

## 8. LUT Management Module

### 8.1 Function of the LUT Management Module

The LUT Management Module acts as the **adaptive memory and routing layer** of the RGB K-Means clustering engine. It stores, selects, updates, validates, and distributes centroid boundary values used by downstream clustering cores.

Unlike a conventional LUT that stores fixed RGB coordinates, the disclosed LUT module stores abstract color-boundary roles:

| Field | Bit Range | Meaning |
|---|---:|---|
| Max boundary | bits 23:16 | Highest intensity reference |
| Mid boundary | bits 15:8 | Middle intensity reference |
| Min boundary | bits 7:0 | Lowest intensity reference |

These generic values are not permanently tied to red, green, or blue. Instead, they are dynamically mapped based on the analyzed hierarchy of the incoming pixel.

---

### 8.2 Max/Mid/Min Memory Compression

The RGB color space contains approximately 16.7 million 24-bit color possibilities. Storing literal RGB mappings for the entire space is impractical in FPGA memory.

The disclosed invention reduces this burden by storing **generic intensity roles** rather than absolute channel positions.

For example, if an incoming pixel has the hierarchy:

```text
R > G > B
```

then:

```text
Max → R
Mid → G
Min → B
```

If a later pixel has the hierarchy:

```text
B > R > G
```

then:

```text
Max → B
Mid → R
Min → G
```

Thus, one generic LUT entry can represent multiple RGB permutations.

---

### 8.3 Channel-Hierarchy Mapper

The channel-hierarchy mapper evaluates each incoming pixel and determines which physical channel is maximum, middle, and minimum.

Possible channel orderings include:

1. R > G > B
2. R > B > G
3. G > R > B
4. G > B > R
5. B > R > G
6. B > G > R

The mapper then assigns the abstract Max/Mid/Min LUT values into actual RGB centroid positions for distance computation.

This allows the LUT subsystem to behave as a **compressed symbolic color model** rather than a literal physical-coordinate table.

---

## 9. Multi-Palette Subsystem

The disclosed LUT management module supports multiple categories of LUT palettes.

### 9.1 Base Intensity Palettes

Base intensity palettes provide general-purpose segmentation for brightness, intensity, and tone grouping.

Examples include:

```text
k_rgb_lut_2_l
k_rgb_lut_3_l
k_rgb_lut_4_l
k_rgb_lut_6_l
```

These palettes may be used for default clustering when no specialized condition is detected.

---

### 9.2 Operational Palettes

Operational palettes handle special pixel conditions such as:

- grayscale pixels,
- low-chroma scenes,
- high-contrast pixels,
- desaturated visual regions,
- neutral lighting conditions.

Examples include:

```text
k_rgb_lut_0_l
k_rgb_lut_1_l
```

These palettes stabilize output when ordinary chromatic segmentation may be unreliable.

---

### 9.3 Profiling Palettes

Profiling palettes are hardcoded or precomputed target-color profiles for application-specific tracking.

Examples include:

```text
k_rgb_lut_41l through k_rgb_lut_59l
```

These may be used to isolate particular target colors, including brown or earth-tone profiles for terrain, soil, object tracking, or agricultural imaging.

---

## 10. Runtime Reconfiguration

### 10.1 Zero-Downtime Adaptability

The LUT module supports runtime reconfiguration without:

- shutting down FPGA hardware,
- regenerating the bitstream,
- recompiling the design,
- stopping the video stream,
- halting downstream processing.

An external host controller can modify active clustering behavior through control signals or ports such as:

```text
k_lut_in
k_lut_out
k_ind_w
k_ind_r
centroid_lut_in
centroid_lut_out
```

---

### 10.2 Direct Entry Overwrite

In one embodiment, write indices from **0 through 120** are used for direct overwrite.

A host controller sends:

```text
write index + 24-bit Max/Mid/Min payload
```

The LUT module decodes the write index and stores the payload into the appropriate LUT entry.

This enables:

- field calibration,
- lighting adaptation,
- camera-specific tuning,
- environmental retargeting,
- live clustering adjustment.

---

### 10.3 Index-Controlled Palette Swapping

In another embodiment, write indices from **201 through 221** operate as command indices.

Instead of writing one entry, these command indices trigger a palette swap. For example:

```text
write index 220 → load brown-tracking profile
```

The system may flush or replace the active working palette with one of the predefined profiling palettes.

This enables instant mode switching, such as:

- normal segmentation mode,
- grayscale mode,
- high-contrast mode,
- vegetation mode,
- soil-detection mode,
- brown-object tracking mode,
- target-color isolation mode.

---

## 11. Shadow LUT Double-Buffering

### 11.1 Problem of Mid-Frame Updates

Updating LUT entries while a video frame is actively being processed can cause:

- visual tearing,
- inconsistent pixel classification,
- cluster discontinuity,
- half-frame palette mismatch,
- metadata-to-pixel inconsistency,
- unstable downstream AI output.

### 11.2 Shadow LUT Solution

To prevent this, the disclosed module uses a **shadow LUT**.

Runtime updates are first written into an inactive memory region. The active LUT remains unchanged while the current frame is processed.

At a safe synchronization point, such as:

- frame boundary,
- blanking interval,
- Start-of-Frame transition,
- End-of-Frame transition,

the hardware atomically switches the active LUT pointer from the old table to the shadow table.

---

## 12. Readback, Validation, and Control Safety

The LUT module may include validation and telemetry logic.

### 12.1 Index Validation

The module may reject or ignore invalid write indices.

Example valid regions:

| Index Range | Function |
|---:|---|
| 0–120 | Direct LUT overwrite |
| 201–221 | Palette swap command |
| Other ranges | Ignored, rejected, or reserved |

### 12.2 Readback Support

A readback path such as:

```text
centroid_lut_out
k_lut_out
```

allows host software to verify that LUT updates were successfully applied.

### 12.3 Deterministic Tie Policy

The system may include a deterministic tie policy. For example, when two clusters have equal distance, the lower cluster index may be selected.

This ensures repeatable classification.

---

## 13. Reversed Indexing Dispatch

In one embodiment, write and read dispatchers controlled by `k_ind_w` and `k_ind_r` apply reversed indexing for certain ranges.

For example:

```text
index 31 maps to 61 - k_ind_w
```

This may be used to maintain structural consistency across LUT slices or mirror palette segments.

---

## 14. Pipelined LUT Distribution

After LUT boundaries are selected and mapped, the values are pushed through a deep pipeline before reaching the clustering cores.

A representative **25-stage shift register pipeline** may be expressed as:

```text
k1_rgb → k2_rgb → ... → k25_rgb
```

At the end of this pipeline, 8-bit index values may be scaled into 10-bit formatting and broadcast to 30 parallel `rgb_cluster_core` instances.

This ensures:

- timing closure,
- deterministic latency,
- synchronized centroid availability,
- stable distribution to parallel cores,
- alignment with pixel and metadata pipelines.

---

## 15. Distance Computation Core

### 15.1 Parallel Clustering Cores

The invention may include **30 parallel distance engines** or `rgb_cluster_core` instances.

Each engine receives:

- incoming RGB pixel data,
- mapped centroid boundary values,
- pipeline-aligned control signals.

Each core computes a distance between the incoming pixel and one centroid.

---

### 15.2 Squared Euclidean Embodiment

In one embodiment, distance is computed as:

```text
D² = (R - Cr)² + (G - Cg)² + (B - Cb)²
```

The square root operation is omitted because comparing squared distances preserves nearest-neighbor ordering.

This improves timing closure and reduces unnecessary hardware overhead.

---

### 15.3 Manhattan Distance Embodiment

In another embodiment, distance is computed as:

```text
D = |R - Cr| + |G - Cg| + |B - Cb|
```

This avoids multipliers and square-root logic and may reduce FPGA slice or DSP usage.

---

## 16. Hierarchical Min-Reduction Tree

After distance computation, the system selects the nearest centroid.

A hierarchical min-reduction tree may perform:

1. Group-level minimum comparisons.
2. Intermediate minimum comparisons.
3. Final global minimum selection.

A representative reduction may reduce 30 thresholds into six group minimums, then two half-array minimums, and finally one global winner.

This structure avoids a long sequential comparator chain and supports higher clock frequency.

---

## 17. Metadata Alignment

The invention includes a metadata delay pipeline that aligns video sideband signals with processed clustered pixels.

Sideband signals may include:

- Valid,
- Start of Frame, SOF,
- End of Frame, EOF,
- End of Line, EOL,
- pixel coordinate markers,
- frame boundary indicators.

The output may have a fixed pipeline latency of approximately **25–35 clock cycles**, with sideband alignment through a matched shift-register path.

---

## 18. Output Format

The output may include one or more of:

- clustered RGB pixel,
- centroid index,
- symbolic cluster ID,
- segmented video signal,
- object mask,
- grayscale-classified output,
- high-contrast-classified output,
- feature map for downstream AI.

In a 10-bit video embodiment, the internal 8-bit clustered value may be restored to 10-bit compatibility by left-shifting and appending low-order zeros.

---

## 19. Example Method of Operation

A method for real-time FPGA RGB color clustering may include:

1. Receiving streaming RGB pixel data.
2. Capturing pixel data and sideband metadata.
3. Quantizing RGB channels for internal computation.
4. Determining Max/Mid/Min channel hierarchy.
5. Detecting grayscale, low-chroma, or high-contrast pixel conditions.
6. Selecting a LUT palette based on mode, pixel condition, or host command.
7. Reading a generic Max/Mid/Min LUT entry.
8. Dynamically mapping the generic LUT entry to physical RGB channels.
9. Distributing the mapped centroid data through a pipeline.
10. Broadcasting the centroid data to multiple parallel cluster cores.
11. Computing distance values in parallel.
12. Reducing distance values through a hierarchical comparator tree.
13. Selecting a winning centroid.
14. Formatting the clustered output pixel.
15. Delaying metadata through a matched sideband pipeline.
16. Outputting the clustered pixel with aligned metadata.
17. Receiving runtime LUT updates from an external host.
18. Writing updates into a shadow LUT.
19. Waiting for a safe frame boundary.
20. Atomically activating the shadow LUT.

---

## 20. Example Deployment

A representative embedded vision system may include:

```text
MIPI Camera Sensor
        ↓
MIPI CSI-2 Receiver
        ↓
Demosaic / Color Reconstruction
        ↓
FPGA RGB K-Means Clustering Core
        ↓
VDMA
        ↓
DDR Memory
        ↓
Processor System / Network Stream
```

Runtime LUT updates may be transmitted from a remote host using TCP or UDP, received by the processing system, and written into programmable logic registers through AXI-Lite.

---

## 21. Advantages of the Invention

### 21.1 Hardware Performance Advantages

- One-pixel-per-clock streaming throughput.
- Deterministic fixed latency.
- Parallel distance computation.
- Reduced CPU and GPU load.
- Reduced memory bandwidth demand.
- FPGA-friendly arithmetic.
- Deeply pipelined timing closure.

### 21.2 LUT Management Advantages

- Max/Mid/Min storage compression.
- Reuse of one generic entry across RGB permutations.
- Multiple palette categories.
- Runtime direct overwrite.
- Command-index palette swapping.
- Frame-safe shadow activation.
- Host-side readback and validation.
- Safe field calibration without bitstream regeneration.

### 21.3 Edge AI Advantages

- Converts raw RGB data into simplified clustered features.
- Reduces downstream AI input complexity.
- Improves segmentation stability.
- Enables color-based object tracking.
- Supports dynamic environmental adaptation.
- Enables compact embedded deployment.

---

# 22. Draft Patent Claims

## Claim 1 — Independent System Claim

A real-time color clustering system comprising:

- a streaming pixel input configured to receive RGB pixel data;
- a channel analyzer configured to determine a relative intensity hierarchy among red, green, and blue channel values of an incoming pixel;
- a lookup table memory configured to store generic intensity boundary entries comprising a maximum boundary value, a middle boundary value, and a minimum boundary value;
- a dynamic channel mapper configured to map the maximum boundary value, the middle boundary value, and the minimum boundary value to physical red, green, and blue centroid channels based on the relative intensity hierarchy;
- a plurality of parallel distance computation circuits configured to compute respective distances between the incoming pixel and a plurality of mapped centroid values;
- a reduction circuit configured to identify a minimum distance among the respective distances;
- an output selection circuit configured to output a clustered pixel value corresponding to a centroid associated with the minimum distance; and
- a metadata alignment pipeline configured to delay one or more video sideband signals to align with the clustered pixel value.

---

## Claim 2 — Max/Mid/Min LUT Entry Format

The system of claim 1, wherein each generic intensity boundary entry is a 24-bit value comprising:

- bits 23 through 16 representing the maximum boundary value;
- bits 15 through 8 representing the middle boundary value; and
- bits 7 through 0 representing the minimum boundary value.

---

## Claim 3 — RGB Permutation Reuse

The system of claim 1, wherein a single generic intensity boundary entry is reusable across a plurality of RGB channel permutations by the dynamic channel mapper.

---

## Claim 4 — Runtime Direct Overwrite

The system of claim 1, further comprising a runtime control interface configured to overwrite one or more lookup table entries during operation of the streaming pixel input.

---

## Claim 5 — Direct Write Index Range

The system of claim 4, wherein write indices from 0 through 120 correspond to direct overwrite locations for writing generic Max/Mid/Min boundary values into the lookup table memory.

---

## Claim 6 — Command-Based Palette Swapping

The system of claim 4, wherein a command index received through the runtime control interface triggers replacement of an active lookup table palette with a predefined profiling palette.

---

## Claim 7 — Palette Swap Index Range

The system of claim 6, wherein command indices from 201 through 221 correspond to predefined palette swap commands.

---

## Claim 8 — Shadow LUT Double-Buffering

The system of claim 1, further comprising:

- an active lookup table;
- a shadow lookup table; and
- a control circuit configured to receive runtime updates into the shadow lookup table and atomically switch from the active lookup table to the shadow lookup table at a video frame boundary.

---

## Claim 9 — Frame-Safe Activation

The system of claim 8, wherein the control circuit prevents activation of the shadow lookup table during an active frame interval to avoid visual tearing or clustering discontinuity.

---

## Claim 10 — Multi-Palette Subsystem

The system of claim 1, wherein the lookup table memory comprises a plurality of palette categories including:

- a base intensity palette;
- an operational palette; and
- a profiling palette.

---

## Claim 11 — Operational Palette Selection

The system of claim 10, wherein the operational palette is selected responsive to detecting a grayscale, low-chroma, or high-contrast condition.

---

## Claim 12 — Profiling Palette

The system of claim 10, wherein the profiling palette comprises a hardcoded target-color profile for application-specific color tracking.

---

## Claim 13 — Readback Verification

The system of claim 1, further comprising a readback output configured to provide stored centroid or boundary values to an external host controller for verification.

---

## Claim 14 — Index Validation

The system of claim 1, further comprising an index validation circuit configured to reject or ignore invalid runtime write commands.

---

## Claim 15 — Reversed Index Dispatch

The system of claim 1, wherein a read or write dispatcher maps at least one received index to a reversed lookup table position for one or more predetermined index ranges.

---

## Claim 16 — Parallel Cluster Cores

The system of claim 1, wherein the plurality of parallel distance computation circuits comprises thirty RGB cluster cores.

---

## Claim 17 — Pipelined LUT Distribution

The system of claim 1, further comprising a multi-stage pipeline configured to distribute mapped centroid values to the plurality of parallel distance computation circuits.

---

## Claim 18 — Twenty-Five Stage Pipeline

The system of claim 17, wherein the multi-stage pipeline comprises approximately twenty-five pipeline stages.

---

## Claim 19 — Squared Euclidean Distance

The system of claim 1, wherein each distance computation circuit computes a squared Euclidean distance without performing a square-root operation.

---

## Claim 20 — Manhattan Distance

The system of claim 1, wherein each distance computation circuit computes a Manhattan distance based on absolute channel differences.

---

## Claim 21 — Hierarchical Reduction Tree

The system of claim 1, wherein the reduction circuit comprises a hierarchical comparator tree configured to reduce distance candidates through multiple pipelined comparison stages.

---

## Claim 22 — Deterministic Throughput

The system of claim 1, wherein the system is configured to process one pixel per clock cycle after an initial pipeline latency.

---

## Claim 23 — Fixed Latency

The system of claim 1, wherein the clustered pixel value is output after a fixed latency of approximately 25 to 35 clock cycles.

---

## Claim 24 — Metadata Signals

The system of claim 1, wherein the one or more video sideband signals comprise at least one of Valid, Start of Frame, End of Frame, or End of Line.

---

## Claim 25 — Edge AI Preprocessing

The system of claim 1, wherein the clustered pixel value is provided to a downstream artificial intelligence inference engine as a reduced-complexity feature representation.

---

# 23. Method Claims

## Claim 26 — Independent Method Claim

A method of real-time RGB color clustering in programmable logic, the method comprising:

1. receiving streaming RGB pixel data;
2. determining a relative channel hierarchy of red, green, and blue values for an incoming pixel;
3. reading a generic lookup table entry comprising a maximum boundary value, a middle boundary value, and a minimum boundary value;
4. mapping the maximum boundary value, middle boundary value, and minimum boundary value to physical RGB centroid channels based on the relative channel hierarchy;
5. computing, in parallel, distances between the incoming pixel and a plurality of mapped centroids;
6. identifying a minimum distance using a reduction circuit;
7. selecting a clustered output value corresponding to the minimum distance;
8. aligning video metadata with the clustered output value; and
9. outputting the clustered output value with the aligned video metadata.

---

## Claim 27 — Runtime LUT Update Method

The method of claim 26, further comprising receiving a runtime update command from an external host controller and writing a new generic Max/Mid/Min boundary value into a lookup table without regenerating an FPGA bitstream.

---

## Claim 28 — Shadow LUT Update Method

The method of claim 27, further comprising:

1. writing the runtime update command into a shadow lookup table;
2. maintaining an active lookup table during a current video frame;
3. detecting a frame boundary; and
4. atomically switching the shadow lookup table into active operation at the frame boundary.

---

## Claim 29 — Palette Swap Method

The method of claim 26, further comprising receiving a command index and loading a predefined profiling palette in response to the command index.

---

## Claim 30 — Grayscale Palette Method

The method of claim 26, further comprising detecting a grayscale or low-chroma condition and selecting an operational palette responsive to the detected grayscale or low-chroma condition.

---

# 24. Abstract

An FPGA-based real-time RGB color clustering engine is disclosed. The engine receives streaming RGB pixel data, determines a Max/Mid/Min channel hierarchy, dynamically maps generic lookup table boundary values to physical RGB channels, computes distances to multiple centroids using parallel hardware clustering cores, selects a nearest centroid through a hierarchical reduction circuit, and outputs a clustered video stream with aligned metadata. A lookup table management module stores generic Max/Mid/Min boundary entries instead of exhaustive RGB coordinates, supports multiple palette categories, permits runtime direct overwrite and command-index palette swapping, stages updates in a shadow lookup table, and atomically activates updated palettes at video frame boundaries to prevent tearing or clustering discontinuity. The system provides deterministic one-pixel-per-clock throughput and is suitable for embedded vision AI, robotics, autonomous systems, industrial inspection, agricultural imaging, and edge-device preprocessing.

---

## 25. Patentability Focus Areas

The strongest patent-facing areas are:

1. **Max/Mid/Min LUT abstraction**  
   Generic intensity-role storage instead of literal RGB coordinates.

2. **Dynamic RGB channel mapping**  
   Hardware mapping of abstract boundaries to physical channels per pixel hierarchy.

3. **Runtime LUT control protocol**  
   Direct overwrite indices and command-index palette swapping.

4. **Frame-safe shadow LUT activation**  
   Double-buffered LUT update with atomic frame-boundary switching.

5. **Integrated deterministic clustering pipeline**  
   LUT abstraction feeding parallel cluster cores with metadata-aligned output.

6. **Multi-palette adaptive color intelligence**  
   Base, operational, and profiling palettes selected dynamically for changing conditions.

---

## 26. Filing Note

This is a **technical invention disclosure draft** suitable for review by a registered patent attorney or patent agent. Before filing, recommended professional steps include:

1. Prior-art search.
2. Claim-scope refinement.
3. Invention ownership and inventorship review.
4. Drawing preparation.
5. Provisional or non-provisional patent application drafting.
6. Verification of public disclosure dates and confidentiality history.
