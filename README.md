# Automated Design and Optimization of Electrochemical Energy Storage Systems for Automotive Powertrains (Project #142)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![MATLAB Compatibility](https://img.shields.io/badge/MATLAB-R2026a-blue.svg)](https://www.mathworks.com/products/matlab.html)
[![Status](https://img.shields.io/badge/Status-Ongoing-success.svg)]()

## Overview

This project, "Battery Pack Design Automation," leverages the advanced capabilities of MATLAB, Simulink, Powertrain Blockset, and Simscape Battery to automate the entire development lifecycle. By mathematically characterizing Lithium-Ion cells, programmatically synthesizing pack architectures, and optimizing these systems within a full vehicle simulation, engineers can navigate the complex trade-space of energy density, thermal management, and mass efficiency with unprecedented precision.

Phase I: Electrochemical Fundamentals and Equivalent Circuit Modeling

The Physics of Lithium-Ion Characterization

A lithium-ion battery is not an ideal voltage source; it is a system governed by the migration of ions between the cathode and anode through an electrolyte. The terminal voltage ($V_t$) observed at the battery tabs is the summation of several distinct potential contributions, each evolving on different time scales. The primary component is the Open Circuit Voltage (OCV).The OCV is strictly a function of the State of Charge (SOC) and Temperature ($T$). It represents the voltage of the battery at equilibrium, when no current is flowing and all internal relaxation processes have settled.

However, during operation, the terminal voltage deviates from the OCV due to internal impedances

- Ohmic Resistance ($R_0$) - This represents the instantaneous voltage drop caused by the ionic conductivity of the electrolyte and the electronic conductivity of the current collectors, tabs, and active material particles.
-  Polarization - Charge Transfer Kinetics: The resistance associated with the electrochemical reactions at the electrode-electrolyte interface. Diffusion: The movement of Lithium ions through the electrolyte and their intercalation into the porous electrode material. Diffusion processes are significantly slower than Ohmic or charge transfer effects, often taking minutes or hours to reach equilibrium.

 To model these dynamics without resorting to computationally expensive Partial Differential Equations (PDEs) used in electrochemical models engineers use Equivalent Circuit Models (ECMs) .

The 3RC Thevenin Equivalent Circuit

For this project, we utilize a 3RC Thevenin Model . This topology places a voltage source (OCV) in series with a resistor ($R_0$) and three parallel Resistor-Capacitor (RC) branches. Each RC branch captures a specific time constant ($\tau = R \cdot C$) of the battery's dynamic response.

- First RC Branch ($\tau_1$): Captures fast charge transfer and double-layer capacitance effects (typically seconds).
- Second RC Branch ($\tau_2$): Captures electrolyte diffusion dynamics (typically tens of seconds).
- Third RC Branch ($\tau_3$): Captures slow solid-state diffusion within the active material (typically hundreds of seconds).

Why use a 3RC model?

While a 1RC model is sufficient for basic SOC estimation, it fails to capture the "long tail" of voltage recovery after a load is removed. In a "Design Automation" context, where we simulate long, complex drive cycles like the FTP75, accurately predicting voltage recovery during coasting or idling is critical for estimating the available power for the next acceleration event. The 3RC model provides the necessary fidelity to predict these dynamics with an error margin often below 10 mV.

Phase II: Data Acquisition and Preprocessing

We utilize the NASA PCoE "Randomized Battery Usage" dataset, specifically datasets RW9, RW10, RW11, and RW12 . The NASA "Random Walk" (RW) datasets are ideal because they intersperse randomized current profiles with Pulsed Load Characterization steps. A pulsed load test applies a specific current for a short duration (eg, 10 minutes) and then allows the battery to rest (eg, 20 minutes).8This "Pulse-Rest" sequence is the "gold standard" for parameter estimation because the rest period isolates the pure diffusion dynamics, allowing the optimization algorithms to accurately fit the RC time constants ($\tau$).

# Table 1: NASA RW Dataset Characteristics

 | Characteristic | Description | Relevance to Modeling | 
 | :--- | :--- | :--- |
 | Battery Type | 18650 Li-ion (LG Chem typically) | Industry standard form factor. |
| Capacity | ~2.1 Ah | Baseline for scaling to pack level. |
| operation | Random Walk (RW) Current | Simulates real-world dynamic usage. |
| Characterization | Periodic Pulsed Load | Provides clean data for transient response fitting. |
| Measurements | Voltage, Current, Temperature, Time | Essential inputs for system identification. |


_ Matlab Coding _

Data Integrity Check

Before proceeding to estimation, the extracted data must pass a rigorous integrity check. Attempting to fit a model to corrupted data is the primary cause of convergence failure in optimization.

- Sample Rate Verification
- Relaxation Completeness
- Noise Floor

  
Phase III: Cell Characterization and Parameter Estimation

With clean pulse data secured, we move to System Identification . This is the process of finding the numerical values ​​for the equivalent circuit components ($E_m, R_0, R_1, C_1, R_2, C_2, R_3, C_3$) such that the model's output voltage matches the experimental voltage.  We utilize the Levenberg-Marquardt or Trust-Region-Reflective algorithms provided by MATLAB's Optimization Toolbox. 






