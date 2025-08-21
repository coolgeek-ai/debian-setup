# Debian Workstation Setup Guide

This guide outlines the steps to set up a Debian Bookworm system with virtualization capabilities, Kali Linux-style configuration, NVIDIA graphics support, and Secure Boot compatibility.

## Prerequisites
- Debian Bookworm installation
- Internet connection
- Basic command line knowledge
- sudo privileges

## Setup Instructions

### Step 1: Replace APT Sources with Official Bookworm Repos

Edit the sources list:
```bash
sudo nano /etc/apt/sources.list
```

Replace the content with:
```
deb https://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src https://deb.debian.org/debian bookworm main contrib non-free non-free-firmware

deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
deb-src https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware

deb https://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb-src https://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Save the file (Ctrl+O, Enter, Ctrl+X).

### Step 2: Upgrade the System

```bash
sudo apt update && sudo apt full-upgrade -y
```

### Step 3: Install Virtualization and Core Components

```bash
sudo apt install --no-install-recommends -y \
  qemu-kvm qemu-system-x86 libvirt-daemon-system libvirt-clients virt-manager \
  gir1.2-spiceclientgtk-3.0 dnsmasq-base qemu-utils iptables \
  git curl zsh zsh-syntax-highlighting zsh-autosuggestions \
  fonts-firacode fonts-cantarell \
  linux-headers-amd64 firmware-misc-nonfree
```

### Step 4: Configure libvirt Permissions and Default Network

```bash
sudo adduser "$USER" libvirt
sudo adduser "$USER" kvm
systemctl restart libvirtd
virsh -c qemu:///system net-autostart default
virsh -c qemu:///system net-start default
```

### Step 5: Configure DNS (IPv4 & IPv6)

```bash
nmcli connection modify "Ethernet" \
  ipv4.ignore-auto-dns yes \
  ipv4.dns "8.8.8.8 1.1.1.1" \
  ipv6.ignore-auto-dns yes \
  ipv6.dns "2001:4860:4860::8888 2606:4700:4700::1111"
nmcli connection down "Ethernet"
nmcli connection up "Ethernet"
```

**Note:** Replace "Ethernet" with your actual connection name (check with `nmcli connection show`).

### Step 6: Apply Kali-style ZSH Configuration

```bash
curl -fsSL https://gitlab.com/kalilinux/packages/kali-defaults/-/raw/kali/master/etc/skel/.zshrc -o "$HOME"/.zshrc
git clone --depth=1 https://gitlab.com/kalilinux/packages/kali-themes.git /tmp/kali-themes
sudo cp -a /tmp/kali-themes/share/. /usr/share/
rm -rf /tmp/kali-themes
chsh -s "$(command -v zsh)"
```

**Note:** Logout and login again for ZSH to take effect.

### Step 7: Configure DKMS Automatic Signing

```bash
sudo mkdir -p /etc/dkms
sudo nano /etc/dkms/framework.conf
```

Add these lines:
```
mok_signing_key="/var/lib/shim-signed/mok/MOK.priv"
mok_certificate="/var/lib/shim-signed/mok/MOK.der"
sign_tool="/etc/dkms/sign_helper.sh"
```

Create the sign helper script:
```bash
sudo nano /etc/dkms/sign_helper.sh
```

Add this content:
```bash
#!/bin/bash
/lib/modules/"$1"/build/scripts/sign-file sha512 /var/lib/shim-signed/mok/MOK.priv /var/lib/shim-signed/mok/MOK.der "$2"
```

Make the script executable:
```bash
sudo chmod +x /etc/dkms/sign_helper.sh
```

### Step 8: Generate and Import Machine Owner Key (MOK)

```bash
sudo mkdir -p /var/lib/shim-signed/mok
sudo rm -rf /var/lib/shim-signed/mok/*

sudo openssl req -nodes -new -x509 -newkey rsa:2048 \
  -keyout /var/lib/shim-signed/mok/MOK.priv -outform DER -out /var/lib/shim-signed/mok/MOK.der \
  -days 36500 -subj "/CN=Nvidia/"
```

Import the key (remember the password you set):
```bash
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
```

### Step 9: Reboot and Complete MOK Enrollment

```
sudo reboot
```
After reboot, follow the blue screen prompt:
> Enroll MOK → Continue → Yes → Enter password → Reboot

### Step 10: Install NVIDIA Driver (DKMS Will Auto-Sign)

```bash
sudo apt purge nvidia* && sudo apt autoremove -y && sudo apt autoclean
sudo apt install -y nvidia-driver

sudo reboot
```

### Step 11: Verify Secure Boot, MOK, and NVIDIA Driver

```bash
sudo dmesg | grep -i secureboot
mokutil --list-enrolled
lsmod | grep nvidia
nvidia-smi
```

## Troubleshooting

### NVIDIA Driver Issues
If the NVIDIA driver module is not signed correctly:
```bash
sudo mokutil --reset
sudo reboot
```

Reboot and follow the blue screen prompt:
> Reset → Continue → Yes → Enter password → Reboot

Then repeat:
> Step 8, 9, and 10.


## References
- [Debian SourcesList](https://wiki.debian.org/SourcesList#sources.list)
- [Debian SecureBoot](https://wiki.debian.org/SecureBoot#Virtualbox_Packages)
- [Debian NvidiaGraphicsDriver](https://wiki.debian.org/NvidiaGraphicsDrivers#Version_535.183.01-1)
- [Whonix for KVM](https://www.whonix.org/wiki/KVM#Debian)
- [Kali Linux Packages/kali-themes](https://gitlab.com/kalilinux/packages/kali-themes)
- [Kali Linux Packages/kali-defaults](https://gitlab.com/kalilinux/packages/kali-defaults/)
