---
title: "Using Hammerspoon as a macOS Window Manager"
date: 2024-11-15T18:52:53-07:00
draft: false
---

Before macOS got built-in [window tiling
features](https://support.apple.com/en-euro/guide/mac-help/mchlef287e5d/15.0/mac/15.0)
there weren't too many great ways to control window panes.

You either had to pay for an app like [Rectangle](https://rectangleapp.com/) or
[Magnet](https://magnet.crowdcafe.com/) and remember to download it whenever
you setup a new computer, or use something like
[Tiles](https://freemacsoft.net/tiles/), or a full-featured window manager like
[Yabai](https://github.com/koekeishiya/yabai).

However, since I was already using [Hammerspoon](https://www.hammerspoon.org/)
for other automation, I figured it would be easiest to just repurpose it as a
window management tool, partially inspired by [this
video](https://youtu.be/JZDt-PRq0uo?si=bo7xwzKH68G72J_Q&t=2507).

# Hammerspoon

[Hammerspoon](https://www.hammerspoon.org/) is an engine that allows you to
write basic Lua scripts to interact with the UI and other elements of macOS. I
use it to develop my own special keyboard shortcuts, such as:

- A set of [quick-launch
keys](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/Launcher.spoon/init.lua)
for various applications.
- Convert my `CAPSLOCK` key into a double `CTRL`/`ESC` [utility
key](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/ControlEscape.spoon/init.lua).
- Dismiss alert notifications [with a
keypress](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/DismissAlerts.spoon/init.lua)
after I've read them.

# Window Management

Changing the window settings is pretty simple. After binding the modifiers and
key with `hs.hotkey.bind({ "alt", "ctrl", "cmd" }, "m", someFunction)`, you
can then get the current focused window with `hs.window.focusedWindow()` and
frame with `win:frame()`. If you want to maximize the window, use
`win:screen():fullFrame()` to get the max size and set the frame to that
size. Divide the max width by 2 and move the origin to the left or center to
half-maximize (tile to one side).

In my setup, I have the following keybinds:

- `CMD + ALT + CTRL + n`: [Move window to next screen](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/MoveWindow.spoon/init.lua#L15).
- `CMD + ALT + CTRL + b`: [Move window to previous screen](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/MoveWindow.spoon/init.lua#L26).
- `CMD + ALT + CTRL + m`: [Maximize window on current screen](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/MoveWindow.spoon/init.lua#L33).
- `CMD + ALT + CTRL + o`: [Half-maximize to the right](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/MoveWindow.spoon/init.lua#L53).
- `CMD + ALT + CTRL + u`: [Half-maximize to the left](https://github.com/nmasur/dotfiles/blob/1022a3998f06819d6b7987d312d62bb7c8bbea15/modules/darwin/hammerspoon/Spoons/MoveWindow.spoon/init.lua#L67).

