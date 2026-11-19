# RevBench Workbench

**Created by Nipun Dilshan Senevirathne**

RevBench is a Windows desktop workbench for reproducible reverse-engineering studies. It runs genuine Ghidra analysis on a pair of benign Windows x64 executables, displays decompilation side by side, and records the evidence needed to repeat the experiment. The workbench, scoring, and export logic run locally and use no AI service.

The core research question is concrete: **Which program facts can an analyst recover, with which version of the analysis engine, within what time budget?** Use the shipped corpus to test the workflow, then assemble a study with independently sourced benign programs and explicit ground truth.

![RevBench Workbench displaying recorded Ghidra 12.2 evidence from the fixture pack](docs/screenshot.png)

## Start here

1. Extract the entire ZIP to a normal local folder. Keep the folders together.
2. Open `Start-RevBench.cmd` or `app/RevBench.Workbench.exe`. The Windows x64 release includes Ghidra 12.2, Temurin JDK 25, and the .NET 10 runtime.
3. Select **Open demo** and choose a sample. Recorded Ghidra evidence opens immediately and is labeled in the status bar. Select **Run analysis** to create a fresh run with your settings. The workbench validates the PE and fingerprints each executable before invoking Ghidra for static analysis.
4. Use the function selectors to inspect each side. The shipped manifest identifies the intended function on each specimen.
5. Select **Start trial** to answer a question using the assigned build in the left pane. Submit the answer to record elapsed time and reveal the result. For a real study, have a coordinator start the trial before a participant sees the samples or answer key.
6. Export **JSON** for detailed evidence and **CSV** for spreadsheet analysis. Use **Run history** to reopen saved runs.

To use a local Ghidra installation, select **Configure Ghidra** and choose the extracted root containing `support/analyzeHeadless.bat` and `Ghidra/application.properties`. The Ghidra 12.2 release uses a 64-bit JDK 25. [Ghidra releases](https://github.com/NationalSecurityAgency/ghidra/releases) · [Ghidra 12.2 setup guide](https://github.com/NationalSecurityAgency/ghidra/blob/Ghidra_12.2_build/GhidraDocs/GettingStarted.md)

See the [study protocol](docs/study-protocol.md) for experimental design, [pack authoring](docs/pack-authoring.md) for the manifest, [architecture and evidence](docs/architecture.md) for implementation and data formats, and [release verification](docs/verification.md) for test results.

## What is in the workbench

| Capability | What it records or does |
| --- | --- |
| Sample packs | Load a versioned manifest with relative specimen paths, questions, answer rubrics, and optional per-build function selectors. The desktop app also accepts one-off manual pairs. |
| Binary provenance | Identifies Windows AMD64 PE images and records SHA-256, file size, image base, entry point, and subsystem. The runner checks that the snapshot given to Ghidra still matches the verified digest. |
| Headless Ghidra analysis | Analyzes each specimen in a separate local project. Exports bounded per-function decompilation, per-function status, counts, and diagnostics. |
| Paired inspection | Side-by-side decompilation with independent selectors. A declared name or entry address identifies the intended function on that build. Ambiguous and missing selections remain explicit. |
| Analyst trials | Randomly selects one build, displays it on the left with the condition label hidden, times the response, and grades against a small human-authored rubric where supplied. A case with a declared target shows only that function during the trial. |
| Run history | Saves reports and submitted trials locally. A saved analysis can be reviewed without repeating the engine run. |
| Evidence export | JSON retains detailed analysis and the selected trial. CSV gives two specimen rows per run and attaches trial fields to the inspected variant. |
| Command-line workflows | Validate packs, run one case, or run a suite. Use the CLI to record a repeatable baseline before running participant trials in the desktop app. |

## Shipped study corpus

| Case | Study task | Compared representations |
| --- | --- | --- |
| Record width classifier | Recover the accepted lower boundary for a fixed-width record. | Two length checks and an unsigned range check. |
| Freight pricing bands | Recover the last weight eligible for the lowest rate. | Structured branches and conditional moves. |
| Telemetry score saturation | Recover the unsigned input at which a score reaches its ceiling. | Early returns and explicit clamping. |

There are two complete packs, with 12 native executables in total. **Open demo** loads `fixtures/manifest.json`, which compares small hand-authored x64 routines. Use **Open sample pack** to open `fixtures/compiler/manifest.json`, which compares the same reference C under ordinary Clang `-O0` and `-O2` builds. All 12 programs passed their eight boundary vectors on Windows. Independent rebuilds reproduced all 12 fixture hashes. Both CLI suites completed against Ghidra 12.2 with Temurin 25 and recovered each named function. See [fixture provenance and rebuild](fixtures/README.md).

## Configuration

| Option | Default | Allowed range |
| --- | --- | --- |
| Ghidra analysis budget per binary | 90 s | 5–600 s |
| Decompilation budget per function | 12 s | 1–120 s |
| Reported functions per binary | 64 | 2–150 |
| Wall limit per binary | 240 s | 20–1,200 s and greater than the analysis budget |

The per-binary wall time includes startup, analysis, and export. A function may time out while other functions are still usable. Reported and discovered counts can differ when the function limit is hit. `completed`, `partial`, `timeout`, `failed`, and `cancelled` entries carry different evidence, so preserve the state and diagnostics when comparing runs.

The default workbench data directory is `%LOCALAPPDATA%\RevBench`. Set `REVBENCH_DATA` before launching to use another local directory. Exports may contain local paths, decompiled code, program names, and analyst answers. Review an export before sharing it.

## Command line

Examples from the ZIP root using the bundled engine. The CLI creates a run folder and emits JSON and CSV for each specimen pair.

```powershell
.\cli\RevBench.Cli.exe --help
.\cli\RevBench.Cli.exe validate --manifest .\fixtures\manifest.json
.\cli\RevBench.Cli.exe analyze --manifest .\fixtures\manifest.json --case record-width `
  --output .\study-output
.\cli\RevBench.Cli.exe suite --manifest .\fixtures\compiler\manifest.json `
  --output .\compiler-study-output
```

Use `--ghidra` to point to another official installation and `--java` to specify its JDK folder. When the bundle is absent, the CLI also checks `GHIDRA_HOME`. `--help` lists budget flags and exit codes. `validate` prints provenance to standard output. `analyze` and `suite` produce two export files per attempted case and write progress to standard error.

## Build and verify from source

The source targets .NET 10. Install the Windows x64 SDK to build and run the verification suite. The desktop runtime alone is sufficient for a framework-dependent release binary. Ghidra with a compatible JDK is needed for live analysis. Node.js is needed only to regenerate the hand-authored fixture corpus. [Install .NET on Windows](https://learn.microsoft.com/dotnet/core/install/windows)

```powershell
powershell -NoProfile -File .\scripts\Build.ps1
powershell -NoProfile -File .\scripts\Test.ps1
powershell -NoProfile -File .\fixtures\Test-Fixtures.ps1
```

`Build.ps1` uses `artifacts/` by default. Use `-SelfContained` to publish the Windows binaries with the .NET runtime and `-OutputDirectory` to select their destination. To start a source build, open `artifacts/app/RevBench.Workbench.exe`. Set `GHIDRA_HOME` or configure Ghidra in the workbench to use an external copy. `Test.ps1` runs the compiled regression checks and the 12 shipped fixture harnesses after their SHA-256s match. The optional fixture verification without native execution checks hashes only.

Open `RevBench.slnx` to work with the solution in Visual Studio, or build it with `dotnet build RevBench.slnx`.

| Path | Contents |
| --- | --- |
| `src/RevBench.App/` | C# and XAML Windows desktop application |
| `src/RevBench.Core/` | Manifests, PE checks, result models, matching, trial scoring, storage, and export |
| `src/RevBench.Engine/` | Windows process lifecycle and bounded Ghidra integration |
| `src/RevBench.Cli/` | Repeatable console workflows |
| `tools/ghidra/` | Java script executed by the real Ghidra engine |
| `fixtures/` | Two sample packs, 12 native binaries, checksums, source, and fixture builders |
| `tests/` | Regression and data-format verification |
| `docs/` | Study design, evidence architecture, and pack authoring |
| `THIRD-PARTY-NOTICES.md` | Bundled component versions, upstream licenses, and sources |

## Troubleshooting and interpretation

| If you see | Check |
| --- | --- |
| Workbench cannot start | Extract the whole archive. If prompted, install the x64 .NET 10 Desktop Runtime and retry. |
| Invalid Ghidra directory | Select the extracted distribution root. Keep the Java exporter next to the built app under `tools/ghidra/`. |
| Ghidra cannot find a compatible Java version | Install the 64-bit JDK specified for that Ghidra release and configure its directory when available. A JRE alone is insufficient. |
| One build is missing a target or shows multiple matches | Confirm the exact function name from that build or author a locator using its own Ghidra entry address. Names and addresses in different images are independent. |
| Analysis is partial or timed out | Inspect both the specimen-level diagnostic and the function statuses. If you change time budgets, record the new settings as a distinct run. |
| A rubric marks an equivalent answer incorrect | Rubric matching compares Unicode-normalized, case-insensitive strings with normalized whitespace. Include accepted formats in the protocol ahead of time and record adjudication separately when needed. |

RevBench measures analysis behavior on the specimens and study tasks you supply. A decompilation status records whether Ghidra emitted a result under the configured budget. A correct analyst answer records recovery of the specific fact asked for. The shipped corpus is a teaching and regression set. For claims about real software, use a larger held-out corpus and disclose the study conditions.

Ghidra and Java are separately maintained third-party tools under their respective terms. RevBench is credited to Nipun Dilshan Senevirathne.
