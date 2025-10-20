# KVM on Raspberry PI

## Getting Your Raspberry Pi Ready for KVM

### Hardware Requirements

To successfully run KVM on a Raspberry Pi, you’ll need specific hardware that supports virtualization. The Raspberry Pi 4 Model B is the recommended platform, with a minimum of 4GB RAM, though 8GB is preferred for running multiple virtual machines smoothly. Earlier models like the Pi 3 technically support virtualization but offer limited performance.

For optimal performance, use a Raspberry Pi 4 with:
– 8GB RAM configuration
– High-quality power supply (official 5.1V/3A USB-C recommended)
– Active cooling solution or heatsink
– Class 10 microSD card (minimum 32GB) or USB 3.0 SSD for storage

### Operating System Preparation

Before diving into KVM setup, you’ll need to prepare your Raspberry Pi’s operating system with the necessary components. Start by ensuring you’re running a 64-bit version of Raspberry Pi OS (I used Ubuntu server 22 LTS), as this is essential for KVM virtualization. Update your system by running ‘sudo apt update’ followed by ‘sudo apt upgrade’ to ensure all packages are current.

Next, install the required KVM packages by executing ‘sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst. These packages provide the core virtualization functionality and management tools you’ll need.

Enable CPU virtualization support by adding ‘arm_64bit=1’ and ‘kvm-arm=on’ to your config.txt file, located in the /boot directory. You can do this using the command ‘sudo nano /boot/config.txt’ (the config file for Ubuntu is in /boot/firmware/config.txt).Go to the very bottom of the file and add the following lines. This tells the Pi firmware to enable 64-bit mode (which should already be on, but it's good to be explicit) and to turn on the KVM hardware support:

```
[all]
arm_64bit=1
kvm-arm=on
```

A system reboot may be necessary for all changes to take effect.
Once it's back online, log in and verify the installation.

```bash
sudo apt install cpu-checker
kvm-ok
```

If everything is set up correctly, you should see: **INFO: /dev/kvm exists KVM acceleration can be used**.

## Installing and Configuring KVM

### Installing KVM Packages

* To get started with KVM on your Raspberry Pi, you’ll need to install several essential packages. Open your terminal and first update your package lists by running:

```bash
sudo apt update && sudo apt upgrade -y
```

* Next, install the core KVM packages with the following command:

```bash
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst -y
```

This command installs:
– qemu-kvm: The main virtualization package
– libvirt-daemon-system: The virtualization daemon
– libvirt-clients: Command-line utilities
– bridge-utils: Network bridging tools
– virtinst: Virtual machine installation tools

* After installation, verify that KVM is properly installed by checking the service status:

```bash
sudo systemctl status libvirtd
```

You should see “active (running)” in the output. To ensure your user can manage virtual machines, add yourself to the required groups:

```bash
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

* Remember to log out and back in for these group changes to take effect. Once completed, you can verify the installation by running:

```bash
virsh list –-all
```

This command should execute without errors, showing an empty list of virtual machines.

### Setting Up Network Bridge

!If you don't already have a bridge setup.

* Setting up a network bridge is essential for allowing your virtual machines to communicate with your local network and access the internet. Start by installing the bridge utilities package using the command:

```bash
sudo apt-get install bridge-utils
```

* Create a new bridge interface by editing the /etc/network/interfaces file. Add the following configuration:

```
auto br0
iface br0 inet dhcp
bridge_ports eth0
bridge_stp off
bridge_waitport 0
bridge_fd 0
```

This configuration creates a bridge named ‘br0’ that uses your Raspberry Pi’s ethernet interface (eth0). The bridge will automatically obtain an IP address through DHCP. After saving the configuration, restart the networking service:

```bash
sudo systemctl restart networking
```

When creating new virtual machines, select ‘br0’ as the network interface. This allows your VMs to appear as separate devices on your local network, making it easier to access services running in your virtual machines. Remember to ensure your firewall rules accommodate the bridge interface if you have custom security configurations in place.

### Configuring KVM Parameters 

Optimizing your KVM parameters is crucial for achieving the best performance on your Raspberry Pi. Start by adjusting the memory allocation – a good rule of thumb is to reserve at least 1GB for the host system and allocate the remaining RAM to your virtual machines.You can fine-tune these Raspberry Pi virtualization tricks through the KVM configuration file.

Key parameters to modify include:
– memory_backing_dir: Set this to a fast storage location
– vnc_listen: Configure as “0.0.0.0” for remote access
– max_cpus: Limit to 3 cores, leaving one for the host system
– cache_mode: Use “writeback” for better performance

On an Ubuntu system using **libvirt**, these parameters are set in two different places: a global configuration file for the KVM/QEMU daemon and the specific XML definition file for each virtual machine.

1. Global Configuration (for all VMs)
The main configuration file for the QEMU daemon managed by libvirt is located at /etc/libvirt/qemu.conf.

You can edit it with:

```bash
sudo nano /etc/libvirt/qemu.conf
```

Inside this file, you can set the following parameters (you may need to uncomment them by removing the #):

* vnc_listen = "0.0.0.0" This is the equivalent of the vnc_listen parameter. It allows you to access your virtual machines' graphical consoles over VNC from other computers on your network, not just from the Pi itself.

* memory_backing_dir = "/path/to/fast/storage" This is the memory_backing_dir parameter. If you have a fast external USB SSD, you could create a directory on it (e.g., /mnt/ssd/kvm_mem) and point this here. For most users, leaving this commented out to use the default location is fine.

After saving changes to this file, you must restart the **libvirtd** service to apply them:

```bash
sudo systemctl restart libvirtd
```

2. Per-VM Configuration (for a specific VM)
Most performance tuning (like CPU, RAM, and disk settings) is done on a per-VM basis. You edit a VM's configuration using **virsh**, the command-line tool for **libvirt**.

* To edit a VM named **my-vm**, run:

```bash
sudo virsh edit my-vm
```

This will open the VM's XML configuration file in your default editor.

* Here is where you apply the other parameters:

- CPU Allocation (the **max_cpus** equivalent): The advice to limit the VM to 3 cores (leaving one for the host) is excellent. You do this by setting the **<vcpu>** tag:

```XML
<vcpu placement='static'>3</vcpu>
```

This assigns the VM 3 virtual CPUs.

- Disk Cache Mode (**cache_mode**): For better disk performance, find the **<disk>** section in the XML. Inside the **<driver>** tag, add **cache='writeback'**:

```XML
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='writeback'/>
  <source file='/var/lib/libvirt/images/my-vm.qcow2'/>
  <target dev='vda' bus='virtio'/>
  ...
</disk>
```

Note: **writeback** is faster but carries a small risk of data corruption if the host (your Pi) loses power suddenly. The safer (but slower) default is **cache='none'**.

- Network Model (**virtio**): The advice to use **model=virtio** is correct and is the standard for high-performance networking. When you create a VM with **virt-install** or **virt-manager**, this is almost always the default. In the XML, it looks like this:

```XML
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
  ...
</interface>
```

After you save and close the XML file, **virsh** will automatically validate it. If the VM is running, you may need to shut it down and start it again (**sudo virsh shutdown my-vm** / **sudo virsh start my-vm**) for all changes to take effect.

3. Host System Tuning (CPU Governor)

The final recommendation is to set the CPU governor to performance mode. This is a host system (Ubuntu) setting, not a KVM setting. It forces your Pi's CPU to always run at its maximum clock speed, which can improve VM responsiveness at the cost of higher power use and heat.

* Install the CPU frequency tools:

```bash
sudo apt install cpufreq-utils
```

Set all CPU cores to **performance** mode:

```bash
sudo cpufreq-set -g performance
```

This setting will reset when you reboot. To make it permanent, you would need to create a **systemd** service or use another method to run this command on boot.


# Create, start and configure KVM virtual machines from the console

This guide outlines the procedure for creating and managing a KVM (Kernel-based Virtual Machine) on a **Raspberry Pi 4** running **Ubuntu Server 22.04 LTS**, all from the command line.

This guide assumes you have already installed the necessary KVM packages (`qemu-kvm`, `libvirt-daemon-system`, `bridge-utils`, `virtinst`) and enabled KVM in `/boot/firmware/config.txt` as per previous steps.

---

## Step 1: Prerequisites

Before creating a VM, you need two things: an ARM64-compatible OS image and the UEFI firmware for QEMU.

### 1. Download an ARM64 OS Image

You **must** use an `aarch64` (ARM64) installer. Standard `x86_64` (Intel/AMD) ISOs will not work. We will use the Ubuntu 22.04 Server image. (or ubunto core 24)

```bash
# Create a directory for your ISOs
mkdir -p ~/iso
cd ~/iso

# Download the ARM64 cloud image
wget [https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-arm64.img](https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-arm64.img)

https://ubuntu.com/download/raspberry-pi/thank-you?version=24&architecture=core-24-arm64+raspi
````

### 2\. Install UEFI Firmware Support

ARM64 VMs require UEFI to boot. This package provides the necessary files.

```bash
sudo apt install qemu-efi-aarch64
```

-----

### Step 2: Prepare Your SSH Key

`cloud-init` will copy your **public** SSH key into the new VM, allowing you to log in. You must have an SSH key pair on your Raspberry Pi (the host).

1.  **Check for an existing key:**

    ```bash
    ls ~/.ssh/id_rsa.pub
    ```

2.  **If you don't have one**, create a new key pair. Press Enter to accept all the defaults:

    ```bash
    ssh-keygen -t rsa
    ```

    Your public key is now at `~/.ssh/id_rsa.pub`.

-----

### Step 3: Prepare the Virtual Disk

The downloaded image is small and will be used as a "base." We need to copy it to the KVM image directory and resize it to its final size (e.g., 20GB).

1.  **Copy the image** to the default libvirt directory:

    ```bash
    # (We'll name the new VM's disk 'ubuntu-ssh-vm.qcow2')
    sudo cp ~/images/jammy-server-cloudimg-arm64.img /var/lib/libvirt/images/ubuntu-ssh-vm.qcow2
    ```

2.  **Resize the image** to 20GB (or your desired size):

    ```bash
    sudo qemu-img resize /var/lib/libvirt/images/ubuntu-ssh-vm.qcow2 20G
    ```

    *(The VM will automatically resize its partitions on the first boot).*

-----

## Step 4: Create and Start the VM

We will use `virt-install` to create, and start the new virtual machine in a single command.

**Note:** Remember to replace `your_username` with your actual username.

```bash
sudo virt-install \
--name ubuntu-vm-01 \
--ram 1024 \
--vcpus 1 \
--arch aarch64 \
--boot uefi \
--disk path=/var/lib/libvirt/images/ubuntu-ssh-vm.qcow2,import,bus=virtio,format=qcow2 \
--network network=default,model=virtio \
--graphics none \
--os-variant ubuntu22.04 \
--cloud-init ssh-key=/home/your_username/.ssh/id_rsa.pub
```

### Command Breakdown:

  * `--name`: The unique name for your VM.
  * `--ram 1024`: Allocates 1GB of RAM to the VM.
  * `--vcpus 2`: Allocates 1 virtual CPU cores.
  * `--arch aarch64`: **(Critical)** Tells KVM to create an ARM64 machine.
  * `--boot uefi`: **(Critical)** Tells the VM to use the ARM UEFI firmware.
  * `--disk ...`: We specify the disk we just prepared and add `--import` to tell KVM it's a pre-installed image.
  * `--graphics none`: This creates a truly headless VM with no VNC or graphical console.
  * `--cloud-init ssh-key=...`: This is the magic. It tells `cloud-init` inside the VM to add your public SSH key to the authorized users.
  * `--network ...`: Connects the VM to the `default` libvirt NAT network.
  * `--os-variant ...`: Optimizes the VM for Ubuntu 22.04.

The VM will start, configure itself in seconds, and be ready on the network.

-----

### Step 5: Get the VM's IP Address

The VM will get an IP address from the `default` libvirt network. You need to find this IP to connect.

1.  Wait about 30 seconds for the VM to boot and get an IP.

2.  Run this command (it may take a few tries):

    ```bash
    sudo virsh domifaddr ubuntu-ssh-vm
    ```

3.  You will see an output like this. You need the `ipv4` address:

    ```
    Name       MAC address          Protocol     Address
    -------------------------------------------------------------------------------
    vnet0      52:54:00:1a:2b:3c    ipv4         192.168.122.101/24
    ```

-----

### Step 6: Connect with SSH

You can now SSH directly into the VM.

  * **Username:** The default for Ubuntu cloud images is `ubuntu`.
  * **IP Address:** The one you just found.

<!-- end list -->

```bash
ssh ubuntu@192.168.122.101
```

You should be logged in immediately without a password, as it's using your SSH key for authentication.

-----

### Step 7: Manage the VM (`virsh`)

You can manage this VM with the same `virsh` commands as any other:

  * **List all VMs:** `virsh list --all`
  * **Start:** `virsh start ubuntu-ssh-vm`
  * **Shut down:** `virsh shutdown ubuntu-ssh-vm`
  * **Force-stop:** `virsh destroy ubuntu-ssh-vm`
  * **Enable auto-start:** `virsh autostart ubuntu-ssh-vm`
  * **Edit configuration:** `virsh edit ubuntu-ssh-vm`

<!-- end list -->
---

### Editing VM Configuration

You can change VM settings (like RAM or CPU) by editing its XML configuration file.

```bash
virsh edit ubuntu-vm-01
```

This will open the VM's XML file in your default editor (like `nano`).

  * **To change RAM:** Find the `<memory ...>...</memory>` tag and change the number.
  * **To change CPUs:** Find the `<vcpu ...>...</vcpu>` tag and change the number.

Save and exit the file. Most changes will only apply after you shut down and restart the VM.

