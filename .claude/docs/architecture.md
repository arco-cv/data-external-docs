# Architecture — data-external-docs

> **Read this first.** This repository is a **scaffold in progress** (Linear issue DP-7180).
> Everything below is labelled with one of three markers, and the distinction is
> load-bearing — do not treat a `[PLANNED]` item as broken or missing code:
>
> - `[PRESENT]` — verified in the working tree today (2026-08-05).
> - `[PLANNED]` — specified in the authoritative TRD / backend plan, **not yet built here**.
> - `[CONVENTION]` — house pattern confirmed in a sibling `arco-cv` repo; not yet applied here.
>
> **What exists today, in full:** `pyproject.toml`, `uv.lock`, an 8-line stub `README.md`,
> `.gitignore`, an empty `src/__init__.py` and an empty `tests/__init__.py`.
> There is **no** `Dockerfile`, **no** `.github/` directory, **no** CI workflow,
> **no** application module and **no** test file.
>
> Authoritative design artifacts live in a **different** repository (`technical-refining`):
> `domains/data-engineering/data-platform/data-external-docs/{trd.md, prd.md, backend/plan.md}`.

---

## 1. What this repo is

`data-external-docs` automatically documents BigQuery column descriptions. It runs as an
**ephemeral Kubernetes pod launched by Airflow** (`GKEStartPodOperator`) — not a service, not a
library. It reads the real schema of a BigQuery dataset, downloads the external API documentation
of the source system, generates PT-BR column descriptions with an LLM, reconciles them into the
versioned `schema.yml` of `arco-cv/dbt`, and opens a Pull Request. **It publishes nothing.**

This repo is **Etapa 1 (propose) only**. Etapa 2 (publish to Metabase) is a Python script that
lives in `arco-cv/dbt`, not here (`D2`). The Airflow DAG that launches the pod lives in
`arco-cv/airflow`, not here (`D3`).

- **Users:** data engineers of the `data-platform` team (trigger + review).
- **Downstream beneficiaries:** data consumers (analysts, PMs, product teams) reading
  descriptions in BigQuery / Metabase.

---

## 2. Directory Tree

### 2.1 Current tree `[PRESENT]` — verified 2026-08-05

```
data-external-docs/
├── .claude/
│   ├── docs/                     # this file
│   └── scaffolder-cache/         # architecture cache produced by the scaffolder
├── .gitignore                    # __pycache__, *.py[cod], .venv/, .pytest_cache/, .mypy_cache/,
│                                 #   .ruff_cache/, .env, *.pem, *.json.key
├── README.md                     # 8 lines (PT-BR): purpose + "Scaffold em construção (DP-7180)"
├── pyproject.toml                # deps + dev group + mypy/ruff/pytest config; [tool.uv] package = false
├── src/
│   └── __init__.py               # EMPTY — no modules yet
├── tests/
│   └── __init__.py               # EMPTY — no test files yet
└── uv.lock                       # 226 KB resolved lockfile, committed
```

**Absent today, stated explicitly so no agent goes hunting:** no `Dockerfile`, no `.github/`,
no CI workflow, no `docs/`, no `CONTRIBUTING.md`, no `Makefile`, no `.env.example`,
no `src/entrypoint.py`, no test files.

**Git state `[PRESENT]`:** single commit `ad7bc51 chore: commit inicial do repositório`;
current branch `feat/dp-7180-scaffold`; `main` also exists.
`.gitignore` and `pyproject.toml` are currently **untracked**.

### 2.2 Target layout `[PLANNED]` — from `backend/plan.md` → `## Architecture`

None of the files below exist yet. The bracketed tags are the Linear task that delivers each one.

```
data-external-docs/
├── Dockerfile                    # [PLANNED] entrypoint with subcommands
├── pyproject.toml                # [PRESENT] deps: anthropic, google-cloud-bigquery, ruamel.yaml, httpx
├── src/
│   ├── entrypoint.py             # [PLANNED] dispatch: generate | generate --dry-run
│   ├── bigquery_schema.py        # [PLANNED, DP-71xx / B2] read + dotted-path flattening
│   ├── external_doc.py           # [PLANNED, DP-71xx / B3] download with SSRF validation
│   ├── descriptions.py           # [PLANNED, DP-71xx / B4] LLM client + 1024-char truncation
│   ├── yaml_reconciler.py        # [PLANNED, DP-71xx / B5] merge BR-006 / EC-003 / EC-004
│   └── github_pr.py              # [PLANNED, DP-71xx / B6] deterministic branch + PR
└── .github/workflows/build.yml   # [PLANNED] build/push to GCP Artifact Registry
```

**`[PLANNED]` layering rule** (verbatim intent from the plan): *"cada módulo é puro em relação
aos demais — recebe dados, devolve dados, sem estado global. O `entrypoint.py` é o único que
orquestra e o único que conhece a ordem."* That purity is what makes execution re-entrant per
table (`D8`) and what lets `M-01` (SSRF defense) be validated in isolation inside
`external_doc.py`.

---

## 3. Architecture Pattern

Single-process **CLI entrypoint dispatching subcommands**, packaged as a Docker image, executed
as an **ephemeral Kubernetes pod launched by Airflow**. Input arrives as **environment
variables**, output is a **GitHub Pull Request**, and failure is communicated as a **process exit
code** surfaced in the Airflow UI. There are no HTTP routes, no server, no request lifecycle, no
persistent state, no database.

### 3.1 Subcommand dispatch surface `[PLANNED]`

Will live in `src/entrypoint.py`. Two surfaces, one code path:

| Surface | Invoked by | Input | Output | External effects |
| --- | --- | --- | --- | --- |
| `generate` (US1) | Docker image entrypoint, launched by `GKEStartPodOperator` in the Airflow DAG (task `[B7]`, in `arco-cv/airflow`) | env vars `DATASET`, `DOC_URL`, `DRY_RUN=false` (contract `C1`) | PR opened or updated in `arco-cv/dbt` (contract `C5`) | **Exclusively the PR** — never writes BigQuery nor Metabase (`BR-003`) |
| `generate --dry-run` (US4) | Same, with `DRY_RUN=true` | Same | Inspectable result on pod stdout | **None.** No PR, nothing changed in dbt, BigQuery or Metabase |

**Hard constraint on `--dry-run`:** it must **not** become a parallel download path — it must go
through the same `external_doc.py`, otherwise `M-01` is only validated in one of the two modes.

**Pod auth `[PLANNED]`:** its **own named** GCP service account (`M-03`, never the namespace
default) plus a **scoped** GitHub credential (`M-02`: one repo, two operations — create/update
branch, open/update PR; cannot merge, cannot write `main`, cannot reach other repos).

### 3.2 Env-var input contract `C1` `[PLANNED]`

Airflow declares typed `Param`s and passes them to the container as env vars **of the same
name**. Never bare `None` — in Airflow 3.x that renders as a JSON editor in the UI.

```python
params={
    "DATASET":  Param(type="string", minLength=1),      # target BigQuery dataset
    "DOC_URL":  Param(type=["null", "string"]),         # external API doc of the source system
    "DRY_RUN":  Param(False, type="boolean"),           # US4 — inspection mode
}
```

**Validation split:** `DATASET` non-empty and `DRY_RUN` boolean are guaranteed by the `Param` on
the Airflow side. **Dataset existence and read permission are only verifiable inside the pod, and
fail there** (`BR-009`). The image reference comes from an Airflow Variable (convention
`var.json.<key>.pod_image`), interpolated via Jinja because `image` is a `template_field`.

### 3.3 Module pipeline `[PLANNED]`

```
Airflow DAG (arco-cv/airflow, [B7])
  └─ GKEStartPodOperator → pod, env: DATASET / DOC_URL / DRY_RUN
       └─ src/entrypoint.py                 # the ONLY orchestrator; the ONLY module that knows the order
            ├─ src/bigquery_schema.py  [B2]
            ├─ src/external_doc.py     [B3]
            ├─ src/descriptions.py     [B4]
            ├─ src/yaml_reconciler.py  [B5]
            └─ src/github_pr.py        [B6]
```

Downstream, **outside this repo**: PR merged → `arco-cv/dbt`'s `trigger-dbt-dag-on-merge.yml`
(needs the `[PLANNED]` `[B8]` extension to match `^arco_dbt/models/raw/.+\.ya?ml$`; today it only
matches `.sql`, so the link is currently broken — risk `R1`) → Airflow dbt run → `persist_docs`
applies descriptions to BigQuery columns → Etapa 2 script in `arco-cv/dbt` mirrors them to
Metabase via `PUT /api/field/:id`.

---

## 4. Module Breakdown

Every module in this section is `[PLANNED]`. **None of these files exist today.** A future agent
opening the repo and not finding them is looking at a correct, in-progress scaffold — not a
broken checkout.

### `src/entrypoint.py` — [PLANNED]
The only orchestrator and the only module that knows the pipeline order. Dispatches the
subcommand from argv (`generate`, `generate --dry-run`) and validates the env-var contract `C1`
before doing anything else. This is the **only** module that DP-7180 puts under pytest — see §7.

### `src/bigquery_schema.py` — [PLANNED, B2] — contract `C2`
Reads the **real** BigQuery schema (metadata only: `get_table` / `INFORMATION_SCHEMA`, never a
`SELECT` over data — `M-06`). Flattens nested `RECORD`/`STRUCT` columns to a **dotted path**
(`parent.child`), which is the key `dbt-bigquery` matches on. Detects a table with no
corresponding dbt model (`EC-004`). Hard-fails on missing dataset, missing read permission, or a
table with no columns (`BR-009`).

### `src/external_doc.py` — [PLANNED, B3] — contract `C1` / `DOC_URL`
Downloads the external API documentation with `httpx`. Carries the **single P0 mitigation of the
whole initiative** (`M-01`): `https://` only; resolve the host **before** connecting; refuse
private / loopback / link-local / metadata IPs; **re-validate every redirect hop** (max 3). Also
enforces size and time ceilings (`M-09`).

### `src/descriptions.py` — [PLANNED, B4] — contract `C3`
LLM client. **One request per table**, covering all columns of that table, with the shared prefix
cached across tables (`D8`). Uses structured output to `GeneratedDescription`. **Filters** the
returned `column_path` against `TableSchema.columns` so an invented column is discarded
(`BR-001` / `BR-002` / `M-04`). Truncates each description to 1024 characters **in code**
(`BR-008` / `D9`). No row values in prompt, log or PR (`M-06`).

### `src/yaml_reconciler.py` — [PLANNED, B5] — contract `C4`
Merges into `arco_dbt/models/raw/<folder>/<source>__<brand>__raw__tests.yaml`. Writes **only**
`models[].columns[].description`; preserves tests, `meta` and comments via `ruamel.yaml`
(`D10` / `M-05`). Implements `BR-006` / `EC-003` / `EC-004` semantics — see §5.

### `src/github_pr.py` — [PLANNED, B6] — contract `C5`
Deterministic branch **per source folder** (`D5`): `docs/raw-descriptions/<folder>`,
base `main`, title `docs(<folder>): descrições de coluna geradas [AUTO RUN DBT]`. The
`[AUTO RUN DBT]` suffix is **mandatory and anchored at end of title** — the downstream gate tests
`[[ "$PR_TITLE" =~ \[AUTO\ RUN\ DBT(\ --full-refresh)?\]$ ]]`. PR body is the execution
transparency report and records `dag_run_id` + user + params (`M-08`). Uses the scoped credential
(`M-02`).

### Contract shapes `[PLANNED]` — `trd.md` → `## Contracts`

All `@dataclass(frozen=True)`:

- `ColumnRef(path: str, type: str, mode: str)` — `path` is the dotted path; `mode` is
  `NULLABLE | REQUIRED | REPEATED`.
- `TableSchema(table: str, dbt_model: str | None, columns: list[ColumnRef])` —
  `dbt_model is None` triggers `EC-004`.
- `GeneratedDescription(column_path: str, description: str)` — `column_path` matches
  `ColumnRef.path`; `description` is PT-BR, already truncated to 1024.

---

## 5. Merge semantics of `C4` `[PLANNED]`

This is where `BR-006` and `EC-003` become code.

| Situation | Action |
| --- | --- |
| Column in schema, no/empty `description` | Fill |
| Column in schema, non-empty `description` | **Do not touch** (`BR-006`) — even if the tool generated it on a previous run |
| Column in YAML but not in current schema | **Remove the entry** (`EC-003`) |
| Column in schema but LLM returned nothing | Leave without `description`; report in PR body |
| Table in dataset with no dbt model | Do not create a model; report in PR body (`EC-004`) |

---

## 6. Error contract — exit codes `[PLANNED]`

There is **no `AppError` class** — this is not an API repo. The failure contract **is the pod
exit code**, per `BR-009`.

| Condition | Exit code | Message |
| --- | --- | --- |
| Dataset nonexistent / no permission / table with no columns | `!= 0` | Explicit, propagated to the Airflow UI |
| `DOC_URL` refused because it resolves to an internal IP (`M-01`) | `!= 0` | Explicit — **does NOT** fall into the degraded path of `EC-001` |
| Second concurrent execution on the same folder (`EC-005`) | `!= 0` | Points at the in-flight execution |
| External doc inaccessible (404, paywall, no text) | `0` | Degrades without enrichment and records it in the PR body (`EC-001`) |
| Column with no approved description in Etapa 2 | `0` | Reported no-op (`BR-005`) |

### 6.1 `EC-001` vs `M-01` — the most-easily-broken distinction in this repo

Verbatim from the plan: *"A distinção entre as linhas de exit `!= 0` e a de `EC-001` é o ponto
mais fácil de errar: doc **inacessível** degrada, doc **recusada por política de rede** falha."*

**`EC-001` — inaccessible doc → DEGRADE, exit 0.**
Triggers: 404, paywall, or the fetch returns no usable text. Behaviour: continue the run without
enrichment, **still produce a proposal covering every column of the real schema**, and record the
degradation in the PR body. This is **not** a failure. The run succeeds.

**`M-01` — URL refused by network policy → HARD FAIL, exit != 0.**
Triggers: scheme is not `https://`; the host resolves to a private / loopback / link-local /
metadata range (`169.254.0.0/16`, `10/8`, `172.16/12`, `192.168/16`, `127/8`, `::1`, `fd00::/8`);
or **any redirect hop** resolves to such an address (max 3 hops). Behaviour: abort with an
explicit message, **no PR**.

**Why the asymmetry is not negotiable:** the pod carries a GCP service account with BigQuery read
plus a GitHub write credential. A `DOC_URL` pointing at the GKE metadata server would turn the
enrichment feature into **SSRF with credential theft**. Validating only the entry URL does not
protect against a redirect to an internal IP. `M-01` is the single **P0** mitigation of the whole
initiative; collapsing it into the `EC-001` degraded path silently removes that defense.

---

## 7. Testing `[PRESENT] + [PLANNED]`

**Today `[PRESENT]`:** `pytest==9.1.1` in the dev group, `testpaths = ["tests"]`,
`tests/__init__.py` empty, **zero test files** — `uv run pytest` collects nothing.
`mypy` covers `files = ["src", "tests"]`, so tests are type-checked too.
**No coverage tooling at all:** no `pytest-cov`, no `coverage`, no `--cov`, no `fail_under`,
no threshold anywhere. No fixtures, no `conftest.py`, no factories, no mocks, no cassettes.

**The decision on record — `trd.md` → `D12`** (*"Validação ponta a ponta, sem suíte unitária"*):
the initiative is validated by running the DAG against a **real pilot dataset** and checking the
full path PR → merge → `persist_docs` → Metabase. Explicit team decision, prioritizing MVP speed
and coverage of the whole integration chain, which is where the integration risk lives.

**The divergence agreed for DP-7180:** `pytest` covers **only** `src/entrypoint.py` — subcommand
dispatch and env-var validation — because those are **pure and cheap** to test. BigQuery, the LLM
and YAML reconciliation stay validated **end-to-end on a pilot dataset**, exactly as `D12`
decided. This is a scoped addition to `D12`, not a reversal. Do not write "the project has no
tests" nor "the project has a test suite" — write the scoped version.

Consequences to respect:
- **No coverage threshold.** Do not add one, do not document one.
- **No CI pytest job** — in this repo or in any `arco-cv` sibling. `[CONVENTION]`
- Do **not** scaffold test files for `bigquery_schema.py`, `external_doc.py`, `descriptions.py`,
  `yaml_reconciler.py` or `github_pr.py` as if unit coverage were expected there.

**Accepted risk `R3`:** `BR-006` (only fill empty) and `EC-003` (remove obsolete) **fail
silently** — a wrong PR still looks like a valid PR. Agreed mitigation: the pilot script **must**
include a second run after human editing, otherwise `US3` is never exercised.

### `[PLANNED]` end-to-end pilot script — mandatory order

1. First run in `dry_run` (`US4`): inspectable result, no PR, nothing changed.
2. First real run (`US1`): PR with one PT-BR description per column of the current schema, all `<=1024` chars.
3. Merge and publication (`US2`): description applied to the BigQuery column, mirrored in Metabase.
4. Re-publication with no change (`US2` AC5): identical result, no side effects.
5. **Second run after human editing (`US3`) — MANDATORY**: edit some descriptions, re-trigger on
   the same dataset, verify (a) edited untouched, (b) empty filled, (c) pending review updated,
   not duplicated.
6. Column removed from schema (`EC-003`): verify the obsolete entry leaves the proposal.
7. Explicit failures (`BR-009`): nonexistent dataset, no permission, table with no columns —
   each `exit != 0` visible in Airflow, no empty PR.
8. Concurrency (`D5` / `EC-005`): two simultaneous triggers on the same folder; the second fails visibly.

**Security test cases** (`trd.md` → `### Test Cases`): `TC-01` internal `DOC_URL` refused
including via redirect (`M-01`); `TC-02` doc-embedded instruction does not alter the column list
(`M-04`); `TC-03` rewrite preserves tests / `meta` / comments (`M-05`); `TC-04` ambiguous
Metabase column is a reported no-op (`M-07`, in `arco-cv/dbt`).

`e2e-tests` repo: **N/A** — that repo covers Isaac's web applications; this initiative has no interface.

---

## 8. CI/CD `[PLANNED]`

**Present today: none.** `[PRESENT]` There is no `.github/` directory at all — no workflows, no
`dependabot.yml`, no `CODEOWNERS`, no PR template, no issue templates.

**The single `[PLANNED]` workflow this repo will get: `.github/workflows/build.yml`** — build the
Docker image and push it to GCP Artifact Registry.

- **Registry / image path:** `us-central1-docker.pkg.dev/<GCP_PROJECT_ID>/data-external-docs/data-external-docs`
- **Tags:** `:<sha7>` on every build, plus `:latest` **only on `main`**.
- **Auth:** `google-github-actions/auth@v2` with `credentials_json: ${{ secrets.GCP_SA_KEY }}`;
  project id from `${{ secrets.GCP_PROJECT_ID }}`.
- **Triggers:** `push` and `pull_request`, both on `main`. **Runner:** `ubuntu-latest`.

**Non-negotiable:** there is **NO deploy step, NO `kubectl`, and NO cluster reference anywhere in
this repo.** The image is only built and pushed. Deployment is not something this repo does — the
pod is launched by the Airflow DAG in `arco-cv/airflow`, which resolves the image from an Airflow
Variable (`var.json.<key>.pod_image`). Cluster (`k8s-sd-data-engineer-prod-1`), namespace
(`dbt-docs-platform-tf`), service account and `GKEStartPodOperator` config all live in
`arco-cv/airflow` (`D3`, `M-03`) — never here.

**Also absent by design:** no pytest job, no lint/typecheck job, no coverage gate, no
release/tagging workflow, no Dependabot.

**`[CONVENTION]` sibling precedent** — `arco-cv/data-dx-connector/.github/workflows/docker_build.yml`.
All four standalone Python pod connectors (`data-dx-connector`, `data-github-connector`,
`zendesk-connector`, `data-recupera-connector`) have exactly this one workflow: checkout → GCP
auth → `gcloud auth configure-docker` → ensure the Artifact Registry repo exists → compute tags →
build → tag/push, job guarded by `if: github.actor != 'dependabot[bot]'`.
Its tag set is a **superset** of this repo's spec: `:<branch>-<sha7>`, `:<branch>`, `:latest`
(main only), `:pr-<number>`. **Prefer this repo's spec**; the sibling scheme is available
precedent, not this repo's contract.

**Related CI in another repo** (`arco-cv/dbt`, task `[B8]`, contract `C6`) — mentioned so nobody
looks for it here: `trigger-dbt-dag-on-merge.yml` currently filters only
`^arco_dbt/models/.+\.sql$` and must gain `^arco_dbt/models/raw/.+\.ya?ml$` without changing
`.sql` behaviour or other layers.

---

## 9. Hard constraints — what this tool must never do

- **`BR-003` — generation does not publish.** The only external effect of Etapa 1 is
  creating/updating the PR. It never alters a description in BigQuery nor in Metabase. Enforced
  structurally: the pod has **no write path** to either, and its service account gets exactly
  three things (`M-03`): BigQuery metadata read, outbound HTTPS, the GitHub credential secret.
- **`BR-011` / `PM-002` — the review gate is structural.** No path exists by which a description
  reaches Metabase without human review, because Etapa 2 reads **exclusively** the description
  already applied to the BigQuery column — never `schema.yml`, never the originally generated
  text (`C7`, `BR-004`). `M-02` depends on this: if the pod could merge, the gate would stop
  being structural and become mere convention.
- **`M-06` — no row data, ever.** Only column name, type and mode plus the external doc text go
  to the LLM. No `SELECT` on any path, no sample values, nothing in prompt, log or PR body.
  Reverting this creates a new processing category with international transfer of potentially
  sensitive data and requires DPO notification before merge.
- **`M-05` — never write anything but `models[].columns[].description`** in the shared YAML, and
  never rewrite it with `pyyaml` (no faithful round-trip; destroys human-maintained comments and
  tests — `D10`). Use `ruamel.yaml`.
- **Never use `max_tokens`** to enforce the 1024-character limit — wrong unit. Truncate in code.
- **`EC-004` — never create a dbt model** for a table without one. Report it in the PR body.
- **Do not write to `arco-cv/data-dbt`** — repo migrating out (Q3 2026).
- **Do not create one `.yml` per model** in `raw/` — contradicts the written convention.
- **Do not widen the dbt trigger filter beyond `raw/`** — the AE team's path must not change.
- **Do not trust `relation: true`** — a no-op in this dbt-bigquery adapter version.

### Known traps

- `bigquery__persist_docs` ignores `for_relation`; only the `for_columns` branch is implemented.
  Irrelevant for the MVP (columns only), but table descriptions do **not** work.
- **Nested columns match by dotted path.** `impl.py:_update_column_dict` recurses into `fields`
  and compares `"{parent}.{name}"`. Declaring `child` alone does not match. Prior art:
  `report__credit__current__credit_eligibility.yml`.
- `policyTags` is skipped in `RECORD`, but `description` is **not** — nested columns take
  descriptions normally.
- The `[AUTO RUN DBT]` gate is at the **END** of the PR title, tested by an anchored regex.
- `sqlfluff-lint` is `continue-on-error` in `slim_ci.yml` — informative, not a gate. What
  actually fails is `check_ownership.py` and `dbt build --select state:modified+1`.

### Prior art worth reading before implementing

- `arco-cv/dbt` → `.claude/skills/create-doc-model/SKILL.md` — **read before implementing
  `yaml_reconciler.py` (`[B5]`)**. It already solves `.yml` synchronization and SQL↔doc drift
  detection; closest prior art.
- `arco-cv/dbt` → `scripts/update_wiki.py` + `llm_wiki_update.yml` — closest analogue to Etapa 1
  already in production in the target repo (generate content, open PR).
- **Negative references, do not copy:** `isaac__*` DAGs (`operators.slack_integration`, raw
  `affinity`, deprecated `generate_pod_affinity()`); per-model ymls in `raw/sheets/` and
  `raw/dlt_sheets/`; `arco-cv/data-dbt` as a write target.

---

## 10. Business rules (`prd.md`) — the behavioral contract

- **BR-001** — the real schema is the single source of truth for the column list, always as of
  execution time. Never the external doc, never previous documentation, never cache.
- **BR-002** — the external doc influences only the text, never the column list.
- **BR-003** — generation does not publish (see §9).
- **BR-004** — publication does not generate text; Etapa 2 only propagates approved descriptions.
- **BR-005** — a column with no approved description is a reported **no-op**, not an error.
- **BR-006** — human editing is untouchable: generation fills only empty or absent descriptions,
  even if the existing text was produced by the tool itself earlier.
- **BR-007** — **one pending review per source system** (not per dataset), covering all its
  datasets; re-running updates the existing review instead of creating another, because the
  versioned documentation of one source lives in a single file.
- **BR-008** — 1024-character ceiling per description (BigQuery's max), so publication does not
  fail after approval.
- **BR-009** — failure is explicit and loud; the tool never creates an empty proposal and never
  ends in silent success.
- **BR-010** — descriptions in PT-BR; the consumer is the internal team.
- **BR-011** — the review gate is structural (see §9).

**Permissions matrix:** `PM-001` no human alters descriptions directly in BigQuery or Metabase —
every change is a consequence of an approval; `PM-002` the publisher publishes only what was
approved; `PM-003` the data consumer is read-only; `PM-004` generation writes only where the
description is empty or absent, making human editing the final authority.

**Edge cases:** `EC-001` inaccessible external doc → degrade, not failure; `EC-002` nested
columns take descriptions like top-level ones, identified by full path; `EC-003` column
renamed/removed after the review is open → reconcile, remove the obsolete entry; `EC-004` table
without a dbt model → report, create nothing; `EC-005` two concurrent generations on the same
source system → the second fails visibly; `EC-006` an interrupted execution must be resumable
with no leftovers — no duplicate review, no orphan review, no partially published documentation.

**Out of scope:** running the dbt pipeline; altering BigQuery descriptions without approval;
creating missing dbt models; documenting Metabase dashboards/questions/metrics; **table**
descriptions (columns only in the MVP); a bespoke daily-use UI; rolling back published
documentation; warehouses other than BigQuery. The existing dbt pipeline in `arco-cv/dbt` is
**not** refactored.

---

## 11. Observability `[PLANNED]`

Structured logs on pod stdout, captured by Airflow:

- `INFO` per processed table: name, column count from schema, descriptions generated, preserved
  by `BR-006`, removed by `EC-003`.
- `WARN` on `EC-001` degradation and on a table without a dbt model (`EC-004`).
- `ERROR` with `exit != 0` for `BR-009`, `M-01` and `EC-005`.

**Forbidden to log:** any row value (`M-06`), the full external-doc content, and the LLM response
body beyond the already-truncated description.

**No metrics stack exists** in these repos. Practical observability is the Airflow UI plus the
per-category counters recorded in the PR body — a deliberate substitute. Alerts:
`on_failure_callback` with `operators.slack_notifier` (conn `slack_conn`) on the DAG — mandatory
repo pattern. No audit table; the trail is `dag_run_id` + user + params in the PR body (`M-08`)
plus git history in `arco-cv/dbt`.

---

## 12. Open dependencies blocking real execution

None are code; all are access or decisions (`backend/plan.md` → `## Dependencies`):
(1) repo creation in the `arco-cv` org — blocks `[B1]` and everything downstream;
(2) pod service account in `dbt-docs-platform-tf` (BigQuery read + internet egress + GitHub
secret) — **no DAG uses this namespace today**, so there is no service-account prior art there,
which reinforces `M-03`;
(3) GitHub credential for the pod (PAT or scoped GitHub App — **not** the Actions
`GITHUB_TOKEN`, because the pod does not run in Actions);
(4) `METABASE_API_TOKEN` as a GitHub secret with a named owner;
(5) runner choice for the Etapa 2 job;
(6) alignment with the AE team about touching `trigger-dbt-dag-on-merge.yml`;
(7) chosen pilot dataset.

---

<!-- scaffold: generated by sdk/scaffold -->
Last scaffolded: 2026-08-05
