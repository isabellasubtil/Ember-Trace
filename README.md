# Brazil's Fire-Season Climate: What Is Changing, and Does Fire Follow?

**NASA Space Apps Challenge 2026 · Be An Earth System Trend Detective**

> **Headline.** The Cerrado warms about 0.5 °C per decade in station data (up to about 1.0 °C per decade in reanalyses), dries, and has a higher vapor pressure deficit (VPD), robustly across data sources and time windows. Fire counts nevertheless *decrease*. Climate explains which years are bad for fire, but not the long-term trend in fire counts.

This README is both the project documentation and a study guide. It walks through every step in order (data, preprocessing, statistics, figures, results, limitations) with enough detail that you can re-derive, defend, or extend each point. Section 13 maps every claim in the slide deck to its evidence and code.

---

## Table of contents

1. [The question and the story](#1-the-question-and-the-story)
2. [Repository layout and how to run everything](#2-repository-layout-and-how-to-run-everything)
3. [Data sources](#3-data-sources)
4. [Preprocessing: from raw files to one comparable grid](#4-preprocessing-from-raw-files-to-one-comparable-grid)
5. [From monthly maps to one value per year](#5-from-monthly-maps-to-one-value-per-year)
6. [The statistical toolkit, explained from first principles](#6-the-statistical-toolkit-explained-from-first-principles)
7. [The sanity test: why we did not use the classic test](#7-the-sanity-test-why-we-did-not-use-the-classic-test)
8. [Analysis pipeline, step by step](#8-analysis-pipeline-step-by-step)
9. [Results](#9-results)
10. [Figures: what each one shows and how to read it](#10-figures-what-each-one-shows-and-how-to-read-it)
11. [Quality assurance: what we checked and what went wrong along the way](#11-quality-assurance-what-we-checked-and-what-went-wrong-along-the-way)
12. [Limitations and threats to validity](#12-limitations-and-threats-to-validity)
13. [Claim-to-evidence map (for the deck and for Q&A)](#13-claim-to-evidence-map-for-the-deck-and-for-qa)
14. [Study guide: questions you should be able to answer](#14-study-guide-questions-you-should-be-able-to-answer)
15. [Privacy, data and AI disclosure](#15-privacy-data-and-ai-disclosure)
16. [Glossary](#16-glossary)
17. [References](#17-references)

---

## 1. The question and the story

**Challenge.** Be An Earth System Trend Detective: use NASA Earth observation data to find, quantify, and explain trends.

**Our question.** Brazil's fire season (August to October) is hot and dry. If the climate in that season is getting hotter and drier, are fires increasing? And how sure can we be that the climate trends are real, given that we only have about 24 years of data?

**Study area.** Three Brazilian biomes, defined by the official IBGE biome polygons: **Amazon**, **Cerrado** (the tropical savanna of central Brazil), and **Pantanal** (the large seasonal wetland). The maps also show Caatinga, Atlantic Forest, and Pampa for context.

**The narrative of the deck** is a detective case, which also maps onto a product-style storyline:

| Beat | Product-style step | What happens |
|---|---|---|
| The case | Problem | Hotter and drier should mean more fire. Does the data agree? |
| The suspect | Approach | Before trusting trends, we found that the classic trend test produces false alarms on short, persistent series |
| The clues | Evidence 1 | Maps of per-pixel trends in rain, soil moisture, VPD, temperature |
| The cross-check | Evidence 2 | Three independent temperature sources and five time windows agree for the Cerrado |
| The twist | Insight | Fire counts fall anyway, yet climate explains the bad years |
| The verdict | So what | Watch VPD and soil moisture for severe seasons; do not read falling counts as falling climate pressure |
| The fine print | Risks and disclosure | Privacy, AI use, limits of the answers |

**Result in one paragraph.** For the Cerrado, temperature trends are significant in all three temperature products (MERRA-2: +0.98 °C/decade, GLDAS: +0.97, CPC stations: +0.53), significant in 15 of 15 combinations of source and time window, and a 2014 step appears in all three. VPD rises (+0.23 kPa/decade, significant) and soil moisture falls (−7.4 %/decade at 10–40 cm, significant). Cerrado fire counts fall by 18 % per decade (significant). Yet after removing each series' own trend, year-to-year fire counts in the Cerrado correlate strongly with VPD (ρ = +0.73) and rain (ρ = −0.63). So climate modulates *which years burn more*, while something else (land use, management, enforcement, or sensor drift) pushes the long-term count down. We do not test those explanations.

---

## 2. Repository layout and how to run everything

### 2.1 Notebooks

| Notebook | Purpose |
|---|---|
| `fogo_ingestao_colab_v4.ipynb` | **Ingestion.** Downloads GLDAS, GPM IMERG, and MERRA-2 from NASA Earthdata, subsets to Brazil, regrids to 0.25°, caches to Google Drive. (Originally also loaded INPE fire counts; the final analysis uses NASA FIRMS instead, see below.) |
| `analysis_trends_drivers_v3_en.ipynb` | **Analysis.** All statistics, all figures, all tables. Fully in English. 32 cells. |

### 2.2 Google Drive layout (kept from the original Portuguese setup)

```
MyDrive/fogo_trends/
  raw/                        # per-granule cache of downloaded NASA files (resumable)
  data/
    gldas_025_mensal.nc       # monthly GLDAS variables on the 0.25° grid
    gldas_sm_pequeno.nc       # small file with only the soil-moisture variables (optional)
    gpm_025_mensal.nc         # monthly precipitation, mm/month
    merra2_025_mensal.nc      # monthly VPD (kPa) and T2M_C (°C)
    cpc_sazonal_8910.nc       # cached CPC Aug-Oct mean temperature, per year, 0.5°
    firms/modis_{year}_Brazil.csv      # downloaded FIRMS annual files
    focos_firms_aqua_mensal.csv        # fire counts per month and biome (Aqua only)
  resultados/
    figures/                  # PNG (300 dpi) + vector PDF for every figure
    sanity_false_positives.csv
    trend_by_biome.csv
    fire_trend_and_correlation.csv
    robustness_temperature_sources.csv
    robustness_time_window.csv
    tendencias_pixel.nc       # per-pixel slope, p-values, FDR mask for each variable
```

The folder names `fogo_trends`, `data`, and `resultados`, and the cached file `focos_firms_aqua_mensal.csv`, are kept in Portuguese on purpose: they already exist on the Drive, and the analysis notebook reads the old column names (`bioma`, `focos`) as well as the new ones (`biome`, `fires`).

### 2.3 Running order

1. **Ingestion notebook** (needs a NASA Earthdata login): run setup, the Earthdata login cell, then `run('gldas')`, `run('gpm')`, `run('merra2')`. Each is resumable because per-granule files are cached in `raw/`.
2. **Analysis notebook**, in order. Cell 13 downloads CPC temperature (OPeNDAP first, plain HTTP fallback of about 380 MB per year-file as a backup) and caches it. Cell 25 downloads FIRMS fire files if `focos_firms_aqua_mensal.csv` is not there. Figures appear inline and are saved to `resultados/figures/`.
3. Delete `cpc_sazonal_8910.nc` or `focos_firms_aqua_mensal.csv` only if you want to force a re-download.

### 2.4 Reproducibility notes

- All randomness comes from `numpy.random.default_rng(seed)` with fixed seeds (the AR(1) surrogate p-value uses seed 0; the sanity simulation uses seed 2). Re-running gives identical numbers *provided the same inputs and the same simulation counts*; p-values from different `nsim` differ slightly (Monte Carlo error).
- Package list: `pymannkendall`, `regionmask`, `geobr`, `statsmodels`, `earthaccess`, `netCDF4`, `xarray`, `numpy`, `pandas`, `scipy`, `matplotlib`, `geopandas`, `requests`.
- The analysis notebook was developed and tested on synthetic data and then run by the team in Google Colab on the real data. The numbers in this README come from the real-data run (the CSV files in `resultados/`).

---

## 3. Data sources

We use **five climate and fire data products** plus one set of boundaries. All are public.

### 3.1 GPM IMERG monthly precipitation

- Product: `GPM_3IMERGM`, version 07 ("Final" run), global, monthly.
- Native resolution about 0.1°. Variable used: `precipitation` (mm/hour, monthly mean rate).
- What it is: satellite-based precipitation estimates that merge passive microwave and infrared data and are calibrated against rain gauges (in the Final run).
- Our conversion: mm/hour × 24 × days in the month → **mm/month**.
- Caveat: the Final run lags real time, so **2025 was incomplete** and the analysis automatically drops it, which gives 2001–2024 (24 years).

### 3.2 GLDAS soil moisture and air temperature

- Product: `GLDAS_NOAH025_M`, version 2.1, monthly, 0.25°.
- Variables: `SoilMoi0_10cm_inst`, `SoilMoi10_40cm_inst` (soil moisture in kg/m², upper layers), `Tair_f_inst` (air temperature, K → °C).
- What it is: output of the **Noah land-surface model** driven by observation-based forcing. This is a **model product**, not a measurement. Soil moisture here is simulated, not observed.

### 3.3 MERRA-2 temperature and humidity

- Product: `M2TMNXSLV`, version 5.12.4, monthly means of single-level diagnostics.
- Native resolution about 0.5° × 0.625°. Variables: `T2M` (2 m air temperature), `QV2M` (2 m specific humidity), `PS` (surface pressure).
- What it is: an atmospheric **reanalysis**, a model that assimilates observations.
- We derive **VPD** from these (Section 4.3).

### 3.4 CPC Global Unified Temperature (station-based)

- Provider: NOAA Physical Sciences Laboratory. Daily `tmax` and `tmin`, 0.5° grid.
- What it is: an analysis built by **interpolating station observations** (on the order of 6–7 thousand stations globally). This is why it acts as an **independent arbiter** between the two model-based temperature products.
- Our use: mean temperature = (Tmax + Tmin) / 2, averaged over August–October of each year, over the Brazil box (lat −35° to 7°, lon 285° to 327°, i.e. −75° to −33°). Interpolated bilinearly to the 0.25° grid.
- Caveat: station density in Brazil is uneven (sparse in the Amazon). The interpolation also smooths local extremes.

### 3.5 NASA FIRMS active fires (MODIS, Aqua only)

- Product: MODIS Collection 6.1 active fire detections, annual country files for Brazil.
- URL pattern: `https://firms.modaps.eosdis.nasa.gov/data/country/modis/{year}/modis_{year}_Brazil.csv`.
- Filters: **Aqua satellite only** (the reference satellite; mixing Terra and Aqua would change the sampling), and `type == 0` (presumed vegetation fire; excludes static sources and offshore detections).
- Aqua launched in 2002, so the fire series covers **2002–2024 (23 seasons)** rather than 2001.
- Each detection is assigned to a biome by a spatial join with the IBGE polygons, then counted per month and biome. We also keep the sum of fire radiative power (FRP) but the final analysis uses counts.
- Caveat: **Aqua's overpass time drifts after about 2022**, which can reduce detections independently of real fire activity. This is an unresolved caveat for the last few years of the fire series.

### 3.6 IBGE biome boundaries

- Loaded through the `geobr` package (`read_biomes(year=2019)`), with a fallback that reads the cached parquet file directly if `geobr` fails, and a last-resort approximate-box fallback marked "for testing only" in the output (it did not trigger on the real run: the run reported "IBGE (official polygons)").
- Each 0.25° pixel is assigned to the biome whose polygon contains the pixel center, using `regionmask` (`overlap=False` to tolerate overlaps at borders).

---

## 4. Preprocessing: from raw files to one comparable grid

### 4.1 The master grid

All variables are put on a single regular grid:

- Bounding box: west −74°, south −34°, east −34°, north 6°.
- Resolution: 0.25°, giving 160 latitudes × 160 longitudes of cell centers.
- Time: monthly, January 2001 to December 2026 in the ingestion notebook (2026 is incomplete and is trimmed later).

### 4.2 Subsetting and regridding

For each NASA granule the ingestion notebook:

1. opens the file (GPM uses the HDF5 group `Grid`),
2. renames coordinates to `lat` and `lon`, sorts by latitude,
3. cuts a one-degree margin around the box,
4. **interpolates linearly** to the master grid,
5. saves the small result to the Drive cache, so a dropped session can resume.

Linear interpolation is an approximation (Section 12). GLDAS is already 0.25°, so for it nothing is invented; GPM (0.1°) is effectively **coarsened**, while MERRA-2 (0.5° × 0.625°) and CPC (0.5°) are **upsampled**, which creates smooth fields with many neighboring pixels that are not independent.

### 4.3 Deriving VPD from MERRA-2

Vapor pressure deficit is the gap between how much water vapor air *could* hold at its temperature and how much it *does* hold. It is the most direct atmospheric measure of how strongly the air pulls moisture out of vegetation, which is why it relates to fire.

Using monthly mean T2M, QV2M, and PS:

```
T_c = T2M − 273.15                                  (°C)
e_s = 0.6108 · exp(17.27 · T_c / (T_c + 237.3))     (saturation vapor pressure, kPa; Tetens/Murray form)
e   = QV2M · PS / (0.622 + 0.378 · QV2M) / 1000     (actual vapor pressure, kPa; PS in Pa)
VPD = max(e_s − e, 0)                               (kPa)
```

Important caveat: this uses **monthly means** of temperature and humidity. Because saturation pressure is a convex function of temperature, VPD from monthly means is *not* the monthly mean of daily VPD (it is typically lower). The *trend* is still informative, but absolute values should not be compared with daily-based VPD.

### 4.4 Unit conversions

- GPM: mm/hour → mm/month (× 24 × days in month).
- GLDAS rain/evaporation (not used in the final analysis): kg m⁻² s⁻¹ → mm/month (× 86400 × days in month).
- GLDAS air temperature: K → °C (subtract 273.15).

---

## 5. From monthly maps to one value per year

This step is crucial, because every statistic downstream treats **one number per year** as one observation.

### 5.1 The fire season

`SEASON = [8, 9, 10]` (August, September, October). `SEASON_NAME = 'Aug–Oct'`. This is the peak of the main fire season over much of the Cerrado and the southern Amazon. A different season (for example May–September) can be tested by changing one line.

### 5.2 Seasonal aggregation rule

For each variable and each year, using only months in `SEASON`:

- **precipitation: sum** of the three months (mm per season),
- **everything else: mean** of the three months,
- **a year is valid only if all three months exist** (`count == 3`), otherwise the pixel-year is `NaN`, never zero.

This aggregation removes the annual cycle, which would otherwise dominate any trend test.

### 5.3 Analysis years

The nominal range is 2001–2025. A year is kept only if every variable has data in it. GPM Final lags, so 2025 is dropped, leaving **2001–2024 (24 years)**. The CPC temperature uses the same years. The fire series covers 2002–2024.

---

## 6. The statistical toolkit, explained from first principles

The core problem: we have **one value per year, about 24 values per series**. The question for each series is "is there a real monotonic trend, how large is it, and how sure are we?"

### 6.1 Sen slope (how big is the trend?)

For every pair of years (i < j), compute the slope (x_j − x_i) / (j − i). The **Sen slope** is the **median** of all these pairwise slopes. It is the robust cousin of the least-squares slope: a few extreme years barely move a median. In the code (`mk_sen_hr`), this is computed for all pixels at once, vectorized.

We report slopes **per decade** (slope × 10). For precipitation and soil moisture, slopes are expressed as **% of the period mean per decade** (slope × 10 × 100 / mean), so that wet and dry regions are comparable. For VPD (kPa) and temperature (°C), we use physical units per decade.

### 6.2 Mann-Kendall test (is the trend distinguishable from noise?)

Count, over all pairs of years (i < j), whether the later value is larger (+1), smaller (−1), or equal (0). The sum is the statistic **S**. If there is an upward trend, S is large and positive; if downward, large and negative; if no trend, near zero.

Under the null hypothesis (independent, identically distributed data, no trend), the variance of S is known:

```
Var(S) = n(n − 1)(2n + 5) / 18
```

and `Z = (S ∓ 1) / sqrt(Var(S))` (the ∓1 is the continuity correction) is approximately standard normal. The two-sided p-value is `2·(1 − Φ(|Z|))`. This is the **classic Mann-Kendall p-value** (`p_naive` in the code).

**The catch:** the null assumes independent years. Climate series are usually **autocorrelated**: a warm year tends to be followed by a warm year. Positive autocorrelation makes S more variable than the formula assumes, so the classic test produces **too many false positives**. Section 7 quantifies this.

### 6.3 Autocorrelation and the AR(1) model

Autocorrelation at lag 1, φ, is the correlation between a year's value and the next year's. An **AR(1) process** is x_t = φ·x_{t−1} + e_t, where e_t is white noise. It is the simplest model of persistence. A φ of 0.8 means the series has strong memory (typical of soil moisture and temperature).

### 6.4 Hamed-Rao modification (correction by variance inflation)

Hamed and Rao (1998) correct the variance of S by estimating the autocorrelation of the **ranks of the detrended series** at every lag and inflating Var(S) by a factor `n/n*`. In our vectorized implementation:

1. remove the Sen trend, rank the residuals, demean,
2. compute the autocorrelation at lags 1 … n−3,
3. keep only lags that are significant (|ρ_k| > z_{0.975}/√n),
4. combine them: `n/n* = 1 + 2/(n(n−1)(n−2)) · Σ (n−k)(n−k−1)(n−k−2)·ρ_k`.

**Our implementation choice:** the factor is clamped to be at least 1 (`max(nns, 1)`), so the correction can only *widen* the null distribution, never narrow it. The reference library, `pymannkendall`, can return NaN when the weighted autocorrelation is negative (this happened once in our tables: Cerrado soil moisture 0–10 cm shows `NaN` in the Hamed-Rao column of `trend_by_biome.csv`). We validated our function against `pymannkendall` on cases where both are defined (slope and p-values identical).

In our sanity test Hamed-Rao helps but is not enough on its own (Section 7).

### 6.5 The AR(1) surrogate test (our primary p-value)

Instead of correcting a formula, we **simulate the null**. For each series:

1. remove the Sen trend and the mean,
2. estimate the lag-1 autocorrelation φ̂,
3. correct φ̂ for **small-sample bias** (Kendall, 1954: the expected value of φ̂ is about φ − (1 + 3φ)/n), giving `φ = (n·φ̂ + 1)/(n − 3)`, clipped to [−0.5, 0.95],
4. simulate many trend-free AR(1) series with that φ and the same length n (burn-in of 50 steps),
5. compute |S| for each simulated series, giving the null distribution of |S| *for series that are as persistent as this one*,
6. the p-value is the fraction of simulated series whose |S| is at least as large as the observed |S|, with the standard "+1" correction: `p = (count + 1) / (nsim + 1)`.

To keep this fast across about 25,600 pixels per variable, φ is **binned into 30 values** between −0.5 and 0.95 and each bin is simulated once (4,000 simulations by default). For the per-biome series (few series) we use 20,000 simulations; for the window-sensitivity table, 10,000.

Because S depends only on the **signs** of pairwise differences, the scale of the noise does not matter, so unit-variance innovations are enough.

This is the **primary p-value** used for maps, tables, and figures, unless a column says otherwise.

### 6.6 Multiple testing and the False Discovery Rate (FDR)

We test about 25,600 pixels per variable. At α = 0.05, **5 % of pixels would be "significant" by chance alone** even with no real trend. The **Benjamini-Hochberg (BH)** procedure controls the *expected share of false discoveries among the pixels declared significant*. Because neighboring pixels are positively correlated, we follow **Wilks (2016)**: for a spatial field, use `α_FDR = 2 × α_global`, which gives roughly the intended global false-rejection behavior. With α = 0.05 this gives **α_FDR = 0.10**.

On maps, **stippling marks only pixels that survive BH-FDR**. This is stricter and much more honest than coloring every pixel with p < 0.05.

We did **not** apply an FDR correction across the rows of the per-biome tables (about 21 variable × biome rows) or across the correlation tables. Those p-values are marked as unadjusted, and Section 12 discusses this.

### 6.7 Pettitt change-point test

Pettitt's test looks for a **single abrupt shift** in the center of a series without assuming where it is. It computes a rank-based statistic U_t at every candidate break t and takes `K = max |U_t|`. The approximate p-value is `p = 2·exp(−6K² / (n³ + n²))` (capped at 1). We report the year **after** the most likely break, and only if p < 0.1.

Caveat: Pettitt assumes independent data, so it can also be too liberal for persistent series. We therefore read its output as *descriptive* ("the data look like a step around 2014") rather than as a formal test. The finding that **the same step year, 2014, appears in three independent products** is stronger evidence than the p-value itself.

### 6.8 Spearman rank correlation, raw and detrended

Spearman's ρ is the correlation between the **ranks** of two series, robust to outliers and monotonic nonlinearity.

**The trap:** if fire counts decline over time and VPD rises over time, they will be negatively correlated *just because both trend*, even if the two have nothing to do with each other year to year. To isolate **year-to-year covariation**, we compute the correlation after **removing each series' own Sen trend** (the "detrended" correlation). This asks: "in years when VPD is higher than its own trend, are fire counts also higher than their own trend?"

For the Cerrado the raw and detrended correlations are very different, which is itself informative (Section 9.5).

### 6.9 Area-weighted biome series

A biome's yearly value is the **mean over its pixels, weighted by cos(latitude)**, because pixels near the equator cover more area than pixels near the poles (on a regular lat-lon grid). The weights are recomputed each year over only the pixels that have data that year, and a year with no data is `NaN`, never zero.

---

## 7. The sanity test: why we did not use the classic test

Because the whole project depends on trend tests, we first tested the tests.

**Setup.** Generate **10,000 synthetic series of length 25 per autocorrelation level**, with **no trend at all** (pure AR(1) noise), for φ = 0, 0.3, 0.6, 0.8. At α = 0.05, an honest test should flag about 5 % of them.

**Results (fraction of trend-free series declared "trending")**

| Autocorrelation φ | Classic Mann-Kendall | Hamed-Rao | AR(1) surrogate | AR(1) surrogate + FDR |
|---|---|---|---|---|
| 0 | 4.7 % | 4.6 % | 6.3 % | 0.0 % |
| 0.3 | 13.3 % | 12.6 % | 8.1 % | 0.0 % |
| 0.6 | 29.1 % | 23.6 % | 11.5 % | 2.6 % |
| 0.8 | **47.1 %** | **35.8 %** | **17.3 %** | **8.6 %** |

These are the exact values in `resultados/sanity_false_positives.csv`. Two details worth noticing: at φ = 0 the AR(1) test is slightly *liberal* (6.3 %), because the simulated null uses an estimated φ that is noisy for short series; and the "+ FDR" column is computed by applying BH across the 10,000 simulated series at α_FDR = 0.10, so 0 % means "no series flagged at all".
**Takeaways**

- The classic test is only honest when there is no autocorrelation. For persistent series (φ = 0.8, typical of temperature or soil moisture) **almost half** of the "significant trends" would be spurious.
- Hamed-Rao helps but does not fix the problem at φ = 0.8.
- The AR(1) surrogate test cuts false positives sharply but is **not perfect** (17 % at φ = 0.8, above the nominal 5 %). We say so openly.
- Adding FDR brings the rate near the nominal level for the most persistent case, and is the reason the maps use stippling on FDR-surviving pixels only.
- The cost is **power**: a more conservative test misses some weak real trends.

This is the justification for the deck's slide 2 ("Before trusting any trend, we caught the classic test lying").

---

## 8. Analysis pipeline, step by step

Each step names the cell group in `analysis_trends_drivers_v3_en.ipynb`. Cell numbers refer to the notebook's 0-based cell index.

### Step 0. Setup (cells 2–3)
Install packages, mount Drive, set `SEASON`, `YEARS`, `ALPHA = 0.05`.

### Step 1. Statistics functions (cell 5)
`mk_sen_hr` (Sen slope, classic p, Hamed-Rao p), `fdr_mask` (BH at 2α), `pettitt`, `p_ar1_surrogate`.

### Step 2. Figure style (cell 7)
A shared style module: validated categorical palette, blue-red diverging colormap with a neutral midpoint (red means "change that favors fire"), thin marks, direct labels, 300 dpi PNG plus vector PDF. `LANG = 'en'` by default; `LANG = 'pt'` renders every figure in Portuguese. `TITLES = False` removes headline titles for use in a paper.

### Step 3. Sanity test (cell 9)
The simulation of Section 7. Writes `sanity_false_positives.csv` and Figure 3.

### Step 4. Seasonal data (cell 11)
Open each NetCDF, aggregate to Aug–Oct per year (Section 5), trim to complete years. Printed output shows valid-year counts and means per variable.

### Step 5. CPC station temperature (cell 13)
Download, subset, average Aug–Oct, interpolate to 0.25°. Cached in `cpc_sazonal_8910.nc`. If CPC is unavailable the notebook continues with two sources.

### Step 6. Biome mask (cell 15)
IBGE polygons to pixels. Prints how many pixels fall in each biome.

### Step 7. Per-pixel trends (cell 17)
For each variable and pixel: Sen slope, classic p, Hamed-Rao p, **AR(1) surrogate p**, then BH-FDR over all valid pixels of that variable. Writes `tendencias_pixel.nc` and a summary table of the **percentage of pixels significant** under three rules (classic uncorrected, AR(1) without FDR, AR(1) with FDR).

### Step 8. Maps (cell 19)
Figure 1 (study area), Figure 2 (four-panel trend maps), Figure S1 (supplementary maps).

### Step 9. Per-biome series and table (cells 21–22)
Area-weighted yearly series per variable and biome. For each: Sen slope per decade, p-values (AR(1) with 20,000 simulations, Hamed-Rao, classic), Pettitt shift year (if p < 0.1) and Pettitt p. Writes `trend_by_biome.csv` and Figure 4.

### Step 10. Fire (cells 25–26)
Ingest FIRMS (Aqua, `type == 0`), spatial-join to biomes, count per month and biome, sum Aug–Oct per year (2002–2024). For each biome: fire trend (Hamed-Rao slope as % of mean per decade, p, Pettitt), then **raw** and **detrended** Spearman correlations with every climate series. Writes `fire_trend_and_correlation.csv` and Figure 5.

### Step 11. Robustness (cells 28–30)
(a) Do the three temperature sources agree? Mean trend over Brazil, spatial correlations between trend maps, per-biome table (Figures 6a and 6c). (b) Does the trend survive changing the time window? Five windows, all variables of interest (Figure 6b, `robustness_time_window.csv`). (c) Is there a Pettitt step in each temperature product (table).

---

## 9. Results

All numbers below come from the real-data run (the CSVs in `resultados/`). "Significant" means p < 0.05 under the **AR(1) surrogate test** unless stated. Trends are per decade.

### 9.1 Where is the climate changing? (per-pixel, Figure 2)

Share of Brazilian pixels with a significant trend **after FDR** (stippling):

| Variable (Aug–Oct) | Pixels significant (AR(1) + FDR) |
|---|---|
| Air temperature, MERRA-2 (T2M) | 45 % |
| Air temperature, CPC (stations) | 40.5 % |
| Vapor pressure deficit | 30 % |
| Soil moisture 0–10 cm | 24 % |
| Soil moisture 10–40 cm | 20 % |
| Precipitation | 6 % |

Temperature and VPD change over large areas. Rain is the weakest signal: only 6 % of pixels survive FDR, so we do not claim a robust rain trend.

### 9.2 Biome trends (Figure 4, `trend_by_biome.csv`)

| Variable | Amazon | Cerrado | Pantanal |
|---|---|---|---|
| Air temperature MERRA-2 (°C/dec) | +0.78 (p = 0.001) | +0.98 (p < 0.001) | +0.92 (p = 0.011) |
| Air temperature GLDAS (°C/dec) | +0.41 (p = 0.060, n.s.) | +0.97 (p < 0.001) | +1.11 (p = 0.001) |
| Air temperature CPC stations (°C/dec) | +0.38 (p < 0.001) | +0.53 (p < 0.001) | +1.12 (p = 0.008) |
| VPD (kPa/dec) | +0.22 (p = 0.006) | +0.23 (p < 0.001) | +0.23 (p = 0.023) |
| Soil moisture 0–10 cm (%/dec) | −3.8 (p = 0.037) | −6.7 (p = 0.003) | −8.0 (n.s., p = 0.20) |
| Soil moisture 10–40 cm (%/dec) | −4.5 (p = 0.032) | −7.4 (p = 0.002) | −12.6 (n.s., p = 0.155) |
| Precipitation (%/dec) | −8.5 (n.s., p = 0.095) | −3.7 (n.s., p = 0.51) | −19.2 (p = 0.030) |

Pettitt change points (p < 0.1): VPD and MERRA-2 temperature step in **2014** in both Amazon and Cerrado; CPC Amazon temperature in **2009**; CPC and GLDAS Cerrado temperature in **2014**; Cerrado soil moisture (10–40 cm) in 2016; Pantanal soil moisture around 2019.

### 9.3 Do independent temperature sources agree? (Figures 6a, 6c)

- **Mean temperature trend over Brazil** (°C/decade): MERRA-2 **0.79**, GLDAS **0.59**, CPC **0.45**.
- **Spatial agreement of the trend maps (correlation across pixels)**: MERRA-2 × GLDAS 0.16–0.18 (low), MERRA-2 × CPC 0.49, GLDAS × CPC 0.26.
- **Interpretation.** The two *model-based* products disagree about *where* the warming is strongest, while the station-based CPC product sits closer to MERRA-2 and gives smaller magnitudes. All three agree on the **sign** and on a **step around 2014 in the Cerrado**. The conservative reading is "the Cerrado has warmed by roughly 0.5 °C per decade in stations, and reanalyses suggest more."

### 9.4 Sensitivity to the time window (Figure 6b, `robustness_time_window.csv`)

Five windows: 2001–2024, 2001–2022, 2001–2020, 2005–2024, 2003–2022. Cerrado, trend per decade (* = p < 0.05):

| Variable | 01–24 | 01–22 | 01–20 | 05–24 | 03–22 |
|---|---|---|---|---|---|
| Temperature MERRA-2 | +0.98* | +0.85* | +0.93* | +1.11* | +1.01* |
| Temperature GLDAS | +0.97* | +0.91* | +0.99* | +1.20* | +0.96* |
| Temperature CPC | +0.53* | +0.46* | +0.52* | +0.57* | +0.48* |
| VPD (kPa) | +0.23* | +0.20* | +0.24* | +0.24* | +0.22* |
| Soil moisture 10–40 cm (%) | −7.4* | −5.7* | −4.9 | −9.9* | −6.3* |
| Precipitation (%) | −3.7 | −2.4 | −8.5 | −1.6 | −0.0 |

- **Cerrado temperature is significant in 15 of 15 source × window combinations.** VPD is significant in all five windows. Soil moisture in four of five.
- **Amazon**: CPC temperature stable and significant in all windows (+0.28 to +0.38); the MERRA-2 and GLDAS Amazon trends are window-dependent.
- **Pantanal**: temperature is **strongly window-dependent** (CPC ranges from +0.74 to +1.51) and significant in only 3 of 5 windows for MERRA-2 and CPC. Do not over-interpret Pantanal trends.
- **Precipitation** is not robust anywhere except the Pantanal in the full window.

### 9.5 Fire (Figure 5, `fire_trend_and_correlation.csv`)

**Trends in Aug–Oct active-fire counts (MODIS Aqua, 2002–2024, n = 23):**

| Biome | Trend (% of mean per decade) | p (Hamed-Rao) | Note |
|---|---|---|---|
| Cerrado | **−18.1** | < 0.001 (AR(1) p about 0.01) | significant; no Pettitt shift |
| Amazon | −28.5 | 0.185 | not significant; Pettitt shift in 2009 |
| Pantanal | −7.7 | 0.561 | not significant |

The Amazon's apparent −28 % is an **abrupt drop around 2009** followed by a flat period, not a steady decline, which is why a linear trend test does not call it significant. We do not interpret this drop causally. (It is consistent with the timing of deforestation-control policy, but we did not test that.)

**Year-to-year correlation with climate, trends removed (Spearman ρ, p in parentheses):**

| Driver | Amazon | Cerrado | Pantanal |
|---|---|---|---|
| Precipitation | −0.40 (0.062) | **−0.63 (0.001)** | **−0.57 (0.005)** |
| Soil moisture 0–10 cm | −0.41 (0.049) | **−0.81 (<0.001)** | **−0.73 (<0.001)** |
| Soil moisture 10–40 cm | **−0.45 (0.032)** | **−0.56 (0.005)** | **−0.68 (<0.001)** |
| VPD | +0.35 (0.104) | **+0.73 (<0.001)** | **+0.67 (0.001)** |
| Temperature MERRA-2 | +0.31 (0.154) | **+0.43 (0.038)** | **+0.42 (0.045)** |
| Temperature GLDAS | +0.16 (0.461) | +0.23 (0.300) | +0.40 (0.062) |
| Temperature CPC | +0.14 (0.533) | **+0.50 (0.016)** | +0.27 (0.209) |

Bold = p < 0.05, unadjusted for multiple comparisons.

**How to read this.** In the Cerrado and Pantanal, years with higher VPD, drier soil, and less rain than their own trend are consistently years with more fires than their own trend. The signs are exactly what fire physics predicts, and the correlations are strong. Moreover, the **raw** (not detrended) correlations in the Cerrado are weaker (for example VPD ρ = +0.28, n.s.), because the climate trend (worsening) and the fire trend (falling) point in opposite directions and mask the year-to-year link. That contrast is the core of "the twist": **climate drives interannual variability; it does not explain the trend**.

### 9.6 What we conclude, and what we do not

**We conclude** (with the caveats of Section 12):

1. The Cerrado's fire-season climate has become hotter, with higher VPD and drier soils, and this holds across three temperature sources and several windows.
2. Active-fire counts in the Cerrado have decreased.
3. Interannual fire variability in the Cerrado and the Pantanal is strongly tied to VPD, soil moisture, and rain.

**We do not conclude:**

- that climate *causes* the interannual variability (confounders such as ENSO, deforestation, agricultural burning calendars, and enforcement also vary from year to year),
- *why* fire counts fall (we did not test land use, policy, or management),
- that fire *risk* is falling. Falling counts under worsening climate pressure suggest suppression by other factors, not safety.

The "so what" on the deck (watch VPD and soil moisture to anticipate severe seasons) is an **implication of the correlations, not a tested forecast**.

---

## 10. Figures: what each one shows and how to read it

All figures are saved as 300 dpi PNG and vector PDF in `resultados/figures/`. Design rules: color follows the entity (each biome keeps one color), red always means "change that favors fire" (so for rain and soil moisture, red means *drier*), thin marks, direct labels, a hairline grid, no dual axes.

| File | Title | What to look at |
|---|---|---|
| `fig1_study_area` | Study area: Brazilian biomes | The three biomes studied, in color; others in gray |
| `fig2_trend_maps` | Where fire-season climate is changing | Four maps of per-pixel trend (precip, soil moisture 10–40 cm, VPD, temperature). Stippling = survives FDR. Red = favors fire |
| `figS1_supplementary_maps` | Supplementary maps | Soil moisture 0–10 cm, GLDAS temperature, CPC temperature |
| `fig3_false_positives` | False positives in trend-free noise | Four tests vs autocorrelation. The horizontal line is the nominal 5 % |
| `fig4_biome_series` | The same trends, biome by biome | Bars = yearly anomaly (red = favors fire), black line = Sen slope, dotted ▼ = Pettitt shift, text = slope, p, shift year |
| `fig5_fire_and_climate` | Does climate explain fire year to year? | (a–c) fire count series with trend; (d) detrended Spearman matrix, * = p < 0.05 |
| `fig6a_temperature_agreement` | Do three temperature sources agree? | Hexbin of per-pixel trends between pairs of sources, with the 1:1 line |
| `fig6b_window_sensitivity` | Does the trend survive changing the window? | Trend per window per biome; filled dot = p < 0.05, hollow = not |
| `fig6c_temperature_three_sources` | Is there a 2014 step in all three sources? | Anomaly series of the three sources per biome, with Sen slopes |

**Reading tip for Figure 4.** The label in each panel places the slope in bold and a second line "p · shift YYYY". Panels are laid out so that text never sits on top of bars (text moves to the corner opposite the trend).

---

## 11. Quality assurance: what we checked and what went wrong along the way

### 11.1 Validation

- **Statistics validated against a reference library.** Slope and p-values of our vectorized Mann-Kendall/Sen/Hamed-Rao match `pymannkendall` wherever the library returns a number.
- **Simulation validation.** The sanity test (Section 7) quantifies how each test behaves on data where the truth is known.
- **Independent data.** CPC station data provide a check that is not derived from the same model family as MERRA-2 or GLDAS.
- **Window sensitivity** guards against conclusions that depend on end points.
- **Detrended correlations** guard against the trap of two series that merely trend.

### 11.2 Issues found and fixed during development (worth knowing for Q&A)

1. **Old fire cache silently skipped the new ingestion.** An earlier INPE CSV existed, so the FIRMS cell skipped. Fixed by using a new filename `focos_firms_aqua_mensal.csv`.
2. **CPC OPeNDAP failures.** `FileNotFoundError` from OPeNDAP. Fixed with an HTTP download fallback (cleans temp files, combines error messages).
3. **Hamed-Rao `NaN`.** Reference library returns NaN when the weighted autocorrelation sum is negative. Our version clamps the correction factor to ≥ 1. One biome row in `trend_by_biome.csv` still shows `NaN` because that column is computed by the library function.
4. **Figure layout bugs.** Subtitle/title collision on the supplementary maps, text overlapping bars in Figure 4, labels crowding in Figure 6b, and 6c annotations overlapping data. All fixed with explicit headroom and layout rules.
5. **A revised interpretation.** We initially suspected the 2014 temperature step might be an artifact of one product. After adding CPC, the three sources co-move and all show a late-period shift. The correct reading is that the step is real in all three and the difference between them is mostly **amplitude**, not timing.
6. **Interpretation of the first raw correlations.** Raw Spearman correlations between fire and the temperature series looked weak or even of the wrong sign for the Cerrado (for example GLDAS temperature ρ = −0.15 raw versus +0.23 detrended; VPD +0.28 raw versus +0.73 detrended). The reason was the opposite trends masking the interannual link. Detrending both series revealed the strong relationships (Section 9.5).

### 11.3 Things you could still verify yourself (cheap, high-value)

- Count the valid pixels per biome in the output of cell 15, and compare with polygon areas.
- Re-run the sanity test with `n = 24` instead of 25, to match the actual series length.
- Recompute one biome series by hand (for example Cerrado VPD) and check the Sen slope with `pymannkendall.sens_slope`.
- Compare the Aqua-only fire counts against the INPE BDQueimadas reference satellite (`AQUA_M-T`) for a year of your choice.

---

## 12. Limitations and threats to validity

Be ready to state these before anyone asks.

**Statistics**

- **Short series.** n = 24 yearly points. Trend estimates are sensitive to the ends (for instance, the 2010 and 2024 extremes). The window analysis addresses this only partly.
- **The AR(1) null is a model.** Real climate variability includes interannual oscillations (ENSO, Atlantic modes) that are not AR(1). The AR(1) test remains somewhat liberal at high persistence (17 % at φ = 0.8 in the sanity test).
- **Pettitt assumes independence**, so its p-values are descriptive.
- **Multiple comparisons beyond the maps.** FDR is applied across pixels, but not across the many rows of the biome and correlation tables. With about 21 correlations in Figure 5's table, one or two nominal p < 0.05 results could occur by chance. Strong correlations (|ρ| > 0.6, p ≤ 0.005) are not at much risk.
- **Spatial dependence.** Neighboring pixels are strongly correlated (and upsampling to 0.25° increases this). BH-FDR is reasonable under positive dependence but is an approximation, and the effective number of independent pixels is much smaller than 25,600.
- **Monte Carlo noise.** AR(1) p-values are simulation-based. With 20,000 simulations the smallest reportable p is about 5e-5, and p-values near the threshold can differ slightly between runs.
- **Detrended correlation with n = 23** has limited power and uses the Sen trend, which can also absorb part of real signal.

**Data**

- **Grid mismatch.** Native resolutions range from 0.1° (GPM) to 0.5° (CPC, MERRA-2). Interpolation to 0.25° does not create information. This affects per-pixel maps (smoothing, correlated neighbors) more than biome-level series.
- **VPD from monthly means** underestimates absolute values and may dampen variability.
- **Model-based products.** GLDAS soil moisture and MERRA-2 variables are model output. Station-based CPC covers temperature only (no equivalent for soil moisture).
- **Station density** in Amazonian CPC data is low.
- **Aqua orbit drift** after about 2022 changes overpass time and can reduce detections, which could contribute to the apparent fire decline. The decline began earlier, but the drift's effect on the last years is unquantified.
- **Active fire ≠ burned area.** Active-fire counts are sensitive to overpass timing, cloud and smoke, fire size and duration. Burned-area products (for example MODIS MCD64A1) measure something different, and could show different trends.
- **Active-fire counts include agricultural and pasture burning**, not only wildfires. In the Cerrado, a large share of fire is human-managed.
- **Biome boundaries** are polygons at the pixel scale (0.25°); border pixels are assigned by center point.

**Interpretation**

- **Correlation is not causation.** ENSO, deforestation, land-use change, fire-management policy, and enforcement are not modeled.
- **Window dependence in the Pantanal** means we do not make strong claims about Pantanal temperature trends.
- **Single season definition.** Everything is for August to October. Other seasons (for example, May–September for the dry-season onset) may behave differently.

---

## 13. Claim-to-evidence map (for the deck and for Q&A)

| # | Claim (deck slide) | Evidence | Where it lives |
|---|---|---|---|
| 1 | The classic Mann-Kendall test flags 47 % of trend-free persistent series; our test with FDR flags 8.6 % (slide 2) | Simulation, 10,000 series per φ, n = 25 | Cell 9, `sanity_false_positives.csv`, Fig. 3 |
| 2 | Temperature and VPD rise over large areas; rain barely moves (slide 3) | Per-pixel AR(1) test, BH-FDR at 0.10: 45 / 40 / 30 / 20–24 / 6 % of pixels | Cell 17 (`resumo` table), `tendencias_pixel.nc`, Fig. 2 |
| 3 | Cerrado warms in three independent sources (slide 4) | +0.98 / +0.97 / +0.53 °C/dec, all p < 0.001 | `trend_by_biome.csv`, `robustness_temperature_sources.csv`, Fig. 6c |
| 4 | Significant in 15 of 15 source × window combinations (slide 4) | Window sensitivity table (Cerrado rows for T2M, GLDAS, CPC) | `robustness_time_window.csv`, Fig. 6b |
| 5 | A 2014 step in all three Cerrado sources, Pettitt p ≤ 0.012 (slide 4) | Pettitt table: MERRA-2 p = 0.001, GLDAS p < 0.001, CPC p = 0.012 | Cell 30 output, `trend_by_biome.csv` |
| 6 | Cerrado fire counts fall 18 % per decade (slide 5) | Hamed-Rao p < 0.001, AR(1) p about 0.01 | `fire_trend_and_correlation.csv`, Fig. 5 |
| 7 | Amazon and Pantanal fire trends not significant (slide 5) | p = 0.185 and 0.561 | same |
| 8 | Cerrado VPD ρ = +0.73, rain ρ = −0.63, trends removed (slide 5) | Spearman on detrended series, p ≤ 0.001 | same, Fig. 5d |
| 9 | Climate explains interannual variability, not the trend (slides 5–6) | Contrast between raw and detrended correlations, plus the opposite trend directions | `fire_trend_and_correlation.csv` (both columns) |
| 10 | Solid because: 3 sources, 5 windows, AR(1) + FDR, detrended correlations (slide 6) | Sections 7, 9.3, 9.4, 6.8 | this README |
| 11 | So what: watch VPD and soil moisture for severe seasons (slide 6) | Implication of the correlations; **not a tested forecast** | Section 9.6 |
| 12 | Only public data; no personal data (slide 7) | All sources are open NASA/NOAA/IBGE datasets | Section 3 |
| 13 | AI used to help write and refine code, slides, and final text (slide 7) | Disclosure | Section 15 |

---

## 14. Study guide: questions you should be able to answer

**Concepts**

1. What does a Mann-Kendall test actually test? (A monotonic trend, using the sign of all pairwise differences; not a linear trend.)
2. Why is the Sen slope preferred over the least-squares slope here? (Median of pairwise slopes, robust to outliers.)
3. Why does autocorrelation inflate false positives? (S varies more than the independent-data formula assumes, so the observed |S| looks more extreme than it is.)
4. Why correct φ̂ for small-sample bias? (With n = 24, φ̂ is biased low, by about (1 + 3φ)/n, so the null would be too optimistic.)
5. What does FDR control, and why α_FDR = 2α? (The expected share of false discoveries among declared discoveries; for spatial fields with positive dependence, Wilks (2016) suggests 2α to approximate the intended global level.)
6. What does the Pettitt test assume and how did we use it? (One change point, independent data; used descriptively.)
7. Why detrend before correlating? (To avoid spurious correlation from two series that simply trend; isolates year-to-year covariation.)
8. What is VPD and why is it relevant to fire? (Atmospheric drying power; higher VPD dries fuels.)

**Defending the work**

9. *"Why should I trust your trends with only 24 points?"* Sanity test, AR(1) null with bias correction, FDR, three sources, five windows, and we say where it fails (Pantanal).
10. *"MERRA-2 and GLDAS disagree. Which is right?"* We do not pick; CPC stations give a conservative magnitude, all three agree on sign and on the Cerrado step.
11. *"Isn't the fire decline just Aqua's orbit drift?"* The decline in the Cerrado is gradual from 2002 and in the Amazon starts around 2009, before the drift (after about 2022). Drift might affect recent years; VIIRS 375 m is the next check.
12. *"So climate does not matter for fire?"* It matters strongly for *which years* burn more (ρ up to 0.7–0.8), not for the long-term count.
13. *"Is your 'so what' a forecast?"* No. It is an implication of year-to-year correlations; a forecast would need out-of-sample validation.
14. *"What about multiple comparisons in the tables?"* Maps use FDR. Tables are unadjusted; the strongest correlations are far from the threshold. We flag it.

**Exercises that would deepen your understanding**

- Reproduce one biome Sen slope by hand with `scipy.stats.theilslopes`.
- Change `SEASON` to `[5, 6, 7, 8, 9]` and see which conclusions survive.
- Run the sanity test with n = 24 and with φ = 0.9. Does the AR(1) test hold up?
- Replace Aqua fire counts with VIIRS counts (2012 onward) and compare the overlap period.
- Add ENSO (for example the Niño 3.4 index) as a covariate in the detrended fire–climate regression and see how much interannual variance it absorbs.

---

## 15. Privacy, data and AI disclosure

**Privacy and data.** Only open, public datasets from NASA (GPM IMERG, GLDAS, MERRA-2, FIRMS), NOAA (CPC Global Unified Temperature), and IBGE (biome boundaries). No personal data is collected, stored, or processed. FIRMS detections are satellite points on a grid; they are not records about people or properties, and we aggregate them by biome and month. A NASA Earthdata account was used only to download the NASA files; credentials are never stored in the notebooks or in this repository.

**AI assistance.** We used AI (Claude and GPT) to help write and refine the code, the slides, and the final text. The team ran every analysis on the real data and reviewed the results; the numbers in this document come from the CSV files produced by those runs.

**Limits of our answers.** Correlation is not causation; Aqua orbit drift after about 2022; 24-year series are sensitive to their ends; the Pantanal depends on the time window (Section 12).

---

## 16. Glossary

- **AR(1)**: first-order autoregressive process; each value depends on the previous value plus noise.
- **Autocorrelation (φ)**: correlation of a series with itself shifted by one step.
- **BH-FDR**: Benjamini-Hochberg procedure controlling the false discovery rate.
- **CPC**: NOAA Climate Prediction Center product, station-based temperature analysis.
- **ENSO**: El Niño–Southern Oscillation, a major driver of interannual climate variability in South America.
- **FDR**: expected fraction of false positives among the results declared significant.
- **FIRMS**: NASA Fire Information for Resource Management System.
- **FRP**: fire radiative power, the energy a detected fire radiates (MW).
- **GLDAS**: Global Land Data Assimilation System; here, the Noah land-surface model output.
- **GPM IMERG**: Global Precipitation Measurement Integrated Multi-satellitE Retrievals.
- **Hamed-Rao**: Mann-Kendall with a variance correction for autocorrelation.
- **MERRA-2**: Modern-Era Retrospective analysis for Research and Applications, Version 2.
- **Pettitt test**: nonparametric test for a single change point.
- **Reanalysis**: a model that assimilates observations to produce a consistent gridded history of the atmosphere.
- **Sen slope**: median of pairwise slopes; robust trend estimator.
- **Spearman ρ**: rank correlation coefficient.
- **VPD**: vapor pressure deficit; the difference between saturation and actual vapor pressure.
- **Stippling**: dots marking pixels that pass the significance rule on a map.

---

## 17. References

Verify DOIs and page numbers against the original sources before final submission.

**Statistics**

- Benjamini, Y., & Hochberg, Y. (1995). Controlling the false discovery rate: a practical and powerful approach to multiple testing. *Journal of the Royal Statistical Society B*, 57(1), 289–300.
- Hamed, K. H., & Rao, A. R. (1998). A modified Mann-Kendall trend test for autocorrelated data. *Journal of Hydrology*, 204, 182–196.
- Hussain, M., & Mahmud, I. (2019). pyMannKendall: a Python package for non parametric Mann Kendall family of trend tests. *Journal of Open Source Software*, 4(39), 1556.
- Kendall, M. G. (1954). Note on bias in the estimation of autocorrelation. *Biometrika*, 41, 403–404.
- Kendall, M. G. (1975). *Rank Correlation Methods* (4th ed.). Griffin.
- Mann, H. B. (1945). Nonparametric tests against trend. *Econometrica*, 13, 245–259.
- Pettitt, A. N. (1979). A non-parametric approach to the change-point problem. *Journal of the Royal Statistical Society C (Applied Statistics)*, 28(2), 126–135.
- Sen, P. K. (1968). Estimates of the regression coefficient based on Kendall's tau. *Journal of the American Statistical Association*, 63, 1379–1389.
- Spearman, C. (1904). The proof and measurement of association between two things. *American Journal of Psychology*, 15, 72–101.
- von Storch, H. (1999). Misuses of statistical analysis in climate research. In *Analysis of Climate Variability* (pp. 11–26). Springer.
- Wilks, D. S. (2016). "The stippling shows statistically significant grid points": How research results are routinely overstated and overinterpreted, and what to do about it. *Bulletin of the American Meteorological Society*, 97(12), 2263–2273.
- Yue, S., Pilon, P., Phinney, B., & Cavadias, G. (2002). The influence of autocorrelation on the ability to detect trend in hydrological series. *Hydrological Processes*, 16, 1807–1829.

**Data products**

- Beaudoing, H., & Rodell, M. (2020). *GLDAS Noah Land Surface Model L4 monthly 0.25 x 0.25 degree V2.1* (GLDAS_NOAH025_M). NASA GES DISC. Rodell, M., et al. (2004). The Global Land Data Assimilation System. *Bulletin of the American Meteorological Society*, 85(3), 381–394.
- Huffman, G. J., et al. *GPM IMERG Final Precipitation L3 1 month 0.1 degree x 0.1 degree V07* (GPM_3IMERGM). NASA GES DISC.
- Gelaro, R., et al. (2017). The Modern-Era Retrospective Analysis for Research and Applications, Version 2 (MERRA-2). *Journal of Climate*, 30(14), 5419–5454. Product: `M2TMNXSLV` (GMAO, 2015, doi from GES DISC).
- Fan, Y., & van den Dool, H. (2008). A global monthly land surface air temperature analysis for 1948–present. *Journal of Geophysical Research*, 113, D01103. Data: CPC Global Unified Temperature, NOAA PSL.
- Giglio, L., Schroeder, W., & Justice, C. O. (2016). The collection 6 MODIS active fire detection algorithm and fire products. *Remote Sensing of Environment*, 178, 31–41. Data: NASA FIRMS, MODIS C6.1.
- IBGE (2019). *Biomas e Sistema Costeiro-Marinho do Brasil*, 1:250 000. Accessed through the `geobr` package (Pereira, R. H. M., & Gonçalves, C. N.).

**Formulas**

- Murray, F. W. (1967). On the computation of saturation vapor pressure. *Journal of Applied Meteorology*, 6, 203–204 (constants 17.27 and 237.3 in the saturation vapor pressure formula).
- Allen, R. G., Pereira, L. S., Raes, D., & Smith, M. (1998). *Crop evapotranspiration: guidelines for computing crop water requirements*. FAO Irrigation and Drainage Paper 56 (vapor pressure formulas).
