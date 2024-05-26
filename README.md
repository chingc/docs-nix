# Nix

Hello, this is a record of my journey in learning [Nix](https://nixos.org). It serves as a quickstart and quick reference.

This document is meant primary for myself, so it won't go too far off topic and it will assume familiarity with the command line.

## What is it?

No more "works on my machine." Create environments that work seamlessly and are easily sharable across platforms.

No more broken builds or mysterious installation processes.

#### Reproducible

Nix builds packages in isolation from each other. This ensures that they are reproducible and don't have undeclared dependencies, so if a package works on one machine, it will also work on another.

#### Declarative

Nix makes it trivial to share development and build environments for your projects, regardless of what programming languages and tools you're using.

#### Reliable

Nix ensures that installing or upgrading one package cannot break other packages. It allows you to roll back to previous versions, and ensures that no package is in an inconsistent state during an upgrade.

## Install

The easiest way to install Nix (Linux, macOS, WSL2) is to use [The Determinate Nix Installer](https://zero-to-nix.com/concepts/nix-installer).

Note: Directions to upgrade or uninstall can be found in their [GitHub repo](https://github.com/DeterminateSystems/nix-installer).

## Create a Shell Environment

Now that Nix is installed, you can use it to create new shell environments with programs that you want to use.

```bash
$ cowsay nope
The program 'cowsay' is currently not installed.

$ echo no chance | lolcat
The program 'lolcat' is currently not installed.
```

Required programs not installed? No problem.

```bash
$ nix shell nixpkgs#cowsay nixpkgs#lolcat

$ cowsay Hello, Nix! | lolcat
```

Note: Initial runs take longer.

Success! You should see a rainbow colored ASCII cow greeting.

Type `exit` or press `CTRL-D` to exit the shell, and the programs won't be available anymore.

```bash
$ exit

$ cowsay no more
The program 'cowsay' is currently not installed.

$ echo all gone | lolcat
The program 'lolcat' is currently not installed.
```

## Run Programs Directly

Creating the shell environment is optional.

```bash
$ nix run nixpkgs#cowsay Hello, Nix!

$ nix run nixpkgs#lolcat -- --help

$ nix run nixpkgs#fortune | nix run nixpkgs#lolcat
```

## Search for Packages

If you can think of it, there's probably a Nix package of it.

Go to <https://search.nixos.org/packages> or use the command line to search for packages.

```bash
$ nix search nixpkgs neovim
```

#### Wrapped vs Unwrapped

The key difference between wrapped and unwrapped packages in NixOS is that wrapped packages are configured to work seamlessly within the NixOS environment, while unwrapped packages are the raw, unmodified versions.

In most cases, users should install the wrapped version of a package, as it is preconfigured to work correctly on NixOS. The unwrapped version is primarily used when further customization or overriding of the package is required, as it serves as the base for creating a new wrapped derivation with additional modifications.

## Reproducible Scripts

A trivial script with non-trivial dependencies.

```bash
#! /bin/bash

curl https://github.com/NixOS/nixpkgs/releases.atom | xml2json | jq .
```

This script fetches XML content from a URL, converts it to JSON, and formats it for better readability.

It requires curl, xml2json, jq, and bash. If any of these dependencies are not present on the system running the script, it will fail partially or altogether.

With Nix, we can declare all dependencies explicitly, and produce a script that will always run on any machine that supports Nix and the required packages taken from Nixpkgs.

```bash
#! /usr/bin/env nix
#! nix shell nixpkgs#bash nixpkgs#curl nixpkgs#jq nixpkgs#python312Packages.xmljson
#! nix --ignore-environment --command bash

curl https://github.com/NixOS/nixpkgs/releases.atom | xml2json | jq .
```

- The `--ignore-environment` option prevents the script from implicitly using programs that may already exist on the system that will run the script.
- The `--command` option specifies the interpreter that will be invoked by nix shell after it has obtained the dependencies and initialized the environment.

As long as the system has Nix, scripts can be written in any language without worrying about dependencies.

```bash
#! /usr/bin/env nix
#! nix shell --override-flake nixpkgs github:NixOS/nixpkgs/44072e24566c5bcc0b7aa9178a0104f4cfffab19
#! nix nixpkgs#python3
#! nix --ignore-environment --command python

print("n", "n^2")
for n in range(1, 10):
    print(n, n * n)
```

In this example `--override-flake` was used to specify a git commit of the Nixpkgs repository. This ensures that the script will always run with the exact same packages versions, everywhere.

Notice that `nix shell` was only specified once when using multiple nix shell shebangs.

## Declarative Shell Environments

So far we've created shell environments and shell scripts that let us run programs without having to install them. We can go even further and create a configuration file that defines an environment. This file can be shared with anyone to recreate the same environment on a different machine.

Create a `flake.nix` file:

```
{
  description = "A fun shell environment";

  # Pin nixpkgs to a specific release
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/44072e24566c5bcc0b7aa9178a0104f4cfffab19";

  inputs.flake-utils.url = "github:numtide/flake-utils";

  outputs = { self, nixpkgs, flake-utils }:

    flake-utils.lib.eachDefaultSystem (system:

      let

        # Explicitly set config and overlays to avoid them being inadvertently overridden by global config
        pkgs = import nixpkgs { inherit system; config = {}; overlays = []; };

      in {

        devShells.default = pkgs.mkShellNoCC {
          # Packages to include
          packages = with pkgs; [
            cowsay
            lolcat
          ];

          # Define environment variables
          env = {
            GREETING = "Hello, Nix!";
          };

          # Run a shell script on environment startup
          shellHook = ''
            echo $GREETING | cowsay | lolcat
          '';
        };

      }

    );
}
```

Enter the environment by running `nix develop` in the same directory as `flake.nix`. Packages defined in the `packages` attribute will be available in `$PATH`.

Note: If the directory you're working in is a git repository you may get an error indicating that `flake.nix` doesn't exist. Stage the file with `git add` to get nix to recognize it.

If you make changes to `flake.nix` just type `exit` or press `CTRL-D` to exit the environment and restart it with `nix develop`.

We've pinned nixpkgs to help create fully reproducible Nix expressions.

Picking the commit can be done via [status.nixos.org](https://status.nixos.org), which list all releases.

## References

- [GitHub: DeterminateSystems/nix-installer](https://github.com/DeterminateSystems/nix-installer)
- [GitHub: NixOS/nix](https://github.com/NixOS/nix)
- [GitHub: NixOS/nixpkgs](https://github.com/NixOS/nixpkgs)
- [Nix Reference Manual: nix search](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-search)
- [Nix Reference Manual: nix shell](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-shell)
- [Nix Reference Manual: nix-shell](https://nix.dev/manual/nix/latest/command-ref/nix-shell)
- [Nix Reference Manual: nix run](https://nix.dev/manual/nix/latest/command-ref/new-cli/nix3-run)
- [NixOS Search Packages](https://search.nixos.org/packages)
- [NixOS Status](https://status.nixos.org)
- [Zero to Nix](https://zero-to-nix.com)

Additional resources: <https://nixos.org/learn>

## TODO

- Why learn Nix section at the very beginning
- Improve What is Nix section
- Cleanup sections
