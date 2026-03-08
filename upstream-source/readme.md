# APFS DKMS: The Developer's Build Manual

This document provides the exact technical steps and logic used to transform the original `linux-apfs-rw` source code into a fully automated DKMS (Dynamic Kernel Module) Debian package. 

---

## 1. Preparing the Workspace

Clone the repo, either mine or the original upstream one:

```bash
git clone --depth 1 https://github.com/nodefive/apfs_dkms.git temp && mv temp/upstream-source ~/Desktop/linux-apfs-rw && rm -rf temp
```

or 

```bash
git clone https://github.com/linux-apfs/linux-apfs-rw.git ~/Desktop/apfs-ubuntu-dkms
```

We begin by creating the standardized Debian package directory structure.

```bash
mkdir -p ~/Desktop/apfs-dkms-pkg/usr/src/apfs-1.0
mkdir -p ~/Desktop/apfs-dkms-pkg/DEBIAN
```

* **`usr/src/apfs-1.0`**: This is where DKMS expects to find the source code.
* **`DEBIAN`**: This folder holds the package metadata and installation scripts.

---

## 2. Importing & Patching the Source

Next, we pull the original source code but apply a critical fix for standalone builds.

```bash
rsync -av --exclude='.git' ~/Desktop/linux-apfs-rw/ ~/Desktop/apfs-dkms-pkg/usr/src/apfs-1.0/
```

* **`rsync`**: We exclude the `.git` folder to keep the package lightweight.
  

Add the version.h header file

```bash
echo '#define APFS_VERSION "1.0"' > ~/Desktop/apfs-dkms-pkg/usr/src/apfs-1.0/version.h
```

* **`version.h`**: The original code relies on git history to generate a version. Since we removed the `.git` folder, we must manually create this header file to prevent the compiler from crashing.

---

## 3. Configuring DKMS

We create the `dkms.conf` file to tell the system how to handle the kernel module.

```bash
nano ~/Desktop/apfs-dkms-pkg/usr/src/apfs-1.0/dkms.conf
```

**File Content:**

```text
PACKAGE_NAME="apfs"
PACKAGE_VERSION="1.0"
BUILT_MODULE_NAME[0]="apfs"
DEST_MODULE_LOCATION[0]="/kernel/fs/apfs/"
AUTOINSTALL="yes"
MAKE[0]="make KERNELRELEASE=$kernelver"
CLEAN="make clean"
```

* **`MAKE[0]`**: Crucial fix. This forces DKMS to use the project's native `Makefile` instead of the generic kernel builder, ensuring it works on modern kernels (6.17+).

---

## 4. Defining Package Metadata

The `control` file tells the Ubuntu package manager what this is and what it needs to run.

```bash
nano ~/Desktop/apfs-dkms-pkg/DEBIAN/control
```

**File Content:**

```text
Package: apfs-dkms
Version: 1.0
Architecture: all
Maintainer: eafer
Depends: dkms, linux-headers-generic, build-essential
Description: Native APFS kernel module with DKMS support for seamless GUI mounting.
```

---

## 5. Automation Scripts (Installation & Removal)

These scripts handle the heavy lifting when the user installs or uninstalls the `.deb`.

### Post-Installation (`postinst`)

```bash
nano ~/Desktop/apfs-dkms-pkg/DEBIAN/postinst
```

**File Content:**

```
#!/bin/sh
set -e
# Add, build, and install the module via DKMS
dkms add -m apfs -v 1.0
dkms build -m apfs -v 1.0
dkms install -m apfs -v 1.0
# Load the module into the current running kernel immediately
modprobe apfs || true
exit 0
```


**Logic:** Registers the source with DKMS, builds it for the current kernel, and loads the driver into memory immediately so no reboot is required.


### Pre-Removal (`prerm`)

```bash
nano ~/Desktop/apfs-dkms-pkg/DEBIAN/prerm
```

**File Content:**

```
#!/bin/sh
set -e
# Unload the module and remove it from DKMS
rmmod apfs || true
dkms remove -m apfs -v 1.0 --all || true
exit 0
```

**Logic:** Safely unloads the driver from the kernel before files are deleted, preventing the system from hanging or crashing.

---

## 6. Permissions & Final Build

Before packaging, we must set the correct permissions to these scripts and then compile the archive.

```bash
chmod 755 ~/Desktop/apfs-dkms-pkg/DEBIAN/postinst
chmod 755 ~/Desktop/apfs-dkms-pkg/DEBIAN/prerm
```

* **`chmod 755`**: Marks the scripts as executable so the installer can run them.

Now build with:

```
dpkg-deb --root-owner-group --build ~/Desktop/apfs-dkms-pkg
```

* **`--root-owner-group`**: Ensures all files inside the package are owned by the system (`root`) rather than your personal user account.

---

## 7. Installation

To install the kernel driver:

```bash
sudo dpkg -i ~/Desktop/apfs-dkms-pkg.deb
```
