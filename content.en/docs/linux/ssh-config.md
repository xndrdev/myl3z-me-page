---
title: SSH Config and Key Login
weight: 40
---

# SSH Config and Key Login

Without configuration every connection is the same typing exercise:

```sh
ssh -i ~/.ssh/id_rsa xander@10.10.10.3
```

Address, user and key do not belong on the command line but in the client configuration at
`~/.ssh/config`. It states per host what the connection should look like — after that the
alias is enough:

```sh
ssh dns01
```

## The entry

```text
Host dns01
    HostName 10.10.10.3
    IdentityFile /home/xander/.ssh/id_rsa
    IdentitiesOnly yes
```

| Line | Meaning |
|------|---------|
| `Host dns01` | the name you type on the command line. Freely chosen and independent of the real hostname; everything up to the next `Host` belongs to this block |
| `HostName 10.10.10.3` | where the connection actually goes — IP or DNS name |
| `IdentityFile` | the **private** key for this connection. Its public counterpart (`.pub`) lives on the server |
| `IdentitiesOnly yes` | offer this key and nothing else |

The block is a set of defaults, not a connection in itself. It applies to anything matching
the alias — including `scp`, `rsync` and `git`, which all use the same SSH client:

```sh
scp report.txt dns01:/tmp/
rsync -a ./config/ dns01:/etc/pihole/
```

> [!NOTE]
> The entry sets no `User`, so SSH falls back to the local username — `xander` here. If the
> account on the server has a different name, add a `User <name>` line, otherwise the login
> fails with a name that does not exist there. For a non-standard port, add `Port 2222`.

## Why IdentitiesOnly

Without that option the client offers the server everything it can find, one after another:
every key in the agent, plus the default filenames in `~/.ssh/`. The `IdentityFile` you
specified is then just one candidate among several.

The problem is on the server side: `sshd` allows only a limited number of attempts per
connection (`MaxAuthTries`, 6 by default). With many keys in the agent you get rejected before
the right one is ever tried:

```text
Received disconnect from 10.10.10.3 port 22:2: Too many authentication failures
```

`IdentitiesOnly yes` restricts the client to the keys named in the block. One attempt, the
right one, done.

## Put the public key on the server

The server has to know the public key. One command handles it — creating directory and file
with correct permissions if needed, and never appending a duplicate:

```sh
ssh-copy-id -i ~/.ssh/id_rsa.pub dns01
```

The first time still goes through the account password. After that:

```sh
ssh dns01        # no password prompt
```

Doing it by hand is the same thing, longer — the contents of the `.pub` file go as one line
into the target user's `~/.ssh/authorized_keys`:

```sh
cat ~/.ssh/id_rsa.pub | ssh dns01 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'
```

## Permissions

SSH refuses to work with overly permissive modes — on both ends, and without leniency. A key
others can read counts as compromised:

| Path | Mode | Side |
|------|------|------|
| `~/.ssh` | `700` | both |
| `~/.ssh/config` | `600` | client |
| `~/.ssh/id_rsa` | `600` | client |
| `~/.ssh/id_rsa.pub` | `644` | client |
| `~/.ssh/authorized_keys` | `600` | server |

```sh
chmod 700 ~/.ssh && chmod 600 ~/.ssh/config ~/.ssh/id_rsa
```

## Verify

Which options actually apply to a host, with all defaults resolved:

```sh
ssh -G dns01
```

If the login fails, verbose mode shows which keys were offered and what the server did with
them:

```sh
ssh -v dns01
```

> [!WARNING]
> Inside the config the **first** value found for each keyword wins, not the last. Specific
> hosts therefore belong at the top, a general `Host *` block at the end of the file.

## Other useful options

| Option | Purpose |
|--------|---------|
| `User <name>` | different username on the server |
| `Port <n>` | different port |
| `ServerAliveInterval 60` | keeps the connection alive instead of letting it drop while idle |
| `ForwardAgent yes` | forwards the local agent — only for hosts you trust |
| `ProxyJump <host>` | connect through a jump host when the target is not directly reachable |

> [!NOTE]
> `id_rsa` still works as long as the key is at least 3072 bits — what OpenSSH 8.8 disabled is
> the old SHA-1 signature `ssh-rsa`, not modern RSA signatures. For new keys
> `ssh-keygen -t ed25519` is the current recommendation: shorter, faster, and no decision
> about key length.
