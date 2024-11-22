---
title: "Nix Turns Installing Software On Its Head"
date: 2024-11-21T17:04:18-07:00
draft: false
---

There are plenty of benefits to using the [Nix package
manager](https://nixos.org/), NixOS, and the rest of the Nix ecosystem.
However, for people unfamiliar with Nix, one of the fundamental paradigm shifts
is how Nix changes what it means to _install_ a software package.

# Traditional Package Management

A traditional package manager will install software programs at the user's whim
to a designated location on disk. These files are automatically exposed to your
environment via a `PATH` or simply placed in a known location to be consumed.

`/usr/bin/python3`

This leads to three problems:

1. There is no coupling between the environment and the package manager. So if
   a package is required by an environment or another package dependency, you
simply have to hope that you already installed it. Now you have an environment
that cannot guarantee that all of its dependencies are available.
2. Since you cannot link a package to its dependant environments, you can never
   be sure if it can be safely removed from your system. What happens if you
uninstall a package that is required by another program or environment? To
avoid the risk of breaking things, you usually end up with a growing pile of
bloat on your system.
3. For software placed in a standard location, you cannot install more than one
   version of the same package. Not only does this make managing multiple
environments painful, but sometimes two or more packages might be incompatible
if they both depend on different versions of the same libraries.

# Nix Architecture

These problems are moot with the Nix architecture. This is because Nix doesn't
really have the concept of "installing" a program. Instead, packages are built
(or downloaded from a cache) and placed in the Nix store, and packages are also
exposed to the environment.

## The Nix Store

The Nix store is simply a location on your system (`/nix/store/`) where
software that is built or downloaded by Nix resides. This is an immutable file
system that can only be altered by the Nix daemon. Every package sits in a
specific build directory with a unique name based on its package name and a
hash (such as `pwvfrkynkyhbnidikifjz5skxq892g5a-ffmpeg-headless-6.1.1-lib/`).

These programs can all be accessed with the full Nix store path, but doing so
would be incredibly tedious and impractical.

## Nix Environments

Every tool in the Nix ecosystem has a way of exposing packages from the Nix
store to the current environment, such as
[NixOS](https://nixos.org/manual/nixos/stable/#sec-package-management),
[Home-Manager](https://nix-community.github.io/home-manager/index.xhtml#ch-introduction),
[Nix profiles](https://nix.dev/manual/nix/2.17/package-management/profiles), or
[Nix shells](https://nixos.wiki/wiki/Development_environment_with_nix-shell).

With these tools, you use the Nix language to declare the environment's or
program's dependencies, and Nix will automatically resolve them to the path in
the Nix store and use them as shared libraries or place them (or symlink them)
onto your current `PATH` to be used like normal programs.

# The Cool Part

Here is the cool part: if you declare a program as a depencency for your
environment, your system, or another program, Nix automatically build (or
download) any dependencies that are missing from the Nix store.

This solves **problem 1**: you have a guarantee (after build time) that your
environment includes the dependencies you have declared. What about problems 2
and 3?

For **problem 2**, Nix doesn't worry about always keeping the exact set of packages
built in the Nix store. Nix keeps track of which dependencies are required by
your different environments, and when you choose to garbage collect it will
simply purge the existing packages that are not declared as dependencies of
anything else. You can set the garbage collector to run on a schedule; if you
accidentally purge too aggressively, the worst case scenario is that you have
to wait for Nix to rebuild or redownload a few packages.

For **problem 3**, the uniqueness of the Nix store paths allow you to reference
multiple packages with the same name. You can expose one version of a package
to one project shell environment and another version to a specific dependant
package, while another lives in your default shell. If they all happen to
require the exact same version, then great --- you get to save some space. But
otherwise, you don't ever have to think about it.

# Fundamental Benefits

The Nix ecosystem has a ton of benefits: declarative configuration, build
guarantees, Nixpkgs availability, rollbacks, lazy functional modules, temporary
environments, but fundamentally they all are sourced from rethinking, from the
ground up, what software installation means.

