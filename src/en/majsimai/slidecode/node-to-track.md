# Node → Track

First, only nodes located **outside the geometric shape of a track** can enter that track.

Specifically:

* B and C nodes cannot enter track `9` (the outermost circle).
* The C node cannot enter track `0` (the central circle).
* B2 and B3 cannot enter track `3`.
* The same rule applies to the other tracks accordingly.

For nodes that lie exactly on a track, there are two special cases:

* A node → track `9`
* C node → tracks `1~8`

In these cases, no tangent entry segment is required.

## Tangent Entry

In all other cases, a suitable tangent must be calculated for entering the track.

Consider the example of A7 entering track 3 counterclockwise (`A7-P3`).

Let the tangent point be $T$. To calculate its coordinates, we first need to calculate $P_3$.

The magnitude of $\overrightarrow{P_3T}$ is obviously the radius of track 3:

$$
r =
R\frac{\cos22.5^\circ}{2}
$$

Therefore, the key problem is to determine its angular position.

Since $\angle A_7TP_3$ is a right angle:

$$
\alpha =
\arccos
\frac{|A_7-P_3|}{r}
$$

Let $\beta$ be the argument of $\overrightarrow{P_3T}$. Then:

$$
\beta =
\arg(\overrightarrow{P_3A_7}) \pm \alpha
$$

The choice between `+` and `−` depends on the entry direction. Since this example enters the track counterclockwise, the `+` sign is used.

Thus, the tangent point is:

$$
T =
P_3 +
R\frac{\cos22.5^\circ}{2}
(\cos\beta+i\sin\beta)
$$

The tangent-entry segment can then be expressed as:

$$
f(t) = A_7+t(T-A_7)
$$

The unit tangent vector and path length are calculated in the same way as above.

After entering the track, how the Slide continues along the track depends on the next instruction, which is discussed below.

---
