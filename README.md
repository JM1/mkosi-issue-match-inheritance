# Reproducer for mkosi's [Match] inheritance issue

[mkosi](https://github.com/systemd/mkosi) supports nested `mkosi.conf.d` directories in `mkosi.images` in order to allow
building multiple images for different distributions. The `[Match]` section of a `mkosi.conf` file in a directory like
[mkosi.images/system] or [mkosi.images/system/mkosi.conf.d/30-centos-fedora] shall apply to the entire directory, for
example including [mkosi.images/system/mkosi.conf.d] or
[mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf.d]. However, inheritance of the `[Match]` section to
nested directories breaks in the second case [mkosi.images/system/mkosi.conf.d/30-centos-fedora].

This repository helps reproducing the issue and shows two workarounds. It produces two images `base` and `system` of
type `directory` for distributions `centos`, `debian`, `fedora` and `ubuntu`. The `system` image will install package
`shim` on `centos` and `fedora` which is not available on `debian` (but on `ubuntu` it is). It will install package
`shim-signed` on `debian` and `ubuntu` which is not available on `centos` and `fedora`.

In practice though, `mkosi` will try to install `shim` when building the `system` image for `debian`. The `[Match]`
section in [mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf] is not applied to
[mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf.d/example.conf] properly. The `system` image build fails.

[mkosi.images/system]: mkosi.images/system
[mkosi.images/system/mkosi.conf.d]: mkosi.images/system/mkosi.conf.d
[mkosi.images/system/mkosi.conf.d/30-centos-fedora]: mkosi.images/system/mkosi.conf.d/30-centos-fedora
[mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf.d]: mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf.d
[mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf]: mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf
[mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf.d/example.conf]: mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf.d/example.conf

This issue is tracked in [mkosi issue #2545](https://github.com/systemd/mkosi/issues/2545).

## Reproduce

On Debian 12 (Bookworm) install the latest code of [mkosi](https://github.com/systemd/mkosi) from its `main` branch and
then run as unprivileged user:

```sh
git clone https://github.com/JM1/mkosi-issue-match-inheritance.git jm1-mkosi-issue-match-inheritance
cd jm1-mkosi-issue-match-inheritance
mkosi build
```

After building the `base` image completed successfully, the build of the `system` image starts but fails with:

```sh
[...]
‣ Building system image
Create subvolume '/home/xxx/.cache/mkosi/mkosi-workspaceoazwngvw/root'
‣  Copying in base trees…
Create a snapshot of '/home/xxx/yyy/jm1-mkosi-issue-match-inheritance/mkosi.output/base' in '/home/xxx/.cache/mkosi/mkosi-workspaceoazwngvw/root'
‣  Installing extra packages for Debian
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Package shim is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source
However the following packages replace it:
  shim-helpers-amd64-signed shim-unsigned

E: Package 'shim' has no installation candidate
‣ "env HOME=/ SYSTEMD_HWDB_UPDATE_BYPASS=1 KERNEL_INSTALL_BYPASS=1 APT_CONFIG=/etc/apt.conf DEBIAN_FRONTEND=noninteractive DEBCONF_INTERACTIVE_SEEN=true INITRD=No apt-get -o APT::Architecture=amd64 -o APT::Architectures=amd64 -o APT::Install-Recommends=false -o APT::Immediate-Configure=off -o APT::Get::Assume-Yes=true -o APT::Get::AutomaticRemove=true -o APT::Get::Allow-Change-Held-Packages=true -o APT::Get::Allow-Remove-Essential=true -o APT::Sandbox::User=root -o Dir::Cache=/var/cache/apt -o Dir::State=/var/lib/apt -o Dir::Log=/var/log/apt -o Dir::State::Status=/buildroot/var/lib/dpkg/status -o Dir::Bin::DPkg=/usr/bin/dpkg -o Debug::NoLocking=true -o DPkg::Options::=--root=/buildroot -o DPkg::Options::=--force-unsafe-io -o DPkg::Options::=--force-architecture -o DPkg::Options::=--force-depends -o DPkg::Options::=--no-debsig -o DPkg::Use-Pty=false -o DPkg::Install::Recursive::Minimum=1000 -o pkgCacheGen::ForceEssential=, install shim shim-signed" returned non-zero exit code 100.
‣  (Fixing ownership of package manager cache directory)
```

## Work around

To workaround this issue, the `[Match]` section from file [mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf]
has to be copied to file [mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf.d/example.conf]:

```
[Match]
Distribution=|centos
Distribution=|fedora
```

Another workaround is to change the `[Match]` section in file
[mkosi.images/system/mkosi.conf.d/30-centos-fedora/mkosi.conf] to list one distribution, for example:

```
[Match]
Distribution=fedora
```

With both workarounds, the `system` image builds as expected:

```
[...]
‣ Building system image
Create subvolume '/home/xxx/.cache/mkosi/mkosi-workspace8b818kyi/root'
‣  Copying in base trees…
Create a snapshot of '/home/xxx/yyy/jm1-mkosi-issue-match-inheritance/mkosi.output/base' in '/home/xxx/.cache/mkosi/mkosi-workspace8b818kyi/root'
‣  Installing extra packages for Debian
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  dmsetup gettext-base grub-common grub-efi-amd64-bin grub2-common libbrotli1 libdevmapper1.02.1 libefiboot1t64 libefivar1t64 libfreetype6 libfuse3-3 libkeyutils1 libpng16-16t64 mokutil shim-helpers-amd64-signed shim-signed-common shim-unsigned
Suggested packages:
  multiboot-doc grub-emu mtools xorriso desktop-base console-setup fuse3
Recommended packages:
  os-prober grub-efi-amd64-signed efibootmgr
The following NEW packages will be installed:
  dmsetup gettext-base grub-common grub-efi-amd64-bin grub2-common libbrotli1 libdevmapper1.02.1 libefiboot1t64 libefivar1t64 libfreetype6 libfuse3-3 libkeyutils1 libpng16-16t64 mokutil shim-helpers-amd64-signed shim-signed shim-signed-common shim-unsigned
0 upgraded, 18 newly installed, 0 to remove and 0 not upgraded.
Need to get 0 B/8213 kB of archives.
After this operation, 40.6 MB of additional disk space will be used.
Selecting previously unselected package gettext-base.
(Reading database ... 3727 files and directories currently installed.)
[...]
Setting up shim-signed:amd64 (1.40+15.7-1) ...
Processing triggers for libc-bin (2.37-15.1) ...
‣  Generating system users
‣  Generating volatile files
‣  Applying presets…
Unit /buildroot/usr/lib/systemd/system/dpkg-db-backup.timer is added as a dependency to a non-existent unit timers.target.
Unit /buildroot/usr/lib/systemd/system/fstrim.timer is added as a dependency to a non-existent unit timers.target.
Unit /buildroot/usr/lib/systemd/system/grub-common.service is added as a dependency to a non-existent unit multi-user.target.
Unit /buildroot/usr/lib/systemd/system/grub-common.service is added as a dependency to a non-existent unit suspend.target.
Unit /buildroot/usr/lib/systemd/system/grub-common.service is added as a dependency to a non-existent unit hibernate.target.
Unit /buildroot/usr/lib/systemd/system/grub-common.service is added as a dependency to a non-existent unit hybrid-sleep.target.
Unit /buildroot/usr/lib/systemd/system/grub-common.service is added as a dependency to a non-existent unit suspend-then-hibernate.target.
‣  Generating hardware database
No hwdb files found, skipping.
‣  /home/xxx/yyy/jm1-mkosi-issue-match-inheritance/mkosi.output/system size is 431.0M.
‣  Fixing ownership of package manager cache directory
```
