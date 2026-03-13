# Calibration and Correction of the Brighter-Fatter Effect in CCDs

```{abstract}
This document is a reference for the calibration and correction of the Brighter-Fatter effect (BFE) in charge-coupled devices. It summarizes the physical basis, the data requirements, and the algorithmic steps needed to derive BFE calibrations from flat-field statistics and to apply corrections to science images. The presentation is implementation-agnostic so that other projects and telescopes can adopt or adapt the procedure. The approach is grounded in the literature (Astier et al., Broughton et al., Coulton et al.) and reflects the methods used for the LSST Camera; an appendix points to the LSST Science Pipelines implementation for comparison.
```

This note is written for **implementers**: instrument teams or data reduction groups who need to add Brighter-Fatter calibration and correction to their own pipelines. It emphasizes **what to measure**, **what models to fit**, and **how to obtain and apply** the calibration products, with equations and references. LSST-specific code and configuration are confined to the appendix.

---

## Physical and theoretical basis

### The brighter-fatter effect

Accumulated charge in a pixel changes the local electric field, so later photoelectrons are deflected into neighboring pixels. Brighter pixels therefore collect more effective area than dimmer ones. As a result, flat-field statistics depart from Poisson behavior and nearby pixels become correlated.

**Astier et al. (2019)** [*A&A* **629**, A36; [arXiv:1905.08677](https://arxiv.org/abs/1905.08677)] establish that:

- The **variance** of uniform (flat) images **flattens at high flux** instead of scaling linearly with the mean ("variance deficit").
- This deficit is linked to the BFE: charge is redistributed into neighbors, creating **correlations** that grow with flux and decay with pixel separation.
- They provide an **analytical model** for the PTC and for covariances as a function of intensity, and tie both to electrostatic quantities used in correction.
- The observed statistics are consistent with **charge redistribution during drift**; the assumption that pixel boundaries shift in strict proportion to source charge is questioned.

**Broughton et al. (2023)** [arXiv:2312.03115] demonstrate on LSST Camera data that the BFE **evolves with flux** (higher-order terms matter), so a single flux-independent kernel is an approximation. They introduce **sampling** the covariance model at a chosen flux to include higher-order terms in the kernel, and recommend enforcing **zero-sum** on the kernel for charge conservation.

### Covariance from flat-field pairs

The covariance between a central pixel \(\mathbf{x}=(0,0)\) and a neighbor at lag \((i,j)\) is defined from the **difference** of two flat fields \(F_1\), \(F_2\) at the same nominal illumination (mean \(\mu = \mathrm{avg}(F_1,F_2)\)). **Broughton et al. (2023)** Eq. 1:

\[
C_{ij} = \frac{1}{2}\,\mathrm{Cov}\bigl(F_1(\mathbf{x})-F_2(\mathbf{x}),\;
F_1(\mathbf{x}')-F_2(\mathbf{x}')\bigr), \qquad \mathbf{x}' = (i,j).
\]

If pixels were independent and Poissonian, \(C_{00}\) would be linear in \(\mu\). The observed flattening of the variance at high flux is the variance deficit. Summing covariances over all lags recovers the lost variance (charge conservation; Downing et al. 2006). Covariances are typically computed from the difference image using an FFT-based method (Astier et al. 2019, Appendix A).

### Full covariance model (Astier et al. / Broughton et al.)

**Astier et al. (2019)** Eq. 20 and **Broughton et al. (2023)** Eq. 2 parameterize the covariance at mean signal \(\mu\) (in ADU) as:

\[
C_{ij}(\mu) = \frac{\mu}{g}\Bigl[\delta_{i0}\delta_{j0}
  + a_{ij}\,\mu g
  + \frac{2}{3}[\mathbf{a}\otimes\mathbf{a}+\mathbf{ab}]_{ij}(\mu g)^2
  + \frac{1}{6}[\ldots]_{ij}(\mu g)^3 + \cdots\Bigr] + \frac{n_{ij}}{g^2}.
\]

- \(a_{ij}\) [1/e⁻]: **fractional pixel area change** from accumulated charge (linear BFE).
- \(b_{ij}\): time-evolving (higher-order) component; often fixed to zero to stabilize the fit.
- \(g\): gain (e⁻/ADU); \(n_{ij}\): noise covariance matrix; \(\otimes\): discrete convolution.

The linear term in \(\mu\) dominates; the \(\mu^2\) and \(\mu^3\) terms are **higher-order BFE** and are non-negligible (e.g. ~20% of variance loss in Astier et al.; 15–30% in Broughton et al. at high flux). Fitting this model to measured covariances as a function of \(\mu\) yields gain, noise, and the **a matrix** (and optionally **b**) per amplifier. When **b** is fixed to zero, the fitted **a** matrix approximately satisfies \(\sum a_{ij} \approx 0\) (Broughton et al.).

### Kernel-based correction (Coulton et al. / Broughton et al.)

**Coulton et al. (2018)** model the deflection field as the gradient of a scalar kernel \(K\). The residual covariance after subtracting Poisson and read noise,

\[
\widetilde{C}_{ij} = C_{ij} - \Bigl(\frac{\mu}{g}\delta_{i0}\delta_{j0} + \frac{n_{ij}}{g^2}\Bigr),
\]

then satisfies (**Broughton et al. (2023)** Eq. 3–4):

\[
\widetilde{C}_{ij} = -\mu^2 \nabla^2 K_{ij}.
\]

So the **Laplacian** of \(K\) is proportional to \(\widetilde{C}_{ij}/\mu^2\). The corrected image is (**Coulton et al.** Eq. 22; **Broughton et al.** Eq. 5):

\[
\hat{F}(\mathbf{x}) = F(\mathbf{x})
  + \frac{1}{2}\nabla\cdot\bigl[F(\mathbf{x})\,\nabla(K\otimes F(\mathbf{x}))\bigr].
\]

The factor \(1/2\) averages the BFE over the exposure. Because the true covariance depends on \(\mu\) (Eq. 2), there is no unique flux at which to evaluate \(\widetilde{C}_{ij}\) to build \(K\). **Broughton et al.** recommend **sampling** the full covariance model at a chosen flux so that higher-order terms are included; the kernel is then applied iteratively until the correction converges.

---

## Data requirements

### Observations

- **Flat-field pairs**: two exposures at the **same nominal illumination** (e.g. same integration time, or flux-matched). Many pairs are needed across a range of flux levels from low signal to near full well.
- **Coverage**: typically from well above read noise up to a safe limit below saturation. Broughton et al. define several "turnoff" levels (PTC turnoff, sCTI turnoff, pCTI turnoff); fitting the covariance model only up to the relevant turnoff avoids contaminating the BFE estimate with other effects.
- **Optional**: bias and dark frames for calibration; photodiode or other independent flux measurement for gain/linearity if desired.

### What to compute from each pair

For each amplifier (or region of interest):

1. **Mean signal** \(\mu\) (e.g. mean of the two flat images or of their average).
2. **Variance** of the difference image (or equivalent) → gives \(C_{00}\) at that \(\mu\).
3. **Covariances** \(C_{ij}\) at lags \((i,j)\) out to a chosen maximum lag (e.g. 8–10 pixels). These are computed from the same difference image; the FFT-based method in Astier et al. (2019) Appendix A is efficient.

The result is a set of **photon transfer curve (PTC) data**: \(\mu\), variance, and covariance matrices as a function of \(\mu\).

---

## Algorithm: covariance measurement and PTC fit

### Step 1: Pair flats and form difference images

- Group flat exposures into pairs at the same nominal flux (e.g. by exposure time or by measured mean level).
- For each pair \((F_1, F_2)\), apply any needed calibration (bias, dark, linearity) so that the difference \(F_1 - F_2\) has zero mean apart from noise.
- Compute the mean signal \(\mu\) (e.g. average of the two flat means) and the difference-image variance and covariances per amplifier.

### Step 2: Compute covariances at each lag

For each pair and amplifier, compute \(C_{ij}\) using Eq. 1. The FFT method (Astier et al. Appendix A) computes all lags at once from the product of FFTs of the difference image and the mask. Ensure the maximum lag is large enough (e.g. 8–10 pixels); Broughton et al. show that summing covariances out to ~8 pixels recovers the Poisson variance.

### Step 3: Fit the full covariance model

Fit Eq. 2 to the measured \(C_{ij}(\mu)\) over the chosen flux range (e.g. up to pCTI turnoff). The fit yields:

- **Gain** \(g\) and **noise** \(n_{ij}\).
- **a matrix** (and optionally **b**). The a matrix has units 1/e⁻ and describes the linear fractional area change per pixel lag.

Store the fitted a matrix (and full model, if needed for kernel sampling) per amplifier. Reject or mask bad amplifiers before downstream steps.

---

## Algorithm: kernel-based calibration and correction

Two calibration products are in common use: (1) a **convolution kernel** \(K\) for the scalar correction (Coulton et al., Broughton et al.), and (2) an **electrostatic distortion model** (pixel boundary shifts) for model-based correction. This section covers the kernel; the next covers the electrostatic approach.

### Step 4a: Build the “pre-kernel” (correlation matrix)

From the PTC data (and optionally the fitted full covariance model), form a single **correlation matrix** per amplifier that will serve as the source term in \(\nabla^2 K = \text{source}\). Options:

- **Average of measured correlations**: At each flux level, convert covariances to electrons, subtract Poisson at (0,0), divide by \(-2\) and by \(\mu^2\) (Coulton et al. Eq. 29). Reject pairs that fail sanity checks (e.g. \(q_{00}>0\) or triangle-inequality violation). Average (e.g. sigma-clipped) over flux to get one correlation matrix. This is flux-independent by construction.
- **Quadratic fit**: Fit the correlation at each lag \((i,j)\) as a function of \(\mu^2\) and use the slope as the flux-independent correlation.
- **a matrix**: Use the fitted **a** matrix from the full covariance model (Eq. 2) directly as the pre-kernel (up to sign and tiling; see literature). This is the linear term only.
- **Sample the full model at a flux (Broughton et al.)**: Evaluate the full covariance model at a **chosen flux** \(\mu_*\) (e.g. where the correction will matter most), form \(\widetilde{C}_{ij}(\mu_*)\) using Eq. 4, and convert to the correlation \(\widetilde{C}_{ij}/\mu_*^2\) that corresponds to the Laplacian of \(K\). This includes higher-order BFE at that flux.

The pre-kernel is usually stored in quarter form and tiled to a full \((2n+1)\times(2n+1)\) symmetric matrix.

### Step 4b: Enforce zero-sum (recommended)

Adjust the (0,0) element of the pre-kernel so that the full kernel sums to zero. This enforces charge conservation (Broughton et al.). Optionally, extrapolate the correlation beyond the measured range (e.g. power-law) before applying the zero-sum so that the finite measurement range does not bias the sum.

### Step 5: Solve for the kernel \(K\)

Solve the discrete Poisson equation \(\nabla^2 K = \text{source}\) with the pre-kernel as the source and zero boundary conditions. Any standard solver (e.g. successive over-relaxation, multigrid, or FFT-based Poisson solver) is suitable. The result is the **Brighter-Fatter kernel** \(K\) per amplifier (or one per detector if averaged).

### Step 6: Validity checks (optional)

Before using the kernel, check for plausibility:

- **Edge**: Kernel values near the boundary should be close to zero.
- **Center**: The center pixel should be the minimum (kernel negative elsewhere).
- **Negativity**: All off-center values should be negative (or below a small fraction of \(|K_{00}|\)).

Flag or exclude amplifiers that fail these checks.

### Step 7: Apply the correction

Correct the measured image \(F\) using Eq. 5. In practice the correction is applied **iteratively**: compute the right-hand side of Eq. 5, add to \(F\), and repeat until the added charge per iteration falls below a threshold (e.g. Coulton et al.; Broughton et al. use a threshold of 10 electrons, typically 2–3 iterations).

---

## Algorithm: electrostatic (model-based) calibration

An alternative to the scalar kernel is to fit a **physical electrostatic model** to the measured **a matrix** and to output **pixel boundary shifts** (and related quantities) for use in a model-based corrector.

### Rationale

The a matrix encodes the linear response of pixel area to accumulated charge. An electrostatic model (charge at different depths, E-field, boundary motion) predicts an a matrix from parameters such as detector thickness, pixel size, and charge-layer depths. Fitting the model to the measured a matrix yields a compact, physically interpretable calibration that can be evaluated at different **conversion depths** (e.g. per filter) if photon absorption depth matters.

### Steps

1. **Input**: Fitted **a matrix** (and its uncertainty) per amplifier from the full covariance fit (Step 3). Optionally average over good amplifiers and compute an uncertainty matrix.
2. **Model**: Define an electrostatic model that predicts the a matrix from parameters (e.g. thickness, pixel size, \(z_q\), \(z_{\mathrm{sh}}\), \(z_{\mathrm{sv}}\), and scale/offset \(\alpha\), \(\beta\)). The model integrates E-fields over pixel boundaries and over charge depth; see the literature and existing implementations (e.g. LSST ip_isr/cp_pipe) for functional forms.
3. **Fit**: Minimize weighted residuals between the model and the measured a matrix (e.g. least-squares with weights \(1/\sigma^2_{ij}\)). Bounds and which parameters to vary are chosen from prior knowledge or from similar detectors.
4. **Output**: From the best-fit parameters, compute **boundary shifts** (e.g. North, South, East, West) and **area-change** arrays. These can be computed for a single conversion depth (e.g. surface) or for a **conversion-depth distribution** (e.g. using filter throughput and silicon absorption (Rajkanan et al. 1979)) to get a per-filter calibration.

This path is more involved than the kernel path but provides a single physical model that can be evaluated at any flux and, with conversion-depth weighting, at any filter.

---

## Summary and choices

| Aspect | Recommendation / note |
|--------|------------------------|
| **Data** | Flat pairs over a wide flux range; compute covariances from the difference image (Eq. 1; FFT method in Astier et al. Appendix A). |
| **PTC / covariance model** | Fit the full model (Eq. 2) to get gain, noise, and the a matrix; restrict the flux range to below turnoff (e.g. pCTI) to avoid contamination. |
| **Kernel construction** | Pre-kernel from averaged correlations, quadratic fit, a matrix, or **sampled full model at a flux** (Broughton et al.). Enforce **zero-sum** on the kernel. Solve \(\nabla^2 K = \text{source}\). |
| **Correction** | Apply Eq. 5 iteratively until convergence. |
| **Electrostatic path** | Optional; fit electrostatic model to a matrix, then compute boundary shifts (and optionally per-filter via conversion depth). |

---

## References

- **Astier et al. (2019)** — "The shape of the Photon Transfer Curve of CCD sensors," *Astronomy & Astrophysics* **629**, A36; [arXiv:1905.08677](https://arxiv.org/abs/1905.08677). Variance deficit, full covariance model (Eq. 20), a matrix, FFT covariance method (Appendix A). Code: [bfptc](https://gitlab.in2p3.fr/astier/bfptc).
- **Broughton et al. (2023)** — "Mitigation of the Brighter-Fatter Effect in the LSST Camera," [arXiv:2312.03115](https://arxiv.org/abs/2312.03115). Covariance from flat pairs (Eq. 1), full model (Eq. 2), \(\widetilde{C}_{ij}\) and Laplacian kernel (Eq. 3–4), correction formula (Eq. 5), flux-dependent BFE, sampling at a flux, zero-sum kernel, turnoffs and flux range.
- **Coulton et al. (2017/2018)** — Scalar kernel correction; [arXiv:1711.06273](https://arxiv.org/abs/1711.06273). Correlation and kernel (Eq. 22–29).
- **Downing et al. (2006)** — Charge conservation and covariance sum.
- **Rajkanan et al. (1979)** — Silicon absorption coefficient; used for conversion-depth distribution in filter-dependent electrostatic correction.

---

## Appendix: LSST Science Pipelines implementation

For readers who want to compare with or reuse the LSST implementation, the following map may be helpful.

- **Covariance extraction**: Flat pairs and covariance computation from the difference image (FFT): `cp_pipe` PTC extract tasks; `CovFastFourierTransform` in `lsst.cp.pipe.utils`.
- **PTC / full covariance fit**: Fit of Eq. 2 (Astier et al. Eq. 20) to measured covariances: `PhotonTransferCurveSolveTask` in `cp_pipe.ptc` with `ptcFitType: "FULLCOVARIANCE"`.
- **Kernel construction**: Pre-kernel options (average, quadratic fit, a matrix, covariance-model sample), zero-sum, SOR solve: `BrighterFatterKernelSolveTask` in `lsst.cp.pipe.cpBfk`. Pipeline: `pipelines/_ingredients/cpBfkLSST.yaml` (full chain from ISR through PTC to BFK).
- **Electrostatic fit**: Fit of electrostatic model to a matrix, boundary shifts, conversion depth: `ElectrostaticBrighterFatterSolveTask` in `lsst.cp.pipe.cpElectrostaticBF`. Pipeline: `pipelines/_ingredients/cpElectrostaticBFDistortionMatrixLSST.yaml`.
- **Correction application**: Kernel or electrostatic correction is applied in the ISR step when the corresponding calibration is attached; see `lsst.ip.isr` and pipeline configuration for `doBrighterFatter`.

All of the above live in the [LSST Science Pipelines](https://pipelines.lsst.io) (packages `cp_pipe`, `ip_isr`).
