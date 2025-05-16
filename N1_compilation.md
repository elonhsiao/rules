# Compiling OpenWrt/ImmortalWrt for Philcomn N1 Router with Nikki Integration

Here is a summary of the steps we followed to successfully compile a custom OpenWrt/ImmortalWrt image for your Philcomn N1 router using the ophub/amlogic-s9xxx-openwrt build system. This guide includes all the commands we used and encountered during the process, along with comments for clarity.

## About Nikki

Nikki is a powerful package that provides transparent proxy capabilities with Mihomo on OpenWrt. It offers the following features:

- Transparent Proxy (Redirect/TPROXY/TUN, IPv4 and/or IPv6)
- Access Control
- Profile Mixin
- Profile Editor
- Scheduled Restart

### Prerequisites for Nikki

To use Nikki, your OpenWrt system must meet these requirements:

- OpenWrt >= 23.05
- Linux Kernel >= 5.13 (the images we build in this guide will satisfy this requirement)
- firewall4

This guide will help you build an OpenWrt image with Nikki pre-installed, saving you the trouble of installing it separately after flashing your router.

## 1. Prerequisites

Ensure your build environment (like Ubuntu 24.04 LTS) has the necessary packages installed.

```bash
# Update the package list to get the latest information on available packages
sudo apt update

# Install essential packages for OpenWrt compilation.
# This command fetches a list of packages recommended by the project for Ubuntu 22.04,
# which should be largely compatible with 24.04.
sudo apt install -y $(curl -fsSL https://tinyurl.com/ubuntu2204-make-openwrt)

# Install the btrfs filesystem utilities. This package provides essential tools
# needed by the build script to work with Btrfs partitions during image creation.
# We found this was missing and caused mount errors.
sudo apt install btrfs-progs

# Optional: Install pigz for faster compression. The build script will use this
# if available, otherwise it falls back to the standard gzip.
sudo apt install pigz
```

### Nikki Dependencies

When building with Nikki integration, the following packages will be included in your OpenWrt image. These are handled automatically by the customization script in Step 4, but it's good to be aware of them:

- ca-bundle: Certificate Authority bundle for secure connections
- curl: Command-line tool for transferring data with URLs
- yq: Lightweight and portable command-line YAML processor
- firewall4: NFT-based firewall for OpenWrt
- ip-full: Full version of IP utilities from iproute2
- kmod-inet-diag: Kernel modules for socket monitoring
- kmod-nft-socket: Kernel module for nftables socket operations
- kmod-nft-tproxy: Kernel module for transparent proxy support
- kmod-tun: Universal TUN/TAP device driver

## 2. Clone the Repository

Clone the ophub/amlogic-s9xxx-openwrt build system repository from GitHub.

```bash
# Navigate to your desired directory where you want to clone the repository (e.g., your home directory)
cd ~

# Clone the repository. Using --depth 1 performs a shallow clone, which is faster
# as it only fetches the latest commit history.
git clone --depth 1 https://github.com/ophub/amlogic-s9xxx-openwrt.git

# Navigate into the cloned repository's main directory
cd amlogic-s9xxx-openwrt
```

## 3. Obtain the Base Root Filesystem

The remake script requires a base OpenWrt/ImmortalWrt root file system archive (rootfs.tar.gz) for the armsr (aarch64) architecture. This file is not built by the script itself but is used as a starting point. You need to download it manually from the project's GitHub Releases.

1. Go to the ophub/amlogic-s9xxx-openwrt GitHub Releases page.
2. Find a suitable and recent release tag (e.g., OpenWrt_official_save_YYYY.MM).
3. In the "Assets" section for that release, download the file named openwrt-armsr-armv8-generic-rootfs.tar.gz.
4. Once downloaded, place this file in the openwrt-armsr directory within your cloned repository.

```bash
# Navigate to the openwrt-armsr directory within your cloned repository
cd ~/amlogic-s9xxx-openwrt/openwrt-armsr

# Assuming the downloaded file is in your Downloads folder, move it to this directory.
# Replace 'openwrt-armsr-armv8-generic-rootfs.tar.gz' if the downloaded filename is different.
# The '.' at the end means move it to the current directory.
mv ~/Downloads/openwrt-armsr-armv8-generic-rootfs.tar.gz .

# Verify the file is now in the directory
ls

# Return to the main repository directory
cd ~/amlogic-s9xxx-openwrt
```

## 4. Create/Edit Customization Script (diy-script.sh)

This is a crucial step for adding custom packages like Nikki (for transparent proxy with Mihomo), Openclash, Dockerman, and their LuCI interfaces. The remake script executes this script during the build process to apply your customizations to the root filesystem.

```bash
# Navigate to the openwrt-script directory
cd openwrt-script

# Create or edit the diy-script.sh file using your preferred text editor (e.g., nano, vim, gedit).
# If the file doesn't exist, this command will create it.
nano diy-script.sh

# Inside the editor, add the commands necessary to include the desired package feeds
# and select the packages you want.

# Here's an example script that adds the Nikki feed for transparent proxy with Mihomo:

#!/bin/bash

# Add Nikki feed for transparent proxy with Mihomo
echo "src-git nikki https://github.com/nikkinikki-org/OpenWrt-nikki.git;main" >> ../feeds.conf.default

# Update all feeds
../scripts/feeds update -a

# Install all feeds
../scripts/feeds install -a

# Select specific packages (uncomment or add as needed)
# For Nikki (transparent proxy with Mihomo)
cat >> .config <<EOF
CONFIG_PACKAGE_nikki=y
CONFIG_PACKAGE_luci-app-nikki=y
CONFIG_PACKAGE_luci-i18n-nikki-zh-cn=y
# Nikki dependencies
CONFIG_PACKAGE_ca-bundle=y
CONFIG_PACKAGE_curl=y
CONFIG_PACKAGE_yq=y
CONFIG_PACKAGE_firewall4=y
CONFIG_PACKAGE_ip-full=y
CONFIG_PACKAGE_kmod-inet-diag=y
CONFIG_PACKAGE_kmod-nft-socket=y
CONFIG_PACKAGE_kmod-nft-tproxy=y
CONFIG_PACKAGE_kmod-tun=y
EOF

# For other common packages (optional)
cat >> .config <<EOF
# Docker support
CONFIG_PACKAGE_dockerd=y
CONFIG_PACKAGE_luci-app-dockerman=y
# OpenClash
CONFIG_PACKAGE_luci-app-openclash=y
EOF

# Make sure to save and close the file after adding your commands.

# Make the script executable. This is necessary for the remake script to run it.
chmod +x diy-script.sh

# Return to the main repository directory
cd ..
```

## 5. Prepare the Build Environment for Remake

Clean up any temporary files or leftover loop devices from previous failed build attempts to ensure a clean start.

```bash
# Check for any active loop devices. Look for devices pointing to .img files
# within your repository's directories (~/amlogic-s9xxx-openwrt/...).
losetup -a

# If you find any relevant loop devices (e.g., /dev/loop3 from our previous attempt),
# detach them using sudo. Replace /dev/loopX with the actual device name.
# sudo losetup -d /dev/loopX

# Clean up temporary build directories created by the script during previous runs.
# These directories contain intermediate files and mounts. Use sudo as they are
# owned by root from the previous script execution.
sudo rm -rf ~/amlogic-s9xxx-openwrt/openwrt/tmp
```

## 6. Run the Compilation Script

Execute the main build script, remake, with superuser privileges. Specify the target board (s905d for Philcomn N1) using the -b option.

```bash
# Navigate to the main repository directory if you are not already there
cd ~/amlogic-s9xxx-openwrt

# Run the remake script.
# -b s905d specifies the target board as s905d (for Philcomn N1).
# The script will automatically select a suitable and likely the latest stable kernel
# for this board by default.
sudo ./remake -b s905d

# Optional: You can specify a particular kernel version using the -k option
# based on available kernels in the project's releases (e.g., -k 6.1.138).
# sudo ./remake -b s905d -k 6.1.138
```

The script will now proceed through several stages: initializing, checking data, finding the rootfs, downloading dependencies (if needed and not cached), querying/downloading kernels, making the image, extracting the rootfs, replacing the kernel, refactoring filesystems, and cleaning up.

## 7. Find the Compiled Image

After the remake script completes successfully ([ SUCCESS ] All process completed successfully.), the compiled OpenWrt image file(s) will be located in the openwrt/out directory within the main repository folder.

```bash
# Navigate to the output directory, starting from the main repository directory
cd ~/amlogic-s9xxx-openwrt/openwrt/out

# List the contents to find the generated image file(s).
# They will typically be in .img.gz format and include the board and kernel version in the name.
ls
```

You should find files similar to openwrt_official_amlogic_s905d_k6.1.138_2025.05.16.img.gz and openwrt_official_amlogic_s905d_k6.12.28_2025.05.16.img.gz (the kernel version and date will match your build).

## 8. Write Image to Bootable Medium and Install

The compiled .img.gz file is the firmware image you will use to install OpenWrt on your Philcomn N1.

### Default Login Information

When you first boot the OpenWrt image, you can access it using the following default credentials:

| Name | Value |
|------|-------|
| Default IP | 192.168.1.1 |
| Default Account | root |
| Default Password | password |
| Default WIFI Name | OpenWrt |
| Default WIFI Password | None |

It is highly recommended to change the password and configure network settings after your first successful login for security.

1. **Download the image**: Transfer the desired .img.gz file from your Ubuntu build system to a computer where you can write it to a USB drive or SD card.

2. **Write the image**: Use an image writing tool like balenaEtcher (cross-platform) or Rufus (Windows) to write the compiled .img.gz file to a USB drive or SD card (minimum recommended size is usually 8GB or larger). This process extracts the image and makes the USB/SD card bootable.

3. **Boot the router**: Insert the bootable USB drive or SD card into your Philcomn N1 router. You will need to configure the router to boot from the external media. This often involves pressing and holding a specific button (like a reset button, sometimes located inside the AV port) while powering on the device, or accessing a boot menu. Consult resources specific to booting your Philcomn N1 from USB/SD.

4. **Install to eMMC (Optional)**: Once OpenWrt boots successfully from the USB/SD card, you can access its web interface (using the default IP and credentials above) or SSH. If you wish to install OpenWrt to the device's internal eMMC storage, follow the installation instructions provided by the ophub/amlogic-s9xxx-openwrt project, typically found within the OpenWrt web interface under "System" -> "Amlogic Service" -> "Install OpenWrt".

## 9. Understanding the Compiled Images

You found two image files in the out directory:
- openwrt_official_amlogic_s905d_k6.1.138_2025.05.16.img.gz
- openwrt_official_amlogic_s905d_k6.12.28_2025.05.16.img.gz

The remake script, when run without specifying a kernel version using the -k option, attempts to build images for all supported kernels listed for the target board (s905d) in the project's configuration files. In this case, it found configurations for both kernel 6.1.y and 6.12.y and built an image for the latest version it found for each series (6.1.138 and 6.12.28).

### Which one should you use?

Both images are valid OpenWrt builds for your s905d device with different kernel versions.

- **6.1.138**: This is a more mature Long-Term Support (LTS) kernel series. It is generally considered very stable and well-tested.
- **6.12.28**: This is a newer kernel series. It might include support for newer hardware or features but could potentially be less stable than the LTS series.

For general use and stability, starting with the 6.1.138 image is often recommended. If you encounter issues with hardware support or specific features, you could then try the 6.12.28 image to see if the newer kernel resolves them. The best kernel can sometimes depend on the specific hardware revisions or components in your Philcomn N1.

## 10. Configuring and Using Nikki

### If Nikki is Pre-installed (via Build Process)

If you've included Nikki in your build as described in the customization script section, it will be pre-installed on your OpenWrt system. Here's how to configure it after booting your new OpenWrt image:

1. **Access the LuCI Web Interface**: Navigate to http://192.168.1.1 in your web browser and log in with the default credentials (username: root, password: password).

2. **Navigate to Nikki Settings**: Go to "Services" -> "Nikki" in the LuCI menu.

3. **Basic Configuration**:
   - Enable Nikki by toggling the switch
   - Select or upload a configuration file (Mihomo/Clash format)
   - Configure proxy settings according to your needs

4. **Advanced Features**:
   - Set up access control rules to determine which devices use the proxy
   - Configure profile mixing options
   - Set scheduled restart if desired

5. **Apply Configuration**: Click "Save & Apply" to activate your settings.

### Alternative Installation Methods

If you didn't include Nikki during the build process or want to install it on an existing OpenWrt system, you have these options:

#### A. Install From Feed (Recommended)

1. Add the Nikki feed:

```bash
# Run this command on your OpenWrt device (via SSH)
curl -s -L https://github.com/nikkinikki-org/OpenWrt-nikki/raw/refs/heads/main/feed.sh | ash
```

2. Install the packages:

```bash
# For opkg-based systems
opkg install nikki
opkg install luci-app-nikki
opkg install luci-i18n-nikki-zh-cn  # Optional: Chinese language pack

# For apk-based systems
apk add nikki
apk add luci-app-nikki
apk add luci-i18n-nikki-zh-cn  # Optional: Chinese language pack
```

#### B. Install From Release

Alternatively, you can use the installation script:

```bash
curl -s -L https://github.com/nikkinikki-org/OpenWrt-nikki/raw/refs/heads/main/install.sh | ash
```

For detailed configuration instructions and troubleshooting, refer to the Nikki wiki at: https://github.com/nikkinikki-org/OpenWrt-nikki/wiki

## Troubleshooting Notes

- **Network/Download Errors** (e.g., gnutls_handshake(), unexpected eof): If you see errors during the script's dependency or kernel download phases, it usually indicates issues with your internet connection, system date/time, or network configurations like firewalls or proxies interfering with secure connections to GitHub. Check these factors and try running the script again.

- **Btrfs Mount Errors** (wrong fs type, bad option, bad superblock, missing codepage): This type of error, especially during the "Extract OpenWrt files" step, typically means that the Btrfs filesystem tools (btrfs-progs) are not correctly installed or there's an issue with Btrfs kernel support on your system. Ensure btrfs-progs is installed (Step 1) and try cleaning up temporary files and leftover loop devices (Step 5).

- **"Device or resource busy" during cleanup**: If you get this error when trying to remove temporary files (Step 5), it means the system still sees those directories as being in use, even after the script failed. Try attempting a lazy unmount on the temporary bootfs and rootfs directories before retrying the rm -rf command:

```bash
# Navigate to the main repository directory
cd ~/amlogic-s9xxx-openwrt
# Attempt lazy unmount (replace kernel version if needed)
sudo umount -l ~/amlogic-s9xxx-openwrt/openwrt/tmp/6.1.138/s905d/bootfs
sudo umount -l ~/amlogic-s9xxx-openwrt/openwrt/tmp/6.1.138/s905d/rootfs
# Then retry the remove command
sudo rm -rf ~/amlogic-s9xxx-openwrt/openwrt/tmp
```