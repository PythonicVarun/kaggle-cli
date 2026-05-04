# Plan: Hackathons CLI Commands + MCP↔CLI Parity Gate

**Repository:** `Kaggle/kaggle-cli` (this repo, `kaggleazure`)
**Target release:** 2.2.0 (cut 2026-05-05)
**Branch:** feature branch off `main`
**Single PR title:** `Add hackathons CLI commands + MCP↔CLI parity gate`

---

## 0. Pre-flight checks (do these first; they de-risk the rest)

1. **Confirm `kagglesdk` ships the new clients we need.**
   - The pinned dep is `kagglesdk >= 0.1.21, < 1.0` in `pyproject.toml`
     and `requirements.lock` shows `0.1.20`. We need symbols
     `HackathonClient.get_hackathon_overview`,
     `HackathonClient.list_hackathon_write_ups`,
     `HackathonClient.download_hackathon_write_ups`, and
     `WriteupsClient.get_resolved_writeup_links`.
   - Action: `pip install 'kagglesdk>=0.1.21'` in a scratch venv and
     `python -c "from kagglesdk.competitions.types import hackathon_service; print(dir(hackathon_service))"`
     to confirm the request/response types and the method names on the
     generated client (`kaggle.hackathons.hackathon_api_client.<method>` is the
     expected access path, mirroring `kaggle.discussions.discussion_api_client`).
   - If the symbols are missing in the latest published `kagglesdk`,
     bump the floor in `pyproject.toml` and re-pin `requirements.lock`
     (run `rye lock` or whichever lock workflow this repo uses — check
     `cicd/` scripts before guessing).
   - If the symbols don't exist anywhere yet, **STOP** and surface a
     dependency on the kagglesdk release before continuing PART 1.

2. **Read `McpClient.cs` once** from
   `https://raw.githubusercontent.com/Kaggle/kaggleazure/ci/Kaggle.Sdk/mcp/McpClient.cs`
   to enumerate the *current* set of `[McpServerTool(Name = "...")]`
   names. This list seeds `tools/mcp_cli_mapping.yaml` (PART 2). Keep
   the raw text on hand — we'll paste tool names directly into the YAML.

3. **Inventory existing CLI command paths** by reading
   `src/kaggle/cli.py` and listing every `add_parser("...")` chain.
   This becomes the "valid CLI paths" set the parity script validates
   mappings against.

---

## PART 1 — `kaggle hackathons` (alias `h`) command group

### 1.1 SDK wrappers in `src/kaggle/api/kaggle_api_extended.py`

Add four new public methods + four `_cli` wrappers, modeled on the
forums methods at `src/kaggle/api/kaggle_api_extended.py:2328-2604`.
Keep them grouped together with a `# ---- Hackathons ----` banner
comment, placed immediately after the forums block.

Imports to add near the existing `kagglesdk.discussions.types.*`
block (line ~101):

```python
from kagglesdk.competitions.types.hackathon_service import (
    GetHackathonOverviewRequest,
    ListHackathonWriteUpsRequest,
    DownloadHackathonWriteUpsRequest,
)
from kagglesdk.discussions.types.writeups_service import (
    GetResolvedWriteupLinksRequest,
)
```
(Exact symbol names confirmed in pre-flight #1.)

Methods to add:

| Public method | CLI wrapper | SDK call (via `build_kaggle_client()`) |
|---|---|---|
| `hackathon_get(competition: str)` | `hackathon_get_cli(competition, csv_display=False, quiet=False)` | `kaggle.hackathons.hackathon_api_client.get_hackathon_overview(req)` |
| `hackathon_list_writeups(competition, page_size=None, page_token=None)` | `hackathon_writeups_list_cli(...)` | `...list_hackathon_write_ups(req)` |
| `hackathon_download_writeups(competition, path: Optional[str] = None) -> str` | `hackathon_writeups_download_cli(competition, path=None, quiet=False)` | `...download_hackathon_write_ups(req)` — returns CSV bytes; write to `path or f"./{competition}-writeups.csv"`, return final path |
| `hackathon_resolve_writeup_links(writeup_id: int)` | `hackathon_writeups_resolve_links_cli(writeup_id, csv_display=False, quiet=False)` | `kaggle.discussions.writeups_api_client.get_resolved_writeup_links(req)` |

Output formatting:
- Default JSON (use existing `print_obj` / `print_table` helpers — check
  what forums does for parity; forums actually uses `print_table` /
  `print_csv`. The proposal says default JSON, so use
  `print(json.dumps(<proto>.to_dict(), indent=2, default=str))`. Use
  the same pattern across the four CLI wrappers).
- `--csv` only on the two list-shaped commands (`writeups list`,
  `get` if it returns a tabular shape) using existing `print_csv`.
- `download` writes the file and prints the final path; respects
  `--quiet`.

Add field tuples `hackathon_writeup_fields = (...)` near
`forum_topic_fields` for tabular printing. Pull names directly from
the proto.

### 1.2 Parser in `src/kaggle/cli.py`

Register in `main()` next to `parse_forums(subparsers)` (line 57):

```python
parse_hackathons(subparsers)
```

Add `parse_hackathons` after `parse_forums` (after line 1604), modeled
exactly on the forums parser:

```
hackathons (alias: h)
├── get <competition> [-v|--csv] [-q|--quiet]
└── writeups
    ├── list <competition> [--page-size N] [--page-token TOK] [-v] [-q]
    ├── download <competition> [-p|--path PATH] [-q]
    └── resolve-links <writeup_id> [-v] [-q]
```

Wire each leaf to its `_cli` method via `set_defaults(func=api.<method>)`.

Update the `Help` class (line 1607+):
- Append `"hackathons", "h"` to `kaggle_choices`.
- Add `hackathons_choices = ["get", "writeups"]`.
- Add `hackathons_writeups_choices = ["list", "download", "resolve-links"]`.
- Extend the `kaggle` help string (line 1677+) with the new tree.
- Add help-string constants: `group_hackathons`, `command_hackathons_get`,
  `command_hackathons_writeups_list`,
  `command_hackathons_writeups_download`,
  `command_hackathons_writeups_resolve_links`,
  `param_hackathon_competition`, `param_writeup_id`,
  `param_writeups_download_path`.

### 1.3 Tests — `src/kaggle/test/test_hackathons_cli.py`

Mirror `test_benchmarks_cli.py:1-60` structure:
- `api` fixture with mocked `build_kaggle_client`.
- One test class per command (`TestGet`, `TestWriteupsList`,
  `TestWriteupsDownload`, `TestWriteupsResolveLinks`,
  `TestCliArgParsing`).
- For `download`: mock the SDK to return canned CSV bytes, run in
  `tmp_path`, assert the file exists at the default
  `<competition>-writeups.csv` and at the explicit `-p` path.
- For arg parsing: build the parser via `cli.main`-style setup
  (see how `test_benchmarks_cli.py` does it) and assert that the
  func/dest fields land where expected.

### 1.4 Docs

- `skills/SKILL.md`:
  - Add `├── hackathons (alias: h)  — Browse hackathons & community write-ups` to the command-group tree (line 32-38).
  - Add a bullet under "Specific Tasks" pointing at `references/hackathons.md`.
- `skills/references/hackathons.md` (new file): mirror the structure
  of `references/benchmarks.md` — short intro, one section per verb
  with usage example and expected output shape.
- `CHANGELOG.md`: under the `### Next` heading (line 4), add bullets:
  - `Add hackathons CLI: \`kaggle hackathons get|writeups list|writeups download|writeups resolve-links\``
  - `Add CI gate for MCP↔CLI parity (\`tools/check_mcp_cli_parity.py\`)`
  Also rename `### Next` to `### 2.2.0` if release-day convention for
  this repo is to do so on the cut PR — check the last release PR for
  precedent rather than guessing.

---

## PART 2 — MCP↔CLI parity coverage gate

### 2.1 `tools/check_mcp_cli_parity.py`

Pure stdlib (argparse, urllib.request, re, sys, pathlib, yaml). YAML
is *the* one allowed extra — `pyyaml` is already a transitive dep
(verify via `pip show pyyaml`); if not, add it to `requirements-test.in`
and re-lock, OR write a tiny line-based parser since the mapping file
is flat `key: value`. Prefer the tiny parser to avoid lock churn.

Module shape:

```
def fetch_mcp_client(local_path: Path | None, url: str | None) -> str: ...
def extract_mcp_tool_names(cs_source: str) -> list[str]:
    # regex: r'\[McpServerTool\s*\(\s*Name\s*=\s*"([^"]+)"'
    ...
def extract_cli_command_paths(cli_py: Path) -> set[str]:
    # Walk src/kaggle/cli.py via `ast`. For each call to add_parser(name=...),
    # track the chain of subparsers parents to assemble dotted/space-separated
    # paths like "hackathons writeups download".
    # Use the variable being assigned to (parser_X_Y_Z) to reconstruct depth,
    # OR (more robust) build from the literal `add_parser("...")` argument
    # tied to the lexically-enclosing add_subparsers call.
    ...
def load_mapping(path: Path) -> dict[str, str]: ...
def main() -> int:
    # Load, diff, render markdown, set exit code.
    ...
```

Required CLI:
```
python tools/check_mcp_cli_parity.py
    [--mcp-client PATH]
    [--mcp-client-url URL]
    [--mapping tools/mcp_cli_mapping.yaml]
    [--cli src/kaggle/cli.py]
```
Defaults:
- `--mcp-client` → `../kaggleazure/Kaggle.Sdk/mcp/McpClient.cs`
- `--mcp-client-url` → `https://raw.githubusercontent.com/Kaggle/kaggleazure/ci/Kaggle.Sdk/mcp/McpClient.cs`

Resolution order: local path if it exists, else fetch URL.

Exit codes:
- `0` — all MCP tools mapped, all mapped CLI paths exist.
- `1` — at least one MCP tool has no mapping entry (NEW unmapped).
- `2` — at least one mapping points to a CLI path that doesn't exist.
  (Use `1` for any failure if simpler — proposal only requires
  non-zero. Pick the simpler form.)

Markdown output to stdout — example:

```
## MCP ↔ CLI parity

- MCP tools: 47
- Mapped to CLI: 41
- Skipped (with reason): 6
- Unmapped (FAIL): 0

| MCP tool | Status | CLI path / reason |
|---|---|---|
| get_hackathon_overview | OK | `hackathons get` |
| download_hackathon_write_ups | OK | `hackathons writeups download` |
| authorize | SKIP | browser-only flow |
| ... | | |
```

### 2.2 `tools/mcp_cli_mapping.yaml`

Flat YAML; one entry per MCP tool. Format:

```yaml
# value is either a CLI command path (space-separated) or "skip: <reason>"
get_hackathon_overview: hackathons get
list_hackathon_write_ups: hackathons writeups list
download_hackathon_write_ups: hackathons writeups download
get_resolved_writeup_links: hackathons writeups resolve-links

# already-shipped CLIs (seed from McpClient.cs scan in pre-flight #2):
list_competitions: competitions list
download_competition: competitions download
# ...etc for datasets / kernels / models / forums / benchmarks

# explicit skips
authorize: "skip: browser-only OAuth flow, no CLI surface"
# anything in McpClient.cs that doesn't yet have a CLI:
some_new_tool: "skip: not yet implemented (TODO)"
```

Seed strategy: start from the McpClient.cs scan (pre-flight #2),
then bucket each into `OK with path` / `skip: not yet implemented (TODO)`.
The four hackathon entries above get real paths because we just built
them. **Any MCP tool that lacks both a CLI and a skip reason MUST be
added — that's the whole point of the gate.**

### 2.3 CI workflow

There is currently NO test workflow under `.github/workflows/` —
only `no-response.yaml`. The proposal says "extend the existing test
workflow"; since one doesn't exist, **add a minimal one** rather than
shoehorn parity-checking into `no-response.yaml`.

New file `.github/workflows/parity.yml`:

```yaml
name: MCP ↔ CLI parity
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  parity:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: python tools/check_mcp_cli_parity.py
```

Before adding a *new* workflow file, double-check `cicd/` — this repo
runs CI through Cloud Build (`cicd/` and `requirements-test.lock`
suggest so). If Cloud Build is the source of truth, add the parity
check as a step in the existing CB config instead, and skip the
Actions workflow. **Decision point — confirm with reviewer if
ambiguous; default to adding both a small GH Action (cheap, visible
on PRs) AND a CB step if CB is what gates merges.**

### 2.4 `tools/README.md`

Short doc, two sections:
1. **Running locally**: `python tools/check_mcp_cli_parity.py` —
   note the `--mcp-client` flag for offline runs against a sibling
   kaggleazure checkout.
2. **Adding a new MCP tool**: when the SDK adds a tool, the gate
   fails. Resolution = either (a) ship a CLI command and add the
   `tool_name: cli path` mapping, or (b) add `tool_name: "skip: <reason>"`
   with a real reason (no empty skips).

---

## PART 3 — verification before opening the PR

Run, in order, from repo root:

1. `pytest src/kaggle/test/test_hackathons_cli.py -q` (new tests
   green in isolation first)
2. `pytest -q` (full suite still passes)
3. `ruff check .`
4. `python tools/check_mcp_cli_parity.py` → exit 0, prints table
5. Manual smoke: `python -m kaggle hackathons --help`,
   `python -m kaggle h writeups --help` to confirm parser tree.

If pre-existing failures show up in unrelated tests, leave them
alone (per task scope rules) and note them in the PR description.

---

## File-change summary (for the PR description)

**New**
- `src/kaggle/test/test_hackathons_cli.py`
- `skills/references/hackathons.md`
- `tools/check_mcp_cli_parity.py`
- `tools/mcp_cli_mapping.yaml`
- `tools/README.md`
- `.github/workflows/parity.yml` (or CB step — see 2.3)

**Modified**
- `src/kaggle/api/kaggle_api_extended.py` (4 public + 4 `_cli` methods, 2 imports)
- `src/kaggle/cli.py` (`parse_hackathons`, register call, `Help` updates)
- `skills/SKILL.md` (command tree + reference link)
- `CHANGELOG.md` (2.2.0 bullets)
- `pyproject.toml` / lockfile **only if** kagglesdk floor needs bumping (see pre-flight #1)

---

## Risks / open questions

1. **kagglesdk version availability** — gating risk; flagged in pre-flight #1.
2. **CI venue (GH Actions vs Cloud Build)** — the only workflow today is
   `no-response.yaml`; need to confirm whether tests/lint actually run via
   Cloud Build configs in `cicd/`. Adjust 2.3 accordingly.
3. **YAML parsing without pyyaml** — minor; tiny custom parser keeps the
   tool zero-dep. Recommended.
4. **`skip: not yet implemented (TODO)` bloat** — the seed file may be
   long. That's intentional; the gate's value is catching *new*
   additions, not existing gaps. Reviewers should accept the long file.
5. **CLI path enumeration via ast** — fragile if `cli.py` uses dynamic
   parser construction. Spot-check the produced path set against a
   manual count before committing the script.
