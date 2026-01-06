---
hasThumbnail: false
math: true
---

## Background

Magnetic levitation is a classic unstable plant problem: the electromagnetic force depends nonlinearly on both coil current and air gap, so open-loop operation is inherently sensitive. In this lab, a steel ball’s vertical position is measured indirectly using an LED + photoresistor (the ball partially blocks light), and the control input is the current driven through an electromagnet via a current amplifier. The project combined **system identification**, **linear modeling**, and **analog controller implementation** to achieve stable levitation.

This project involved identifying a linearized plant model around a chosen equilibrium height, designing a lead/lag style compensator, implmenting the full controller on a breadboard, and successfully levitating the ball by calibrating offsets, tuning, and debuggin the full loop. 

## High-Level Model and Control Approach

### Nonlinear dynamics and measurement

A common modeling form for MagLev is:

- Mechanical dynamics  
  $$
  m\ddot{x} = f(I, x) - mg
  $$
- Sensor output (photoresistor voltage)  
  $$
  y = h(x)
  $$

$x$ is vertical position and $I$ is coil current. Both $f(\cdot)$ and $h(\cdot)$ are nonlinear.

### Linearization around an equilibrium

We choose an equilibrium point $(x_0, I_0)$ where the ball is balanced (magnetic force approximately cancels gravity). For small deviations $\delta x = x - x_0$, $\delta I = I - I_0$, a first-order linearization gives:

$$
m\,\delta\ddot{x} \approx K_i\,\delta I + K_x\,\delta x
$$
$$
\delta y \approx a\,\delta x
$$

- $a$ captures the **sensor slope** near equilibrium (volts per meter).
- $K_i$ captures how magnetic force changes with current (newtons per amp).
- $K_x$ captures how magnetic force changes with gap (newtons per meter).

This produces an unstable second-order plant.

### Plant transfer function

Combining the linearized mechanics and sensor mapping yields a transfer from current deviation to sensed voltage deviation:

$$
G(s) = \frac{Y(s)}{I(s)} \;\approx\; \frac{aK_i}{m\left(s^2 - \frac{K_x}{m}\right)}
$$

The current amplifier can be modeled as a gain block $K_a$ mapping controller voltage to coil current.

### Compensator structure

Rather than pure proportional control (which can leave poor margins or instability), we introduced a compensator of the form:

$$
G_c(s) = K_c \frac{1 + s/z_c}{1 + s/p_c}
$$

By placing the zero and pole strategically, the compensator can add phase lead (improve stability) or lag (shape low-frequency gain), while also meeting a target open-loop DC gain.

## Lab 5a — System Identification + Offset Circuit

In this lab:
- Determined how the photoresistor voltage changes with ball height near the operating point
- Estimated how much additional magnetic force is produced for a small change in current
- Estimated how the force changes with gap at fixed equilibrium current
- Built and verified front-end circuitry that produces an error-like signal around 0 V


## Lab 5b — Controller Design + Full System Levitation

In the following week, using the identified parameters, I designed a first-order compensator:

$$
G_c(s) = K_c \frac{1 + s/z_c}{1 + s/p_c}
$$

The design goal was to achieve a target open-loop DC gain, use a pole/zero ratio that shapes the transient resopnse and stability margin, and place the compensator zero near the loop’s crossover neighborhod to improve phase behavior.

This was doen by using simulation (root locus) to pick reasonable pole/zero locations, and then solving for component values that implement those dynamics.

To implement $G_c(s)$ in hardware, I selected resistor/capacitor values that realize the same pole/zero form with op-amps. 

The full loop included:
- Desired position / output offset conditioning
- The compensator
- A current-offset stage (to re-center the amplifier input around the equilibrium current)
- The current amplifier driving the electromagnet coil
