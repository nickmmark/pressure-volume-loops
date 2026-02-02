# pressure-volume-loops

A simple open-source web-based simulator visualizes left ventricular pressure–volume (PV) loops under normal physiology, systolic heart failure, and several forms of mechanical circulatory support including:
* [Intra-aortic balloon pump](https://onepagericu.com/iabp) (IABP)
* [Percutaneous Microaxial flow pump](https://onepagericu.com/impella) (e.g. Impella)
* [Veno-arterial ECMO](https://onepagericu.com/ecmo-fundamentals) (VA-ECMO)

Try it out [here](https://nickmmark.github.io/pressure-volume-loops/).

![](https://github.com/nickmmark/pressure-volume-loops/blob/main/hemodynamic_simulator_demo.gif)

## Description of PV Loops
### Factors that effect the Pressure Volume Loop
| Factor    | Effect on Loop                                                                                        |
|-----------|-------------------------------------------------------------------------------------------------------|
| Preload   | Increases the width of the loop ($V_{ed}$ moves right).                                               |
| Afterload | The loop becomes taller and narrower; the top-left corner follows the ESPVR line.                     |
| Inotropy  | Increases the slope of the ESPVR, allowing for a smaller $V_{es}$ and larger SV at the same pressure. |


#### ESPVR (End-Systolic Pressure-Volume Relationship)
This line at the top of the plot represents the maximum pressure the ventricle can develop at any given volume. It is a measure of contractility (Inotropy). It is often modeled linearly as:
$$P_{es} = E_{es} \cdot (V_{es} - V_0)$$
* $E_{es}$: The slope, known as end-systolic elastance. A steeper slope indicates higher contractility.
* $V_0$: The theoretical volume at zero pressure.
The app allows the user to adjust both $E_{es}$ and $V_0$

#### EDPVR (End-Diastolic Pressure-Volume Relationship)
This curve at the bottom of the plot describes the passive filling properties (stiffness/compliance) of the ventricle. Unlike the ESPVR, it is non-linear and is modeled exponentially:$$P_{ed} = A \cdot e^{\beta V_{ed}}$$
* $\beta$: Represents the stiffness constant of the ventricular wall.

#### Stroke Volume
Stroke Volume (SV): $V_{ed} - V_{es}$ (The width of the loop).

#### Ejection Fraction
Ejection Fraction (EF): $\frac{SV}{V_{ed}} \times 100$.

#### Stroke Work
Stroke Work (SW): $\oint P \, dV$ (The area inside the loop).


## Calculations


### IABP Calculations
* Pressure Reduction: The model simulates the "inflation/deflation" cycle by reducing the effective $MAP$ during systole (systolic unloading).Formula Change:
  $MAP_{effective} = MAP_{baseline} - \Delta P_{iabp}$.
* Loop Geometry: The loop remains largely rectangular. However, Point B (Aortic Opening) occurs at a lower pressure, and Point C (Aortic Closure) occurs at a lower volume, slightly increasing the $SV$ and shifting the loop slightly to the left.
* Isovolumetric Phases: Unlike the Impella, these remain vertical because the IABP does not remove blood from the ventricle; it only reduces the resistance against which the heart must pump.

### Impella Calculations
Calculating the shape of the Impella augmented PV curves turns out to be quite tricky...

The application uses a dynamic simulation approach, treating the heart as a variable pressure-volume reservoir coupled with a continuous-flow pump. Unlike static geometric models, this version solves a system of differential equations over a discretized cardiac cycle, $T$, where $dt = 2\text{ms}$.

#### Ventricular Elastance $E(t)$ 
The "squeeze" of the heart is modeled as a normalized activation function:
``` math
E(t) = (E_{es} - E_{min}) \cdot \sin\left(\frac{\pi t}{T_s}\right)^n + E_{min}
```
* $n = 8$: The shape factor. A higher $n$ creates the sharp peak required for point convergence.
* $E_{es}$: End-Systolic Elastance (Inotropy).
* $T_s$: Duration of systole (~0.3s).

#### Instantaneous Pressure $P(t)$
Ventricular pressure is the sum of the active contraction (governed by $E(t)$) and the passive stretching of the myocardium (EDPVR).
``` math
P(t) = \max \left( E(t) \cdot (V(t) - V_0), \ P_{off} + A \cdot e^{\beta \cdot V(t)} \right)
```

#### Volume Continuity & Continuous Drain
The change in volume $V$ is the sum of native ejection ($Q_{out}$), venous return ($Q_{in}$), and the continuous suction from the Impella pump ($Q_{impella}$).
``` math
Q_{\text{out}} =
\begin{cases}
\dfrac{P(t) - \text{MAP}}{R_{\text{valve}}}, & P(t) > \text{MAP} \\
0, & P(t) \le \text{MAP}
\end{cases}
```
At high support levels (P7-P9), the pump removes volume so rapidly that $P(t)$ exceeds $MAP$ for a negligible duration, forcing $V_B \approx V_C$.

This is implemented as:
```javascript
function calculateConvergence(config, pLevel) {
    const flowDrain = (1.5 + (pLevel * 0.65)) * 1000 / 60; // mL/sec
    const map = config.map - (pLevel * 0.8); // Afterload reduction
    
    // Ts is the systolic duration. 
    // High support reduces the 'Effective' ejection time (dt_eject)
    const dt_eject = Math.max(0.01, 0.2 - (pLevel * 0.02)); 
    
    // Point B: Volume at Aortic Valve Opening
    // Volume is lost during the slanted isovolumetric contraction phase
    const vB = config.edv - (flowDrain * 0.05); // 0.05s is contraction time
    
    // Point C: Volume at Aortic Valve Closure
    // Convergence occurs because vC = vB - (NativeSV + PumpDrainSV)
    // At high P-levels, NativeSV approaches 0.
    const nativeSV = Math.max(2, (config.ees * 5) / pLevel); 
    const vC = vB - (nativeSV + (flowDrain * dt_eject));

    return { 
        opening: { v: vB, p: map }, 
        closure: { v: vC, p: map },
        isConverged: (vB - vC) < 5 
    };
}
```

### ECMO Calculations
VA-ECMO is modeled as a retrograde flow system that increases the total afterload on the left ventricle.
* Pressure Increase: The retrograde arterial flow significantly raises the $MAP$. Formula Change:
    $MAP_{effective} = MAP_{baseline} + \Delta P_{ecmo}$.Loop Geometry:
* The loop shifts to the right and becomes "taller".
* Stalling Effect: In severe cases, the $MAP$ exceeds the heart's maximum pressure generation ($P_{max} = E_{es} \cdot (V - V_0)$). If this happens, the Aortic Valve never opens ($Q_{out} = 0$), and the loop collapses into a single vertical line on the right side of the graph—a clinical state known as "LV distention".The "EcMella" Interaction: When Impella is added to ECMO, the $Q_{impella}$ term is re-introduced to the $dV/dt$ equation, which "vents" the LV, slanting the vertical lines and pulling the loop back to the left.

## App Mechanics


## Versions
Current version: 1.2.1

1.0.0 - first released version
1.2.0 - mobile optimized (larger controls, thicker lines)
1.2.1 - greyed out controls when MCS not selected

## License
Available open-source under an MIT license.


## References
Sagawa, K. (1981). [The ventricular pressure-volume diagram revisited.](https://www.ahajournals.org/doi/pdf/10.1161/01.cir.63.6.1223) Circulation Research.

Sunagawa, K., et al. (1984). [Optimal coupling of the left ventricle with the arterial system.](https://pubmed.ncbi.nlm.nih.gov/8147838/) American Journal of Physiology.

Burzotta, F., et al. (2019). Impella ventricular support and the pressure-volume relationship. JACC. (Direct visualization of the "teardrop" loop and point convergence).

Uriel, N., et al. (2012). Hemodynamic transition during Impella support. Journal of Heart and Lung Transplantation. (Clinical observation of slanted isovolumetric phases).

