+++
title = "Precision loss detection in control systems."
date = 2024-10-09
draft = false
[taxonomies]
  tags = ["Math", "Control System"]
[extra]
  math = true
  math_auto_render = true
  toc = true
  keywords = "Math, Control System"
+++

This article outlines a practical way to detect when runtime data storage starts eroding meaningful precision.

## What is floating-point precision loss?

Computers store floating-point numbers with a fixed number of bits. Values like `1.0` or `0.5` fit exactly, but irrational or long values such as $\pi$ must be rounded to the closest representable number, introducing **error due to data storage**.

Modern hardware typically offers two common formats:

- A 32-bit float uses a 23-bit significand (mantissa), which gives roughly 7 decimal digits of precision.
- A 64-bit float (`double`) uses a 52-bit significand, giving roughly 16 decimal digits of precision.

If we store $\pi$ in these formats:

$$float\ A=\pi$$
$$double\ B=\pi$$

$$A=3.1415927+\epsilon_A$$
$$B=3.141592653589793+\epsilon_B$$

Here, $\epsilon_A$ and $\epsilon_B$ are the storage errors. Exact values like `0.5` or `1.0` fit losslessly because their binary representation is finite.

### Storage error is proportional to the value

Storage error grows with magnitude: a 32-bit float holding `123456789.0` has a larger absolute error than one holding `1.0`. A quick approximation for the absolute storage error when storing a value $x$ is:

$$\epsilon_{float}\approx\frac{|x|}{2^{23}}$$

$$\epsilon_{double}\approx\frac{|x|}{2^{52}}$$

This reflects that each format provides a fixed number of significant bits; the error scales with the size of the value.

## Data uncertainty (data tolerance)

All input data arrive with some uncertainty, regardless of the source.

### Example: measurement error

Sensors report physical values with a stated tolerance. For instance, an NTC temperature sensor might guarantee readings within $\pm 0.1\,C$ at 99.9% confidence. Using $\epsilon_{prob}$ to denote the tolerance at a given probability:

$$\epsilon_{0.999} < 0.1\,C$$

If a control loop is designed around $\epsilon_{0.999}$, the implied failure rate is $1-0.999=0.001$. Any system providing data should also provide the associated tolerance information.

### Tolerance propagation

Within a system—whether SISO or MIMO—uncertainty propagates through every operation. For an input $x$ and operations such as

- addition/subtraction/multiplication/division: $$x+a,\ x-a,\ x\times a,\ \frac{x}{a}$$
- trigonometric functions: $$\sin(x),\ \cos(x),\ \tan(x)$$
- power/logarithm: $$x^a,\ \log_a(x)$$

each output carries an updated tolerance derived from the input tolerances. For a general function $y=f(x,a,b,\dots)$ with independent, approximately normal inputs, the first-order propagation is

$$\sigma_y^2=\left(\sigma_x\frac{\partial f}{\partial x}\right)^2+\left(\sigma_a\frac{\partial f}{\partial a}\right)^2+\left(\sigma_b\frac{\partial f}{\partial b}\right)^2+\dots$$

This first-order approximation is usually sufficient in practice. The same idea extends to vector-valued functions using a covariance matrix and the Jacobian.

## Precision loss detection principle

1. Track the propagated uncertainty ($\sigma$) for every intermediate result using the formulas above.
2. Compute the storage error ($\epsilon$) after writing that result into the chosen floating-point format.
3. Compare storage error to uncertainty. If storage error exceeds a fraction $p$ of the uncertainty, flag a precision-loss event.

Mathematically, for a result $y$:

$$\text{if}\ \epsilon_y > p\,\sigma_y \text{ then precision is considered lost}$$

Commonly $p$ is set to a small number (for example, $p=0.01$ for a 1% threshold). When triggered, downstream computations should treat the value as unreliable.

## Why care about error propagation?

- **Optimize data processing:** Shape algorithms to keep propagated uncertainty low, not just runtime or memory.
- **Data validity detection:** Let software self-check results at runtime and halt or retry when precision loss is detected.
- **Measure reliability:** In safety-critical systems, reported tolerances provide a quantitative measure of output trustworthiness.

## Conclusion

Every data path carries uncertainty. Keep the storage-induced error far below that inherent uncertainty; otherwise, the stored value no longer reflects the real-world signal you are trying to control.
