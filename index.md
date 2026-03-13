# Calibration and Correction of the Brighter-Fatter Effect in LSSTCam

```{abstract}
This living technote will be updated with descriptions of all the algorithms for Brighter-Fatter calibration and correction in the LSST DM Science Pipelines. It is intended to serve as a reference document, and it will include detailed descriptions of the algorithms used and deployed for specific data previews and data releases. It will also contain the verification and acceptance criteria used for calibration deployment.
```


(what-is-the-brighter-fatter-effect)=
## What is the Brighter-Fatter effect?

Correlations between neighboring pixels have been shown to arise at high signal levels in charge-coupled devices (CCDs). This is attributed to and has been accurately modeled by considering that captured photocharges in the potential wells distort the boundaries of pixels. of pixels produce significant transverse electric fields on incoming photocharges. During the integration of an exposure, photo-electrons deflect into neighboring pixels in reaction to quasistatic changes in effective pixel area from the accumulated charges in the potential wells of the pixels, causing the measured light profile of a bright source to differ from that source’s intrinsic surface brightness profile. This effect, dubbed the brighter-fatter effect (BFE), broadens intrinsic surface brightness profiles. The magnitude of the BFE depends on the surface brightness profile of the source itself and thus cannot be modeled solely by its location on a sensor. The consequence is that the BFE breaks the critical assumption of experimental imaging analysis that the pixels are independent light collectors that perfectly obey Poisson statistics.

Flat-field images have been shown to exhibit non-linear behavior at high signal levels due to the dynamic behavior of charges collecting in the potential wells of pixels. The variance of a uniformly illuminated flat does not scale linearly with the mean flux as would be predicted from simple shot-noise statistics. Instead, we observe variance deficit at higher signal levels. This is because the counts in nearby pixels covary in a way that grows with flux and falls off with distance.

The Brighter-Fatter corrections implemented in the LSST Data Management Science Pipelines come in two flavors that can be swapped in/out in instrument signature removal (ISR), but require two separate calibrations.

1. Scalar Kernel Correction: based off of Coulton et al. (2019)
2. Vector Electrostatic Model Correction: based off of Astier et al. (2023)

Both correction are derived from the same input measurements, but have different assumptions and configuration parameters.

All calibrations are defined data products that are released with data previews and data releases alongside downstream science data products and are data Butler accessible from all repositories.

**Key characteristics of the BFE**

- The strength of the BFE varies non-linearly with signal level due to complicated "feedback" mechanisms between the collected charges (including longitudinal fields that increase the drift time of the electrons); we refer to these effects as "higher-order BF".
- The BFE may vary per detector and per amplifier, however across the LSSTCam focal plane the same detector species have actually quite similar BF statstics. We currently compute a single BF calibration per-detector, however we are considering in the future implementing a single calibration per physical detector type (ITL-type vs E2V-type).
- The BFE is chromatic: charges are stored at the bottom of the potential well, which corresponds to a few microns above the bottom of the pixel; most of the deflection of an electron drifting to the bottom of the potential well therefore occurs in the last several microns. Bluer photon convert to electrons closer to the top of the silicon than redder photons (photons in bands u$\rightarrow$r mostly convert near the top and are therefore achromatic and the same calibration can be used for these bands, but photons in bands i$\rightarrow$y vary drastically and each require different calibrations.)
- The BFE is stronger in one (+y parallel) direction than in the other (+x serial) direction.

**Previous literature on the BFE** 

Key studies establishing the theoretical and observational basis for the Brighter-Fatter Effect include:

- **Downing et al. (2006)** — Identified covariance sum rules and charge conservation in CCDs.
- **Holland et al. (2014)** — Presented physical models and sensor effects influencing pixel correlations.
- **Lage et al. (2017)** — Provided detailed modeling of electric fields and boundary shifts relevant to the BFE.
- **Astier et al. (2019)** [*A&A* **629**, A36; [arXiv:1905.08677](https://arxiv.org/abs/1905.08677)] — Showed that variance deficit and pixel correlations are linked and provided an analytical model for both. The observed behavior is consistent with charge being redistributed during drift in the sensor.
- **Broughton et al. (2024)** [arXiv:2312.03115] — Demonstrated on LSST Camera data that the effect **depends on flux** (higher-order terms matter), so a single flux-independent correction kernel is an approximation; they recommend building the kernel from the covariance model evaluated at a chosen flux and enforcing **zero-sum** on the kernel for charge conservation.


## What do we measure?

To calibrate the BFE we need the covariance between pixel counts as a function of mean signal level. Those covariances are measured from pairs of flat-field exposures at the same nominal illumination.

The photon transfer curve take two flats $F_1$ and $F_2$ at the same flux level. Their difference $F_1 - F_2$ removes the fixed pattern and leaves mainly photon noise and any BFE-induced correlations. The covariance between a central pixel and a neighbor at lag $(i,j)$ is defined (Broughton et al. 2023, Eq. 1) as:

$$
C_{ij} = \frac{1}{2}\,\mathrm{Cov}\bigl(F_1(\mathbf{x})-F_2(\mathbf{x}),\;
F_1(\mathbf{x}')-F_2(\mathbf{x}')\bigr), \qquad \mathbf{x}' = (i,j).
$$

The factor 1/2 comes from the pair-difference construction. By repeating this for many flat pairs at different flux levels, we obtain the variance and covariances as a function of mean signal ($\mu$), which the Photon Transfer curve (PTC). The observed flattening of the variance at high flux is the variance deficit (roughly 30\% loss at a signal level of $0.75\times$ saturation level). Summing covariances over all lags recovers the lost variance (charge conservation; Downing et al. 2006). In practice, covariances are computed from the difference image using an FFT-based method (Astier et al. 2019, Appendix A). From this information, we can fit an analytic model to this PTC so we can extract gain, noise, and the BFE-related parameters that are use for calibration. If pixels were independent and Poissonian, the variance $C_{00}$ would be linear in the mean flux $\mu$. 

**Quantifying the BFE**

Astier et al. (2019) Eq. 20 and Broughton et al. (2023) Eq. 2 parameterize the covariance at mean signal $\mu$ (in ADU) as:

$$
C_{ij}(\mu) = \frac{\mu}{g}\Bigl[\delta_{i0}\delta_{j0}
  + a_{ij}\,\mu g
  + \frac{2}{3}[\mathbf{a}\otimes\mathbf{a}+\mathbf{ab}]_{ij}(\mu g)^2
  + \frac{1}{6}[\ldots]_{ij}(\mu g)^3 + \cdots\Bigr] + \frac{n_{ij}}{g^2}.
$$

- $g$ is the gain (electrons per ADU). $n_{ij}$ is the noise covariance matrix (read noise, etc.).
- $a_{ij}$ (units 1/e⁻) is the **fractional pixel area change** from accumulated charge—the linear part of the BFE. It is often called the **a matrix**.
- $b_{ij}$ is a higher-order, time-evolving term; in practice it is often fixed to zero to stabilize the fit.
- $\otimes$ denotes discrete convolution. The terms in $\mu^2$ and $\mu^3$ are **higher-order BFE**; they are not negligible (e.g. ~20% of variance loss in Astier et al.; 15–30% in Broughton et al. at high flux).

Fitting this model to the measured $C_{ij}(\mu)$ over a chosen flux range yields gain $g$, noise $n_{ij}$, and the **a matrix** (and optionally **b**) per amplifier. When **b** is fixed to zero, the fitted **a** matrix approximately satisfies $\sum a_{ij} \approx 0$ (zero-sum; Broughton et al.). The flux range should be restricted to below “turnoff” (e.g. pCTI turnoff) so that other effects near saturation do not contaminate the BFE estimate.

**Note: that this equation is derived from a Taylor expansion of the photon transfer curve, and does not derive from electrostatics.**

**Note: in practice we found that approximately 500 flat pairs (or 1000 exposures) gives us the statistics needed to fit the BFE coefficients out to a lag of 10 pixels.**

**Note: the BFE as measured in flat fields has been shown to be different between flat-field images and the magnitude dependence of size/shape of PSF stars because fundamentally the charge distributions per pixel are vastly different between the two images, and it is believed that the increased charge gradients produce a larger BFE (see Broughton et al. (2024) for more details).**

(procedure-overview)=
## Calibration Generation

All BFE calibration tasks take as input a single PTC data product, produced per-detector in a single filter band.

### Scalar Kernel Correction (Coulton et al. 2019)

#### Calibration generation

Coulton et al. (2018) model the deflection field as the gradient of a scalar kernel, $K$. In that picture, the part of the covariance that is due to the BFE (after subtracting Poisson and read noise) is proportional to the Laplacian of $K$. We define the deficit covariance as:

$$
\widetilde{C}_{ij} = C_{ij} - \Bigl(\frac{\mu}{g}\delta_{i0}\delta_{j0} + \frac{n_{ij}}{g^2}\Bigr).
$$

If we assume that there exists a "kernel" which is a Green's function that represents something like a the dimensionless electric field of the deflecting charge, then we can define the kernel from the measured covariances by solving the Poisson equation (Broughton et al. 2023, Eq. 3–4)

$$
\widetilde{C}_{ij} = -\mu^2 \nabla^2 K_{ij}.
$$

To compute the kernel, we measure $\widetilde{C}_{ij}$ and solve the Poisson equation to get the kernel $K$. In practice, $\widetilde{C}_{ij}$ is a function of flux because the BFE varies in some complicated way with signal level, so theoretically, we can measure the covariances and any flux that we want ($\nabla^2 K = \text{source}$). We call the "source" the "pre-kernel".

From the PTC data we form a single correlation matrix per amplifier that will serve as the source term in $\nabla^2 K = \text{source}$. Three options exist to compute the source of the pre-kernel:

1. **Average of measured correlations.** At each flux level, convert covariances to electrons, subtract Poisson at (0,0), divide by $-2$ and by $\mu^2$ (Coulton et al. Eq. 29). Reject pairs that fail sanity checks (e.g. $q_{00}>0$ or triangle-inequality violation). Average (e.g. sigma-clipped) over flux. This gives a flux-independent correlation matrix.

2. **Quadratic fit.** Fit the correlation at each lag $(i,j)$ as a function of $\mu^2$ and use the slope as the flux-independent correlation.

3. **Use the a matrix.** Use the fitted **a** matrix from Eq. 2 directly as the pre-kernel (up to sign and tiling). This uses only the linear term. This is effectively measuring the amplitude of the BFE in the limit of zero signal.

4. **Sample the full model at a flux (Broughton et al.).** Evaluate the full covariance model at a chosen flux $\mu_*$, form $\widetilde{C}_{ij}(\mu_*)$ with Eq. 4, and convert to $\widetilde{C}_{ij}/\mu_*^2$. 

In the implementation, we have the option of subtracting off long-range covariances in the covariance matrix. In in-lab testing of the LSSTCamera, we found that "weather" inside the camera cryostat due to an air leak caused long-range correlations that average out over a detector. We have the option of fitting a 2D linear model to these long range correlations (lag of 20-40px) and subtracting them off. This issue has since been fixed, and we no longer subtract these long range correlations as we have now measured them to be consistent with zero and unncessary.

The pre-kernel is solved in a single $(+x, +y)$ quadrant and tiled to a $(2n+1)\times(2n+1)$ symmetric matrix. 

To computationally solve the discrete Poisson equation $\nabla^2 K = \text{source}$ with the pre-kernel as the source, we enforce zero boundary conditions (kernel must be zero at the edges). Any standard solver (successive over-relaxation, multigrid, or FFT-based Poisson solver) works. 

After computing the kernel, we have the option to adjust the (0,0) element of the pre-kernel so that the correlations sum to zero. This enforces charge conservation, and we find that the Coulton et al. (2018) algorithm is unstable enough that it is necessary to turn this configuration on. Optionally, we can fit a model to the central $n \times n$ pixels of the kernel and integrate the measured correlations + model beyond the measured range (e.g. power-law) before applying the zero-sum so that the finite range does not bias the sum.

#### Validity checks
A valid kernel satisfies three criteria:
1. Values near the boundary are close to zero, 
2. The center pixel is the minimum,
3. Off-center values are negative (or below a small, configurable fraction of $|K_{00}|$). 

We flag and exclude amplifiers that fail as "bad amps" in the brighter-fatter kernel calibration data product.

#### Apply the correction

While we calculate the brighter-fatter kernel per-amplifier, we ultimately average the good amplifiers together to construct a detector-level kernel that ultimately gets used to correct a single detector image in any band.

The corrected is computed according to Gauss's law as:

$$
\hat{F}(\mathbf{x}) = F(\mathbf{x})
  + \frac{1}{2}\nabla\cdot\bigl[F(\mathbf{x})\,\nabla(K\otimes F(\mathbf{x}))\bigr].
$$

The factor 1/2 averages the BFE over the exposure since the first photon that falls into an empty pixel does not experience any BFE, while the last electron to fall into the pixel experiences double the effect. In practice the correction is applied iteratively: we update $F$ with the right-hand side, then repeat until the total amount of shifted charge across a single detector image in an iteration falls below a threshold (e.g. 10 electrons; typically 2–3 iterations if the calibration is good). 

**Note: The factor of 1/2 that averages the BFE correction assumes that the flux of charge into a pixel does not change during the integration of the exposure. In practice, a slowly-varying PSF during integration will cause this assumption to break. However, simulated tests show that the impact of varying PSF on the misestimation of the magnitude of the BFE is small, and is likely to be noise at the downstream point of PSF estimation and fitting across an image.**

**Note: The convolution operation happens using SciPy's `fftconvolve` method, which internal tests showed gives us our required speed performance with no loss in precision to 64-bt precision compared to a brute force approach.**


### Vector Electrostatic Model Correction

The electrostatic (vector) correction assumes that pixel boundary shifts arise from a physical model: charge at different depths in the sensor produces electric fields that deflect drifting electrons and shift effective pixel boundaries. The measured **a matrix** from the PTC (Eq. 2) encodes the linear response of pixel area to accumulated charge. We fit an electrostatic model—parameterized by detector thickness, pixel size, charge-layer depths ($z_q$, $z_{\mathrm{sh}}$, $z_{\mathrm{sv}}$), and scale/offset ($\alpha$, $\beta$)—to the measured a matrix by minimizing weighted residuals (e.g. least-squares with weights $1/\sigma^2_{ij}$). The model integrates E-fields over pixel boundaries and over charge depth; it can be averaged over a **conversion-depth distribution** (filter throughput × silicon absorption, Rajkanan et al. 1979) to produce a per-filter calibration. The output is an **electrostatic BFE distortion matrix**: fitted physical parameters plus pixel boundary shifts (aN, aS, aE, aW) and area-change arrays, optionally per filter.

#### Calibration generation

Inputs are the fitted **a matrix** (and its uncertainty) per amplifier from the full covariance PTC fit. We optionally average the a matrix over good amplifiers and compute an uncertainty matrix; bad amplifiers are excluded. We then fit the electrostatic model to the mean a matrix over a chosen pixel range (fit range ≤ PTC covariance range). Initial parameter values and which parameters are varied are set in configuration (e.g. thickness and pixel size often fixed from known hardware; charge depths and scale/offset varied). The fit is performed with a standard minimizer (e.g. least-squares); if the first fit fails, the implementation may retry with looser tolerances. From the best-fit parameters we compute **boundary shifts** (North, South, East, West) and **area-change** arrays. When per-filter correction is enabled, we compute a conversion-depth distribution for each filter (flat SED over the filter band, silicon absorption vs wavelength) and integrate the electrostatic model over that distribution to obtain boundary shifts and area-change arrays per filter. The result is stored as the electrostatic BFE distortion matrix calibration (one per detector, with optional per-filter arrays).

#### Validity checks

We assess the electrostatic calibration using fit quality and physical plausibility. Typical checks include: reduced $\chi^2$ and residuals of the model vs the measured a matrix; parameter uncertainties from the fit covariance; and sanity of the fitted geometry (e.g. charge depths within the detector thickness). Optionally we compare the predicted a matrix from the best-fit model to the measured a matrix over the fit range. Amplifiers or detectors with failed fits or implausible parameters can be flagged or excluded from the calibration product. There is no single universal acceptance threshold; projects may define criteria (e.g. $\chi^2$/dof below a limit, or parameter errors below a fraction of the parameter value) for deployment.

#### Apply the calibration

The electrostatic correction is applied in instrument signature removal (ISR) when the electrostatic BFE distortion matrix calibration is attached and BFE correction is enabled. The corrector uses the boundary-shift arrays (aN, aS, aE, aW) and area-change arrays to redistribute flux according to the modeled pixel boundary motion (or equivalently to apply the deflection field implied by the electrostatic solution). For a given exposure, the calibration may be chosen by filter so that the per-filter boundary shifts (when available) are used; otherwise the single default (e.g. surface-conversion) calibration is used. The correction is applied to the detector image before downstream processing (e.g. source detection and measurement). Iteration may be used (analogous to the kernel correction) until the applied shift converges.



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

## Comparison of the two corrections

The scalar kernel correction (Coulton et al. / Broughton et al.) and the electrostatic (model-based) correction both consume the same PTC-derived inputs (covariances and a matrix) but differ in their assumptions, outputs, and how they are validated and applied.

**Scalar kernel correction**

- **Main assumption:** The deflection field can be written as the gradient of a scalar kernel $K$; the BFE-induced covariance is $-\mu^2 \nabla^2 K$. The kernel is often treated as flux-independent (or sampled at one flux to absorb higher-order terms).
- **Input from PTC:** Raw covariances and/or the fitted a matrix (or full covariance model sampled at a flux).
- **Output:** A convolution kernel $K$ per amplifier (or per detector).
- **Chromatic / per-filter:** Single kernel per detector (or amp) unless multiple calibrations are produced for different bands.
- **Application:** Correct the image with Eq. 5 (iterative convolution with $K$). Simple to plug into ISR.
- **Acceptance / validity:** Kernel checks: edge values ≈ 0, center is minimum, off-center values negative; iterative correction convergence (e.g. total shifted charge below a threshold per iteration). Reject or flag amps that fail.
- **When to prefer:** Straightforward to implement and run; well suited when a single effective kernel per detector is acceptable and chromatic effects are modest (e.g. u–r).
- **Limitations:** Flux-independent kernel is an approximation (higher-order BFE); sampling at one flux improves this but does not remove the approximation. Flat-field–derived kernel can perform differently on stars (Broughton et al.).

**Electrostatic (model-based) correction**

- **Main assumption:** Pixel boundary shifts arise from a physical electrostatic model (charge layers, E-fields, detector geometry). The measured a matrix is fit by this model; the model can then be evaluated at any flux and at any conversion depth.
- **Input from PTC:** Fitted a matrix (and its uncertainty) only.
- **Output:** Fitted physical parameters plus pixel boundary shifts (aN, aS, aE, aW, area-change arrays), optionally per filter via conversion-depth weighting.
- **Chromatic / per-filter:** Naturally supports per-filter calibration via conversion-depth distribution (filter throughput × silicon absorption).
- **Application:** Correct using the boundary-shift or area-change arrays in a model-based corrector (pixel boundaries or flux redistribution).
- **Acceptance / validity:** Fit quality (e.g. $\chi^2$, residuals), parameter uncertainties, and physical plausibility of the fitted geometry. Optionally compare predicted vs measured a matrix.
- **When to prefer:** When per-filter or physically interpretable calibration is desired, or when extrapolation to flux/conversion depth not covered by flats is needed.
- **Limitations:** Requires a concrete electrostatic model and more tuning (parameter bounds, conversion-depth model). More complex pipeline and validation.

In short: the kernel correction is empirical and easy to deploy; the electrostatic correction is model-driven and more flexible for chromatic and multi-flux use at the cost of complexity and validation effort.

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
