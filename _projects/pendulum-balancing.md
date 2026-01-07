---
hasThumbnail: false
math: true

---

## Background

The inverted pendulum is a classic control problem: the upright equilibrium is open-loop unstable, so keeping the rod balanced requires continuous feedback. This project was done as part of a two-part lab sequence where the goal was to simultaneously control the cart position **x** and pendulum angle **θ** using only the motor voltage (a SIMO system).

Because the lab focuses on small deviations about upright, the plant is linearized using the small-angle approximation (sinθ ≈ θ, cosθ ≈ 1).

<video class="rounded-xl" style="width: 50%; margin: 0 auto; display: block;" controls playsinline preload="metadata">
  <source src="{{ site.baseurl }}../assets/videos/pendulum.MOV" type="video/quicktime">
  Your browser does not support the video tag.
</video>

<br>

## System overview
- **Measured outputs:** cart position **x** and pendulum angle *θ* (encoders).
- **Control input:** motor voltage **V** (force on cart through motor dynamics).
- **State definition:**
  $$
  \mathbf{x} = \begin{bmatrix} x & \dot{x} & \theta & \dot{\theta} \end{bmatrix}^T
  $$


---

## Modeling (high level)
Under the small-angle approximation, the linearized equations of motion can be written as:
$$
(M+m)\ddot{x} + mL_p\ddot{\theta} = F_a
$$
$$
mL_p\ddot{x} + \frac{4mL_p^2}{3}\ddot{\theta} - mgL_p\theta = 0
$$

These dynamics are combined with the motor dynamics (force as a function of motor voltage and cart motion) to produce a state-space model (A, B, C, D) with outputs y = [x, θ]^T.

We can notice that the upright configuration is unstable in open loop (one eigenvalue lies in the right half-plane), which motivates feedback control.

## Lab 6a — Pole placement with full-state feedback

### Goal
Design and implement a full-state feedback controller that stabilizes the upright equilibrium and enables cart position tracking (including sinusoidal references).

### Control law (concept)
For design, we assume the full state is available:
$$
u = -K\,\mathbf{x}
$$

The gain K = [k1 k2 k3 k4] is chosen so the closed-loop eigenvalues of (A − BK) match desired pole locations . In our implementation, we solved for K in MATLAB and verified it via pole placement.

### Reference tracking structure
To track position commands, we used:
$$
u = K(r - \mathbf{x})
$$
so that the reference r has the same dimension as the state.

For experiments, we used a sinusoidal cart position reference of the form:
$$
r = [M\sin(\omega t),\ 0,\ 0,\ 0]^T
$$

### Observations/Limitations
We noticed that numerical differentiation amplified noise, producing spiky, low-quality velocity estimates. This noise propagated into the control input and could manifest as jittery motion and audible high-frequency behavior. We also saw persistent small oscillations about equilibrium due to modeling linearization, sensor noise, and derivative-based state estimates.

# Lab 6b — Upgraded design with a Luenberger observer

We improved the controller by replacing derivative blocks with a state estimator, improving velocity estimates and control smoothness—especially in the presence of measurement noise.

## Observer equations (concept)
A Luenberger observer has the form:
$$
\dot{\hat{\mathbf{x}}} = A\hat{\mathbf{x}} + Bu + L(y - \hat{y})
$$
where y = Cx and ŷ = Cx̂.

Intuitively:
- **Predictor:** A x̂ + B u rolls the model forward
- **Corrector:** L(y − ŷ) pulls the estimate toward the measured outputs

The controller then uses the estimated state:
$$
u(t) = K\big(r(t) - \hat{\mathbf{x}}(t)\big)
$$

## Choosing observer poles (design idea)
The observer gain L is selected so that (A − LC) is stable, and typically faster than the closed-loop plant so that the estimate converges quickly. We used MATLAB pole placement to compute L for a desired set of observer poles (provided by the lab).

Qualitatively, the observer-based controller produced smoother motion and reduced the “step-like” jitter observed with derivative-based state estimation, especially during sinusoidal tracking and small perturbations.

We also compared our controllers under injected sensor noise and evaluated performance using metrics like:
- **Position deviation** (e.g., standard deviation of cart position about its reference)
- **High-frequency noise power** (relative spectral power above a chosen cutoff)

We noticed that faster observer poles improve responsiveness but can pass more measurement noise into the estimate, while slower poles smooth noise but can lag the true state.
