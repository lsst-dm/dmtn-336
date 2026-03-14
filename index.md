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
- $a_{ij}$ (units 1/e⁻) is the **fractional pixel area change** from accumulated charge—the linear part of the BFE. It is often called the $a$ matrix.
- $b_{ij}$ is a higher-order, time-evolving term; in practice it is often fixed to zero to stabilize the fit.
- $\otimes$ denotes discrete convolution. The terms in $\mu^2$ and $\mu^3$ are **higher-order BFE**; they are not negligible (e.g. ~20% of variance loss in Astier et al.; 15–30% in Broughton et al. at high flux).

Fitting this model to the measured $C_{ij}(\mu)$ over a chosen flux range yields gain $g$, noise $n_{ij}$, and the $a$ matrix (and optionally **b**) per amplifier. When **b** is fixed to zero, the fitted **a** matrix approximately satisfies $\sum a_{ij} \approx 0$ (zero-sum; Broughton et al.). The flux range should be restricted to below “turnoff” (e.g. pCTI turnoff) so that other effects near saturation do not contaminate the BFE estimate.

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

#### Correction

While we calculate the brighter-fatter kernel per-amplifier, we ultimately average the good amplifiers together to construct a detector-level kernel that ultimately gets used to correct a single detector image in any band.

The corrected is computed according to Gauss's law as:

$$
\hat{F}(\mathbf{x}) = F(\mathbf{x})
  + \frac{1}{2}\nabla\cdot\bigl[F(\mathbf{x})\,\nabla(K\otimes F(\mathbf{x}))\bigr].
$$

The factor 1/2 averages the BFE over the exposure since the first photon that falls into an empty pixel does not experience any BFE, while the last electron to fall into the pixel experiences double the effect. In practice the correction is applied iteratively: we update $F$ with the right-hand side, then repeat until the total amount of shifted charge across a single detector image in an iteration falls below a threshold (e.g. 10 electrons; typically 2–3 iterations if the calibration is good). 

#### Flux-Conserving Correction

An alternative implementation of the scalar kernel correction, selectable in ISR as `COULTON18_FLUX_CONSERVING`, uses the same kernel $K$ and the same underlying equation (Eq. 5) but applies the divergence in a pixelised, flux-conserving way. While we enforce the "zero sum" rule mentioned earlier, there is a limitation in the standard Coulton et al. formulation that allows the application of the normal correction to no longer conserve charge. The issue arises because the Coulton correction assumes the electric field represented by the kernel is discrete, when in reality, it is continuous. Derivatives of the convolution $K\otimes F$ during the correction happen discretely, and the discrete derivative stencil can in principle allow small flux leaks. The "flux conserving correction" computes a correction that gets added to the image to account for the difference between a discrete and continuous derivative. At each boundary it computes the flux that would have been transferred across a given edge and adds/subtracts that flux from the two adjacent pixels so that total flux is exactly conserved (donor pixel loses the same amount the receiving pixel gains). This pixelised flux transfer is applied along both axes and yields a correction array that is then added to the image. The result is improved correction of the cores of stars where flux conservation matters most. 

The flux-conserving version also handles image edges differently. The image to be corrected is padded (with zeros) by an amount equal to the size of the kernel, and the convolution is performed on the padded array so that wrap-around is avoided; the output is trimmed back to the original image dimensions. This avoids discontinuities at the image boundary (though it is still not a physical model for the edges, and may be less appropriate since we know that the electric fields at the edges of images are not the same as the centers of the images). In practice, we mask these edges as "EDGES" in the bit-mask plane of the exposure. The correction is applied iteratively with the same convergence criterion as the standard method (total absolute change below a threshold, or maximum iterations). The flux-conserving implementation is due to L. Miller (DM-38555) and is available in `ip_isr` as `fluxConservingBrighterFatterCorrection`.

**Note: The factor of 1/2 that averages the BFE correction assumes that the flux of charge into a pixel does not change during the integration of the exposure. In practice, a slowly-varying PSF during integration will cause this assumption to break. However, simulated tests show that the impact of varying PSF on the misestimation of the magnitude of the BFE is small, and is likely to be noise at the downstream point of PSF estimation and fitting across an image.**

**Note: The standard (non–flux-conserving) convolution uses SciPy's `fftconvolve` method; the flux-conserving path uses `afwMath.convolve` with padding as described above.**


### Vector Electrostatic Model Correction

The electrostatic correction was first implemented for Hyper-Suprime Cam (HSC) by Astier et al. (2023) to account for more complicated charge interactions in the pixel. Realistically, when an image is integrating, individual pixel boundaries can shift differently from one another. Flat field statistics can not provide enough information to derive the distortions to individual pixel boundaries without additional modeling from electrostatics (instead of an $a$ matrix one would have a vector of $a_N$, $a_S$, $a_E$, $a_W$ matrices). The electrostatic correction assumes that pixel boundary shifts arise from a physical electrostatic model: charge at different depths in the sensor produces electric fields that deflect drifting electrons and shift effective pixel boundaries. The measured $a$ matrix from the PTC encodes the linear response of pixel area to accumulated charge summed over all boundaries of the pixel (area lost by all N,S,E,W boundaries). We fit an electrostatic model to the measured $a$ matrix, parameterized by:

1. **Detector thickness:** The physical thickness of the silicon sensor, which influences the drift path and integration depth of electrons.
2. **Pixel size:** The physical width and height of a pixel, setting the spatial scale for charge deflection and field evaluation.
3. **Height of stored charge cloud ($z_q$):** The vertical (depth) location within the sensor where charge accumulates during exposure.
4. **Height of electric field saddle points ($z_{\mathrm{sh}}$, $z_{\mathrm{sv}}$):** The heights at which the electric field saddle points occur on the horizontal ($z_{\mathrm{sh}}$) and vertical ($z_{\mathrm{sv}}$) pixel boundaries, affecting how the transverse field integrates across boundaries.
5. **Charge cloud shape:** The assumed geometry of the stored charge, typically modeled as a flat rectangle characterized by length $a$ and width $b$.
6. **Scale/offset ($\alpha$, $\beta$):** Fitting parameters that adjust the overall scale ($\alpha$) and offset ($\beta$) of the model to best match observations.

The BFE calibration is therefore not a kernel, but a **model** that can be evaluated to compute $a_N$, $a_S$, $a_E$, and $a_W$ (and hence the amount of charge lost/gained at each pixel boundary). The model can be averaged over a conversion-depth distribution (filter throughput × silicon absorption, Rajkanan et al. 1979) to produce a per-filter calibration. The output is an **electrostatic BFE distortion matrix**: fitted physical parameters plus pixel boundary shifts (aN, aS, aE, aW) and area-change arrays, optionally per filter.

#### Algorithm

Inputs are the average $a$ matrix per detector (and its uncertainty estimated as the elementwise scatter over all "good" amplifiers) from the full covariance PTC fit. The measured $a$ encodes the linear relation between pixel area change and charge content (Astier & Regnault 2023, Eq. 1):

$$
\frac{\delta A}{A} = g \sum_{i,j} a_{ij}\, Q_{ij},
$$

where $A$ is the pixel area, $\delta A$ its change, $Q_{ij}$ the charge (in electrons or ADU with gain $g$), and $a_{ij}$ the sensor coefficients (el$^{-1}$). By parity, $a_{ij} = a_{|i||j|}$; for uniform illumination the **sum rule** $\sum_{i,j} a_{ij} = 0$ holds. The full $a$ matrix is the sum of area changes at the four pixel boundaries (N, S, E, W).

Given the set of physical electrostatic model parameters, we compute the boundary shifts at each pixel edge by integrating the transverse electric field along the drift path (from the relevant saddle height $z_{\mathrm{sh}}$ or $z_{\mathrm{sv}}$ to the integration depth $z_f$). With $\alpha$ the overall scale from the fit (Astier & Regnault 2023):

$$
a_N = -\alpha \int_{z_{\mathrm{sh}}}^{z_f} E_y\,\mathrm{d}z \quad \text{(North)}, \qquad
a_S,\, a_E,\, a_W \quad \text{analogously},
$$

with the field evaluated at the pixel boundary. The model computes the E-field from the charge distribution (Coulomb + image charges), integrates via Gauss's Law to get $a_N$, $a_S$, $a_E$, $a_W$, and sums them elementwise to get the predicted $a$ matrix. We update the parameters in a least-squares fit to the measured $a$ matrix over a chosen pixel range (fit range ≤ total computed PTC covariance range). 

This mode allows us to optionally compute the chromatic component of the BFE, because we could choose to integrate the electric fields up to the top of the silicon sensor (as if all photons converted to electrons at the top) or we could assume an incident flat SED of light and use an empirical model the of the conversion depth in silicon @ 173K (operating temperature of the LSSTCam CCDs). In this way we can compute the model for the filter that the PTC data was taken in. Additionally, since the calibration is just the electrostatic model, so we can compute a per-filter correction for any image in any filter we want to apply the BF correction to. In other words, we can compute an electrostatic BF calibration in any filter, even if we only have PTC data in one filter.

We use an empirical model for the conversion depth in silicon at a given temperature from Rajkanan et al. (1979), which was also used for the HSC implementation.
Python functions used to compute the Rajkanan et al. model is included in the released software.

To save time in the instrument signature removal portion, we can pre-compute the BF distortion matrices per filter and have them ready-to-go to apply to any exposure taken in any of those filters.

We expect that the BFE is weaker in the redder bands, and we find that $a_{00}$ in y band is 6\% weaker than in e.g. u band.

#### Validity checks

We assess the electrostatic calibration using fit quality and physical plausibility. Typical checks include: reduced $\chi^2$ and residuals of the model vs the measured a matrix; parameter uncertainties from the fit covariance; and sanity of the fitted geometry (e.g. charge depths within the detector thickness). Optionally we compare the predicted a matrix from the best-fit model to the measured a matrix over the fit range. Amplifiers or detectors with failed fits or implausible parameters can be flagged or excluded from the calibration product. There is no single universal acceptance threshold; projects may define criteria (e.g. $\chi^2$/dof below a limit, or parameter errors below a fraction of the parameter value) for deployment.

#### Correction

The electrostatic correction is applied in instrument signature removal (ISR). The corrector uses the boundary-shift arrays (aN, aS, aE, aW)—the integrated transverse E-field along the drift path from the electrostatic fit (Algorithm above)—and area-change arrays to redistribute flux according to the modeled pixel boundary motion (or equivalently to apply the deflection field implied by the electrostatic solution). If per-filter corrections are turned on, the calibration can compute on the fly the distortion matrices needed to correct the exposure in its native filter. the calibration may be chosen by filter so that the per-filter boundary shifts (when available) are used; otherwise the single default (e.g. surface-conversion) calibration is used.

From the boundary-shift arrays, which are computed in quadrants, we compute two tiled ($(2n+1)\times(2n+1)$) matrices, to model the vertical pixel boundary shifts and the horizontal pixel boundary shifts. Note that these are no longer required to be symmetric, which is more realistic!

This correction is not performed iteratively. Instead of having to do multiple convolutions until a convergence condition is met. We simply perform exactly 2 convolutions with each of the vertical and horizontal boundary shifts. 


## Comparison of the two corrections

The scalar kernel correction (Coulton et al. / Broughton et al.) and the electrostatic (model-based) correction both consume the same PTC-derived inputs (covariances and a matrix) but differ in their assumptions, outputs, and how they are validated and applied, as well as their overall performance.

**Scalar kernel correction**

- **Main assumption:** The deflection field can be written as the gradient of a scalar kernel $K$; the BFE-induced covariance is $-\mu^2 \nabla^2 K$. The kernel is often treated as flux-independent (or sampled at one flux to absorb higher-order terms). Does not rely on electrostatics (non-parametric).
- **Input from PTC:** Raw covariances and/or the fitted a matrix (or full covariance model sampled at a flux).
- **Output:** A convolution kernel $K$ per amplifier (always applied per detector).
- **Chromatic / per-filter:** Single kernel per detector unless multiple calibrations are produced for different bands.
- **Application:** Correct the image with Eq. 5 (iterative convolution with $K$). No fixed number of convolution operations.
- **Acceptance / validity:** Kernel checks: edge values ≈ 0, center is minimum, off-center values negative; iterative correction convergence (e.g. total shifted charge below a threshold per iteration). Reject or flag amps that fail.
- **Limitations:** Flux-independent kernel is an approximation (higher-order BFE); sampling at one flux improves this but does not remove the approximation. In addition, it cannot account for differences in pixel boundaries and non-zero curl of the pixel boundary displacements. Flat-field–derived kernel can perform differently on stars.

**Electrostatic (model-based) correction**

- **Main assumption:** Pixel boundary shifts arise from a physical electrostatic model (charge layers, E-fields, detector geometry). The measured a matrix is fit by this model; the model can then be evaluated at any flux and at any conversion depth.
- **Input from PTC:** Fitted a matrix (and its uncertainty) only.
- **Output:** Fitted physical parameters plus pixel boundary shifts (aN, aS, aE, aW, area-change arrays), computed per-detector, and optionally per filter via conversion-depth weighting.
- **Chromatic / per-filter:** Naturally supports per-filter calibration via conversion-depth distribution (filter throughput × silicon absorption).
- **Application:** Correct using the boundary-shift or area-change arrays in a model-based corrector (pixel boundaries or flux redistribution).
- **Acceptance / validity:** Fit quality (e.g. $\chi^2$, residuals), parameter uncertainties.
- **Limitations:** Requires a concrete electrostatic model and more tuning (parameter bounds, conversion-depth model). More complex pipeline and validation.

In short: the kernel correction is empirical and easy to deploy; the electrostatic correction is model-driven and more flexible for chromatic and multi-flux use at the cost of complexity and validation effort.

**Note: Before the electrostatic BF correction was implemented, we had BF correction turned off for the Prompt Processing (PP) pipeline because having some images take 1-2 iterations and some images take 3+ iterations of the kernel to be corrected meant that images finished processing non-uniformly, which made it difficult for processing difference imaging sources. With the electrostatic BF correction, we only have a fixed number (2) convolution operations to do, so we were able to turn on BF correction for PP.**


## Algorithms used in Data Releases

- DP1 used the kernel-based implementation with no color corrections and no flux-conserving correction. The PTC data used to compute the calibration was sparsely sampled in flux space, and was only computed for i-band. We observed a distinct over-correction of the BFE in DP1.

- DP2 used the electrostatic implementation with no color corrections (assumes all photons convert near the top of the silicon chip).

We found in DP2 that the kernel-based implementation was useful for in-lab artificial star tests in the LSSTCam focal plane, however we found the correction over-corrected by more than 10\% (!) in on-sky images. We then implemented and switched to the elesctrostatic correction and are now meeting the LSST Y10 required threshold for the slope of residual PSF size vs magnitude space for DP2. We have also found that the elesctrostatic algorithm works so well that we were able to use that fact that shape measurement algorithms (HSM) are sensitive to residual sky background to actually characterize the amount of residual sky background oversubtraction in the galactic plane regions by comparing the observed size vs magnitude plots to the expected ones with perfect BF calibration, thereby recovering accurate PSF photometry in dense stellar regions. 

**Note: We are currently running internal tests of the filter correction and might turn this on for DR1.**

## References

- **Astier et al. (2019)** — "The shape of the Photon Transfer Curve of CCD sensors," *Astronomy & Astrophysics* **629**, A36; [arXiv:1905.08677](https://arxiv.org/abs/1905.08677). Variance deficit, full covariance model (Eq. 20), a matrix, FFT covariance method (Appendix A). Code: [bfptc](https://gitlab.in2p3.fr/astier/bfptc).
- **Broughton et al. (2023)** — "Mitigation of the Brighter-Fatter Effect in the LSST Camera," [arXiv:2312.03115](https://arxiv.org/abs/2312.03115). Covariance from flat pairs (Eq. 1), full model (Eq. 2), $\widetilde{C}_{ij}$ and Laplacian kernel (Eq. 3–4), correction formula (Eq. 5), flux-dependent BFE, sampling at a flux, zero-sum kernel, turnoffs and flux range.
- **Coulton et al. (2017/2018)** — Scalar kernel correction; [arXiv:1711.06273](https://arxiv.org/abs/1711.06273). Correlation and kernel (Eq. 22–29).
- **Downing et al. (2006)** — Charge conservation and covariance sum.
- **Rajkanan et al. (1979)** — Silicon absorption coefficient; used for conversion-depth distribution in filter-dependent electrostatic correction.
