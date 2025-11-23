# Option B: Building Configapult - A Custom Configuration Management Tool

**Date:** 2025-11-23
**Purpose:** Deep exploration of building a custom tool for configuration management with focus on format selection, implementation choices, and architecture.

---

## Table of Contents

1. [Configuration Format Analysis](#configuration-format-analysis)
2. [Format Recommendation](#format-recommendation)
3. [Implementation Language Choice](#implementation-language-choice)
4. [Architecture & Key Features](#architecture--key-features)
5. [Dependencies & Technology Stack](#dependencies--technology-stack)
6. [Merge Strategies](#merge-strategies)
7. [Open Questions](#open-questions)
8. [Prototype Roadmap](#prototype-roadmap)
9. [Sources](#sources)

---

## Configuration Format Analysis

### The YAML Problem

You're right to avoid YAML. The problems are well-documented:

- **Indentation errors** are hard to debug
- **Complex spec** with surprising edge cases (Norway problem: `NO` → `false`)
- **Security issues** (arbitrary code execution in some parsers)
- **Type coercion surprises** (`1.0` vs `"1.0"` vs `1`)
- **Difficult to generate programmatically** while maintaining readability

### Format Options Evaluated

#### 1. **TOML** ⭐ Recommended for Config Files

**Pros:**
- [Simple, predictable syntax](https://www.anbowell.com/blog/an-in-depth-comparison-of-json-yaml-and-toml/) - "Tom's Obvious, Minimal Language"
- [No indentation sensitivity](https://martin-ueding.de/posts/json-vs-yaml-vs-toml/) - avoids YAML's main footgun
- [Better error messages](https://dev.to/leapcell/json-vs-yaml-vs-toml-vs-xml-best-data-format-in-2025-5444) than YAML
- Supports comments (unlike JSON)
- [Native Python support](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/) (stdlib `tomllib` for reading, `tomlkit` for writing with formatting)
- [Industry momentum](https://blog.heroku.com/why-buildpacks-use-toml) - used by Rust (Cargo.toml), Python (pyproject.toml), Hugo

**Cons:**
- Less widespread than YAML in cloud-native ecosystem
- [Handles deeply nested data less elegantly](https://leapcell.io/blog/json-yaml-toml-xml-best-choice-2025) than YAML
- Array of tables syntax can be verbose

**Example:**
```toml
# .configapult.toml
version = "1"

[sources.company-standards]
git = "https://github.com/company/configs"
ref = "v2.1.0"

[sources.local-base]
path = ".config-templates/"

[[targets]]
file = "pyproject.toml"
base = "company-standards://python/pyproject.toml"
overlays = ["local-base://pyproject.overlay.toml"]
managed_keys = ["tool.ruff.*", "tool.mypy.strict"]
local_keys = ["tool.poetry.dependencies"]
```

**Verdict:** ✅ **Best choice for .configapult.toml config file**

---

#### 2. **Python** (as configuration)

**Pros:**
- You already know it
- Full programming language (functions, imports, logic)
- Type hints for validation
- Can import shared configuration libraries
- No new syntax to learn

**Cons:**
- Security risk (arbitrary code execution)
- Harder to statically analyze
- Overkill for most configuration needs
- Requires Python runtime to read

**Example:**
```python
# .configapult.py
from configapult.api import Source, Target, GitSource

sources = {
    'company-standards': GitSource(
        url='https://github.com/company/configs',
        ref='v2.1.0',
    ),
    'local-base': Source(path='.config-templates/'),
}

targets = [
    Target(
        file='pyproject.toml',
        base='company-standards://python/pyproject.toml',
        overlays=['local-base://pyproject.overlay.toml'],
        managed_keys=['tool.ruff.*', 'tool.mypy.strict'],
        local_keys=['tool.poetry.dependencies'],
    ),
]
```

**Verdict:** ⚠️ **Good for advanced users, but security concerns make it unsuitable as default**

---

#### 3. **Pkl** (Apple's Configuration Language)

**Pros:**
- [Type-safe with validation](https://pkl-lang.org/index.html) - catch errors before deployment
- [Programmable](https://medium.com/quick-programming/looking-at-apples-new-configuration-language-pkl-869109a3995c) - classes, functions, conditionals, loops
- [Multi-format output](https://configu.com/blog/apple-pkl-code-example-concepts-how-to-get-started/) - generates JSON, YAML, TOML, etc.
- [IDE support](https://www.infoq.com/news/2024/02/apple-pkl-configuration-lang/) - IntelliJ, VS Code, Neovim
- Strong typing prevents configuration errors

**Cons:**
- New language (released Feb 2024) - learning curve
- Requires separate Pkl runtime/compiler
- Smaller ecosystem than alternatives
- Overkill for simple configuration

**Example:**
```pkl
// .configapult.pkl
version = "1"

sources {
  ["company-standards"] {
    git = "https://github.com/company/configs"
    ref = "v2.1.0"
  }
}

targets {
  new {
    file = "pyproject.toml"
    base = "company-standards://python/pyproject.toml"
    managedKeys = List("tool.ruff.*", "tool.mypy.strict")
  }
}
```

**Verdict:** ⚠️ **Interesting but too new/complex for this use case**

---

#### 4. **HCL** (HashiCorp Configuration Language)

**Pros:**
- [Human-readable by design](https://scalr.com/learning-center/the-developers-guide-to-hcl-part-1-introduction/) - influenced by nginx/libucl
- [Declarative](https://developer.hashicorp.com/terraform/language) - focus on desired state
- [Hierarchical and composable](https://hcl.readthedocs.io/en/latest/language_design.html)
- Proven in production (Terraform, Nomad, Vault)
- Better [error messages](https://spacelift.io/blog/hcl-hashicorp-configuration-language) than YAML

**Cons:**
- Requires Go HCL parser library
- Less familiar to Python developers
- Terraform association may feel heavy
- No native Python library (need to shell out or use incomplete ports)

**Example:**
```hcl
# .configapult.hcl
version = "1"

source "company-standards" {
  git = "https://github.com/company/configs"
  ref = "v2.1.0"
}

target {
  file = "pyproject.toml"
  base = "company-standards://python/pyproject.toml"

  managed_keys = [
    "tool.ruff.*",
    "tool.mypy.strict",
  ]
}
```

**Verdict:** ⚠️ **Good design but poor Python ecosystem support**

---

#### 5. **Nickel** (Tweag's Configuration Language)

**Pros:**
- ["JSON with functions"](https://www.tweag.io/blog/2020-10-22-nickel-open-sourcing/) - simple core concept
- [Gradual typing](https://github.com/tweag/nickel/blob/master/RATIONALE.md) - types optional, add as needed
- [Composable via record merging](https://nickel-lang.org/) with metadata
- [Built-in testing](https://github.com/tweag/nickel/releases) (`nickel test` command)
- [Package manager](https://github.com/tweag/nickel/releases) (experimental in v1.11)

**Cons:**
- New language (2020, still maturing)
- Smaller ecosystem
- Requires Nickel runtime (Rust binary)
- Another syntax to learn

**Example:**
```nickel
# .configapult.ncl
{
  version = "1",
  sources = {
    company-standards = {
      git = "https://github.com/company/configs",
      ref = "v2.1.0",
    },
  },
  targets = [
    {
      file = "pyproject.toml",
      base = "company-standards://python/pyproject.toml",
      managed_keys = ["tool.ruff.*", "tool.mypy.strict"],
    },
  ],
}
```

**Verdict:** ⚠️ **Promising but immature**

---

#### 6. **Starlark** (Python Dialect for Configuration)

**Pros:**
- [Python-like syntax](https://laurent.le-brun.eu/blog/an-overview-of-starlark) - familiar to Python devs
- [Deterministic](https://github.com/bazelbuild/starlark) - same input = same output
- [Hermetic](https://lobste.rs/s/erkm24/5_levels_configuration_languages) - no filesystem/network access
- [Safe for untrusted code](https://cirrus-ci.org/guide/programming-tasks/) - can't do harm
- Used in production (Bazel, Buck2, Drone CI)

**Cons:**
- Not Python (subtle differences trip people up)
- Requires embedding Starlark interpreter
- No native TOML generation (would need custom functions)
- More complex than needed for config files

**Example:**
```python
# .configapult.star
version = "1"

sources = {
    "company-standards": {
        "git": "https://github.com/company/configs",
        "ref": "v2.1.0",
    },
}

targets = [
    {
        "file": "pyproject.toml",
        "base": "company-standards://python/pyproject.toml",
        "managed_keys": ["tool.ruff.*"],
    },
]
```

**Verdict:** ⚠️ **Overkill for configuration, but interesting for advanced templating**

---

#### 7. **RON** (Rusty Object Notation)

**Pros:**
- [Rust-syntax inspired](https://github.com/ron-rs/ron) but readable
- [Supports complex types](https://docs.rs/ron) - structs, enums, tuples
- [Trailing commas, comments](http://kvark.github.io/format/data/json/2017/08/09/rusty-object-notation.html)
- [Serde integration](https://crates.io/crates/ron/0.5.1/dependencies) for Rust
- [49M+ downloads](https://lib.rs/crates/ron) - proven in Rust ecosystem

**Cons:**
- Rust-ecosystem only (no good Python parser)
- Unfamiliar syntax to Python developers
- Not widely known outside Rust

**Example:**
```ron
// .configapult.ron
(
    version: "1",
    sources: {
        "company-standards": (
            git: "https://github.com/company/configs",
            ref: "v2.1.0",
        ),
    },
    targets: [
        (
            file: "pyproject.toml",
            base: "company-standards://python/pyproject.toml",
            managed_keys: ["tool.ruff.*"],
        ),
    ],
)
```

**Verdict:** ❌ **Only if building in Rust**

---

### Format Comparison Matrix

| Format | Readability | Python Support | Type Safety | Complexity | Ecosystem |
|--------|-------------|----------------|-------------|------------|-----------|
| **TOML** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Python | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Pkl | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ |
| HCL | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Nickel | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Starlark | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| RON | ⭐⭐⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |

---

## Format Recommendation

### Primary: **TOML**

**Use TOML for the `.configapult.toml` configuration file**

**Reasons:**
1. ✅ No YAML footguns (indentation, type coercion, complexity)
2. ✅ Excellent Python support (stdlib + tomlkit for formatting preservation)
3. ✅ Familiar to Python developers (pyproject.toml, Cargo.toml)
4. ✅ Simple, predictable syntax with good error messages
5. ✅ Supports comments
6. ✅ Growing ecosystem adoption

### Secondary: **Python** (Optional, Advanced Mode)

**Allow `.configapult.py` for power users who need:**
- Dynamic configuration generation
- Complex logic (conditionals, loops)
- Shared configuration libraries
- Programmatic source/target definition

**Safety measures:**
- Disabled by default
- Explicit opt-in (`--allow-python-config`)
- Clear security warnings
- Sandboxed execution (no filesystem access outside project)

### Template Syntax: **Jinja2** (for inline templates)

When users need templating in overlay files:
- Industry standard (widely known)
- Excellent Python support
- Rich filter library
- Your existing ecosystem already uses it (copier uses Jinja2)

**Example:**
```toml
[[targets]]
file = "pyproject.toml"
overlays_inline = """
[tool.poetry]
name = "{{ project_name }}"
version = "{{ version }}"
"""
template_vars = { project_name = "configapult", version = "0.1.0" }
```

---

## Implementation Language Choice

### The Options

1. **Python** - Native ecosystem, rich libraries, slower execution
2. **Rust** - Fast, single binary, harder development
3. **Go** - Good middle ground, easy cross-compilation

### Decision Matrix

| Criteria | Python | Rust | Go |
|----------|--------|------|-----|
| **Development Speed** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **Runtime Speed** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Distribution** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **TOML Support** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Ecosystem Fit** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **Learning Curve** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

### Recommendation: **Python** (with optional Rust future)

**Start with Python because:**

1. **Speed of iteration** - Build MVP in days, not weeks
2. **Rich ecosystem** - tomlkit, ruamel.yaml, jinja2, pydantic all battle-tested
3. **Your expertise** - You know Python deeply
4. **Target audience** - Python developers (pyproject.toml, ruff, mypy)
5. **Pre-commit integration** - Python hooks are standard
6. **Incremental adoption** - Easy to script and integrate

**Distribution strategy:**
- **Phase 1 (MVP):** PyPI package (`pip install configapult`)
- **Phase 2 (Polish):** PyInstaller/PyOxidizer for single binary
- **Phase 3 (Optional):** Rust rewrite if speed becomes issue

**Why not Rust/Go initially:**
- [Single binary distribution](https://stackoverflow.com/questions/46811800/is-it-possible-to-build-a-single-binary-for-cli-python-script-like-go-and-rust) is nice but not critical for MVP
- TOML parsing in Python is more mature (tomlkit preserves formatting)
- Config management is not CPU-bound (I/O and git operations dominate)
- [PyOxidizer](https://github.com/indygreg/PyOxidizer) can produce single binary later

**Rust migration path:**
If tool becomes popular and speed matters:
1. Keep Python as reference implementation
2. Rewrite core in Rust with [PyO3](https://pyo3.rs/v0.15.1/building_and_distribution.html) bindings
3. Use [Maturin](https://medium.com/@MatthieuL49/a-mixed-rust-python-project-24491e2af424) for mixed Python/Rust project
4. Or full Rust rewrite with lessons learned

---

## Architecture & Key Features

### Core Architecture

```
configapult/
├── cli.py              # Typer-based CLI entry point
├── config.py           # .configapult.toml parsing & validation
├── sources/
│   ├── base.py         # Abstract source interface
│   ├── local.py        # Local filesystem source
│   ├── git.py          # Git repository source
│   └── http.py         # HTTP(S) download source
├── targets/
│   ├── base.py         # Abstract target interface
│   ├── toml.py         # TOML file handling (tomlkit)
│   ├── yaml.py         # YAML file handling (ruamel.yaml)
│   └── json.py         # JSON file handling
├── merge/
│   ├── strategies.py   # Merge strategy implementations
│   └── selectors.py    # Key path selection (glob patterns)
├── drift/
│   ├── detector.py     # Drift detection logic
│   └── reporter.py     # Drift reporting (rich tables)
├── lock.py             # Lock file management
└── templates.py        # Jinja2 template rendering
```

### Key Features

#### 1. **Declarative Configuration**

```toml
# .configapult.toml
version = "1"

# Define configuration sources
[sources.company]
git = "https://github.com/company/config-templates"
ref = "v2.1.0"

[sources.local]
path = ".config/"

# Define merge targets
[[targets]]
file = "pyproject.toml"
base = "company://python/base.toml"
overlays = [
    "company://python/ruff-strict.toml",
    "local://pyproject.overlay.toml",
]

# Control what gets synced vs stays local
managed_keys = [
    "tool.ruff.lint.*",
    "tool.ruff.format.*",
    "tool.mypy.strict",
]
local_keys = [
    "tool.poetry.name",
    "tool.poetry.version",
    "tool.poetry.dependencies",
]

# Merge strategy
merge_strategy = "deep"  # deep, shallow, or custom
```

#### 2. **CLI Commands**

```bash
# Initialize new config
configapult init

# Validate configuration
configapult check
# Output:
# ✓ .configapult.toml is valid
# ✓ All sources are accessible
# ✓ All target files exist
# ✗ pyproject.toml has 3 drift issues

# Show what would change (dry-run)
configapult diff
# Output:
# pyproject.toml:
#   tool.ruff.line-length: 88 → 120
#   + tool.ruff.lint.select: ["E", "F", "I"]
#   - tool.ruff.ignore: []

# Apply configuration
configapult sync
# Output:
# Fetching company:v2.1.0... ✓
# Merging pyproject.toml... ✓
# Writing .configapult.lock... ✓

# Detect drift from managed config
configapult drift
# Output:
# Drift detected in 1 file:
#
# pyproject.toml:
#   tool.ruff.line-length
#     Expected: 120
#     Actual: 88
#     Managed by: company://python/ruff-strict.toml

# Update source versions
configapult update [source-name]
# Checks for new versions, updates lock file

# Show configuration status
configapult status
# Output:
# Sources:
#   company: v2.1.0 (latest: v2.2.0) ⚠️
#   local: .config/ ✓
# Targets:
#   pyproject.toml: synced ✓
#   .pylintrc: 1 drift ⚠️
```

#### 3. **Merge Strategies**

Support multiple [merge approaches](https://stackoverflow.com/questions/27936772/how-to-deep-merge-instead-of-shallow-merge):

**Deep Merge (default):**
```python
# Base
tool.ruff.lint.select = ["E", "F"]

# Overlay
tool.ruff.lint.select = ["I"]

# Result (extends)
tool.ruff.lint.select = ["E", "F", "I"]
```

**Shallow Merge:**
```python
# Base
tool.ruff.lint.select = ["E", "F"]

# Overlay
tool.ruff.lint.select = ["I"]

# Result (replaces)
tool.ruff.lint.select = ["I"]
```

**Custom Strategies:**
```toml
[[targets]]
file = "pyproject.toml"
merge_strategy = "deep"

# Per-key override
[targets.merge_overrides]
"tool.ruff.lint.select" = "append"     # Concatenate arrays
"tool.ruff.lint.ignore" = "replace"    # Replace entire value
"tool.poetry.dependencies" = "local"   # Never override
```

#### 4. **Managed vs Local Keys**

Track which configuration is "managed" (from sources) vs "local" (project-specific):

```toml
managed_keys = [
    "tool.ruff.*",        # All ruff config managed
    "tool.mypy.strict",   # Specific mypy setting
]

local_keys = [
    "tool.poetry.*",      # Poetry config stays local
    "tool.mypy.python_version",  # Override allowed
]
```

**Drift detection respects these boundaries:**
- Changes to `managed_keys` → drift warning
- Changes to `local_keys` → ignored (intentional)
- Changes to unlisted keys → info message

#### 5. **Lock Files**

```yaml
# .configapult.lock (YAML for readability, not user-edited)
version: 1
generated_at: 2025-11-23T10:30:00Z

sources:
  company:
    git: https://github.com/company/config-templates
    ref: v2.1.0
    resolved_commit: a1b2c3d4e5f6...
    resolved_at: 2025-11-23T10:30:00Z

targets:
  pyproject.toml:
    last_synced: 2025-11-23T10:30:00Z
    checksum: sha256:abc123...
    applied_sources:
      - company://python/base.toml
      - company://python/ruff-strict.toml
      - local://pyproject.overlay.toml
    managed_keys:
      - tool.ruff.lint.*
      - tool.ruff.format.*
```

#### 6. **Git Source Resolution**

```python
# Fetch from git at specific version
company:v2.1.0://python/base.toml

# Implementation:
1. Clone repo to ~/.cache/configapult/sources/company-a1b2c3
2. Checkout ref v2.1.0
3. Read python/base.toml
4. Cache for reuse
5. Lock resolved commit hash
```

#### 7. **Template Support (Optional)**

```toml
[[targets]]
file = "pyproject.toml"
template = true
template_vars = { project = "configapult", version = "0.1.0" }

# In overlay file:
# [tool.poetry]
# name = "{{ project }}"
# version = "{{ version }}"
```

#### 8. **Selective Key Merging with Glob Patterns**

```toml
managed_keys = [
    "tool.ruff.*",           # All of tool.ruff
    "tool.mypy.strict",      # Specific key
    "tool.*.check_untyped_defs",  # Pattern across tools
]
```

Implementation:
```python
import fnmatch

def key_matches_pattern(key_path: str, pattern: str) -> bool:
    """Check if key path matches glob pattern."""
    return fnmatch.fnmatch(key_path, pattern)
```

---

## Dependencies & Technology Stack

### Core Dependencies

```toml
[tool.poetry.dependencies]
python = "^3.9"

# CLI framework
typer = "^0.12.0"      # Click-based with type hints
rich = "^13.7.0"       # Beautiful terminal output

# Config parsing
tomlkit = "^0.12.0"    # TOML with formatting preservation
pydantic = "^2.5.0"    # Config validation

# File formats (optional, installed on demand)
ruamel-yaml = "^0.18.0"  # YAML with formatting preservation

# Templating (optional)
jinja2 = "^3.1.0"      # Template engine

# Git operations
gitpython = "^3.1.0"   # Git repository operations

# HTTP downloads
httpx = "^0.27.0"      # Async HTTP client

# Utilities
python-dotenv = "^1.0.0"  # Environment variables
platformdirs = "^4.0.0"   # Cross-platform cache dirs
```

### Dev Dependencies

```toml
[tool.poetry.group.dev.dependencies]
pytest = "^8.0.0"
pytest-cov = "^4.1.0"
mypy = "^1.8.0"
ruff = "^0.2.0"
pre-commit = "^3.6.0"

# Testing fixtures
pytest-mock = "^3.12.0"
pytest-asyncio = "^0.23.0"

# Build & distribution
build = "^1.0.0"
twine = "^5.0.0"
```

### Why These Choices?

**[Typer over Click](https://typer.tiangolo.com/alternatives/):**
- [Built on Click](https://johal.in/click-vs-typer-comparison-choosing-cli-frameworks-for-python-application-distribution/) but simpler with type hints
- [Auto-completion support](https://medium.com/@mohd_nass/navigating-the-cli-landscape-in-python-a-comparative-study-of-argparse-click-and-typer-480ebbb7172f)
- [Rich integration](https://python.plainenglish.io/building-command-line-tools-in-python-click-vs-argparse-vs-typer-514442c25a56) for beautiful output
- Modern, pythonic API

**[Rich](https://ewels.github.io/rich-click/documentation/introduction_to_click/):**
- Gorgeous terminal output
- Tables for drift reports
- Progress bars for sync operations
- Syntax highlighting for diffs

**tomlkit over tomllib:**
- stdlib tomllib (Python 3.11+) is read-only
- tomlkit preserves formatting, comments, and structure
- Critical for config files users hand-edit

**Pydantic:**
- Validation of .configapult.toml structure
- Type-safe configuration models
- Great error messages

**GitPython over subprocess:**
- Higher-level API
- Cross-platform
- Better error handling

---

## Merge Strategies

### Strategy Implementations

Based on [research into merge patterns](https://docs.gruntwork.io/guides/stay-up-to-date/terraform/how-to-dry-your-reference-architecture/deployment-walkthrough/optional-even-dryer-configuration/):

```python
from typing import Any, Dict
from enum import Enum

class MergeStrategy(str, Enum):
    DEEP = "deep"
    SHALLOW = "shallow"
    REPLACE = "replace"
    APPEND = "append"
    PREPEND = "prepend"
    LOCAL = "local"

def merge(base: Dict, overlay: Dict, strategy: MergeStrategy) -> Dict:
    """Merge overlay into base using specified strategy."""

    if strategy == MergeStrategy.REPLACE:
        # Complete replacement
        return overlay.copy()

    elif strategy == MergeStrategy.SHALLOW:
        # Top-level merge only
        result = base.copy()
        result.update(overlay)
        return result

    elif strategy == MergeStrategy.DEEP:
        # Recursive merge
        return deep_merge(base, overlay)

    elif strategy == MergeStrategy.LOCAL:
        # Always prefer base (local wins)
        return base.copy()

    # ... more strategies

def deep_merge(base: Dict, overlay: Dict) -> Dict:
    """
    Recursively merge overlay into base.

    Rules:
    - Scalars: overlay wins
    - Lists: concatenate (configurable)
    - Dicts: recurse
    """
    result = base.copy()

    for key, overlay_value in overlay.items():
        if key not in result:
            # New key, just add
            result[key] = overlay_value
        else:
            base_value = result[key]

            if isinstance(base_value, dict) and isinstance(overlay_value, dict):
                # Both dicts, recurse
                result[key] = deep_merge(base_value, overlay_value)
            elif isinstance(base_value, list) and isinstance(overlay_value, list):
                # Both lists, concatenate (default) or replace
                result[key] = base_value + overlay_value
            else:
                # Scalar or type mismatch, overlay wins
                result[key] = overlay_value

    return result
```

### Per-Key Strategy Override

```toml
[[targets]]
file = "pyproject.toml"
merge_strategy = "deep"  # Default for all keys

[targets.merge_overrides]
# Override specific keys
"tool.ruff.lint.select" = "append"      # Add to array
"tool.ruff.lint.ignore" = "replace"     # Replace array
"tool.poetry.dependencies" = "local"    # Never touch
"tool.mypy" = "shallow"                 # Shallow merge this section
```

---

## Open Questions

### 1. **Mono-repo Support Design**

**Question:** How should configapult handle multiple packages in a mono-repo?

**Options:**

**A) Multiple config files (one per package)**
```
mono-repo/
├── packages/
│   ├── pkg-a/
│   │   ├── .configapult.toml
│   │   └── pyproject.toml
│   └── pkg-b/
│       ├── .configapult.toml
│       └── pyproject.toml
```
- ✅ Simple, isolated configs
- ❌ Duplication across packages

**B) Single config file with multiple targets**
```toml
# .configapult.toml (at root)
[[targets]]
file = "packages/*/pyproject.toml"  # Glob pattern
base = "company://python/base.toml"

[[targets]]
file = "packages/pkg-a/pyproject.toml"
overlays = ["local://pkg-a-overrides.toml"]
```
- ✅ Centralized configuration
- ✅ DRY - reuse sources
- ❌ More complex

**C) Hierarchical configs (inherit from parent)**
```
mono-repo/
├── .configapult.toml       # Base config
└── packages/
    ├── pkg-a/
    │   └── .configapult.toml  # Inherits + extends
    └── pkg-b/
        └── .configapult.toml
```
- ✅ Balance of DRY and isolation
- ✅ Packages can override
- ❌ Most complex implementation

**Recommendation:** Start with **B** (single config with globs), add **C** (inheritance) later if needed.

---

### 2. **Drift Remediation**

**Question:** Should `configapult drift` just report, or also fix drift?

**Options:**

**A) Report only (recommended for MVP)**
```bash
configapult drift
# Reports drift, user runs `configapult sync` to fix
```
- ✅ Safe, explicit
- ✅ User controls when changes happen
- ❌ Extra step

**B) Report with optional fix**
```bash
configapult drift              # Report only
configapult drift --fix        # Fix automatically
configapult drift --interactive  # Ask for each change
```
- ✅ Flexible
- ⚠️ --fix could surprise users

**C) Auto-fix is default behavior**
```bash
configapult sync  # Always syncs (current design)
```
- ❌ Unexpected changes

**Recommendation:** **A** for MVP, add **B** in v0.2.0

---

### 3. **Version Constraints**

**Question:** Should sources support version ranges or only exact refs?

**Options:**

**A) Exact refs only (simpler)**
```toml
[sources.company]
git = "https://github.com/company/configs"
ref = "v2.1.0"  # Exact tag/commit
```
- ✅ Reproducible
- ✅ Lock file is simpler
- ❌ Manual updates

**B) Version ranges**
```toml
[sources.company]
git = "https://github.com/company/configs"
version = "^2.1.0"  # Semver range
```
- ✅ Auto-update to compatible versions
- ❌ Less reproducible
- ❌ Need semver parsing

**Recommendation:** **A** (exact refs) for MVP. `configapult update` can help with updates.

---

### 4. **Overlay Order Semantics**

**Question:** How should overlay order work?

**Current design:**
```toml
overlays = [
    "company://python/base.toml",
    "company://python/ruff-strict.toml",
    "local://overrides.toml",
]
# Applied left-to-right (later wins)
```

**Alternative: Priority levels**
```toml
overlays = [
    { source = "company://python/base.toml", priority = 10 },
    { source = "local://overrides.toml", priority = 100 },
]
# Higher priority wins, not order-dependent
```

**Recommendation:** Keep order-based (simpler, more predictable).

---

### 5. **Cache Management**

**Question:** How should git sources be cached?

**Design:**
```
~/.cache/configapult/
└── sources/
    └── company-a1b2c3d4/  # Hash of (url + ref)
        └── ...cloned repo...
```

**Cache invalidation:**
- Lock file tracks resolved commit
- If ref changes, re-clone
- `configapult cache clean` to purge

**TTL for branches:**
- Tags/commits: cache forever
- Branches (main, develop): TTL of 1 hour?

**Open question:** Should we support local cache dir override for corporate proxies?

---

### 6. **Template Variable Sources**

**Question:** Where should template variables come from?

**Options:**

**A) Inline in config**
```toml
[[targets]]
template_vars = { project = "foo", version = "1.0" }
```

**B) From environment**
```toml
[[targets]]
template_vars_env = ["PROJECT_NAME", "VERSION"]
```

**C) From file**
```toml
[[targets]]
template_vars_file = "vars.toml"
```

**D) All of the above (merge)**
```toml
[[targets]]
template_vars = { default = "value" }
template_vars_file = "vars.toml"  # Overrides inline
template_vars_env = ["VAR"]        # Overrides file
```

**Recommendation:** **D** (all sources, env wins, then file, then inline)

---

### 7. **CI/CD Integration**

**Question:** How should this integrate with CI/CD?

**Features needed:**
- `configapult check --strict` → exit 1 if drift detected
- `configapult diff --json` → machine-readable output
- Pre-commit hook → run check on commit

**Open questions:**
- Should we auto-commit changes? (No, too magical)
- Should we create PRs for updates? (Maybe via GitHub Action)
- Should we support `--dry-run` on all commands? (Yes)

---

### 8. **Selective Sync**

**Question:** Should users be able to sync specific targets?

```bash
configapult sync pyproject.toml  # Only sync this file
configapult sync --all           # Sync everything (default)
```

**Use case:** "I only want to update ruff config, leave pyright alone"

**Recommendation:** Yes, add target selection to all commands.

---

### 9. **Config File Discovery**

**Question:** Where should configapult look for config?

**Search path:**
1. `./.configapult.toml` (current dir)
2. `./.configapult/config.toml` (dedicated dir)
3. `./pyproject.toml` (embedded in `[tool.configapult]`)
4. Walk up to git root
5. `~/.config/configapult/config.toml` (global defaults)

**Recommendation:** Support 1, 3, and 5 initially.

---

### 10. **Error Handling Philosophy**

**Question:** How strict should validation be?

**Options:**

**A) Fail fast (strict)**
- Missing source → error
- Invalid config → error
- Drift detected → error (with --strict flag)

**B) Warn and continue**
- Missing source → warning, skip
- Invalid config → warning, use defaults
- Drift detected → info

**Recommendation:** **A** (fail fast) by default, but add `--lenient` flag for CI environments that need it.

---

## Prototype Roadmap

### Phase 0: Validation (1 week)

**Goal:** Prove core concept with minimal code

**Deliverables:**
- [ ] Spike: Parse TOML with tomlkit
- [ ] Spike: Merge two TOML files with deep merge
- [ ] Spike: Detect changes between expected and actual
- [ ] Decision: Commit to building this?

### Phase 1: MVP (2-3 weeks)

**Goal:** Working tool for single use case

**Features:**
- [ ] Parse `.configapult.toml` (local sources only)
- [ ] TOML target support only
- [ ] Deep merge strategy
- [ ] `configapult sync` command
- [ ] `configapult check` command
- [ ] Basic CLI with Typer + Rich
- [ ] Unit tests for merge logic

**Example config:**
```toml
version = "1"

[sources.local]
path = ".config-templates/"

[[targets]]
file = "pyproject.toml"
base = "local://base.toml"
overlays = ["local://ruff.toml"]
```

**Success criteria:**
- Can sync pyproject.toml from local templates
- Can detect drift
- Clear error messages

### Phase 2: Git Sources (1-2 weeks)

**Goal:** Support remote configuration sources

**Features:**
- [ ] Git source type
- [ ] Clone to cache directory
- [ ] Lock file generation
- [ ] `configapult update` command
- [ ] Resolved commit tracking

**Example config:**
```toml
[sources.company]
git = "https://github.com/company/configs"
ref = "v2.1.0"

[[targets]]
file = "pyproject.toml"
base = "company://python/base.toml"
```

### Phase 3: Managed Keys (1 week)

**Goal:** Track which config is managed vs local

**Features:**
- [ ] `managed_keys` configuration
- [ ] `local_keys` configuration
- [ ] Drift detection respects boundaries
- [ ] `configapult drift` command

**Example:**
```toml
[[targets]]
file = "pyproject.toml"
managed_keys = ["tool.ruff.*"]
local_keys = ["tool.poetry.dependencies"]
```

### Phase 4: Multi-Format Support (1 week)

**Goal:** Support YAML and JSON targets

**Features:**
- [ ] YAML target handler (ruamel.yaml)
- [ ] JSON target handler
- [ ] Auto-detect format from extension
- [ ] Format-specific merge strategies

### Phase 5: Polish & DX (2 weeks)

**Goal:** Make it delightful to use

**Features:**
- [ ] `configapult init` wizard
- [ ] `configapult diff` with syntax highlighting
- [ ] `configapult status` dashboard
- [ ] Rich progress bars
- [ ] Beautiful error messages
- [ ] Shell completion
- [ ] Pre-commit hook template
- [ ] Documentation site

### Phase 6: Advanced Features (2-3 weeks)

**Goal:** Power user features

**Features:**
- [ ] Custom merge strategies
- [ ] Template support (Jinja2)
- [ ] Mono-repo glob patterns
- [ ] Python config file support (`.configapult.py`)
- [ ] HTTP(S) sources
- [ ] Per-key merge overrides

### Phase 7: Distribution (1 week)

**Goal:** Easy installation

**Features:**
- [ ] PyPI package
- [ ] Documentation site (MkDocs)
- [ ] GitHub Actions examples
- [ ] Pre-commit hook published
- [ ] Optional: PyInstaller single binary

**Total time estimate:** 11-15 weeks for full-featured v1.0

---

## Sources

### Configuration Formats
- [TOML vs YAML vs JSON comparison](https://www.anbowell.com/blog/an-in-depth-comparison-of-json-yaml-and-toml/)
- [Why buildpacks use TOML](https://blog.heroku.com/why-buildpacks-use-toml)
- [Apple Pkl configuration language](https://pkl-lang.org/index.html)
- [Pkl introduction](https://medium.com/quick-programming/looking-at-apples-new-configuration-language-pkl-869109a3995c)
- [HCL design philosophy](https://scalr.com/learning-center/the-developers-guide-to-hcl-part-1-introduction/)
- [Terraform HCL documentation](https://developer.hashicorp.com/terraform/language)
- [Nickel configuration language](https://nickel-lang.org/)
- [Nickel rationale](https://github.com/tweag/nickel/blob/master/RATIONALE.md)
- [Starlark overview](https://laurent.le-brun.eu/blog/an-overview-of-starlark)
- [RON Rusty Object Notation](https://github.com/ron-rs/ron)

### Python Ecosystem
- [Python packaging pyproject.toml guide](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/)
- [Poetry pyproject.toml docs](https://python-poetry.org/docs/pyproject/)

### CLI Frameworks
- [Click vs Typer comparison](https://johal.in/click-vs-typer-comparison-choosing-cli-frameworks-for-python-application-distribution/)
- [Typer documentation](https://typer.tiangolo.com/)
- [Python CLI tools comparison](https://medium.com/@mohd_nass/navigating-the-cli-landscape-in-python-a-comparative-study-of-argparse-click-and-typer-480ebbb7172f)

### Merge Strategies
- [Deep vs shallow merge](https://stackoverflow.com/questions/27936772/how-to-deep-merge-instead-of-shallow-merge)
- [Terraform deepmerge](https://github.com/isometry/terraform-provider-deepmerge)
- [Puppet Hiera merging](https://www.puppet.com/docs/puppet/7/hiera_merging.html)

### Distribution
- [Python single binary distribution](https://stackoverflow.com/questions/46811800/is-it-possible-to-build-a-single-binary-for-cli-python-script-like-go-and-rust)
- [PyOxidizer](https://github.com/indygreg/PyOxidizer)
- [PyO3 building and distribution](https://pyo3.rs/v0.15.1/building_and_distribution.html)
- [Mixed Rust Python projects](https://medium.com/@MatthieuL49/a-mixed-rust-python-project-24491e2af424)

---

## Next Steps

1. **Review this document** - Does this vision align with your needs?
2. **Decide on scope** - MVP only? Or commit to full roadmap?
3. **Start Phase 0** - Validate with spike code
4. **Get feedback** - Share with potential users early

**Key decision points:**
- ✅ TOML for config format
- ✅ Python for implementation
- ✅ Typer + Rich for CLI
- ⚠️ Mono-repo strategy needs validation
- ⚠️ Some open questions need user research

Ready to start prototyping? 🚀
