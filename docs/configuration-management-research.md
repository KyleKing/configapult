# Configuration Management Research

**Date:** 2025-11-23
**Context:** Researching alternatives and approaches to configuration management beyond copier, particularly for mono-repo scenarios and flexible configuration reuse.

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Use Cases](#use-cases)
3. [Tools Research](#tools-research)
4. [Analysis by Use Case](#analysis-by-use-case)
5. [Developer Experience (DX) Considerations](#developer-experience-dx-considerations)
6. [Recommendations](#recommendations)
7. [Implementation Patterns](#implementation-patterns)
8. [Sources](#sources)

---

## Executive Summary

After researching 15+ configuration management tools and approaches, the landscape breaks into several categories:

1. **Template Sync Tools** (copier, cruft, cookiecutter) - Good for initial scaffolding, poor for ongoing sync
2. **Declarative Sync Tools** (vendir) - Excellent for vendoring dependencies, not for local config management
3. **Overlay/Patch Systems** (kustomize, ytt) - Powerful layering, but Kubernetes-focused
4. **Configuration Languages** (jsonnet, CUE, dhall) - Type-safe generation, steep learning curves
5. **Template Engines** (templr, go templates) - Simple templating, manual management
6. **Symlink Managers** (stow, chezmoi) - Dotfile-focused, not suitable for multi-format configs
7. **Merge Utilities** (config-merge, custom scripts) - Tactical solutions, require orchestration

**Key Finding:** No existing tool perfectly addresses mono-repo configuration de-duplication with drift detection and flexible merging. There's an opportunity to build something purpose-built.

---

## Use Cases

Based on the initial request, here are the prioritized use cases:

### 1. Mono-Repo Configuration De-duplication
**Problem:** Tools like ruff, pyright, and other linters don't always support mono-repos well. You can't always symlink configuration files, leading to duplication.

**Requirements:**
- Handle TOML/YAML/JSON configuration formats
- Support partial merging (some keys from template, some local)
- Allow intentional divergence tracking
- Detect drift from "blessed" configuration

**Current State:**
- Ruff: [First-class monorepo support](https://github.com/astral-sh/ruff) with hierarchical config
- Pyright: [Supports monorepos](https://github.com/microsoft/pyright/issues/4366) via `executionEnvironments`
- Many tools still don't support config inheritance

### 2. Centralized Configuration Store
**Problem:** Need a versioned, reusable configuration library that can be referenced across repos and folders.

**Requirements:**
- Version-controlled configuration templates
- Selective inclusion (pick specific sections)
- Cross-repository sharing
- Local override capability

### 3. AI Tooling Support
**Problem:** Claude Skills and similar AI tools need consistent, discoverable configuration patterns.

**Requirements:**
- Standardized structure for AI discovery
- Clear metadata about configuration purpose
- Template-driven generation of AI-friendly artifacts

### 4. Selective Code Sharing
**Problem:** Sometimes you just want to copy files/functions between projects without creating a Python package (especially useful for Docker builds).

**Requirements:**
- File/function selection and copying
- Dependency tracking
- Update notifications
- Simple, not package-based

---

## Tools Research

### Template-Based Project Scaffolding

#### Copier
**What:** Template-based project scaffolding with update capability
**Pros:**
- Rich templating with Jinja2
- Can update existing projects
- Good for initial scaffolding

**Cons:**
- Not designed for ongoing sync of small config snippets
- "Anti-goal" is to reduce variability
- Becomes complicated with many small variations
- Poor mono-repo support

**Best For:** Initial project creation from templates

#### Cruft
**What:** [Cookiecutter wrapper](https://cruft.github.io/cruft/) with template sync capability
**Pros:**
- `.cruft.json` tracks template version and parameters
- `cruft check` validates template freshness
- `cruft update` merges template changes
- Skip patterns for manual files
- CI/CD integration

**Cons:**
- Still whole-project focused, not config-snippet focused
- Merge conflicts on updates
- Same limitations as cookiecutter/copier

**Best For:** Projects that want to stay in sync with a master template

**Sources:** [cruft docs](https://cruft.github.io/cruft/), [Platform Engineering with Cookiecutter](https://john-miller.dev/posts/cookiecutter-with-cruft-for-platform-engineering/)

---

### Declarative Content Synchronization

#### Vendir
**What:** [Declarative directory sync](https://carvel.dev/vendir) tool from Carvel (Kubernetes ecosystem)
**Pros:**
- Declarative `vendir.yml` configuration
- Multiple sources (git, docker, helm, HTTP)
- Lock files for reproducibility
- Selective path inclusion
- Manual override support

**Cons:**
- Focused on vendoring external dependencies
- Not designed for template merging or local customization
- More of a "download and organize" tool

**Best For:** Vendoring external dependencies into your repo

**Example Use:**
```yaml
apiVersion: vendir.k14s.io/v1alpha1
kind: Config
directories:
- path: vendor
  contents:
  - path: github.com/example/lib
    git:
      url: https://github.com/example/lib
      ref: v1.0.0
```

---

### Configuration Overlay & Patching

#### Kustomize
**What:** [Kubernetes-native configuration management](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) using overlays
**Pros:**
- Powerful overlay/patch pattern (base + overlays)
- No templates, works with raw YAML
- Strategic merge patches + JSON patches
- Great for environment-specific configs

**Cons:**
- Kubernetes YAML focused
- Doesn't work well with TOML/other formats
- Learning curve for patch strategies

**Best For:** Multi-environment Kubernetes configs

**Pattern:**
```
base/
  deployment.yaml
overlays/
  dev/
    kustomization.yaml  # patches for dev
  prod/
    kustomization.yaml  # patches for prod
```

**Sources:** [Kubernetes docs](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/), [Glasskube tutorial](https://glasskube.dev/blog/patching-with-kustomize/)

#### ytt (YAML Templating Tool)
**What:** [Carvel's YAML templating](https://carvel.dev/ytt/) with overlay support
**Pros:**
- Understands YAML structure (not text-based)
- Overlay package for declarative patching
- Starlark (Python-like) for logic
- Deterministic, hermetic execution
- No filesystem/network access in templates

**Cons:**
- YAML-only
- Starlark learning curve
- Kubernetes ecosystem focused

**Best For:** Complex YAML templating with type safety

**Example:**
```yaml
#@ load("@ytt:overlay", "overlay")

#@overlay/match by=overlay.subset({"kind": "Deployment"})
---
spec:
  replicas: 3
```

**Sources:** [ytt docs](https://carvel.dev/ytt/), [overlay documentation](https://github.com/carvel-dev/ytt/blob/develop/docs/ytt-overlays.md)

---

### Configuration Languages

#### Jsonnet
**What:** Google-born (2014) data templating language
**Pros:**
- Functions and imports for code reuse
- Outputs JSON (can convert to YAML)
- Mature, widely adopted (Kubernetes, Prometheus)
- Rich standard library

**Cons:**
- Not statically typed
- Limited to JSON output
- Learning curve for custom syntax

**Best For:** Complex, programmable JSON/YAML generation

#### CUE
**What:** [Logic programming approach](https://github.com/cue-lang/cue/discussions/669) to configuration
**Pros:**
- Data and schema are both data (unification)
- Strong typing with inference
- Excellent composability model
- Validation built-in

**Cons:**
- Steepest learning curve of the three
- Newer, smaller ecosystem
- Paradigm shift from imperative thinking

**Best For:** Type-safe configuration with validation

#### Dhall
**What:** [Statically-typed functional](https://pv.wtf/posts/taming-the-beast) configuration language
**Pros:**
- Strongest type system of the three
- Totality (guaranteed termination)
- Compile-time error prevention

**Cons:**
- Most complex type system
- Highest learning curve
- Smallest ecosystem

**Best For:** Maximum safety and correctness

**Comparison:** All three try to bring structure to YAML/JSON. Jsonnet is simplest but least safe. Dhall is safest but most complex. CUE is a middle ground with unique unification model.

**Sources:** [Taming the Beast comparison](https://pv.wtf/posts/taming-the-beast), [CUE comparisons](https://github.com/cue-lang/cue/discussions/669)

---

### Template Engines

#### templr
**What:** [Go template-based](https://github.com/kanopi/templr) config renderer
**Pros:**
- Simple Go template syntax
- Data files (YAML/JSON) + templates
- Lint mode for validation
- Guard protection against overwrites
- 100+ Sprig functions

**Cons:**
- Manual workflow (run to regenerate)
- No automatic drift detection
- Template syntax can be verbose

**Best For:** CI/CD config generation with validation

**Example:**
```bash
templr render -in config.tpl -data values.yaml -out config.yaml
```

---

### Merge & Sync Utilities

#### config-merge
**What:** [CLI tool](https://github.com/boxboat/config-merge) for merging JSON/TOML/YAML
**Pros:**
- Simple merge semantics (lodash merge)
- Multi-format support
- Environment variable substitution

**Cons:**
- Simple merge only (no advanced strategies)
- No drift detection
- Manual orchestration required

**Best For:** Quick file merging in scripts

#### Custom Sync Scripts
**Example:** [mdformat-plugin-template sync_pyproject.py](https://github.com/KyleKing/mdformat-plugin-template/blob/f0f1b37809bfa80c004c21313a4dfe8d3f5127df/sync_pyproject.py#L17)

**Pattern:**
```python
# Parse both files with tomlkit
ctt_doc = tomlkit.parse(ctt_pyproject.read_text())
tl_doc = tomlkit.parse(tl_pyproject.read_text())

# Define which keys to sync (blacklist approach)
synced_keys = {*ctt_doc["tool"].keys()} - {"pytest-watcher", "mypy", ...}

# Copy keys from template to target
for key in synced_keys:
    tl_doc["tool"][key] = ctt_doc["tool"][key]

# Write back
tl_pyproject.write_text(tomlkit.dumps(tl_doc))
```

**Pros:**
- Full control over merge logic
- Preserves formatting with tomlkit
- Can implement any merge strategy

**Cons:**
- One-off, not reusable
- No drift detection
- Requires Python/tomlkit knowledge

---

### Symlink & Dotfile Managers

#### GNU Stow
**What:** [Symlink farm manager](https://www.gnu.org/software/stow/)
**Pros:**
- Simple symlink management
- Mirror directory structure
- `.stow-local-ignore` for exclusions

**Cons:**
- Symlinks only (doesn't work when tools can't follow symlinks)
- No templating or merging
- Directory structure must match exactly

**Best For:** Dotfile management with symlinks

**Sources:** [System Crafters guide](https://systemcrafters.net/managing-your-dotfiles/using-gnu-stow/)

#### chezmoi
**What:** [Cross-platform dotfile manager](https://www.chezmoi.io/) with templates
**Pros:**
- Template support (text/template + Sprig)
- Machine-to-machine differences
- Password manager integration
- Git integration
- Cross-platform (Linux/macOS/Windows)

**Cons:**
- Dotfile-focused (home directory)
- Not designed for project configuration
- Opinionated directory structure

**Best For:** Personal dotfile management across machines

**Sources:** [chezmoi docs](https://www.chezmoi.io/), [Frictionless Dotfile Management](https://marcusb.org/posts/2025/01/frictionless-dotfile-management-with-chezmoi/)

---

### Configuration Drift Detection

#### Infrastructure Tools
- **Terraform/OpenTofu:** `terraform plan` shows drift from IaC state
- **Ansible:** Check mode for dry-run drift detection
- **Puppet/Chef:** Continuous enforcement with drift alerts
- **driftctl:** [Open-source CLI](https://snyk.io/blog/tools-infrastructure-drift-detection/) for infrastructure drift
- **ArgoCD/Flux:** GitOps with automatic drift detection and remediation

**Pattern:** Compare "desired state" (in git/IaC) vs "actual state" (running systems)

**Key Capability:** Periodic scanning + baseline comparison + alerting

**Sources:** [Wiz Configuration Drift](https://www.wiz.io/academy/configuration-drift), [Spacelift drift management](https://spacelift.io/blog/drift-management), [Snyk drift detection tools](https://snyk.io/blog/tools-infrastructure-drift-detection/)

---

## Analysis by Use Case

### Use Case 1: Mono-Repo Configuration De-duplication

**Best Existing Solutions:**
1. **Native tool support** (if available)
   - Ruff and Pyright now support hierarchical configs
   - Check if your tools support this before building custom solutions

2. **Custom merge scripts** (like sync_pyproject.py)
   - Full control over merge strategy
   - Can use tomlkit/ruamel.yaml to preserve formatting
   - Good for 1-2 config files

3. **kustomize-style overlays** (adapted for TOML/YAML)
   - Base configuration + environment-specific patches
   - Would need custom tooling for non-YAML formats

**Gaps:**
- No good "drift detection" for config files
- Hard to track intentional vs unintentional divergence
- No standard way to mark "this key is managed, this key is local"

**Opportunity:** Build a tool that:
- Manages base + overlay pattern for TOML/YAML/JSON
- Tracks which keys are "managed" vs "local"
- Detects and reports drift
- Supports mono-repo directory structures

---

### Use Case 2: Centralized Configuration Store

**Best Existing Solutions:**
1. **vendir** for vendoring configurations
   - Pull from git repos at specific versions
   - Lock files for reproducibility
   - Good for read-only vendoring

2. **Git submodules/subtrees**
   - Native git solutions
   - Subtrees better for vendoring
   - No selective inclusion

3. **Custom templating** (templr, ytt)
   - Define templates in central repo
   - Render locally with project-specific values
   - Manual workflow

**Gaps:**
- No selective key inclusion from remote configs
- Hard to version "blessed" configurations
- No easy "use ruff config v2.1.0 with local overrides"

**Opportunity:** Build a system like:
```yaml
# .configapult.yml
sources:
  - name: company-standards
    git: https://github.com/company/config-templates
    ref: v1.2.3

configs:
  - target: pyproject.toml
    merge:
      - source: company-standards
        path: ruff.toml
        keys: ["tool.ruff.lint", "tool.ruff.format"]
      - local: pyproject.local.toml
        keys: ["tool.ruff.exclude"]
```

---

### Use Case 3: AI Tooling Support

**Requirements:**
- Standardized, discoverable structure
- Metadata about configuration purpose
- Template-driven generation

**Best Existing Solutions:**
1. **JSON Schema** for configuration validation
2. **Conventional directory structures**
3. **Metadata files** (like .copier-answers.yml)

**Opportunity:**
- Define standard metadata format for AI tools
- Auto-generate from templates
- Make configurations "AI-readable" with context

---

### Use Case 4: Selective Code Sharing

**Best Existing Solutions:**
1. **Git sparse checkout** for selective file pulling
2. **vendir** for vendoring specific paths
3. **Copy-paste with tracking** (manual)

**Gaps:**
- No dependency tracking for copied code
- No update notifications
- Hard to know if copied code is stale

**Opportunity:**
- Track copied files in manifest
- Check for updates in source
- Alert on divergence
- Like vendir but for individual files/functions

---

## Developer Experience (DX) Considerations

Based on research into [mise](https://mise.jdx.dev/) (by jdx), great DX means:

### 1. **Zero to Hero in Minutes**
- Single binary, no dependencies
- Works immediately with sensible defaults
- Progressive disclosure of complexity

**mise example:** `curl https://mise.run | sh` → instant install

### 2. **Intuitive Commands**
- Verb-based CLI (sync, check, diff, update)
- Clear, helpful error messages
- `--help` that actually helps

**mise example:** `mise install node@20` → just works

### 3. **Fast Feedback Loops**
- Instant validation
- Dry-run mode for safety
- Clear diff output

### 4. **Transparency**
- Show what's happening
- Make state visible
- Easy to undo mistakes

### 5. **Smart Defaults, Easy Overrides**
- Convention over configuration
- But escape hatches everywhere
- User can override any behavior

### 6. **Plays Well With Others**
- Doesn't fight existing tools
- Generates artifacts that look hand-written
- Can be incrementally adopted

**Sources:** [mise discussions](https://github.com/jdx/mise/discussions/4057), [mise docs](https://mise.jdx.dev/)

---

## Recommendations

### Option A: Use Existing Tools Creatively

**For mono-repo configs:**
1. Check if tools support native hierarchical configs (Ruff, Pyright do)
2. Use custom merge scripts with tomlkit/ruamel.yaml
3. Add pre-commit hooks to detect drift

**For centralized configs:**
1. Use vendir to vendor configuration templates
2. Use templr or ytt for rendering
3. Store in git, reference by version

**For code sharing:**
1. Use vendir with selective paths
2. Track in lock file
3. Manual update workflow

**Pros:** Leverage existing, tested tools
**Cons:** Duct-tape solution, manual orchestration

---

### Option B: Build Configapult

Based on research, here's what a purpose-built tool could be:

#### Core Concept
**"Overlay-based configuration management with drift detection"**

Think: kustomize's overlay pattern + vendir's declarative sync + drift detection + multi-format support

#### Key Features

1. **Declarative Configuration**
```yaml
# .configapult.yml
version: 1

sources:
  local-base:
    path: .config-templates/

  company-standards:
    git:
      url: https://github.com/company/configs
      ref: v2.1.0

targets:
  pyproject.toml:
    base: company-standards://python/pyproject.toml
    overlays:
      - local-base://pyproject.overlay.toml
      - inline:
          tool.poetry.name: configapult
          tool.poetry.version: ${VERSION}
    managed_keys:
      - tool.ruff.*
      - tool.mypy.strict
    local_keys:
      - tool.poetry.dependencies
```

2. **Commands**
```bash
configapult sync          # Apply configuration
configapult check         # Validate against sources
configapult diff          # Show what would change
configapult drift         # Show unmanaged changes
configapult update        # Update source versions
```

3. **Merge Strategies**
- Deep merge (default)
- Shallow merge
- Replace
- Array append/prepend
- Custom merge functions

4. **Drift Detection**
- Track "managed" vs "local" keys
- Show unexpected changes
- Allow intentional divergence
- Report on drift across mono-repo

5. **Lock Files**
```yaml
# .configapult.lock
sources:
  company-standards:
    resolved_ref: a1b2c3d4...
    resolved_at: 2025-11-23T10:00:00Z

targets:
  pyproject.toml:
    last_synced: 2025-11-23T10:00:00Z
    checksum: abc123...
    managed_keys: [...]
```

6. **Format Support**
- TOML (via tomlkit)
- YAML (via ruamel.yaml)
- JSON
- Plugin system for others

#### Architecture

**Language:** Python (matches your ecosystem) or Rust (for speed, single binary)

**Libraries:**
- `tomlkit` - TOML with formatting preservation
- `ruamel.yaml` - YAML with formatting preservation
- `click` - Beautiful CLI
- `rich` - Terminal output
- `pydantic` - Config validation

**Distribution:**
- PyPI package
- Optional: compiled binary with PyInstaller
- Pre-commit hook integration

#### Development Phases

**Phase 1: MVP**
- Single file, single source, TOML only
- Basic merge (deep)
- Sync and check commands
- Local sources only

**Phase 2: Multi-Source**
- Git sources with version tracking
- Multiple targets
- Lock files
- YAML support

**Phase 3: Drift Detection**
- Managed vs local keys
- Drift reporting
- Update notifications

**Phase 4: Advanced Features**
- Custom merge strategies
- Mono-repo awareness
- Template support (Jinja2)
- Plugin system

**Phase 5: Polish**
- Great error messages
- Interactive mode
- Documentation site
- CI/CD examples

---

### Option C: Hybrid Approach

Start with existing tools, extract patterns, build focused tool:

1. **Immediate (1-2 weeks):**
   - Use ruff/pyright native hierarchical configs
   - Create custom merge scripts for remaining tools
   - Document patterns in this repo

2. **Short-term (1-2 months):**
   - Generalize merge scripts into reusable module
   - Add drift detection to pre-commit hooks
   - Build CLI wrapper around scripts

3. **Medium-term (3-6 months):**
   - Extract into standalone tool (configapult)
   - Add git source support
   - Build lock file system

4. **Long-term (6+ months):**
   - Polish DX
   - Add advanced features
   - Open source and promote

---

## Implementation Patterns

### Pattern 1: Selective Key Merging

```python
from tomlkit import parse, dumps

def merge_selective(base_doc, overlay_doc, managed_keys):
    """Merge only specified keys from overlay into base."""
    result = base_doc.copy()

    for key_path in managed_keys:
        # key_path like "tool.ruff.lint"
        parts = key_path.split('.')

        # Navigate and copy
        src = overlay_doc
        dst = result
        for part in parts[:-1]:
            src = src.get(part, {})
            if part not in dst:
                dst[part] = {}
            dst = dst[part]

        # Copy final key
        if parts[-1] in src:
            dst[parts[-1]] = src[parts[-1]]

    return result
```

### Pattern 2: Drift Detection

```python
def detect_drift(current_file, expected_config, managed_keys):
    """Compare current file against expected for managed keys."""
    current = parse(current_file.read_text())

    drifts = []
    for key_path in managed_keys:
        current_val = get_nested(current, key_path)
        expected_val = get_nested(expected_config, key_path)

        if current_val != expected_val:
            drifts.append({
                'key': key_path,
                'current': current_val,
                'expected': expected_val,
            })

    return drifts
```

### Pattern 3: Git Source Resolution

```python
import tempfile
from pathlib import Path
import subprocess

def fetch_git_source(url, ref):
    """Fetch config from git at specific ref."""
    with tempfile.TemporaryDirectory() as tmpdir:
        # Clone with depth 1 for speed
        subprocess.run([
            'git', 'clone', '--depth', '1',
            '--branch', ref,
            url, tmpdir
        ])

        # Read config files
        configs = {}
        for config_file in Path(tmpdir).rglob('*.toml'):
            configs[str(config_file.relative_to(tmpdir))] = \
                config_file.read_text()

        return configs
```

---

## Sources

### Tools & Documentation
- [mise (jdx's tool)](https://mise.jdx.dev/)
- [cruft - Cookiecutter template sync](https://cruft.github.io/cruft/)
- [vendir - Declarative directory sync](https://carvel.dev/vendir)
- [ytt - YAML templating](https://carvel.dev/ytt/)
- [templr - Go template rendering](https://github.com/kanopi/templr)
- [kustomize - Kubernetes config management](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [GNU Stow - Symlink manager](https://www.gnu.org/software/stow/)
- [chezmoi - Dotfile manager](https://www.chezmoi.io/)
- [config-merge - File merging tool](https://github.com/boxboat/config-merge)

### Comparisons & Guides
- [Taming the Beast: Jsonnet, Dhall, CUE comparison](https://pv.wtf/posts/taming-the-beast)
- [CUE language comparisons](https://github.com/cue-lang/cue/discussions/669)
- [Python Monorepo Guide - Tweag](https://www.tweag.io/blog/2023-04-04-python-monorepo-1/)
- [Cracking the Python Monorepo](https://gafni.dev/blog/cracking-the-python-monorepo/)
- [Ruff monorepo support](https://github.com/astral-sh/ruff)
- [Pyright config inheritance](https://github.com/microsoft/pyright/issues/4366)

### Configuration Drift
- [Wiz - Configuration Drift Explained](https://www.wiz.io/academy/configuration-drift)
- [Spacelift - Drift Management](https://spacelift.io/blog/drift-management)
- [Snyk - Infrastructure Drift Detection Tools](https://snyk.io/blog/tools-infrastructure-drift-detection/)

### Community Resources
- [mise 2025 roadmap](https://github.com/jdx/mise/discussions/4057)
- [Platform Engineering with Cookiecutter/Cruft](https://john-miller.dev/posts/cookiecutter-with-cruft-for-platform-engineering/)
- [Kustomize patching tutorial](https://glasskube.dev/blog/patching-with-kustomize/)
- [Using GNU Stow for dotfiles](https://systemcrafters.net/managing-your-dotfiles/using-gnu-stow/)
- [sync_pyproject.py example](https://github.com/KyleKing/mdformat-plugin-template/blob/f0f1b37809bfa80c004c21313a4dfe8d3f5127df/sync_pyproject.py#L17)

---

## Next Steps

1. **Decision Point:** Choose Option A (use existing), B (build configapult), or C (hybrid)

2. **If building configapult:**
   - Validate core concept with prototype
   - Start with Phase 1 MVP
   - Get feedback from real usage
   - Iterate on DX

3. **If using existing:**
   - Document chosen patterns
   - Create reusable scripts
   - Set up drift detection
   - Monitor pain points

4. **Either way:**
   - Leverage ruff/pyright native monorepo support where possible
   - Start small, iterate based on actual needs
   - Focus on DX from day one
