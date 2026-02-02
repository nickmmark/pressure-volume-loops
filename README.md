# pressure-volume-loops

A simple open-source web-based simulator visualizes left ventricular pressure–volume (PV) loops under normal physiology, systolic heart failure, and several forms of mechanical circulatory support including:
* [Intra-aortic balloon pump](https://onepagericu.com/iabp) (IABP)
* [Percutaneous Microaxial flow pump](https://onepagericu.com/impella) (e.g. Impella)
* [Veno-arterial ECMO](https://onepagericu.com/ecmo-fundamentals) (VA-ECMO)


### Calculations


### Impella Calculations
Calculating the shape of the Impella augmented PV curves turns out to be quite tricky...

The application uses a dynamic simulation approach, treating the heart as a variable pressure-volume reservoir coupled with a continuous-flow pump. Unlike static geometric models, this version solves a system of differential equations over a discretized cardiac cycle, $T$, where $dt = 2\text{ms}$.

#### Ventricular Elastance $E(t)$ 
The "squeeze" of the heart is modeled as a normalized activation function:
``` math
E(t) = (E_{es} - E_{min}) \cdot \sin\left(\frac{\pi t}{T_s}\right)^n + E_{min}
```

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

## References
Sagawa, K. (1981). [The ventricular pressure-volume diagram revisited.](https://www.ahajournals.org/doi/pdf/10.1161/01.cir.63.6.1223) Circulation Research.

