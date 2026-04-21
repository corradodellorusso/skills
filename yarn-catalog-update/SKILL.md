---
name: yarn-catalog-update
description: >
  Updates dependency versions in a Yarn Catalog. Detects outdated packages, interactively
  asks for major/minor/patch preference (or auto-picks latest minor in non-interactive mode),
  edits the catalog in place, and prints a change summary. Use when updating dependencies,
  bumping package versions, or auditing the Yarn catalog for available upgrades.
---

# yarn-catalog-update

Checks every dependency declared in the repository's Yarn Catalog for available updates,
lets the user choose the target version (interactive) or auto-selects the latest minor
(non-interactive), rewrites the catalog, and prints a summary.

## When to use

- When the user asks to *update dependencies*, *bump packages*, or *check for outdated packages*
- When the user asks to *update the Yarn catalog* or *upgrade catalog versions*
- As part of routine dependency maintenance on a Yarn 4+ workspace

## Instructions

### 1. Guard: verify the repository uses Yarn with a Catalog

Check that Yarn is the package manager and that a catalog is defined:

```bash
# Confirm Yarn is active (yarn.lock must exist)
if [ ! -f yarn.lock ]; then
  echo "ERROR: No yarn.lock found. This repository does not use Yarn."
  exit 1
fi

# Confirm Yarn version supports catalogs (≥ 4.4.0)
YARN_VERSION=$(yarn --version 2>/dev/null)
```

Detect the catalog. Yarn 4+ catalogs are defined in `.yarnrc.yml` at the repository root
under the `catalog:` key (default catalog) and/or `catalogs:` key (named catalogs):

```bash
if [ ! -f .yarnrc.yml ]; then
  echo "ERROR: No .yarnrc.yml found."
  exit 1
fi

HAS_CATALOG=$(grep -E '^catalog:|^catalogs:' .yarnrc.yml | wc -l)
```

If `HAS_CATALOG` is `0`, **stop immediately** and tell the user:
> "No Yarn Catalog found in `.yarnrc.yml`. Add a `catalog:` block to your `.yarnrc.yml`
> and upgrade to Yarn ≥ 4.10.0 to use this feature."

### 2. Read all catalog entries

Extract the full set of package → version-range pairs from `.yarnrc.yml`. Handle both the
default `catalog:` block and named `catalogs:` blocks.

Use `yq` (YAML processor) to parse the file:

```bash
# Default catalog  →  packages under the top-level "catalog" key
yq '.catalog | to_entries[] | {"catalog": "default", "name": .key, "range": .value}' .yarnrc.yml

# Named catalogs  →  packages under "catalogs.<name>"
yq '.catalogs | to_entries[] | .key as $cat | .value | to_entries[] | {"catalog": $cat, "name": .key, "range": .value}' .yarnrc.yml
```

If `yq` is not installed, fall back to reading the raw YAML with Python:

```bash
python3 -c "
import yaml, json, sys
with open('.yarnrc.yml') as f:
    cfg = yaml.safe_load(f)
entries = []
for pkg, rng in (cfg.get('catalog') or {}).items():
    entries.append({'catalog': 'default', 'name': pkg, 'range': rng})
for cat, pkgs in (cfg.get('catalogs') or {}).items():
    for pkg, rng in pkgs.items():
        entries.append({'catalog': cat, 'name': pkg, 'range': rng})
print(json.dumps(entries))
"
```

Collect the result into a list of records:
`{ catalog, name, currentRange }` (e.g. `{ catalog: "default", name: "react", range: "^18.3.1" }`).

### 3. Resolve the current pinned version and fetch available versions

For every package in the list, query the npm registry:

```bash
# Latest version
LATEST=$(npm view <package> version 2>/dev/null)

# All published versions (for filtering minor/patch)
ALL_VERSIONS=$(npm view <package> versions --json 2>/dev/null)
```

Extract and **preserve** the range prefix (`^`, `~`, `>=`, etc.) from `currentRange` — it
will be reapplied to the new target version when writing back. Extract the bare version
separately for comparison:

```bash
# Extract the prefix (everything before the first digit)
RANGE_PREFIX=$(echo "$currentRange" | grep -oP '^[^0-9]*')

# Extract the bare version (everything from the first digit onward)
CURRENT_VERSION=$(echo "$currentRange" | grep -oP '[0-9].*')
```

Determine what updates are available:
- **patch**: same major.minor, higher patch (e.g. 18.2.0 → 18.2.1)
- **minor**: same major, higher minor (e.g. 18.2.0 → 18.3.0)
- **major**: higher major (e.g. 18.2.0 → 19.0.0)

Skip packages that are already at the latest version.

### 4. Choose the target version

#### Interactive mode (default when running in a terminal with a user present)

For each package that has at least one update available, display a prompt:

```
📦 react  |  current: ^18.2.0  |  available: patch → 18.2.1  |  minor → 18.3.0  |  major → 19.1.0
  Pick update type [patch / minor / major / skip]:
```

Only show options that are actually available. Accept the user's input before moving to the
next package.

#### Non-interactive mode (autopilot / CI / no TTY)

Automatically select the **latest minor** version (same major, highest available minor+patch).
Do **not** prompt. Collect all packages that have a new major available — these will be
reported at the end (step 6).

### 5. Rewrite the catalog in `.yarnrc.yml`

For each package where a target version was chosen (interactive) or auto-selected
(non-interactive), compose the new range by reapplying the `RANGE_PREFIX` extracted in
step 3 to the chosen target version:

```bash
NEW_RANGE="${RANGE_PREFIX}${TARGET_VERSION}"
# e.g.  "^" + "18.3.0"  →  "^18.3.0"
#       "~" + "4.17.21" →  "~4.17.21"
#       ""  + "5.4.5"   →  "5.4.5"   (exact pin)
```

Apply all updates to `.yarnrc.yml` atomically using `yq`:

```bash
# Default catalog entry
yq -i '.catalog.react = "^18.3.0"' .yarnrc.yml

# Named catalog entry
yq -i '.catalogs.react18.react = "^18.3.0"' .yarnrc.yml
```

If `yq` is not available, use Python to load, patch, and rewrite the file in one operation:

```bash
python3 -c "
import yaml, sys
with open('.yarnrc.yml') as f:
    cfg = yaml.safe_load(f)
# Apply patches passed as JSON on stdin
import json
patches = json.loads(sys.argv[1])
for p in patches:
    if p['catalog'] == 'default':
        cfg.setdefault('catalog', {})[p['name']] = p['newRange']
    else:
        cfg.setdefault('catalogs', {}).setdefault(p['catalog'], {})[p['name']] = p['newRange']
with open('.yarnrc.yml.tmp', 'w') as f:
    yaml.dump(cfg, f, default_flow_style=False, allow_unicode=True)
import os; os.replace('.yarnrc.yml.tmp', '.yarnrc.yml')
" '<patches-json>'
```

Always write to a `.tmp` file first and rename atomically to avoid corrupting `.yarnrc.yml`
on failure.

### 6. Print the summary

Render a Markdown table of all changes made:

| Package | Catalog | Previous Range | New Range | Update Type |
|---------|---------|---------------|-----------|-------------|
| react   | default | ^18.2.0        | ^18.3.0   | minor       |
| lodash  | default | ~4.17.20       | ~4.17.21  | patch       |

Below the table, if any packages were skipped (already up to date or user chose "skip"),
list them briefly.

If running in **non-interactive mode**, append a notice for packages where a new **major**
version is available but was not applied:

> **Major updates available (not applied):**
> - `react`: 19.1.0 is available (currently pinned to ^18.3.0)
> - `typescript`: 6.0.0 is available (currently pinned to ^5.4.5)
>
> Re-run in interactive mode to upgrade these, or update them manually.

### 7. Remind the user to install

Always end with:

> Changes written to `.yarnrc.yml`. Run **`yarn install`** to update `yarn.lock` and
> apply the new versions across all workspaces.

## Key constraints

- **Never** modify `yarn.lock` directly — always let `yarn install` regenerate it.
- Preserve the original range prefix (`^`, `~`, exact) per package; do not normalise ranges.
- Do not upgrade a package to a major version without explicit user confirmation (interactive)
  or explicit skip notice (non-interactive).
- If `npm view` fails for a package (private registry, network error, etc.), skip it and
  warn the user rather than aborting the whole run.
- All file writes must be atomic (write to `.tmp` then `mv`) to avoid corrupting
  `.yarnrc.yml` on failure.
- Never expose auth tokens, `.npmrc` credentials, or registry secrets in output.
