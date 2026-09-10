# Instructions

Instructions are divided into two categories according to their behavior:

* **Node instructions**: move to a specific point.
* **Track instructions**: rotate along a circular track.

## Node Instructions

These instructions cause the Slide to travel in a straight line toward a specific position in the judgment area.

* **`X` instruction**: represents the starting point of the Star Slide. **The letter `X` must be omitted in the actual code; only its parameter is written.** The parameter is `1~8`, corresponding to the eight starting points on the judgment line.

* **`A` instruction**: move in a straight line to the A area. The parameter is `1~8`.
  * Note: the target point lies on the **judgment line** corresponding to the A area, and adjacent positions are allowed.

* **`B` instruction**: move in a straight line to the B area. The parameter is `1~8`.
  * Note: the target point is not at the center of the B-area Touch sensor. It is slightly closer to the center of the screen, at the corresponding geometric intersection.

* **`C` instruction**: move to the C area, i.e. the exact center of the screen. **This instruction must not be followed by a parameter.**

* **`K` instruction**: represents the endpoint of the Slide. Its visual behavior is exactly the same as the `A` instruction, but it marks the end of the entire Slide for judgment. **A Slide Code must end with a `K` instruction and its parameter.** The parameter is `1~8`.

## Track Instructions

These instructions cause the Slide to rotate along a specific circular track. If the Slide enters a track from outside, a tangent entry path is calculated automatically.

* **`P` instruction**: rotate **counterclockwise** around the specified circle.
* **`Q` instruction**: rotate **clockwise** around the specified circle.

### Track Parameters

A `P` or `Q` instruction must be followed by a digit `0~9`, representing one of ten different circles:

* **Circle `0`**: the central circle, corresponding to the circle used by `p` and `q` in the standard syntax.
* **Circles `1~8`**: the side circles, corresponding to the circles used by `pp` and `qq` in the standard syntax. The number of each circle corresponds to the D/E area it passes through. For example, the circle passing through D3 is circle `3`.
* **Circle `9`**: the outermost circle, corresponding to the judgment-line circle used by `<` and `>` in the standard syntax.

---
