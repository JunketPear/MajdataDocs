# Track → Track

For a Track → Track transition, both track instructions must have the same direction:

* `P` can only connect to `P`.
* `Q` can only connect to `Q`.

`P` and `Q` cannot be connected directly. A node instruction must be placed between them.

## Same Track: Track 3 → Track 3

When the two track instructions refer to the same track, for example:

```text
Track 3 → Track 3
```

the meaning is to travel **one complete revolution** around the current track.

The parametric equation is:

$$
f(t) =
r\left(
\cos(\beta_0\pm2\pi t)
+i\sin(\beta_0\pm2\pi t)
\right)
$$

where $\beta_0$ is the angular position of the current path endpoint relative to the center of the track. It is the angle calculated for the tangent point in the previous step.

The sign depends on the travel direction. The unit tangent vector and path length are calculated in the same way as above.

---
