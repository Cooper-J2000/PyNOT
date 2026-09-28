# Local modifications (`local/alfosc-tweaks`)

This branch tracks the local modifications applied to our PyNOT-redux installation for
NOT/ALFOSC spectroscopy. Branch base: **v2.4.4** (`3ce1ac1`, upstream `master`, 2026-10-03);
previously branched off v2.4.1 (`53ad8d82`), rebased onto upstream `master` on 2026-09-28 and
again on 2026-10-03. Each item is a separate commit so the branch can be rebased onto a new
upstream release without hunting for patch context. Item 5 has since been merged upstream
(PR #56) and is *no longer* carried by this branch.

| # | Commit subject | Files | Upstream status |
|---|---|---|---|
| 1 | Same-night HeNe preference for arc lamp selection | `pynot/data/organizer.py`, `pynot/redux.py`, `pynot/response.py` | not submitted |
| 2 | mask1D: weight bad pixels instead of killing whole columns | `pynot/extraction.py`, `pynot/extract_gui.py` | not submitted |
| 3 | Local defaults: non-interactive, `prefer_lamp: HeNe` | `pynot/calib/default_options.yml` | local preference only |
| 4 | Locally calibrated 11-line `al-gr4` pixel table | `pynot/calib/al-gr4_pixeltable.dat` | instrument-specific data |
| 5 | Response function: keep first/last reference points (far-red flux fix) | `pynot/response.py` | **merged upstream: PR #56** (v2.4.4, merge `90856c7`) — local patch dropped |

## 1. Arc lamp selection: prefer same-night HeNe

Upstream picks the arc that matches the image (grism/slit/filter) and is **closest in
time**, ignoring the lamp type. Daytime calibration arcs (`He`, `Ne`, `ThAr`) therefore
win over the night-time `HeNe` arcs. Combined with a global pixel table indexed by grism
only, this produced an **11.45 Å systematic residual** in the wavelength solution of the
standard star HD19445 (2026-08-20 night).

`organizer.select_arc_frame()` restricts the candidates to arcs of the preferred lamp on
the same night (MJD ± 0.4 d), then takes the closest in time, and falls back to the
original behaviour when no same-night arc of that lamp exists. Both selection call sites
are patched:

- `redux.py` — science frames (`grism=True, slit=True`)
- `response.py` (`task_response`) — standard star / response function
  (`grism=True, slit=False, filter=True`)

Toggle in the parameter file: `identify: prefer_lamp: HeNe` (default `HeNe`, empty
string disables). Result: HD19445 residual 11.45 Å → 1.27 Å; Balmer line positions
improved from −9…+17 Å to 0…+5 Å.

## 2. mask1D: bad pixels weighted by profile flux

Root cause of long gaps in extracted spectra: `astroscrappy` flags spurious cosmic rays
in the sky region, and the upstream 1D mask test

```python
mask1D = np.sum((1-M)*P, axis=0) / np.sum((1-M)*P**2, axis=0) > 0
```

is satisfied by *any* bad pixel with a non-zero profile value. Since the Moffat wings are
non-zero everywhere, a single spurious flag far from the aperture core masked the entire
column — e.g. a contiguous 202-pixel hole at 4827–5511 Å in one 2026-08-08 frame.

Fix: mask a column only when the bad pixels carry more than 5 % of the total profile
flux:

```python
bad_weight_frac = np.sum((1-M)*P, axis=0) / np.sum(P, axis=0)
mask1D = bad_weight_frac > 0.05
```

Applied to both extraction paths (`extraction.py` for automated extraction,
`extract_gui.py` for interactive extraction). Bad pixels in the aperture core are still
masked as before.

## 3. Local default options

`calib/default_options.yml` as shipped is interactive for `identify`; we run the pipeline
non-interactively by default and enable the GUI explicitly with `-i` / `--no-int` or per
step in the parameter file:

- `identify.interactive: False`, `extract.interactive: False`, `response.interactive: False`
- `identify.prefer_lamp: HeNe` (see item 1)

## 4. Locally calibrated `al-gr4` pixel table

`calib/al-gr4_pixeltable.dat` is replaced by our own 11-line table (interactively
identified, 2026-08-09) used as the global wavelength reference for Grism #4. Re-identify
the wavelength solution whenever the night-to-night drift exceeds `fit_window`
(10 px ≈ 34 Å).

## 5. Response function endpoints — merged upstream, no longer carried here

`response.py` implements the response calculation twice. The standalone
`calculate_response()` protects the ends of the curve
(`good[:3] = True; good[-3:] = True`), but the copy used by the pipeline inside
`task_response()` had those two lines commented out, so the bare criterion was applied:

```python
good = np.abs(flux0 - med_flux_tab) < options['response']['kappa']*noise   # kappa = 2
noise = 1.5 * MAD(flux0 - med_flux_tab)
```

That criterion is *absolute*, while the standard star's count rate varies by a large factor
across the spectrum, and `median_filter` runs along the *index* axis of the reference table,
whose wavelength sampling jumps to 400–500 Å in the far red (rows beyond the extracted range
are dropped). For HD19445 with ALFOSC/grism 4 the last three usable rows — 8370, 8780 and
9300 Å, i.e. every point that constrains the sensitivity above ~8000 Å — were rejected, the
response spline was extrapolated past ~8100 Å and the derived sensitivity came out
0.25–0.42 mag too low. Flux-calibrated spectra were therefore 20–45 % too bright for
λ > 8400 Å.

We carried the fix locally (commit `c7c2ba6`) while it was verified on two nights of
NOT/ALFOSC grism-4 data, then submitted it upstream. Jens-Kristian Krogager merged it on
2026-10-03 (**PR #56**, merge commit `90856c7`, released in **v2.4.4**), so the local commit
was **dropped** in the v2.4.4 rebase: `pynot/response.py` in this branch is now identical to
upstream. Do not re-add it — just make sure the installed version is ≥ v2.4.4.

## Rebuilding an installation from this branch

```bash
git clone -b local/alfosc-tweaks https://github.com/Cooper-J2000/PyNOT.git
~/miniconda3/envs/pynot/bin/pip install --no-deps --force-reinstall ./PyNOT
# note: the wheel ships code only -- copy calib/ data files (standard star library, pixel
# tables) from the source tree as needed
```

The original (pre-change) `al-gr4` table is kept in the working copy of our installation
as `calib/al-gr4_pixeltable.dat.orig`.

## Reporting upstream

- Most of the upstream fixes this installation depends on are already merged:
  PR #47 (NumPy 2 compatibility in `pynot phot`, from this fork), #50–#52 (auto-extraction
  `OverflowError`), #49 (`response.match_slit` / `match_date`), #54 (`save_database`
  in-place `+=` leak, from issue #53).
- Item 5 is **merged**: PR #56 (`response: keep first/last reference points in
  task_response`), submitted 2026-09-28, merged by jkrogager on 2026-10-03 (merge commit
  `90856c7`, released in v2.4.4, and the head branch was deleted from the fork afterwards).
- Items 1 and 2 above are the remaining candidates for an upstream PR.
