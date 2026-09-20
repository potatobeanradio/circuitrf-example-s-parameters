# S-Parameters — a circuitRF workspace

This repository **is** a circuitRF workspace. The `.cws` at the root is the workspace file; each
folder beside it is a cell. There is nothing to unpack and nothing to install into.

**To open it:** in circuitRF, *File ▸ Clone Workspace…*, paste

```
https://github.com/potatobeanradio/circuitrf-example-s-parameters.git
```

and choose where to put it. Headless, the same thing is

```
circuitrf history clone https://github.com/potatobeanradio/circuitrf-example-s-parameters.git ./s-parameters
```

A clone arrives with the versions its author kept and **no restore points** — those belong to the
machine they were taken on, and circuitRF starts a safety net of your own on your first boundary.

The same workspace ships inside circuitRF under *Tools ▸ Examples*; this copy exists so the clone
path has something public to point at.

---

Two testbenches, both driven by **Terms** — the element that defines an S-parameter port and its
reference impedance. Run either from the Analyses panel, then open the result in a Data Display.

## Amplifier

A two-port Touchstone file (`potentially_unstable_amp.s2p`) placed as an **SnP** block between two
50 Ω Terms, swept 1–5 GHz. The part is not unconditionally stable across that band, which is the
point of the measurement blocks:

| Block | What it computes |
|---|---|
| `MeasGain` | `S11_dB`, `S21_dB`, `S12_dB`, `S22_dB`, `VSWR_in` |
| `MeasStability` | `Delta`, Rollett `K`, `MU`, and `MSG_dB` |

**None of those stability numbers has to be written out.** `K`, μ, μ′, |Δ| and MAG/MSG are
**built into circuitRF**: add a trace in a Data Display, pick the SP1 network as its source, and
choose the metric from the list under the S-parameter elements — no measurement block, no
expression, and the equation each one uses is written out in *Reference ▸ Derived Metrics*. They are
spelled out here anyway because this is the example that shows you **how to write a custom
equation**, and a formula you can check against a built-in answer is the one worth learning on.
Delete `MeasStability` and the testbench still plots every one of them.

Three things in those expressions are worth copying:

- **`SP1.S(2,1)` is S21.** The `i` and `j` arguments are 1-based **port numbers**, not array
  indices — `S(2,1)` is not "row 2, column 1 of a zero-based matrix".
- **`x*x`, not `x^2`.** Only `+ - * /` broadcast over a result cube; the power operator does not,
  and the run reports the measurement as failed rather than guessing.
- **A measure can use an earlier one.** `K` reads `Delta`, which is declared above it. Order is
  declaration order, across blocks as well as within one.

## CoupledInductors

Two 1 nH inductors with a 500 pH **Mutual** between them. Small enough to check by hand, and it
shows `phase()` — which is in **degrees**, not radians.

It also shows **how to get a real coupling factor out of s-parameters**, which is not `mag(S21)`.
Each winding is grounded at one end, so the two-port impedances are the windings themselves —
Z₁₁ = jωL₁, Z₂₂ = jωL₂, Z₂₁ = jωM — and the measurements convert S to Z and read them off:

```
Zden = (1-S11)*(1-S22) - S12*S21          the S → Z denominator
wL1  = 50*imag(((1+S11)*(1-S22) + S12*S21)/Zden)     Im(Z11) = 2*pi*f*L1
wL2  = 50*imag(((1-S11)*(1+S22) + S12*S21)/Zden)     Im(Z22) = 2*pi*f*L2
wM   = 50*imag(2*S21/Zden)                           Im(Z21) = 2*pi*f*M
k    = wM/sqrt(wL1*wL2)                              M/sqrt(L1*L2)
```

**`k` comes out flat at 0.500000000 across the whole 0.1–5 GHz sweep**, which is the point:
`mag(S21)` on this circuit runs from 0.013 to 0.436 over the same band. A coupling factor is a
property of the windings and does not move with frequency; a transmission magnitude does.

Two things worth noticing:

- **Both `2πf` and the 50 Ω cancel in `k`.** It is a ratio of impedances that are all `jω ×`
  something, so the frequency divides out and the extraction needs no frequency at all. That matters
  because a measurement expression has no way to reach the sweep's own frequency axis — which is
  also why `wL1`, `wL2` and `wM` are published as **reactances in ohms** rather than henries.
  Divide by `2πf` for the inductance: 0.628 Ω at 100 MHz is 1 nH.
- **The 50 in those three lines is the port reference impedance**, the `Z` on Term1 and Term2. It
  scales all three the same way and is gone by the time `k` is formed, so it only matters if you
  want the ohms to be real ohms.
