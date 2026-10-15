---
name: jasa-acs-qc
description: Quality-control audit of a JASA ACS (or any statistics) submission — main/supplement consistency, numerical cross-checks, ACS-structure checks and proof checking — with verified findings in a Word report.
---

# JASA ACS Quality-Control (QC) Audit

This skill audits a statistics manuscript (main PDF + supplement PDF) for **verifiable defects**:
things that are wrong, missing or inconsistent as printed, and that the authors cannot dispute
once shown. It complements an editorial review (significance, novelty, fit); it does not replace
it. Use it when asked to "QC", "audit", "check consistency", "check the proofs", "check the
numbers", or "verify the supplement" of a submission, for JASA Applications and Case Studies or
any journal of similar standard.

The guiding principle: **reading finds the loud errors; scripted, mechanical passes find the ones
that decide whether a theorem or a table is true.** Every finding must be verified before it is
written, and the report must say what was checked and found correct, not only what was wrong.

---

## 0. Ground rules (apply to every module)

1. **Render before asserting any equation-level finding.** `pdftotext` (both `-raw` and `-layout`)
   drops outer brackets, misplaces subscripts and reorders inline symbols. Before calling any
   display or inline expression wrong — including in a grammar/typo comment — render the page and
   look at it:
   ```bash
   pdftoppm -f P -l P -r 110 -png supp.pdf page   # then view page-P.png
   ```
   Supplements often print page numbers that continue from the main text (printed p.48 = PDF
   p.13). Compute the offset once and record it.
2. **Grep before writing "the paper never does X".** A wrong absence claim discredits the whole
   report. Search both PDFs for the term and its variants (e.g. "sensitivity", "leave-one-out",
   "outlier", "robust", "clip", "odds ratio") and mention the search in the finding when it
   strengthens it.
3. **Look for the authors' own competing usage before calling a step wrong.** The strongest
   form of a finding is "equation (A) and equation (B) cannot both hold", using two places in the
   paper itself.
4. **Check whether two differing numbers belong to different experiments** before calling them
   inconsistent. Different replicate counts across tables are usually correct and only a
   captioning gap. Downgrade rather than defend.
5. **When a derivation, fixed point or algorithm step is disputed, simulate it.** Implement the
   paper's own estimator/iteration on a tiny synthetic instance and evaluate both candidate forms.
   A reproduced number ("the printed step raises the objective from 350.6 to 354.0; the
   majorizer minimizer lowers it to 307.6") ends the argument.
6. **Confidentiality.** Manuscripts under review are confidential. Work only on local copies; do
   not upload the text to outside services or search the web for the manuscript itself.

---

## 1. Setup

```bash
pdftotext -layout main.pdf main_layout.txt;  pdftotext -raw main.pdf main_raw.txt
pdftotext -layout supp.pdf supp_layout.txt;  pdftotext -raw supp.pdf supp_raw.txt
pdfinfo main.pdf | grep Pages;  pdfinfo supp.pdf | grep Pages
```
Insert page markers if needed (`pdftotext -f N -l N` per page) so every finding can cite a page.
JASA proof PDFs carry a "For Peer Review Only" watermark whose letters get interleaved into the
extracted text; single stray letters from that phrase inside words are artifacts, not typos.

Record: manuscript ID, title, page counts, supplement page offset, list of theorems/lemmas/
assumptions, list of tables/figures in each file.

Create a task list with one item per module below plus a final "verify and build report" item.

---

## 2. Module A — Main ↔ supplement consistency (five passes)

**Pass A1 — Resolve every pointer.** Regex all main→supplement references and all
supplement→main references, check each target exists and contains what the citing text says.
Then invert: list every object defined in the supplement and count citations to it, allowing for
range citations ("Figures S5–S31").
```python
import re, collections
main, supp = open('main_raw.txt').read(), open('supp_raw.txt').read()
pat = r'\b(Section|Table|Figure|Remark|Algorithm|Lemma|Theorem|Proposition|Corollary|Assumption|Condition|Appendix|Equation)\s*\(?(S?\.?[A-Z]?\d+(?:\.\d+)*)\)?'
cited_in_main = collections.Counter(re.findall(pat, main))
cited_in_supp = collections.Counter(re.findall(pat, supp))
defined_supp  = set(re.findall(r'^\s*(Table|Figure|Algorithm)\s+(S?\d+)[:.]', supp, re.M))
# report: cited-but-undefined (dangling) and defined-but-never-cited (orphaned)
```
Finds dangling references (a cited "Table 3" that does not exist), wrong targets, and orphans.

**Pass A2 — Diff each theorem statement against the assumptions its proof actually uses.**
Highest-yield pass. Extract the statement (main text) and the proof (supplement); list the
conditions term by term. Three recurring patterns:
- the proof **adds** a condition the theorem omits — then check whether the paper's own
  simulations or data analysis violate it;
- the proof **drops** a condition the theorem states but the argument still needs;
- a supporting lemma is proved under a **stronger** condition than the theorem claims (e.g. a
  plug-in assumed consistent while the theorem says "not necessarily equal to the true values").
Also test every "under the same assumptions as Theorem k" — the later proof usually adds one.

**Pass A3 — Match per-item displays one-to-one.** When the supplement has one figure per row of
a main table, check: same count, same identifiers, same order, no duplicates, and every number
quoted in a caption equals the table cell (signs included). Diff the shared columns of a main
table and its extended supplementary version digit for digit.

**Pass A4 — Diff the reference lists.** Works discussed in the main text but listed only in the
supplement's bibliography (or vice versa); duplicate author-year entries cited identically;
citations with no entry.

**Pass A5 — Diff every shared description of the same object.** Cohort size, inclusion criteria,
covariate list and count, exposure/outcome prevalence, tuning constants, replicate counts,
notation for the same event. Line the two descriptions up in a small table in the report — the
table itself is the argument.

---

## 3. Module B — Numerical and claims verification

Run whichever apply; each has repeatedly found real errors in published-quality submissions.

**B1 — Marginal reconciliation.** When two descriptive tables partition the same cohort along
different axes (e.g. by site and by year), their size-weighted marginals must agree on every
variable, and both must sum to the stated N. Compute it.

**B2 — SE-vs-rate consistency for simulation tables printed as `p (SE)`.** Infer the replication
count R from the bulk via R = p(1−p)/SE², then flag entries inconsistent with it. Catches
transposed digits that eyeballing misses (e.g. a coverage printed `0.552 (0.010)` whose SE
matches 0.952).
```python
import numpy as np
def se_check(pairs, tol=0.35):            # pairs = [(label, p, se), ...]
    R = np.median([p*(1-p)/se**2 for _, p, se in pairs if 0 < p < 1 and se > 0])
    out = []
    for lab, p, se in pairs:
        exp = np.sqrt(max(p*(1-p), 1e-12)/R)
        if abs(se-exp)/exp > tol:
            q = (1 + np.sqrt(max(1-4*se**2*R, 0)))/2   # rates whose SE equals the printed SE
            out.append((lab, p, se, round(exp,4), (round(1-q,3), round(q,3))))
    return round(R), out
```

**B3 — Convergence-rate check.** Interval lengths, RMSEs or SEs across increasing n (or T)
should shrink at the rate the paper's own theory implies (usually n^(−1/2)). Compute
log(ratio)/log(n ratio). A main-text table behaving like n^(−1) and a supplementary table like
n^(−1/2) for the same estimator is a finding.

**B4 — Recompute the paper's own statistic; check orderings.** When a p-value or score is a
monotone function of a statistic with reported inputs (e.g. p = F̂₀(T)), the ranking by p must
equal the ranking by T. Compute T for every row from the reported parameter estimates and count
discordant pairs. Also run the weaker strict-dominance check (if row A beats row B on every
input, A's p must be no larger). If dominance passes but the ordering fails, the table is
consistent with *some* monotone statistic but not with the reported parameters — say exactly
that, then grid-search the parameter for a concordant fit ("your ordering matches α ≈ 0.28, not
the printed α = 0.944").

**B5 — Check stated "true" parameter values against the paper's own estimation tables.** When
the text says "we generate from Beta(a,b), so the true C = …", derive C from the paper's own
parameterisation, verify numerically, and compare with the mean column at the largest sample
size. Common failure: confusing B(a,b), 1/B(a,b) and 1/(αB(a,b)). The table is the truth.

**B6 — Arithmetic in the text.** Percentages vs counts, totals in flow diagrams vs analysis N,
CIs that do not contain their point estimate, odds ratios vs reported coefficients (exp(β)),
p-values vs reported estimate/SE.

---

## 4. Module C — ACS structure QC (mechanical, not editorial)

These are counted, not judged; the editorial verdict is left to the reviewer.
- **Questions → answers traceability.** List every scientific question stated in Section 2
  (or the introduction). For each, find the subsection/table/figure that answers it. Report
  unanswered questions and analyses that answer no stated question.
- **Data-analysis share.** Count main-text pages devoted to the real-data analysis (exclude
  references and appendices); report the percentage against the ACS expectation of > 25 %.
- **Simulation placement.** Pages of simulation material in the main text (ACS expects ≤ ~1 page
  of summary, details in the supplement).
- **Model checking / sensitivity / baseline.** Grep-verified presence or absence of residual or
  posterior-predictive checks, sensitivity analyses, comparison with at least one competing
  method, and uncertainty quantification on the headline results.
- **Reproducibility materials.** Code and data availability statement present; ACS
  reproducibility materials referenced; software versions/seeds stated where results depend on
  them.

---

## 5. Module D — Proof check (when there is a theory appendix)

Work theorem by theorem. For each, run the checks below; record only what you have verified
(Rules 0.1 and 0.5 apply with full force).

1. **Hypothesis reconciliation.** List every condition the proof invokes; diff against the
   stated hypotheses. Flag every "≫", "sufficiently large" and "regularity conditions" left
   undefined. Check whether two assumptions jointly force a parameter to be bounded (which makes
   stated dependence on it in the rate illusory).
2. **Substituted-bound check.** Re-derive each substituted norm/probability bound from the cited
   condition. The tell is the authors' own competing usage of the same quantity elsewhere.
3. **Units/dimension check.** A probability bound cannot depend on a dimensioned quantity; a
   pseudo-inverse applied to a Gaussian vector gives N(0, V†), not N(0, I) — standardisation
   needs V^(−1/2).
4. **Algebraic identity check.** Substitute symbols into displayed factorisations and
   expansions; look for sign errors and spurious factors.
5. **Limit/quantile check.** Evaluate limiting statements literally (Φ(·) → 1 − α/2 vs α/2;
   sup over a fixed-radius ball that is +∞; definitions missing "under the local alternatives").
   Check probability statements are defined at every boundary case the text discusses.
6. **Decoration audit.** Hats, stars, indices, dimensions: γ vs γ̂, k vs K vs K+1, matrix sizes
   vs vector lengths, multinomial totals.
7. **Existence-of-the-object check.** Does the object the theorem is about exist for the
   estimator actually used (e.g. a transport map requires an atomless distribution, but a forest
   classifier outputs finitely many values)? Does the criterion have a minimiser (joint MLE with
   free per-row latent scores and binary outcomes is typically unbounded below without a
   ridge)? Does the *implemented* statistic equal the one the theorem covers?
8. **Specification vs approximation.** If validity rests on a parametric family the paper
   elsewhere calls an *approximation*, the null estimand is not zero and √n·bias → ∞ for fixed
   dimension; state the exact-membership hypothesis the theorem silently needs.
9. **Concavity/uniqueness.** "Concave in γ for fixed λ" does not imply a unique joint maximiser;
   profile out the nuisance and check the profile.
10. **Tangent-space support.** Efficiency of a plug-in using a declared region S holds only if
    the score's support equals S.
11. **Upper-bound vs improvement.** "A improves over B" proved by comparing two upper bounds is
    a Gap; it needs a lower bound for B or a direct comparison. Look for a counterexample cell in
    the paper's own simulation tables.
12. **Monotone descent.** "The objective is non-increasing by construction" holds for exact
    block minimisers and MM steps, not for gradient steps with unspecified step size; verify
    numerically if in doubt.
13. **Check the imports.** Paraphrased external theorems are load-bearing: compare against the
    exact numbered statement in the cited source (norm used, constants, conditions).

---

## 6. Severity and classification

- Consistency / numerical findings (Modules A–C): **Critical** (changes a reported result or the
  truth of a stated theorem), **Major** (must be fixed for the paper to be correct or
  reproducible), **Minor** (presentation, captioning, cross-reference).
- Proof findings (Module D): **Error** (false as written), **Gap** (hypothesis missing or step
  asserted without argument), **Defect** (typographic). Authors act on these very differently.
- **Verified as correct / consistent** — a mandatory category, roughly a third of the report,
  specific enough to prove the check was done ("all 27 figure captions match Table S7 exactly";
  "all 64 shared cells of Table 1 and Table S6 agree"; "identity (S.12) verified by
  substitution"; "Theorem 2's rate reproduced numerically at n = 500, 2000"). It tells the authors
  where not to spend effort, shows the audit was systematic, and makes the critical findings
  harder to wave away. Also list passes that found nothing.

---

## 7. Report

Produce one Word document, `<ID>_QC_report.docx` (split into `<ID>_consistency_audit.docx` and
`<ID>_proof_check.docx` if the user prefers), saved beside the manuscript.

Structure:
1. Title block — manuscript ID, title, files audited (with page counts), date, "QC audit".
2. Count box — counts by severity (Critical/Major/Minor; Error/Gap/Defect) and number of
   verified-consistent checks.
3. Lettered sections, one per module run, each a table with columns
   **# | Severity/Kind | Location (main/supp, page, object) | Finding | Suggested fix**.
   Put side-by-side comparison tables (Pass A5, B1) directly under the finding they support.
4. Verified as correct / consistent.
5. Priority for the revision — the 3–6 items the authors should fix first, **derived from this
   paper's findings only** (never carried over from a previous report).
6. Methods note — one paragraph listing the passes run and tools used.

Minimal generator (python-docx; `pip install python-docx --break-system-packages` if missing):
```python
from docx import Document
from docx.shared import Pt
def build(path, meta, counts, sections, verified, priority):
    d = Document(); st = d.styles['Normal']; st.font.name = 'Calibri'; st.font.size = Pt(10.5)
    d.add_heading(f"QC audit — {meta['id']}", 0); d.add_paragraph(meta['title'])
    d.add_paragraph(meta['files'])
    t = d.add_table(rows=1, cols=len(counts)); t.style = 'Light Grid Accent 1'
    for c, (k, v) in zip(t.rows[0].cells, counts.items()): c.text = f"{k}\n{v}"
    for letter, (name, rows) in zip("ABCDEFG", sections):
        d.add_heading(f"{letter}. {name}", 1)
        tb = d.add_table(rows=1, cols=5); tb.style = 'Light Grid Accent 1'
        for c, h in zip(tb.rows[0].cells, ["#", "Severity", "Location", "Finding", "Suggested fix"]): c.text = h
        for r in rows:
            cells = tb.add_row().cells
            for c, v in zip(cells, r): c.text = str(v)
    d.add_heading("Verified as correct / consistent", 1)
    for v in verified: d.add_paragraph(v, style='List Bullet')
    d.add_heading("Priority for the revision", 1)
    for p in priority: d.add_paragraph(p, style='List Number')
    d.save(path)
```

---

## 8. Final verification before delivering

- [ ] Every equation-level finding was checked on a **rendered page**, not extracted text.
- [ ] Every absence claim was **grepped** in both files.
- [ ] Every numerical finding was **recomputed by code**, and the code's output matches the text.
- [ ] Differences between tables were checked for "different experiment" explanations.
- [ ] Page numbers cited match the printed (not PDF) numbering, or say which is used.
- [ ] Severity counts in the count box equal the number of rows in the tables.
- [ ] Priority list comes from this manuscript's findings.
- [ ] Open the .docx (or convert to PDF and view a page) to confirm it renders.

When delivering, give the severity counts and the two or three findings that matter most in a
few sentences; the report carries the rest.
