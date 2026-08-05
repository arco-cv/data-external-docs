# CLAUDE.md — data-external-docs

> **This repo is a scaffold in progress (DP-7180).** Only `pyproject.toml`, `uv.lock`, a stub
> `README.md`, `.gitignore`, and empty `src/__init__.py` / `tests/__init__.py` exist today.
> There is **no `Dockerfile`, no `.github/`, no CI workflow, and no application code**.
> Items marked `[PLANNED]` are specified in the TRD/backend plan but **not yet built** — if you
> cannot find `src/entrypoint.py` or `src/yaml_reconciler.py`, the repo is not broken.

## Overview

Automatic documentation of BigQuery column descriptions for the `data-platform` team: it reads a
dataset's real schema, downloads the source system's external API docs, generates PT-BR column
descriptions with an LLM, reconciles them into the versioned `schema.yml` of `arco-cv/dbt`, and
opens a Pull Request — **it publishes nothing**.

- **Type:** CLI (subcommand-dispatching entrypoint), packaged as a Docker image and run as an
  **ephemeral Kubernetes pod launched by Airflow** (`GKEStartPodOperator`). **Not a service, not a
  library.** No HTTP routes, no server, no persistent state, no database. Input arrives as env
  vars; output is a GitHub PR; failure is a process exit code surfaced in the Airflow UI.
- **Stack:** Python >=3.11, `uv`, `google-cloud-bigquery`, `anthropic`, `httpx`, `ruamel.yaml`.
- **Scope:** Etapa 1 (propose) only. Etapa 2 (publish to Metabase) lives in `arco-cv/dbt` (`D2`);
  the DAG that launches this pod lives in `arco-cv/airflow` (`D3`). Neither is in this repo.

## Commands

```bash
uv sync                  # install (incl. dev group; uv.lock committed, so reproducible)
uv run pytest            # test — testpaths = ["tests"]; collects nothing today (zero test files)
uv run ruff check .      # lint
uv run ruff format .     # format
uv run mypy              # typecheck — files = ["src", "tests"] configured, no path arg needed
```

There is **no Makefile**, **no `[project.scripts]`**, and no task runner — the above are the
direct `uv` invocations the declared tooling supports.

<!-- TODO: verify dev command — no dev server exists; [PLANNED] local run is `uv run python -m src.entrypoint generate` -->
<!-- TODO: verify build command — no Dockerfile exists; [PLANNED] `docker build -t data-external-docs .` -->

Do **not** document a `--cov` variant: no coverage plugin is declared and no threshold exists.

## Architecture

Single-process CLI entrypoint dispatching subcommands, run as an ephemeral pod. Two surfaces share
one code path: `generate` (opens/updates the PR) and `generate --dry-run` (inspectable stdout, no
external effect at all). `[PLANNED]` modules are **pure data-in/data-out** — only `entrypoint.py`
orchestrates and knows the order, which is what makes execution re-entrant per table (`D8`) and
lets the SSRF defense (`M-01`) be validated in isolation inside `external_doc.py`. The pipeline is
`entrypoint.py` → `bigquery_schema.py` (flatten nested columns to dotted path) → `external_doc.py`
(fetch + SSRF validation) → `descriptions.py` (one LLM call per table, truncate to 1024) →
`yaml_reconciler.py` (merge via `ruamel.yaml`) → `github_pr.py` (deterministic branch + PR).
Downstream and outside this repo: PR merged → `arco-cv/dbt` trigger → dbt run → `persist_docs`
writes BigQuery columns → Etapa 2 mirrors to Metabase.

**Full detail, contracts, exit codes and module breakdown:** [`.claude/docs/architecture.md`](.claude/docs/architecture.md)

## Tech Stack

| Layer | Technology | Version |
| --- | --- | --- |
| Runtime | Python | `>=3.11` (mypy target `3.11`) `[PRESENT]` |
| Package manager | `uv` | lockfile `version = 1`; `[tool.uv] package = false` `[PRESENT]` |
| BigQuery client | `google-cloud-bigquery` | `3.42.3` `[PRESENT]` — metadata reads only, never `SELECT` (`M-06`) |
| LLM SDK | `anthropic` | `0.120.2` `[PRESENT]` — `[PLANNED]` model `claude-opus-5`, structured output via `output_config.format` |
| HTTP client | `httpx` | `0.28.1` `[PRESENT]` — `[PLANNED]` host of the `M-01` SSRF validation |
| YAML library | `ruamel.yaml` | `0.19.1` `[PRESENT]` — deliberate over `pyyaml` (`D10`, `M-05`) |
| Test runner | `pytest` | `9.1.1` `[PRESENT]` dev group; zero test files yet |
| Linter / formatter | `ruff` | `0.16.1` `[PRESENT]` |
| Type checker | `mypy` | `2.3.0` `[PRESENT]` |
| Coverage tool | none | absent by design `[PRESENT]` |
| Container | Docker | `[PLANNED]` — no Dockerfile yet |
| CI platform | GitHub Actions | `[PLANNED]` — no workflow yet |
| Image registry | GCP Artifact Registry | `[PLANNED]` `us-central1-docker.pkg.dev/<GCP_PROJECT_ID>/data-external-docs/data-external-docs` |
| Orchestrator (external) | Airflow 3.1.3 on GKE | `[PLANNED]`/external — lives in `arco-cv/airflow` (`D3`) |

`pydantic` and `jiter` arrive transitively via `anthropic` — do **not** import them as first-class deps.

**Explicitly NOT in this stack** (do not add or assume): no web framework, no database/ORM, no
migrations, no Unleash feature flags (`D11`), no metrics stack, no `pyyaml`, no Metabase client,
no `kubectl`/Helm/ArgoCD manifests.

## Testing

- **Framework:** `pytest` 9.1.1 (dev group). **Run:** `uv run pytest`. **Location:** `tests/`,
  `testpaths = ["tests"]`. `mypy` also covers `tests`, so test files are type-checked.
- **Today:** `tests/__init__.py` is empty; **zero test files**; `uv run pytest` collects nothing.
- **Decision on record — `trd.md` `D12`** (*"Validação ponta a ponta, sem suíte unitária"*): the
  initiative is validated by running the DAG against a **real pilot dataset** and checking the
  full path PR → merge → `persist_docs` → Metabase.
- **DP-7180 divergence (scoped addition to `D12`, not a reversal):** pytest covers **only
  `src/entrypoint.py`** — subcommand dispatch and env-var validation — because those are pure and
  cheap. BigQuery, the LLM and YAML reconciliation stay validated **end-to-end** on the pilot
  dataset. Never write "this project has no tests" nor "this project has a test suite"; write the
  scoped version.
- **Mocking:** none, by design. No `conftest.py`, no fixtures, no factories, no VCR cassettes.
  Do **not** introduce mocks for BigQuery, Anthropic or the GitHub API.
- **No coverage threshold and no CI pytest job** — neither here nor in any `arco-cv` sibling
  (all four pod connectors have exactly one workflow, `docker_build.yml`). Do not add either.
- Do **not** scaffold test files for `bigquery_schema.py`, `external_doc.py`, `descriptions.py`,
  `yaml_reconciler.py` or `github_pr.py` as if unit coverage were expected there.

## Standards

Tooling conventions (all `[PRESENT]` in `pyproject.toml`):

- **ruff `line-length = 120`**; `[tool.ruff.lint] select = ["E4","E7","E9","F"]` — do not widen
  the rule set without a decision.
- **`quote-style = "double"`, `indent-style = "space"`** (`ruff format`).
- **mypy over `files = ["src", "tests"]`**, `python_version = "3.11"`, `ignore_missing_imports = true`.
- **Python `>=3.11`**; all runtime deps pinned with `==` (no ranges). Keep them pinned.

Architectural constraints (from the TRD Do's & Don'ts — these are not style preferences):

- **Never write descriptions directly to BigQuery or Metabase.** The only external effect of this
  tool is creating/updating the PR (`BR-003`). Writing outside `persist_docs` breaks the
  structural review gate (`BR-011`).
- **Never rewrite the target YAML with `pyyaml`** — it has no faithful round-trip and destroys
  human-maintained comments and tests. Use **`ruamel.yaml`** (`D10`, `M-05`), and write **only**
  `models[].columns[].description`.
- **Never use `max_tokens` to enforce the 1024-character limit** — wrong unit. Truncate the
  returned text **in code** (`BR-008`, `D9`).
- **Never touch a human-written description** — generation fills only empty/absent ones, even if
  the tool wrote the existing text on a previous run (`BR-006`).
- **Never send row data to the LLM, logs or PR body** — column name, type and mode only (`M-06`).
  No `SELECT` on any path.
- **Never create a dbt model** for a table that lacks one — report it in the PR body (`EC-004`).
- **`EC-001` vs `M-01`:** an **inaccessible** doc (404/paywall/no text) **degrades** and the run
  still succeeds at **exit 0**; a URL **refused by network policy** (non-`https`, private/loopback/
  link-local/metadata IP, or any redirect hop resolving to one) is a **hard failure, exit != 0,
  no PR**. This is the easiest thing to get wrong here — see architecture.md §6.1.
- **Never write to `arco-cv/data-dbt`** (migrating out, Q3 2026), and never widen the dbt trigger
  filter beyond `arco_dbt/models/raw/`.

## Divergences from sibling arco-cv pod repos

Deliberate choices. Do **not** "fix" them back toward the sibling pattern.

1. **`pyproject.toml` + `uv` instead of `requirements.txt` + `python-decouple`.** The four sibling
   pod connectors use the latter; the inline contract in DP-7180 specified `pyproject.toml`, and it
   lets ruff/mypy/pytest config live in one file.
2. **Subcommand via argv** (`python -m src.entrypoint generate`) instead of dispatch from an env
   var through a `PROCESSORS` dict. The siblings use the env-var form; the inline contract
   specified argv, and it matches the DAG passing `arguments=["generate"]`.
3. **A pytest suite at all.** The four siblings have zero tests and no pytest CI job. Required by
   this issue's Definition of Done, and scoped narrowly per `D12` as described under Testing.

## CI/CD

**No workflow exists today** — there is no `.github/` directory at all (no workflows, no
`dependabot.yml`, no `CODEOWNERS`, no PR/issue templates).

`[PLANNED]` — the single workflow this repo will get, `.github/workflows/build.yml`: build the
Docker image and push it to GCP Artifact Registry.

- **Image:** `us-central1-docker.pkg.dev/<GCP_PROJECT_ID>/data-external-docs/data-external-docs`
- **Tags:** `:<sha7>` on every build, `:latest` only on `main`. (The sibling
  `docker_build.yml` uses a richer superset — `:<branch>-<sha7>`, `:<branch>`, `:latest`,
  `:pr-N`. Prefer this repo's spec; the sibling scheme is precedent, not this repo's contract.)
- **Auth:** `google-github-actions/auth@v2`, `credentials_json: ${{ secrets.GCP_SA_KEY }}`,
  project id from `${{ secrets.GCP_PROJECT_ID }}`.
- **Triggers:** `push` and `pull_request` on `main`. **Runner:** `ubuntu-latest`.

**This repo has NO deploy step, NO `kubectl`, and NO cluster reference — image publication only.**
This was excluded deliberately: the named reference workflow in the source issue *does* have a
deploy step, and it was left out on purpose. The pod is launched by the Airflow DAG in
`arco-cv/airflow`, which resolves the image from an Airflow Variable
(`var.json.<key>.pod_image`); cluster, namespace, service account and `GKEStartPodOperator` config
all live there (`D3`, `M-03`). Also absent by design: no pytest job, no lint/typecheck job, no
coverage gate, no release workflow, no Dependabot.

## Agent Workflow

- Branch `<type>/ISSUE-ID-slug` off `main` (e.g. `feat/dp-7180-scaffold` — the **current**
  branch). Base branch is always `main`.
- **Conventional Commits** for subjects; commit messages in Portuguese are the observed
  convention (`chore: commit inicial do repositório`).
- Flow: **draft PR → review → merge.** The pod's own GitHub credential cannot merge and cannot
  write `main` (`M-02`) — that is structural, not convention.

## Maintenance

<!-- scaffold: generated by sdk/scaffold -->
Last scaffolded: 2026-08-05
