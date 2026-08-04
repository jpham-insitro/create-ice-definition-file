# ICE Definition File Generator (Claude skill)

Automates preparation of the two files required for **Synthego ICE** (Inference of CRISPR Edits) batch upload at [ice.synthego.com](https://ice.synthego.com):

1. a **definition spreadsheet** (`Label | Control File | Experiment File | Guide Sequence | Donor Sequence`) using the official ICE template headers, and
2. a **flat zip** of matched `.ab1` Sanger traces.

It discovers `.ab1` files recursively, classifies each as control/experiment from the filename, pairs them by gene, auto-fills multiplex guide sequences, validates against Synthego's rules, and exports a ready-to-upload `definition.xlsx`, `ab1.zip`, and a provenance report.

## Install

This is a [Claude](https://claude.com) skill. Install it via **Settings → Capabilities**, or drop the `.skill` bundle into Cowork's *Save skill* prompt.

## Usage

```
/ice-definition-file
```

The skill then asks for a session name and one or more folders to search, discovers and pairs traces, collects guide sequences, and writes the outputs.

## Contents

- `SKILL.md` — the skill definition (frontmatter + instructions).

## License

Internal use.
