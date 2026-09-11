# Track → Node

Regardless of the previous instruction, the path is now located on the track and must leave the track along a tangent toward the target node.

Likewise, a node located **inside the geometric shape of the track** cannot be reached.

For nodes that lie exactly on a track:

* Track `9` → A node
* Tracks `1~8` → C node

No tangent exit segment is required.

In all other cases, a tangent must first be calculated. The calculation is the same as above, except that the sign used when calculating the angular position is reversed — because **exiting counterclockwise is equivalent to entering clockwise**.

This gives us the angular position $\beta$ of the tangent point relative to the track center.

If the target node lies exactly on the track, $\beta$ is simply the angular position of that node relative to the center of the track.

We also have the angular position $\beta_0$ of the current path endpoint relative to the track center. This is the tangent-point angle calculated in the previous step.

## Traveling Along the Track

We simply travel along the track from angle $\beta_0$ to angle $\beta$ in the specified direction.

Suppose the current track has radius $r$. The parametric equation is:

$$
f(t) =
r\left(
\cos(\beta_0+t(\beta-\beta_0))
+i\sin(\beta_0+t(\beta-\beta_0))
\right)
$$

The unit tangent vector is:

$$
\vec g(t)
= \pm
\left(
-\sin(\beta_0+t(\beta-\beta_0))
+i\cos(\beta_0+t(\beta-\beta_0))
\right)
$$

* Counterclockwise: use the `+` sign.
* Clockwise: use the `−` sign.

The path length is:

$$
l = r|\beta-\beta_0|
$$

If necessary, a tangent segment from the track to the target node is then appended. The two parametric segments are combined into the corresponding piecewise path function.

## Important: Always Take the Shortest Arc

When traveling along a track, the path should always use the **shortest possible arc** in the specified direction. Under normal circumstances, the path must never make an additional full revolution.

## Special Case: Identical Angles

There is one exception.

If:

$$
\beta=\beta_0
$$

the Slide should travel around the track **exactly one full revolution**. This also prevents the implementation from producing a zero-length segment and potentially encountering a division-by-zero error.

Due to floating-point errors in the actual implementation, the following tolerance is used:

$$
|\beta-\beta_0| < 0.001\text{ rad}
$$

That is, if the angular difference is within **0.057°**, the two angles are considered to represent the same point, and the Slide should make one complete revolution.

## Angle Normalization

There is another issue in the implementation: because $\beta$ and $\beta_0$ are obtained through multiple floating-point calculations, they may not necessarily lie within the principal range.

Python's principal argument range is:

$$
[-\pi,\pi]
$$

Therefore, both angles must first be normalized to this range before calculating the difference $(\beta-\beta_0)$.

In Python, `math.remainder()` can be used to calculate the remainder modulo $2\pi$.

Then, depending on the travel direction:

* For counterclockwise travel, if $\beta < \beta_0$, add $2\pi$ to $\beta$.
* For clockwise travel, apply the opposite adjustment.

---
