---
title: NextCloud Behind Caddy on NixOS
date: "2023-08-29T11:56:26Z"
tags: [  ]
draft: false
---

NextCloud can technically run behind [any reverse
proxy](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/reverse_proxy_configuration.html)
but their documentation is sparse and don't include full examples.

The NextCloud [NixOS
module](https://github.com/NixOS/nixpkgs/blob/nixos-unstable/nixos/modules/services/web-apps/nextcloud.nix)
automatically enables nginx by default. Since I run Caddy bound to 443 for the
rest of my services, I didn't want to have to proxy to a local nginx service
just to hit NextCloud. They have [some
documentation](https://nixos.org/manual/nixos/stable/index.html#module-services-nextcloud-httpd)
for using Apache httpd, but I had to adjust for Caddy.

[Here](https://github.com/nmasur/dotfiles/blob/67251a6d8d67805b6bfa15016d6960b215550a47/modules/nixos/services/nextcloud.nix)
is my full configuration on my system, but I'll break it down to explain it in
detail.

First, after enabling NextCloud, you should at least have the following in your
configuration.

```nix
services.nextcloud.config.trustedProxies = [ "127.0.0.1" ];
```

This allows any local reverse proxy (including nginx, Caddy, etc.) to connect
to the NextCloud service.

Next, we disable nginx, since enabling the NextCloud module will enable nginx
by default:

```nix
services.nginx.enable = false;
```

We have to make sure that Caddy's user will have access to talk to the
NextCloud PHP service and access to the files served by NextCloud.

```nix
services.phpfpm.pools.nextcloud.settings = {
  "listen.owner" = config.services.caddy.user;
  "listen.group" = config.services.caddy.group;
};
users.users.caddy.extraGroups = [ "nextcloud" ];
```

(Your users and groups for Caddy and NextCloud may be different, so keep that
in mind).

The next step is where things get serious because we have to generate Caddy's
config file. You could use the Caddyfile format for brevity, but I have chosen
to generate a JSON file for a few reasons:

1. The format is easier to organize and manipulate from Nix expressions. I am
   serving multiple sites from the same Caddy service, so combining all routes
into a single JSON file is cleanest for me with Nix.
2. The JSON format has [more options](https://caddyserver.com/docs/json/) that
   can give you more control over settings, especially when they differ between
routes or hostnames.

My general Caddy setup looks like this — first, I establish a list of sets
option called `caddy.routes`, so I can add them from services in multiple
places:

```nix
options.caddy.routes = lib.mkOption {
  type = lib.types.listOf lib.types.attrs;
  description = "Caddy JSON routes for http servers";
  default = [ ];
};
```

Then I use this for my base Caddy config, sticking the routes into
`apps.http.servers.<something>.routes`:

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

The routes themselves are mostly written like the following, and can be added
anywhere in your config (even multiple times, which will merge into one list
automatically):

```nix
caddy.routes = [{
  match  = [{ host = [ "my.host.name" ]; }];
  handle = [{
    handler   = "reverse_proxy";
    upstreams = [{
      dial = "localhost:1234";
    }]
  }]
}]
```

In the above example, Caddy will match on the public hostname of your service,
and then act as a reverse-proxy by connecting to the local port `1234` on your
machine where the service is running. This is how I setup most of my web
services.

With NextCloud, things will be a little different. The best Caddy reference I
found was [this Reddit
comment](https://www.reddit.com/r/NextCloud/comments/gn7fdl/looking_for_caddy_v2_sample_config_for_nextcloud/frjj50c/)
with a working Caddyfile. However, we need to translate it for our JSON
configuration.

Step one is to set a single route with everything inside the handle attribute.

```nix
caddy.routes = [{
  match = [{ host = [ "your.nextcloud.domain" ]; }];
  handle = [{
    handler = "subroute";
    routes = [
      # ... Everything goes in here
    ];
  }];
  terminal = true;
}];
```

All of our NextCloud routes I describe below will be _subroutes_ of the
original handle, which matches on the hostname of our NextCloud service.

The first subroute sets variables and headers for the other subroutes. The
variable is required for working with apps, and the header enforces the use of
HTTPS on the browser.

```nix
{
  handle = [
    {
      handler = "vars";
      root = config.services.nextcloud.package;
    }
    {
      handler = "headers";
      response.set.Strict-Transport-Security =
        [ "max-age=31536000;" ];
    }
  ];
}
```

The next subroute makes use of this `root` variable to serve content from the
`/nix-apps/` and `/store-apps/` directories, which are the built-in and
marketplace plugins for NextCloud. You will likely need this even if you are
just using NextCloud out of the box.

```nix
{
  match = [{ path = [ "/nix-apps*" "/store-apps*" ]; }];
  handle = [{
    handler = "vars";
    root = config.services.nextcloud.home;
  }];
}
```

Next, we have a subroute for managing CardDAV and CalDAV traffic (for contacts,
calendars). These URLs are redirected to a specific PHP location explained in
the [official
docs](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/reverse_proxy_configuration.html#service-discovery).

```nix
{
  match =
    [{ path = [ "/.well-known/carddav" "/.well-known/caldav" ]; }];
  handle = [{
    handler = "static_response";
    headers = { Location = [ "/remote.php/dav" ]; };
    status_code = 301;
  }];
}
```

The next subroute is used to deny access to sensitive or internal-only files by
returning a 404 error response.

```nix
{
  match = [{
    path = [
      "/.htaccess"
      "/data/*"
      "/config/*"
      "/db_structure"
      "/.xml"
      "/README"
      "/3rdparty/*"
      "/lib/*"
      "/templates/*"
      "/occ"
      "/console.php"
    ];
  }];
  handle = [{
    handler = "static_response";
    status_code = 404;
  }];
}
```

We also need a subroute to redirect `/index.php` requests to the base homepage
instead of trying to use a PHP file that doesn't exist.

```nix
{
  match = [{
    file = { try_files = [ "{http.request.uri.path}/index.php" ]; };
    not = [{ path = [ "*/" ]; }];
  }];
  handle = [{
    handler = "static_response";
    headers = { Location = [ "{http.request.orig_uri.path}/" ]; };
    status_code = 308;
  }];
}
```

For incoming requests, we rewrite their paths to be relative to the current
directory.

```nix
{
  match = [{
    file = {
      split_path = [ ".php" ];
      try_files = [
        "{http.request.uri.path}"
        "{http.request.uri.path}/index.php"
        "index.php"
      ];
    };
  }];
  handle = [{
    handler = "rewrite";
    uri = "{http.matchers.file.relative}";
  }];
}
```

Of course, we eventually have to send PHP traffic to the actual PHP service,
which in this case is not listening on a local port by default, but on a UNIX
socket.

```nix
{
  match = [{ path = [ "*.php" ]; }];
  handle = [{
    handler = "reverse_proxy";
    transport = {
      protocol = "fastcgi";
      split_path = [ ".php" ];
    };
    upstreams = [{ dial = "unix//run/phpfpm/nextcloud.sock"; }];
  }];
}
```

Finally, all other requests are served simply as static files:

```nix
{ handle = [{ handler = "file_server"; }]; }
```

That's it! Now Caddy will serve as the ingress to NextCloud directly.
