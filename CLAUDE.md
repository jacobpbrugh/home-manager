# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Home Manager is a system for managing user environments using the Nix package manager. It enables declarative configuration of user-specific packages and dotfiles, supporting standalone installations, NixOS modules, and nix-darwin modules.

## Development Commands

### Formatting
```bash
make format
# Or use the flake formatter directly:
nix fmt
```

Formatting uses treefmt with:
- `nixfmt` for Nix code formatting
- `deadnix` for removing unused Nix code
- `keep-sorted` for maintaining sorted lists (respects `# keep-sorted start/end` markers)

### Testing
```bash
# Run all tests
make all-tests

# Run a specific test
make test TEST=<test-name>

# Test installation
make test-install

# Run tests via the flake
nix run .#tests

# Run tests via Python script directly
./tests/tests.py
```

Tests use NMT (Nix Module Test) framework and are located in `tests/modules/`.

### Building Documentation
```bash
# Build HTML documentation
nix build .#docs-html

# Build and open documentation
nix build .#docs-htmlOpenTool

# Build JSON options
nix build .#docs-json

# Build manpages
nix build .#docs-manpages
```

### Development with local clone
```bash
# Use local checkout temporarily
home-manager -I home-manager=$HOME/devel/home-manager

# Or with flakes:
home-manager --override-input home-manager ~/devel/home-manager
```

## Architecture

### Module System

Home Manager uses the NixOS module system with a custom evaluation entry point at `modules/default.nix`:

1. **Module Discovery**: `modules/modules.nix` dynamically imports all modules from:
   - Core modules (listed explicitly)
   - All `.nix` files in `modules/programs/`
   - All `.nix` files in `modules/services/`

2. **Module Structure**: Each module typically includes:
   - `options` - NixOS-style option definitions using `mkOption`
   - `config` - Implementation that activates when options are set
   - `meta.maintainers` - List of module maintainers

3. **Extended Library**: Custom Home Manager library functions available at `lib.hm`:
   - `dag` - Directed Acyclic Graph for ordering (e.g., shell init scripts)
   - `gvariant` - GVariant type constructors for dconf
   - `types` - Custom NixOS types
   - `generators` - Configuration file generators
   - Shell-specific helpers (bash, zsh, nushell)

### DAG (Directed Acyclic Graph)

The DAG system (`lib.hm.dag`) enables ordering of configuration elements:
- `entryAnywhere` - No ordering constraints
- `entryAfter ["dep1" "dep2"]` - Must come after specified entries
- `entryBefore ["dep1" "dep2"]` - Must come before specified entries

Used extensively for shell initialization, systemd services, and activation scripts.

### Directory Structure

- `modules/` - All Home Manager modules
  - `programs/` - Program-specific modules (git, vim, etc.)
  - `services/` - Service modules (systemd user services, etc.)
  - `misc/` - Cross-cutting concerns (XDG, fonts, etc.)
  - `lib/` - Home Manager-specific library functions
- `lib/` - Additional library utilities
- `tests/` - Test suite using NMT
  - `modules/` - Module tests (mirror structure of `modules/`)
  - `lib/` - Library function tests
- `docs/` - Manual and documentation source
- `nixos/` - NixOS integration module
- `nix-darwin/` - nix-darwin integration module
- `home-manager/` - Standalone CLI tool

### Test Structure

Tests use the NMT framework with derivation scrubbing for fast evaluation:

1. Test modules define a configuration using Home Manager options
2. Test assertions check the generated activation package outputs
3. Use `nmt.script` with shell commands like:
   - `assertFileExists path` - Check file exists
   - `assertFileContent actual expected` - Compare file contents
   - `assertFileRegex actual regex` - Match regex pattern

Example test structure:
```nix
{ lib, pkgs, ... }:
{
  programs.example.enable = true;
  programs.example.setting = "value";

  nmt.script = ''
    assertFileExists home-files/.config/example/config
    assertFileContent home-files/.config/example/config ${./expected.conf}
  '';
}
```

## Writing Modules

### Basic Module Template

```nix
{ config, lib, pkgs, ... }:
let
  inherit (lib) mkEnableOption mkOption types mkIf;
  cfg = config.programs.example;
in
{
  meta.maintainers = [ lib.maintainers.yourname ];

  options.programs.example = {
    enable = mkEnableOption "example program";

    package = lib.mkPackageOption pkgs "example" { };

    settings = mkOption {
      type = types.attrs;
      default = { };
      description = "Configuration settings.";
    };
  };

  config = mkIf cfg.enable {
    home.packages = [ cfg.package ];

    xdg.configFile."example/config.toml".source =
      (pkgs.formats.toml { }).generate "config.toml" cfg.settings;
  };
}
```

### Key Conventions

- Always use `mkIf cfg.enable` to guard configuration changes
- Prefer `xdg.configFile` over `home.file` for XDG-compliant applications
- Use `lib.mkPackageOption` for package options with default
- Settings should typically use `types.attrs` or format-specific types
- Include tests in `tests/modules/<category>/<module-name>/`

## Code Style

- Use `inherit` to reduce repetition of `lib.*` qualifiers
- Maintain sorted lists between `# keep-sorted start` and `# keep-sorted end` markers
- Follow Nixpkgs formatting conventions (enforced by nixfmt)
- Remove unused let bindings (checked by deadnix)
- Prefer explicit types for all options

## Flake Outputs

The flake provides:
- `nixosModules.home-manager` - NixOS integration
- `darwinModules.home-manager` - nix-darwin integration
- `packages.<system>.home-manager` - Standalone CLI
- `packages.<system>.tests` - Test runner
- `packages.<system>.docs-*` - Documentation builders
- `formatter` - Code formatter (treefmt)
- `templates` - Project templates for different installation modes

## Integration Modes

1. **Standalone**: User manages their home directory independently via `home-manager` CLI
2. **NixOS Module**: Integrates with system config, built during `nixos-rebuild`
3. **nix-darwin**: Integrates with macOS system config, built during `darwin-rebuild`

Each mode has different constraints and available options (e.g., systemd only on Linux).
