# Macroalgae Primary Production 

This is a description of **p4zmacroprod.F90**.
This module simulates the **net primary production of macroalgae**, distinguishing between two functional types:

- **Temperate Brown Macroalgae**
- **Tropical Red Macroalgae**

Light, temperature, nutrient, and iron limitations are considered to compute the **net primary production (NPP)**.

## Model Structure

For each 3D ocean grid cell where there is a macroalgae initial seed (`macrogebco > 0`), the following processes are computed:

---

## Growth Limitations

### 1. Temperature Dependence

The temperature growth limitation is given by a Gaussian function, following Wu et al (2023):

$$
f_T = \exp\left(-2.3 \left(\frac{T - T_{\text{opt}}}{T_x - T_{\text{opt}}}\right)^2 \right)
$$

Where:

- \( T \): in situ temperature
- \( T_{\text{opt}} \): optimal temperature
- \( T_x \): limiting temperature (either `xtmax` or `xtmin`)



### 2. Light Limitation

Light limitation uses a peaked function following Wu et al (2023):

$$
f_{\text{PAR}} = \left( \frac{E}{E_{\text{opt}}} \right) \cdot \exp\left(1 - \frac{E}{E_{\text{opt}}} \right)
$$

Where:
- \( E \): total PAR at depth
- \( E_{\text{opt}} \): optimal light for macroalgae

### 3. Macro nutrient Limitation (Michaelis-Menten)

Nitrogen (NO₃) and phosphorus (PO₄) limitations are computed with a Michaelis-Menten function:

$$
L_{NO_3} = \frac{[NO_3]}{K_{NO_3} + [NO_3]} \quad ; \quad L_{PO_4} = \frac{[PO_4]}{K_{PO_4} + [PO_4]}
$$

Combined as:

$$
f_{\text{nutrient}} = \min(L_{NO_3}, L_{PO_4})
$$

### 4. Iron Limitation

Iron limitation is computed with a Michaelis-Menten function, based on Paine et al. (2023):

$$
f_{Fe} = f_{Fe}^{\text{max}} \cdot \frac{[Fe]}{K_{Fe} + [Fe]}
$$

Then:

$$
f_{\text{nut}} = \min(f_{\text{nutrient}}, f_{Fe})
$$

---

## Gross Production

Gross primary production:

$$
P = \mu_{\text{max}} \cdot f_T \cdot f_{\text{nut}} \cdot f_{\text{PAR}} \cdot \Delta t
$$

Then corrected for **sea ice cover**:

$$
P = P \cdot (1 - \text{frac}_{\text{ice}})
$$

---

## Respiration

Respiration is temperature-dependent:

$$
R = \mu_{\text{resp}} \cdot \text{coef}_{\text{resp}}^{T - 20} \cdot P
$$

---

## Net Primary Production

Finally, NPP is computed as:

$$
NPP = (P - R) \cdot \text{B}
$$

$$
\frac{dB}{dt} = \text{NPP} 
$$

---

## Nutrient Update

The uptake of nutrients (PO₄, NO₃, DIC, Fe) is proportional to the NPP and stoichiometric ratios.

$$
\frac{dDIC}{dt} = - NPP \cdot mask_{3D}
$$

$$
\frac{dNO_3^{-}}{dt} = - NPP \cdot mask_{3D} \cdot \frac{Q_{C:N}^{PHY}}{Q_{C:N}^{MAC}} 
$$

$$
\frac{dPO_4^{3-}}{dt} = - NPP \cdot mask_{3D} \cdot \frac{Q_{C:P}^{PHY}}{Q_{C:P}^{MAC}}
$$

$$
\frac{dDFe}{dt} = - NPP \cdot mask_{3D} \cdot \frac{1}{Q_{C:Fe}^{MAC}}
$$

$$
\frac{dTA}{dt} = + r_{NO_3} \cdot NPP \cdot mask_{3D} \cdot \frac{Q_{C:N}^{PHY}}{Q_{C:N}^{MAC}} 
$$
---

## Variable Mapping

| **Variable in Fortran**       | **Description**                    | **Equation Symbol**             |
|-------------------------------|------------------------------------|---------------------------------|
| `ts`                          | Temperature                        | \( T \)                         |
| `etot`                        | Light at depth                     | \( E \)                         |
| `tr(...,jpno3,...)`           | NO₃ concentration                  | \( [NO_3] \)                    |
| `tr(...,jppo4,...)`           | PO₄ concentration                  | \( [PO_4] \)                    |
| `tr(...,jpfer,...)`           | Iron concentration                 | \( [Fe] \)                      |
| `fr_i`                        | Sea ice fraction                   | \( \text{frac}_{\text{ice}} \)  |
| `trbtropred`, `trbtempbrown`  | Biomass array                      | \( B \)                         |
| `macrogebco(:,:,:)`           | Bathymetry mask (% cell area)      | \( \text{mask}_{3D}(:,:,:) \)   |
| `xstep`                     | Time step                        | \( \Delta t \)           |



---

## Input Parameters (from `nampismacprod` namelist)

| **Variable in Fortran**     | **Description**                     | **Equation Symbol**             |
|-----------------------------|-------------------------------------|---------------------------------|
| `toptmacb`, `toptmacr`      | Optimal temperature                 | \( T_{\text{opt}} \)            |
| `xtmaxmacb`, `xtminmacb`    | Temperature limits (brown)         | \( T_x \)                       |
| `xparoptmacb`, `xparoptmacr`| Optimal light                      | \( E_{\text{opt}} \)            |
| `concmno3b`, `concmno3r`    | NO₃ half-sat. constants             | \( K_{NO_3} \)                  |
| `concmpo4b`, `concmpo4r`    | PO₄ half-sat. constants             | \( K_{PO_4} \)                  |
| `concmfe`                   | Fe half-sat. constant               | \( K_{Fe} \)                    |
| `xfegmacmax`                | Fe limitation max factor            | \( f_{Fe}^{\text{max}} \)       |
| `xgmacmaxb`, `xgmacmaxr`    | Max growth rate                    | \( \mu_{\text{max}} \)          |
| `xrpmacmax`                 | Max respiration rate               | \( \mu_{\text{resp}} \)         |
| `xrespmaccoef`              | Respiration temp. coefficient      | \( \text{coef}_{\text{resp}} \) |

---

## References

- Wu et al., 2023. 
- Paine et al., 2023. 
---

## Output Variables

These variables are output using `iom_put`:

| **Output**     | **Description**                           |
|----------------|-------------------------------------------|
| `MACPRODB`     | Gross production (Brown)                  |
| `MACPRODR`     | Gross production (Red)                    |
| `MACNPPB`      | Net primary production (Brown)            |
| `MACNPPR`      | Net primary production (Red)              |
| `MACNPP`       | Total net primary production              |
| `MACRESP`      | Respiration (all macroalgae)              |
