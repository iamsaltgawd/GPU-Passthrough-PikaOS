# [Unfinished] GPU Passthrough on a Laptop Running PikaOS (Debian-based)

## Abstract
ATM this guide will assist in setting up CPU and GPU virtualization. You can follow [the guide made by asus-linux team](https://asus-linux.org/guides/vfio-guide/), as it covers setting up a Windows 10/11 VM with QEMU/KVM enhancements for a laptop with hybrid graphics (Intel/AMD + NVIDIA).

## Prerequisites
- **CPU:** At least a 6-core Intel/AMD CPU with hyper-threading/SMT (12 threads total) and virtualization support (VT-d/AMD-Vi).
- **GPU:** Intel/AMD iGPU + NVIDIA GPU with a MUXed configuration.
- **(Optional)** Installed **NVIDIA driver** ([guide](https://rpmfusion.org/Howto/NVIDIA)).

## Pre-setup
### Enable Intel VT-d/AMD Vi Virtualization
- Access BIOS/UEFI settings and enable **VT-d/IOMMU** (may be listed as "SVM Mode" or similar).

### Enable IOMMU Grouping
- Check IOMMU status:
```shell
lspci | grep NVIDIA
lspci -s 01:00.
```
- For AMD CPUs, IOMMU is enabled by default.
- For Intel CPUs, modify GRUB:
```shell
sudo nano /boot/refind_linux.conf # For rEFInd
```
Append:
```shell
... intel_iommu=on iommu=pt
```
If using GRUB:
```shell
sudo nano /etc/default/grub
```
Add before `GRUB_CMDLINE_LINUX`:
```shell
intel_iommu=on iommu=pt
```
Apply changes and reboot:
```shell
sudo grub2-mkconfig -o /boot/grub2/grub.cfg # BIOS
sudo shutdown -r now
```

## Using Supergfxctl to Bind VFIO to GPU
Supergfxctl allows switching between GPU modes and binding VFIO to the dGPU.

### Add VFIO Mode to Supergfxctl
Edit configuration:
```shell
sudo nano /etc/supergfxd.conf
```
Set:
```json
{
  "mode": "Hybrid",
  "vfio_enable": true,
  "vfio_save": true,
  "compute_save": false,
  "always_reboot": false,
  "no_logind": false,
  "logout_timeout_s": 180,
  "hotplug_type": "None"
}
```
Restart service:
```shell
sudo systemctl restart supergfxd.service
```
Check available modes:
```shell
supergfxctl -s
```
Expected output: `[Integrated, Hybrid, Vfio]`

## Switching Between Hybrid and VFIO Mode
### Manual Switching
1. Switch to `Integrated` mode, log out, then switch to `VFIO`.
2. To return, switch to `Integrated`, then `Hybrid`, and log out.

### Automating the Switching Process
Create a script:
```shell
sudo nano /usr/local/bin/switch-gpu.sh
```
Paste:
```bash
#!/bin/bash
CONFIG_FILE="$HOME/.gpu_mode"  # temp file for storing the mode
CURRENT_MODE=$(supergfxctl -g) # verifies current mode

if [ -z "$1" ]; then
    echo "Use the following arguments: "
        echo "  -m,     Get the current mode"
        echo "  -h,     Set graphics mode to Hybrid"
        echo "  -v,     Set graphics mode to Vfio"
        exit 1
fi

case "$1" in
    -v)
        if [ "$CURRENT_MODE" == "VFIO" ]; then
            echo "[INFO] You are already in VFIO mode."
            exit 0
        fi

        echo "[INFO] Changing to Integrated before VFIO..."
        sudo supergfxctl -m Integrated
        echo "vfio" > "$CONFIG_FILE"
        echo "[INFO] Logout mandatory"
        sleep 2
        kill -9 -1
        ;;

    -h)
        if [ "$CURRENT_MODE" == "Hybrid" ]; then
            echo "[INFO] You are already in Hybrid mode."
            exit 0
        fi

        echo "[INFO] Changing to Integrated before Hybrid..."
        sudo supergfxctl -m Integrated
        sleep 1
        sudo supergfxctl -m Hybrid
        echo "[INFO] Logout mandatory"
        sleep 2
        kill -9 -1
        ;;
    -m)
        supergfxctl -g
        ;;
    *)
        echo "Use the following arguments: "
        echo "  -m,     Get the current mode"
        echo "  -h,     Set graphics mode to Hybrid"
        echo "  -v,     Set graphics mode to Vfio"
        exit 1
        ;;
esac
```
Make it executable:
```shell
chmod +x /usr/local/bin/switch-gpu.sh
```
Add an alias:
```shell
echo 'alias switchgpu="bash /usr/local/bin/switch-gpu.sh"' >> ~/.bash_aliases
source ~/.bashrc
```
### Auto-Switch on Login
Create an auto-switch script:
```shell
mkdir -p ~/.config/autostart-scripts
sudo nano ~/.config/autostart-scripts/gpu-autoswitch.sh
```
Paste:
```bash
#!/bin/bash

CONFIG_FILE="$HOME/.gpu_mode"

if [[ -f "$CONFIG_FILE" ]]; then
    MODE=$(cat "$CONFIG_FILE")
    if [[ "$MODE" == "vfio" ]]; then
        echo "[INFO] Switching to VFIO..."
        sudo supergfxctl -m Vfio
        rm "$CONFIG_FILE"
    fi
fi

```
Make it executable:
```shell
chmod +x ~/.config/autostart-scripts/gpu-autoswitch.sh
```
Enable autostart:
```shell
sudo nano ~/.config/autostart/gpu-autoswitch.desktop
```
Paste:
```ini
[Desktop Entry]
Type=Application
Exec=~/.config/autostart-scripts/gpu-autoswitch.sh
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=GPU AutoSwitch
Comment=Auto switch GPU mode at login
```
Grant permission to run supergfxctl without sudo:
```shell
sudo visudo
```
Add at the end:
```shell
$USER ALL=(ALL) NOPASSWD: /usr/bin/supergfxctl
```

## References
- [VFIO GPU Passthrough Guide](https://asus-linux.org/wiki/vfio-guide)
- [Optimus dGPU Passthrough Guide](https://github.com/mysteryx93/GPU-Passthrough-with-Optimus-Manager-Guide)
- [Supergfxctl Documentation](https://gitlab.com/asus-linux/supergfxctl)
