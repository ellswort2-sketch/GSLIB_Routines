# Field Kriging — KT3D browser adaptation

## Quick start for soil or yield data

1. Save a normal table as CSV UTF-8. Row 1 contains column names; later rows contain observations. GEO-EAS is also accepted using its title/count/name-lines/whitespace format, illustrated in the page.
2. Select file format, load observations, and choose X, Y, optional Z, and the variable. Local X/Y are suitable if their units are verified. If Z is omitted, observations and uploaded targets use the first Z grid center, matching KT3D (normally set this to 0 for a 2D field). Use projected coordinates for geographic data. Do not use latitude/longitude degrees as meters.
3. Enter a fitted variogram model. Experimental semivariances are not model parameters. For an isotropic spherical model, enter the same range in all three directions. Total sill = nugget + partial sill(s). The supplied COVA3 uses exp(−3h/a) for exponential and exp(−9(h/a)²) for Gaussian; see the in-page model explanation.
4. Set search radii, search angles, and minimum/maximum neighbors. Azimuth is 0 north–south and 90 east–west. Search orientation and range are independent of the variogram model.
5. First try leave-location-out cross-validation with 1 × 1 × 1 support. Examine mean error and RMSE, not just the map. Standardized RMSE uses nodes with positive kriging standard deviation.
6. Select regular grid. Choose grid spacing; click Fit XY grid to sample bounds, then verify origin and counts. Grid origins are cell centers. X varies fastest, followed by Y then Z.
7. Leave discretization at 1 × 1 × 1 for point estimates. Increasing it approximates block averages over the grid cell dimensions, with at most 64 discretization points. To set dimensions for uploaded nodes, set grid widths before switching target mode.
8. Calculate, view estimates/variance/standard deviation, and download results CSV and Settings JSON. Import CSV to QGIS as points using x and y and the original CRS. Clip to your field boundary in QGIS. There is no automatic boundary mask or CRS transformation.

## Features

- Continuous KT3D methods: 0 simple kriging, 1 ordinary / polynomial-drift kriging, 2 source-compatible nonstationary SK, and 3 external drift.
- 2D or 3D; up to four nested spherical, exponential, Gaussian, power, or hole-effect structures, with independently rotated anisotropy.
- Point and discretized rectangular-block covariance calculations, including KT3D's block nugget treatment.
- Ellipsoidal searches, maximum/minimum samples, optional per-octant limits, sorted by anisotropic search distance.
- Regular grids, uploaded targets (KT3D jackknife convention), and leave-location-out cross-validation.
- Nine polynomial drift flags and source-style trend-only estimation.
- KT3D.PAR import, CSV/GEO-EAS input, downloadable example, interactive maps and numeric tables.
- Results CSV, GEO-EAS, settings JSON, map PNG.
- Worker-based computation with progress and cancellation. Browser limits: 20,000 retained observations; 100,000 targets; 64 neighbors; 64 block discretization points. Large settings can still be slow.

## Advanced methods

Type 3 requires numeric secondary data at samples and targets. Grid secondary CSV must have one row per grid node in X-fastest order; choose its secondary column. Target uploads have a secondary-field selector. Block secondary values represent the secondary mean over the block.

Type 2 deliberately follows the supplied KT3D estimation loop: each target's secondary value is the local SK mean used for ALL neighbors. It does not subtract individual sample means before interpolation. Sample secondary values are not required for type 2. This distinction matters when comparing with other locally varying-mean implementations.

Power follows the supplied COVA3 exactly: total zero-distance covariance includes 999 per power structure, and each nonzero-distance power term contributes Cmax − coefficient × h^exponent. With a nugget or other structures this differs from using a separate 999 offset for that term. The major-range slot is the exponent, 0 < exponent ≤ 2; minor/exponent and vertical/exponent are anisotropy ratios. The “partial sill” is a coefficient; no finite sill exists. The hole-effect cosine model is not generally valid in arbitrary multidimensional settings. Negative variances are explicitly flagged rather than hidden.

Cross-validation and uploaded-target modes exclude every observation with Manhattan coordinate distance < 1e-6 from the target, matching KT3D's location exclusion. This includes colocated duplicates. Validation uses point support. Imported koption 0/1/2 maps to grid/cross-validation/uploaded targets. A secondary file in the PAR is not opened automatically. Original debug paths are informational.

## Compatibility and limits

This is a JavaScript reimplementation of the supplied KT3D.F version 2.000 continuous-variable equations, not a compilation of the original executable. The GSLIB library source has now been supplied and reviewed. All ten required routines are included in reference/gslib/. The browser implements covariance, rotation, neighborhood search and a pivoted linear solver. See the validation notes below for archived and freshly compiled Fortran checks. Full binary equivalence for arbitrary inputs has not been certified.

- Double precision replaces the source's mixed real/double precision.
- KD-tree search replaces GSLIB's superblocks; exact distance ties use input order. Octant caps use the exact unrotated coordinate-plane rules in SRCHSUPR. Caps are applied before validation excludes colocated samples, matching the source.
- Constraints are rescaled for numerical stability; mathematically equivalent rescaling may produce rounding differences. Small pivots are treated as singular; no automatic regularization is added.
- Original single-neighbor SK special case is retained: estimate equals the observation, with variance Cbb − 2Cb + C(0). Ordinary/drift estimation requires more neighbors than constraints, as in the source.
- Covariance coincidence threshold uses squared anisotropic distance under the first structure < 1e-10, matching COVA3. KT3D block nugget coincidence and validation tests use the supplied 1e-6 tolerance.
- Primary values trim at minimum inclusive / maximum exclusive. Secondary values must be finite. As in KT3D, target secondary values also use the primary-value trimming limits; excluded targets receive a trimmed_target_secondary status.
- Indicator-distribution mode (negative koption), original debug output, and automatic variogram fitting are not implemented. Unsupported indicator parameters are rejected explicitly.
- Duplicate samples are not silently merged. A singular system leaves that node unestimated. Review the status column.
- The map is an XY slice. The canvas is a preview, not a georeferenced raster. CSV estimates are point locations; block averages represent their stated support.
- Kriging variance is conditional on the model; it does not include variogram estimation, measurement bias, or all agronomic uncertainty.

## Output

CSV: x,y,z,estimate,variance,sd,count,observed,error,standardized_error,status. Missing results are blank, with a status explaining the reason. Error = estimate − observed. GEO-EAS is numeric and uses -999 for missing values; use CSV when you need failure reasons. Table previews the first 300 nodes, exports contain all nodes. Settings JSON records parameters, units, field selections, grid, and retained/excluded counts; it does not contain original observations and is not a session-import file.

## Validation performed

- Analytic two-sample ordinary and simple kriging values and variances.
- Exact sample reproduction, with and without nugget.
- Search anisotropy/orientation, validation exclusions, polynomial/external drift reproduction, block constant preservation, singular duplicate detection, supplied PAR parsing and rejection of unsupported indicator mode.
- Independent NumPy 3D block ordinary-kriging solution: 35 samples, 12 targets, eight discretization points; maximum difference in estimate/variance < 5e-14.
- Browser example: 64 samples, all 841 grid targets estimated; cross-validation all 64 locations estimated.
- Archived GSLIB reference outputs: hw2p2 point OK, hw2p3 block SK, and 16-node lab9 phosphorus OK agree within 0.001 (the archived output is printed to three decimals).
- Fresh compiled Fortran comparison: 19 configurations / 477 targets, covering all five models, SK/OK/type 2/external drift with point and block support, full quadratic drift, trend, octant limits, cross-validation, and nested power. No estimated/unestimated mismatches occurred. For spherical/exponential/Gaussian and non-power/non-hole configurations, maximum estimate difference < 0.000011 and maximum variance difference < 0.000005. Hole-effect estimate difference reached 0.00256; nested-power estimate difference reached 0.00175 and variance difference reached 0.00141. These are bounded checks on the supplied mixed-precision Fortran versus JavaScript double precision, not universal accuracy guarantees.
- The compiled comparison changes only output formatting to print more digits; calculation code and library sources are unchanged.
- Source-specific regression checks: Gaussian exponent, nested power offset, anisotropic coincidence threshold, and octant boundary rules.
- Full clipped yield smoke test: 5,590 observations and 5,025 targets with illustrative settings. This verifies execution, not model suitability for that field.

Run `node test.cjs` in source/ for the numerical checks. Open source/index.html via a static server; the delivered single HTML is the recommended student version.

Original Stanford GSLIB copyright/license is retained in the HTML, source, and LICENSE-GSLIB.txt. Original supplied KT3D files are included under reference/ in the ZIP.

## Source alignment update (version 1.1)

The first delivery used exp(−3(h/a)²) for Gaussian, whereas this supplied COVA3 uses exp(−9(h/a)²). Version 1.1 corrects the Gaussian formula, the power covariance offset in nested models, covariance coincidence distance, octant coordinate-plane boundaries and validation ordering, missing-Z fallback, and target secondary trimming. Recalculate any earlier Gaussian or nested-power estimates. Spherical and exponential formulas are unchanged.

Required GSLIB routines: COVA3, SETROT, SETSUPR, PICKSUP (file PICKSUPR.F), SRCHSUPR, KTSOL, CHKNAM, SQDIST, SORTEM, GETINDX. READPARM, KT3D, and MAKEPAR are already defined in KT3D.F. No additional nonintrinsic library calls were found in those ten routines.

For the Gaussian formula shown in other references: exp(−h²/b²), exp(−3h²/c²), and this COVA3's exp(−9h²/a²) describe the same family when a = 3b = √3c. The numerical range parameter must be converted; using the same number in all three formulas changes the spatial model.

Fresh Fortran verification uses gfortran with `-std=legacy -ffixed-line-length-none -fallow-argument-mismatch`. Only output formats are widened for comparison. The verification scripts/report are in verification/. The compiler and executable are not needed to use the page.
