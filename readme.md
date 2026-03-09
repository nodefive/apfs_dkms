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
   sudo apt update
   sudo apt install dkms linux-headers-generic build-essential
   sudo dpkg -i apfs-dkms.deb
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

---

## Signing the APFS Kernel Module for Secure Boot

This guide covers the installation and digital signing of the driver. This is required for Linux systems with **Secure Boot** enabled to prevent "module verification failed" errors and ensure the driver loads successfully.

---

### Prerequisites
Ensure you have the necessary tools installed for building and signing modules:
```bash
sudo apt update
sudo apt install openssl kmod mokutil zstd dkms
```

---

### 1. Installation
Install the APFS driver from the `.deb` package.

1. Download the latest `.deb` from [apfs_dkms](https://github.com/nodefive/apfs_dkms).

2. Install the package:
   ```bash
   sudo apt install ./apfs-dkms_1.0_all.deb
   ```

3. Confirm DKMS has installed the module:
   ```bash
   dkms status
   ```

---

### 2. Generate and Enroll the Signing Key (MOK)
You must create a Machine Owner Key (MOK) that your computer's BIOS/UEFI will trust.

1. **Generate the Key Pair:**
   ```bash
   openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=APFS Driver/"
   ```

2. **Import the Key:**
   ```bash
   sudo mokutil --import MOK.der
   ```
   *Create a temporary password when prompted.*

3. **Reboot and Enroll:**
   * Restart your PC.
   * At the blue **"Perform MOK management"** screen, select **Enroll MOK**.
   * Select **Continue** and then **Yes**.
   * Enter the password you created and reboot.

---

### 3. Signing the Module
Modern Linux distributions compress modules with `.zst`. To avoid corrupting the file, you must decompress it, sign it, and re-compress it.

1. **Find the module path:**
   ```bash
   MOD_PATH=$(modinfo -n apfs)
   ```

2. **Decompress:**
   ```bash
   sudo zstd -d --rm "$MOD_PATH"
   ```

3. **Sign the Binary:**
   ```bash
   # The file is now at the same path without the .zst extension
   RAW_KO="${MOD_PATH%.zst}"
   sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 ./MOK.priv ./MOK.der "$RAW_KO"
   ```

4. **Re-compress:**
   ```bash
   sudo zstd --rm "$RAW_KO"
   ```

---

### 4. Final Activation & Verification
Tell the kernel to update its database and load the signed driver.

1. **Update Dependencies:**
   ```bash
   sudo depmod -a
   ```

2. **Load the Module:**
   ```bash
   sudo modprobe apfs
   ```

3. **Verify Success:**
   * Check for the signer: `modinfo apfs | grep -E "signer|sig_key"`
   * Verify it is active: `lsmod | grep apfs`
   * Check logs for errors: `sudo dmesg | tail -n 10 | grep apfs`

---

#### Final consideration - Kernel Updates
If you update your Linux kernel, you will need to repeat **Step 3** to sign the module for the new kernel version. KEEP your .priv key!

    **MOK.priv** (The Private Key): Every time your system downloads a Linux kernel update, DKMS will automatically rebuild a fresh, unsigned version of the APFS module for that new kernel. You will need this .priv file to sign the new module. If you delete it, you will have to generate a completely new key, reboot, and go through the blue-screen BIOS enrollment process all over again.

    **MOK.der** (The Public Key): This key is already safely injected into your motherboard's firmware. While you don't strictly need it to sign future modules, you should keep it in case you ever want to cleanly remove (un-enroll) this specific key from your BIOS using mokutil --delete.

### Security Warning

MOK.priv is literally the master key to bypass your computer's Secure Boot. If a malicious script or attacker gets ahold of that file, they can sign a stealthy rootkit, and your kernel will happily load it. So keep this file in a safe place!
