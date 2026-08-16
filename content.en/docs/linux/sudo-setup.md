---
title: Adding sudo
weight: 50
---

# Adding sudo

A fresh Debian install may have no `sudo` at all — and which way it goes depends on a choice
made during installation whose consequence is never spelled out:

| Root password in the installer | Result |
|--------------------------------|--------|
| set | root is enabled, `sudo` is **not** installed |
| left empty | root is locked, `sudo` is installed and the first user is added to the `sudo` group |

So if you set a root password, you are met with `sudo: command not found` afterwards and have
to add it yourself.

## Install and enable the user

```sh
su -
apt install sudo
usermod -aG sudo xander
exit
```

`su -` with the dash matters: it starts a login shell with root's environment. Without the
dash your own `PATH` stays in effect, and it does not contain `/usr/sbin` — commands like
`usermod` then appear to be missing.

> [!WARNING]
> `usermod -aG` — the `-a` for *append* must not be left out. Without it the command replaces
> every secondary group the user has with the one given. Dropping out of groups like `audio`,
> `video` or `docker` that way only shows up later, as broken permissions.

## Log in again

Group membership is evaluated by the kernel at login. Nothing changes in a session that is
already open, not even after `usermod` — the shell carries the groups it started with. So log
out and reconnect:

```sh
exit
ssh dns01
```

Check what actually applies:

```sh
id                # sudo has to show up in the group list
sudo -v           # asks for your own password once, then stays quiet
sudo -l           # lists what is permitted
```

> [!NOTE]
> `newgrp sudo` pulls the group into the running shell without logging out. That is a stopgap
> for the moment — child processes of other sessions still will not see the group.

## Custom rules: sudoers.d, not sudoers

`/etc/sudoers` is not a file you open in an editor. A syntax error in it locks you out of
`sudo` entirely — and with root locked at the same time, the only way back is a live system.
`visudo` prevents exactly that: it checks the syntax on save and refuses to install broken
rules.

Custom rules do not belong in the main file but in one of their own next to it, which leaves
the main file untouched and upgradable:

```sh
sudo visudo -f /etc/sudoers.d/xander
```

Two rules apply to the filename: no dot and no tilde in it, or `sudo` silently ignores the
file. Permissions have to be `0440` — `visudo` sets them itself.

## The password prompt

By default `sudo` asks for *your own* password, not root's, and remembers it for fifteen
minutes per terminal.

```sh
sudo -k           # drop the timestamp right away
```

The duration can be set per user:

```text
Defaults:xander timestamp_timeout=30
```

Turning the prompt off entirely is common too:

```text
xander ALL=(ALL) NOPASSWD: ALL
```

> [!WARNING]
> On a server reachable by SSH key, the password prompt is the last hurdle between a leaked
> key and root. `NOPASSWD` removes it — defensible for individual, tightly scoped commands
> that some automation needs:
>
> ```text
> xander ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart pihole-FTL
> ```
>
> As a blanket grant for `ALL` it is not.
