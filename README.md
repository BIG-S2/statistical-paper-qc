# Quality Control for Statistical Papers

An AI-assisted workflow for checking statistical manuscripts for verifiable errors in numerical results, mathematical arguments, and consistency between the main paper and its supplement.

Suggested repository name: `statistical-paper-qc`.

The repository currently contains the `jasa-acs-qc` Claude skill and its help guide. The workflow was designed for JASA Applications and Case Studies (ACS); its consistency, numerical, and proof checks can also support other statistics submissions. Journal-specific structure checks should be adapted to the target journal's current requirements.

## What it checks

| Module | Scope |
| --- | --- |
| A — Main/supplement consistency | Cross-references, theorem assumptions, tables, figures, references, and shared descriptions. |
| B — Numerical verification | Counts, percentages, simulation summaries, convergence rates, reported statistics, and stated parameter values. |
| C — ACS structure | Scientific-question traceability, allocation of manuscript space, and presence of model checking and reproducibility information. |
| D — Proof checking | Hypotheses, algebra, dimensions, limits, existence, optimization claims, and use of external results. |

This is a skill-driven review workflow, not a standalone automated validator. Findings require verification against the manuscript and appropriate statistical judgment. It complements editorial assessment of significance, novelty, and journal fit.

## Repository contents

```text
.
├── README.md
├── jasa-acs-qc.skill            # Original installable Claude skill archive
├── jasa-acs-qc_help.pdf         # Original help guide
└── skills/
    └── jasa-acs-qc/
        └── SKILL.md            # Editable source extracted from the archive
```

## Getting started

The bundled [help guide](jasa-acs-qc_help.pdf) describes installation of [jasa-acs-qc.skill](jasa-acs-qc.skill) as a custom Claude skill. The [readable skill source](skills/jasa-acs-qc/SKILL.md) contains the full workflow and example code.

Provide a main manuscript PDF and, preferably, its supplement. Include a manuscript ID and any scope restrictions, for example:

```text
QC main.pdf and supp.pdf. ID: EXAMPLE-001.
Check consistency, numerical results, and proofs; skip the ACS structure module.
```

The workflow calls for a Word report containing verified findings, their locations and severity, suggested fixes, checks that passed, and revision priorities. Local tools used by the workflow include Poppler (`pdftotext`, `pdftoppm`, `pdfinfo`), Python, NumPy, and python-docx; OCR may be needed for scanned PDFs.

## Verification principles

- Inspect rendered pages before reporting equation errors.
- Search both documents before claiming that something is absent.
- Recompute numerical findings with code and account for rounding and experimental settings.
- State the page-numbering convention and provide precise locations.
- Record successful checks as well as defects.

## Working on the skill

Edit `skills/jasa-acs-qc/SKILL.md`. To rebuild the installable archive from that source, run from the repository root:

```bash
python - <<'PY'
from pathlib import Path
from zipfile import ZIP_DEFLATED, ZipFile

source = Path("skills/jasa-acs-qc/SKILL.md")
with ZipFile("jasa-acs-qc.skill", "w", compression=ZIP_DEFLATED) as archive:
    archive.write(source, "jasa-acs-qc/SKILL.md")
PY
```

## Confidential material

Keep manuscripts, supplements, extracted text, data, and generated reports outside the repository or in the ignored `private/` directory. The repository's `.gitignore` also excludes common local input and output directories. Review files before committing; ignore rules alone do not guarantee confidentiality.

The bundled workflow calls for local handling of manuscript materials and prohibits uploading their text to outside services or searching the web for the manuscript itself.
