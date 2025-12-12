# HLS Acceleration & DSE for Neural Network Inference

**Authors:** Stavros Mitropoulos & Ioannis Danias  
**Course:** Embedded Systems Design (NTUA)  

## 1. Project Overview
This project focuses on the **Hardware/Software Co-design** and **acceleration** of a Neural Network (Generative Adversarial Network - GAN) on an FPGA using the Xilinx SDSoC environment.

**Note:** We did not design the Neural Network algorithm itself. Our work focused exclusively on **High-Level Synthesis (HLS) optimizations** and **Design Space Exploration (DSE)** to map the provided algorithm efficiently onto the FPGA hardware.

## 2. Optimization Strategy (HLS)
We transformed the initial C++ software implementation into a fully pipelined hardware accelerator. Our optimization strategy involved the following methods:

### Pipeline & Parallelism
* **Pipelining:** We applied `#pragma HLS PIPELINE II=1` to the inner loops of the network layers to achieve an Initiation Interval of 1, allowing the hardware to process one data element per clock cycle.
* **Dataflow:** We used `#pragma HLS DATAFLOW` to enable task-level parallelism, allowing different layers (functions) of the network to execute concurrently.

### Memory & Resource Management
* **Array Partitioning:** To match the throughput required by pipelining, we partitioned memory arrays (`xbuf`, `layer_1_out`, `layer_2_out`) using `#pragma HLS ARRAY_PARTITION`. This increased memory bandwidth by allowing simultaneous access to data elements.
* **DSP Allocation:** We strictly limited DSP usage to 80 (the maximum available on the Zybo board) using `#pragma HLS ALLOCATION`, distributing them across the layers to prevent resource overflow.

### Data Types
* **Fixed-Point Arithmetic:** We converted floating-point operations to fixed-point using `ap_fixed` types. This significantly reduced latency and resource usage compared to floating-point arithmetic. We explored various bit-widths (BITS) defined in `network.h`.

## 3. Design Space Exploration (DSE) Findings

We conducted extensive DSE to find the optimal trade-off between hardware performance (speedup) and output accuracy (PSNR).

### A. Performance Improvements
* **Initial Unoptimized Implementation:** achieved a speedup of **~2.16x** compared to software.
* **Final Optimized Implementation:** achieved a speedup of **~120.36x** compared to software.
* The design is fully pipelined. The highest latency was observed in the `read_input` and `layer_3` loops due to communication with the external DRAM (AXI DMA).

### B. Bit-Width Precision Analysis
We tested the accelerator with different quantization levels (4, 5, 6, 8, and 10 bits). The results showed a clear trade-off between resource usage and image quality:

| Bit Width | Max Pixel Error | PSNR (dB) | Visual Quality |
| :--- | :--- | :--- | :--- |
| **4 Bits** | 255 | 13.52 | Significant distortion; digits barely recognizable. |
| **5 Bits** | 229 | 19.41 | Poor quality. |
| **6 Bits** | 93 | 29.24 | Improved recognizability. |
| **8 Bits** | 13 | 47.07 | **Optimal Trade-off.** High fidelity with minimal error. |
| **10 Bits** | 4 | 53.77 | Near-identical to SW, but reduces speedup to ~85x. |

**Conclusion:** We selected **8 bits** as the "sweet spot," offering excellent accuracy (PSNR ~47dB) while maintaining the target speedup of ~120x.

## 4. Final Resource Utilization
For the optimized 8-bit configuration, the resource utilization on the FPGA was:

* **DSP:** 80/80 (100%) - The design utilizes all available DSP slices for multiplication.
* **BRAM:** 40/60 (66.67%).
* **LUT:** 7335/17600 (41.68%).
* **FF:** 7584/35200 (21.55%).

## 5. File Structure
* **`final.cpp`**: The hardware-accelerated C++ code containing the HLS pragmas (Partitioning, Allocation, Pipelining).
* **`main.cpp`**: The software testbench used to parse the dataset, run the hardware/software comparison, and calculate speedup.
* **`network.h`**: Header file defining the neural network dimensions and `ap_fixed` data types.
* **`LICENSE`**: MIT License covering the software distribution.
