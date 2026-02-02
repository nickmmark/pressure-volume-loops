# pressure-volume-loops

A simple open-source web-based simulator visualizes left ventricular pressure–volume (PV) loops under normal physiology, systolic heart failure, and several forms of mechanical circulatory support including:
* [Intra-aortic balloon pump](https://onepagericu.com/iabp) (IABP)
* [Percutaneous Microaxial flow pump](https://onepagericu.com/impella) (e.g. Impella)
* [Veno-arterial ECMO](https://onepagericu.com/ecmo-fundamentals) (VA-ECMO)


### Calculations
The application uses a dynamic simulation approach, treating the heart as a variable pressure-volume reservoir coupled with a continuous-flow pump. Unlike static geometric models, this version solves a system of differential equations over a discretized cardiac cycle, $T$, where $dt = 2\text{ms}$.

#### Ventricular Elastance $E(t)$ 
The "squeeze" of the heart is modeled as a normalized activation function:
```math
Et = (Ees - Emin) * Math.pow(Math.sin(Math.PI * t / Ts), 8) + Emin
```


## References
