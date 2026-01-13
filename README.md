# Battery Pack Design Automation (Project #142)

[![View Battery Pack Design Automation on File Exchange](https://www.mathworks.com/matlabcentral/images/matlab-file-exchange.svg)](https://www.mathworks.com/matlabcentral/fileexchange/your-link-here)
[![Open in MATLAB Online](https://www.mathworks.com/images/responsive/global/open-in-matlab-online.svg)](https://matlab.mathworks.com/open/github/v1?repo=your-github-username/your-repo-name)

**Project ID:** 142  
**Author:** Harshit Shah  
**University:** Otto Von Guericke University, Magdeburg  

## 1. Project Overview

This repository contains a Model-Based Design (MBD) workflow for the automated design and optimization of a high-voltage battery pack for automotive applications. 

The project addresses the challenge of navigating the complex trade-off space between **Range**, **Mass**, **Cost**, and **Thermal Performance**. Moving beyond heuristic methods, this project leverages **MATLAB**, **Simulink**, and **Simscape Battery** to create a "Digital Twin" of the battery system.

### Key Distinction: Custom Architecture
Unlike standard solutions that rely on the pre-built *EV Reference Application*, this project features a **custom-developed vehicle architecture**. This approach allows for:
* Granular control over the powertrain energy integration.
* Flexible optimization of the *Cell-to-Pack* synthesis process.

## 2. Motivation
As the automotive industry shifts toward electrification, the battery pack remains the most critical system, dictating vehicle range, acceleration, and safety. 
Battery pack automation is a challenging problem because of the complexity of the system, downstream impacts on safety-critical design, and the need for rigorous multi-objective optimization.

## 3. Project Workflow
The solution follows a rigorous "V-Cycle" development process:
1.  **Data Extraction:** Processing raw NASA Randomized Battery Usage datasets.
2.  **Cell Characterization:** Identifying 3RC Equivalent Circuit Model (ECM) parameters.
3.  **Pack Synthesis:** Automating the assembly of modules using `Simscape Battery` builders.
4.  **System Integration:** Validating the pack within a custom Full Electric Vehicle (FEV) model.
5.  **Optimization:** Using `surrogateopt` to maximize range under mass/volume constraints.

---
---

## 4. Phase I: Data Acquisition & Preprocessing

**Objective:** Extract high-fidelity dynamic response data from the NASA Prognostics Center of Excellence (PCOE) "Randomized Battery Usage" dataset to parameterize the equivalent circuit model.

### Methodology
1.  **Data Selection:** Used Dataset **RW9**, specifically extracting the **Pulsed Load (Discharge)** steps. These steps provide the necessary excitation (impulse response) and relaxation (zero-state response) to identify cell impedance.
2.  **Signal Conditioning:**
    * Inverted current polarity to match Simscape convention (Discharge = Positive).
    * Upsampled data to **10 Hz** using PCHIP interpolation for consistent time-stepping.
    * Applied **Zero-Phase Filtering (`filtfilt`)** to remove sensor noise (RMS ~6.38 mV) without introducing phase lag.
3.  **Integrity Check:** Verifed voltage relaxation to ensure the battery reached equilibrium ($dV/dt \approx 0$).

### Results
The extracted pulse covers the full discharge phase with adequate rest time for OCV recovery.

![Pulse Extraction](Images/1_pulse_extraction.png)

*Figure 1: Extracted voltage and current profiles from NASA RW9 dataset (Fresh State).*

**Usage:**
Run the extraction script to generate the processed `.mat` file:
```matlab
% Run from /Scripts folder
run('Preprocessing')
