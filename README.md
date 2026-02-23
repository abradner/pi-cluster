# Raspberry Pi Proxmox + Talos Kubernetes Cluster Setup

## The Architecture: The "Why"

This repository exists to automate the deployment of a highly specific, highly resilient edge computing architecture: **Running a virtualized Kubernetes Control Plane on Raspberry Pi 5 hardware via Proxmox VE (arm64).**

While running Kubernetes natively on Raspberry Pis is common, this project makes several distinct, advanced architectural choices:

### 1. Why Proxmox and Virtualized Control Planes?
Typically, running a Talos Control Plane uses the entire physical node. In this lab, the actual Kubernetes *workloads* will run on a separate fleet of bare-metal Raspberry Pi Compute Modules (CM4/CM5). 
These three Pi 5 nodes exist *exclusively* to host the Control Plane.

By virtualizing the Control Planes inside Proxmox instead of flashing Talos directly to the metal:
- **Resiliency & Backup:** A virtualized Control Plane can be cleanly snapshotted by Proxmox. If an `etcd` database corrupts or a K8s upgrade spectacularly fails, you can roll back the entire Control Plane VM in seconds. 
- **Hardware Abstraction:** If a Pi hardware component fails, the Proxmox VM disk image can be seamlessly migrated or restored onto any other Pi in the cluster. It sits "above the hardware" as a drop-in replacement.
- **Resource Segmentation:** An 8GB Pi has plenty of overhead. Virtualization allows us to cleanly allocate 4GB to the Kubernetes API, leaving the remaining memory for robust ZFS caching on the host without risking out-of-memory kernel panics killing the K8s API server.

### 2. Why ZFS Mirroring?
Raspberry Pis are notorious for micro-SD card failures under heavy I/O workloads (like Kubernetes databases). 
- **The Choice:** We completely bypass the SD card for data storage. The playbook enforces the creation of a `data` ZFS pool mirrored across two physical NVMe SSDs hat-mounted on the Pi.
- **The Benefit:** ZFS provides real-time data scrubbing, bit-rot protection, and automatic failover if an NVMe drive dies. The VMs boot and run entirely off this redundant block storage.

### 3. Networking: The Backplane
A Proxmox cluster (Corosync) expects high availability. If the primary network switch reboots, Corosync will panic and start fencing (rebooting) nodes.
To prevent this, the playbook automatically discovers optional secondary USB Network Adapters and binds them into a dedicated, isolated Backplane subnet (`10.10.99.x`). The cluster heartbeat travels over this direct cable, isolating it from home network chaos.

### 4. TLS Certificates (The acme.sh Workaround)
Proxmox has a native ACME client, but Debian Trixie's updated security patches currently break the `pvenode` local REST deployment logic for some cryptographic operations. 
We avoided the native client and instead implemented `acme.sh` pushing directly via `DNS-01` to Cloudflare. This completely bypasses the broken system dependencies and ensures the Proxmox GUI is always served via valid, trusted TLS without opening any firewall ports (HTTP-01).

### The Shortcomings (What you need to know)
- **ARM64 Translation:** Proxmox officially supports `x86_64`. The `arm64` port relies on specific kernel forks (PXVirt repository). While incredibly stable, it requires careful manipulation of hardware abstractions (we use `aarch64` and `OVMF` in the Talos VMs) to boot successfully. 
- **SD Card Fragility:** The base OS still boots from an SD card. While all high-I/O data was moved to the NVMe ZFS array to mitigate wear, the SD card remains a single point of failure for the host bootloader.
- **Talos Kubelet Runtimes:** The Talos Image Factory doesn't provide natively compiled GPU or NPU drivers (like Hailo-8) for ARM64 by default. If you intend to do edge AI, you will need to passthrough those PCIe devices directly to bare-metal workers rather than virtualized Control Planes.

---

## 1. Proxmox Bare-Metal Automation

This guide covers how to deploy the Proxmox environment to a fresh Raspberry Pi node using the provided Ansible playbook.

## Prerequisites

1.  Flash a Raspberry Pi 5 with **Raspberry Pi OS Lite (64-bit)**.
2.  During the imaging process (e.g., in Raspberry Pi Imager), ensure you configure:
    *   **Hostname**: e.g., `k8sctl-pi5r`
    *   **Username and Password**: Make sure you set these since you want to be able to run headless
    *   **SSH**: Enable SSH (use key-based authentication, don't use password authentication).
3.  Boot the Pi, connect it to your network, and ensure you can SSH into it from your main computer (but don't do anything else).
4.  Ensure Ansible is installed on your main computer.
    *  I've used [mise](https://mise.jdx.dev/getting-started.html) to manage my ansible installation (see `mise.toml`)
    *  so you should be able to `mise install` and be good to go.
    *  You could use `brew`, you _should_ try `mise`.

## Inventory Configuration
1.  Copy `hosts.example.ini` to `hosts.ini`.
2.  Open `hosts.ini` and add your new node under the `[pve_nodes]` group.
3.  Provide the `ansible_host` (the IP address the Pi currently received from DHCP)

    ```ini
    [pve_nodes]
    # left node
    k8sctl-pi5l ansible_host=10.10.80.50
    # right node
    k8sctl-pi5r ansible_host=10.10.80.51
    ```

    *   You can find the current IP address of the Pi by running `ip addr show` on the Pi or in your router's DHCP client list.
    *   Consider setting a static lease on your router/dhcp server for the Pis. We'll lock it in with a static config as well, but this will ensure your DHCP doesn't double-allocate this address to a new client.
    *   Reboot your Pis after any dhcp/ip related changes. Simplest way to avoid pain.
4.  Set the variables in `[pve_nodes:vars]` according to your use case:
    ```ini
    [pve_nodes:vars]
    user_name=your_username
    locale=en_AU.UTF-8 # G'day

    # (Optional) ACME.sh TLS Automation
    # acme_domain_suffix=k8s.myhomelab.com
    # acme_email=your_email@example.com
    # acme_dns_provider=dns_cf # Check acme.sh documentation for your DNS API provider code
    # acme_env='{"CF_Token": "your_cloudflare_api_token"}' # Provide the required environment variables for your DNS API as a JSON dictionary
    ```
    If the optional `acme_` variables are fully populated, the playbook will automatically install `acme.sh`, use the DNS-01 challenge to issue an FQDN certificate natively for the node (`inventory_hostname.acme_domain_suffix`), and deploy it to the Proxmox GUI.

## Running the Playbook

To provision the node, you will run the `proxmox-pi-setup.yaml` playbook.

Ansible will rely on the user and password you already created during the Raspberry Pi Imager process (via cloud-init) to set up Proxmox Administrator access later. You just need to provide your `sudo` password to Ansible so it can run elevated commands.

**Read the notes below first**, but the command to run is:

```bash
ansible-playbook proxmox-pi-setup.yaml -l k8sctl-pi5r -K -e tailscale_auth_key="tskey-auth-..."
```

The playbook will take about 25min all up, mostly during the ZFS and Proxmox installation phase. Once it completes, it will reboot the Pi.

1.  Wait a few minutes for the Pi to reboot.
2.  Navigate to `https://<node_ip>:8006` in your web browser.
3.  Log in using your username (`your_username`) and password, ensuring the Realm is set to **Linux PAM standard authentication**.

## Notes

### Anatomy of the command

Make sure you understand what the switches mean and adjust to your needs:
*   `-l k8sctl-pi5r`: Limits the execution to just the one node
    * prevents re-running against existing nodes
    * you **can omit this** if you're ready to roll out all your nodes at once.
*   `-K`: Prompts you for the BECOME password (aka the `sudo` password for the Pi user).
    * This one is essential.
*   `-e tailscale_auth_key="..."`: (Optional) Provides your Tailscale auth key. If omitted, the Tailscale installation step will be skipped.
    * Make sure you created a `Reusable` key


### Proxmox Administrator Setup:

If the `Grant user Administrator rights in Proxmox` task fails during the playbook run (which can happen occasionally if the PVE services haven't fully started before the reboot), you can manually grant access once you SSH back into the node after it reboots:

```bash
sudo pveum acl modify / -user your_username@pam -role Administrator
```

### Workaround Raspberry Pi OS first-boot initramfs corruption

There is a step in the playbook that attempts to fix a first-boot initramfs corruption issue on Raspberry Pi OS:

```yaml
    - name: Workaround Raspberry Pi OS first-boot initramfs corruption
      shell: |
        if [ -f /etc/initramfs-tools/update-initramfs.conf ] && ! grep -q 'update_initramfs=' /etc/initramfs-tools/update-initramfs.conf; then
          rm -f /etc/initramfs-tools/update-initramfs.conf
          dpkg --configure -a || true
          apt-get update && apt-get install -y --reinstall initramfs-tools e2fsprogs
        fi
      changed_when: false
```

This step exists because I've _consistently_ noticed corruption on first boot of Raspberry Pi OS on the Pi 5 in the current build of Pi OS Lite (2025-12-04 trixie):

```bash
$ sudo apt full-upgrade
Summary:
  Upgrading: 0, Installing: 0, Removing: 0, Not Upgrading: 0
  2 not fully installed or removed.
  Space needed: 0 B / 9,772 MB available

Continue? [Y/n]
Setting up initramfs-tools (0.148.3+rpt2) ...
/usr/sbin/update-initramfs: 1: /etc/initramfs-tools/update-initramfs.conf: ��Eݞ1i��1i: not found
dpkg: error processing package initramfs-tools (--configure):
 installed initramfs-tools package post-installation script subprocess returned error exit status 127
Setting up e2fsprogs (1.47.2-3+b7) ...
/usr/sbin/update-initramfs: 1: /etc/initramfs-tools/update-initramfs.conf: ��Eݞ1i��1i: not found
dpkg: error processing package e2fsprogs (--configure):
 installed e2fsprogs package post-installation script subprocess returned error exit status 127
Errors were encountered while processing:
 initramfs-tools
 e2fsprogs
Error: Sub-process /usr/bin/dpkg returned an error code (1)
```

The current working theory is:
> When you flash a Raspberry Pi OS image to an SD card or NVMe and boot it for the very first time, the OS runs an automated resize process (resize2fs) during the boot sequence to expand the root filesystem to fill the rest of the disk.
>
> If the storage media is slightly slow, if power fluctuates even slightly, or if there is a bug in the specific release's partition expansion logic, a race condition occurs. 
>
> /etc/initramfs-tools/update-initramfs.conf appears to be sitting exactly on the structural boundary of the disk sector that gets tangled during the expansion.
>
> The weird `Eݞ1i1i` characters aren't a misconfigured locales, rather, they are literal raw binary garbage from the disk sector!

If you're seeing issues with the playbook, this is a good place to start.
And if the issue is resolved with future releases: this step should become inert and can be deleted safely.
