# Plan: nmaci → pip-installable package + nma-actions repo

## Goal

Standardize CI/CD across all NMA course repos by:
1. Converting `nmaci` from a collection of downloaded scripts into a proper pip-installable Python package
2. Creating a new `nma-actions` repo with reusable GitHub Actions that all course repos reference
3. Migrating course repos to use `nma-actions` instead of inline workflow logic

This is a 3-phase project. Phase 1 (nmaci package) must be completed before the others.

---

## Phase 1: nmaci as a pip package ✅ COMPLETE

### What was done
- Added `pyproject.toml` with hatchling build backend and all dependencies
- Moved all scripts from `nmaci/scripts/` → `nmaci/src/nmaci/`
- Refactored 5 scripts that used `sys.argv` at module level to use `main(arglist=None)`
- Added `src/nmaci/cli.py` — single entry point dispatching all subcommands
- Replaced `scripts/` with thin shims for backward compat with existing course repo CI
- Updated tests to use `from nmaci.* import ...` and `sys.executable -m nmaci`
- Moved test fixture notebooks to `tests/tutorials/`, updated kernelspec to `python3`
- Replaced Makefile with direct pytest invocation in CI
- Switched CI to use `uv` + `pip install -e ".[dev]"`
- All 16 tests pass

### New repo structure

```
nmaci/
  pyproject.toml
  src/
    nmaci/
      __init__.py
      process_notebooks.py
      verify_exercises.py
      select_notebooks.py
      make_pr_comment.py
      generate_tutorial_readmes.py
      find_unreferenced_content.py
      generate_book.py
      generate_book_dl.py
      generate_book_precourse.py
      extract_links.py
      lint_tutorial.py
      parse_html_for_errors.py
      cli.py
      chatify/
        __init__.py
        process_notebooks.py
        install_and_load_chatify.py
        install_davos.py
  tests/
    test_process_notebooks.py      (refactor existing + add more)
    test_verify_exercises.py
    test_select_notebooks.py
    test_extract_links.py
    tutorials/                     (test fixture notebooks, moved from nmaci/tutorials/)
  scripts/                         (keep temporarily during migration, then remove)
```

### pyproject.toml

```toml
[build-system]
requires = ["setuptools>=61", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "nmaci"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = [
    "nbformat",
    "nbconvert",
    "notebook",
    "pillow",
    "flake8",
    "fuzzywuzzy[speedup]",
    "pyyaml",
    "beautifulsoup4",
    "decorator==5.0.9",
    "Jinja2==3.0.0",
    "jupyter-client",
]

[project.scripts]
nmaci = "nmaci.cli:main"
```

### CLI design (cli.py)

Single entry point dispatching subcommands:

```
nmaci process-notebooks  <files> [--execute | --check-execution] [--check-only] ...
nmaci verify-exercises   <files> [--c <commit_msg>]
nmaci select-notebooks   <files>
nmaci make-pr-comment    <files> [--branch <branch>] [--o <outfile>]
nmaci generate-readmes
nmaci find-unreferenced
nmaci generate-book      [student|instructor]
nmaci generate-book-dl   [student|instructor]
nmaci extract-links      [--noyoutube]
nmaci lint-tutorial      <files>
nmaci parse-html         [student|instructor]
```

Each subcommand calls the existing `main(arglist)` function from the corresponding module — minimal refactoring of script internals needed.

### Migration steps

1. Add `pyproject.toml`
2. Create `src/nmaci/` directory, move scripts into it
3. Add `__init__.py` files
4. Write `cli.py` with subcommand dispatch
5. Refactor tests to use imports instead of subprocess (where possible — keep subprocess tests for CLI validation)
6. Add tests for untested scripts (`verify_exercises`, `select_notebooks`, `extract_links`)
7. Verify `pip install -e .` works locally
8. Verify `pip install git+https://github.com/neuromatch/nmaci@main` works

### Notes on existing code
- All scripts already have `main(arglist)` functions — CLI wiring is straightforward
- `process_notebooks.py` reads env vars (`ORG`, `NMA_REPO`, `NMA_MAIN_BRANCH`) — no change needed
- `chatify/` is experimental and loosely coupled — package it as `nmaci.chatify` subpackage
- Three `generate_book` variants exist (generic, DL, precourse) — keep all three for now
- Existing subprocess-based tests in `test_process_notebooks.py` can be kept as integration tests;
  new unit tests should import directly from `nmaci.*`

---

## Phase 2: nma-actions repo

### Goal

New repo `neuromatch/nma-actions` containing reusable GitHub Actions and workflows.
Course repos reference these instead of defining their own.

### Repo structure

```
nma-actions/
  README.md
  setup-python-env/
    action.yml           (composite action)
  setup-ci-tools/
    action.yml           (composite action — installs nmaci via pip)
  setup-rendering-deps/
    action.yml           (composite action — fonts, ffmpeg, graphviz)
  .github/
    workflows/
      notebook-pr.yml    (reusable workflow — 3-job parallel architecture)
      publish-book.yml   (reusable workflow)
      tutorial-readme-pr.yml  (reusable workflow)
      download-slides.yml     (reusable workflow)
```

### setup-ci-tools (simplified by pip package)

```yaml
- name: Install nmaci
  shell: bash
  run: |
    BRANCH=$(python3 -c "
    import os, re
    msg = os.environ.get('COMMIT_MESSAGE', '')
    m = re.search(r'nmaci:([\w-]+)', msg)
    print(m.group(1) if m else '${{ inputs.branch }}')
    ")
    pip install git+https://github.com/neuromatch/nmaci@$BRANCH
```

No tarball download, no extraction, no `.gitignore` mutation.

### Reusable workflow inputs

`notebook-pr.yml` would accept inputs for course-specific values:

```yaml
on:
  workflow_call:
    inputs:
      nma-repo:
        required: true
        type: string
      nma-main-branch:
        required: false
        default: main
        type: string
      org:
        required: false
        default: neuromatch
        type: string
```

Course repos call it with:
```yaml
uses: neuromatch/nma-actions/.github/workflows/notebook-pr.yml@main
with:
  nma-repo: NeuroAI_Course
  org: neuromatch
```

### Architecture: parallel matrix (based on neuro-ai-course-content)

The `notebook-pr` reusable workflow uses the most modern pattern (from `neuro-ai-course-content`):
- `setup` job: detect changed notebooks, build matrix
- `process` job: parallel per-notebook execution
- `finalize` job: aggregate artifacts, verify, commit, push

This replaces the older monolithic approach in `course-content` and `course-content-dl`.

---

## Phase 3: Migrate course repos

### Order of migration
1. `neuro-ai-course-content` — already closest to the target architecture
2. `course-content` — older but well-understood
3. `course-content-dl` — similar to course-content
4. `climate-course-content` — separate phase (custom runner, specialized deps)

### Per-repo changes
- Replace local `.github/actions/` with references to `neuromatch/nma-actions/...@main`
- Replace inline `notebook-pr.yaml` job logic with `uses: neuromatch/nma-actions/.github/workflows/notebook-pr.yml@main`
- Pass course-specific values as `with:` inputs
- Remove `wget nmaci tarball` patterns — now just `pip install nmaci`

### climate-course-content (future phase)
- Add `setup-climate-env` action to `nma-actions` for ESMF/cartopy/HDF5/GEOS deps
- Shared actions (`setup-ci-tools`, `setup-rendering-deps`) can be adopted immediately
- Custom runner stays in the course repo workflow — that's infrastructure, not action logic

---

## Decision log

- **Language for actions:** Composite YAML (not TypeScript) — logic goes in Python (nmaci package), testable with pytest. No TS learning curve needed.
- **CLI style:** Single `nmaci` entry point with subcommands (not one entry point per script)
- **Chatify:** Package as `nmaci.chatify` subpackage — experimental, keep loosely coupled
- **Three generate_book variants:** Keep all three for now, consolidate later if appropriate
- **climate-course-content:** Deferred to a separate phase — too specialized to include in initial standardization
