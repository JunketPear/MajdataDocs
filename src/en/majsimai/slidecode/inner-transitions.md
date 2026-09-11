# Transitions Between Tracks 0–8

Next, consider transitions between tracks `0~8`, for example:

```text
Track 0 → Track 3
```

This syntax means that the Slide transfers between the two tracks along a suitable **external common tangent**.

First, note that the two tangent points of an external common tangent have the same angular position relative to their respective circle centers, because both radii are perpendicular to the same tangent line.

Therefore, once we calculate the angular position of the tangent point relative to the circle centers, the entire tangent segment is uniquely determined.

## Tracks 1~8

Because tracks `1~8` all have the same radius, calculating the tangent-point angles of their external common tangent is relatively simple.

We only need the perpendicular direction to the line connecting their centers.

For example, consider counterclockwise travel from track 8 to track 3.

Take the vector from the center of track 8 to the center of track 3:

$$
\overrightarrow{P_8P_3}
$$

For counterclockwise travel, simply rotate this vector **90° clockwise** and take its argument. This gives the angular position of the tangent point.

For clockwise travel, rotate the vector **90° counterclockwise** instead.

---

## Track 0 and Tracks 1~8

The transition between track 0 and tracks `1~8` is slightly more complicated because their radii are different.

First, identify the larger circle — in this case, one of tracks `1~8`.

Construct an auxiliary circle centered at the center of the larger circle, with a radius equal to the difference between the two track radii, as shown by the purple circle in the diagram.

Then apply the tangent-calculation procedure described earlier to find a tangent from the center of the smaller circle to this auxiliary circle.

The resulting tangent-point angle is the angular position of the tangent point of the external common tangent.

The appropriate tangent must then be selected according to the travel direction:

* For counterclockwise travel, use the tangent from the small circle toward the large circle in the counterclockwise direction.
* For clockwise travel, use the tangent from the large circle toward the small circle in the clockwise direction.

---
