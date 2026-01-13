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
``` 

---

## 5. Phase II: Cell Characterization & Parameter Estimation

**Objective:** Identify the electrochemical parameters ($R_0, R_1, C_1, ...$) of a 3RC Thevenin Equivalent Circuit Model (ECM) that minimize the error between simulated and experimental voltage.

### Methodology
* **Model Topology:** 3RC (Three parallel Resistor-Capacitor branches in series with a resistor).
* **Optimization Algorithm:** Non-Linear Least Squares (`lsqnonlin`) using the **Trust-Region-Reflective** algorithm.
* **Cost Function:** Minimized the sum of squared errors between measured voltage ($V_{meas}$) and model voltage ($V_{model}$).

### Results
The optimization converged with an exit flag of 3 (change in residual < tolerance), achieving a high-fidelity fit.

* **Final RMSE:** **4.34 mV** (0.1% error relative to nominal voltage).

**Estimated Parameters:**

| Component | Resistance ($\Omega$) | Capacitance (F) | Time Constant $\tau$ (s) | Dynamics Captured |
| :--- | :--- | :--- | :--- | :--- |
| **Series** ($R_0$) | 0.050 | N/A | Instant | Ohmic Drop |
| **Branch 1** | 0.001 | 100.0 | **0.10 s** | Double Layer / Charge Transfer |
| **Branch 2** | 0.0078 | 557.5 | **4.33 s** | Solid Electrolyte Interphase (SEI) |
| **Branch 3** | 0.120 | 2000.0 | **239.5 s** | Diffusion (Slow) |

![Model Validation](Images/2_model_fit.png)
*Figure 2: Validation of the 3RC model against experimental pulse data. The model (red dashed) captures both the instantaneous drop and the slow diffusion tail.*

**Usage:**
Run the estimation script to generate the parameters file:
```matlab
% Run from /Scripts folder
run('Cell_Characterization_and_Parameter_Estimation')

```

## 6. Phase III: Automated Pack Synthesis

**Objective:** Programmatically generate a detailed Simscape Battery model from the estimated cell parameters, scaling up to a full electric vehicle pack.

### Methodology
Using the **Simscape Battery Builder API**, a "Bottom-Up" architecture was defined in MATLAB code to automatically generate the Simulink block. This approach avoids manual block placement and ensures consistent parameter propagation.

* **Cell Format:** 21700 Cylindrical (NMC Chemistry).
* **Thermal Strategy:** Cell-based thermal resistance (Passive/Liquid cooling ready).
* **Architecture:** **90S 38P** (90 Series, 38 Parallel).

### Pack Specifications

| Attribute | Value | Derivation |
| :--- | :--- | :--- |
| **Configuration** | **90s 38p** | 38 Parallel Cells $\times$ 90 Series Groups |
| **Nominal Voltage** | **333 V** | $90 \times 3.7 \text{ V}$ |
| **Total Capacity** | **182.4 Ah** | $38 \times 4.8 \text{ Ah}$ |
| **Total Energy** | **60.7 kWh** | $333 \text{ V} \times 182.4 \text{ Ah}$ |
| **Estimated Mass** | **232 kg** | Cells only ($90 \times 38 \times 0.068 \text{ kg}$) |

![Pack Library Block](Images/3_pack_library.png)
*Figure 3: The automatically generated battery pack block in Simulink, ready for vehicle integration.*

**Usage:**
Run the builder script to generate the `.slx` library file:
```matlab
% Run from /Scripts folder
run('The_Battery_Builder_API')
```


