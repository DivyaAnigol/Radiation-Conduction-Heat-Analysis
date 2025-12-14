# Radiation-Conduction-Heat-Analysis
A MATLAB implementation focused on designing and analysing heater configurations for thermal processing. The code integrates geometric modelling, angle-weighted radiative flux computation, and conduction-based temperature evolution to evaluate thermal uniformity within a chamber environment.

## Technical Content
This application is developed in the context of Selective Laser Sintering (SLS) 3D printing, where accurate control of heat flux and temperature distribution on the powder bed is critical for achieving uniform sintering during subsequent laser exposure. The model focuses on understanding preheating behavior and thermal uniformity of the powder bed, which directly influence laser-material interaction, part quality, and process stability.

**AIM:** Analysis of radiative and conductive heat transfer from the IR heaters on the bed in a defined system.

**TOOL USED:** MATLAB

**CODE & LOGIC:**

**Overview:** This script models radiative and conductive heat transfer inside a closed thermal processing chamber equipped with cylindrical infrared (IR) heaters. The simulation computes angle-weighted radiative heat flux on a powder bed, converts absorbed energy into temperature rise, and applies 2D heat conduction to predict final temperature distribution.

**1. Material Properties - Radiative Physics:** The chamber is defined as a rectangular enclosure using polygonal faces with assigned material properties. PA12 powder is modeled with higher emissivity to represent radiative absorption, while polished stainless steel walls are modeled with high reflectivity to account for radiation confinement.
```
%% Material Properties
PA12 = struct('name', 'PA12 Powder', ...
			  'color', [0.6 0.6 0.6], ...
			  'reflectivity', 0.05, ...
			  'emissivity', 0.46);
SS = struct('name', 'Polished Stainless Steel', ...
			'color', [0.8 0.8 0.8], ...
			'reflectivity', 0.85, ...
			'emissivity', 0.15);
```
**Physics relevance:**
Controls radiation absorption, reflection, and emission behavior (Kirchhoff’s law).


**2. Infrared Heater Modeling:** Eight cylindrical IR heaters are positioned near the chamber roof. Each heater is modeled as a rotatable 3D cylinder with partial ceramic coating to represent directional radiation. Axis–angle rotation matrices ensure correct spatial alignment of heaters along arbitrary orientations.
```
% Assume emission is directed downward
	emission_dir = [0 0 -1];
```
**Physics relevance:**
Models anisotropic radiation instead of non-physical isotropic emission.
```
% cos(θ) with vertical emission %This is calculating cos(θ) — the cosine of the angle between the ray and the vertical direction.
	dot_prod = -(vec_z ./ dist3);  
	dot_prod (dot_prod < 0) = 0;
```
**Physics relevance:**
Implements Lambert’s cosine law → angular attenuation of radiative intensity.


**3. Heater Power Control:** Heater power is controlled using pulse-width modulation (PWM) fractions scaled by a maximum rated power. This allows simulation of asymmetric or zone-based heating strategies commonly used in industrial thermal systems.
```
%% Heater PWM and Power Assignment (using decimals from 0 to 1)
PWM_frac = [0.1, 0.5, 0.1, 0.5, 0.1, 0.5, 0.1, 0.5];  % Each between 0 and 1
max_power = 120;  % Max power per heater in Watts
heater_power = PWM_frac * max_power;
```
**Physics relevance:**
This segment converts PWM duty cycle into average thermal power, physically modeling how IR heater radiation intensity is controlled by electrical power modulation in real heating systems.


**4. Radiative Heat Flux Calculation:** Radiative heat transfer from heaters to the powder bed is calculated using a cosine-weighted emission model. An approximate view-factor formulation accounts for angular attenuation and inverse-square distance effects. Heat flux contributions from all heaters are superimposed spatially.
```
% --- Approximate View Factor ---
	VF =  dot_prod ./ (pi * dist3.^2 + 1e-6);  % Add small value to avoid div/0
```
**Physics relevance:**
Captures inverse-square spreading + orientation dependence of radiation.
```
% Updated flux with view factor
	flux = heater_power(i) * VF;
	heat_flux = heat_flux + flux;
```
**Physics relevance:**
Linear superposition of radiative contributions from multiple heaters.
```
% 1. Radiative absorption
absorbed_flux = heat_flux * epsilon;  
Physics relevance:
Applies emissivity → not all incident radiation is absorbed.

% 2. Radiative loss (using T0 as base temp for loss)
T0_K = T0 + 273.15;
q_loss = epsilon * sigma * (T0_K^4 - Tamb^4);  % W/mm²
total_loss = q_loss * time;  % J/mm² lost over time
```
**Physics relevance:**
Accounts for T⁴ radiation losses → prevents unrealistic overheating.


**5. Energy Balance and Temperature Rise:** Absorbed radiative energy is calculated using powder emissivity. Radiative losses to the ambient environment are computed using the Stefan–Boltzmann law. Net absorbed energy is converted into temperature rise using material density, specific heat, and absorption depth.
```
% 3. Net energy gain per mm² (J/mm²)
energy_input = absorbed_flux * time;   % J/mm²
net_energy = energy_input - total_loss;

% 4. Compute temp rise
dT_map = net_energy / (rho * depth * c);
dT_map(dT_map < 0) = 0;  % Clamp negatives

% 5. Final absolute temperature
T_abs = T0 + dT_map;

% 6. Cap temperature to PA12 degradation limit
T_abs(T_abs > 180) = 180;
```
**Physics relevance:**
Direct application of energy conservation to compute ΔT. Enforces PA12 degradation/melting limit → real process constraint.


**6. Heat Conduction Simulation:** A 2D explicit finite-difference scheme models lateral heat conduction within the powder bed. Thermal diffusivity governs heat spreading over time, reducing localized hotspots generated by radiative heating.
```
%% --- 2D Heat Conduction Simulation ---
T_conduction = T_abs;
dx = 10; dt = 0.5; time_total = 30; steps = round(time_total / dt);
k = 0.00022; % W/mm-K
alpha = k / (rho * c); r = alpha * dt / dx^2;
for t = 1:steps
	T_next = T_conduction;
	for i = 2:size(T_conduction,1)-1
		for j = 2:size(T_conduction,2)-1
			T_next(i,j) = T_conduction(i,j) + r * ( ...
				T_conduction(i+1,j) + T_conduction(i-1,j) + ...
				T_conduction(i,j+1) + T_conduction(i,j-1) - ...
				4 * T_conduction(i,j));
		end
	end
	T_conduction = T_next;
end
```
**Physics relevance:**
Explicit finite-difference solution of the 2D heat equation.


**7. Results and Visualization:** The script generates heat flux maps, temperature contour plots, and annotated hotspot locations. These outputs enable evaluation of heating uniformity, peak temperatures, and thermal gradients relevant to powder-based thermal processes.

**A. 3D schematic of an SLS processing chamber showing the spatial arrangement of cylindrical IR heaters around the powder bed for controlled thermal preheating.**

<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/dbb64b5f-7e70-425e-bc01-7bdf7cfb0286" />

**B. Normalized heat flux distribution on the powder bed showing maximum radiative intensity at the center and gradual decay toward the edges, indicating controlled thermal uniformity for SLS preheating.**

<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/8950f979-3b25-4855-89ee-b2b86ace24d2" />

**C. Final temperature distribution on the powder bed after radiative heating and conductive spreading, highlighting a central hotspot and smooth thermal gradients across the build area.**

<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/b1b2036e-1414-4b5f-8b05-2f592216c237" />

**D. Temperature contour map after conduction showing radially symmetric heat diffusion from the center, illustrating smooth thermal gradients and stabilized powder bed heating.**

<img width="500" height="500" alt="image" src="https://github.com/user-attachments/assets/86f9ec35-d326-400e-91fb-879af0f14b8f" />





