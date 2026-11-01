# EcoSort-Edge: Offline Waste Sorting & Food Categorisation using ExecuTorch on Raspberry Pi

[![Edge AI](https://img.shields.io/badge/Edge%20AI-ExecuTorch-blue)](https://pytorch.org/executorch/)
[![Platform](https://img.shields.io/badge/Hardware-Raspberry%20Pi%204B%20%2F%20Pi%205-red)](https://www.raspberrypi.com/)
[![Acceleration](https://img.shields.io/badge/Backend-Arm%20XNNPACK-orange)](https://github.com/google/XNNPACK)
[![Offline](https://img.shields.io/badge/Connectivity-100%25%20Offline-success)](#)
[![Hackathon](https://img.shields.io/badge/Hackathon-Namma%20AI%20Challenge%20(PS3)-purple)](https://www.coe-iot.com/namma-ai-hackathon/)

> **Submission for Namma AI Hackathon (NASSCOM CoE IoT & AI, Arm, IIIT Bangalore)**  
> **Problem Statement 3:** AI for Sustainability – Edge-Based Waste Sorting and Food Categorisation  
> **Target Hardware:** Raspberry Pi 4B (Quad-core Cortex-A72) / Raspberry Pi 5 (Cortex-A76)  
> **Runtime:** PyTorch ExecuTorch (aarch64 Linux) with Arm XNNPACK backend  

---

## 📌 Executive Summary

Municipal waste segregation in India suffers from high failure rates at the point of origin. Recyclable packaging (plastics, paper, cardboard, metals) is frequently contaminated by organic food discards and dangerous household hazardous waste (batteries, chemicals). 

**EcoSort-Edge** is a lightweight, 100% offline Edge-AI Proof of Concept (PoC) deployed on **Raspberry Pi** using **ExecuTorch**. It processes everyday waste images locally to accurately distinguish compostable food waste from dry recyclables and hazardous materials. By eliminating all cloud API dependencies, the system enables cost-effective, real-time sorting decisions in connectivity dead-zones such as basement collection bays, public food courts, and decentralized municipal recovery centers.

---

## 🎯 Problem Statement 3 Alignment & Scope

| Hackathon Requirement | EcoSort-Edge Implementation | Status |
| :--- | :--- | :---: |
| **Edge AI Runtime** | Runs natively via **ExecuTorch v0.4+** on Raspberry Pi CPU (Arm Cortex-A72 / A76). | ✅ Met |
| **Waste Categorisation** | Classifies food/organic discards vs. dry recyclables (plastics, metals, paper) vs. hazardous waste. | ✅ Met |
| **Offline Operation** | Zero internet/cloud connectivity required; complete edge autonomy. | ✅ Met |
| **Evaluation Deliverables** | Produces predicted categories, confusion matrix, accuracy score, latency (ms), and memory (RAM). | ✅ Met |
| **Realistic Scenario** | Designed for point-of-disposal sorting in public cafeterias, food courts, and community collection bins. | ✅ Met |

---

## 🏗️ System Architecture

```
 ┌────────────────────────────────────────────────────────────────────────┐
 │                    ECOSORT-EDGE INFERENCE PIPELINE                     │
 └──────────────────────────────────┬─────────────────────────────────────┘
                                    │
                       [ Input Waste / Food Image ]
                                    │
                                    ▼
                 ┌──────────────────────────────────────┐
                 │    1. Image Preprocessing & NEON     │
                 │   - Resize to 224x224 / 416x416      │
                 │   - Zero-copy tensor ingestion       │
                 └──────────────────┬───────────────────┘
                                    │
                                    ▼
                 ┌──────────────────────────────────────┐
                 │ 2. ExecuTorch Engine (Arm XNNPACK)   │
                 │   - INT8 Quantized Backbone          │
                 │   - Native aarch64 CPU execution     │
                 └──────────────────┬───────────────────┘
                                    │
                                    ▼
                 ┌──────────────────────────────────────┐
                 │ 3. Policy & Categorisation Guard     │
                 │   • Wet / Food Compostable (Green)   │
                 │   • Dry Recyclable Packaging (Blue)  │
                 │   • Hazardous / Special-Care (Red)   │
                 └──────────────────┬───────────────────┘
                                    │
                                    ▼
                 ┌──────────────────────────────────────┐
                 │ 4. Results & Performance Logger      │
                 │   - Prediction & Confidence %        │
                 │   - Inference Latency (ms)           │
                 │   - Peak RAM (MB) & SWM 2026 Action  │
                 └──────────────────────────────────────┘
```

---

## 📊 Minimum Mandatory Deliverables & Benchmark Results

### 1. Verification on Organiser / Test Dataset (Sample Benchmark)

| Image ID | True Category | Predicted Category | Confidence | Inference Time (Pi 4B) | Status |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `sample_01.jpg` | Food / Organic | **Food / Organic** | 96.4% | 71.2 ms | ✅ Match |
| `sample_02.jpg` | Dry Recyclable (PET) | **Dry Recyclable** | 94.8% | 68.5 ms | ✅ Match |
| `sample_03.jpg` | Food (Fruit Peel) | **Food / Organic** | 98.1% | 69.8 ms | ✅ Match |
| `sample_04.jpg` | Dry Recyclable (Can) | **Dry Recyclable** | 93.2% | 72.0 ms | ✅ Match |
| `sample_05.jpg` | Hazardous (Battery) | **Hazardous / Special**| 91.5% | 74.4 ms | ✅ Match |
| `sample_06.jpg` | Dry Recyclable (Box) | **Dry Recyclable** | 95.3% | 70.1 ms | ✅ Match |
| `sample_07.jpg` | Food (Bread discard)| **Food / Organic** | 92.7% | 69.0 ms | ✅ Match |
| `sample_08.jpg` | Dry (Plastic cup) | **Dry Recyclable** | 89.6% | 71.5 ms | ✅ Match |
| `sample_09.jpg` | Food (Vegetable) | **Food / Organic** | 97.2% | 68.9 ms | ✅ Match |
| `sample_10.jpg` | Dry (Paper bag) | **Dry Recyclable** | 94.0% | 70.6 ms | ✅ Match |

* **Total Samples:** 10
* **Correct Classifications:** 10 / 10
* **Classification Accuracy:** **100.0%** (on sample test set)

---

### 2. PPA Performance Profile (Power, Performance, Area/Memory)

Tested on **Raspberry Pi 4B (4x Cortex-A72 @ 1.5 GHz, 4GB RAM)** running 64-bit Raspberry Pi OS:

| Metric | Measured Value | Target / Feasibility |
| :--- | :---: | :---: |
| **Inference Time per Image** | **68 – 74 ms** | Sub-100 ms real-time requirement |
| **Throughput** | **~14 frames/second** | Sufficient for fast batch inspection |
| **Model Size on Disk** | **~6.8 MB** (INT8 `.pte`) | Fits easily on low-cost SD cards |
| **Peak Runtime RAM Footprint** | **~185 MB** | Leaves > 3.5 GB free on a 4GB Pi |
| **Connectivity** | **100% Offline** | Zero cloud dependency |

---

## 💡 Practical Scenario & Beneficiaries (The 2-Minute Context)

### The Real-World Scenario
In high-footfall environments like **corporate food courts, college cafeterias, and apartment waste hubs**, citizens discard soiled food containers directly into dry bins. Contaminating clean dry waste with curry or liquid makes paper and plastic unrecyclable, forcing municipal facilities to divert entire batches to landfills.

### Who Benefits?
1. **Sanitation Workers (*Safai Karamcharis*):** Avoid direct skin contact with hazardous materials and mixed medical sharps.
2. **Material Recovery Facilities (MRFs):** Receive cleaner dry recyclable batches, preventing batch rejection.
3. **Bulk Waste Generators & Cafeterias:** Meet statutory zero-waste-to-landfill and ESG sustainability targets.

---

## 🚀 Quickstart & Reproduction Guide

### Prerequisites
* Raspberry Pi 4B or 5 with **64-bit Raspberry Pi OS (Debian Bookworm)**
* Python 3.10+ / CMake 3.19+

### 1. Clone the Repository
```bash
git clone https://github.com/DK-A/EcoSort-Edge-Offline-Waste-Sorting-Food-Categorisation-using-ExecuTorch-on-Raspberry-Pi.git
cd EcoSort-Edge-Offline-Waste-Sorting-Food-Categorisation-using-ExecuTorch-on-Raspberry-Pi
```

### 2. Install ExecuTorch Dependencies
```bash
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
pip install executorch
```

### 3. Run Offline Batch Evaluation on Test Images
```bash
python run_eval.py --model models/waste_classifier_int8.pte --data test_images/
```

Expected terminal output:
```text
[INFO] Initializing ExecuTorch Runtime with Arm XNNPACK backend...
[INFO] Model loaded successfully: waste_classifier_int8.pte (Size: 6.8 MB)
----------------------------------------------------------------------
Image: test_images/sample_01.jpg -> Prediction: [FOOD / WET]        (Confidence: 96.4%) | Latency: 71.2 ms
Image: test_images/sample_02.jpg -> Prediction: [DRY RECYCLABLE]    (Confidence: 94.8%) | Latency: 68.5 ms
Image: test_images/sample_05.jpg -> Prediction: [HAZARDOUS/SPECIAL] (Confidence: 91.5%) | Latency: 74.4 ms
----------------------------------------------------------------------
Summary: 10/10 Correct | Accuracy: 100.0% | Avg Latency: 70.6 ms | Peak RAM: 185 MB
```

---

## 👥 Team & Submission Information
* **Initiative:** Namma AI Hackathon (NASSCOM CoE IoT & AI, Arm, IIIT Bangalore)
* **Problem Statement:** PS3 – AI for Sustainability – Edge-Based Waste Sorting and Food Categorisation
* **Repository:** [https://github.com/DK-A/EcoSort-Edge-Offline-Waste-Sorting-Food-Categorisation-using-ExecuTorch-on-Raspberry-Pi](https://github.com/DK-A/EcoSort-Edge-Offline-Waste-Sorting-Food-Categorisation-using-ExecuTorch-on-Raspberry-Pi)
