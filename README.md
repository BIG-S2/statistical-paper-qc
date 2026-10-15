# Quality Control for Statistical Papers

An AI-assisted workflow for checking statistical manuscripts for verifiable errors in numerical results, mathematical arguments, and consistency between the main paper and its supplement.

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

### 1. Install the skill

#### Claude (web or desktop app)

1. Download [jasa-acs-qc.skill](jasa-acs-qc.skill) using GitHub's **Download raw file** button. This is already a ZIP archive containing `jasa-acs-qc/SKILL.md`; save a copy named `jasa-acs-qc.zip`. Do not upload the ZIP of the entire repository.
2. Enable **Code execution and file creation** in Claude's **Settings > Capabilities**. For managed accounts, the organization must allow code execution and custom skills.
3. Open **Customize > Skills**, choose **+ > Create skill > Upload a skill**, and upload `jasa-acs-qc.zip`.
4. Enable `jasa-acs-qc` in the skills list.

These interface steps follow [Claude's official skills guide](https://support.claude.com/en/articles/12512180-use-skills-in-claude), checked on 2026-10-15. Account settings and interface labels may change. The original [PDF help guide](jasa-acs-qc_help.pdf) explains the audit modules in detail; use the steps above for installation.

If you have cloned the repository, make the uploadable copy with:

```bash
cp jasa-acs-qc.skill jasa-acs-qc.zip
```

#### Claude Code

Claude Code loads skills from folders on disk; no upload is needed. From the root of a clone of this repository, install the skill for your user account:

```bash
mkdir -p ~/.claude/skills
cp -r skills/jasa-acs-qc ~/.claude/skills/
```

To have the installed skill follow `git pull` and local edits, link it instead of copying:

```bash
mkdir -p ~/.claude/skills
ln -s "$PWD/skills/jasa-acs-qc" ~/.claude/skills/jasa-acs-qc
```

To make the skill available in one project only, place the folder at `<project>/.claude/skills/jasa-acs-qc/` instead.

Claude Code picks up new skills without a restart. If `~/.claude/skills/` did not exist when the session started, run `/reload-skills`. To confirm the installation, type `/` in a session and look for `jasa-acs-qc` in the menu.

These steps follow the [Claude Code skills documentation](https://code.claude.com/docs/en/skills), checked on 2026-10-15.

### 2. Prepare the inputs

| Input | Required? | Details |
| --- | --- | --- |
| Main manuscript PDF | Yes | Prefer a searchable PDF. Scanned pages need OCR and visual verification. |
| Supplement PDF | Recommended | Needed for main/supplement cross-checks. |
| Manuscript ID | Optional | Used to identify the report; e.g. `EXAMPLE-001`. |
| Scope | Optional | Specify modules, sections, tables, or theorems to check. |

Use material you are authorized to process in the chosen environment. For a first trial, use a public or synthetic manuscript. Supply the PDFs in the conversation, or accessible local file paths when using Claude Code or another agent with filesystem access.

### 3. Request an audit

Use Claude Opus 5.5 or a more capable model, such as Claude Fable 5.1, with at least high reasoning effort; proof checks and numerical recomputation depend on long chains of careful reasoning. In the Claude apps, choose the model in the model selector before starting the conversation. For Claude Code, see [Running in Claude Code](#running-in-claude-code). Model names are current as of 2026-10-15.

For a statistics paper outside JASA ACS, start with:

```text
Use jasa-acs-qc to audit main.pdf and supp.pdf. ID: EXAMPLE-001.
Run Modules A, B, and D; skip the journal-specific ACS structure module.
Verify findings against the rendered pages and recompute numerical claims.
List checks that passed, checks that could not be completed, and suggested fixes.
Produce EXAMPLE-001_QC_report.docx.
```

For JASA ACS, request Modules A–D and check the journal's current requirements before interpreting Module C. To narrow the work, use prompts such as:

```text
Use jasa-acs-qc to verify only the numbers in Tables 2 and S4.
Show the recomputation supporting each finding.
```

```text
Use jasa-acs-qc to check Theorems 1–3 against their supplementary proofs.
Report errors, gaps, typographic defects, and verified steps separately.
```

#### Running in Claude Code

1. Put the PDFs in a working folder outside this repository, or in the repository's ignored `private/` directory.
2. Check the local tools. Claude Code runs commands on your machine, so Poppler, NumPy, and python-docx must be available there; see [Runtime requirements](#runtime-requirements-and-troubleshooting). If you use a virtual environment, give Claude its interpreter path in the prompt.
3. Start Claude Code in the working folder with a suitable model and effort level:

   ```bash
   cd /path/to/review-folder
   claude --model opus --effort high
   ```

   Use `--model fable` for the more capable Fable model where your plan or provider offers it. In a running session, switch with `/model` and `/effort`; `/model` without an argument opens the model picker and shows the active model.

4. Invoke the skill with `/jasa-acs-qc` followed by the request, or start the prompt with `Use jasa-acs-qc` as in the examples above:

   ```text
   /jasa-acs-qc Audit main.pdf and supp.pdf in this folder. ID: EXAMPLE-001.
   Run Modules A, B, and D; skip the journal-specific ACS structure module.
   For Python steps, use /path/to/.venv/bin/python.
   Produce EXAMPLE-001_QC_report.docx in this folder.
   ```

5. Claude Code asks for permission before running shell commands such as `pdftotext`, `pdftoppm`, and `python`. Review each command before approving it.

The report is saved in the working folder, next to the manuscript. Claude Code runs the tools locally, but the text and page images it reads are sent to the model; see [Confidential material](#confidential-material).

### 4. Retrieve and review the report

The expected output is `<ID>_QC_report.docx`, containing files audited, page numbering, severity counts, findings with precise locations and suggested fixes, verified checks, revision priorities, and a methods note. Request a downloadable file in a hosted session; with filesystem access, the skill specifies saving it beside the manuscript.

To split the output, request `<ID>_consistency_audit.docx` and `<ID>_proof_check.docx`. Review each finding and its supporting evidence before acting on it. A completed audit does not establish that a manuscript is error-free; proof checking here is not formal verification.

## Runtime requirements and troubleshooting

The executing agent needs PDF text extraction and rendering, code execution for numerical checks, and Word document creation. The skill uses Poppler (`pdftotext`, `pdftoppm`, `pdfinfo`), Python, NumPy, and python-docx. The repository contains instructions and code examples; it does not provide a command-line audit program or an end-to-end validated example report.

For a local Python environment, install the Python dependencies in a virtual environment:

```bash
python -m venv .venv
. .venv/bin/activate
python -m pip install numpy python-docx
```

Poppler is a separate system dependency; check availability with `pdftotext -v` and `pdftoppm -v`. On a managed server, use an available software module or ask the administrator to provide it. A hosted Claude session uses its own execution environment, so installing packages on your laptop does not install them there.

| Situation | What to do |
| --- | --- |
| No supplement | Request main-only checks and have the report identify skipped cross-document checks. |
| Scanned PDF | Obtain OCR text or a searchable PDF; still inspect page images before asserting equation errors. |
| Skill not invoked | Confirm it is enabled and explicitly say `Use jasa-acs-qc` in the prompt. In Claude Code, check that `~/.claude/skills/jasa-acs-qc/SKILL.md` exists and invoke `/jasa-acs-qc` directly. |
| Large manuscript or appendix | Start with named tables or theorems and continue in batches; track what remains unchecked. |
| A tool or dependency is unavailable | Ask for the limitation and affected checks to be recorded; do not label unperformed checks as verified. |
| A finding seems incorrect | Ask for the exact page image, assumptions, and numerical reproduction. |

For the full audit instructions and code examples, see [SKILL.md](skills/jasa-acs-qc/SKILL.md).

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

A Claude Code installation made by copying does not follow these edits; copy the folder again, or use the linked installation described in [Claude Code](#claude-code). In the Claude apps, upload the rebuilt archive.

## Confidential material

Keep manuscripts, supplements, extracted text, data, and generated reports outside the repository or in the ignored `private/` directory. The repository's `.gitignore` also excludes common local input and output directories. Review files before committing; ignore rules alone do not guarantee confidentiality.

The bundled workflow calls for local handling of manuscript materials and prohibits uploading their text to outside services or searching the web for the manuscript itself. Installing a skill does not make a hosted AI service run offline. For confidential manuscripts, use only a processing environment permitted by the applicable review and data-handling requirements.
