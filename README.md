# ACME Demo Registry

An AART registry. It holds packaged
artifacts - skills, agents, commands, MCP servers, memory and guidelines - that AART installs into
a consumer project.

Its registry id is `acme`. Consumers name it when they add this registry as a source.

## What is in here

| Path | What it is |
|---|---|
| `aart-cli-registry.json` | The registry marker: id, display name, and the AART version window it declares |
| `.aart-cli-version` | The AART version CI runs. One line. Bump it in a pull request |
| `aart-cli-source.json` | Where artifacts and collections live in this tree |
| `artifacts/` | One directory per packaged artifact |
| `collections/` | Named groups of artifacts installed together |
| `registry/` | Approved version records and the catalogs derived from them. Generated |
| `.github/workflows/` | The registry quality gate |

The JSON files and the workflows are **managed**: AART regenerates them and refuses to run against
a copy that was hand-edited. This README is not managed. Edit it freely.

## Everyday commands

Every mutation prepares files and stops so you can read them. Re-run the same command with `--yes`
to finalize. AART never pushes.

```sh
# Discover author manifests in a clean source checkout, then review their exact Candidates
aart-cli registry scan --help
aart-cli registry promote --help

# Copy content an upstream never packaged, recording where it came from
aart-cli registry vendor skill code-review --source . \
  --url https://github.com/acme/prompts.git --ref main --path prompts/code-review \
  --artifact-version 1.0.0 --summary "Review code." --profile claude --platform darwin

# See what moved upstream since a vendored copy was taken
aart-cli registry revendor skill code-review --source .

# Build, validate, audit, and commit - review first, then finalize
aart-cli registry publish --source .
aart-cli registry publish --source . --yes
```

Run the gates yourself at any time:

```sh
aart-cli registry format --source . --check
aart-cli registry validate --source .
aart-cli registry build --source . --check
aart-cli registry audit --source .
aart-cli registry test --source . --compatibility latest
```

## Pointing CI at AART

Two separate questions, kept in two separate places.

**Which AART version** is a decision about this registry, so it lives in Git:

```
.aart-cli-version
0.1.0
```

Bump it in a pull request. The gates then run against the new version **before** the change is
merged, so a version that breaks this registry fails in review rather than after. `git blame`
answers "when did we move to 0.2.0", and a bad bump is one `git revert` away. None of that is
possible when the version lives in a settings page.

The version is also **proved, not just claimed**. After fetching, CI compares `aart-cli --version`
against this file and fails if they differ - which catches a moved tag, an index that resolved to
something else, and a CI image with a stale AART baked into it.

**Where this deployment fetches that version from** is a fact about your instance, not about the
registry, so it stays in repository variables. Four ways in; the **first variable that is set
wins**, and they are never combined:

| Order | Variable | Example | How it fetches |
|---|---|---|---|
| 1 | `AART_CLI_PACKAGE` | `aart-cli=={version}` | `pip` from `AART_CLI_PIP_INDEX_URL` |
| 2 | `AART_CLI_WHEEL_URL` | `https://host/.../v{version}/aart_cli-{version}-py3-none-any.whl` | fetch, then unzip |
| 3 | `AART_CLI_TOOL_PATH` | `/opt/aart` | Already on the runner |
| 4 | `AART_CLI_TOOL_URL` | `https://ghe.corp/platform/aart-cli.git` | `git clone` at `v` + the pin. Private copy: name a secret in `AART_CLI_GIT_CREDENTIALS_SECRET` |

`{version}` is replaced with whatever `.aart-cli-version` says, so the version appears **once**, in
Git, and never in a settings page. Set `AART_CLI_REF` to override the pin for one registry - the run
then says so out loud and the version check is switched off, because you asked for a different
build on purpose.

The order runs from the most governed supply chain to the least. That matters when you migrate:
stand up an internal index later, set `AART_CLI_PACKAGE`, and it takes over. You do not have to unset
anything first.

**Set none of them** and the first run stops and says so, listing these four. Where AART comes
from is a fact about your deployment, and nothing here can guess it: a shipped default would send
every company's registry to a repository nobody in it had chosen.

**Set them on the organisation, not here.** GitHub resolves a repository variable over an
organisation one, so one organisation variable configures every registry your company has, and any
single registry can still override it.

Which arm actually answered is printed by the run:

```
AART: aart-cli 0.1.0  via index https://nexus.corp/pypi/simple (aart-cli==0.1.0)
```

### The other variables

| Variable | Default | What it does |
|---|---|---|
| `AART_CLI_PIP_INDEX_URL` | `https://pypi.org/simple` | Index used by `AART_CLI_PACKAGE` |
| `AART_CLI_REPOSITORY` | unset | `owner/name` of your AART repository, combined with this instance's own URL. Shorter than `AART_CLI_TOOL_URL` when the copy is on this instance |
| `AART_CLI_GIT_CREDENTIALS_SECRET` | unset | **Name** of a secret holding a token, or `user:token`, for the `git clone` arm. A bare token is used as `x-access-token`. Without it the clone is anonymous, and a private copy answers `could not read Username` |
| `AART_CLI_REF` | `v` + the pin | Escape hatch: a branch or tag instead of `.aart-cli-version`. Switches the version check off |
| `AART_CLI_RUNNER` | `["ubuntu-latest"]` | JSON array of runner labels. Must be JSON, not a bare word |
| `AART_CLI_CI_IMAGE` | unset | Container image for the jobs. Unset means the runner's own environment |
| `AART_CLI_PYTHON` | `python3` | The interpreter's name inside that image |

## Protecting `main`

Require one status check: **`registry-quality-gate`**.

Do not require the gate jobs themselves. Each is a matrix, so the name GitHub reports carries a
compatibility arm, and the quality job is emitted in two container shapes of which only one ever
runs on your instance. A rule naming the shape you do not use would never be satisfied - except
that GitHub counts a *skipped* required check as satisfied, so it would pass, having proven
nothing. `registry-quality-gate` has the same name in every configuration, runs whatever its
dependencies did, and fails if any arm failed or if no arm ran at all.

## The version window

`aart-cli-registry.json` declares the *range* of AART versions this registry supports. The quality
gate runs at **both** ends of it, which is what `compatibility: [minimum, latest]` means in the
workflow.

That is a different statement from `.aart-cli-version`. The window says which versions this registry
claims to work with; the pin says which single version CI actually runs. Keep the pin inside the
window - a pin outside it is a registry contradicting itself.

