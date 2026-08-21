---
title: Installing ddev
weight: 20
---

# Installing ddev

Once per machine. After that, every further project is a matter of minutes.

Exactly two things are needed: **Docker** and **ddev**. Everything else — PHP, Composer, Node,
MySQL — ddev ships in containers.

## Step 1: Docker

ddev needs a Docker provider. Which one barely matters, as long as it runs.

{{< tabs "docker" >}}
{{% tab "Linux" %}}
Docker CE straight from the distribution, the quickest option:

```bash
# Arch
sudo pacman -S docker docker-buildx
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"

# Debian / Ubuntu
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
```

After `usermod`, log out and back in once, otherwise the permissions on the Docker socket are
missing.
{{% /tab %}}
{{% tab "macOS" %}}
Docker Desktop, OrbStack or Rancher Desktop. On Apple Silicon, OrbStack is noticeably lighter
on resources:

```bash
brew install --cask orbstack
# or
brew install --cask docker
```
{{% /tab %}}
{{% tab "Windows" %}}
Via WSL2 — Docker runs *inside* the Linux distribution, not on Windows:

```powershell
wsl --install --no-distribution
wsl --update
wsl --install Ubuntu --name DDEV
```

Then continue inside the WSL2 distribution as you would on Linux. Projects have to live in the
Linux filesystem (`~/projects/`), not under `/mnt/c/` — otherwise file access is unusably
slow.
{{% /tab %}}
{{< /tabs >}}

Check that Docker is running:

```bash
docker ps
```

An empty table instead of an error message means it works.

## Step 2: ddev

{{< tabs "ddev" >}}
{{% tab "Arch" %}}
```bash
yay -S ddev-bin
```

The variant used on this machine.
{{% /tab %}}
{{% tab "Debian / Ubuntu" %}}
```bash
sudo apt-get update && sudo apt-get install -y curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://pkg.ddev.com/apt/gpg.key | sudo tee /etc/apt/keyrings/ddev.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/ddev.asc
printf "Types: deb\nURIs: https://pkg.ddev.com/apt/\nSuites: *\nComponents: *\nSigned-By: /etc/apt/keyrings/ddev.asc\n" | sudo tee /etc/apt/sources.list.d/ddev.sources >/dev/null
sudo apt-get update && sudo apt-get install -y ddev
```
{{% /tab %}}
{{% tab "macOS" %}}
```bash
brew install ddev/ddev/ddev
```
{{% /tab %}}
{{% tab "Anything else" %}}
The install script works on every Linux variant and on macOS:

```bash
curl -fsSL https://ddev.com/install.sh | bash
```

For a specific version:

```bash
curl -fsSL https://ddev.com/install.sh | bash -s v1.25.2
```
{{% /tab %}}
{{< /tabs >}}

## Step 3: Certificates

For `https://project.ddev.site` to work without a browser warning, the local certificate
authority has to be registered once:

```bash
mkcert -install
```

> [!NOTE]
> Skipping this still leaves a working shop — but every request starts with a certificate
> warning, and the administration behaves differently in edge cases. Run it once, never think
> about it again.

## Verifying

```bash
ddev version
```

For reference, the state these notes are based on:

```text
DDEV version      v1.25.2
docker            29.5.1
docker-compose    v5.1.3
docker-platform   linux-docker
```

## Updating

The same way it was installed:

```bash
yay -S ddev-bin                    # Arch
sudo apt-get update && sudo apt-get install --only-upgrade ddev   # Debian/Ubuntu
brew upgrade ddev/ddev/ddev        # macOS
```

After a ddev update the project containers have to be rebuilt — ddev points this out on the
next `ddev start`. When in doubt:

```bash
ddev restart
```

> [!WARNING]
> Larger version jumps can change the container images. Before updating inside a project with
> data worth keeping: take a `ddev snapshot`.

## Next

That is the groundwork: [Setting Up Shopware 6]({{< relref "shopware-setup" >}}).
