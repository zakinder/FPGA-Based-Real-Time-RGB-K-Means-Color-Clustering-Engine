# Formal Patent-Style Technical Disclosure

## FPGA LUT Management Module for Multi-Palette Color Clustering

**Inventor / Applicant:** Sakinder Ali  
**Technical Domain:** FPGA-based video processing, real-time color clustering, hardware-accelerated image quantization, LUT-based adaptive palette selection.  
**Disclosure Basis:** Technical blueprint titled *FPGA LUT Management Module for Multi-Palette Color Clustering*.

---

## 1. Title

**FPGA-Based Dynamic LUT Management System for Real-Time Multi-Palette RGB Color Clustering and Quantized Video Output**

---

## 2. Field of the Invention

The present disclosure relates generally to digital image and video processing hardware. More particularly, the disclosure relates to a field-programmable gate array, FPGA, architecture for real-time RGB color clustering using dynamically selectable look-up table, LUT, banks, parallel centroid-distance engines, hierarchical threshold reduction, and latency-aligned output arbitration.

The disclosed system may be used in embedded vision systems, FPGA image-processing pipelines, machine vision, industrial inspection, robotics, autonomous systems, color quantization, image segmentation, and video-stream simplification.

---

## 3. Background

Conventional software-based K-Means color clustering performs iterative centroid calculation and pixel assignment in a sequential or semi-parallel manner. While useful in offline image processing, such approaches are often unsuitable for deterministic high-throughput video pipelines because they require repeated passes over image data, dynamic memory access, and iterative convergence.

Modern video-processing applications require low-latency classification of pixels at rates suitable for high-definition streams, including pixel clocks such as 148.5 MHz for 1080p-class video. Software clustering may create bottlenecks when attempting to classify each incoming pixel in real time.

Existing FPGA image-processing systems may perform thresholding, filtering, or fixed color mapping, but they often lack a flexible multi-palette architecture that can switch between adaptive learned centroid banks and pre-trained stable centroid banks during operation. There is therefore a need for a deterministic hardware architecture that supports:

1. Real-time single-pass RGB pixel classification.
2. Multiple LUT banks for palette selection.
3. Runtime centroid updates through a control interface.
4. Parallel distance computation against multiple color centroids.
5. Hierarchical selection of the nearest centroid.
6. Cycle-accurate alignment of RGB output and video sideband signals.

---

## 4. Summary of the Invention

The disclosed invention provides an FPGA-based LUT management and color clustering module for real-time video streams. The system receives RGB pixel data, selects one or more LUT banks according to a configurable clustering policy, computes distances between the incoming pixel and a plurality of centroid colors in parallel, reduces the resulting distance values through a hierarchical minimum-selection tree, and outputs a quantized or clustered RGB pixel synchronized with delayed video-control sideband signals.

In one embodiment, the module processes pixels through a deeply pipelined RTL architecture and supports one-pixel-per-cycle throughput. A plurality of clustering engines, for example thirty parallel clustering instances, compare a synchronized RGB pixel against respective centroid entries. Distance values are calculated without square-root operations by comparing squared distances, thereby reducing resource usage and increasing timing closure feasibility.

The system includes a dynamic LUT memory architecture having live-learning LUT variants and archived or pre-trained LUT variants. A control register, such as a configuration register mapped through AXI4-Lite, selects the active policy or LUT bank. A data stream interface, such as an AXI4-Stream-compatible interface, carries input and output RGB pixel data together with video sideband signals.

---

## 5. Brief Description of the Drawings

Although drawings are not included in this text disclosure, the following figures may be prepared from the described invention:

- **Figure 1:** Top-level block diagram of the FPGA RGB clustering module.
- **Figure 2:** AXI4-Stream data plane and AXI4-Lite control plane integration.
- **Figure 3:** Dynamic LUT bank dispatcher showing live-learning and archived LUT banks.
- **Figure 4:** Chromatic decision engine selecting LUT policies based on RGB dominance and intensity thresholds.
- **Figure 5:** Parallel clustering array containing thirty centroid-distance engines.
- **Figure 6:** Hierarchical minimum-threshold reduction tree.
- **Figure 7:** Temporal alignment pipeline for RGB samples and sideband signals.
- **Figure 8:** Final output arbitration and diagnostic fallback state.

---

## 6. Detailed Description of the Invention

### 6.1 System Overview

The disclosed FPGA module implements a hardware-accelerated RGB color clustering engine. The system receives a video stream containing RGB pixel values and associated timing or framing signals. The pixel is converted or normalized into an internal representation and is evaluated against multiple centroid values stored in LUT banks.

The module avoids iterative software-style clustering during active streaming. Instead, the system performs deterministic per-pixel classification using preloaded or dynamically updated centroid values. This allows the architecture to operate in a continuous video stream without stalling the data path.

In one implementation, the system includes:

| Functional Block | Purpose |
|---|---|
| Input formatter | Receives RGB pixel stream and sideband signals |
| LUT management module | Stores and dispatches centroid values |
| Policy selector | Chooses active LUT behavior based on configuration and pixel properties |
| Parallel clustering array | Computes pixel-to-centroid distances |
| Minimum-reduction tree | Selects nearest centroid |
| Output arbiter | Assigns final clustered RGB output |
| Synchronization pipeline | Aligns sideband signals with processed pixels |

### 6.2 Data Plane

The data plane may follow an AXI4-Stream-compatible video pattern. The input stream may include RGB pixel data and sideband signals such as valid, start of frame, end of frame, and end of line.

The output stream delivers a clustered or quantized RGB pixel with correspondingly delayed sideband signals. This enables the module to be inserted into a larger FPGA video pipeline without disrupting frame timing.

In one embodiment, the input RGB format includes 10-bit red, green, and blue channels. Internal processing may reduce these channels to 8-bit values using a bit-slice conversion, such as selecting bits 9 downto 2. When formatting the output, the clustered 8-bit value may be left-shifted by two bits to restore alignment with a 10-bit output representation.

### 6.3 Control Plane

The module may include a register-mapped control interface, such as an AXI4-Lite interface. A processor or embedded controller can write configuration values to control registers. In one embodiment, a configuration register, identified in the source blueprint as `cfigReg45`, maps to a signal named `k_lut_selected`.

The control plane may support:

1. Selection of clustering policy.
2. Selection of active LUT bank.
3. Runtime write access to centroid entries.
4. Diagnostic read-back of LUT values.
5. Switching between adaptive and archived palette modes.

Example control and LUT signals include:

| Signal | Function |
|---|---|
| `clk` | System clock |
| `rst_l` | Active-low reset |
| `k_lut_selected` | Active clustering policy selector |
| `k_lut_in` | RGB centroid write input |
| `k_lut_out` | RGB centroid read-back output |
| `k_ind_w` | LUT write index |
| `k_ind_r` | LUT read index |

---

## 7. Dynamic LUT Architecture

### 7.1 Live-Learning and Archived LUT Banks

The memory architecture may include multiple LUT banks. Some LUT banks may be dynamically updated during runtime and may be characterized as live-learning banks. Other LUT banks may hold pre-trained, stable, or archived centroid values.

For formal patent terminology, these may be described as adaptive LUT banks configured to receive updated centroid values based on runtime or frame-based statistics, and static LUT banks configured to store predetermined centroid values for stable color quantization.

The combination allows the system to respond to current scene content while preserving known stable palettes for predictable operation.

### 7.2 LUT Write Dispatching

The LUT dispatcher may route incoming centroid values to selected LUT banks based on the write index. In one embodiment, index ranges are mapped as follows:

| Write Index Range | Target LUT Bank | Indexing Mode |
|---|---|---|
| 0-30 | `k_rgb_lut_4ll` | Direct indexing |
| 31-60 | `k_rgb_lut_2ll` | Reversed indexing |
| 61-90 | `k_rgb_lut_3ll` | Reversed indexing |
| 91-120 | `k_rgb_lut_6ll` | Reversed indexing |

A reversed index may be calculated using an offset subtraction operation. For example, a write index in the range 31-60 may be mapped using `61 - k_ind_w`.

This indexing structure allows multiple LUT banks to be addressed through a unified index space while preserving distinct internal memory layouts.

### 7.3 Adaptive and Static Selection

The dispatcher may select adaptive LUT variants when the write or selection index falls within a first range. In one embodiment, indices less than or equal to 90 activate adaptive live-learning variants. A second range, such as indices 201-221, may activate stable archived variants.

This allows a single control mechanism to switch between runtime-updated clustering behavior and predetermined color knowledge.

---

## 8. Chromatic Decision Engine

The chromatic decision engine determines which LUT or clustering template should be used for an incoming pixel. The decision may be based on maximum RGB channel value, minimum RGB channel value, middle RGB channel value, difference between maximum and minimum channels, dominant color channel, and configured policy value.

The system may compute `rgb_max`, `rgb_min`, `rgb_mid`, channel dominance, and a contrast value such as `rgb_max - rgb_min`.

The selected LUT may receive a permuted RGB ordering, such as `(max, min, mid)` or `(max, mid, min)`, depending on the policy.

### 8.1 Policy 0: Contrast-Based Selection

In one embodiment, Policy 0 performs a contrast check. If the difference between the maximum and minimum RGB channel values is greater than or equal to a threshold, such as 100, a high-contrast decision branch is activated.

Policy 0 may include grayscale detection. For example, if the absolute difference between red and green is less than or equal to a small threshold, and the absolute difference between red and blue is also less than or equal to the same or another small threshold, the pixel may be treated as grayscale.

Example grayscale condition:

```text
|R - G| <= 6 and |R - B| <= 6
```

If grayscale is detected, a grayscale or neutral LUT may be selected. If red, green, or blue dominance is detected, a corresponding LUT template may be selected.

### 8.2 Policy 1: Intensity-Gated Selection

Policy 1 may simplify decision-making by using intensity rather than contrast. In one embodiment, the maximum RGB channel is compared to a threshold, such as 100. If the maximum channel satisfies the threshold, a corresponding LUT path is selected.

This policy may be useful for stable operation in moderate lighting conditions where full contrast analysis is unnecessary.

### 8.3 Policy 9: Three-Tiered Intensity Selection

Policy 9 may divide the pixel intensity domain into three tiers:

| Intensity Range | Example Threshold | LUT Behavior |
|---|---:|---|
| High intensity | `max >= 170` | Vivid or saturated palette |
| Medium intensity | `max >= 85` | Moderate palette |
| Low intensity | `max < 85` | Subdued or low-light palette |

This permits finer-grained control of color clustering based on brightness or saturation intensity.

---

## 9. Parallel Clustering Array

The system includes a plurality of clustering engines operating in parallel. In one embodiment, thirty clustering instances are instantiated, identified as `color_k1_clustering_inst` through `color_k30_clustering_inst`.

Each clustering instance receives a shared synchronized RGB pixel input, a unique centroid value from the active LUT bank, and control or enable signals where required. Each clustering instance outputs a distance threshold value representing the distance between the input pixel and its assigned centroid.

### 9.1 Distance Computation

The distance calculation may be performed using squared Euclidean distance without square-root calculation. For RGB channels, the distance may be calculated as:

```text
D = (Rin - Rc)^2 + (Gin - Gc)^2 + (Bin - Bc)^2
```

Where `Rin`, `Gin`, and `Bin` are input pixel channels; `Rc`, `Gc`, and `Bc` are centroid channels; and `D` is the squared distance value.

Because square-root calculation is unnecessary when comparing distances, the system may compare squared distances directly. This reduces DSP, LUT, and timing overhead.

### 9.2 One-Pixel-Per-Cycle Throughput

By unrolling the centroid comparison into parallel clustering instances, the system can evaluate all centroid candidates for a pixel during the same pipeline interval. Once the pipeline is filled, the architecture may sustain one pixel per clock cycle.

This is particularly advantageous for high-resolution video streams requiring deterministic timing.

---

## 10. Hierarchical Minimum-Threshold Reduction

The outputs of the parallel clustering instances are reduced through a hierarchical minimum-selection tree. The purpose of the reduction tree is to determine which centroid has the smallest distance from the incoming pixel.

In one embodiment, thirty threshold values are first grouped into six groups of five. Each group produces a local minimum. The local minima are then reduced into two half-array minima, and the final global minimum is selected from those two values.

| Stage | Operation |
|---|---|
| Stage 1 | 30 thresholds reduced into 6 local minima |
| Stage 2 | 6 local minima reduced into 2 half-array minima |
| Stage 3 | 2 half-array minima reduced into 1 global minimum |

The final global minimum identifies the nearest centroid and therefore the appropriate clustered output color.

---

## 11. Temporal Synchronization and Output Arbitration

Because the clustering computation is pipelined, RGB pixel data and video sideband signals must be delayed by matching amounts. Misalignment could cause frame corruption, incorrect output timing, or sideband mismatches.

In one embodiment, the system includes a sideband delay pipeline having approximately 31 stages, an RGB staging pipeline having approximately 27 stages, and a delayed threshold-record array used for final output matching.

The sideband pipeline may be tapped at a specific stage, such as stage 25, to align valid, start-of-frame, end-of-line, and end-of-frame signals with the clustered pixel output.

### 11.1 Final Output Selection

After the global minimum threshold is identified, the output arbiter compares the global minimum against delayed threshold records. When a match is found, the RGB value corresponding to the matching centroid is assigned to the output.

If no match is found, a diagnostic fallback value may be assigned. In one embodiment, the output RGB value is set to all logic ones, indicating an error or unmatched threshold state.

---

## 12. Example Implementation Details

The invention may be implemented in synthesizable VHDL or another register-transfer-level hardware description language. The following implementation details are examples and do not limit the scope of the invention:

1. RGB input channels may be 10-bit channels.
2. Internal clustering may operate on 8-bit channel values.
3. Thirty centroid comparisons may be performed in parallel.
4. A no-square-root squared-distance mode may be used.
5. LUT banks may store 24-bit RGB triplets.
6. A configuration register may select between policies 0, 1, and 9.
7. The system may target a pixel clock of 148.5 MHz.
8. The system may support 1080p video at 60 frames per second.
9. Sideband signals may be delayed through a multi-stage shift register.
10. Output RGB values may be reconstructed by left-shifting 8-bit clustered values.

---

## 13. Advantages

The disclosed invention provides several technical advantages:

1. **Deterministic real-time operation:** The architecture avoids frame-stalling iterative software clustering and supports streaming classification.
2. **High throughput:** Parallel centroid-distance engines enable one-pixel-per-cycle processing after pipeline fill.
3. **Reduced computational burden:** Squared-distance comparison eliminates square-root operations.
4. **Dynamic palette control:** Runtime LUT update and selection allow adaptive color clustering.
5. **Stable archived palettes:** Pre-trained LUT banks provide deterministic fallback or fixed behavior.
6. **Efficient hardware mapping:** The design is suitable for FPGA implementation using LUTs, registers, BRAM, and DSP slices.
7. **Video-pipeline compatibility:** AXI-style stream and control-plane integration support deployment in SoC FPGA systems.
8. **Latency-safe output:** Sideband and RGB pipeline alignment reduces the risk of frame corruption.

---

## 14. Example Patent Claims

### Claim 1 - Independent System Claim

A hardware-implemented color clustering system comprising:

- a video input interface configured to receive a stream of RGB pixel values and video sideband signals;
- a look-up table management circuit comprising a plurality of RGB centroid look-up table banks;
- a policy selection circuit configured to select at least one of the plurality of RGB centroid look-up table banks based on a control value and at least one property of an incoming RGB pixel;
- a plurality of parallel clustering circuits, each clustering circuit configured to compute a distance value between the incoming RGB pixel and a corresponding RGB centroid value;
- a hierarchical reduction circuit configured to determine a minimum distance value from among the distance values generated by the plurality of parallel clustering circuits; and
- an output arbitration circuit configured to output a clustered RGB pixel corresponding to the RGB centroid associated with the minimum distance value;

wherein the video sideband signals are delayed through a synchronization pipeline to align the video sideband signals with the clustered RGB pixel.

### Claim 2 - LUT Bank Claim

The system of claim 1, wherein the plurality of RGB centroid look-up table banks includes at least one adaptive look-up table bank configured to receive runtime centroid updates and at least one static look-up table bank configured to store predetermined centroid values.

### Claim 3 - Control Register Claim

The system of claim 1, further comprising a register-mapped control interface configured to receive a configuration value that selects a clustering policy from among a plurality of clustering policies.

### Claim 4 - AXI Interface Claim

The system of claim 1, wherein the video input interface is compatible with an AXI4-Stream data plane and the register-mapped control interface is compatible with an AXI4-Lite control plane.

### Claim 5 - Policy-Based Selection Claim

The system of claim 1, wherein the policy selection circuit selects a look-up table bank based on at least one of a maximum RGB channel value, a minimum RGB channel value, a contrast value between the maximum RGB channel value and the minimum RGB channel value, a dominant RGB channel, or an intensity threshold.

### Claim 6 - Contrast Policy Claim

The system of claim 5, wherein a first clustering policy selects a look-up table bank based on whether a difference between the maximum RGB channel value and the minimum RGB channel value is greater than or equal to a contrast threshold.

### Claim 7 - Grayscale Detection Claim

The system of claim 6, wherein the first clustering policy detects a grayscale pixel when a difference between a red channel and a green channel is less than or equal to a first threshold and a difference between the red channel and a blue channel is less than or equal to a second threshold.

### Claim 8 - Intensity Policy Claim

The system of claim 5, wherein a second clustering policy selects a look-up table bank based on whether the maximum RGB channel value is greater than or equal to an intensity threshold.

### Claim 9 - Multi-Tier Intensity Claim

The system of claim 5, wherein a third clustering policy selects among at least three look-up table behaviors corresponding to high-intensity, medium-intensity, and low-intensity pixel ranges.

### Claim 10 - Parallel Clustering Claim

The system of claim 1, wherein the plurality of parallel clustering circuits comprises thirty clustering circuits configured to evaluate thirty RGB centroid values concurrently.

### Claim 11 - Squared Distance Claim

The system of claim 1, wherein each clustering circuit computes a squared distance value without performing a square-root operation.

### Claim 12 - Hierarchical Reduction Claim

The system of claim 1, wherein the hierarchical reduction circuit comprises a first reduction stage configured to reduce thirty distance values into six local minimum values; a second reduction stage configured to reduce the six local minimum values into two intermediate minimum values; and a third reduction stage configured to select the minimum distance value from the two intermediate minimum values.

### Claim 13 - Output Fallback Claim

The system of claim 1, wherein the output arbitration circuit is configured to output a diagnostic fallback RGB value when the minimum distance value does not match a delayed threshold record.

### Claim 14 - Bit-Depth Conversion Claim

The system of claim 1, further comprising a bit-depth conversion circuit configured to convert 10-bit RGB input channels into 8-bit internal channel values and to convert clustered 8-bit channel values into 10-bit output channel values.

### Claim 15 - Method Claim

A method for real-time FPGA-based RGB color clustering, comprising:

1. receiving an RGB pixel and associated video sideband signals;
2. selecting a centroid look-up table bank from among a plurality of centroid look-up table banks based on a clustering policy;
3. computing, in parallel, a plurality of distance values between the RGB pixel and a plurality of RGB centroid values;
4. selecting a minimum distance value through a hierarchical reduction tree;
5. identifying a centroid corresponding to the minimum distance value;
6. outputting a clustered RGB pixel corresponding to the identified centroid; and
7. delaying the video sideband signals to align with the clustered RGB pixel.

### Claim 16 - Runtime Update Claim

The method of claim 15, further comprising updating at least one centroid value in an adaptive look-up table bank through a register-mapped control interface while maintaining streaming operation of the RGB pixel flow.

### Claim 17 - Dominance Permutation Claim

The method of claim 15, further comprising permuting RGB channel order based on a dominant channel of the RGB pixel before selecting a look-up table behavior.

### Claim 18 - Video Processing Claim

The system of claim 1, wherein the system is configured to process high-definition video at one pixel per clock cycle after pipeline initialization.

---

## 15. Abstract

An FPGA-based dynamic LUT management system for real-time RGB color clustering is disclosed. The system receives streaming RGB pixel data and video sideband signals, selects one or more centroid look-up table banks according to configurable chromatic policies, computes parallel distance values between each pixel and multiple RGB centroids, reduces the distance values through a hierarchical minimum-selection tree, and outputs a clustered RGB pixel aligned with delayed video sideband signals. The architecture supports adaptive runtime LUT updates, archived pre-trained LUT banks, squared-distance computation without square-root operations, and deterministic one-pixel-per-cycle video processing suitable for embedded vision and high-definition FPGA video pipelines.

---

## 16. Suggested Patent Filing Classification Keywords

- FPGA video processing
- Real-time RGB clustering
- Color quantization hardware
- LUT-based centroid selection
- Parallel K-Means hardware accelerator
- AXI4-Stream image processing
- AXI4-Lite control interface
- Hardware color segmentation
- Squared-distance clustering
- Hierarchical minimum reduction
- Adaptive color palette FPGA
- Multi-palette video clustering

---

## 17. Recommended Next Step

This disclosure can be strengthened further by adding formal drawings, numbered element references, an expanded claim hierarchy, alternative embodiments, a VHDL implementation appendix, and a prior-art distinction table.
