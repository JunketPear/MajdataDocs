# Geometric Definitions

We first define the geometry.

The center of the C area is used as the origin of the complex plane. The positive real axis points to the right, and the positive imaginary axis points upward.

The unit length is defined as 100 pixels. Therefore, the radius of the judgment line is:

$$
R = 4.8
$$

The specific numerical value will not be used below; all formulas are expressed in terms of `R`.

## Nodes

The angular positions of the eight outer buttons are:

$$
\theta_n = 112.5^\circ - n \times 45^\circ
$$

The angular positions of the D/E areas are:

$$
\phi_n = 135^\circ - n \times 45^\circ
$$

There are 17 nodes in total, corresponding to `A1~8`, `B1~8`, and `C`.

The nodes corresponding to the `X` and `K` instructions are identical to those of the `A` instruction.

### A Nodes

The A1~8 nodes lie on the judgment line. Their coordinates are:

$$
A_n = R(\cos\theta_n + i\sin\theta_n)
$$

### B Nodes

The positions of the B1~8 nodes are determined by the geometry shown in the diagram. Their coordinates are:

$$
B_n =
R\frac{\cos 67.5^\circ}{\cos 22.5^\circ}
(\cos\theta_n + i\sin\theta_n)
$$

### C Node

The C node is simply the origin:

$$
C = 0
$$

---

## Circular Tracks

There are ten circular tracks, numbered `0~9`.

### Track 0

Track `0` is centered at C and tangent to the line $A_5A_8$.

Its parametric equation is:

$$
f_0(t) =
R\cos67.5^\circ
(\cos t + i\sin t)
$$

### Tracks 1~8

Tracks `1~8` are circles passing through C and tangent to the lines connecting adjacent A areas.

Their diameters are:

$$
R\cos22.5^\circ
$$

Therefore, their centers are:

$$
P_n =
R\frac{\cos22.5^\circ}{2}
(\cos\theta_n + i\sin\theta_n)
$$

Their parametric equations are:

$$
f_n(t) =
P_n +
R\frac{\cos22.5^\circ}{2}
(\cos t + i\sin t)
$$

### Track 9

Track `9` is the outermost circle, i.e. the judgment line itself.

Its parametric equation is:

$$
f_9(t) = R(\cos t + i\sin t)
$$

A diagram is omitted here.

---
