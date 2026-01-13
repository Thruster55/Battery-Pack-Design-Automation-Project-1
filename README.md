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

## 7. Phase IV: Unit Testing & Thermal Verification

**Objective:** Isolate the generated battery pack block to verify electrical connectivity and thermal behavior before full vehicle integration.

### Test Setup
* **Test Bench:** A simplified Simulink model connecting the `MyAutomatedPackLib` block to a **Constant Current Load**.
* **Conditions:** Constant Discharge current to stress the thermal path.
* **Pass Criteria:**
    1.  **Voltage Response:** Must decrease monotonically (no discontinuities) with instantaneous ohmic drops.
    2.  **Thermal Response:** Temperature at the thermal port ($H$) must rise, confirming the thermal circuit is active (not floating).

### Results
The unit test confirmed valid physical responses. As shown in the results, the voltage follows the expected discharge curve defined by the OCV-SOC relationship, while the cell temperature rises due to Joule heating ($I^2 R$), validating the thermal interface.

![Unit Test Results](Images/4_unit_test.png)
*Figure 4: Unit test showing Voltage Sag (Top) and Temperature Rise (Bottom) under constant load.*

---

## 8. Phase V: System Integration & Drive Cycle Validation

**Objective:** Validate the battery pack's performance within a **Custom Electric Vehicle (EV) Architecture** under realistic driving conditions.

### Vehicle Architecture
Unlike standard reference applications, this project utilizes a **custom-developed vehicle model** to allow granular control over energy flows:
* **Glider Model:** Physics-based longitudinal dynamics (Drag, Rolling Resistance, Inertia).
* **Powertrain:** Efficiency-map based Motor/Inverter integration.
* **Driver:** PID-based Longitudinal Driver configured to track regulatory drive cycles.
* **Power Source:** The `MyAutomatedPackLib` block acts as the primary DC link.

### Validation Scenario: FTP-75
The system was simulated using the **FTP-75 (Federal Test Procedure)**, a rigorous city driving cycle characterized by frequent stops and transient accelerations.

**Validation Results:**
* **Cycle Completion:** The vehicle successfully completed the full drive cycle without depleting the battery or hitting thermal limits.
* **Distance Covered:** **17.77 km** (Matches standard FTP-75 length).
* **Speed Tracking:** The driver model successfully maintained the reference speed (Yellow vs. Blue trace), proving the battery pack delivered sufficient power ($P = V \times I$) for all acceleration events.

![Drive Cycle Validation](Images/5_drive_cycle_results.png)
*Figure 5: Drive Cycle Validation. Top: Speed tracking (Reference vs Actual). Bottom: Battery Voltage and Current response confirming dynamic load handling.*

**Usage:**
To run the validation model:
1. Ensure `MyAutomatedPackLib` is generated (Phase III).
2. Open and run the validation model:
```matlab
open('Drive_Cycle_Val')
sim('Drive_Cycle_Val')
```
## 9. Phase VI: Multi-Objective Design Optimization

**Objective:** Determine the optimal battery pack sizing ($N_s, N_p$) to maximize vehicle range while strictly adhering to mass, volume, and component voltage limits.

### Methodology: Grid Search
Since the design space is discrete (you cannot have 0.5 cells), a **Grid Search** algorithm was implemented to evaluate every valid combination of Series ($N_s$) and Parallel ($N_p$) cells.

* **Design Variables:**
    * $N_s \in [90, 108]$ (Voltage scaling)
    * $N_p \in [35, 55]$ (Capacity scaling)
* **Constraints:**
    * $\text{Mass} < 500 \text{ kg}$ (Vehicle weight limit)
    * $\text{Voltage} < 450 \text{ V}$ (Inverter MOSFET breakdown limit)
* **Physics Model:**
    The optimizer uses a physics-based scaling law. Range is calculated relative to a baseline simulation, penalized by the added mass of larger packs:


### Results
The algorithm evaluated 210 unique design points. The surface plot below visualizes the trade-off. The red circle indicates the global maximum that satisfies all constraints.

| Parameter | Optimal Value | Constraint Check |
| :--- | :--- | :--- |
| **Configuration** | **90S 38P** | Selected Design |
| **Pack Voltage** | **333 V** | Pass (< 450 V) |
| **Pack Mass** | **302 kg** | Pass (< 500 kg) |
| **Predicted Range** | **300 km** | Maximized |

![Optimization Surface](Images/6_optimization_surface.png)
*Figure 6: Design Space Visualization. The z-axis represents Range. The optimization surface is cut off where constraints (Mass/Voltage) are violated.*

**Usage:**
Run the optimization script to visualize the trade-off space:
```matlab
% Run from /Scripts folder
run('Phase6_Optimization')
```

## 10. Conclusion & Future Work

### Project Summary
This project successfully demonstrates the efficacy of **Design Automation** in modern battery engineering. By implementing a rigorous "V-Cycle" development process, the workflow replaced traditional heuristic methods with a data-driven, mathematically optimized approach. The final deliverable—a **333V, 182Ah Battery Pack** validated against the **FTP-75** drive cycle—proves that automated synthesis can accelerate the development of high-voltage energy storage systems.

### Tools & Technologies Utilized
The project leveraged the full power of the MATLAB & Simulink ecosystem:
* **MATLAB:** For data processing, signal conditioning (`filtfilt`), and script-based automation.
* **Optimization Toolbox:** Used `lsqnonlin` (Trust-Region-Reflective) for high-fidelity 3RC parameter estimation.
* **Simscape Battery:** Employed the `BatteryBuilder` API to programmatically synthesize the pack architecture, thermal paths, and electrical connections.
* **Powertrain Blockset:** Provided the validated vehicle glider, motor, and driver models for system-level integration.
* **Global Optimization Toolbox:** Enabled the `surrogateopt` / Grid Search strategy to solve the multi-objective sizing problem.

### Applications
This automated workflow is directly applicable to:
1.  **Rapid Prototyping:** Automotive OEMs can instantly generate valid simulation models for varying voltage/capacity requirements without manual block connectivity.
2.  **Sizing Studies:** System engineers can perform "What-If" analyses (e.g., "How does changing cell mass by 10% affect vehicle range?") in seconds.
3.  **BMS Development:** The high-fidelity 3RC plant model serves as a perfect "Digital Twin" for testing State-of-Charge (SOC) and State-of-Health (SOH) algorithms.

### Advanced Extension: Solid-State Batteries (SSB)
To extend this work to **Solid-State Batteries** (a key frontier in energy storage), the fundamental workflow remains valid, but specific physical domains require adaptation:
* **Phase II (Physics):** SSBs exhibit distinct diffusion dynamics due to the solid electrolyte interface.The 3RC model should be replaced with a **Warburg Impedance** element or a **Fractional-Order Model** to capture these non-linearities.
* **Phase VI (Constraints):** Optimization constraints must evolve. SSBs often operate efficiently at higher temperatures but require strict **Mechanical Pressure** constraints to maintain stack integrity. The optimization cost function would need to penalize volume expansion (swelling) alongside mass and cost.

By mastering this workflow, this project establishes a scalable framework for the future of sustainable transportation: combining **Electrification**, **Simulation**, and **Optimization** into a unified design tool.
