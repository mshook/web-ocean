# How the wave simulation works

It doesn't solve fluid dynamics; there's no Navier–Stokes solver. It uses Tessendorf's method from linear (Airy) wave theory: the sea is modelled as a sum of millions of independent sine waves, each obeying the physical dispersion relation. A GPU FFT turns those waves into a height and displacement map every frame. All of it is in [`js/wgsl/sim.js`](../js/wgsl/sim.js), driven from [`js/main.js`](../js/main.js).

## 1. A statistical spectrum fixes how much energy each wave gets (`SIM_INIT`)

For every wave vector **k** on a 256×256 grid, `spectrum()` in `sim.js` returns the wave energy from oceanographic models:

- **Wind sea:** a JONSWAP spectrum with limited fetch. Wind speed and fetch distance set the peak frequency and energy (`ωp = 22·(g²/UF)^⅓`, peak-enhancement factor γ = 3.3). Donelan–Banner spreading distributes the energy by direction.
- **Swell:** a second, narrow JONSWAP centred on the swell wavelength and scaled to the chosen significant height Hs. Its directional spreading is tight (cos²ˢ), which gives long, regular crests.
- The spectrum is converted from frequency to wavenumber with the Jacobian `dω/dk / k`, and the tiniest waves are damped (`exp(-k²·0.0012²)`).

Each frequency bin then gets a Gaussian random amplitude and phase, `h0(k) = gauss · sqrt(S(k)·Δk²)`. This is the initial state. It is computed once and recomputed only when a setting changes (`needSpectrum` in `main.js`).

## 2. Each wave advances in time with its own speed (`SIM_EVOLVE`)

This is the physics of the motion. Every frame, each wave's phase advances by its own frequency from the dispersion relation for deep water, including surface tension:

```
ω(k) = sqrt(g·k + (σ/ρ)·k³)        // σ/ρ = 7.4e-5
h(k,t) = h0(k)·e^{-iωt} + conj(h0(-k))·e^{iωt}
```

Long waves travel faster than short ones, so the patterns keep re-forming instead of sliding along rigidly. The `k³` term covers capillary ripples at millimetre scale. The two conjugate terms keep the height real-valued and make each wave travel in its own direction.

The frequencies are rounded to multiples of `2π/600 s`, so the whole sea repeats exactly every 10 minutes and time can wrap without a visible jump.

Several fields are built in frequency space from the same `h(k,t)`:

| Field | Formula | Purpose |
|---|---|---|
| Height | `h` | vertical motion |
| Horizontal displacement | `-i·λ·(k/\|k\|)·h` | "choppy" waves: points move toward crests, sharpening them into trochoid shapes (λ = Choppiness slider) |
| Slopes | `i·k·h` | normals for shading |
| Displacement Jacobian terms | `-λ·kₓk_z/\|k\|·h`, etc. | detect where the surface folds over itself, which drives whitecaps |

## 3. An inverse FFT turns frequencies into a surface (`FFT_ROWS`, `FFT_COLS`)

Radix-2 inverse FFTs in compute shaders, done in workgroup shared memory: one pass over rows, then one over columns. Each pass handles 8 complex channels at once. The column pass writes three texture arrays, one layer per cascade: displacement, derivatives, and squared slopes for the LEAN roughness mips.

## 4. Six cascades cover scales from a kilometre down to millimetres

A single 256² patch can't capture both kilometre swells and millimetre ripples, so there are six patches (`CASC_L` in `main.js`): 1103 m, 211 m, 39.7 m, 7.4 m, 1.39 m and 0.26 m. Each cascade owns a band of wavenumbers between `kMin` and `kMax` (`writeSimStatic()` in `main.js`), so no wavelength is counted twice. The two finest cascades are toned down (×0.85 and ×0.6) for looks.

In the ocean vertex shader ([`js/wgsl/ocean.js`](../js/wgsl/ocean.js)), each vertex sums the displacement from all six textures, sampled at its world position. A mip level is picked by mesh density so distant vertices don't alias. The result moves the vertex horizontally as well as vertically. The same sum, run in `HEIGHT_PROBE`, keeps the camera above the water.

## 5. Foam is the only part that keeps state between frames

The wave motion is a closed-form function of time and carries nothing over from one frame to the next. Foam is the exception. Where the Jacobian drops below a threshold (the surface is folding, so the wave is breaking), foam is added to two camera-centred world-space maps. It then decays over time, drifts downwind and breaks into streaks (`FOAM_UPDATE` in `sim.js`).

## What it doesn't model

Because this is linear superposition, the waves never interact:

- no energy transfer between waves
- no real breaking dynamics (foam is triggered by folding, but crests don't collapse)
- no wind slowly building the sea up over time (the spectrum is fully developed for the chosen fetch the moment a setting changes)
- no shallow water, refraction or shoreline

The choppy displacement is the usual cheap stand-in for the steeper crests and flatter troughs of real nonlinear (Stokes) waves.
