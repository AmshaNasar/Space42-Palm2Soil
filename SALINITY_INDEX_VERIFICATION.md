# Salinity index verification

Checked against published sources before use in the Palm2Soil Proof of Concept.
Two problems were found and fixed. **Verdict: all four formulas were correct;
the naming and the band treatment were not.**

---

## 1. The formulas — verified

| Index as implemented | Formula | Attributed to |
|---|---|---|
| `SI_Khan` | √(Green × Red) | Khan et al. |
| `SI_Douaoui` | √(Green² + Red² + NIR²) | Douaoui et al., 2006 |
| `BI` | √(Red² + NIR²) | Brightness Index, Khan et al. |
| `CRSI` | √[((NIR × Red) − (Green × Blue)) / ((NIR × Red) + (Green × Blue))] | Scudiero et al., 2014 / 2015 |

Cross-checked against:

- the **CRAN `SoilSaltIndex`** package documentation, which gives each formula
  with its primary attribution
- **Rafik et al. (2022)**, *Remote Sensing*, which tabulates a wider set of
  indices with sensor and source for each
- **Azabdaftari & Sunar (2016)**, ISPRS Archives XLI-B7, which attributes
  Salinity Index 2 and 4 to Douaoui (2006) and Salinity Index 1 to
  Khan, Rastoskuev et al. (2001)

All four formulas as originally written were **arithmetically correct**. No
equation changed.

> Caveat on sourcing: the equations in both PDFs above are embedded as images,
> so they could not be extracted as text and were read from surrounding tables
> and secondary tabulations. Before the final MVP submission, someone should
> confirm them against the original Khan, Douaoui and Scudiero papers directly.

---

## 2. Problem one — the names are ambiguous

**"SI1" does not mean one thing in the literature.** Examples found while
checking:

| Source | Name | Formula | Sensor |
|---|---|---|---|
| CRAN SoilSaltIndex | SI1 | √(G × R) | generic |
| Rafik et al., ref [55] | SI-1 | B4 / B5 | Landsat 5 TM |
| Rafik et al., ref [57] | S1 | B1 / B3 | IRS-1B LISS-II |
| Rafik et al., ref [52] | SI-KHAN | (B1 × B3)^0.5 | IRS-1B LISS-II |

A ratio, a different ratio, and a geometric mean — all called some variant of
"SI1". Azabdaftari & Sunar additionally attribute "Salinity Index 1" to Khan,
Rastoskuev **et al. 2001**, where the CRAN package cites Khan **et al. 2005**.

**There is also no stable band numbering.** `B1` means a different wavelength in
IRS-1B LISS-II, Landsat 5 TM and SPOT-2. A formula written in band numbers
cannot be carried between sensors without translating to wavelengths first.

### What we changed

Indices are no longer called `SI1` and `SI2`. They are named for their source —
`SI_Khan`, `SI_Douaoui` — and the code comments carry the formula in band terms.
If a reviewer asks "which SI1 is that?", the answer is in the column name.

**Recommendation for the write-up:** never cite a salinity index by number alone.
Give the formula, the source, and the wavelengths used.

---

## 3. Problem two — broadband formulas on narrow bands

This was the real defect in our implementation.

Every one of these indices was developed for a **broadband** sensor — Landsat,
SPOT, IRS-1B, Sentinel-2 — where "green" means a band 30–90 nm wide. Our first
implementation picked a **single ~5 nm Tanager band** per colour. A 5 nm sample
at 570 nm is not the same measurement as a 34 nm Sentinel-2 green band centred
at 560 nm, so the resulting numbers were not comparable with any published
threshold or validation study.

### What we changed

The salinity indices are now computed on **broadband-equivalent reflectance**:
the Tanager spectrum is integrated through ESA's measured Sentinel-2 response
functions (COPE-GSEG-EOPG-TN-15-0007, S2-SRF v5) before the formula is applied.

| Colour | Was (single band) | Now (broadband equivalent) |
|---|---|---|
| Blue | 490 nm | S-2 B2, centre 493 nm, FWHM 64 nm |
| Green | 570 nm | S-2 B3, centre 560 nm, FWHM 34 nm |
| Red | 665 nm | S-2 B4, centre 665 nm, FWHM 29 nm |
| NIR | 860 nm | S-2 B8A, centre 865 nm, FWHM 19 nm |

Note the green band moved 10 nm — our 570 nm pick was inherited from the
repository's BSI definition, not from the salinity literature.

### How much it mattered

| Index | Narrow-band mean | Broadband mean | Bias, as % of sd | Correlation |
|---|---|---|---|---|
| SI_Khan | 0.1942 | 0.1910 | 6% | r = 0.9991 |
| SI_Douaoui | 0.4715 | 0.4702 | 2% | r = 0.9997 |
| BI | 0.4341 | 0.4368 | 4% | r = 0.9999 |
| CRSI | 0.7688 | 0.7796 | **16%** | r = 0.9965 |

**Honest reading: it barely mattered.** Correlations are 0.996 or better, so
nothing in the parcel ranking changes materially. CRSI shifted most because it
is the only one using the blue band, where the broad S-2 B2 differs most from a
narrow sample.

We made the change anyway, for two reasons: the broadband version is what the
published literature means, and it is the version we can defend if asked.

---

## 4. What remains unverified

- **No calibration to dS/m.** These indices are computed and ranked; none is
  tied to a measured salinity value. Thresholds from the literature cannot be
  applied without local ground truth.
- **Source papers not read directly.** Formulas were confirmed from secondary
  tabulations because the primary equations are images in the PDFs.
- **Applicability to Tanager not established.** These indices were validated on
  broadband sensors over specific regions. That they compute cleanly here does
  not mean they are predictive here.

For the Proof of Concept these indices are **descriptive context**, not the
prioritisation mechanism. The ranking is driven by the NDVI–NDRE chlorophyll
residual, which is defined in the notebook and does not depend on any of them.

---

## 5. Sources

- CRAN, *SoilSaltIndex: Soil Salinity Indices Generation using Satellite Data* —
  https://cran.r-project.org/web/packages/SoilSaltIndex/vignettes/SoilSaltIndex.html
- Rafik, A. et al. (2022), *Remote Sensing* —
  https://horizon.documentation.ird.fr/exl-doc/pleins_textes/2022-05/010084646.pdf
- Azabdaftari, A. & Sunar, F. (2016), *Soil Salinity Mapping Using Multitemporal
  Landsat Data*, ISPRS Archives XLI-B7 —
  https://isprs-archives.copernicus.org/articles/XLI-B7/3/2016/isprs-archives-XLI-B7-3-2016.pdf
- ESA/Copernicus, *Sentinel-2 Spectral Response Functions*,
  COPE-GSEG-EOPG-TN-15-0007 v5 — https://sentiwiki.copernicus.eu/web/s2-documents

Primary papers still to consult directly: Khan et al.; Douaoui et al. (2006);
Scudiero et al. (2014, 2015).
