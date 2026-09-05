---
name: "language-toolchain-detection"
description: "Determine a repository's language, package manager, and install/build/test/lint commands from its own manifests, and record them as a profile every other stage reads. The single place in this workflow where any language is named. Use before generating or validating code."
version: 6
created: "2026-08-20"
updated: "2026-09-05"
---

# Language and toolchain detection

Every other agent in this workflow is language-agnostic. They ask this profile what to run,
what to call things, and where tests go. If a language name appears anywhere else in the
pipeline, it leaked from here.

A script writes `50-validation/Toolchain-Profile.json` during preflight. Your job is to
**confirm, correct and complete it** against what is actually in the repository, because the
script uses heuristics and a real repository often overrides them in its own config.

## Evidence beats convention, always

Every command in the profile carries `evidence` — the reason it was chosen. Two grades:

- **Declared** — the repository says so. A script in `package.json`, a `[tool.poetry.scripts]`
  entry, a Makefile target, a CI workflow step, a documented command in CONTRIBUTING.
- **Conventional** — the language's usual command, chosen because nothing declared one.

The distinction is not bookkeeping. When a check fails, the fixer must know whether the *code*
is wrong or the *command* is wrong, and a declared command that fails means the code, while a
conventional command that fails might mean neither.

**The CI configuration is the best evidence in the repository.** `.github/workflows/`,
`.gitlab-ci.yml`, `bitbucket-pipelines.yml`, `Jenkinsfile` — whatever runs there is what the
project considers correct, verbatim. Read it before falling back to convention.

## Order of investigation

1. `50-validation/Toolchain-Profile.json` — what preflight already found.
2. Manifests: `pyproject.toml`, `requirements*.txt`, `Pipfile`, `package.json`, `pom.xml`,
   `build.gradle*`, `go.mod`, `Cargo.toml`, `*.csproj`, `*.sln`, `mix.exs`, `Gemfile`,
   `composer.json`.
3. Lockfiles, which settle the package manager where the manifest does not: `uv.lock`,
   `poetry.lock`, `pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`, `Cargo.lock`.
4. CI configuration.
5. `Makefile`, `Taskfile.yml`, `justfile` — a target named `test` outranks any convention.
6. Tool config: `pytest.ini`, `tox.ini`, `setup.cfg`, `jest.config.*`, `.golangci.yml`.
7. README / CONTRIBUTING, last. Documentation drifts; lockfiles do not.

## Multiple languages

Real repositories mix them: a Python service with a TypeScript CDK app, a Java backend with a
Node frontend.

Record the **primary** language — the one the specification targets — and list the others in
`alsoDetected` with their own root paths. Do not silently pick one. Where it is genuinely
ambiguous which the spec targets, raise a gap: generating Python into a repository whose
service is Java is not a defect anyone reviews their way out of.

## Python specifics

This is the profile that gets exercised first, so get it right:

- `pyproject.toml` with `[tool.poetry]` → `poetry install`, and commands run under
  `poetry run`.
- `pyproject.toml` with `uv.lock` → `uv sync`, commands under `uv run`.
- `pyproject.toml` alone → `pip install -e .`, commands run bare.
- `requirements.txt` only → `pip install -r requirements.txt`.
- **Look for dev requirements** — `requirements-dev.txt`, a `[project.optional-dependencies]`
  `dev` extra, a poetry dev group. Tests will not import without them, and a missing test
  dependency looks exactly like broken generated code.
- Test runner: `pytest` unless `unittest` is evidently in use. Check `pytest.ini`,
  `tool.pytest.ini_options`, `tox.ini`, and the shape of the existing tests.
- **When commands run bare** (a `pip install -r requirements.txt` / plain-`pyproject` project with
  no `poetry run` / `uv run` prefix), invoke every tool in **module form** — `python -m pytest`,
  `python -m pytest --cov=…`, `python -m ruff`, `python -m mypy` — NOT the bare console script
  (`pytest`, `ruff`). A console script is only on `PATH` when the package manager put it there;
  `python -m <tool>` works whenever the tool is importable, which is what Run Validation actually
  has. Bare `pytest` "command not found" is a false FAIL on code that runs fine under `python -m`.
- Lint and types: `ruff`, `flake8`, `pylint`, `mypy`, `pyright` — record a `lint`/`typecheck`
  command **only when the tool is both configured AND actually installed** (importable in the
  environment Run Validation uses). A tool the repo configures but the environment does not have must
  NOT be recorded: Run Validation would execute a missing binary and report FAIL where the check
  should be SKIPPED. If you cannot confirm it is installed, leave the command `null` and say so in
  `warnings`. A lint failure on code the project never linted is noise that costs a retry.
- Python usually has **no build step**. Record `build: null` with the reason. Reporting a
  missing build as a failure sends the fixer after a problem that does not exist.

## Other languages, briefly

The same discipline; the table is just filled in more thinly. Confirm each against the
repository rather than trusting the row.

| Language | Install | Build | Test | Lint |
|---|---|---|---|---|
| Node | `npm ci` / `pnpm i` / `yarn` | `npm run build` if declared | `npm test` | `npm run lint` if declared |
| Java (maven) | `mvn -B dependency:go-offline` | `mvn -B compile` | `mvn -B test` | via plugin only |
| Java (gradle) | — | `./gradlew build -x test` | `./gradlew test` | via plugin only |
| Go | `go mod download` | `go build ./...` | `go test ./...` | `go vet ./...` |
| Rust | — | `cargo build` | `cargo test` | `cargo clippy` |
| .NET | `dotnet restore` | `dotnet build --no-restore` | `dotnet test --no-build` | — |

For Node and Java, `npm run <script>` and gradle/maven goals should come from the manifest,
not from this table. The table is the fallback.

## Output

Update `50-validation/Toolchain-Profile.json` in place:

```json
{
  "schemaVersion": "1.0",
  "status": "OK|PARTIAL",
  "language": "python",
  "languageVersion": "3.11",
  "packageManager": "poetry",
  "manifests": ["pyproject.toml"],
  "alsoDetected": [{"language": "typescript", "root": "infra/"}],
  "commands": {
    "install": "poetry install",
    "installDev": null,
    "build": null,
    "test": "poetry run pytest -q",
    "coverage": "poetry run pytest -q --cov=. --cov-report=term-missing",
    "lint": "poetry run ruff check .",
    "typecheck": "poetry run mypy src"
  },
  "coverageRegex": "TOTAL\\s+\\d+\\s+\\d+\\s+(\\d+(?:\\.\\d+)?)%",
  "evidence": {
    "install": "declared: pyproject.toml has [tool.poetry]",
    "test": "declared: .github/workflows/ci.yml runs 'poetry run pytest -q'",
    "lint": "declared: ruff configured in pyproject.toml",
    "build": "python project with no build step; tests are the gate"
  },
  "testFramework": "pytest",
  "testPathConventions": ["tests/", "test_*.py"],
  "runPrefix": "poetry run ",
  "warnings": []
}
```

`runPrefix` saves every downstream agent from re-deriving whether commands need `poetry run`
or `uv run` in front of them.

## Coverage is a SEPARATE command

Line coverage must exceed 80%, and the validation stage enforces it. Record a `coverage` command
distinct from `test`, plus a `coverageRegex` that extracts the total line-% from that command's
output. Keeping it separate is deliberate: the plain `test` command stays the authoritative
pass/fail and is never broken by a missing coverage tool. Per language, the recipe is the one the
ecosystem already uses — Python `pytest --cov --cov-report=term-missing` (regex on the `TOTAL`
row), Node/jest `--coverage --coverageReporters=text` (the `All files` row), Go `go test -cover`
(`coverage: NN.N% of statements`). A language with no clean built-in recipe records no coverage
command; coverage is then reported as null, never a failure.

## A standards profile can add required checks

If `00-inputs/Standards-Profile.json` is present, read its `requiredCommands`. A standards document
that mandates a lint, a type check or a coverage floor is naming a validation gate, so fold those
commands into the profile's `commands` (and `coverageRegex` / coverage floor) with `evidence` that
says "required by Standards-Profile.json rule STD-0xx". Two caveats: the command must actually be
runnable in the repo's toolchain (a mandated `ruff` needs `ruff` installed or declared — if it is
not, record it under `evidence` as required-but-unavailable rather than inventing an invocation),
and a standards check never *replaces* a check the repository already runs, it only adds. When the
file is absent this section is a no-op and detection is exactly as above.

## A monorepo has no manifest at its root

This is common and it is the case that has already cost a run. On 2026-08-20 a monorepo of
some forty projects under a couple of top-level directories -- each with its own dependency
manifest, none at the repo root -- was profiled as `language: unknown`. Nothing was
installed, every validation check was skipped, and the run generated 22 files including 8
test files that were never executed. It reported success.

So when the checkout root holds no manifest, **do not conclude the language is unknown.** Look
one to four levels down for project roots, skipping `node_modules`, `.venv`, `site-packages`,
`vendor`, `__pycache__` and the like. The preflight script now does this and records what it
found in `candidateRoots[]`.

Two things follow, and both matter:

**The language is decided; the directory is not.** `candidateRoots` lists every project root
found. Which one this run targets is not knowable at preflight time -- it depends on a
specification that has not been written yet. So the profile carries `status: PARTIAL` and a
warning saying the commands are right for the language but must be run inside the target
project. Run Validation resolves the directory from `40-codegen/Generated-Files.json`, because
the generated files are the only artifact that knows where the run actually wrote.

**Do not prune directories by build-output name.** `dist`, `build`, `target`, `out` and `bin`
are ordinary project names in a monorepo. Pruning them by name lost a real project in testing:
a container directory matched a build-output name and the doubled project dir nested inside it
was never reached. Prune only what is never a project root and always enormous.

When you correct the profile as RepositoryDiscoveryAgent, keep `candidateRoots` — a later stage
reads it. If you can tell from the specification which project is being targeted, say so in a
`targetRoot` field; if you cannot, leave it out rather than guessing.

## An unknown language is a PARTIAL, not a failure

An empty repository, or one with no recognisable manifest, is legitimate — especially on a
create run into a fresh repo.

Record `language: "unknown"`, set `status: PARTIAL`, and say what you looked for. The
specification stage will name the language from the requirements and the standards document,
and the code agents will create the manifest as part of the scaffold. Do not guess a language
from a single file extension and commit the pipeline to it.
