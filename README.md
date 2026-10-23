# Field Variogram

A browser-local direct experimental semivariogram tool adapted from the supplied GAMV1.F and GV1.PAR. This page implements the **type-1 direct semivariogram requested by GV1.PAR**, one variable per run, with 2D/3D pair geometry. It is not a translation of all ten continuity measures in the original source.

## Use / publish

Download `Experimental_Variogram.html` and open it in a current desktop browser, or upload that single file alongside the existing Yield Window HTML in your GitHub Pages repository. The new page is accessed by appending `/Experimental_Variogram.html` to the repository's Pages address. Keep the exact filename and letter case. Do not replace the existing root index.html unless you intend to change the homepage.

The source ZIP also includes a `variogram/` folder with index.html, style.css, engine.js, and app.js. Upload that entire folder to use a separate `/variogram/` subpage. Both distributions have the same functionality. No package installation, build, API key, server, CDN, or external library is needed. This delivery is packaged for GitHub Pages and has not been published to the user's GitHub account.

## Load observations

CSV: header row with coordinate and value columns. GEO-EAS: title, column count, one column-name line per column, then whitespace-separated numeric records. Choose the matching file format. Uploading a new file or changing fields invalidates prior results. There is no separate Apply-fields step.

For Onarga_RawYieldClippedXY.csv:

- X: X_coord; Y: Y_coord; Z: None (Z=0).
- Variable: Yld Vol(Dr for raw yield. Choose yield_estimate for exported moving-average grids.
- Coordinate label: EPSG:32616, provided that is the verified source CRS; distance units: m.
- Do not use LON/LAT directly with meter lags. Soil locations must first be aligned to the yield CRS.

Coordinates and all distance settings must have matching linear units. This tool accepts local Cartesian grids too, but does not infer or transform a CRS. For 3D data, Z must be in the same distance units as X/Y. Blank/nonfinite coordinate or value rows are excluded. Trimming uses min <= value < max. No-data markers such as -999 must be excluded through the trimming range if applicable. No automatic detrending, declustering, outlier cleaning, or random subsampling is applied.

## Parameter file and defaults

GV1.PAR defaults are embedded: 20 lags, spacing 5, tolerance 2.5, no sill standardization, one direct variogram. Directions (azimuth, angular tolerance, horizontal bandwidth, dip, dip tolerance, vertical bandwidth):

1. 0, 180, 160, 0, 180, 160
2. 0, 180, 160, 0, 2, 1.5
3. 0, 2, 1.5, 0, 180, 160

Import a GAMV PAR file to populate controls. Local data/output paths are informational; the browser cannot read arbitrary paths on a user's computer. Column references are 1-based, and are mapped by position to the selected data columns. The supplied PAR refers to an old 4-column GEO-EAS file. **After importing it for a CSV with additional columns, manually reselect the intended X/Y/Z/value fields.** Unsupported measures or multiple variograms are rejected explicitly.

## Directions

GSLIB azimuth: 0 north/south, 90 east/west, clockwise from north. This differs from Yield Window's angle convention (0 east/west).

The source transforms dip using declination = 90 - dip. In this implementation positive dip tilts toward positive Z, following the supplied code's mathematics. Tolerances are half-angles; bandwidths are maximum perpendicular distances. Nonpositive azimuth/dip tolerance becomes 45 degrees, as in the source; nonpositive lag tolerance becomes half the spacing. Tolerances above 90 do not remove bandwidth or dip filtering.

For planar data, supplied directions 1 and 2 select the same pairs (Z is zero and their horizontal tests are identical). Direction 3 is a narrow north/south filter. The **Omni + east–west + north–south** preset gives an explicit agricultural-field comparison: 90-degree omni tolerance with broad bands, and directional 22.5-degree tolerances with 15-unit horizontal bands. These are starting settings, not a prescribed optimum. Adjust bands and angle tolerances and examine pair counts.

## Calculation and pair counting

Gamma = sum[(v_i - v_j)^2] / (2 N). Distance is the actual average separation of accepted pairs in the bin, not automatically the nominal lag.

GAMV1-compatible mode reproduces i <= j, including self pairs, and adds reversed contributions when azimuth tolerance >=90. A direct semivariogram's value is unchanged by those reversals, but its count doubles. The CSV also reports accepted unordered-pair counts so the distinction is visible. Self pairs are not independent observations. Unique mode instead uses i < j, omits self pairs, and never adds reversals; it still accepts distinct observations with coincident coordinates.

Like the source, output has nlag + 2 bins:

- lag_index 1: zero-distance (h <= epsilon), including coincident records and self pairs if enabled.
- lag_index 2: near-zero, centered at distance zero for positive-distance records within tolerance.
- lag_index 3 onward: positive lags 1..nlag.

A nonzero pair may contribute to multiple lag bins when tolerances overlap. Boundaries are inclusive. The global cutoff is (nlag + 0.5 - epsilon) * spacing, even if a larger tolerance would admit more distant records. This retains the supplied program's behavior. If using unusually large tolerances, account for this cutoff.

Normalization divides by the **population** variance of retained values (denominator n). Constant variables cannot be standardized. Tail/head means follow the source's orientation/reversal behavior. Empty bins remain blank rather than being plotted as zero. Zero and near-zero bins are omitted from the chart by default but retained in all exports.

## Plot and exports

The chart compares directions, supports actual-mean or nominal lag distances, and can hide bins with low contribution counts. Minimum count affects only the plot, not the result table or downloads. Distinct symbols/dashes accompany colors. Point labels/tooltips and a tabulated alternative provide the numerical values.

The reference line is the data's population variance (or 1 if standardized). It is **not a fitted sill**; no nugget/range/sill model is fitted. Stationarity and sampling support affect interpretation.

Download the CSV, scalable SVG plot, settings JSON, or GAMV-style .var text. CSV contains direction, label, azimuth, dip, source lag index, bin type, nominal/actual distance, semivariance, contribution count, unordered count, tail mean, and head mean. The .var uses NaN for empty-bin values and is GAMV-style, not promised compatible with every legacy plotting routine. Settings preserve all six direction parameters, trimming, selected fields, CRS label, and statistics. Download results before starting a new run or reloading.

## Compatibility, testing, and limitations

The supplied source references gamv1.inc, which was not attached. The adaptation explicitly uses epsilon=1e-6, JavaScript double precision, and browser-sized limits. No executable Fortran comparison was possible: the include and compiler were unavailable. Results can differ at floating-point angle/bin boundaries because the old routine uses single-precision direction arrays and a truncated PI constant. The adaptation uses full Math.PI. Single-variable trimming consistently excludes the upper bound; the source has inconsistent upper-bound tests in some multivariable branches.

Numerical checks include hand-computed semivariances and pair counts, empty bins, overlapping bins, coincident coordinates, normalization, direction filters, CSV/GEO-EAS/PAR parsing, a separate Python reference over 97 3D points and four directions (all counts, distances, semivariances, and head/tail means), and all 5,590 Onarga yield records. Source defaults produced 66 populated output bins and population variance 443.0345591205632 for that yield file.

Browser QA verified loading simulated records, applying the field-direction preset, completing a Web Worker calculation, rendering three directional curves, and populating the lag table and statistics. The download controls were enabled; exported-file round trips were not separately browser-verified. Use the simulated field to familiarize students with the controls.

Run `node test.cjs` inside the variogram source folder for the bundled checks. The optional --field test references the original working upload path and is only for the development environment.

The app supports at most 15,000 valid observations and 12 directions. It evaluates all unordered pairs (O(n²)), with early distance rejection. A Web Worker keeps the interface responsive, reports progress, and can be canceled. Larger datasets need a spatial-index implementation or prior defensible subsetting. No types 2–10, cross-variograms, or automatic model fitting are included.

## Attribution

See LICENSE-GSLIB.txt for the original notice, retained verbatim from the supplied file. Original: A.G. Journel (1978); revisions: K. Guertin (1980). The adapted engine and HTML retain the copyright and redistribution condition. The source package includes the supplied GV1.PAR example.
