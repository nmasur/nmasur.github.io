---
title: "Package Language Servers Into Neovim Using Nix"
date: 2024-11-16T21:25:05-07:00
tags: [ ]
draft: false
---

The introduction of the [language server
protocol](https://microsoft.github.io/language-server-protocol/) has made it
easier to use a variety of different text editors for programming with hints
and autocompletion instead of requiring more complex IDEs.

Unfortunately, it also means that most text editors don't come bundled with
their own language server and instead expect it to be installed separately on
your machine. While there are some tools that intend to make it easier to
install language servers from inside Neovim, such as
[mason.nvim](https://github.com/williamboman/mason.nvim) or
[lazy-lsp.nvim](https://github.com/dundalek/lazy-lsp.nvim?tab=readme-ov-file),
I prefer the guarantee that the Neovim package contains everything that I need
to use the LSP at install time.

# nix2vim

The flake [nix2vim](https://github.com/gytis-ivaskevicius/nix2vim) is a
framework for building a Neovim package and Lua configuration with Nix. It
comes with a function called
[neovimBuilder](https://github.com/gytis-ivaskevicius/nix2vim/blob/master/lib/neovim-builder.nix)
that can be paired with imported modules to add plugins, `setup` and `use`
statements, `vim` options settings all in the Nix language for translation
into Lua. You can also add `lua` sections if you need to bail out to normal
Lua code.

One of the benefits of this is that you can use Nix expressions inside of the
Neovim configuration, which means that nixpkgs can be included directly and
rendered into Nix store paths when the final output is transpiled to Lua.

# LSP Configuration

It's easy enough to configure the LSP and include its package at the same time. In [my config](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/common/neovim/config/lsp.nix), I include the language server program alongside its settings:

```nix
use.lspconfig.lua_ls.setup = dsl.callWith {
  settings = {
    Lua = {
      diagnostics = {
        globals = [
          "vim"
          "hs"
        ];
      };
    };
  };
  capabilities = dsl.rawLua "require('cmp_nvim_lsp').default_capabilities()";
  cmd = [ "${pkgs.lua-language-server}/bin/lua-language-server" ];
};
```

In this case, I'm using `dsl` functions to mimic certain Lua calls and to
bail out for more complex code. For the actual language server command, I'm
identifying the store path to the [nixpkgs
object](https://github.com/NixOS/nixpkgs/blob/9599296566c88826a42398af0981fe065d577aa0/pkgs/development/tools/language-servers/lua-language-server/default.nix)
and including the program directly.

When the Neovim package is built, the language server becomes a dependency, so
therefore you have a guarantee that the LSP is ready when you launch Neovim.
You can also imagine calling the Neovim package [as a
function](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/common/neovim/package/default.nix)
and passing the specific version of the language server based on the needs of
the project, and/or including only the language servers you need for those
projects with a project-specific Neovim.
