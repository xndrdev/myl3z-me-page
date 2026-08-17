---
title: Base Packages After the Install
weight: 60
---

# Base Packages After the Install

A netinst installation without a desktop deliberately ships very little. That is the right
starting point for a server — it keeps the attack surface and the update workload small — but
it means tools only become noticeable when you need them. This page collects what was added
to the homelab machine after the fact.

Whether something is missing at all is answered by:

```sh
command -v curl        # path if present, otherwise empty and exit code 1
dpkg -l curl           # package status: ii = installed and configured
```

## curl

```sh
sudo apt install curl
```

`curl` fetches data over HTTP, HTTPS and a dozen other protocols and writes it to stdout by
default. On a server without a browser it is the standard tool for anything arriving over the
network: install scripts, API calls, the quick check whether a service answers at all.

| Option | Effect |
|--------|--------|
| `-s` | silent — no progress meter, but no error messages either |
| `-S` | show errors despite `-s`. Hence the usual pairing `-sS` |
| `-L` | follow redirects. Without it a moved URL returns only the redirect page |
| `-o file` / `-O` | write to a file instead of stdout; `-O` takes the name from the URL |
| `-I` | fetch only the response headers |
| `-f` | fail with exit code 22 on HTTP errors instead of printing the error page |
| `-m 10` | give up after 10 seconds |
| `-w '%{http_code}'` | print selected values from the response |

A service check you can drop into a script:

```sh
curl -sS -m 5 -o /dev/null -w '%{http_code}\n' http://10.10.10.3/
```

> [!NOTE]
> `wget` and `curl` overlap but are built differently: `wget` is made for downloading files and
> can mirror whole directories recursively, `curl` for the single request whose response gets
> processed further. On a Debian system `wget` is usually present already — for scripts that
> pipe output onward, `curl` is still the more natural fit.

### Running scripts from the internet

Plenty of projects install themselves with a one-liner in this shape:

```sh
curl -sSL https://install.example.net | bash
```

That is convenient and simultaneously the broadest statement of trust available: the contents
run unseen with the privileges of the calling shell, and what the server delivers can differ
between two runs. If you would rather not trust blindly, separate download from execution:

```sh
curl -sSL https://install.example.net -o install.sh
less install.sh
bash install.sh
```

The detour costs two minutes and turns an act of faith into a decision.

> [!WARNING]
> If an HTTPS call fails certificate verification, the `ca-certificates` package is usually
> missing or the system clock is wrong. `-k` disables the check and makes the call succeed —
> it fixes nothing and removes exactly the protection HTTPS is there for.

## What tends to be missing next

Not a recommendation to install things on spec, but a list of the packages most often reached
for on a fresh Debian server:

| Package | For |
|---------|-----|
| `git` | version configuration, clone repositories |
| `htop` | processes and load at a glance, nicer than `top` |
| `rsync` | transfer files and backups, changes only |
| `tmux` | sessions that survive a dropped connection |
| `ncdu` | find out what is eating the disk |
| `unattended-upgrades` | apply security updates automatically |

Each of these gets added here once it actually lands on the machine.
