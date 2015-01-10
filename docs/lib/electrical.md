



# Analog electrical models

This library of components is modeled after the
Modelica.Electrical.Analog library.

Voltage nodes with type `Voltage` are the main Unknown type used in
electrical circuits. `voltage` nodes can be single floating point
unknowns representing a single voltage node. A `Voltage` can also be
an array representing multiphase circuits or multiple node positions.
Lastly, `Voltage` unknowns can also be complex for use with
quasiphasor-type solutions.

The type `ElectricalNode` is a Union type that can be an Array, a
number, an expression, or an Unknown. This is used in model functions
to allow passing a `Voltage` node or a real value (like 0.0 for
ground).

### Example

```julia
function ex_ChuaCircuit()
    n1 = Voltage("n1")
    n2 = Voltage("n2")
    n3 = Voltage(4.0, "n3")
    g = 0.0
    function NonlinearResistor(n1::ElectricalNode, n2::ElectricalNode, Ga, Gb, Ve)
        i = Current(compatible_values(n1, n2))
        v = Voltage(compatible_values(n1, n2))
        @equations begin
            Branch(n1, n2, v, i)
            i = ifelse(v < -Ve, Gb .* (v + Ve) - Ga .* Ve,
                       ifelse(v > Ve, Gb .* (v - Ve) + Ga*Ve, Ga*v))
        end
    end
    @equations begin
        Resistor(n1, g, 12.5e-3) 
        Inductor(n1, n2, 18.0)
        Resistor(n2, n3, 1 / 0.565) 
        Capacitor(n2, g, 100.0)
        Capacitor(n3, g, 10.0)
        NonlinearResistor(n3, g, -0.757576, -0.409091, 1.0)
    end
end

y = sim(ex_ChuaCircuit(), 200.0)
wplot(y)
```




## SeriesProbe

Connect a series current probe between two nodes. This is
vectorizable.

```julia
SeriesProbe(n1, n2, name::String)
```

### Arguments

* `n1` : Positive node
* `n2` : Negative node
* `name::String` : The name of the probe

### Example

```julia
function model()
    n1 = Voltage("n1")
    n2 = Voltage()
    g = 0.0
    Equation[
        SineVoltage(n1, g, 100.0)
        SeriesProbe(n1, n2, "current")
        Resistor(n2, g, 2.0)
    ]
end
y = sim(model())
```

[Sims/src/../lib/electrical.jl:90](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L90)



## BranchHeatPort

Wrap argument `model` with a heat port that captures the power
generated by the electrical device. This is vectorizable.

```julia
BranchHeatPort(n1::ElectricalNode, n2::ElectricalNode, hp::HeatPort,
               model::Function, args...)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `hp::HeatPort` : Heat port [K]                
* `model::Function` : Model to wrap
* `args...` : Arguments passed to `model`  

### Examples

Here's an example of a definition defining a Resistor that uses a heat
port (a Temperature) in terms of another model:

```julia
function Resistor(n1::ElectricalNode, n2::ElectricalNode, R::Signal, hp::Temperature, T_ref::Signal, alpha::Signal) 
    BranchHeatPort(n1, n2, hp, Resistor, R .* (1 + alpha .* (hp - T_ref)))
end
```

[Sims/src/../lib/electrical.jl:129](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L129)




# Basics




## Resistor

The linear resistor connects the branch voltage `v` with the branch
current `i` by `i*R = v`. The Resistance `R` is allowed to be positive,
zero, or negative. 

```julia
Resistor(n1::ElectricalNode, n2::ElectricalNode, R::Signal)
Resistor(n1::ElectricalNode, n2::ElectricalNode, 
         R = 1.0, T = 293.15, T_ref = 300.15, alpha = 0.0)
Resistor(n1::ElectricalNode, n2::ElectricalNode;             # keyword-arg version
         R = 1.0, T = 293.15, T_ref = 300.15, alpha = 0.0)
Resistor(n1::ElectricalNode, n2::ElectricalNode,
         R::Signal, hp::Temperature, T_ref::Signal, alpha::Signal) 
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]

### Keyword/Optional Arguments

* `R::Signal` : Resistance at temperature `T_ref` [ohms], default = 1.0 ohms
* `hp::HeatPort` : Heat port [K], optional                
* `T::HeatPort` : Fixed device temperature or HeatPort [K], default = `T_ref`
* `T_ref::Signal` : Reference temperature [K], default = 300.15K
* `alpha::Signal` : Temperature coefficient of resistance (`R_actual = R*(1 + alpha*(T_heatPort - T_ref))`) [1/K], default = 0.0

### Details

The resistance `R` is optionally temperature dependent according to
the following equation:

    R = R_ref*(1 + alpha*(heatPort.T - T_ref))
        
With the optional `hp` HeatPort argument, the power will be dissipated
into this HeatPort.

The resistance `R` can be a constant numeric value or an Unknown,
meaning it can vary with time. *Note*: it is recommended that the R
signal should not cross the zero value. Otherwise, depending on the
surrounding circuit, the probability of singularities is high.

This device is vectorizable using array inputs for one or both of
`n1` and `n2`.

### Example

```julia
function model()
    n1 = Voltage("n1")
    g = 0.0
    Equation[
        SineVoltage(n1, g, 100.0)
        Resistor(n1, g, R = 3.0, T = 330.0, alpha = 1.0)
    ]
end
y = sim(model())
```

[Sims/src/../lib/electrical.jl:218](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L218)



## Capacitor

The linear capacitor connects the branch voltage `v` with the branch
current `i` by `i = C * dv/dt`. 

```julia
Capacitor(n1::ElectricalNode, n2::ElectricalNode, C::Signal = 1.0) 
Capacitor(n1::ElectricalNode, n2::ElectricalNode; C::Signal = 1.0) 
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]

### Keyword/Optional Arguments

* `C::Signal` : Capacitance [F], default = 1.0 F

### Details

`C` can be a constant numeric value or an Unknown, meaning it can vary
with time. If `C` is a constant, it may be positive, zero, or
negative. If `C` is a signal, it should be greater than zero.

This device is vectorizable using array inputs for one or both of `n1`
and `n2`.

### Example

```julia    
function model()
    n1 = Voltage("n1")
    g = 0.0
    Equation[
        SineVoltage(n1, g, 100.0)
        Resistor(n1, g, 3.0)
        Capacitor(n1, g, 1.0)
    ]
end
```

[Sims/src/../lib/electrical.jl:286](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L286)



## Inductor

The linear inductor connects the branch voltage `v` with the branch
current `i` by `v = L * di/dt`. 

```julia
Inductor(n1::ElectricalNode, n2::ElectricalNode, L::Signal = 1.0) 
Inductor(n1::ElectricalNode, n2::ElectricalNode; L::Signal = 1.0)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]

### Keyword/Optional Arguments

* `L::Signal` : Inductance [H], default = 1.0 H

### Details

`L` can be a constant numeric value or an Unknown,
meaning it can vary with time. If `L` is a constant, it may be
positive, zero, or negative. If `L` is a signal, it should be
greater than zero.

This device is vectorizable using array inputs for one or both of
`n1` and `n2`

### Example

```julia
function model()
    n1 = Voltage("n1")
    g = 0.0
    Equation[
        SineVoltage(n1, g, 100.0)
        Resistor(n1, g, 3.0)
        Inductor(n1, g, 6.0)
    ]
end
```

[Sims/src/../lib/electrical.jl:341](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L341)



## SaturatingInductor

TBD

[Sims/src/../lib/electrical.jl:356](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L356)



## Transformer

The transformer is a two port. The left port voltage `v1`, left port
current `i1`, right port voltage `v2` and right port current `i2` are
connected by the following relation:

    | v1 |         | L1   M  |  | i1' |
    |    |    =    |         |  |     |
    | v2 |         | M    L2 |  | i2' |

`L1`, `L2`, and `M` are the primary, secondary, and coupling inductances
respectively.

```julia
Transformer(p1::ElectricalNode, n1::ElectricalNode, p2::ElectricalNode, n2::ElectricalNode, 
            L1 = 1.0, L2 = 1.0, M = 1.0)
Transformer(p1::ElectricalNode, n1::ElectricalNode, p2::ElectricalNode, n2::ElectricalNode; 
            L1 = 1.0, L2 = 1.0, M = 1.0)
```
### Arguments

* `p1::ElectricalNode` : Positive electrical node of the left port (potential `p1 > n1` for positive voltage drop v1) [V]
* `n1::ElectricalNode` : Negative electrical node of the left port [V]
* `p2::ElectricalNode` : Positive electrical node of the right port (potential `p2 > n2` for positive voltage drop v2) [V]
* `n2::ElectricalNode` : Negative electrical node of the right port [V]

### Keyword/Optional Arguments

* `L1::Signal` : Primary inductance [H], default = 1.0 H
* `L2::Signal` : Secondary inductance [H], default = 1.0 H
* `M::Signal`  : Coupling inductance [H], default = 1.0 H

[Sims/src/../lib/electrical.jl:457](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L457)



## EMF

EMF transforms electrical energy into rotational mechanical energy. It
is used as basic building block of an electrical motor. The mechanical
connector `flange` can be connected to elements of the rotational
library. 

```julia
EMF(n1::ElectricalNode, n2::ElectricalNode, flange::Flange,
    support_flange = 0.0, k = 1.0)
EMF(n1::ElectricalNode, n2::ElectricalNode, flange::Flange;
    support_flange = 0.0, k = 1.0)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `flange::Flange` : Rotational shaft

### Keyword/Optional Arguments

* `support_flange` : Support/housing of the EMF shaft 
* `k` : Transformation coefficient [N.m/A] 

[Sims/src/../lib/electrical.jl:501](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L501)




# Ideal




## IdealDiode

This is an ideal switch which is **open** (off), if it is reversed
biased (voltage drop less than 0) **closed** (on), if it is conducting
(`current > 0`). This is the behaviour if all parameters are exactly
zero. Note, there are circuits, where this ideal description with zero
resistance and zero cinductance is not possible. In order to prevent
singularities during switching, the opened diode has a small
conductance `Gon` and the closed diode has a low resistance `Roff`
which is default.

The parameter `Vknee` which is the forward threshold voltage, allows
to displace the knee point along the `Gon`-characteristic until `v =
Vknee`. 

```julia
IdealDiode(n1::ElectricalNode, n2::ElectricalNode, 
           Vknee = 0.0, Ron = 1e-5, Goff = 1e-5)
IdealDiode(n1::ElectricalNode, n2::ElectricalNode; 
           Vknee = 0.0, Ron = 1e-5, Goff = 1e-5)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]

### Keyword/Optional Arguments

* `Vknee` : Forward threshold voltage [V], default = 0.0
* `Ron` : Closed diode resistance [Ohm], default = 1.E-5
* `Goff` : Opened diode conductance [S], default = 1.E-5


[Sims/src/../lib/electrical.jl:566](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L566)



## IdealThyristor

This is an ideal thyristor model which is **open** (off), if the
voltage drop is less than 0 or `fire` is false **closed** (on), if the
voltage drop is greater or equal 0 and `fire` is true.

This is the behaviour if all parameters are exactly zero. Note, there
are circuits, where this ideal description with zero resistance and
zero cinductance is not possible. In order to prevent singularities
during switching, the opened thyristor has a small conductance `Goff`
and the closed thyristor has a low resistance `Ron` which is default.

The parameter `Vknee` which is the forward threshold voltage, allows to
displace the knee point along the `Goff`-characteristic until `v =
Vknee`. 

```julia
IdealThyristor(n1::ElectricalNode, n2::ElectricalNode, fire::Discrete, 
               Vknee = 0.0, Ron = 1e-5, Goff = 1e-5)
IdealThyristor(n1::ElectricalNode, n2::ElectricalNode, fire::Discrete; 
               Vknee = 0.0, Ron = 1e-5, Goff = 1e-5)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `fire::Discrete` : Discrete bool variable indicating firing of the thyristor

### Keyword/Optional Arguments

* `Vknee` : Forward threshold voltage [V], default = 0.0
* `Ron` : Closed thyristor resistance [Ohm], default = 1.E-5
* `Goff` : Opened thyristor conductance [S], default = 1.E-5

[Sims/src/../lib/electrical.jl:619](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L619)



## IdealGTOThyristor

This is an ideal GTO thyristor model which is **open** (off), if the
voltage drop is less than 0 or `fire` is false **closed** (on), if the
voltage drop is greater or equal 0 and `fire` is true.

This is the behaviour if all parameters are exactly zero.  Note, there
are circuits, where this ideal description with zero resistance and
zero cinductance is not possible. In order to prevent singularities
during switching, the opened thyristor has a small conductance `Goff`
and the closed thyristor has a low resistance `Ron` which is default.

The parameter `Vknee` which is the forward threshold voltage, allows
to displace the knee point along the `Goff`-characteristic until `v =
Vknee`.

```julia
IdealGTOThyristor(n1::ElectricalNode, n2::ElectricalNode, fire::Discrete, 
                  Vknee = 0.0, Ron = 1e-5, Goff = 1e-5)
IdealGTOThyristor(n1::ElectricalNode, n2::ElectricalNode, fire::Discrete; 
                  Vknee = 0.0, Ron = 1e-5, Goff = 1e-5)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `fire::Discrete` : Discrete bool variable indicating firing of the thyristor

### Keyword/Optional Arguments

* `Vknee` : Forward threshold voltage [V], default = 0.0
* `Ron` : Closed thyristor resistance [Ohm], default = 1.E-5
* `Goff` : Opened thyristor conductance [S], default = 1.E-5

[Sims/src/../lib/electrical.jl:675](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L675)



## IdealOpAmp

The ideal OpAmp is a two-port device. The left port is fixed to `v1=0` and
`i1=0` (nullator). At the right port, both any voltage `v2` and any
current `i2` are possible (norator).

The ideal OpAmp with three pins is of exactly the same behaviour as the
ideal OpAmp with four pins. Only the negative output pin is left out.
Both the input voltage and current are fixed to zero (nullator). At the
output pin both any voltage `v2` and any current `i2` are possible.

```julia
IdealOpAmp(p1::ElectricalNode, n1::ElectricalNode, p2::ElectricalNode, n2::ElectricalNode)
IdealOpAmp(p1::ElectricalNode, n1::ElectricalNode, p2::ElectricalNode)
```
### Arguments

* `p1::ElectricalNode` : Positive electrical node of the left port (potential `p1 > n1` for positive voltage drop v1) [V]
* `n1::ElectricalNode` : Negative electrical node of the left port [V]
* `p2::ElectricalNode` : Positive electrical node of the right port (potential `p2 > n2` for positive voltage drop v2) [V]
* `n2::ElectricalNode` : Negative electrical node of the right port [V], defaults to 0.0 V

[Sims/src/../lib/electrical.jl:717](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L717)



## IdealOpeningSwitch

The ideal opening switch has a positive pin `p` and a negative pin `n`. The
switching behaviour is controlled by the input signal `control`. If
control is true, pin p is not connected with negative pin n. Otherwise,
pin p is connected with negative pin n.

In order to prevent singularities during switching, the opened switch
has a (very low) conductance `Goff` and the closed switch has a (very low)
resistance `Ron`. The limiting case is also allowed, i.e., the resistance
Ron of the closed switch could be exactly zero and the conductance Goff
of the open switch could be also exactly zero. Note, there are circuits,
where a description with zero Ron or zero Goff is not possible.

```julia
IdealOpeningSwitch(n1::ElectricalNode, n2::ElectricalNode, control::Discrete,
                   Ron = 1e-5, Goff = 1e-5)
IdealOpeningSwitch(n1::ElectricalNode, n2::ElectricalNode, control::Discrete;
                   Ron = 1e-5, Goff = 1e-5)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `control::Discrete` : true => switch open, false => n1-n2 connected

### Keyword/Optional Arguments

* `Ron` : Closed switch resistance [Ohm], default = 1.E-5
* `Goff` : Opened switch conductance [S], default = 1.E-5

[Sims/src/../lib/electrical.jl:765](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L765)



## IdealClosingSwitch

The ideal closing switch has a positive pin `p` and a negative pin `n`. The
switching behaviour is controlled by input signal `control`. If control is
true, pin p is connected with negative pin n. Otherwise, pin p is not
connected with negative pin n.

In order to prevent singularities during switching, the opened switch
has a (very low) conductance `Goff` and the closed switch has a (very low)
resistance `Ron`. The limiting case is also allowed, i.e., the resistance
Ron of the closed switch could be exactly zero and the conductance Goff
of the open switch could be also exactly zero. Note, there are circuits,
where a description with zero Ron or zero Goff is not possible.

```julia
IdealClosingSwitch(n1::ElectricalNode, n2::ElectricalNode, control::Discrete,
                   Ron = 1e-5, Goff = 1e-5)
IdealClosingSwitch(n1::ElectricalNode, n2::ElectricalNode, control::Discrete;
                   Ron = 1e-5, Goff = 1e-5)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `control::Discrete` : true => n1-n2 connected, false => switch open

### Keyword/Optional Arguments

* `Ron` : Closed switch resistance [Ohm], default = 1.E-5
* `Goff` : Opened switch conductance [S], default = 1.E-5

[Sims/src/../lib/electrical.jl:813](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L813)



## ControlledIdealOpeningSwitch

TBD

[Sims/src/../lib/electrical.jl:833](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L833)



## ControlledIdealClosingSwitch

TBD

[Sims/src/../lib/electrical.jl:856](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L856)



## ControlledOpenerWithArc

This model is an extension to the `IdealOpeningSwitch`.

The basic model interupts the current through the switch in an
infinitesimal time span. If an inductive circuit is connected, the
voltage across the switch is limited only by numerics. In order to give
a better idea for the voltage across the switch, a simple arc model is
added:

When the Boolean input `control` signals to open the switch, a voltage
across the opened switch is impressed. This voltage starts with `V0`
(simulating the voltage drop of the arc roots), then rising with slope
`dVdt` (simulating the rising voltage of an extending arc) until a
maximum voltage `Vmax` is reached.


         | voltage
    Vmax |      +-----
         |     /
         |    /
    V0   |   +
         |   |
         +---+-------- time

This arc voltage tends to lower the current following through the
switch; it depends on the connected circuit, when the arc is quenched.
Once the arc is quenched, i.e., the current flowing through the switch
gets zero, the equation for the off-state is activated `i=Goff*v`.

When the Boolean input `control` signals to close the switch again,
the switch is closed immediately, i.e., the equation for the on-state is
activated `v=Ron*i`.

Please note: In an AC circuit, at least the arc quenches when the next
natural zero-crossing of the current occurs. In a DC circuit, the arc
will not quench if the arc voltage is not sufficient that a
zero-crossing of the current occurs.

This model is the same as ControlledOpenerWithArc, but the switch
is closed when `control > level`. 

```julia
ControlledOpenerWithArc(n1::ElectricalNode, n2::ElectricalNode, control::Signal,
                        level = 0.5,  Ron = 1e-5,  Goff = 1e-5,  V0 = 30.0,  dVdt = 10e3,  Vmax = 60.0)
ControlledOpenerWithArc(n1::ElectricalNode, n2::ElectricalNode, control::Signal;
                        level = 0.5,  Ron = 1e-5,  Goff = 1e-5,  V0 = 30.0,  dVdt = 10e3,  Vmax = 60.0)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `control::Signal` : `control > level` the switch is opened, otherwise closed

### Keyword/Optional Arguments

* `level` : Switch level [V], default = 0.5
* `Ron` : Closed switch resistance [Ohm], default = 1.E-5
* `Goff` : Opened switch conductance [S], default = 1.E-5
* `V0` : Initial arc voltage [V], default = 30.0
* `dVdt` : Arc voltage slope [V/s], default = 10e3
* `Vmax` : Max. arc voltage [V], default = 60.0

[Sims/src/../lib/electrical.jl:929](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L929)



## ControlledCloserWithArc

This model is the same as ControlledOpenerWithArc, but the switch
is closed when `control > level`. 

```julia
ControlledCloserWithArc(n1::ElectricalNode, n2::ElectricalNode, control::Signal,
                        level = 0.5,  Ron = 1e-5,  Goff = 1e-5,  V0 = 30.0,  dVdt = 10e3,  Vmax = 60.0)
ControlledCloserWithArc(n1::ElectricalNode, n2::ElectricalNode, control::Signal;
                        level = 0.5,  Ron = 1e-5,  Goff = 1e-5,  V0 = 30.0,  dVdt = 10e3,  Vmax = 60.0)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `control::Signal` : `control > level` the switch is closed, otherwise open

### Keyword/Optional Arguments

* `level` : Switch level [V], default = 0.5
* `Ron` : Closed switch resistance [Ohm], default = 1.E-5
* `Goff` : Opened switch conductance [S], default = 1.E-5
* `V0` : Initial arc voltage [V], default = 30.0
* `dVdt` : Arc voltage slope [V/s], default = 10e3
* `Vmax` : Max. arc voltage [V], default = 60.0

[Sims/src/../lib/electrical.jl:995](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L995)




# Semiconductors




## Diode

The simple diode is a one port. It consists of the diode itself and an
parallel ohmic resistance `R`. The diode formula is:

    i  =  ids * ( e^(v/vt) - 1 )

If the exponent `v/vt` reaches the limit `maxex`, the diode
characterisic is linearly continued to avoid overflow.

```julia
Diode(n1::ElectricalNode, n2::ElectricalNode, 
      Ids = 1e-6,  Vt = 0.04,  Maxexp = 15,  R = 1e8)
Diode(n1::ElectricalNode, n2::ElectricalNode; 
      Ids = 1e-6,  Vt = 0.04,  Maxexp = 15,  R = 1e8)
Diode(n1::ElectricalNode, n2::ElectricalNode, hp::HeatPort,
      Ids = 1e-6,  Vt = 0.04,  Maxexp = 15,  R = 1e8)
Diode(n1::ElectricalNode, n2::ElectricalNode; hp::HeatPort;
      Ids = 1e-6,  Vt = 0.04,  Maxexp = 15,  R = 1e8)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `hp::HeatPort` : Heat port [K]                

### Keyword/Optional Arguments

* `Ids` : Saturation current [A], default = 1.e-6
* `Vt` : Voltage equivalent of temperature (kT/qn) [V], default = 0.04
* `Maxexp` : Max. exponent for linear continuation, default = 15.0
* `R` : Parallel ohmic resistance [Ohm], default = 1.e8

[Sims/src/../lib/electrical.jl:1047](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L1047)



## ZDiode

TBD

[Sims/src/../lib/electrical.jl:1072](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L1072)



## HeatingDiode

The simple diode is an electrical one port, where a heat port is added,
which is defined in the Thermal library. It consists of the
diode itself and an parallel ohmic resistance `R`. The diode formula is:

    i  =  ids * ( e^(v/vt_t) - 1 )

where `vt_t` depends on the temperature of the heat port:

    vt_t = k*temp/q

If the exponent `v/vt_t` reaches the limit `maxex`, the diode
characterisic is linearly continued to avoid overflow. The thermal
power is calculated by `i*v`.

```julia
HeatingDiode(n1::ElectricalNode, n2::ElectricalNode, 
             T = 293.15,  Ids = 1e-6,  Maxexp = 15,  R = 1e8,  EG = 1.11,  N = 1.0,  TNOM = 300.15,  XTI = 3.0)
HeatingDiode(n1::ElectricalNode, n2::ElectricalNode; 
             T = 293.15,  Ids = 1e-6,  Maxexp = 15,  R = 1e8,  EG = 1.11,  N = 1.0,  TNOM = 300.15,  XTI = 3.0)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]

### Keyword/Optional Arguments

* `T` : Heat port [K], default = 293.15
* `Ids` : Saturation current [A], default = 1.e-6
* `Maxexp` : Max. exponent for linear continuation, default = 15.0
* `R` : Parallel ohmic resistance [Ohm], default = 1.e8
* `EG` : Activation energy, default = 1.11
* `N` : Emmission coefficient, default = 1.0
* `TNOM` : Parameter measurement temperature [K], default = 300.15
* `XTI` : Temperature exponent of saturation current, default = 3.0

[Sims/src/../lib/electrical.jl:1135](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L1135)




# Sources




## SignalVoltage

The signal voltage source is a parameterless converter of real valued
signals into a source voltage.

This voltage source may be vectorized.

```julia
SignalVoltage(n1::ElectricalNode, n2::ElectricalNode, V::Signal)  
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `V::Signal` : Voltage between n1 and n2 (= n1 - n2) as an input signal

[Sims/src/../lib/electrical.jl:1180](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L1180)



## SineVoltage

A sinusoidal voltage source. An offset parameter is introduced,
which is added to the value calculated by the blocks source. The
startTime parameter allows to shift the blocks source behavior on the
time axis.

This voltage source may be vectorized.

```julia
SineVoltage(n1::ElectricalNode, n2::ElectricalNode, 
            V = 1.0,  f = 1.0,  ang = 0.0,  offset = 0.0)
SineVoltage(n1::ElectricalNode, n2::ElectricalNode; 
            V = 1.0,  f = 1.0,  ang = 0.0,  offset = 0.0)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]

### Keyword/Optional Arguments

* `V` : Amplitude of sine wave [V], default = 1.0
* `phase` : Phase of sine wave [rad], default = 0.0
* `freqHz` : Frequency of sine wave [Hz], default = 1.0
* `offset` : Voltage offset [V], default = 0.0
* `startTime` : Time offset [s], default = 0.0

[Sims/src/../lib/electrical.jl:1217](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L1217)



## StepVoltage

A step voltage source. An event is introduced at the transition.
Probably cannot be vectorized.

```julia
StepVoltage(n1::ElectricalNode, n2::ElectricalNode, 
            V = 1.0,  start = 0.0,  offset = 0.0)
StepVoltage(n1::ElectricalNode, n2::ElectricalNode; 
            V = 1.0,  start = 0.0,  offset = 0.0)
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]

### Keyword/Optional Arguments

* `V` : Height of step [V], default = 1.0
* `offset` : Voltage offset [V], default = 0.0
* `startTime` : Time offset [s], default = 0.0

[Sims/src/../lib/electrical.jl:1249](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L1249)



## SignalCurrent

The signal current source is a parameterless converter of real valued
signals into a current voltage.

This current source may be vectorized.

```julia
SignalCurrent(n1::ElectricalNode, n2::ElectricalNode, I::Signal)  
```

### Arguments

* `n1::ElectricalNode` : Positive electrical node [V]
* `n2::ElectricalNode` : Negative electrical node [V]
* `I::Signal` : Current flowing from n1 to n2 as an input signal

[Sims/src/../lib/electrical.jl:1284](https://github.com/tshort/Sims.jl/tree/41fa42185a92c02017ceab02d9b448fb8286c66e/src/../lib/electrical.jl#L1284)
