# APFS DKMS for Linux

A plug-and-play Dynamic Kernel Module Support (DKMS) package for Apple File System (APFS) on Linux. 

This repository provides a pre-configured `.deb` package that builds the APFS kernel module directly into your Linux kernel. Unlike userspace FUSE drivers (`apfs-fuse`), this kernel-level integration allows seamless mounting and unmounting of APFS drives directly through standard GUI file managers (like GNOME Files/Nautilus) without throwing `udisks2` errors.

## Credits & Upstream
This project packages the incredible C source code from the [linux-apfs-rw](https://github.com/linux-apfs/linux-apfs-rw) repository. All credit for the underlying filesystem driver and reverse-engineering goes to the original authors. This repository simply wraps their work into an automated DKMS Debian package for easy desktop integration.

## Installation (Quick Start)
If you just want to install the driver and mount your drives seamlessly:

1. Download the latest `.deb` package from the `releases/` folder (or the GitHub Releases tab).
2. Install it using `dpkg`:
3. 
   ```bash
   sudo dpkg -i apfs-dkms-pkg.deb
   ```

   *Note: Installation takes a few moments as DKMS will compile the module against your active kernel in the background.*

3. Plug in your APFS formatted drive and click it in your file manager. It will mount natively.

## How to Build from Source
If you want to modify the DKMS configuration or rebuild the `.deb` package yourself, follow these steps.

### Build Requirements
Ensure your system has the necessary build tools and kernel headers:

```bash
sudo apt update
sudo apt install dkms linux-headers-generic build-essential
```

### Package Instructions

1. Clone this repository:
2. 
   ```bash
   cd ~/Desktop
   git clone https://github.com/nodefive/apfs_dkms.git
   cd apfs_dkms/apfs-dkms
   ```

2. Build the Debian package. The `--root-owner-group` flag is required to ensure the files inside the `.deb` are owned by `root`, preventing permission errors on the target system.
   
   ```bash
   dpkg-deb --root-owner-group --build . apfs-dkms.deb
   ```

3. The built package will be named `apfs-ubuntu-dkms.deb` and placed in the parent directory.

### Kernel Module Build & Install

```bash
sudo apt install apfs-dkms
```

### Check Module

```bash
sudo lsmod | grep apfs
```

## Uninstall

To completely remove the driver and DKMS build from your system:

```bash
sudo apt remove apfs-dkms
```
