# Track Transitions Involving Track 9

Transitions involving track `9`, the outermost track, are considerably more complicated.

First:

* Direct transitions between track `0` and track `9` are **forbidden**.
* Tracks `1~8` are internally contained by track `9`, so there is no common tangent between them.

Instead, an appropriate **common tangent circle** is used to perform the transition.

## Common Tangent Circle Parameters

The derivation of this auxiliary circle is rather complicated, so the resulting formulas are given directly here without explanation.

First, define:

$$
\gamma = 45^\circ
$$

This angle is defined as shown in the diagram. A value of `45°` provides a suitable result.

Then define:

$$
b = \frac{\cos22.5^\circ}{2}
$$

$$
a = 1-b
$$

$$
s =
\frac{a^2+b^2-2ab\cos\gamma}
{2a-2b\cos\gamma}
$$

$$
r = R(b+s)
$$

$$
\beta =
\arccos
\frac{s^2+b^2-(a-s)^2}
{2bs}
$$

where:

* $r$ is the radius of the common tangent circle used for the transition.
* $\beta$ is the angle shown in the diagram.

After that, the process is straightforward: simply travel along the current track until reaching the point labeled **A** or **B** in the diagram, and then follow the arc of the common tangent circle into the other track.
