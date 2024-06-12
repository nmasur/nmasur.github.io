---
title: Caddy Cloudflare DNS on NixOS
date: "2023-09-10T13:12:26Z"
tags: [  ]
draft: false
---

If you serve Caddy from behind Cloudflare and enforce, you may run into an
issue with auto-provisioning ACME SSL certs.

By default, Caddy uses the HTTP validation method which requires no setup but
does involve serving traffic on HTTP / port 80, including through Cloudflare. I
prefer Cloudflare to enforce using HTTPS / port 443 everywhere, which means
that every time I want to validate a new cert I have to temporarily disable the
HTTPS enforcement and/or set the SSL settings to "flexible" instead of
"strict".

For hands-free automation, this is a non-starter!

The solution is to switch from HTTP validation to TXT validation, which doesn't
require serving HTTP but _does_ require updating your DNS records to receive an
SSL cert.

## Updating DNS with Caddy

Caddy has a [plugin](https://github.com/caddy-dns/cloudflare) to connect to
Cloudflare's DNS service and automatically manage the records for cert
validation, but it's not packaged in NixOS.

So the next step is to recompile Caddy with the Cloudflare DNS plugin using an
overlay. Here's what my overlay looks like:

```nix
_final: prev:

let

  plugins = [ "github.com/caddy-dns/cloudflare" ];
  goImports =
    prev.lib.flip prev.lib.concatMapStrings plugins (pkg: "   _ \"${pkg}\"\n");
  goGets = prev.lib.flip prev.lib.concatMapStrings plugins
    (pkg: "go get ${pkg}\n      ");
  main = ''
    package main
    import (
    	caddycmd "github.com/caddyserver/caddy/v2/cmd"
    	_ "github.com/caddyserver/caddy/v2/modules/standard"
    ${goImports}
    )
    func main() {
    	caddycmd.Main()
    }
  '';

in {
  caddy-cloudflare = prev.buildGo120Module {
    pname = "caddy-cloudflare";
    version = prev.caddy.version;
    runVend = true;

    subPackages = [ "cmd/caddy" ];

    src = prev.caddy.src;

    vendorSha256 = "sha256:mwIsWJYKuEZpOU38qZOG1LEh4QpK4EO0/8l4UGsroU8=";

    overrideModAttrs = (_: {
      preBuild = ''
        echo '${main}' > cmd/caddy/main.go
        ${goGets}
      '';
      postInstall = "cp go.sum go.mod $out/ && ls $out/";
    });

    postPatch = ''
      echo '${main}' > cmd/caddy/main.go
      cat cmd/caddy/main.go
    '';

    postConfigure = ''
      cp vendor/go.sum ./
      cp vendor/go.mod ./
    '';

    meta = with prev.lib; {
      homepage = "https://caddyserver.com";
      description =
        "Fast, cross-platform HTTP/2 web server with automatic HTTPS";
      license = licenses.asl20;
      maintainers = with maintainers; [ Br1ght0ne techknowlogick ];
    };
  };
}
```

The primary change we're making here is pulling in the Go module for the
plugin, and updating the resulting `vendorSha256`.

Once we have made our overlay (I gave it a separate `pname` above called
`caddy-cloudflare`) we can use it in our Caddy settings.

```nix
services.caddy.package = pkgs.caddy-cloudflare;
```

You could also just override the `caddy` package and not bother to update
`services.caddy.package`, but I like that 1) I have to choose to use it
explicitly, and 2) I can conditionally enable this version of the package only
for machines that use Cloudflare, in case I 

In my [previous post](./nextcloud-caddy-nixos.md), I explained that I use the
JSON config format instead of a traditional Caddyfile, which gives me more
control over options as well as the ability to merge together routes from
different services.

```nix
services.caddy = {
  adapter = "''"; # Required to enable JSON
  configFile = pkgs.writeText "Caddyfile" (builtins.toJSON {
    apps.http.servers.main = {
      listen = [ ":443" ];
      routes = config.caddy.routes;
    };
  });
};
```

Adding the following into the JSON configuration will enable the new Cloudflare
DNS plugin as an ACME DNS provider, which will automatically create the
necessary records to solve the LetsEncrypt verification challenges.

```nix
apps.tls.automation.policies = [{
  issuers = [{
    module = "acme";
    challenges = {
      dns = {
        provider = {
          name = "cloudflare";
          api_token = "{env.CF_API_TOKEN}";
        };
        resolvers = [ "1.1.1.1" ];
      };
    };
}];
```

However, we'll need an API token for our Cloudflare account in order to make
those changes to our DNS as securely as possible.

## Setting the Cloudflare API Key

Check the [Cloudflare
docs](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
for how to generate an API token from your profile. For our purposes, the token
should have the following permissions:

- Zone permission: DNS -> Edit
- Zone resources: Include specific zone / all zones -> (Your zone)

You should take this API token and write it out into a secret text file with
the following format:

```
CF_API_TOKEN=yourtokenhere
```

This is because we will load it as an environment file, so it must be a file
that a shell can evaluate.

Next, you'll need to securely deploy your new secret file to a location on your
disk. You could use one of the following methods:

- Manually copy the text file to a path on your target system.
- Deploy using [agenix](https://github.com/ryantm/agenix) or
[sops-nix](https://github.com/Mic92/sops-nix).
- Build [your own](https://xeiaso.net/blog/nixos-encrypted-secrets-2021-01-20)
secrets management tooling.

In any case, make sure the file permissions are set to be read by `caddy:caddy`
(or whatever user is running the `caddy` service) with 0400 access.

Then you'll need to set the following to load the secret as an environment file
for the systemd service:

```nix
systemd.services.caddy.serviceConfig.EnvironmentFile = /path/to/your/secret
```

I've also found that I needed to enable `CAP_NET_BIND_SERVICE` to grant the
systemd service the ability to bind to lower ports (such as 443):

```nix
systemd.services.caddy.serviceConfig.AmbientCapabilities = "CAP_NET_BIND_SERVICE";
```

At this point, your Caddy service should be able to modify DNS records and
receive LetsEncrypt certificates automatically! Take a look at [my Cloudflare
configuration
settings](https://github.com/nmasur/dotfiles/blob/d2b1d9528136032788d5cfb59f1f36f73b08d7ec/modules/nixos/services/cloudflare.nix)
and [my Caddy
overlay](https://github.com/nmasur/dotfiles/blob/d2b1d9528136032788d5cfb59f1f36f73b08d7ec/overlays/caddy.nix)
as examples.

