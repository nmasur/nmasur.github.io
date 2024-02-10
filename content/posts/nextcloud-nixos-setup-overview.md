---
title: Nextcloud NixOS Setup Overview
date: "2024-02-06T20:30:53-05:00"
tags: [  ]
draft: false
---

[Here is my setup](https://github.com/nmasur/dotfiles/blob/e7cdfc1453649a757aab5f922ef95f69b3aea492/modules/nixos/services/nextcloud.nix).

I just use `services.nextcloud.enable = true` to run Nextcloud, since I don't run any of my services in containers or Flatpaks. Systemd feels most native to NixOS.

You get Nextcloud, Redis, and the MySQL database basically for free. I also use the Nix-packaged Nextcloud plugins and, for any missing, I [pin their releases to my flake](https://github.com/nmasur/dotfiles/blob/e7cdfc1453649a757aab5f922ef95f69b3aea492/flake.nix#L190-L214) and [add them as pkgs with an overlay](https://github.com/nmasur/dotfiles/blob/e7cdfc1453649a757aab5f922ef95f69b3aea492/overlays/nextcloud-apps.nix).

The biggest change I made was to [disable nginx](https://github.com/nmasur/dotfiles/blob/e7cdfc1453649a757aab5f922ef95f69b3aea492/modules/nixos/services/nextcloud.nix#L38-L39) because I already use [Caddy](https://github.com/nmasur/dotfiles/blob/e7cdfc1453649a757aab5f922ef95f69b3aea492/modules/nixos/services/caddy.nix) as my reverse proxy for all my services. This means I had to [write matching rules](./nextcloud-caddy-nixos.md) to convert the recommended nginx config into Caddy routes.

I wouldn't suggest that everyone use this exact approach, but I do recommend starting with the native NixOS services and tweaking their settings to fit your needs rather than immediately jumping to containers or Flatpaks.
