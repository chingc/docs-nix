# Nix

Hello, this is a record of my journey learning [Nix](https://nixos.org).

I'm writing this mainly as a quickstart and quick reference for my future self, but I hope others find it helpful.

## What is it?

Nix is a purely functional package manager. It helps to avoid the installation and versioning pain points of software development.

That doesn't sound like much. However...

Suppose you need golang, node, and python. You can create a shell environment that has those tools with a single command.

```bash
$ go; node; python
zsh: command not found: go
zsh: command not found: node
zsh: command not found: python

$ nix shell nixpkgs#go nixpkgs#nodejs nixpkgs#python3

$ go version
go version go1.22.3 darwin/arm64

$ node --version
v20.12.2

$ python -V
Python 3.11.9

$ exit

$ go; node; python
zsh: command not found: go
zsh: command not found: node
zsh: command not found: python
```

We were able to install golang, node, and python instantly.

We skipped the setup docs, we didn't edit our shell profile, and we didn't have to worry about `PATH` or other environment variables.

Best of all, we didn't pollute the user environment. Everything is gone once you `exit` or `Ctrl-D` the shell.

Other benefits?

Nix builds packages in isolation from each other. This ensures that they are reproducible and don't have undeclared dependencies, so if a package works on one machine, it will also work on another.

Nix makes it trivial to share development and build environments for your projects, regardless of what programming languages and tools you're using.

## Install

The easiest way to install Nix (Linux, macOS, WSL2) is to use [The Determinate Nix Installer](https://zero-to-nix.com/concepts/nix-installer).

It installs Nix with flake support and the unified CLI feature already enabled. It also stores a receipt for the install process to allow for a clean uninstall.

Note: Directions to upgrade or uninstall can be found in their [GitHub repo](https://github.com/DeterminateSystems/nix-installer).

## Run Programs Directly

We've already seen how a shell environment can be created with `nix shell`.

With `nix run` we can skip creating the shell and run programs directly.

```bash
$ nix run nixpkgs#cowsay Hello, Nix!

$ nix run nixpkgs#lolcat -- --help

$ nix run nixpkgs#fortune | nix run nixpkgs#lolcat
```

## Reproducible Scripts

A trivial script with non-trivial dependencies.

```bash
#! /bin/bash

curl https://github.com/NixOS/nixpkgs/releases.atom | xml2json | jq .
```

This script fetches XML content from a URL, converts it to JSON, and formats it for better readability.

It requires curl, xml2json, jq, and bash. If any of these dependencies are not present on the system running the script, it will fail partially or altogether.

With Nix, we can declare all dependencies explicitly, and produce a script that will always run on any machine that supports Nix.

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

In this example `--override-flake` was used to specify a git commit hash of the Nixpkgs repository. This ensures that the script will always run with the exact same package versions, everywhere.

Notice that `nix shell` was only specified once when using multiple nix shell shebangs.

## Declarative Shell Environments

We can create a file that defines an environment. This can be shared with anyone to recreate the same environment on a different machine.

Create a `flake.nix` file:

```
{
  description = "A fun shell environment";

  # Pin to a specific commit for reproducibility
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/44072e24566c5bcc0b7aa9178a0104f4cfffab19";

  inputs.flake-utils.url = "github:numtide/flake-utils";

  outputs = { self, nixpkgs, flake-utils }:

    flake-utils.lib.eachDefaultSystem (system:

      let

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

Enter the environment by running `nix develop` in the same directory as `flake.nix`.

Note: If the directory you're working in is a git repository you may get an error indicating `flake.nix` doesn't exist. Stage the file with `git add` to get nix to recognize it. I don't know. It's weird like that.

If you make changes to `flake.nix` just `exit` or `Ctrl-D` to exit the environment and restart it with `nix develop`.

## More on Versioning

Without pinning or locking your tools and dependencies to specific versions you'll eventually hit the point of development where a bug is affecting others but somehow it "works on my machine". It's often very unpleasant and very difficult to debug.

We've already seen a few examples of version pinning by using the git commit hash of the Nixpkgs repository. These hashes can be found at [status.nixos.org](https://status.nixos.org).

Let's recap the different ways to version pin.

## Search for Packages

If you can think of it, there's probably a Nix package of it.

Go to <https://search.nixos.org/packages> or use the command line to search for packages.

```bash
$ nix search nixpkgs neovim
```

#### Wrapped vs Unwrapped

The key difference between wrapped and unwrapped packages in NixOS is that wrapped packages are configured to work seamlessly within the NixOS environment, while unwrapped packages are the raw, unmodified versions.

In most cases, users should install the wrapped version of a package, as it is preconfigured to work correctly on NixOS. The unwrapped version is primarily used when further customization or overriding of the package is required, as it serves as the base for creating a new wrapped derivation with additional modifications.

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
