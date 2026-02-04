# Tutorial Distrobox no Void Linux

Distrobox allows you to use other Linux distributions within your Void Linux
without compromising the base system (host).
The logic is as usual: **clean system, isolated tests, zero workarounds**.

---

## What is the proposal here

Install distrobox on Void Linux, and use containers to run other distributions safely.

This eliminates the risk of:
- break system dependencies
- pollute the host with packages from other distros
- turn Void into Frankenstein

Note: this guide assumes that Void Linux is already installed.

---

## First of all: chililinux repository

The `distrobox` package does not exist in the official Void repositories.
But it was packaged by the VoidLinuxBR Community, so it is necessary to add the chililinux repository (official Void mirror in Brazil - <https://xmirror.voidlinux.org/>).

Execute **exactly** the commands below:
```bash
sudo sh -c "{
  echo 'repository=https://repo-fastly.voidlinux.org/current'
  echo 'repository=https://void.chililinux.com/voidlinux/current'
} > /etc/xbps.d/00-repository-main.conf"
```

---

## Updating the base system

Before installing anything, make sure your system is up to date:

```bash
sudo xbps-install -Syu xbps
sudo xbps-install -Syu libssh2 xtools
sudo xbps-install -Suy
xcheckrestart
```
If `xcheckrestart` indicates restart, restart.

---

## Installing Distrobox and dependencies

Now, install the necessary packages:

```bash
sudo xbps-install -Syf voidbr-distrobox podman docker crun
```

Important:
After installing `crun`, it is mandatory to restart the system:

```bash
sudo reboot
```

---

## About distribution compatibility

Not every distro works well in containers.
Before choosing, consult the official list:

https://distrobox.it/compatibility/#containers-distros

This avoids wasted time and headaches.

---

## Creating the first container (Debian)

As an example, Debian Testing will be used.

```bash
distrobox create -Y --name debian --image docker.io/library/debian:testing
```

What's happening here:
- `distrobox create` creates the container
- `-Y` avoids interactive questions
- `--name` defines the name of the container
- `--image` defines the base image

To see all available options:

```bash
distrobox --help
```

---

## Entering the container

After pulling the image, enter the container:

```bash
distrobox enter debian
```

Within Debian, usage is normal:

```bash
sudo apt update
sudo apt upgrade
sudo apt autoremove
sudo apt install firefox
```

You are literally inside another distro.

---

## Executing commands without entering the container

You can also run commands directly from the host.

Example: installing Firefox on Debian without going into it:

```bash
distrobox enter debian -- sudo apt install -y firefox-esr-l10n-pt-br
```

Practical, fast and traditional.

---

## Exporting applications to the host system

Distrobox allows you to export applications from the container
to the VoidLinuxBR graphical menu.

Example: export Firefox from the Debian container:

```bash
distrobox enter debian -- distrobox-export --app firefox
```

The application will appear in the graphical environment menu
as if he were native.

---

## Updating all containers

To update all containers at once,
execute no host:

```bash
distrobox-upgrade --all -v
```

---

## Listing existing containers

To see all created containers:

```bash
distrobox list
```

Name, status and image used are displayed.

---

## Stopping a container

If you just need to stop the container:

```bash
distrobox stop debian
```

---

## Removing a container

To remove the Distrobox container:

```bash
distrobox rm debian
```

If you want to also remove the Podman image:

```bash
podman rmi -f [IMAGE ID]
```

---

## Final observations

- Use containers for testing, not for cluttering the host
- Adjust names and images as needed
- Always consult the official documentation:
https://distrobox.it
- Test first on VM or lab machine

Distrobox is a tool for those who like control,
insulation and well-maintained system.

---

## 📜 Credits

Created by: Robson Nakane <theblizzard1983@hotmail.com>
Community: Void Linux Brazil <https://github.com/voidlinuxbr>
Distribution: Chili Linux <https://chililinux.com>
Distribution: VoidBR <https://github.com/voidlinuxbr>

---

## ⚖️ Disclaimer (Legal Notice)

THIS SOFTWARE/TUTORIAL IS PROVIDED "AS IS" WITH ABSOLUTELY NO WARRANTY
OF ANY KIND, WHETHER EXPRESS OR IMPLIED, INCLUDING, BUT NOT LIMITED TO,
WARRANTIES OF MERCHANTABILITY OR FITNESS FOR A PARTICULAR PURPOSE.

THE USE OF THIS SOFTWARE IS THE ENTIRE RESPONSIBILITY OF THE USER.

AT NO TIME SHALL THE AUTHOR OR CONTRIBUTORS BE RESPONSIBLE FOR
ANY DAMAGE, LOSS OF DATA OR SYSTEM FAILURE ARISING OUT OF THE USE
OF THIS PROGRAM.

---

Copyright (C) 2026 Robson Nakane
