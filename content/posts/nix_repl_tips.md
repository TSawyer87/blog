+++
date = '2025-05-14T19:18:13-04:00'
draft = false
title = 'Nix_repl_tips'
author = "T Sawyer"
+++

# Nix Repl List available commands

{{< figure src="/images/gruv11.png" alt="gruv11" width="1000" >}}

- [Nix Reference Manual Nix Repl](https://nix.dev/manual/nix/2.17/command-ref/new-cli/nix3-repl)

- The command: `nix repl` - starts an interactive environment for evaluating
  Nix expressions.

- This command provides an interactive environment for evaluating Nix expressions. (REPL stands for 'read–eval–print loop'.)

On startup, it loads the Nix expressions named files and adds them into the
lexical scope. You can load addition files using the `:l <filename>` command,
or reload all files using `:r`.

List available commands with `:?`:

```nix
nix repl
Nix 2.24.11
Type :? for help.
nix-repl> :?
```

## Load Nix expressions Directly

You can quickly evaluate a random Nix expression:

```nix
nix repl --expr '{a = { b = 3; c = 4; }; }'

Welcome to Nix 2.11.0. Type :? for help.

Loading installable ''...
Added 1 variables.
nix-repl> a
{ b = 3; c = 4; }
```

- The `--expr` flag is helpful to prime directly the Nix REPL with valuable data
  or values.

```nix
nix repl --expr 'import <nixpkgs>{}'
nix-repl> helix.name
"helix-25.01.1"
nix-repl> helix.drvPath
"/nix/store/1rxncc2jk2l2gc1f8m0442ak1yn1s5gj-helix-25.01.1.drv"
nix-repl> drv = runCommand "hello" { buildInputs = [ hello ]; } "hello; hello > $out"
nix-repl> :b drv
nix-repl> builtins.readFile drv
"Hello, world!\n"
nix-repl> :log drv
Hello, world!
```

- `:b` is for build derivation

### Load Flakes

We can use the `--expr` flag to load a random Nix Flake directly:

```nix
nix repl --expr 'builtins.getFlake "github:nix-community/ethereum.nix"'
```

Also, you can load a flake directly inside the REPL with `:load-flake` or `:lf`:

```nix
nix repl

nix-repl> :lf github:nix-community/home-manager
# or
nix-repl> :lf /path/to/your/flake
```

## Debugging with a Flake REPL output

- One way to do this is to launch the repl with `nix repl` and inside the repl
  type `:lf /path/to/flake`. Or `nixos-rebuild repl --flake /path/to/flake` the
  latter provides a helpful welcome script showing what is loaded into your
  repl's scope.

I like to create a simple repl output to load your flake into the environment
with `nix repl .#repl`.

First, we'll create a REPL environment to inspect and debug our flake's outputs,
packages, and configurations. Define a `repl` output in `flake.nix` for easy
access with `nix repl .#repl`:

```nix
# flake.nix
outputs = { self, nixpkgs, ... }: let
  pkgs = import nixpkgs { system = "x86_64-linux"; };
in {
  repl = import ./repl.nix {
    inherit (pkgs) lib;
    flake = self;
    pkgs = pkgs;
  };
};
```

And in `repl.nix`:

```nix
# repl.nix
{
  lib,
  flake,
  pkgs,
}: {
  inherit flake pkgs lib;
  configs = flake.nixosConfigurations;
  # inherit (flake.outputs) userVars;
}
# Accepts `lib`, `flake`, `pkgs` from `flake.nix` as arguments
# Attributes: flake: all flake outputs (flake.outputs, flake.inputs)
# run `nix repl .#repl` to load the REPL environment
# :l <nixpkgs>  # load additional Nixpkgs if needed
# :p flake.inputs.nixpkgs.rev # nixpkgs revision
# :p flake.inputs.home-manager.rev
# flake.outputs.packages.x86_64-linux.default # inspect default package
# pkgs.helix # access helix package
# lib.version # check lib version
# configs.magic.config.environment.systemPackages # list packages
# configs.magic.config.home-manager.users.jr.home.packages # home packages
# :p configs.magic.config.home-manager.users.jr.programs.git.userName
# Debugging
# :p builtins.typeOf configs.magic (should be `set`)
# :p builtins.attrNames configs.magic
# :p configs.magic.config # errors indicate issues
# :p configs.magic.config.environment # isolate the module or issue
# :p builtins.attrNames configs.magic.config.home-manager.users.jr # home attrs
# :p configs.magic.config.home-manager.users.jr.programs.git.enable # true/false
#  :p lib.filterAttrs (n: v: lib.hasPrefix "firefox" n) pkgs
# :p configs.magic.config.stylix # check theming
# :p configs.magic.config.home-manager.users.jr.stylix
# :p lib.mapAttrsToList (name: cfg: name) configs
```

> ❗: Replace `magic` with your host name

### Usage

Load REPL environment with:
`nix repl .#repl`

Attributes:

```nix
nix-repl> builtins.attrNames flake.inputs
[
  "dont-track-me"
  "helix"
  "home-manager"
  "hyprland"
  "neovim-nightly-overlay"
  "nixpkgs"
  "nvf"
  "rose-pine-hyprcursor"
  "stylix"
  "treefmt-nix"
  "wallpapers"
  "wezterm"
  "yazi"
]
nix-repl> builtins.attrNames flake.outputs
[
  "checks"
  "devShells"
  "formatter"
  "nixosConfigurations"
  "packages"
  "repl"
]
nix-repl> flake.outputs.formatter
{
  x86_64-linux = «derivation /nix/store/q71q00wmh1gnjzdrw5nrvwbr6k414036-treefmt.drv»;
}
```

- Inspect the default package output:

```nix
nix-repl> flake.outputs.packages.x86_64-linux.default
«derivation /nix/store/6kp660mm62saryskpa1f2p6zwfalcx2w-default-tools.drv»
```

- From here out I'll leave out the `nix-repl>` prefix just know that it's there.

- Check lib version(Nixpkgs `lib` attribute):

```nix
lib.version
"25.05pre-git"
```

- List systemPackages and home.packages, my hostname is `magic` list yours in
  its place:

```nix
configs.magic.config.environment.systemPackages
# list home.packages
configs.magic.config.home-manager.users.jr.home.packages
```

- Or an individual value:

```nix
:p configs.magic.config.home-manager.users.jr.programs.git.userName
TSawyer87
```

### Debugging

- Check if the module system is fully evaluating, anything other than a "set"
  the configuration isn't fully evaluated (e.g. "lambda" might indicate an
  unevaluated thunk):

```nix
:p builtins.typeOf configs.magic
set
```

- Debugging Module System:

- Check if `configs.magic` is a valid configuration:

```nix
:p builtins.attrNames configs.magic
```
