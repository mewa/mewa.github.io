---
categories:
- XMonad
- Haskell
- Shell
comments: true
date: "2017-09-30T00:00:00Z"
title: Launching terminal emulator in current working directory in XMonad
---

Recently I got a bit fed up with my [XMonad](http://xmonad.org) configuration and decided to add some of the missing bits. After all I made the switch with *productivity* in mind so it would be silly to endure even the slightest tradeoffs. If you don't know what XMonad is -- it's an *extremely* customizable tiling window manager -- the default configuration is, however, pretty crude, so it doesn't really make sense to switch if you're not going to tweak it, even if just slightly.

To the point -- one of the things I was missing was the ability to open a new terminal emulator window in the same working directory as the one I had focused. I felt that existing solutions such as the [WorkspaceDir](https://hackage.haskell.org/package/xmonad-contrib-0.13/docs/XMonad-Layout-WorkspaceDir.html) extension were lacking and not exactly what I was looking for. And so I had to write one myself. Since I figured I couldn't be the only one in need I decided I'd share my snippet. 
<!--more-->

### The Code

The solution involves overriding the "new terminal" binding and changing the directory to focused window's `CWD`. In order to do so we have to extract it first -- depending on what OS you are sitting on the actual script may vary -- so just for the record, the script shown below was written for a Linux-based distro, although it should work on other operating systems with a proper `/proc` filesystem too.

1. Add key binding -- note, I'm using EZConfig here
```Haskell
("S-M-<Return>", mkTerm)
```

{:start="2"}
2. Add terminal launcher
```Haskell
mkTerm = withWindowSet launchTerminal

launchTerminal ws = case peek ws of
       Nothing -> runInTerm "" "$SHELL"
       Just xid -> terminalInCwd xid
```

{:start="3"}
3. Detect current working directory of focused window. You may need a different way of obtaining the CWD from PID if not on Linux (I was being lazy, sorry).
```Haskell
terminalInCwd xid = let
  hex = showHex xid " "
  shInCwd = "'cd $(readlink /proc/$(ps --ppid $(xprop -id 0x" ++ hex
    ++ "_NET_WM_PID | cut -d\" \" -f3) -o pid= | tr -d \" \")/cwd) && $SHELL'"
  in runInTerm "" $ "sh -c " ++ shInCwd
```

{:start="4"}
4. Last but not least -- make sure to add missing imports
```Haskell
import Numeric
import XMonad.Core
import XMonad.StackSet
import XMonad.Util.Run
```

### That's it

From now on you can let your fingers rest a bit more -- enjoy! And make sure to plan carefully what you're going to do with the time saved as well!

As usual, if you want to have a closer look at a config actually using this it's avaialable as part of my [dotfiles repo](https://github.com/mewa/dotfiles/blob/8728fd8/.xmonad/xmonad.hs#L23-L31).
