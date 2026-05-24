# FPGA-Based Real-Time RGB K-Means Color Clustering Engine

[![FPGA](https://img.shields.io/badge/Platform-FPGA-blue)](#)
[![Language](https://img.shields.io/badge/RTL-VHDL-orange)](#)
[![Interface](https://img.shields.io/badge/Video-AXI4--Stream-green)](#)
[![Control](https://img.shields.io/badge/Control-AXI4--Lite-purple)](#)
[![Status](https://img.shields.io/badge/Status-Research%20%2F%20Technical%20Design-lightgrey)](#)

## Overview

This repository documents a **high-performance FPGA-based RGB K-Means Color Clustering Engine** for deterministic, real-time image and video processing.

The system converts the K-Means assignment stage into a deeply pipelined hardware datapath capable of classifying RGB pixels at video rate. Instead of performing iterative clustering in software, the FPGA receives a continuous pixel stream, compares each pixel against a centroid palette, selects the nearest centroid, and emits a clustered output pixel with aligned video metadata.

The architecture is designed for embedded machine vision, robotics, industrial inspection, AI preprocessing, and edge vision systems that require predictable latency and line-rate throughput.

---

## Key Features

- **Real-time RGB color clustering** for image and video streams
- **One pixel per clock cycle** after pipeline fill
- **Deterministic fixed-latency datapath** for real-time control loops
- **Parallel centroid comparison engines** for high-throughput classification
- **Squared Euclidean RGB distance** for nearest-centroid selection
- **Pipelined comparator reduction tree** for global minimum selection
- **Max/Mid/Min color abstraction** to reduce LUT and BRAM footprint
- **Runtime LUT and palette reconfiguration** through host control
- **Frame-safe shadow activation** to prevent mid-frame visual tearing
- **AXI4-Stream video pipeline compatibility**
- **AXI4-Lite style host-control integration**
- **Targeted support for AMD/Xilinx FPGA platforms such as Kria-class SOMs**

---

## System Purpose

A 1080p60 RGB video stream contains approximately **124.4 million pixels per second**. If every pixel is compared against a centroid codebook, the workload quickly becomes too expensive for software systems that require strict deterministic latency.

This engine addresses that bottleneck by moving pixel-level color assignment into FPGA logic. The FPGA performs the assignment stage of K-Means clustering directly in the streaming video path.

```text
Input Pixel: P(R, G, B)

For each centroid C[k]:
    D[k] = (R - C[k].R)^2 + (G - C[k].G)^2 + (B - C[k].B)^2

Winning_Index = argmin(D[0], D[1], ..., D[N-1])
Output Pixel   = Centroid_RGB[Winning_Index]
```

---

## High-Level Architecture

```text
┌────────────────────┐
│ RGB Pixel Input    │
│ AXI4-Stream Video  │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Input Capture      │
│ RGB Quantization   │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Max/Mid/Min        │
│ Channel Analysis   │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ LUT / Palette      │
│ Selection Layer    │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Parallel Distance  │
│ Engines            │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Comparator Tree    │
│ Min Reduction      │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Output Mapping     │
│ Clustered RGB Out  │
└────────────────────┘
```

---

## Pipeline Organization

| Stage | Function | Description |
|---|---|---|
| S0 | Input Capture | Receives RGB pixel and video sideband metadata |
| S1-S2 | Channel Analysis | Quantizes RGB values and identifies Max/Mid/Min hierarchy |
| S3 | LUT Selection | Selects active LUT bank or classification strategy |
| S4-S28 | LUT Pipeline | Aligns centroid and boundary data through pipeline stages |
| S29-S31 | Distance Computation | Computes parallel RGB distance values |
| S32-S34 | Minimum Reduction | Finds nearest centroid through comparator tree |
| S35 | Output Mapping | Emits clustered RGB value and aligned metadata |

---

## Distance Metric

The core design uses **squared Euclidean distance**:

```text
D² = (R - Cr)² + (G - Cg)² + (B - Cb)²
```

The square-root operation is not required because the square root is monotonic. Comparing squared distances produces the same nearest-centroid index as comparing full Euclidean distances, while avoiding expensive square-root hardware.

---

## Max/Mid/Min LUT Abstraction

A direct RGB lookup table for the full 8-bit RGB color cube would require storage for more than 16 million RGB combinations. This design reduces memory pressure by normalizing each pixel into an intensity hierarchy:

```text
Max = highest RGB channel value
Mid = middle RGB channel value
Min = lowest RGB channel value
```

The LUT stores generic intensity roles instead of fixed physical RGB channel positions:

| LUT Field | Meaning |
|---|---|
| `[23:16]` | Max intensity boundary |
| `[15:8]` | Mid intensity boundary |
| `[7:0]` | Min intensity boundary |

The hardware maps these generic values back to physical R, G, and B channels based on the incoming pixel’s channel ordering. This allows one normalized LUT representation to support multiple RGB permutations and significantly reduces memory footprint.

---

## Runtime Palette Management

The engine supports host-controlled runtime adaptation through LUT updates and palette selection.

### Runtime Update Flow

```text
Host Software / Python
        │
        ▼
TCP Payload / Control Message
        │
        ▼
lwIP Server on Processing System
        │
        ▼
AXI4-Lite Register Writes
        │
        ▼
FPGA Programmable Logic
        │
        ▼
Active / Shadow LUT Bank
```

### Frame-Safe Shadow Activation

To prevent inconsistent classification inside a single frame, palette updates can be staged in an inactive bank and activated only at a safe frame boundary, such as vertical blanking, EOF, or SOF transition.

---

## Performance Summary

| Metric | Target / Description |
|---|---|
| Throughput | 1 pixel per clock cycle after pipeline fill |
| Target Frequency | Up to 200 MHz class operation |
| Peak Pixel Rate | Up to 200 million pixels per second at 200 MHz |
| Latency | Approximately 25-35 clock cycles depending on configuration |
| Input Format | RGB video stream |
| Internal Precision | 8-bit RGB internal processing |
| Output Format | Clustered RGB output |
| Distance Engines | Parallel centroid comparison engines |
| Reduction Network | Pipelined comparator tree |
| Control Interface | AXI4-Lite style register control |
| Video Interface | AXI4-Stream style video datapath |

---

## Applications

### Industrial Inspection

- Color-based defect detection
- Surface classification
- Product sorting
- Inline quality control

### Robotics and Autonomous Systems

- Object segmentation
- Terrain detection
- Target tracking
- Low-latency perception preprocessing

### Embedded AI and Machine Vision

- Image simplification before neural-network inference
- Color quantization
- Segmentation preprocessing
- Edge vision acceleration

### UAVs and Edge Platforms

- Lightweight real-time vision
- Low-power color classification
- Reduced dependency on CPU/GPU frame buffering

---

## Repository Contents

This repository contains technical documents, feature blueprints, research material, and patent-style disclosures for the FPGA RGB K-Means color clustering system.

| File | Purpose |
|---|---|
| `README.md` | Main project overview |
| `FPGA_RGB_KMeans_Research_Paper.md` | Research-paper style explanation of the architecture |
| `Technical Specification FPGA-Based Real-Time RGB K-Means Color Clustering Engine.md` | Technical specification and performance summary |
| `Technical Design Specification High-Performance FPGA RGB K-Means Clustering Engine.md` | Detailed datapath and design specification |
| `FPGA Color Clustering Intelligence Engine.md` | System-level feature and value description |
| `FPGA LUT Management Module for Real-Time Color Clustering.md` | LUT management module description |
| `FPGA_LUT_Management_Patent_Disclosure.md` | Patent-style disclosure for LUT management |
| `FPGA_RGB_KMeans_LUT_Patent_Disclosure.md` | Patent-style disclosure for RGB K-Means LUT architecture |
| `System Integration Manual Host-Controller Communications for the K-Means Cluster Engine.md` | Host-controller integration workflow |
| `Feature Deck ... .md` files | Modular feature summaries for engine subsystems |
| `Formal Feature Blueprint ... .md` files | Structured subsystem blueprints |

---

## Suggested Implementation Flow

1. **Define centroid palette** for the target vision task.
2. **Serialize centroid RGB values** from host software.
3. **Write palette data** into FPGA-accessible LUT registers or memory banks.
4. **Stream video pixels** through the AXI4-Stream datapath.
5. **Compute nearest centroid** using parallel RGB distance engines.
6. **Reduce distance results** through a comparator tree.
7. **Map each pixel** to the selected centroid output color.
8. **Preserve video metadata** through matched pipeline delay registers.
9. **Activate new palettes** at frame-safe boundaries when reconfiguration is required.

---

## Verification Strategy

Recommended verification steps include:

- Compare RTL output against a Python or MATLAB golden reference model.
- Validate RGB quantization behavior.
- Verify squared-distance arithmetic for every centroid.
- Confirm tie-breaking behavior for equal distances.
- Check AXI4-Stream metadata alignment through the full pipeline.
- Validate LUT readback through host-control registers.
- Test palette updates at frame boundaries.
- Run directed tests for grayscale, high-contrast, and application-specific palettes.

---

## Future Work

- Hardware-assisted centroid learning
- Runtime-selectable centroid count
- Multi-pixel-per-clock datapath for 4K and higher frame rates
- Additional color spaces such as HSV, YCbCr, CIELAB, and normalized RGB
- Formal verification of AXI protocol behavior
- Post-synthesis resource and timing reports across FPGA families
- Complete VHDL source packaging with simulation testbenches

---

## Author

**Sakinder Ali**

Project domain: FPGA acceleration, real-time video processing, color clustering, embedded machine vision, and hardware-optimized AI preprocessing.

---

## License

No explicit license is currently defined in the repository. Add a license file before public reuse, redistribution, or commercial deployment.

Suggested options:

- MIT License for permissive open-source release
- Apache 2.0 for permissive release with patent grant language
- Proprietary license for controlled technical disclosure

---

## Citation

```text
S. Ali, FPGA-Based Real-Time RGB K-Means Color Clustering Engine,
GitHub repository, 2026.
```
