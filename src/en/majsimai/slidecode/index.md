# Slide Code Overview

**Slide Code** is a new syntax for describing Slides. A long and complex slide path can be represented by a short sequence of characters, for example:

`5Q9A1P98CQ49K5`

A Slide Code consists of only two types of elements:

* **Instructions**: uppercase letters `A, B, C, K, P, Q`\
  (Note that the starting instruction `X` is omitted in the actual syntax.)
* **Parameters**: digits `0~9`

## Reading guide

- [Instructions](./instructions)
- [Expansion and Shorthand Rules](./shorthand)
- [Geometric Definitions](./geometry)
- [Node → Node](./node-to-node)
- [Node → Track](./node-to-track)
- [Track → Node](./track-to-node)
- [Track → Track](./track-to-track)
- [Transitions Between Tracks 0–8](./inner-transitions)
- [Track Transitions Involving Track 9](./outer-transitions)

## Parsing overview

When parsing a Slide Code, only four types of transitions need to be considered:

1. Node → Node
2. Node → Track
3. Track → Node
4. Track → Track

---
