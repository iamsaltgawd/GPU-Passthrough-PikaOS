# [Unfinished] GPU Passthrough on a Laptop Running PikaOS (Debian-based)

## Abstract
ATM this guide will assist in setting up CPU and GPU virtualization. You can follow [the guide made by asus-linux team](https://asus-linux.org/guides/vfio-guide/), as it covers setting up a Windows 10/11 VM with QEMU/KVM enhancements for a laptop with hybrid graphics (Intel/AMD + NVIDIA).

## Prerequisites
- **CPU:** At least a 6-core Intel/AMD CPU with hyper-threading/SMT (12 threads total) and virtualization support (VT-d/AMD-Vi).
- **GPU:** Intel/AMD iGPU + NVIDIA GPU with a MUXed configuration.
- **RAM:** Minimum 12GB of RAM. The more the better.
- **Storage:** Allocate **256GB – 512GB** for the virtual disk. Strongly recommending **SSD** instead of **HDD** for better performance.
- **Display Options for the Windows 10/11 VM:**
    - Dedicated Monitor (Recommended)
    - Looking Glass (a good alternative)
- **(Optional)** Installed **NVIDIA driver** ([guide](https://rpmfusion.org/Howto/NVIDIA)).
```
Tested with ROG Zephyrus G14 GA401IU 1.0 (AMD Ryzen 9 4900HS + NVIDIA Geforce 1660 Ti Mobile) with PikaOS 4.0 (based on Debian), using NVIDIA driver
```
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
The command you will be using is `gpuswitch`.
## Installing Dependencies

First things first, make sure your system has all the necessary packages installed. Run the following:
```shell
sudo apt update
sudo apt -y install qemu-kvm libvirt-daemon  bridge-utils virtinst libvirt-daemon-system
```
If you're planning to use a GUI to manage your VMs (which I am, because convenience), install virt-manager and some extra tools:
```shell
sudo apt -y install vim libguestfs-tools libosinfo-bin  qemu-system virt-manager
```
### Preparing ISO Files
Create a dedicated folder in your home directory to store the Windows ISO. I'm using a [custom lightweight windows build](https://windowsxlite.com/Micro10-22H2/), but you can use whatever fits your needs.
<p align="center">
	 <img src=./media/Screenshot_20250302_190743.png height=300px>
</p>

You'll also need the **virtio-win drivers** for optimal performance. Head over to [**this link**](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md), grab the **Latest virtio-win ISO**, and drop it into the same folder.
### Creating the VM
- Open virt-manager from your application menu.
- Click **Create a new VM**, then select **Local install media**.
<p align="center">
	 <img src=./media/1.png height=500px>
 </p>

- Click **Browse**, select your Windows ISO, and confirm.
<p align="center">
	 <img src=./media/2.png height=500px>
 </p>

- Click the **+** button in the bottom left, name the storage pool *(eg: `win10VM`)*, and point it to your VM folder.
<p align="center">
	 <img src=./media/3.png height=500px>
</p>

- Select the newly created pool, choose your **Windows ISO**, and hit **Choose Volume**.
<p align="center">
	<img src=./media/4.png height=500px>
</p>

- Assign **RAM** and **CPU cores** based on what your system can handle. At least 4GB RAM and 2–4 cores are recommended for Windows 10/11. You can modify these settings again later.
<p align="center">
	<img src=./media/5.png height=500px>
</p>

### Creating the virtual disk
Instead of using the default storage path (`/var/lib/libvirt/images`), store your virtual disk in your VM folder:
- Click "**Select or create a custom image**", then **Manage**
<p align="center">
	<img src=./media/6.png height=500px>
</p>

- Choose your pool, hit **+** at the top, name your virtual disk, set a size and make sure the format **qcow2** is selected. 

<p align="center">
	<img src=./media/7.png height=500px>
</p>

- Select the disk that you have just created. Then click **Choose Volume**. Make sure the path is correct.
<p align="center">
	<img src=./media/8.png height=500px>
</p>

- Name your VM (no spaces). Tick **Customize configuration before install** so we can tweak things before booting up. Then, click **Finish**.
<p align="center">
	<img src=./media/9.png height=500px>
</p>

### Adjusting VM Settings
#### Overview
- Set **Chipset** to **Q35**.
- Change the firmware to **UEFI x86_64: /../OVMF_CODE_4M.fd** (or `OVMF_CODE_4m.secboot.fd` for Windows 11, to enable secure boot). Click **Apply**.
<p align="center">
	<img src=./media/10.png height=500px>
</p>

#### CPU
- **Manually set CPU topology**, and edit your **sockets**, **cores** and **threads**. Avoid maxing out your host’s CPU - leave at least one or two cores free.
<p align="center">
	<img src=./media/11.png height=250px>
</p>

- Run `lscpu` if you are unsure about your CPU layout. Check under the `VendorID`. This is the output in my terminal:
    ```shell
    Vendor ID:                AuthenticAMD
      Model name:             AMD Ryzen 9 4900HS with Radeon Graphics
      CPU family:           23
      Model:                96
      Thread(s) per core:   2
      Core(s) per socket:   8
      Socket(s):            1
    ```
#### Memory
- Make sure there's enough RAM for both the host and the VM.
- **"Enable shared memory"** is necessary if you plan to use **Looking Glass**.
<p align="center">
	<img src=./media/12.png height=200px>
</p>

#### Storage
- Go to **SATA Disk 1**, change **Disk bus** type to **virtio**.
- In advanced options, change **Cache mode** to  **none** and **Discard mode** to **unmap**. 
- Ensure **SATA CDROM 1** is *read-only*.
<p align="center">
	<img src=./media/13.png height=200px>
</p>

#### Network
- In the **NIC** tab, change the device model from e1000e to **virtio** for better performance. Make sure the link state is **active**
<p align="center">
	<img src=./media/14.png height=180px>
</p>

##### Video
- Leave **QXL** as the video model.

#### Adding VirtIo Drivers
- Click **Add Hardware** in the bottom left corner, select **Storage**
- Select **virtio-win ISO file** *(same directory as your Windows ISO file)*
- Set the **Device Type** to **CDROM device**
- Enable **Readonly** under **Advanced options**
- Click **Finish** to add the Hardware
- Click **Begin Installation** in the top left corner.
<p align="center">
	<img src=./media/15.png height=500px>
</p>


## Windows Installation
The installation process is standard, but you'll need to load the VirtIO storage drivers manually.

**NOTE:** If the installer doesn't detect your virtual disk:
- Click **Load driver**, then **Browse**.
- Navigate to `This PC > CD Drive (virtio-win) > viostor > w10/w11 > amd64`
<p align="center">
	<img src=./media/16.png height=300px>
</p>

- The **Red Hat VirtIO SCSI Controller** should be selected. Click Next.
<p align="center">
	<img src=./media/17.png height=300px>
</p>

- Now your drive should appear. But we’re not done yet.

To fix networking, install the VirtIO network drivers:
- Click **Load driver**, then **Browse**.
- Navigate to `CD Drive (virtio-win) > NetKVM > w10/w11 > amd64`.
- The **Red Hat VirtIO Ethernet Adapter** should appear, and select it to install. 

<p align="center">
	<img src=./media/18.png height=300px>
</p>

- Now select **Drive 0  Unallocated Space** and hit **Next**.

- The system will install and reboot a few times—just let it do its thing. Once you’re in Windows, complete the initial setup and shut down the VM. Now we can move on to GPU passthrough.

## Configuring the VM for GPU Passthrough

Now that your VM is set up, it's time to add your NVIDIA GPU to the virtual machine and install the necessary drivers in Windows 10/11. Before proceeding, you'll need to modify the VM's XML configuration.

### Adding dGPU to the hardware list
- Identify GPU's PCI details. Run 
```shell
lspci -nks 1:00.0
```
Example output
```shell
01:00.0 0300: 10de:2191 (rev a1)
Subsystem: 1043:17ef # This is your dGPU's sub-vendor/device ID
Kernel driver in use: nvidia
Kernel modules: nvidia
```
From this, take note of your **subsystem ID**, **vendor ID**, and **device ID**. In this case, the subsystem ID is `1043:3a44`, with a vendor ID of `17ef` and a device ID of `17ef`. 

- Next, enable XML editing in virt-manager. Open **Edit > Preferences**, navigate to the General tab, check **Enable XML editing**, and click **Close**.

<p align="center">
	<img src=./media/19.png height=350px>
</p>

- Select your VM, then click the second at the top left. Click **Add Hardware** in the bottom left corner, then choose **PCI Host Device** from the list. Select **both** PCI devices labeled NVIDIA Corporation XXXX, then click **Finish**. If another PCI device named "NVIDIA Corporation" appears, repeat the process. 

<p align="center">
	<img src=./media/20.png height=500px>
</p>

### Editing the VM XML Configuration

- Open the **Overview** tab, then switch to the **XML** tab. Modify the `<domain>` tag at the top of the file as follows *(DO NOT APPLY YET!)*:
```xml
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
```
<p align="center">
	<img src=./media/21.png height=200px>
</p>

- Scroll to the bottom of the XML file. Just before `</domain>`, add the following block, adjusting the values accordingly:
```xml
  <qemu:override>
    <qemu:device alias="hostdev0">
      <qemu:frontend>
        <qemu:property name="x-pci-sub-vendor-id" type="unsigned" value="4163"/>
        <qemu:property name="x-pci-sub-device-id" type="unsigned" value="6127"/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
```

To determine the correct values, convert your **subsystem ID** from hexadecimal to decimal. Use an [online converter](https://www.rapidtables.com/convert/number/hex-to-decimal.html). For example, `1043` (hex) becomes **4163** (decimal), and `17ef` (hex) becomes **6127**(decimal). Once you've updated the values, click **Apply**.
<p align="center">
	<img src=./media/22.png height=500px>
</p>

### Testing the VM and Installing NVIDIA Drivers
- Boot into Windows and log in.
- Download the latest NVIDIA drivers from the [**official NVIDIA website**](https://www.nvidia.com/en-us/geforce/drivers/) that match your GPU.

- Install the drivers, then open **Device Manager** and check under D*isplay Adapters*. If everything is set up correctly, your GPU should appear without any error messages.

# VM Enhancements

## Looking Glass – Low-Latency VM Display Without a Second Monitor
Since I don’t use a secondary monitor, **Looking Glass** is a must-have. It offers **insanely low latency** and **zero compression/artifacts** by sharing the video memory directly from the host to the VM. More details [here](https://looking-glass.io/).
### Install Dependencies
First, install the required packages:
```shell
sudo apt install cmake gcc g++ libglvnd-dev fontconfig libspice-protocol-dev make nettle-dev \
    pkg-config binutils libxi-dev libxinerama-dev libxcursor-dev \
    libxpresent-dev libxkbcommon-x11-dev libwayland-dev wayland-protocols \
    libxss-dev libxrandr-dev fonts-dejavu-core dkms linux-headers-$(uname -r)
```
Check wether you're using **PipeWire** or **PulseAudio**
```shell
pactl info | grep "Server Name"
```
If it returns `Server Name: PulseAudio`, run
```shell
sudo apt install libpulse-dev libsamplerate-dev
```

If it returns `Name: PipeWire`, run
```shell
sudo apt install libpipewire-0.3-dev libpulse-dev libsamplerate-dev
```

### Download & Extract Looking Glass
Download the **source package** from [here](https://looking-glass.io/downloads). 

Make a directory called `build` in your home directory. This is just a good practice to reserve a folder to any tools that you want to build from source. 
```shell
mkdir ~/build
```
Now extract the downloaded package to the `build` folder
```shell
tar -xvf ~/Downloads/looking-glass-B6.tar.gz -C ~/build
```
### Compile the client 
Navigate to the client directory and create a `build` folder
```shell
cd ~/build/looking-glass-B6/client
mkdir build && cd build
```

If you are using Wayland, install `libdecor` first
```shell
sudo apt-get install libdecor-0-dev
cmake -DENABLE_LIBDECOR=ON ../
```
Otherwise, just run
```shell
cmake ../
```

Finally, build the client
```shell
make
```

Once done, move the client to `/usr/local/bin` for easier access:
```shell
sudo cp looking-glass-client /usr/local/bin
```

- Test if it's working
```shell
looking-glass-client -h
``` 

### Configuring the VM
In virt-manager open the Windows VM settings. Under **Memory**, check **Enabled shared memory**. 

Then, edit the XML configuration:
- Add this between `<memballoon/>` and `</devices>`
```XML
<shmem name='looking-glass'>
  <model type='ivshmem-plain'/>
  <size unit='M'>32</size>
</shmem>
```
*Adjust the memory size if using 2K/4K resolutions. See this guide for details See this [guide](https://looking-glass.io/docs/B5.0.1/install/#determining-memory) for details.*

- Change the memory ballooning model for performance:
```xml
<memballoon model="none"/>
```
### Set Permissions for Shared Memory
Create a new config file

```shell
sudo nano /etc/tmpfiles.d/10-looking-glass.conf
```
Add the following line (replace `user` with your actual username):
```
#Type Path               Mode UID  GID Age Argument

f /dev/shm/looking-glass 0660 user kvm -
```

Then, manually set permissions 
```shell
sudo touch /dev/shm/looking-glass
sudo chown user:kvm /dev/shm/looking-glass
sudo chmod 660 /dev/shm/looking-glass
```



###  Installing Looking Glass in Windows
**Disable secure boot**
- Boot into BIOS (**F2** or **Del** at startup).
- Navigate to **Secure Boot Configuration**.
- Uncheck **Attempt Secure Boot**.
- Save and exit (**F10**).

**Install Looking Glass Host Application**

- Download the Windows Host Binary from [here](https://looking-glass.io/downloads). 

- Extract and instatll it.

**(Optional) Use a Virtual Display Driver instead of a Dummy HDMI Plug**
- Download [this driver](https://github.com/ge9/IddSampleDriver/releases/tag/0.0.1.2).

- Extract it to `C:\\IddSampleDriver`.
- Open PowerShell as admin and run
```powershell
cd C:\IddSampleDriver
CertMgr.exe /add IddSampleDriver.cer /s /r localMachine root
```
- Edit `C:\IddSampleDriver\option.txt` to match your resolution.
- Install the driver via **Device Manager**: **Action  > Add legacy hardware > Install hardware that I manually select from a list (Advanced) > Have disk > Browse** and select `IddSampleDriver.inf`

### Running Looking Glass
Start your VM, then launch
```
looking-glass-client
```
Press **ScrollLk + F** for fullscreen. More keybindings [here](https://looking-glass.io/docs/B5.0.1/install/#default-key-bindings).
- If you want to remap the "master" key, find the key code [here](https://github.com/torvalds/linux/blob/master/include/uapi/linux/input-event-codes.h) and run
```
looking-glass-client -m <number>
```
e.g: Right Ctrl is 97, so you should type `looking-glass-client -m 97`.

## References
- [VFIO GPU Passthrough Guide](https://asus-linux.org/wiki/vfio-guide)
- [Optimus dGPU Passthrough Guide](https://github.com/mysteryx93/GPU-Passthrough-with-Optimus-Manager-Guide)
- [Supergfxctl Documentation](https://gitlab.com/asus-linux/supergfxctl)
- [Custom Windows Builds](https://windowsxlite.com/Micro10/)