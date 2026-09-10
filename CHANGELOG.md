# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

The version number tracks the ATC-138 methodology this codebase implements. It is not an API contract, and it continues the lineage that began in the MATLAB [PBEE-Recovery](https://github.com/OpenPBEE/PBEE-Recovery) repository. Released versions carry two segments, `1.3` and `1.4`, with a third added for a patch release such as `1.4.1`.

A change to the methodology means a change to the assessment logic, the fragility library or the damage-state attributes. Any of those makes the next release a minor one, and every such change is listed in that release's Methodology section, because they change simulated recovery times. A patch release carries corrections, documentation, packaging and tooling, and no methodology change. A correction can still change results, and where it does, the Fixed entry states which models are affected.

## [Unreleased]

---

## [1.4] - 2026-09-10

This release brings the component library up to the current ATC-138 fragility set, raises the safety class assigned to the third damage state of gravity shear tabs, and corrects defects in the red-tag and fire-suppression logic. The applied methodology changes alter simulated recovery times; the Methodology entries below state the scope of each. The packaging configuration is committed, so the published package can be rebuilt from the repository, and releases now go to PyPI automatically.

### Methodology

- **Gravity shear tab damage state 3 raised to Safety Class 3.** The third damage state of the bolted shear tab gravity connections, matched by the pattern `B1031\.00\w+` and covering `B1031.001` in the packaged component table, now maps to safety class 3 instead of safety class 2 in `damage_state_attribute_mapping.csv`.

    *Why:* the damage state is described as "complete separation of shear tab, close to complete loss of vertical load resistance", which matches the safety class 3 definition of severe damage that compromises both the lateral and the vertical load carrying capacity of the component. The proposal for the change cites the experimental work behind the FEMA P-58 fragility, which consistently reports complete fracture of the tab at the median drift assigned to this damage state, which implies a significant loss of vertical load carrying capacity (Deierlein and Victorsson 2008, FEMA P-58/BD-3.8.3; Liu and Astaneh-Asl 2000, *Journal of Structural Engineering* 126(1), 32-39). The same mapping was applied during development of the 2026 NEHRP Seismic Provisions Appendix A, where the component was used in its updated state to prequalify several lateral force-resisting systems (Pham et al. 2026, Functional Recovery Design Parameters for Five Additional Seismic Force-Resisting Systems in the 2026 NEHRP Recommended Provisions, *Earthquake Spectra*, in review).

    *Scope:* safety class filters are cumulative, since `red_tag.py` selects every component at or above the class being evaluated, so this damage state now counts toward the safety class 3 tally as well as the classes below it. The share of the shear tab population that has to be damaged before the system trips a red tag therefore falls from 25% to 10%, and the inspection trigger, set at half the red tag thresholds, falls from 12.5% to 5%. The change author's assessment is that the upgrade occurs at a drift demand that would generally already trigger red-tagging through other global consequences such as residual drift, so a large shift in outcomes is not expected.

- **Component library updated to the current ATC-138 fragility set.** 326 entries were retired from `component_attributes.csv` and 42 were added, with 16 new rows in `damage_state_attribute_mapping.csv` covering the added components.

    The retired entries are product-specific fragilities for proprietary systems: CoreBrace buckling-restrained braces (300 entries), three lines of proprietary steel moment connections, DuraFuse (8 entries), SidePlate (4) and Simpson Strong-Tie Yield-Link (4), the DuraFuse repairable residual drift component (1), and Taylor viscous dampers (6). Two unreinforced masonry infill wall entries and one light-frame stair placeholder were retired as well.

    Twenty-four of the added entries are generic steel buckling-restrained braces, described by conformance vintage, beam-to-column connection, brace configuration and brace weight: 12 conforming to AISC 341-22 and 12 for pre-AISC 341-22 designs. The other 18 are slender concrete walls from the ATC-138 SCSD update, described by wall thickness, height and length; 9 of those are marked ATC-138 Beta and are load-bearing variants copied from `B1044.091`. The new damage-state rows extend fault-tree coverage to the generic BRBs, both pre- and post-2016, and to flexure-controlled reinforced concrete shear walls in load-bearing and non-bearing configurations.

    *Why:* a component attribute entry gives the tool the unit, quantity and system assignment it needs to place a component in the fault tree, but a complete assessment also needs the component's fragility definition to establish when the damage occurs. The retired entries could not be carried through an assessment from publicly available model data. Describing these component types by their engineering characteristics instead of by product line makes the library's coverage explicit and keeps it usable from public data.

    *Scope:* a model that references a retired fragility id now prints `Warning: skipping components with missing component attributes: <ids>` and leaves those components out of the assessment. The run completes and does not fail, so recovery times will be shorter than they were at v1.3. Check the run log if you have v1.3 results to compare. Anyone who holds the supporting data for a retired component can restore it by placing an edited `component_attributes.csv` in the model directory, which overrides the packaged table; see "Customizing Static Tables" in the README.

### Added

- **Damage consequences in the recovery output.** The recovery output file now carries the damage consequences alongside the functionality results, as additional top-level keys: `red_tag`, `red_tag_impact`, `inspection_trigger`, `repair_cost_ratio_total`, `repair_cost_ratio_engineering`, `simulated_replacement_time`, and the door-racking counts. At v1.3 these were computed and then discarded. Existing keys are unchanged, so a consumer that reads the file today keeps working.
- **Custom output file name.** `run_analysis()` takes an `output_file` argument and the CLI exposes it as `--output_file`. The default remains `recovery_outputs.json`.
- **Duplicate-input validation on the Pelicun path.** `convert_pelicun()` now stops with an explicit message when the damage or loss tables contain duplicate location-direction-damage-state rows distinguished by a `uid` suffix. Previously the duplicates were carried through silently.
- **Two more reference models in the test suite.** The suite grew from two archetypes to four, adding a one-story reinforced concrete shear wall model and a twelve-story reinforced concrete moment frame, each with a 20-run reference set. The test itself runs each model once with a fixed seed and compares it against that reference; the pass criteria are in `tests/compare_runs.py`.
- **Automated PyPI publishing.** Publishing a GitHub Release builds the wheel and sdist, verifies that the release tag and the version in `pyproject.toml` agree, checks the package metadata, and uploads to PyPI through Trusted Publishing after a maintainer approves the deployment. No API token is stored in the repository. The README's "Releases" section describes the flow.
- **An MIT `LICENSE` file and a `CITATION.cff`.** The citation file declares `cff-version` 1.2.0 and `type: software`, and both DOI fields carry a bare DOI as the schema requires. The software DOI is now the Zenodo concept DOI, `10.5281/zenodo.20045327`, which resolves to the newest release. A citation generated from this file therefore points at the current release; the v1.3 file named that one version specifically.

### Changed

- **Output serialization.** Recovery outputs are converted from NumPy types with a recursive routine that handles arbitrary nesting depth and descends into lists and tuples, replacing hand-written loops that stopped at five dictionary levels and descended only into dictionaries. All four reference models still match their references. The new routine also converts NumPy scalar types that the old loops passed through untouched.
- **Reference comparison for the two new models.** The one-story shear wall and twelve-story moment frame models are compared against seeded Python runs; their MATLAB reference output was removed. The two original archetypes keep their MATLAB comparison, so parity with the MATLAB implementation is still covered by the suite, on those two models.
- **High-level comparison tolerance.** The high-level metric now passes when the relative difference is within 4% or the absolute difference is within 2 days; before, only the relative check applied. `tests/detailed_inspection_guide.md` was renamed to `tests/README.md`.
- **README.** Import examples updated for installed-package usage, a stale testing link corrected, and new pointers to the changelog and to the release process.
- **Packaging.** The build backend is now hatchling, and the wheel and sdist exclude cache and Finder metadata files. License metadata is declared as the SPDX expression `MIT` together with `license-files`, matching the `LICENSE` file the repository carries; the published 1.3.0 package declared no license. The project URLs now properly point at the OpenPBEE repository.
- **Version numbering.** The version in `pyproject.toml` and `CITATION.cff` now uses two segments, `1.4`, so it matches the `v1.4` tag exactly.

### Fixed

- **Red-tag evaluation failed on components with an alternate structural system.** A component's primary and alternate structural system identifiers were added together instead of being collected into one set, so a component assigned to structural system 5 with 6 as its alternate contributed a phantom system 11 to the list of systems to evaluate. No component matches system 11, so the tool raised `ValueError: zero-size array to reduction operation fmax which has no identity`. The next entry's empty-series guard is the other half of this defect.

    *Scope:* models containing any of the six components that carry a non-zero alternate structural system, namely the steel column base plates (`B1031.011a-c`) and the welded column splices (`B1031.021a-c`), each assigned to structural system 5 with 6 as the alternate, and with red-tag calculation enabled, which is the default. Such a model produced no results at all at v1.3, so there are no wrong numbers to re-check, only runs that could not complete. Components with no alternate system were unaffected.

- **Non-numeric replacement times could fail the red-tag and recovery-metric calculations.** `simulated_replacement_time` is now coerced to a numeric type before it reaches the red-tag logic, the recovery-metric extraction and the repair-schedule helpers. The red-tag path also gained guards for a structural series that matched no components and for division by a zero component count; that case is now treated as not tagged.

    *Scope:* runs whose replacement time reached these functions as a non-numeric value, and red-tag runs with an empty structural series. Both previously raised or produced `NaN`.

- **Fire suppression recovery read the wrong quantity.** The building-wide fire suppression branch read a dictionary key that does not exist, and a second call passed an array where NumPy expected an axis, which raises a `TypeError`. Neither path produced a wrong number; both stopped the run.

    *Scope:* runs with the fire watch option disabled and fire sprinkler components present. The fire watch option is enabled by default, so a default run never reached this code.

- **Component list built from named columns instead of positions.** The input builder identified the metadata columns of `comp_population.csv` by position, so a file whose `story` and `dir` columns sat elsewhere produced a corrupted component list. The component list is now built by matching those names. The column-filtering step further down still assumes the metadata columns come first, so such a file is not yet handled end to end.
- **`np.row_stack` replaced with `np.vstack`.** The two are aliases, so results are unchanged; the former is deprecated in current NumPy.
- **Recovery trajectory plots.** Line styles and draw order were adjusted so overlapping trajectories stay legible.

### Removed

- **`requirements.txt`.** Every dependency it listed is declared in `pyproject.toml`, which has been the single source of truth since v1.3. Install with `pip install -e .` or `pip install atc138`.

---

## [1.3] - 2026-03-31

The first release of the Python codebase. It converts the MATLAB [PBEE-Recovery v1.2](https://github.com/OpenPBEE/PBEE-Recovery/releases/tag/v1.2) implementation into an installable Python package and reproduces its results. Development continues on the Python codebase alone; the MATLAB codebase is deprecated.

### Methodology

- **Aligned with the ATC-138-6 / FEMA P-58-8 report.** The assessment method implemented here is the one documented in the ATC-138-6/FEMA P-58-8 report. The methodology changes relative to the previous MATLAB release, v1.2, were made in the MATLAB codebase and are inherited here: temporary repair times were developed, the impedance factor models were updated, component safety class mapping was updated, and existing fragility models were revised. The [PBEE-Recovery v1.2 release notes](https://github.com/OpenPBEE/PBEE-Recovery/releases/tag/v1.2) document them, and the full fragility model details are released with the report.

- **No methodology change originates in the Python port.** The port was written to reproduce MATLAB PBEE-Recovery v1.2: establish a Python implementation that matches the MATLAB one, and defer method changes to later versions. Outputs from the two codebases were compared, and the differences in calculated recovery times fall within the expected range of variation from probabilistic sampling, so for practical purposes the two produce the same results.

### Added

- **Installable Python package.** The codebase is installable with `pip install -e .` and importable as `atc138`, with the static data tables bundled inside the package.
- **Command-line interface.** `atc138 <input_dir> <output_dir>`, also reachable as `python -m atc138.cli`, replaces the practice of editing a variable at the top of a driver script. It takes explicit input and output directories, a `--seed` for reproducible runs, and a `--force_rebuild` flag that regenerates `simulated_inputs.json` even when one is present.
- **Built-in input builder.** Input generation moved into the package as `atc138.input_builder` and runs automatically when `simulated_inputs.json` is missing, so the earlier step of copying a script out of the repository and hand-editing it is no longer necessary. A user's `optional_inputs.json` is deep-merged over the packaged `default_inputs.json`, so a partial override no longer has to restate every default.
- **Pelicun integration.** `convert_pelicun()` builds the native input set from Pelicun output (`DMG_sample.csv`, `DV_repair_sample.csv`, `CMP_QNT.csv`, `DL_summary.csv`, `general_inputs.json`). Two conversions in it are worth knowing about, because both are easy to get wrong by hand. Damage quantities are converted from Pelicun's physical units into the P-58 component unit counts the engine expects, using `unit_qty` from `component_attributes.csv`, with metric-to-imperial conversion where needed. Pelicun does not track which side of a building a cladding component sits on, so side assignment is drawn at random, optionally weighted by a `side_damage_ratio.csv`.
- **Overridable static tables.** Placing a copy of any of the seven packaged data tables in a model directory overrides the packaged version for that run, which is how a project supplies its own component attributes, damage-state mapping, systems, subsystems, tenant function requirements, impeding factor medians or temporary repair classes.
- **Integration test harness and continuous integration.** A pytest suite compares full model runs against MATLAB reference output. It reports high-level, area-under-curve and pointwise metrics, and gates the test on the high-level ones; the other two are printed for inspection behind flags that are off by default. GitHub Actions runs the suite across Python 3.9 through 3.14 on Linux, macOS and Windows.

### Changed

- **Source layout and module names.** The code moved into a `src/atc138/` package, and the MATLAB-derived file names were mapped to new ones: `main_PBEE_recovery.py` is now `engine.py`, `driver_PBEE_recovery.py` is `driver.py`, `fn_red_tag.py` is `red_tag.py`, and `static_tables/` is `atc138/data/`. Code that imported the old module paths needs updating.
- **Example models** moved from `inputs/example_inputs/<model>/` to `examples/<model>/`.

### Removed

- **The `inputs/` directory and the copy-and-edit input workflow**, including `inputs/Inputs2Copy/build_input.py`, superseded by the built-in input builder.
- **`plotModel_PBEErecovery.py`**, whose plotting logic moved into `plotters/main_plot_functionality.py` as `plot_results()`.
- **`demo.ipynb`**, which described the pre-package layout and is superseded by the README.

---

## Earlier releases

Versions 1.0.0 through 1.2 were released from the MATLAB [PBEE-Recovery](https://github.com/OpenPBEE/PBEE-Recovery) repository, and its [release notes](https://github.com/OpenPBEE/PBEE-Recovery/releases) document them. Version 1.3 is the first release of this Python codebase and continues that numbering.
