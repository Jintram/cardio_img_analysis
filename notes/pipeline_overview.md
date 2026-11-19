# Contractility analysis pipeline: overview

> This overview was drafted by Claude (an AI assistant) from the code, then revised through back-and-forth with the author, who corrected and refined it. Check it against the code before relying on details.

Scripts covered:
- `dev/time_ale_contractility_dataset2_2023-10-11_edited202403.py`: main workflow
- `dev/thomas_contr-relax-time.py`: prototype of the contraction/relaxation times
- `functions/custom_functions_contractility.py` and `functions/custom_functions_movies.py`: helper functions used by both

## Setup / preprocessing

- **Metadata:** read from an Excel sheet (sample name, condition, ROI, reference frame, movie duration, frame count). Frame interval: `dt = duration_movie / total_frames`.
- **QC movies (optional):** MP4s from frames 0–500, every 10th frame (`custom_functions_movies.py`).
- **ROI selection:** drawn per sample with `cv2.selectROI`, pasted into the Excel and reloaded. ROI crops are saved as PDFs.
- **Cropping:** the first `NR_FRAMES` (1500) frames are cropped to the ROI and saved as `.npy`, to speed up the later steps.

## Core signal: image correlation

- **Trace:** Pearson correlation between all ROI pixels of each frame and one reference frame. The trace used is `1 − R`: about 0 at rest, higher during a beat.
- **Reference frame calibration:** a preliminary run over 250 frames is plotted. The reference frame is adjusted in the Excel until it lies in a relaxed state (outside a beat), then the run is repeated.
- **Full run:** correlation over all 1500 frames. The result (`list_traces_corrcontr`) is pickled.

## Peak detection (`find_features_and_plot`)

- **Smoothing:** Savitzky–Golay filter (window about 0.2 s, order 3). The smoothed trace is used only here, for peak detection and the normalization constants. All metrics are computed on the unsmoothed trace.
- **Peaks:** `scipy.signal.find_peaks` with a minimum distance of 0.5 s and a minimum height at the 80th percentile of the smoothed trace.
- **First and second peak:** the 2nd and 3rd detected peaks. The very first is skipped because a movie may start mid-beat.
- **Normalization constants:** the 2% and 98% quantiles of the smoothed trace.

## Final metrics

Every metric comes from a single beat per recording (or a single interval, for the interpeak time); nothing is averaged.

- **Interpeak time:** time between the first and second peak (frames × `dt`). Contractions per minute = 60 / interpeak time.
- **Peak duration (width at half-max):** computed on the unsmoothed trace, normalized to its 2nd–98th percentile range (no baseline correction). Within ± half the interpeak distance around the second peak, the frames closest to 0.5 on each side are found. The duration is their time difference, so its resolution is `dt`.
- **Peak height:** raw `1 − R` at the first peak (not normalized, not baseline-corrected).
- **Contraction and relaxation times** (`determine_contractile_times`, prototyped in `thomas_contr-relax-time.py`):
  - **Baseline correction:** the reference frame is set to NaN (artifact). A rolling-minimum baseline is taken (window about 2× the interpeak time), padded, fitted with a 4th-order polynomial and subtracted. The amplitude is kept (no normalization).
  - **Threshold:** a line at 5% of the baseline-corrected first-peak height (`PEAK_BASE_FRACTION = 0.05`).
  - **Crossings:** around the second peak, crossings of the unsmoothed, baseline-corrected trace with the threshold line are found (linear interpolation + `brentq`). The crossings closest to the peak on the left and right are used.
  - **Contraction time** = peak time − left crossing.
  - **Relaxation time** = right crossing − peak time.
  - **Fractions:** each time is also reported as a fraction of their sum.
- **Grouping and export:** samples are grouped by regex search terms on the `condition` column. Peak duration, interpeak time and peak height go to `final_overview_values.xlsx`. The contraction and relaxation times are returned as dictionaries but not written to Excel.

## Output plots

- **`analysis/peakfinder_corr_*.pdf`:** raw and smoothed trace, with the detected peaks.
- **`final_overview_traces.pdf`:** traces per condition, normalized to the 2nd–98th percentiles and aligned on the first peak. **Not baseline-corrected.**
- **`analysis/contractrelax_trace_*.pdf`:** top panel shows the raw trace with the polynomial baseline; bottom panel shows the baseline-corrected trace with the 5% line and crossings.

## Caveats / known issues

- **Mixed peaks:** the threshold for contraction and relaxation is based on the first peak's height but applied to the second peak.
- **Noise at the 5% threshold:** the trace is unsmoothed and the 5% level is close to the noise floor, so crossings can come from noise. Taking the crossing closest to the peak biases the contraction and relaxation times toward shorter values. Running the crossing search on the smoothed, baseline-corrected trace would be more robust.
- **Peak index offset:** the peak index comes from the smoothed trace, so it can be off by a frame or two from the raw maximum.
- **Side effect:** `determine_contractile_times` sets the reference frame to NaN directly in the arrays stored in `list_traces_1min`, which changes the caller's data.
- **Main script:** it is meant to be run section by section. Line 249 calls `read_sample_information(...)` without the `cuslibc.` prefix, and line 341 has a `def` without a body; both fail if run as-is.

