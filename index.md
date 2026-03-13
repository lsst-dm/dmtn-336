# Calibration and Correction of the Brighter-Fatter Effect in CCDs

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

The factor $1/2$ averages the BFE over the exposure. In practice the correction is applied **iteratively**: we update $F$ with the right-hand side, then repeat until the total amount of shifted charge across a single detector image in an iteration falls below a threshold (e.g. 10 electrons; typically 2–3 iterations if the calibration is good).

**Note: Because the true covariance depends on $\mu$, there is no single “correct” flux at which to evaluate $\widetilde{C}_{ij}$ to build $K$. Broughton et al. recommend **sampling** the full covariance model at a chosen flux (e.g. where correction matters most) so that higher-order terms are included in the kernel, though there is currenly no stable implementated pipeline for this in the software.**


(procedure-overview)=
## Calibration Generation

All BFE calibration tasks take as input a single PTC data product, produced per-detector in a single filter band.

### Scalar Kernel Correction (Coulton et al. 2019)

From the PTC data we form a single correlation matrix per amplifier that will serve as the source term in $\nabla^2 K = \text{source}$. Common options:

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
