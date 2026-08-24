# Forge
Forge - The Universal Linux Package Manager - lets you install any package from any package repo, from source tarball to well-known package managers like Pacman, APT and DNF.

> [!CAUTION]
> Do not run "sudo rm -rf /" LOL 😂 
> (seriously, don't try this, I even gave the wrong command purposely)

# Installation

Clone this repo with the CLI command **git**:

```bash
git clone https://github.com/johnfg-archbtw/forge
```

> [!NOTE]
> If you are quite advanced in Linux, you may notice that there aren't any Makefile nor any compile script, and that's the point, this project is 100% Bash!

Then cd to the forge/ dir, and run the automated installation script, as root:

```bash
cd forge/
sudo ./install-forge
```

# Bit of History of Forge

This idea originated from Linux From Scratch (LFS), when I succesfully followed the whole book and booted into the OS. 

![Linux From Scratch Logo](https://www.linuxfromscratch.org/images/linux-from-scratch.png)

After installing internet drivers then WPA Supplicant and DHCP, I realized that writing every command after another finished — to build and install the package — took way too much, so the idea of creating an automatic package installer popped in my mind.

> [!NOTE]
> Before you say about Automated Linux From Scratch (ALFS), that project wasn't updated not even a single bit since 2017. 😭
>
>  I don't even know how to use that since there's no updated documentation that explains how to build that, and any Makefile there just fails. 

Then I created a little recipe-based PM (like Gentoo's Portage, that relies on ebuild files to build the packages sources), that I would still have to write every command inside the recipe files, but that is WAY better than having to wait for the commands to finish (that REALLY take that long once you enter to packages like GCC, LLVM or QT 6).

Later on, I got working a new subcommands for Forge that would fetch a package directly from BLFS homepage, then get specific HTML patterns to write the recipe files AUTOMATICALLY (damn that's really good). If you want to know how that works, see below. Spoiler: the subcommand name is sketch.

Then I added other LFS' books support to sketch, SLFS and LFS (much later I added GLFS, MLFS

> [!TIP]
> I use Arch btw
> ![Arch Linux Logo](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f9/Archlinux-logo-standard-version.svg/1280px-Archlinux-logo-standard-version.svg.png?utm_source=commons.wikimedia.org&utm_campaign=index&utm_content=thumbnail&_=20240926043220)



