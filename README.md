# fkzys Ecosystem Specification

This document defines **ecosystem-wide conventions** for all projects in the fkzys ecosystem. It serves two purposes:

1. **Human-readable specification** — reference for developers working across multiple projects
2. **Machine-consumable context** — provides LLM assistants with the rules and patterns needed to generate code that conforms to ecosystem standards

When generating code for any fkzys project, assistants must follow the rules defined in this specification. They are derived from real projects, not generic best practices.

## Status

| Attribute | Value |
|-----------|-------|
| Scope | Ecosystem-wide (all fkzys projects) |
| Authority | Definitive — overrides project-local conventions where conflicts exist |
| Last updated | 2026-04-09 |
| License | CC BY-SA 4.0 (text) / AGPL-3.0-or-later (code patterns) — see §14 |

## How to Read This Document

- **Sections 0–10, 13, 15** — Technical specification: patterns, conventions, code structures, CI
- **Section 11** — Code generation protocol: rules for assistants generating code in this ecosystem
- **Section 12** — Quick reference: common snippets and commands
- **Section 14** — Licensing

Project-specific information lives in the **repository's own documentation**:
- `README.md` — build, install, dependencies, configuration
- `CHANGELOG.md` — version history
- `TODO.md` — planned work
- `tests.md` — test suite documentation (per-project, but follows ecosystem testing patterns)

## 0. SCOPE & APPLICABILITY

This specification defines **ecosystem-wide patterns** — conventions that apply to all projects in the fkzys ecosystem, regardless of language or purpose.

### What belongs here
- Universal patterns: error handling, config parsing, test structure, Makefile conventions
- Security rules: `verify-lib`, whitelist config, no `eval`, ownership checks
- Testing philosophy: isolation, temp dirs, mock frameworks, root-only guards
- Build/install conventions: `PREFIX=/usr`, `DESTDIR=`, license paths

### What does NOT belong here
- Project-specific dependencies (e.g. GTK4, ffmpeg, xUnit)
- UI toolkit migration details
- Domain logic (subtitle parsing, video processing, Anki integration)
- Changelog entries, TODO lists, screenshots
- Infrastructure-as-code deployment patterns (Jinja2 services, SOPS secrets, Terraform) — covered in §9

### What is NOT a package

The following project types are **infrastructure or data**, not installable packages. Conventions like `Makefile` (PREFIX/DESTDIR), `depends`, `bin/`, `lib/`, `tests/README.md`, and `verify-lib` do not apply to them:

| Type | Examples | What applies |
|------|----------|--------------|
| Infrastructure-as-code | `infra`, `tf-infra` | §4 (Python patterns), §9 (SOPS, Jinja2), README |
| Container images | `sing-box` | Dockerfile conventions, CI validation |
| Dotfiles / config repos | `root_m`, `dotfiles` | dotm patterns (`dotm.toml`, `perms`, `.sops.yaml`) |
| Data / rule sets | `sing-box_srs` | `build.sh` with `set -euo pipefail` |
| Profiles / documentation | `gitlab-profile`, `packages` | README only |

### 0.1 RULE PRIORITY
When rules conflict, apply in this order:
1. Security (`verify-lib`, ownership checks, no `eval`, no `/tmp` for scripts, `SecureDir`)
2. Correctness (`set -euo pipefail`, error handling, config validation)
3. Consistency (naming, structure, Makefile targets)
4. Convenience (shortcuts, defaults)

### 0.2 AGENT BEHAVIOR
These rules apply to LLM assistants interacting with the ecosystem.

1. **Verify before concluding.** Never assume system behavior based on theory. Check logs, process trees, file contents (hex if needed), and registry lookups before making claims about why something does or doesn't work.
2. **No system modifications without request.** Never execute or suggest `sudo`, `rm /...`, `systemctl restart`, `make install`, or package removal without explicit user request. Describe options — user decides.
3. **No secret leakage.** Tests, documentation, and examples must never contain real secret values or secret-derived data. Use abstract identifiers (`"value"`, `"fallback"`, `"app"`, `"setting"`).
4. **No fabricated credentials or signatures.** Never generate `Signed-off-by`, `git config user.*`, or credential-like values without user-provided context.
5. **Ask when uncertain.** About paths, versions, flags, or ambiguous instructions before generating code or executing commands.

## 1. REPOSITORY STRUCTURE

### Shell / Python / C Projects
| Path | Purpose |
|------|---------|
| `bin/` | Entry points: shell scripts, compiled binaries |
| `lib/` | Shared libraries: `common.sh`, Python modules, C code |
| `etc/` | Default configuration files (installed to `/etc/`) |
| `completions/` | `_cmd` (zsh), `cmd.bash` (bash) |
| `man/` | Markdown sources (`.md`) + compiled roff (`.8`/`.5`) |
| `tests/` | Test suite. If tests are present, SHOULD include `tests/README.md` with table and instructions |
| `depends` | Dependencies. Format: `system:pkg` or `gitpkg:pkg`. `#` for comments |
| `Makefile` | Targets: `build` (if needed), `install`, `uninstall`, `clean`, `test`, `man` |
| `backup/` | Migration/backup scripts |
| `hooks/` | Pacman/systemd hooks |
| `systemd/` | Service units: `user/` or `system/` |
| `extras/` | Additional scripts (wrappers, helpers) |

### Go Projects
| Path | Purpose |
|------|---------|
| `cmd/<name>/` | Entry point (`main.go`, `version.go`) |
| `internal/` | Private packages (`config`, `engine`, `tmpl`, `safetemp`, etc.) |
| `tests.md` | Test documentation (instead of `tests/README.md`) |
| `go.mod` / `go.sum` | Module definition and dependencies |
| `Makefile` | `build`, `install`, `uninstall`, `test`, `test-root`, `clean` |

### C# / .NET Projects
| Path | Purpose |
|------|---------|
| `<ProjectName>/` | Main application source (`.cs` files) |
| `<ProjectName>.Tests/` | xUnit test project (`*_Tests.cs`) |
| `<ProjectName>.csproj` | SDK-style project file (`net10.0`, `AllowUnsafeBlocks`, etc.) |
| `<ProjectName>.Tests.csproj` | Test project, references main project |
| `tests.md` | Test documentation (instead of `tests/README.md`) |
| `Makefile` | `build`, `install`, `uninstall`, `test`, `clean` |

## 2. SHELL (BASH)

### Header & Strictness
```bash
#!/bin/bash
# /usr/bin/project-name
#
# Utility description.
#
set -euo pipefail
```
- System scripts: `#!/bin/bash`
- Libraries: `#!/usr/bin/env bash`
- If a file is both entry point and library (rare), use `#!/bin/bash`.

**Exceptions (no `set -euo pipefail` required):**
- **Test files** — use `set -uo pipefail` (no `-e`). Tests must continue executing when assertions fail so failures can be counted and reported. See §7 for test harness patterns.
- **Wrapper scripts** — thin scripts that `cd` to a config directory and `exec` a binary (e.g. `dist/subs2srs.sh`). These are simple launchers, not business logic. If the `exec` fails, there is nothing left to do.
- **Intentional guard scripts** — scripts that rely on conditional control flow incompatible with `errexit`. Must include a comment explaining the omission.

### CLI Conventions
All bash entry points MUST support:

- **`-V` / `--version`** — Print program name and version, then exit 0
- **`--help` / `-h`** — Print usage information, then exit 0

Both are mandatory for all user-facing shell scripts. The project must also provide:
- **man page** (`man/project.8.md`) — compiled via the Makefile `man` target
- **shell completions** (`completions/_project` for zsh, `completions/project.bash` for bash) — see §8

#### Version Function

The `version` function lives in `lib/common.sh`, if that file is present in the project. The entry point sources `common.sh` and delegates to it:

```bash
case "${1:-}" in
    -V|--version) cmd_version; exit 0 ;;
    -h|--help)    print_usage; exit 0 ;;
    *)            main "$@" ;;
esac
```

This keeps version output consistent across all projects and centralises the source of truth in a single library file.

### Secure Library Sourcing
```bash
readonly LIBDIR="/usr/lib/project"
_src() { local p; p=$(verify-lib "$1" "$LIBDIR/") && source "$p" || exit 1; }
_src "${LIBDIR}/common.sh"
```

### Error Handling
```bash
echo "ERROR: Description of the error" >&2; exit 1
```
- All errors go to stderr
- Prefix `ERROR:` for fatal, `WARN:` for non-fatal
- `exit 1` on fatal errors

### Config Parsing (whitelist-based, NO eval)
```bash
load_config() {
    [[ -f "$CONFIG_FILE" ]] || return 0

    # Verify ownership
    local owner
    owner=$(stat -c %u "$CONFIG_FILE" 2>/dev/null)
    if [[ "$owner" != "0" ]]; then
        echo "ERROR: $CONFIG_FILE not owned by root (owner uid: $owner)" >&2
        return 1
    fi

    local -a allowed=(KEY1 KEY2 KEY3)

    while IFS='=' read -r key value; do
        key="${key#"${key%%[![:space:]]*}"}"
        key="${key%"${key##*[![:space:]]}"}"
        value="${value#"${value%%[![:space:]]*}"}"
        value="${value%"${value##*[![:space:]]}"}"
        value="${value%% #*}"
        value="${value%"${value##*[![:space:]]}"}"
        [[ "$key" =~ ^#.*$ || -z "$key" ]] && continue

        local valid=0
        for a in "${allowed[@]}"; do
            [[ "$key" == "$a" ]] && { valid=1; break; }
        done

        if [[ $valid -eq 1 ]]; then
            value="${value#\"}"; value="${value%\"}"
            value="${value#\'}"; value="${value%\'}"
            printf -v "$key" '%s' "$value"
        else
            echo "WARN: Unknown config key ignored: $key" >&2
        fi
    done < "$CONFIG_FILE"
}
```

### Cleanup Trap
```bash
cleanup() {
    local exit_code=$?
    set +e
    [[ $exit_code -ne 0 ]] && echo ":: cleaning up..."
    # ... cleanup logic ...
    return $exit_code
}
trap cleanup EXIT
```

### Variable Validation
```bash
[[ -n "${VAR:-}" ]] || { echo "ERROR: VAR not defined" >&2; exit 1; }
```

### Nameref for Arrays (bwrap-common pattern)
```bash
bwrap_base() {
    local -n _arr=$1
    _arr+=(--ro-bind /usr /usr --proc /proc)
}
```

### Preserve/Restore shopt (NO eval)
```bash
# Save state explicitly
local _had_nullglob=false
shopt -q nullglob && _had_nullglob=true

shopt -s nullglob
# ... glob operations ...

# Restore explicitly
if $_had_nullglob; then
    shopt -s nullglob
else
    shopt -u nullglob
fi
```

### Input Validation
```bash
if [[ ! "$INPUT" =~ ^[a-zA-Z0-9_-]+$ ]]; then
    echo "ERROR: Invalid input" >&2; exit 1
fi
```

## 3. MAKEFILE

### Shell / Python / C Base Structure
```makefile
.PHONY: install uninstall clean test man

PREFIX     = /usr
SYSCONFDIR = /etc
DESTDIR    =
pkgname    = project-name

BINDIR       = $(PREFIX)/bin
LIBDIR       = $(PREFIX)/lib/project
SHAREDIR     = $(PREFIX)/share
MANDIR       = $(SHAREDIR)/man
ZSH_COMPDIR  = $(SHAREDIR)/zsh/site-functions
BASH_COMPDIR = $(SHAREDIR)/bash-completion/completions
LICENSEDIR   = $(SHAREDIR)/licenses/$(pkgname)

MANPAGES = man/project.8

man: $(MANPAGES)

man/%.8: man/%.8.md
	pandoc -s -t man -o $@ $<

clean:
	rm -f $(MANPAGES)

test:
	bash tests/test.sh

install:
	install -Dm755 bin/project $(DESTDIR)$(BINDIR)/project
	install -Dm644 lib/common.sh $(DESTDIR)$(LIBDIR)/common.sh
	install -Dm644 completions/_project $(DESTDIR)$(ZSH_COMPDIR)/_project
	install -Dm644 completions/project.bash $(DESTDIR)$(BASH_COMPDIR)/project
	install -Dm644 man/project.8 $(DESTDIR)$(MANDIR)/man8/project.8
	install -Dm644 LICENSE $(DESTDIR)$(LICENSEDIR)/LICENSE

	@if [ ! -f "$(DESTDIR)$(SYSCONFDIR)/project.conf" ]; then \
		install -Dm644 etc/project.conf "$(DESTDIR)$(SYSCONFDIR)/project.conf"; \
		echo "Installed default config"; \
	else \
		echo "Config exists, skipping (see etc/project.conf for defaults)"; \
	fi

uninstall:
	rm -f $(DESTDIR)$(BINDIR)/project
	rm -rf $(DESTDIR)$(LIBDIR)/
	rm -f $(DESTDIR)$(ZSH_COMPDIR)/_project
	rm -f $(DESTDIR)$(BASH_COMPDIR)/project
	rm -f $(DESTDIR)$(MANDIR)/man8/project.8
	rm -rf $(DESTDIR)$(LICENSEDIR)/
	@echo "Note: $(SYSCONFDIR)/project.conf preserved. Remove manually if needed."
```

### Go Makefile
```makefile
.PHONY: build install uninstall test test-root clean

PREFIX   = /usr
DESTDIR  =
pkgname  = project-name

BINDIR     = $(PREFIX)/bin
LICENSEDIR = $(PREFIX)/share/licenses/$(pkgname)

VERSION ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)

BINARY = project-name

build:
	CGO_ENABLED=0 go build -trimpath -buildmode=pie -ldflags "-X main.version=$(VERSION)" -o $(BINARY) ./cmd/project-name/

test:
	go test ./...

test-root:
	sudo go test ./internal/perms/ -v -count=1

clean:
	rm -f $(BINARY)

install: build
	install -Dm755 $(BINARY) $(DESTDIR)$(BINDIR)/$(BINARY)
	install -Dm644 LICENSE   $(DESTDIR)$(LICENSEDIR)/LICENSE

uninstall:
	rm -f  $(DESTDIR)$(BINDIR)/$(BINARY)
	rm -rf $(DESTDIR)$(LICENSEDIR)/
```

### C# / .NET Makefile
```makefile
.PHONY: build install uninstall test clean

PREFIX   = /usr
DESTDIR  =
pkgname  = project-name

BINDIR     = $(PREFIX)/bin
LIBDIR     = $(PREFIX)/lib/$(pkgname)
LICENSEDIR = $(PREFIX)/share/licenses/$(pkgname)

PROJECT = project-name
TESTS   = $(PROJECT).Tests

build:
	dotnet build $(PROJECT)/$(PROJECT).csproj -c Release

test:
	dotnet test $(TESTS)/$(TESTS).csproj --no-build -v normal

clean:
	dotnet clean $(PROJECT)/$(PROJECT).csproj
	rm -rf $(PROJECT)/bin $(PROJECT)/obj
	rm -rf $(TESTS)/bin $(TESTS)/obj

install: build
	install -Dm755 $(PROJECT)/bin/Release/net10.0/$(PROJECT) $(DESTDIR)$(BINDIR)/$(PROJECT)
	install -Dm644 LICENSE $(DESTDIR)$(LICENSEDIR)/LICENSE

uninstall:
	rm -f $(DESTDIR)$(BINDIR)/$(PROJECT)
	rm -rf $(DESTDIR)$(LICENSEDIR)/
```

### Key Conventions
- `PREFIX = /usr` (not `/usr/local`)
- `DESTDIR =` (empty by default)
- Config is never overwritten if it already exists
- `man` target generates from `.md` via `pandoc`
- `test` target SHOULD be present if the project contains tests. If the project has no test suite, the target may be omitted. Language-specific: `bash tests/test.sh` (shell), `go test ./...` (Go), `pytest` (Python), `dotnet test` (C#).
- `clean` target MUST undo build artifacts, if any are present. For projects with no build step (e.g. pure shell libraries, install-only packages), it may be omitted or be a no-op.
- The Makefile MUST install the project `LICENSE` to `$(SHAREDIR)/licenses/$(pkgname)/LICENSE`.
- Go: `CGO_ENABLED=0`, `-trimpath`, `-buildmode=pie`, version via `-ldflags`
- C#: `dotnet build -c Release`, `dotnet test --no-build`

## 4. PYTHON

### Entry Point
```python
#!/usr/bin/env python3
"""
CLI entry point for project-name.
"""

from __future__ import annotations

import argparse
import sys

def main() -> None:
    parser = argparse.ArgumentParser(prog="project-name")
    parser.add_argument("--version", action="version", version="%(prog)s 0.1.0")
    args = parser.parse_args()
    # ... logic ...

if __name__ == "__main__":
    main()
```

### `__main__.py`
```python
"""Allow running as `python -m project_name`."""

from .cli import main

main()
```

### Error Handling
```python
print(f"Error: description of the error", file=sys.stderr)
sys.exit(1)
```

### Tests (class-based assertion groups, compatible with pytest runner)
```python
"""Tests for project_name/module.py — pure parsing functions."""

import os
import sys

sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..'))
from project_name.module import function_under_test


class TestFunctionUnderTest:
    def test_normal_case(self):
        assert function_under_test("input") == "expected"

    def test_edge_case(self):
        assert function_under_test("") is None
```

### SOPS Helper (Infrastructure)
```python
import subprocess
import sys
from pathlib import Path
import yaml


def decrypt_sops(file_path: Path) -> dict:
    try:
        result = subprocess.run(
            ['sops', '-d', str(file_path)],
            capture_output=True, text=True, check=True
        )
        return yaml.safe_load(result.stdout)
    except subprocess.CalledProcessError as e:
        print(f"SOPS decryption error: {e.stderr}", file=sys.stderr)
        sys.exit(1)
    except FileNotFoundError:
        print("sops not found in PATH", file=sys.stderr)
        sys.exit(1)
```

## 5. C

### Style & Flags
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <limits.h>
#include <errno.h>
```

### Makefile
```makefile
PREFIX  = /usr
DESTDIR =

CC     ?= cc
CFLAGS ?= -O2 -Wall -Wextra -Werror

.PHONY: build install uninstall clean

build:
	$(CC) $(CFLAGS) -o project project.c

install:
	install -Dm755 project $(DESTDIR)$(PREFIX)/bin/project

uninstall:
	rm -f $(DESTDIR)$(PREFIX)/bin/project

clean:
	rm -f project
```

### Error Handling
```c
fprintf(stderr, "project: description of error: %s\n", strerror(errno));
return 1;
```

### Safe Path Handling
```c
char *real = realpath(file, NULL);
if (!real) {
    fprintf(stderr, "project: cannot resolve %s: %s\n", file, strerror(errno));
    return 1;
}
// ... use real ...
free(real);
```

### Security: `verify-lib`
All library sourcing must pass through `verify-lib`. It resolves symlinks, validates ownership (uid/gid 0), checks for group/world writability, and walks the directory chain to prevent TOCTOU or namespace escape attacks.

**Usage:**
```bash
_src() { local p; p=$(verify-lib "$1" "$LIBDIR/") && source "$p" || exit 1; }
```

**Implementation (`verify-lib.c`):**
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/statvfs.h>
#include <limits.h>
#include <errno.h>

/* Check if running inside a non-init user namespace */
static int in_user_ns(void) {
    FILE *f = fopen("/proc/self/uid_map", "r");
    if (!f) return 0;
    unsigned int inner, count;
    unsigned long long outer;
    int lines = 0, trivial = 0;
    while (fscanf(f, "%u %llu %u", &inner, &outer, &count) == 3) {
        lines++;
        if (inner == 0 && outer == 0 && count >= 1000) trivial = 1;
    }
    fclose(f);
    return !(lines == 1 && trivial);
}

/* Read kernel overflow uid (shown for unmapped uids in user ns) */
static unsigned int get_overflow_uid(void) {
    FILE *f = fopen("/proc/sys/kernel/overflowuid", "r");
    if (!f) return 65534;
    unsigned int uid = 65534;
    if (fscanf(f, "%u", &uid) != 1) uid = 65534;
    fclose(f);
    return uid;
}

/* Check if path resides on a read-only mount */
static int on_readonly_mount(const char *path) {
    struct statvfs sv;
    if (statvfs(path, &sv) != 0) return 0;
    return (sv.f_flag & ST_RDONLY) != 0;
}

static int verify_dir_chain(const char *path, const char *prefix,
                            int userns, unsigned int overflow_uid) {
    char buf[PATH_MAX];
    struct stat st;
    size_t prefix_len = strlen(prefix);
    if (strnlen(path, PATH_MAX) >= PATH_MAX) return 0;
    strncpy(buf, path, PATH_MAX - 1);
    buf[PATH_MAX - 1] = '\0';

    while (strlen(buf) >= prefix_len) {
        if (lstat(buf, &st) != 0) {
            fprintf(stderr, "verify-lib: cannot stat %s: %s\n", buf, strerror(errno));
            return 0;
        }
        if (st.st_uid != 0) {
            if (!(userns && st.st_uid == overflow_uid && on_readonly_mount(buf))) {
                fprintf(stderr, "verify-lib: %s uid=%d, expected 0\n", buf, st.st_uid);
                return 0;
            }
        }
        if ((st.st_mode & S_IWGRP) && st.st_gid != 0) {
            if (!(userns && st.st_gid == overflow_uid && on_readonly_mount(buf))) {
                fprintf(stderr, "verify-lib: %s group-writable with gid=%d\n", buf, st.st_gid);
                return 0;
            }
        }
        if ((st.st_mode & S_IWOTH) && !(st.st_mode & S_ISVTX)) {
            fprintf(stderr, "verify-lib: %s world-writable without sticky\n", buf);
            return 0;
        }
        char *slash = strrchr(buf, '/');
        if (!slash || slash == buf) break;
        *slash = '\0';
    }
    return 1;
}

int main(int argc, char *argv[]) {
    if (argc < 2 || argc > 3) {
        fprintf(stderr, "usage: verify-lib <file> [prefix]\n");
        return 1;
    }
    const char *file = argv[1];
    const char *prefix = argc == 3 ? argv[2] : "/usr/lib/";
    int userns = in_user_ns();
    unsigned int overflow_uid = get_overflow_uid();

    char *real = realpath(file, NULL);
    if (!real) {
        fprintf(stderr, "verify-lib: cannot resolve %s: %s\n", file, strerror(errno));
        return 1;
    }
    if (strncmp(real, prefix, strlen(prefix)) != 0) {
        fprintf(stderr, "verify-lib: %s resolves outside %s\n", real, prefix);
        free(real); return 1;
    }
    struct stat st;
    if (lstat(real, &st) != 0) {
        fprintf(stderr, "verify-lib: cannot stat %s: %s\n", real, strerror(errno));
        free(real); return 1;
    }
    if (!S_ISREG(st.st_mode)) {
        fprintf(stderr, "verify-lib: %s not a regular file\n", real);
        free(real); return 1;
    }
    if (st.st_uid != 0 || st.st_gid != 0) {
        if (userns && st.st_uid == overflow_uid && st.st_gid == overflow_uid && on_readonly_mount(real)) {
            /* unmapped root on ro mount inside user ns */
        } else {
            fprintf(stderr, "verify-lib: %s ownership %d:%d, expected 0:0\n", real, st.st_uid, st.st_gid);
            free(real); return 1;
        }
    }
    if (st.st_mode & (S_IWGRP | S_IWOTH)) {
        fprintf(stderr, "verify-lib: %s writable by non-root (mode=%04o)\n", real, st.st_mode & 07777);
        free(real); return 1;
    }
    if (userns && !on_readonly_mount(real)) {
        fprintf(stderr, "verify-lib: %s on writable mount in user ns\n", real);
        free(real); return 1;
    }
    if (!verify_dir_chain(real, prefix, userns, overflow_uid)) {
        free(real); return 1;
    }
    printf("%s\n", real);
    free(real);
    return 0;
}
```

## 6. GO

### CLI Structure (manual parsing, no flag/cobra)
```go
package main

import (
    "fmt"
    "os"
)

func main() {
    if err := run(); err != nil {
        fmt.Fprintf(os.Stderr, "project: %v\n", err)
        os.Exit(1)
    }
}

func run() error {
    args := os.Args[1:]
    if len(args) == 0 { return usageError() }
    cmd := args[0]
    flags := args[1:]
    switch cmd {
    case "subcmd": return cmdSubcmd(flags)
    case "help", "--help", "-h": printUsage(); return nil
    case "version", "--version", "-V": cmdVersion(); return nil
    default: return fmt.Errorf("unknown command %q\nrun 'project help' for usage", cmd)
    }
}
```

### Error Handling
```go
return fmt.Errorf("operation failed: %w", err)
```
- Wrap errors with `%w` for context
- Print to stderr in `main()` only: `fmt.Fprintf(os.Stderr, "project: %v\n", err)`
- `os.Exit(1)` on fatal errors

### Config (TOML with BurntSushi)
```go
import "github.com/BurntSushi/toml"

type Config struct {
    Dest    string `toml:"dest"`
    Shell   string `toml:"shell"`
    Prompts map[string]PromptConfig `toml:"prompts"`
}

func Load(path string) (*Config, error) {
    var cfg Config
    if _, err := toml.DecodeFile(path, &cfg); err != nil {
        return nil, fmt.Errorf("parse %s: %w", path, err)
    }
    if err := cfg.validate(); err != nil {
        return nil, fmt.Errorf("%s: %w", path, err)
    }
    return &cfg, nil
}

func (c *Config) validate() error {
    if c.Dest == "" { return fmt.Errorf("dest is required") }
    return nil
}
```

### Template Rendering (text/template)
```go
import (
    "bytes"
    "text/template"
)

func Render(content string, name string, data map[string]any) ([]byte, error) {
    tmpl, err := template.New(name).
        Funcs(FuncMap()).
        Option("missingkey=error").
        Parse(content)
    if err != nil {
        return nil, fmt.Errorf("parse template %s: %w", name, err)
    }
    var buf bytes.Buffer
    if err := tmpl.Execute(&buf, data); err != nil {
        return nil, fmt.Errorf("execute template %s: %w", name, err)
    }
    return buf.Bytes(), nil
}
```

### Secure Temp Directory
Prevents symlink race attacks in `/tmp`. Uses XDG runtime dir first, falls back to user state dir. Always `0700`.
```go
package safetemp

import (
    "os"
    "path/filepath"
)

// SecureDir returns a directory suitable for temporary files that should
// not be accessible to other users. The directory is created with mode 0700
// if it does not exist.
//
// Priority:
//  1. $XDG_RUNTIME_DIR/<project>/       — typically /run/user/<uid>, mode 0700
//  2. $HOME/.local/state/<project>/tmp/ — user state directory, mode 0700
//  3. ""                                — fallback (caller should handle)
func SecureDir(project string) string {
    dirs := secureDirs(project)
    for _, dir := range dirs {
        if err := os.MkdirAll(dir, 0o700); err == nil {
            return dir
        }
    }
    return ""
}

func secureDirs(project string) []string {
    var result []string
    if dir := os.Getenv("XDG_RUNTIME_DIR"); dir != "" {
        result = append(result, filepath.Join(dir, project))
    }
    if home, err := os.UserHomeDir(); err == nil {
        result = append(result, filepath.Join(home, ".local", "state", project, "tmp"))
    }
    return result
}
```

### Script Execution (secure)
```go
func execScript(content []byte, shell string) error {
    dir := safetemp.SecureDir("project")
    tmp, err := os.CreateTemp(dir, "project-script-*.sh")
    if err != nil { return err }
    defer os.Remove(tmp.Name())

    if _, err := tmp.Write(content); err != nil { tmp.Close(); return err }
    tmp.Close()

    if err := os.Chmod(tmp.Name(), 0o700); err != nil { return err }

    cmd := exec.Command(shell, tmp.Name())
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.Stdin = os.Stdin
    return cmd.Run()
}
```

### Tests (standard testing package)
```go
package config

import (
    "os"
    "path/filepath"
    "testing"
)

func TestExpandHome(t *testing.T) {
    tests := []struct {
        name  string
        input string
        want  string
    }{
        {"tilde only", "~", "/home/user"},
        {"tilde path", "~/.config", "/home/user/.config"},
        {"absolute", "/etc/passwd", "/etc/passwd"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := expandHome(tt.input)
            if got != tt.want {
                t.Errorf("expandHome(%q) = %q, want %q", tt.input, got, tt.want)
            }
        })
    }
}
```

### Root-Only Tests
```go
func skipIfNotRoot(t *testing.T) {
    if os.Geteuid() != 0 { t.Skip("requires root") }
}

func TestApplyActions(t *testing.T) {
    skipIfNotRoot(t)
    // ... tests that require chmod/chown ...
}
```

### Dependency Injection for Testing
```go
// Inject isDirFunc to avoid real filesystem lookups in tests
func ComputeActions(rules []PermRule, managedPaths []string, dest string, isDirFunc func(string) bool) []Action {
    if isDirFunc == nil {
        isDirFunc = func(p string) bool {
            info, err := os.Stat(p)
            return err == nil && info.IsDir()
        }
    }
    // ...
}
```

### Build Flags
```makefile
CGO_ENABLED=0 go build -trimpath -buildmode=pie -ldflags "-X main.version=$(VERSION)" -o $(BINARY) ./cmd/project/
```

## 7. TESTING

### Test documentation: `tests/README.md` vs `tests.md`
- **`tests/README.md`** — for Shell/Python/C projects where `tests/` is a directory containing test scripts (`test_config.sh`, `test_module.py`, etc.). If the project contains a test suite, this file SHOULD be present inside that directory.
- **`tests.md`** — for Go and C# projects where tests are part of the build system (`*_test.go` packages, `*.Tests.csproj` projects) and there is no standalone `tests/` script directory. If the project contains a test suite, this file SHOULD be present at the repository root.

### Shell/Python: `tests/README.md`

````markdown
# Tests

## Overview

| File | Language | Framework | What it tests |
|------|----------|-----------|---------------|
| `test_config.sh` | Bash | Custom assertions | Config loading, parsing, quoting |
| `test_integration.sh` | Bash | Custom assertions | End-to-end flow |
| `test_module.py` | Python | pytest | Pure functions |

## Running

```bash
# All tests
make test

# Individual suites
bash tests/test_config.sh
python -m pytest tests/test_module.py -v
```

## How they work

### Bash unit tests
All unit test files source `test_harness.sh`, which provides:
- **Assertion functions**: `ok`/`fail`/`assert_eq`/`assert_match`/`assert_contains`/`assert_rc`/`run_cmd`
- **Mock call tracking**: `mock_call_count`, `mock_last_args`, `mock_clear_log`
- **Temporary directory**: `$TESTDIR` cleaned up via `trap EXIT`
- **Global state isolation**: `reset_globals()` between test sections
- **Mock framework**: `make_mock` (writes scripts to `$MOCK_BIN` with call logging)
- **Default mocks**: `stat`, `findmnt`, `mountpoint`, `python3`, `mount`, `btrfs`, `flock`, `df`

### Python tests
Standard pytest suites. No system access — all filesystem operations use `tmp_path`, all subprocess calls are mocked.

## Test environment
- Bash tests create a temporary directory (`mktemp -d`) cleaned up via `trap EXIT`
- No root privileges required
- No real disks, partitions, or volumes are touched
- Python tests use pytest's `tmp_path` fixture
````

### `test_harness.sh` — Standard Pattern
```bash
#!/usr/bin/env bash
# tests/test_harness.sh
#
# Shared test harness for unit tests.
# Sourced by individual test files — NOT run directly.

set -uo pipefail
# Note: no -e. Tests must continue running when assertions fail
# so failures can be counted and reported by summary().

PASS=0; FAIL=0; TESTS=0

# ── Test helpers ─────────────────────────────────────────────
ok() { PASS=$((PASS + 1)); TESTS=$((TESTS + 1)); echo "  ✓ $1"; }
fail() { FAIL=$((FAIL + 1)); TESTS=$((TESTS + 1)); echo "  ✗ $1"; }

assert_eq() {
    local desc="$1" expected="$2" actual="$3"
    if [[ "$expected" == "$actual" ]]; then ok "$desc"
    else fail "$desc (expected='$expected', got='$actual')"; fi
}

assert_match() {
    local desc="$1" pattern="$2" actual="$3"
    if [[ "$actual" =~ $pattern ]]; then ok "$desc"
    else fail "$desc (pattern='$pattern' not found in '$actual')"; fi
}

assert_contains() {
    local desc="$1" needle="$2" haystack="$3"
    if [[ "$haystack" == *"$needle"* ]]; then ok "$desc"
    else fail "$desc (needle='$needle' not in output)"; fi
}

assert_not_contains() {
    local desc="$1" needle="$2" haystack="$3"
    if [[ "$haystack" != *"$needle"* ]]; then ok "$desc"
    else fail "$desc (needle='$needle' unexpectedly found)"; fi
}

assert_file_exists() {
    local desc="$1" path="$2"
    if [[ -e "$path" ]]; then ok "$desc"
    else fail "$desc (missing: $path)"; fi
}

assert_file_not_exists() {
    local desc="$1" path="$2"
    if [[ ! -e "$path" ]]; then ok "$desc"
    else fail "$desc (unexpected: $path)"; fi
}

assert_file_contains() {
    local desc="$1" needle="$2" file="$3"
    if grep -qF "$needle" "$file" 2>/dev/null; then ok "$desc"
    else fail "$desc (needle='$needle' not in $file)"; fi
}

# Run command in subshell, capture rc + combined stdout/stderr.
# Sets globals: _rc, _out
run_cmd() {
    _rc=0; _out=$("$@" 2>&1) || _rc=$?
}

assert_rc() {
    local desc="$1" expected="$2"; shift 2
    local rc=0; "$@" >/dev/null 2>&1 || rc=$?
    assert_eq "$desc" "$expected" "$rc"
}

section() { echo ""; echo "── $1 ──"; }

# ── Mock call tracking ──────────────────────────────────────
mock_call_count() {
    local name="$1" log="${TESTDIR}/mock_calls_${name}.log"
    if [[ -f "$log" ]]; then wc -l < "$log" | tr -d ' '; else echo "0"; fi
}
mock_last_args() {
    local name="$1" log="${TESTDIR}/mock_calls_${name}.log"
    if [[ -f "$log" ]]; then tail -1 "$log"; else echo ""; fi
}
mock_clear_log() {
    local name="$1" log="${TESTDIR}/mock_calls_${name}.log"
    : > "$log"
}

# ── Setup test environment ───────────────────────────────────
TESTDIR=$(mktemp -d)
trap 'rm -rf "$TESTDIR"' EXIT
MOCK_BIN="${TESTDIR}/mock_bin"
mkdir -p "$MOCK_BIN"
ORIG_PATH="$PATH"
export PATH="${MOCK_BIN}:${PATH}"

make_mock() {
    local name="$1"; shift
    local body="${*:-exit 0}"
    local log_file="${TESTDIR}/mock_calls_${name}.log"
    : > "$log_file"
    cat > "${MOCK_BIN}/${name}" <<ENDSCRIPT
#!/bin/bash
printf '%s\n' "\$*" >> "${log_file}"
${body}
ENDSCRIPT
    chmod +x "${MOCK_BIN}/${name}"
}

make_mock_in() {
    local dir="$1" name="$2"; shift 2
    local body="${*:-exit 0}"
    mkdir -p "$dir"
    cat > "${dir}/${name}" <<ENDSCRIPT
#!/bin/bash
${body}
ENDSCRIPT
    chmod +x "${dir}/${name}"
}

# ── Reset project globals ────────────────────────────────────
# Override this in your test file to clear project-specific state.
reset_globals() { :; }

# ── Default mocks ────────────────────────────────────────────
REAL_STAT=$(command -v stat 2>/dev/null || echo /usr/bin/stat)
make_mock stat "
if [[ \"\${1:-}\" == \"-c\" && \"\${2:-}\" == \"%u\" ]]; then
    echo \"0\"
else
    exec \"${REAL_STAT}\" \"\$@\"
fi
"
make_mock findmnt    'echo ""'
make_mock mountpoint 'exit 0'
make_mock python3    'echo ""'
make_mock mount      'exit 0'
make_mock btrfs      'exit 0'
make_mock flock      'exit 0'
make_mock df         'echo ""'

# ── Summary ──────────────────────────────────────────────────
summary() {
    local name="${0##*/}"
    echo ""
    echo "════════════════════════════════════"
    echo " ${name}: ${PASS} passed, ${FAIL} failed (total: ${TESTS})"
    echo "════════════════════════════════════"
    if [[ $FAIL -ne 0 ]]; then exit 1; fi
    exit 0
}
```

### Go: `tests.md`

````markdown
# Tests

## Overview

| Package | File | What it tests |
|---------|------|---------------|
| `internal/config` | `config_test.go` | Parsing, validation, defaults |
| `internal/engine` | `status_test.go` | Status reporting, template rendering |
| `internal/perms` | `apply_test.go` | Permission computation and application |

## Running

```bash
# All tests (no root)
make test

# Individual package
go test ./internal/config/ -v

# Perms tests that require root (chmod/chown/full pipeline)
make test-root
```

## How they work

### Unit tests
All tests use Go's standard `testing` package with `t.TempDir()` for filesystem isolation. No external test frameworks.

### Root-only tests
Guarded by `skipIfNotRoot`. Run via `make test-root` (sudo).

## Test environment
- All tests create temporary directories via `t.TempDir()`, cleaned up automatically
- No root privileges required except `internal/perms` apply tests
- No real home directories or system files are touched
- Root-only tests skip with `t.Skip("requires root")` when run as non-root
````

### C# / .NET: `tests.md`

````markdown
# Tests

## Overview

| Project | File | Framework | What it tests |
|---------|------|-----------|---------------|
| `subs2srs.Tests` | `UtilsSubsTests.cs` | xUnit | Time formatting, padding, overlap |
| `subs2srs.Tests` | `PrefIOTests.cs` | xUnit | JSON round-trip, migration, defaults |
| `subs2srs.Tests` | `ProjectIOTests.cs` | xUnit | `.s2s.json` save/load, corruption handling |

## Running

```bash
# All tests
make test

# Individual suite
dotnet test subs2srs.Tests/subs2srs.Tests.csproj --filter "FullyQualifiedName~UtilsSubsTests"
```

## How they work

### xUnit suites
- **Parallelization disabled**: `[assembly: CollectionBehavior(DisableTestParallelization = true)]` prevents race conditions on mutable static state.
- **Singleton reset**: `Settings.Instance.reset()` called in constructor and `Dispose()` to isolate test state.
- **Temp directories**: `Path.GetTempPath()` + `Guid` creates isolated dirs. Cleaned up via `IDisposable.Dispose()`.
- **Mocking**: External CLI tools are not invoked. File I/O tests use real temp files.

## Test environment
- All tests create temporary directories via `Path.Combine(Path.GetTempPath(), ...)` and clean up in `Dispose()`
- No root privileges required
- No real media files, subtitles, or system paths are touched
- Tests run sequentially to avoid singleton pollution
````

## 8. COMPLETIONS

### Zsh (`_cmd`)
```zsh
#compdef project-name

_project_name() {
    local context state state_descr line
    typeset -A opt_args

    _arguments -C \
        '(- *)'{-h,--help}'[Show help]' \
        '(- *)'{-V,--version}'[Show version]' \
        '1:command:((
            subcmd1\:"Description"
            subcmd2\:"Description"
        ))' \
        '*::arg:->args' \
        && return

    case $state in
        args)
            case ${words[1]} in
                subcmd1)
                    _arguments \
                        '(-n --dry-run)'{-n,--dry-run}'[Dry run]' \
                        '*:arg:_files'
                    ;;
            esac
            ;;
    esac
}

_project_name "$@"
```

### Bash (`cmd.bash`)
```bash
# completions/project-name.bash
# bash completion for project-name

_project_name() {
    local cur prev words cword
    _init_completion || return

    if [[ $cword -eq 1 ]]; then
        COMPREPLY=( $(compgen -W "subcmd1 subcmd2 -h --help -V --version" -- "$cur") )
        return
    fi

    case "${words[1]}" in
        subcmd1)
            COMPREPLY=( $(compgen -W "-n --dry-run" -- "$cur") )
            ;;
    esac
}

complete -F _project_name project-name
```

## 8.1. MAN PAGES

All user-facing CLI tools MUST provide a man page. Man pages are written in Markdown and compiled to roff via `pandoc -s -t man`. Source files live in `man/`.

### Naming

| File pattern | Section | Purpose |
|--------------|---------|---------|
| `project-name.8.md` | 8 | CLI commands, system utilities |
| `project-name.conf.5.md` | 5 | Configuration file formats |

### YAML Front Matter

```yaml
---
title: PROJECT-NAME
section: 8
header: System Administration
footer: project-name
---
```

| Field | Purpose |
|-------|---------|
| `title` | Uppercase project name (matches `.TH` title) |
| `section` | Man section: `8` for commands, `5` for file formats |
| `header` | Center header (e.g., `System Administration`, `File Formats`) |
| `footer` | Lower-right corner — project name only. MUST NOT contain version, date, or other metadata. |

Omit `date`.

### Required Sections (section 8 — commands)

```markdown
# NAME

project-name — short description

# SYNOPSIS

**project-name** \<command\> [options]

# DESCRIPTION

What the tool does, how it works.

# COMMANDS

**apply** [-n|\--dry-run]
:   What this command does.

**status**
:   What this command does.

# OPTIONS

**-h**, **\--help**
:   Show usage and exit.

**-V**, **\--version**
:   Print version and exit.

# EXAMPLES

Typical usage:

    project-name apply

Preview:

    project-name apply --dry-run

# EXIT STATUS

**0**
:   Success.

**1**
:   Error. Common causes.

# FILES

**~/.local/state/project-name/\***
:   State files.

# SEE ALSO

**related-tool**(8)
```

### Required Sections (section 5 — config files)

```markdown
# NAME

project.conf — configuration for project-name

# SYNOPSIS

*/etc/project.conf*

# DESCRIPTION

What the config file is for, who reads it.

Format description (KEY=VALUE, TOML, YAML, etc.). Security requirements (ownership, no eval).

# OPTIONS

**KEY_NAME**
:   Description. Default: *value*.

# SECURITY

Ownership requirements, parsing restrictions, what is rejected.

# EXAMPLES

Minimal config:

    # /etc/project.conf
    KEY_NAME = value

# SEE ALSO

**project-name**(8)
```

### Makefile Integration

```makefile
MANPAGES = man/project-name.8

man: $(MANPAGES)

man/%.8: man/%.8.md
	pandoc -s -t man -o $@ $<

clean:
	rm -f $(MANPAGES)

install:
	install -Dm644 man/project-name.8 $(DESTDIR)$(MANDIR)/man8/project-name.8

uninstall:
	rm -f $(DESTDIR)$(MANDIR)/man8/project-name.8
```

The `install` target MUST only install files — it MUST NOT trigger build steps
(`man`, `build`, etc.). Build artifacts are the maintainer's responsibility to
produce before running `make install`.

## 9. INFRASTRUCTURE (Python + Jinja2 + SOPS)

Infrastructure projects (`infra`, `tf-infra`) manage server configurations, DNS records, and service deployments. They are **not installable packages** — conventions like `Makefile` (PREFIX/DESTDIR), `depends`, `bin/`, `lib/`, `tests/README.md`, and `verify-lib` do not apply.

What does apply:
- §4 Python patterns: entry points, error handling (`sys.exit(1)`, stderr), SOPS helper
- Jinja2 templating for service configs
- SOPS-encrypted secrets (`.sops.yaml`, `secrets.enc.yaml`)
- Tests use pytest with mocked SSH/HTTP (§7 Python test patterns)

### ServiceDeployer Pattern
```python
class ServiceDeployer:
    def __init__(self, config: dict):
        self.files = config['files']
        self.setup_dirs = config.get('setup_dirs', [])
        self.restart_cmd = config.get('restart_cmd')
        self.templates_dir = config['templates_dir']
        self.secrets_file = config['secrets_file']

    def _get_env(self):
        return create_jinja_env(self.templates_dir)

    def deploy(self, hosts, secrets, env, no_restart=False):
        target, port = resolve_target(hosts, secrets['host'])
        for entry in self.files:
            tpl, rp, opts = entry[0], entry[1], entry[2] if len(entry) == 3 else {}
            rendered = env.get_template(tpl).render(**secrets)
            # rsync + chown/chmod via SSH
```

### Jinja2 Templates
```jinja2
# Managed by infra repo — {{ instance_name }}
HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ssh-ed25519
AllowUsers {{ common.ssh_allowed_users | join(' ') }}
Port {{ instance.ssh_port | default(common.ssh_port) }}

{% for user in common.ssh_otp_users %}
Match User {{ user }}
    AuthenticationMethods keyboard-interactive
{% endfor %}
```

## 10. DEPENDS

Format: one line per dependency. Comments use `#`.
```
# system
system:python3
system:btrfs
system:ukify
gitpkg:verify-lib
```

**Dependency types:**
- `system:<pkg>` — Package from the system repository manager (pacman, apt, dnf). Installed via standard package manager.
- `gitpkg:<name>` — Project managed by `gitpkg`. Resolved from configured base URLs or collections. Cloned, built, and installed via `gitpkg install <name>`.
- `# comment` — Ignored by parsers.

**Go projects do not use `depends`.** Dependencies are managed by `go.mod`/`go.sum`. The `depends` file is only for shell, Python, C, and other projects without a native dependency manager.

## 11. CODE GENERATION PROTOCOL

This section defines how code must be generated when working with fkzys projects. Assistants and developers creating new code must follow these rules.

1. **Verify structure** (`Makefile`, `lib/`, `tests/` or `tests.md`) before adding files. For infrastructure projects (§9), these do not apply. Go projects use `go.mod` instead of `depends`.
2. **Apply standards automatically**: `set -euo pipefail` (except in test files, wrappers, and guard scripts — see §2), whitelist config parser, `verify-lib` sourcing, `printf -v` instead of `eval`, explicit shopt restore.
3. **No placeholders**. Provide complete, runnable code.
4. **Include tests** or update test documentation when adding features. Test files use `set -uo pipefail` (no `-e`) — failures are counted, not caught by errexit.
5. **Flag** `eval`, `chmod 777`, hardcoded secrets, missing ownership checks, `/tmp` usage for scripts.
6. **Ask if uncertain** about paths, versions, or flags before generating.
7. **When editing this specification**, new top-level sections are appended at the end with the next sequential number. Subsections MUST use dot notation (e.g., §8.1 MAN PAGES). Existing section numbers MUST NOT be changed.

## 12. QUICK TEMPLATES

- Shell header: `#!/bin/bash\nset -euo pipefail`
- Library header: `#!/usr/bin/env bash`
- Test file header: `#!/usr/bin/env bash\nset -uo pipefail` (no `-e` — see §2 exceptions)
- Error: `echo "ERROR: msg" >&2; exit 1`
- Config check: `[[ -n "${VAR:-}" ]] || { echo "ERROR: VAR not defined" >&2; exit 1; }`
- Make install: `install -Dm755 bin/cmd $(DESTDIR)$(PREFIX)/bin/cmd`
- Python entry: `if __name__ == "__main__": main()`
- Go main: `func main() { if err := run(); err != nil { fmt.Fprintf(os.Stderr, "project: %v\n", err); os.Exit(1) } }`
- Go build: `CGO_ENABLED=0 go build -trimpath -buildmode=pie -ldflags "-X main.version=$(VERSION)"`
- C# build: `dotnet build Project/Project.csproj -c Release`
- Test run: `make test` or `bash tests/test.sh` or `python -m pytest tests/ -v` or `go test ./...` or `dotnet test Tests/Tests.csproj`
- Man YAML: `title: NAME\nsection: 8\nheader: System Administration\nfooter: project-name` (no `date`)
- Man compile: `pandoc -s -t man cmd.8.md -o cmd.8`
- SOPS: `sops -d secrets.enc.yaml`

## 13. C# / .NET

### Resource Management
```csharp
// Always use `using` for IDisposable (StreamReader, FileStream, XmlReader, etc.)
using var reader = new StreamReader(path, encoding);
// No explicit Close() needed — disposed on scope exit or exception.
```

### Async / UI Thread Marshaling
```csharp
// Install a custom SynchronizationContext in Program.cs to marshal
// `await` continuations back to the UI main loop (GTK/WinForms/etc.).
private async void OnButtonClicked(object sender, EventArgs e)
{
    sender.SetSensitive(false);
    var result = await Task.Run(() => HeavyComputation());
    UpdateUI(result); // Safe: runs on main thread
    sender.SetSensitive(true);
}
```

### Parallel Processing with Cancellation
```csharp
var parallelOptions = new ParallelOptions { MaxDegreeOfParallelism = MaxThreads };
Parallel.ForEach(workItems, parallelOptions, (item, state) =>
{
    if (cancellationToken.IsCancellationRequested) { state.Stop(); return; }
    Process(item);
});
```

### Testing (xUnit)
- Disable parallelization if tests share mutable static state: `[assembly: CollectionBehavior(DisableTestParallelization = true)]`
- Reset singletons in constructor/dispose to avoid pollution.
- Use `Path.GetTempPath()` + `Guid` for isolated temp dirs. Clean up in `IDisposable.Dispose()`.
- Mock external CLI tools or use temp files for I/O tests.

### File I/O Safety
- Atomic writes: write to `.tmp` extension, then `File.Move(tmp, final, overwrite: true)`.
- Never hardcode paths: use `Path.Combine`, `Environment.GetFolderPath`.
- Validate extensions before parsing: `Path.GetExtension(path).ToLowerInvariant()`.
- Always wrap `StreamReader`/`FileStream` in `using` to prevent descriptor leaks.

## 14. VERSIONING

### Git Tags

All releases MUST be tagged with annotated, signed git tags:

```bash
git tag -s -a v0.0.1 -m 'v0.0.1'
git push origin v0.0.1
```

- **`-s`** — sign the tag
- **`-a`** — create an annotated tag (stores tagger, date, message as a full Git object)
- Both flags are explicit: `-s` implies `-a`, but writing both makes intent clear

Version format: `v<major>.<minor>.<patch>` (semantic versioning with `v` prefix).

### Signed Commits

All commits MUST be signed:

```bash
git commit -S -m 'description'
```

- **`-S`** — sign the commit

This applies to both human-authored and LLM-generated commits. When LLM assistants generate code, the human reviewer must sign the resulting commit.

### Conventional Commits

All commits in ecosystem projects MUST follow the Conventional Commits format:

```
<type>: <description>
```

| Type | Purpose |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `ci` | CI configuration changes |
| `refactor` | Code change, no behavior change |
| `test` | Test additions or changes |
| `chore` | Maintenance, dependency updates, config |

Examples:

```
feat: add --separate-home flag for isolated home subvolumes
fix: handle missing ESP mount point gracefully
docs: clarify test target rules in §3
ci: call test scripts directly instead of make test
```

Scope may be added in parentheses: `fix(cli): handle --dry-run with custom tag`.

Commits to this specification follow the same Conventional Commits format.

## 15. CI (GitHub Actions)

All projects SHOULD have `.github/workflows/ci.yml` with path-filtered triggers.

### Path Filtering

CI triggers MUST use `paths` (whitelist) to avoid running on unrelated changes
(README edits, LICENSE updates, asset changes). Example:

```yaml
on:
  push:
    paths:
      - 'bin/**'
      - 'lib/**'
      - 'tests/**'
      - '.github/workflows/ci.yml'
  pull_request:
    paths:
      - 'bin/**'
      - 'lib/**'
      - 'tests/**'
      - '.github/workflows/ci.yml'
```

`paths-ignore` (blacklist) is NOT used — prefer explicit whitelisting.

### Test Execution

CI MUST call test commands directly, NOT through `make test`. This keeps CI
output explicit and avoids coupling to the Makefile.

| Language | CI command |
|----------|-----------|
| Shell | `bash tests/test.sh` or loop over `tests/test_*.sh` |
| Python | `python -m pytest tests/ -v` |
| Go | `go test ./... -v -count=1` |
| C# | `dotnet test Project.Tests/Project.Tests.csproj` |

Example — Shell project with multiple test files:

```yaml
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: |
          for t in tests/test_config.sh tests/test_cli.sh tests/test_commands.sh; do
            echo "━━━ $$t ━━━"
            bash "$$t" || exit 1
          done
```

Example — Go project with `dorny/paths-filter`:

```yaml
  changes:
    runs-on: ubuntu-latest
    outputs:
      go: ${{ steps.filter.outputs.go }}
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - uses: dorny/paths-filter@v4
        id: filter
        with:
          filters: |
            go:
              - '**.go'
              - 'go.mod'
              - 'go.sum'
              - '.github/workflows/ci.yml'

  test:
    needs: changes
    if: needs.changes.outputs.go == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v6
        with:
          go-version: '1.24'
      - run: go test ./... -v -count=1
```

### Linting

Shell projects SHOULD run `shellcheck` in CI. Python projects SHOULD run
`ruff` and `mypy`. C projects SHOULD run `cppcheck`. Go projects SHOULD
run `go vet`.

## COPYRIGHT & LICENSING

Copyright (c) 2026 fkzys

This specification and its embedded architectural patterns are dual-licensed. You may use this project under either the Open Source (Copyleft) licenses or a Commercial License.

### 1. Open Source (Copyleft)

Under the open-source model, the following copyleft licenses apply:

* **Specification Text & Documentation:** Licensed under the **Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)**. Any modified versions or derived specifications must be shared under the identical license.
* **Code Snippets & Architecture:** All embedded code blocks, Makefiles, bash functions (e.g., `test_harness.sh`), C sources (e.g., `verify-lib.c`), Go structures, and infrastructure templates (Jinja2/Terraform patterns) are licensed under the **GNU Affero General Public License v3.0 or later (AGPL-3.0-or-later)**.

**Note on Implementations:**
Any tools, infrastructure-as-code deployments (§9), or systems that incorporate, copy, or adapt the code snippets and architectural patterns defined in this specification are considered derivative works. These must be distributed under the **AGPL-3.0-or-later** license. This explicitly includes network-interacting infrastructure covered by the AGPL network interaction clause.

### 2. Commercial License

If you wish to use this specification, implement its patterns, or use its associated tools (`gitpkg`, `infra`, `verify-lib`, etc.) in a proprietary, closed-source product, or if your policies prohibit the use of AGPLv3 software, a **Commercial License** is available.

A commercial license grants a legal waiver from the AGPLv3 and CC BY-SA 4.0 copyleft requirements, permitting the use, modification, and integration of this work into private or commercial infrastructure without the obligation to disclose source code.

Please contact **[main.component985@passfwd.com]** for commercial licensing inquiries.

---
*Unless required by applicable law or agreed to in writing, the specification and code distributed under the Open Source licenses are distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.*
