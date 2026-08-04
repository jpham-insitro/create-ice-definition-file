---
name: ice-definition-file
description: >-
  Automates preparation of the two files required for ICE (Inference of CRISPR
  Edits) batch upload at ice.synthego.com: a definition spreadsheet with sample
  labels, guide sequences, and file pairings, and a flat zip of matched .ab1
  Sanger files. Use whenever the user wants to build ICE batch-upload files,
  pair control/experiment .ab1 traces, generate an ICE definition spreadsheet,
  or prepare Sanger files for Synthego ICE. Triggers on: "ICE definition file",
  "ICE batch upload", "prepare ab1 files for ICE", "pair my Sanger files",
  "make the ICE spreadsheet", or /ice-definition-file. Uses three parallel
  sub-agents for file discovery, gene-based pair resolution, and validation
  coordinated via a shared manifest.json, then exports a ready-to-upload
  definition.xlsx (official Synthego template headers) and ab1.zip with a
  provenance report of any unresolved pairs.
---

# ICE Definition File Generator

Automates preparation of the two files required for ICE (Inference of CRISPR Edits) batch upload at ice.synthego.com: a definition spreadsheet with sample labels, guide sequences, and file pairings, and a flat zip of matched .ab1 Sanger files. Uses three parallel sub-agents for file discovery, pair resolution, and validation — coordinated via a shared manifest.json. Exports a ready-to-upload definition.xlsx and ab1.zip with a provenance report of any unresolved pairs.

## What it does

1. Asks for a **Session Name** and one or more **search root paths**
2. Spawns three parallel sub-agents via the Agent tool:
   - **Agent 1** — recursively discovers all `.ab1` files under all roots, classifies each as control/experiment by locating a role token (`wt`/`ko`/etc.) anywhere in the filename, extracts the gene, writes `manifest.json`
   - **Agent 2** — resolves control/experiment pairs by **gene**, flags ambiguous/orphan files, optionally consults Benchling for gene targets
   - **Agent 3** — validates every experiment maps to exactly one control, flags orphans
3. Merges results, collects guide sequences, validates inputs
4. Exports `[Session Name]-definition.xlsx` (official Synthego template headers), `[Session Name]-ab1.zip`, and `[Session Name]-report.md`

## Usage

```
/ice-definition-file
```

---

## Output format contract (IMPORTANT — do not deviate)

The Synthego ICE batch-upload definition spreadsheet **must** use these exact column headers, in this exact order (verbatim, matching `template_ice_definitions.xlsx`):

```
Label | Control File | Experiment File | Guide Sequence | Donor Sequence
```

Rules from Synthego's own instructions:
- **Label** — sample name, 255 character limit.
- **Control File** — filename of the unedited `.ab1` (must also be in the zip).
- **Experiment File** — filename of the edited `.ab1` (must also be in the zip).
- **Guide Sequence** — 10–50 nt, **no PAM**. For multiplex edits, up to 3 guides in a comma-separated list.
- **Donor Sequence** — optional (HDR only), 15–300 nt including homology arms.
- Every filename referenced in the sheet must be present in the flat `.zip`.

Do **not** rename or reorder these columns.

---

## Skill instructions

You are helping the user generate ICE batch-upload files. Follow the steps below in order.

---

### STEP 1 — Session name

Ask:
> What would you like to name this session? This will be used to name your output files (e.g. `MyExperiment-definition.xlsx`).

Wait for the answer. Store as `SESSION_NAME`.

---

### STEP 2 — Search roots and output folder

Ask:
> What is the full path (or paths) to search for `.ab1` files? I'll search recursively — you can provide one folder or several separated by commas.
> Where should output files be saved? (Press Enter to use the first search root.)

Store root paths as `SEARCH_ROOTS` (list). Store output path as `OUTPUT_FOLDER` (default: first root).

Set `MANIFEST_PATH = OUTPUT_FOLDER + "/manifest.json"`.

If any search root is nested inside another, keep only the top-level root to avoid discovering the same files twice.

---

### STEP 3 — Parallel sub-agent execution

Spawn all three agents simultaneously using the Agent tool. Pass `SEARCH_ROOTS`, `MANIFEST_PATH`, `SESSION_NAME`, and `OUTPUT_FOLDER` in each prompt. Do not wait for one to finish before launching the others — launch all three in a single message.

---

#### Agent 1 — Recursive .ab1 Discovery + Classification

**Prompt:**
> You are Agent 1 in an ICE definition file workflow. Your job is to recursively discover every `.ab1` file under the given search roots, classify each as control or experiment, extract its gene, and write a manifest.
>
> Search roots: `__SEARCH_ROOTS__`
> Manifest output path: `__MANIFEST_PATH__`
>
> Run this Python script via Bash:
> ```python
> import json
> from pathlib import Path
> from collections import Counter
>
> SEARCH_ROOTS  = __SEARCH_ROOTS__
> MANIFEST_PATH = Path("__MANIFEST_PATH__")
>
> # Role tokens may appear ANYWHERE in the underscore-delimited filename,
> # e.g. "B1_ADAT2_wt_amp_fwd-seq_f" or "C2_CYCS_ko_run1_amp_fwd-seq_f_R".
> CONTROL_TOKENS    = {"control", "ctrl", "unedited", "wt", "wildtype"}
> EXPERIMENT_TOKENS = {"experiment", "exp", "edited", "sample", "treated", "ko", "ki"}
>
> def classify(stem):
>     """Locate the control/experiment token in the filename; take the gene as
>     the token immediately preceding it. Uses the LAST role token found, so
>     descriptor prefixes like 'low_ko_RYK_control' resolve to the true role
>     (control), not the earlier 'ko'."""
>     tokens = stem.split("_")
>     low    = [t.lower() for t in tokens]
>     role, idx = "unclassified", None
>     for i, t in enumerate(low):          # scan all tokens; keep the LAST match
>         if t in CONTROL_TOKENS:
>             role, idx = "control", i
>         elif t in EXPERIMENT_TOKENS:
>             role, idx = "experiment", i
>     if idx is None:
>         gene = ""
>     elif idx >= 1:
>         gene = tokens[idx - 1]          # B1_ADAT2_wt_... -> ADAT2
>     elif len(tokens) > 1:
>         gene = tokens[idx + 1]          # wt_ADAT2_...     -> ADAT2
>     else:
>         gene = stem
>     return role, gene
>
> records = []
> for root in SEARCH_ROOTS:
>     for f in sorted(Path(root).rglob("*.ab1")):
>         role, gene = classify(f.stem)
>         records.append({
>             "filename":  f.name,
>             "full_path": str(f.resolve()),
>             "root":      str(root),
>             "rel_path":  str(f.relative_to(root)),
>             "stem":      f.stem,
>             "gene":      gene,
>             "role":      role,
>         })
>
> manifest = {
>     "session":      "__SESSION_NAME__",
>     "search_roots": [str(r) for r in SEARCH_ROOTS],
>     "total_ab1":    len(records),
>     "files":        records,
>     "pairs":        [],   # filled by Agent 2
>     "orphans":      [],   # filled by Agent 3
>     "validation":   {},   # filled by Agent 3
> }
> MANIFEST_PATH.parent.mkdir(parents=True, exist_ok=True)
> MANIFEST_PATH.write_text(json.dumps(manifest, indent=2))
> print(f"Agent 1 complete: {len(records)} .ab1 files written to {MANIFEST_PATH}")
> print("Breakdown:", dict(Counter(r['role'] for r in records)))
> for r in records:
>     print(f"  [{r['role']:12s}] {r['filename']}  (gene: {r['gene']})")
> ```
>
> Report the total file count, the classified breakdown (control / experiment / unclassified), and any roots that returned zero files. Return the full printed output.

---

#### Agent 2 — Gene-Based Pair Resolution + Benchling Lookup

**Prompt:**
> You are Agent 2 in an ICE definition file workflow. Your job is to read the manifest written by Agent 1, resolve control/experiment pairs by **gene**, and (best-effort) look up gene targets in Benchling.
>
> Manifest path: `__MANIFEST_PATH__`
>
> **Wait until the manifest file exists before starting** — poll with `Path(MANIFEST_PATH).exists()` in a short loop.
>
> Pairing rule: each experiment is matched to the control that shares its gene. Exactly one control per gene → a pair. No control for that gene → orphan experiment. More than one control for the same gene → ambiguous (needs manual resolution). Multiple experiments for the same gene are allowed and each becomes its own pair against the shared control.
>
> Run this Python script via Bash:
> ```python
> import json, time, re
> from pathlib import Path
> from collections import defaultdict
>
> MANIFEST_PATH = Path("__MANIFEST_PATH__")
>
> # Wait for Agent 1 to finish writing the manifest (max 60s)
> for _ in range(60):
>     if MANIFEST_PATH.exists():
>         break
>     time.sleep(1)
> else:
>     raise RuntimeError("Manifest not found after 60s — Agent 1 may have failed.")
>
> manifest = json.loads(MANIFEST_PATH.read_text())
> files = manifest["files"]
>
> controls     = [f for f in files if f["role"] == "control"]
> experiments  = [f for f in files if f["role"] == "experiment"]
> unclassified = [f for f in files if f["role"] == "unclassified"]
>
> ctrl_by_gene = defaultdict(list)
> for c in controls:
>     ctrl_by_gene[c["gene"].upper()].append(c)
>
> pairs, ambiguous = [], []
> matched_ctrl = set()
> ambiguous_genes = set()
> for exp in experiments:
>     g = exp["gene"].upper()
>     cands = ctrl_by_gene.get(g, [])
>     if len(cands) == 1:
>         pairs.append({
>             "pair_id":     len(pairs) + 1,
>             "gene":        exp["gene"],
>             "control":     cands[0],
>             "experiment":  exp,
>             "match_type":  "gene",
>             "gene_symbol": "",
>         })
>         matched_ctrl.add(cands[0]["filename"])
>     elif len(cands) == 0:
>         pass  # orphan experiment (collected below)
>     else:
>         ambiguous_genes.add(g)
>         ambiguous.append({
>             "gene":            exp["gene"],
>             "ctrl_candidates": [c["filename"] for c in cands],
>             "exp_candidates":  [exp["filename"]],
>             "reason":          "multiple_controls_same_gene",
>         })
>
> matched_exp = {p["experiment"]["filename"] for p in pairs}
> unmatched_exp  = [e for e in experiments
>                   if e["filename"] not in matched_exp
>                   and e["gene"].upper() not in ambiguous_genes]
> unmatched_ctrl = [c for c in controls if c["filename"] not in matched_ctrl]
>
> # ── Optional Benchling gene-symbol enrichment (best-effort) ──────────────────
> ins_re = re.compile(r'INS-\d+-\d+', re.IGNORECASE)
> all_ins_ids = list({m for f in files for m in ins_re.findall(f["stem"])})
> gene_map = {}
> if all_ins_ids:
>     try:
>         from insitro_core.labdata.benchling_utils.client import get_sdk_client
>         client = get_sdk_client(benchling_env='prod', secret_name='BenchlingAPI/prod/lab_read')
>         for i in range(0, len(all_ins_ids), 100):
>             chunk = all_ins_ids[i:i+100]
>             for page in client.molecules.list(entity_registry_ids_any_of=chunk, page_size=100):
>                 for mol in page:
>                     if mol.entity_registry_id:
>                         fld = mol.fields.additional_properties if mol.fields else {}
>                         tgt = fld.get('Target')
>                         gene_map[mol.entity_registry_id] = tgt.text_value if tgt and tgt.text_value else ''
>     except Exception as e:
>         print(f"BENCHLING_WARN: {e} — gene_symbol falls back to filename gene token")
>
> for p in pairs:
>     ins = ins_re.findall(p["experiment"]["stem"])
>     p["gene_symbol"] = (gene_map.get(ins[0], "") if ins else "") or p["gene"]
>
> manifest["pairs"]           = pairs
> manifest["unmatched_ctrl"]  = unmatched_ctrl
> manifest["unmatched_exp"]   = unmatched_exp
> manifest["ambiguous"]       = ambiguous
> manifest["unclassified"]    = unclassified
> manifest["benchling_resolved"] = len([p for p in pairs if p["gene_symbol"]])
> MANIFEST_PATH.write_text(json.dumps(manifest, indent=2))
>
> print(f"Agent 2 complete: {len(pairs)} gene-based pairs resolved")
> print(f"  Unmatched controls:    {len(unmatched_ctrl)}")
> print(f"  Unmatched experiments: {len(unmatched_exp)}")
> print(f"  Ambiguous:             {len(ambiguous)}")
> print(f"  Benchling resolved:    {manifest['benchling_resolved']}/{len(pairs)}")
> ```
>
> Report the pair resolution summary. Return the full printed output.

---

#### Agent 3 — Validation + Orphan Detection

**Prompt:**
> You are Agent 3 in an ICE definition file workflow. Your job is to read the manifest after Agent 2 has resolved pairs, validate that every experiment maps to exactly one control, and flag orphans.
>
> Manifest path: `__MANIFEST_PATH__`
>
> **Wait until the manifest contains a non-empty "pairs" key** — poll with a short loop.
>
> Run this Python script via Bash:
> ```python
> import json, time
> from pathlib import Path
>
> MANIFEST_PATH = Path("__MANIFEST_PATH__")
>
> # Wait for Agent 2 to finish writing pairs (max 120s)
> for _ in range(120):
>     data = json.loads(MANIFEST_PATH.read_text())
>     if data.get("pairs") or data.get("unmatched_ctrl") is not None:
>         break
>     time.sleep(1)
> else:
>     raise RuntimeError("Pairs not found after 120s — Agent 2 may have failed.")
>
> pairs          = data["pairs"]
> unmatched_ctrl = data.get("unmatched_ctrl", [])
> unmatched_exp  = data.get("unmatched_exp", [])
> ambiguous      = data.get("ambiguous", [])
> unclassified   = data.get("unclassified", [])
>
> errors, warnings, orphans = [], [], []
>
> # Rule 1: every experiment must map to exactly one control
> exp_to_ctrl = {}
> for pair in pairs:
>     exp_name = pair["experiment"]["filename"]
>     if exp_name in exp_to_ctrl:
>         errors.append(f"DUPLICATE: {exp_name} maps to multiple controls: "
>                       f"{exp_to_ctrl[exp_name]} and {pair['control']['filename']}")
>     exp_to_ctrl[exp_name] = pair["control"]["filename"]
>
> # Rule 2: unmatched experiments are orphans
> for f in unmatched_exp:
>     orphans.append({"file": f["filename"], "path": f["full_path"], "reason": "no_control_found"})
>
> # Rule 3: unmatched controls are orphans
> for f in unmatched_ctrl:
>     orphans.append({"file": f["filename"], "path": f["full_path"], "reason": "no_experiment_found"})
>
> # Rule 4: ambiguous genes require user resolution
> for a in ambiguous:
>     warnings.append(f"AMBIGUOUS: gene {a['gene']} — "
>                     f"ctrl candidates: {a['ctrl_candidates']}, "
>                     f"exp candidates: {a['exp_candidates']}")
>
> # Rule 5: unclassified files are warnings
> for f in unclassified:
>     warnings.append(f"UNCLASSIFIED: {f['filename']} (no control/experiment token found)")
>
> validation = {
>     "status":         "PASS" if not errors else "FAIL",
>     "total_pairs":    len(pairs),
>     "errors":         errors,
>     "warnings":       warnings,
>     "orphans":        orphans,
>     "ready_to_export": len(errors) == 0,
> }
>
> data["validation"] = validation
> data["orphans"]    = orphans
> MANIFEST_PATH.write_text(json.dumps(data, indent=2))
>
> print(f"Agent 3 complete — validation: {validation['status']}")
> print(f"  Pairs validated: {len(pairs)}")
> print(f"  Errors:          {len(errors)}")
> print(f"  Warnings:        {len(warnings)}")
> print(f"  Orphans:         {len(orphans)}")
> if errors:
>     print("\nERRORS (block export):")
>     for e in errors: print(f"  x {e}")
> if warnings:
>     print("\nWARNINGS (require review):")
>     for w in warnings: print(f"  ! {w}")
> if orphans:
>     print("\nORPHANS:")
>     for o in orphans: print(f"  -> {o['file']} ({o['reason']})")
> ```
>
> Report the validation status. Return the full printed output.

---

### STEP 4 — Merge results and surface to user

After all three agents complete, read `manifest.json` and present a consolidated summary:

```
Discovery:   N .ab1 files found across M search roots
Pairs:       X matched (gene-based)
Orphans:     N files with no matching pair
Ambiguous:   N genes requiring manual resolution
Validation:  PASS / FAIL
```

If validation status is FAIL or orphans/ambiguous exist, present them in a table and ask:
> Some files couldn't be paired automatically. Would you like to assign any of these manually before I generate the definition file? For each, tell me which control goes with which experiment.

Collect manual pair assignments as `MANUAL_PAIRS = [{"control": "...", "experiment": "..."}]`. Add them to the manifest pairs list with `match_type: "manual"`. Re-run Agent 3 validation on the updated manifest before proceeding.

If validation is PASS (or user chooses to proceed with orphans excluded), advance to Step 5.

---

### STEP 5 — Guide sequences

Ask:
> Do all your samples use the **same guide RNA sequence**, or does each sample have a **different guide**? You can also point me to an existing file (text, CSV, Excel) with the sequences and I'll read them.

**If the user points to a file:** read it via Bash, match entries to pairs by **gene name** (case-insensitive). Many guide sheets have `gene_name` plus `guide_#1/#2/#3` columns — join the (up to 3) non-empty guides into a comma-separated multiplex string. Present the parsed mapping for confirmation before proceeding.

**If same guide for all:** ask for the sequence (10–50 nt, DNA or RNA, up to 3 comma-separated for multiplex). Store as `DEFAULT_GUIDE_SEQUENCE`.

**If different per pair:** present all pairs in a single table and ask the user to fill in Guide sequence (required), Label (optional), and Donor sequence (optional, HDR only) for each.

Validate all sequences: each guide 10–50 nt, only A/C/G/T/U characters, **no PAM**, up to 3 for multiplex. Donor (if any) 15–300 nt. If any fail, report the exact error and ask for a correction before continuing.

Build `PAIR_METADATA = {control_filename: {guide, label, donor}}`. Because pairing is gene-based, guides keyed by the shared control apply to every experiment of that gene.

---

### STEP 6 — Generate definition file, archive, and report

Once guide sequences are confirmed and validation is PASS, run the following via Bash:

```python
import json, zipfile
from pathlib import Path
from datetime import datetime, timezone
from collections import Counter
import pandas as pd

MANIFEST_PATH = Path("__MANIFEST_PATH__")
OUTPUT_FOLDER = Path("__OUTPUT_FOLDER__")
SESSION_NAME  = "__SESSION_NAME__"
PAIR_METADATA = __PAIR_METADATA__          # {control_filename: {guide,label,donor}}
DEFAULT_GUIDE = "__DEFAULT_GUIDE_SEQUENCE__"

manifest = json.loads(MANIFEST_PATH.read_text())
pairs    = manifest["pairs"]

# ── Pass 1: base label + guide/donor per pair ───────────────────────────────
for p in pairs:
    meta = PAIR_METADATA.get(p["control"]["filename"], {})
    p["_guide"] = meta.get("guide", DEFAULT_GUIDE)
    p["_donor"] = meta.get("donor", "") or ""
    p["_base"]  = (meta.get("label") or p.get("gene_symbol") or p.get("gene")
                   or Path(p["experiment"]["filename"]).stem)

# ── Pass 2: make labels unique (multiple experiments can share one gene) ─────
base_counts = Counter(p["_base"] for p in pairs)
used = set()
rows = []
for p in pairs:
    base = p["_base"]
    if base_counts[base] == 1:
        label = base
    else:
        tok = p["experiment"]["stem"].split("_")[-1]   # distinguishing token
        label = f"{base}_{tok}"
        i = 2
        while label in used:
            label = f"{base}_{tok}_{i}"; i += 1
    used.add(label)
    rows.append({
        "Label":           label[:255],
        "Control File":    p["control"]["filename"],
        "Experiment File": p["experiment"]["filename"],
        "Guide Sequence":  p["_guide"],
        "Donor Sequence":  p["_donor"],
    })

# Official Synthego ICE template headers & order — DO NOT rename or reorder.
COLS = ["Label", "Control File", "Experiment File", "Guide Sequence", "Donor Sequence"]
df = pd.DataFrame(rows)[COLS].sort_values("Label").reset_index(drop=True)

xlsx_path = OUTPUT_FOLDER / f"{SESSION_NAME}-definition.xlsx"
df.to_excel(xlsx_path, index=False)
print(f"Saved definition: {xlsx_path}")

# ── Write ab1 zip (flat, deduped by filename) ───────────────────────────────
zip_path = OUTPUT_FOLDER / f"{SESSION_NAME}-ab1.zip"
written = set()
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zf:
    for p in pairs:
        for role in ("control", "experiment"):
            src = Path(p[role]["full_path"])
            if src.exists() and src.name not in written:
                zf.write(src, src.name)
                written.add(src.name)
print(f"Saved archive:    {zip_path} ({len(written)} unique files)")

# ── Write provenance report ─────────────────────────────────────────────────
validation = manifest.get("validation", {})
orphans    = manifest.get("orphans", [])
ambiguous  = manifest.get("ambiguous", [])
now        = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M UTC")

report_lines = [
    "# ICE Definition File — Session Report",
    f"**Session:** {SESSION_NAME}  ",
    f"**Generated:** {now}",
    "",
    "## Summary",
    "",
    "| Metric | Value |",
    "|---|---|",
    f"| Total .ab1 files found | {manifest['total_ab1']} |",
    f"| Pairs in definition file | {len(pairs)} |",
    f"| Gene matches | {sum(1 for p in pairs if p['match_type']=='gene')} |",
    f"| Manual matches | {sum(1 for p in pairs if p['match_type']=='manual')} |",
    f"| Orphaned files | {len(orphans)} |",
    f"| Ambiguous (unresolved) | {len(ambiguous)} |",
    f"| Validation | {validation.get('status','N/A')} |",
    "",
    "## Pairs Included",
    "",
    df.to_markdown(index=False),
]

if orphans:
    report_lines += ["", "## Orphaned Files (excluded from definition)", "| File | Reason |", "|---|---|"]
    for o in orphans:
        report_lines.append(f"| `{o['file']}` | {o['reason']} |")

if ambiguous:
    report_lines += ["", "## Ambiguous Pairs (require manual review)", "| Gene | Control candidates | Experiment candidates |", "|---|---|---|"]
    for a in ambiguous:
        report_lines.append(f"| `{a['gene']}` | {', '.join(a['ctrl_candidates'])} | {', '.join(a['exp_candidates'])} |")

if validation.get("warnings"):
    report_lines += ["", "## Warnings"]
    for w in validation["warnings"]:
        report_lines.append(f"- {w}")

report_path = OUTPUT_FOLDER / f"{SESSION_NAME}-report.md"
report_path.write_text("\n".join(report_lines))
print(f"Saved report:     {report_path}")
```

---

### STEP 7 — Report results to user

> **Done!** Your ICE files are ready:
>
> - **Definition file:** `[SESSION_NAME]-definition.xlsx`
> - **Archive:** `[SESSION_NAME]-ab1.zip`
> - **Report:** `[SESSION_NAME]-report.md`
>
> Upload both files at ice.synthego.com — drop the `.zip` into the sequences box and the `.xlsx` into the definitions box.

Show the pairs table from the report. If any orphans or ambiguous pairs were excluded, surface them clearly so the user knows what was left out.

---

## Manifest schema

```json
{
  "session":             "string",
  "search_roots":        ["path1", "path2"],
  "total_ab1":           42,
  "files":               [{"filename","full_path","root","rel_path","stem","gene","role"}],
  "pairs":               [{"pair_id","gene","control":{},"experiment":{},"match_type","gene_symbol"}],
  "unmatched_ctrl":      [],
  "unmatched_exp":       [],
  "ambiguous":           [{"gene","ctrl_candidates","exp_candidates","reason"}],
  "unclassified":        [],
  "orphans":             [{"file","path","reason"}],
  "benchling_resolved":  12,
  "validation":          {"status","total_pairs","errors","warnings","orphans","ready_to_export"}
}
```

---

## Reference files
- Definition template (headers): `ice_example&instruction_files/ice_batch_upload_files/template_ice_definitions.xlsx`
- Example definition: `ice_example&instruction_files/ice_batch_upload_files/example_ice_definitions.xlsx`
- Example zip: `ice_example&instruction_files/ice_batch_upload_files/example_ice.zip`
- Synthego batch-upload instructions: `ice_example&instruction_files/ice_batch_upload_files/Instructions.txt`
