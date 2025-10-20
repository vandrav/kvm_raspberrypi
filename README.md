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
--import
--disk path=/var/lib/libvirt/images/ubuntu-ssh-vm.qcow2,bus=virtio,format=qcow2 \
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
  * `--import ells virt-install to skip the operating system installation step.Instead of booting from an ISO file or network installer, it assumes the disk image you are providing (the .qcow2 file) is already a pre-installed, bootable system.`
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
    sudo virsh domifaddr ubuntu-vm-01
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
  * **Start:** `virsh start ubuntu-vm-01`
  * **Shut down:** `virsh shutdown ubuntu-vm-01`
  * **Force-stop:** `virsh destroy ubuntu-vm-01`
  * **Enable auto-start:** `virsh autostart ubuntu-vm-01`
  * **Edit configuration:** `virsh edit ubuntu-vm-01`

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


Yes, absolutely. That's a great idea.

Here is a new "Troubleshooting" section for your `README.md` that combines all the problems we solved. You can just append this to the end of the file.

-----

````markdown
##  troubleshooting

Here are some common errors and how to fix them.

---

### Problem: `virsh undefine` fails with "cannot undefine domain with nvram"

* **Symptom:** You run `sudo virsh undefine ubuntu-vm-01` and get this error:
    ```
    error: Failed to undefine domain 'ubuntu-vm-01'
    error: Requested operation is not valid: cannot undefine domain with nvram
    ```
* **Reason:** The VM was created with `--boot uefi`, which stores boot settings in a separate `nvram` file. `virsh` won't delete the VM unless you also agree to delete that file.
* **Solution:** Add the `--nvram` flag to the command:
    ```bash
    sudo virsh undefine ubuntu-vm-01 --nvram
    ```

---

### Problem: `virt-install` fails with "Size must be specified for non existent volume"

* **Symptom:** You run `virt-install` and get this error:
    ```
    ERROR    Error: --disk path=...: Size must be specified for non existent volume ...
    ```
* **Reason:** You used the `--import` flag, which tells `virt-install` to use an *existing* disk. However, the disk file at the path you specified (e.g., `/var/lib/libvirt/images/ubuntu-vm-01.qcow2`) does not exist. This is usually due to a typo or a missed "copy" step.
* **Solution:** Make sure you have copied the base cloud image to the correct path *before* running `virt-install`.
    ```bash
    # Check if the file exists
    ls /var/lib/libvirt/images/ubuntu-vm-01.qcow2
    
    # If not, create it:
    sudo cp ~/images/jammy-server-cloudimg-arm64.img /var/lib/libvirt/images/ubuntu-vm-01.qcow2
    ```

---

### Problem: VM boots, but has no IP address

This is the most common and complex issue.

* **Symptom:** The VM starts, but `sudo virsh net-dhcp-leases default` shows an empty list.
* **Diagnosis:** Connect to the VM's text console to see its boot messages.
    ```bash
    sudo virsh console ubuntu-vm-01
    ```
    (To exit the console, press **`Ctrl`** + **`]`**)

* **You See:** The VM boots but hangs, and you see these errors:
    ```
    [FAILED] Failed to start Wait for Network to be Configured.
    ...
    [FAILED] Failed to start OpenBSD Secure Shell server.
    ...
    ubuntu login:
    ```
* **Reason:** The default Ubuntu cloud image is not automatically configuring its network. The `cloud-init` process fails because it can't detect the `virtio` network card and get an IP.
* **Solution:** We must force the network configuration by passing a `cloud-init` user-data file.

**1. Full Cleanup:**
You must start from a fresh disk, as `cloud-init` only runs on the first boot.
```bash
sudo virsh destroy ubuntu-vm-01
sudo virsh undefine ubuntu-vm-01 --nvram
sudo rm /var/lib/libvirt/images/ubuntu-vm-01.qcow2
````

**2. Create `user-data.yaml`:**
Create a file in your home directory (`nano ~/user-data.yaml`). This file will:

1.  Set a fallback password (`ubuntu`) so you can log in via the console.
2.  Add your SSH key.
3.  **Explicitly** tell the VM to use the `virtio_net` driver and get an IP via DHCP.

<!-- end list -->

```yaml
#cloud-config
user: ubuntu
password: ubuntu
chpasswd: { expire: False }
ssh_pwauth: True
ssh_authorized_keys:
  - ssh-rsa YOUR_SSH_PUBLIC_KEY_HERE

network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      match:
        driver: virtio_net
```

**Important:** Replace `ssh-rsa YOUR_SSH_PUBLIC_KEY_HERE` with your *actual* public key (get it with `cat ~/.ssh/id_rsa.pub`).

**3. Re-create the VM (with the fix):**
First, create a fresh disk.

```bash
sudo cp ~/images/jammy-server-cloudimg-arm64.img /var/lib/libvirt/images/ubuntu-vm-01.qcow2
sudo qemu-img resize /var/lib/libvirt/images/ubuntu-vm-01.qcow2 20G
```

Now, run `virt-install`, replacing the `--cloud-init ssh-key=...` flag with `--cloud-init user-data=...`. (Assuming your username is `w`).

```bash
sudo virt-install \
--name ubuntu-vm-01 \
--ram 1024 \
--vcpus 1 \
--arch aarch64 \
--boot uefi \
--import \
--disk path=/var/lib/libvirt/images/ubuntu-vm-01.qcow2,bus=virtio,format=qcow2 \
--network network=default,model=virtio \
--graphics none \
--os-variant ubuntu22.04 \
--cloud-init user-data=/home/w/user-data.yaml
```

After 60 seconds, check `sudo virsh net-dhcp-leases default` again. It should now have an IP.

```
```

This is incredibly frustrating. The "Login incorrect" error is the final clue.

It confirms that your `user-data.yaml` file is **not being processed at all**. If it were, it would have set the password to `ubuntu`, and you would have been able to log in. The YAML syntax is likely perfect, but the `cloud-init` process *inside* this specific cloud image is failing to find or read it.

At this point, we've proven the `cloud-init` method is unreliable for your setup.

-----

### A New Plan: The "Installer ISO" Method

Let's abandon the cloud image completely. It's the source of the problem.

We'll use the **standard installer ISO** (the `.iso` file, not the `.img` file). This is a slower, manual installation, but it is **far more reliable** and completely bypasses `cloud-init`.

The process will be:

1.  Create a VM with an empty disk and the installer ISO attached.
2.  Connect to the VM's **text console** (no VNC/graphics needed).
3.  Perform the standard Ubuntu Server installation *inside the console*.
4.  Reboot. The VM will now have an IP and the SSH server you enabled.

-----

### Step 1: Full Cleanup (Last Time)

Let's clear out the failed VM.

```bash
sudo virsh destroy ubuntu-vm-01
sudo virsh undefine ubuntu-vm-01 --nvram
sudo rm /var/lib/libvirt/images/ubuntu-vm-01.qcow2
```

We can also delete the `user-data.yaml` file; we won't need it.

-----

### Step 2: Download the ISO

Here are the commands to create the `iso` directory and download the correct **ARM64** installer file.

```bash
# Create the directory
mkdir -p /home/w/iso

# Go into that directory
cd /home/w/iso

# Download the Ubuntu Server 22.04.5 ARM64 ISO
wget https://cdimage.ubuntu.com/releases/22.04/release/ubuntu-22.04.5-live-server-arm64.iso
```

# Move the ISO file:**

```bash
sudo mv /home/w/iso/ubuntu-22.04.5-live-server-arm64.iso /var/lib/libvirt/images/
```
-----
### Step 3: The New `virt-install` Command

This command is different.

  * It does **not** use `--import`.
  * It does **not** use `--cloud-init`.
  * It creates a **new, empty disk** by specifying `size=20`.
  * It attaches the installer ISO using `--cdrom`.
  * It uses `--extra-args` to force the installer into our text console.

*(Note: I'm setting RAM to 2048MB. 1024MB is very small for an OS installation and can cause it to fail.)*

```bash
sudo virt-install \
--name ubuntu-vm-01 \
--ram 2048 \
--vcpus 1 \
--arch aarch64 \
--boot uefi \
--disk path=/var/lib/libvirt/images/ubuntu-vm-01.qcow2,size=10,format=qcow2 \
--network network=default,model=virtio \
--graphics none \
--os-variant ubuntu22.04 \
--cdrom /var/lib/libvirt/images/ubuntu-22.04.5-live-server-arm64.iso
```

-----

### Step 4: Install Ubuntu Server (In Your Console)

When you run the command above, your terminal will immediately connect to the VM's console. You will see the **Ubuntu Server installer** boot up in text mode.

1.  Follow the on-screen installer prompts (select language, keyboard, etc.).
2.  **Network:** It will ask to configure the network. It should auto-detect `eth0` and successfully get an IP from `192.168.122.x` via DHCP. Just confirm and proceed.
3.  **Storage:** Use the default "Use entire disk" option.
4.  **Profile Setup:** This is the most important part.
      * Set your server's name (e.g., `ubuntu-vm-01`).
      * Set your **username** (e.g., `ubuntu` or `w`).
      * Set your **password**.
5.  **SSH Setup:** The installer will ask you:
      * **Install OpenSSH server:** **Press Spacebar to select [X]**.
      * It will ask to import SSH keys. You can skip this, as we'll just use the password for the first login.
6.  **Install:** Let the installation run. It will take several minutes.
7.  **Reboot:** When it's finished, it will say "Reboot Now". Select it.

-----

### Step 5: First Login via SSH

The VM will reboot, and you will be disconnected from the console (it may hang, just press Enter). Press `Ctrl` + `]` to exit if needed.

The VM is now running.

1.  **Find its IP address:**

    ```bash
    sudo virsh net-dhcp-leases default
    ```

    This time, it *will* have an IP, assigned by the installer.

2.  **SSH into your new VM:**
    Use the username and password **you just created** during the installation.

    ```bash
    ssh your_new_username@THE_NEW_IP_ADDRESS
    ```

This method is guaranteed to work because it bypasses all the cloud-init automation that was failing.