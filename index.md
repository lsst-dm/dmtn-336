# Calibration and Correction of the Brighter-Fatter Effect in CCDs

```{abstract}
This document is a reference for the calibration and correction of the Brighter-Fatter effect (BFE) in charge-coupled devices. It summarizes the physical basis, the data requirements, and the algorithmic steps needed to derive BFE calibrations from flat-field statistics and to apply corrections to science images. The presentation is implementation-agnostic so that other projects and telescopes can adopt or adapt the procedure. The approach is grounded in the literature (Astier et al., Broughton et al., Coulton et al.) and reflects the methods used for the LSST Camera; an appendix points to the LSST Science Pipelines implementation for comparison.
```

This note is written for **implementers**: instrument teams or data reduction groups who need to add Brighter-Fatter calibration and correction to their own pipelines. It emphasizes **what to measure**, **what models to fit**, and **how to obtain and apply** the calibration products. LSST-specific code and configuration are confined to the appendix.

**How to use this document.** If you are new to the BFE, start with [What is the Brighter-Fatter effect?](#what-is-the-brighter-fatter-effect). If you already know the physics and want to implement the procedure, go to [Data requirements](#data-requirements) and then [Procedure overview](#procedure-overview). The [Summary table](#summary-and-recommended-choices) at the end gives a compact checklist of choices and recommendations.

---

(what-is-the-brighter-fatter-effect)=
## What is the Brighter-Fatter effect?

**The physical picture.** In a CCD, charge builds up in each pixel during an exposure. That charge alters the electric field inside the sensor. As a result, later-arriving photoelectrons are deflected slightly and may land in a neighboring pixel instead of the one they would have hit in an empty detector. Brighter pixels therefore collect *more* charge than they would if pixels were independent—they effectively have a larger area—and dimmer pixels collect less. This is the **Brighter-Fatter effect (BFE)**.

**Consequences.** Flat-field images no longer follow simple Poisson statistics: the variance of a uniform flat does not scale linearly with the mean flux. Instead, the variance **flattens at high flux** (“variance deficit”). At the same time, **nearby pixels become correlated**: their counts covary in a way that grows with flux and falls off with distance. For science that depends on accurate shapes or fluxes (e.g. weak lensing or photometry), the BFE must be calibrated and corrected.

**What the literature shows.** **Astier et al. (2019)** [*A&A* **629**, A36; [arXiv:1905.08677](https://arxiv.org/abs/1905.08677)] showed that this variance deficit and the pixel correlations are linked, and they gave an analytical model for both. The observed behavior is consistent with charge being redistributed during drift in the sensor. **Broughton et al. (2023)** [arXiv:2312.03115] demonstrated on LSST Camera data that the effect **depends on flux** (higher-order terms matter), so a single flux-independent correction kernel is an approximation; they recommend building the kernel from the covariance model evaluated at a chosen flux and enforcing **zero-sum** on the kernel for charge conservation.

---

## What we measure: covariances from flat-field pairs

To calibrate the BFE we need the **covariance** between pixel counts as a function of mean signal level. Those covariances are measured from **pairs of flat-field exposures** at the same nominal illumination.

**Idea.** Take two flats $F_1$ and $F_2$ at the same flux level. Their difference $F_1 - F_2$ removes the fixed pattern and leaves mainly photon noise and any BFE-induced correlations. The covariance between a central pixel and a neighbor at lag $(i,j)$ is defined (Broughton et al. 2023, Eq. 1) as:

$$
C_{ij} = \frac{1}{2}\,\mathrm{Cov}\bigl(F_1(\mathbf{x})-F_2(\mathbf{x}),\;
F_1(\mathbf{x}')-F_2(\mathbf{x}')\bigr), \qquad \mathbf{x}' = (i,j).
$$

The factor $1/2$ comes from the pair-difference construction. If pixels were independent and Poissonian, the variance $C_{00}$ would be linear in the mean flux $\mu$. The observed flattening of the variance at high flux is the variance deficit. Summing covariances over all lags recovers the lost variance (charge conservation; Downing et al. 2006). In practice, covariances are computed from the difference image using an FFT-based method (Astier et al. 2019, Appendix A).

**Photon transfer curve (PTC).** By repeating this for many flat pairs at different flux levels, we obtain the variance and covariances as a function of mean signal $\mu$. That relationship is the **photon transfer curve (PTC)**. The next step is to fit a model to this PTC so we can extract gain, noise, and the BFE-related “a matrix” used for correction.

---

## The full covariance model and the a matrix

**Model.** Astier et al. (2019) Eq. 20 and Broughton et al. (2023) Eq. 2 parameterize the covariance at mean signal $\mu$ (in ADU) as:

$$
C_{ij}(\mu) = \frac{\mu}{g}\Bigl[\delta_{i0}\delta_{j0}
  + a_{ij}\,\mu g
  + \frac{2}{3}[\mathbf{a}\otimes\mathbf{a}+\mathbf{ab}]_{ij}(\mu g)^2
  + \frac{1}{6}[\ldots]_{ij}(\mu g)^3 + \cdots\Bigr] + \frac{n_{ij}}{g^2}.
$$

**What the symbols mean.**

- $g$ is the gain (electrons per ADU). $n_{ij}$ is the noise covariance matrix (read noise, etc.).
- $a_{ij}$ (units 1/e⁻) is the **fractional pixel area change** from accumulated charge—the linear part of the BFE. It is often called the **a matrix**.
- $b_{ij}$ is a higher-order, time-evolving term; in practice it is often fixed to zero to stabilize the fit.
- $\otimes$ denotes discrete convolution. The terms in $\mu^2$ and $\mu^3$ are **higher-order BFE**; they are not negligible (e.g. ~20% of variance loss in Astier et al.; 15–30% in Broughton et al. at high flux).

**Fitting.** Fitting this model to the measured $C_{ij}(\mu)$ over a chosen flux range yields gain $g$, noise $n_{ij}$, and the **a matrix** (and optionally **b**) per amplifier. When **b** is fixed to zero, the fitted **a** matrix approximately satisfies $\sum a_{ij} \approx 0$ (zero-sum; Broughton et al.). The flux range should be restricted to below “turnoff” (e.g. pCTI turnoff) so that other effects near saturation do not contaminate the BFE estimate.

---

## From covariances to a correction kernel

**Kernel-based correction.** Coulton et al. (2018) model the deflection field as the gradient of a scalar **kernel** $K$. In that picture, the part of the covariance that is due to the BFE (after subtracting Poisson and read noise) is proportional to the Laplacian of $K$. Define the residual covariance:

$$
\widetilde{C}_{ij} = C_{ij} - \Bigl(\frac{\mu}{g}\delta_{i0}\delta_{j0} + \frac{n_{ij}}{g^2}\Bigr).
$$

Then (Broughton et al. 2023, Eq. 3–4):

$$
\widetilde{C}_{ij} = -\mu^2 \nabla^2 K_{ij}.
$$

So once we have $\widetilde{C}_{ij}$ (or its flux-independent part $\widetilde{C}_{ij}/\mu^2$), we can solve the Poisson equation $\nabla^2 K = \text{source}$ to get the kernel $K$.

**Applying the correction.** The corrected image is (Coulton et al. Eq. 22; Broughton et al. Eq. 5):

$$
\hat{F}(\mathbf{x}) = F(\mathbf{x})
  + \frac{1}{2}\nabla\cdot\bigl[F(\mathbf{x})\,\nabla(K\otimes F(\mathbf{x}))\bigr].
$$

The factor $1/2$ averages the BFE over the exposure. In practice the correction is applied **iteratively**: update $F$ with the right-hand side, then repeat until the change per iteration is below a threshold (e.g. 10 electrons; typically 2–3 iterations).

**Choice of flux.** Because the true covariance depends on $\mu$, there is no single “correct” flux at which to evaluate $\widetilde{C}_{ij}$ to build $K$. Broughton et al. recommend **sampling** the full covariance model at a chosen flux (e.g. where correction matters most) so that higher-order terms are included in the kernel.

---

(data-requirements)=
## Data requirements

**What to observe.** You need **flat-field pairs**: two exposures at the **same nominal illumination** (e.g. same integration time or flux-matched). Many such pairs are needed across a **range of flux levels**, from well above read noise up to a safe limit below saturation. Optional: bias and dark frames for calibration; an independent flux measurement (e.g. photodiode) for gain or linearity if desired.

**What to compute from each pair.** For each amplifier (or region of interest):

1. **Mean signal** $\mu$ (e.g. average of the two flat means).
2. **Variance** of the difference image → gives $C_{00}$ at that $\mu$.
3. **Covariances** $C_{ij}$ at lags $(i,j)$ out to a chosen maximum (e.g. 8–10 pixels), from the same difference image (FFT method in Astier et al. Appendix A).

The result is a set of **PTC data**: $\mu$, variance, and covariance matrices as a function of $\mu$.

---

(procedure-overview)=
## Procedure overview

The calibration has three main stages.

1. **Measure covariances and fit the PTC.** Pair flats, form difference images, compute $C_{ij}$ at each lag (Eq. 1), then fit the full covariance model (Eq. 2) over the chosen flux range. You obtain gain, noise, and the **a matrix** per amplifier.
2. **Build a correction kernel (kernel-based correction).** From the PTC data (and optionally the fitted model), form a single “pre-kernel” (correlation matrix) per amplifier, enforce zero-sum, solve $\nabla^2 K = \text{source}$ for $K$, and optionally check the kernel for validity. Apply the correction with Eq. 5 iteratively.
3. **Or fit an electrostatic model (model-based correction).** Alternatively, fit a physical electrostatic model to the measured a matrix and compute pixel boundary shifts (and optionally per-filter via conversion depth). This path is more involved but gives a compact, physics-based calibration.

The sections below spell out the steps in more detail.

---

## Step-by-step: covariance measurement and PTC fit

**Step 1: Pair flats and form difference images.** Group flat exposures into pairs at the same nominal flux (e.g. by exposure time or by measured mean). For each pair $(F_1, F_2)$, apply calibration (bias, dark, linearity) so that the difference $F_1 - F_2$ has zero mean apart from noise. Compute the mean signal $\mu$ and the variance and covariances of the difference image per amplifier.

**Step 2: Compute covariances.** For each pair and amplifier, compute $C_{ij}$ using Eq. 1. The FFT method (Astier et al. Appendix A) gives all lags at once. Use a maximum lag of at least 8–10 pixels; Broughton et al. show that summing covariances out to ~8 pixels recovers the Poisson variance.

**Step 3: Fit the full covariance model.** Fit Eq. 2 to the measured $C_{ij}(\mu)$ over the chosen flux range (e.g. up to pCTI turnoff). The fit yields **gain** $g$, **noise** $n_{ij}$, and the **a matrix** (and optionally **b**) per amplifier. Store the a matrix and, if you will use the “sample at a flux” option for the kernel, the full fitted model. Reject or mask bad amplifiers before the next stage.

---

## Step-by-step: kernel-based calibration and correction

**Step 4a: Build the “pre-kernel” (correlation matrix).** From the PTC data (and optionally the fitted full model), form a single correlation matrix per amplifier that will serve as the source term in $\nabla^2 K = \text{source}$. Common options:

- **Average of measured correlations.** At each flux level, convert covariances to electrons, subtract Poisson at (0,0), divide by $-2$ and by $\mu^2$ (Coulton et al. Eq. 29). Reject pairs that fail sanity checks (e.g. $q_{00}>0$ or triangle-inequality violation). Average (e.g. sigma-clipped) over flux. This gives a flux-independent correlation matrix.
- **Quadratic fit.** Fit the correlation at each lag $(i,j)$ as a function of $\mu^2$ and use the slope as the flux-independent correlation.
- **Use the a matrix.** Use the fitted **a** matrix from Eq. 2 directly as the pre-kernel (up to sign and tiling). This uses only the linear term.
- **Sample the full model at a flux (Broughton et al.).** Evaluate the full covariance model at a chosen flux $\mu_*$, form $\widetilde{C}_{ij}(\mu_*)$ with Eq. 4, and convert to $\widetilde{C}_{ij}/\mu_*^2$. This includes higher-order BFE at that flux.

The pre-kernel is usually stored in quarter form and tiled to a full $(2n+1)\times(2n+1)$ symmetric matrix.

**Step 4b: Enforce zero-sum.** Adjust the (0,0) element of the pre-kernel so that the full kernel sums to zero. This enforces charge conservation (Broughton et al.). Optionally, extrapolate the correlation beyond the measured range (e.g. power-law) before applying the zero-sum so that the finite range does not bias the sum.

**Step 5: Solve for the kernel $K$.** Solve the discrete Poisson equation $\nabla^2 K = \text{source}$ with the pre-kernel as the source and zero boundary conditions. Any standard solver (successive over-relaxation, multigrid, or FFT-based Poisson solver) works. The result is the **Brighter-Fatter kernel** $K$ per amplifier (or one per detector if averaged).

**Step 6: Validity checks (optional).** Before using the kernel, check: (1) values near the boundary are close to zero, (2) the center pixel is the minimum (kernel negative elsewhere), (3) off-center values are negative (or below a small fraction of $|K_{00}|$). Flag or exclude amplifiers that fail.

**Step 7: Apply the correction.** Correct the measured image $F$ using Eq. 5. Apply the correction **iteratively** until the added charge per iteration is below a threshold (e.g. 10 electrons; typically 2–3 iterations).

---

## Electrostatic (model-based) calibration

**When to consider it.** An alternative to the scalar kernel is to fit a **physical electrostatic model** to the measured **a matrix** and output **pixel boundary shifts** for a model-based corrector. This path is more involved but gives a single physical model that can be evaluated at any flux and, with conversion-depth weighting, at any filter.

**Idea.** The a matrix encodes the linear response of pixel area to accumulated charge. An electrostatic model (charge at different depths, E-field, boundary motion) predicts an a matrix from parameters such as detector thickness, pixel size, and charge-layer depths. Fitting the model to the measured a matrix yields a compact calibration. From the best-fit parameters you then compute boundary shifts (North, South, East, West) and area-change arrays, either at a single conversion depth (e.g. surface) or for a **conversion-depth distribution** (e.g. using filter throughput and silicon absorption, Rajkanan et al. 1979) for per-filter calibration.

**Steps in brief.** (1) Input: fitted a matrix (and its uncertainty) from the full covariance fit (Step 3); optionally average over good amplifiers. (2) Define an electrostatic model that predicts the a matrix from parameters (e.g. thickness, pixel size, $z_q$, $z_{\mathrm{sh}}$, $z_{\mathrm{sv}}$, scale/offset $\alpha$, $\beta$). (3) Fit by minimizing weighted residuals (e.g. least-squares with weights $1/\sigma^2_{ij}$). (4) From the best-fit parameters, compute boundary shifts and area-change arrays (and optionally per-filter via conversion depth). See the literature and existing implementations (e.g. LSST ip_isr/cp_pipe) for functional forms.

---

(summary-and-recommended-choices)=
## Summary and recommended choices

| Aspect | Recommendation |
|--------|----------------|
| **Data** | Flat pairs over a wide flux range; compute covariances from the difference image (Eq. 1; FFT method in Astier et al. Appendix A). |
| **PTC / covariance model** | Fit the full model (Eq. 2) to get gain, noise, and the a matrix; restrict the flux range to below turnoff (e.g. pCTI) to avoid contamination. |
| **Kernel construction** | Pre-kernel from averaged correlations, quadratic fit, a matrix, or **sampled full model at a flux** (Broughton et al.). Enforce **zero-sum** on the kernel. Solve $\nabla^2 K = \text{source}$. |
| **Correction** | Apply Eq. 5 iteratively until convergence. |
| **Electrostatic path** | Optional; fit electrostatic model to a matrix, then compute boundary shifts (and optionally per-filter via conversion depth). |

---

## References

- **Astier et al. (2019)** — "The shape of the Photon Transfer Curve of CCD sensors," *Astronomy & Astrophysics* **629**, A36; [arXiv:1905.08677](https://arxiv.org/abs/1905.08677). Variance deficit, full covariance model (Eq. 20), a matrix, FFT covariance method (Appendix A). Code: [bfptc](https://gitlab.in2p3.fr/astier/bfptc).
- **Broughton et al. (2023)** — "Mitigation of the Brighter-Fatter Effect in the LSST Camera," [arXiv:2312.03115](https://arxiv.org/abs/2312.03115). Covariance from flat pairs (Eq. 1), full model (Eq. 2), $\widetilde{C}_{ij}$ and Laplacian kernel (Eq. 3–4), correction formula (Eq. 5), flux-dependent BFE, sampling at a flux, zero-sum kernel, turnoffs and flux range.
- **Coulton et al. (2017/2018)** — Scalar kernel correction; [arXiv:1711.06273](https://arxiv.org/abs/1711.06273). Correlation and kernel (Eq. 22–29).
- **Downing et al. (2006)** — Charge conservation and covariance sum.
- **Rajkanan et al. (1979)** — Silicon absorption coefficient; used for conversion-depth distribution in filter-dependent electrostatic correction.

---

## Appendix: LSST Science Pipelines implementation

For readers who want to compare with or reuse the LSST implementation:

- **Covariance extraction:** Flat pairs and FFT covariance from the difference image: `cp_pipe` PTC extract tasks; `CovFastFourierTransform` in `lsst.cp.pipe.utils`.
- **PTC / full covariance fit:** Fit of Eq. 2 (Astier et al. Eq. 20): `PhotonTransferCurveSolveTask` in `cp_pipe.ptc` with `ptcFitType: "FULLCOVARIANCE"`.
- **Kernel construction:** Pre-kernel options, zero-sum, SOR solve: `BrighterFatterKernelSolveTask` in `lsst.cp.pipe.cpBfk`. Pipeline: `pipelines/_ingredients/cpBfkLSST.yaml`.
- **Electrostatic fit:** Electrostatic model fit to a matrix, boundary shifts, conversion depth: `ElectrostaticBrighterFatterSolveTask` in `lsst.cp.pipe.cpElectrostaticBF`. Pipeline: `pipelines/_ingredients/cpElectrostaticBFDistortionMatrixLSST.yaml`.
- **Correction application:** Kernel or electrostatic correction in ISR when the calibration is attached; see `lsst.ip.isr` and `doBrighterFatter`.

All of the above are in the [LSST Science Pipelines](https://pipelines.lsst.io) (packages `cp_pipe`, `ip_isr`).
