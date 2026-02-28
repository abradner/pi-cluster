# Raspberry Pi Proxmox + Talos Kubernetes Cluster Setup

> [!IMPORTANT]
> This readme is long, but with good reason: You're about to spend a good chunk of money and you'll have expectations on how that works. I want you to go into that journey _well informed_ and not have regrets or understanding gaps.

## The Philosophy:

First up: This is a **very opinionated guide**. There are plenty of substitutions you could make, but each decision point wraps a balance of trade-offs. It's important to fully understand them before changing things and I'll try to articulate them as clearly as possible.

If there are gaps in the explanation, issues with my reasoning or other strictly-better options: please open an issue and we can discuss there.

### A major caveat:
The number one priority in this project is **data resilience**, _not_ uptime. 

I'd anticipate >99% uptime (<7 hours degraded service in a month) but <99.95% (more than 45 minutes a month) in my environment.

There are a few small changes that can be made to add more nines, but if that's your priority then this _isn't_ your guide.


## The Architecture:

### The "What":
This repository exists to automate the deployment of a **highly resilient** homelab/smb k8s cluster using **comodity hardware** for **less than the cost of a modest annual AWS bill** (including ongoing costs).

### The "Why":
You've heard of the Fast/Cheap/Good triangle - you can pick two. For this analogy I'll say:
- "Fast" means "Enough performance per watt" - Every workload is different.
- "Cheap" means "Low upfront and ongoing hardware-related cost" 
- "Good" means "Reliable and maintainable"

Given the wide variety of power costs, internet costs and workloads it's tricky to incorporate that directly so my "fast" and "cheap" might be a controversial take but bare with me.

Anyway:
- Enterprise gear is fast and good, but not cheap.
- Commodity hardware is cheap and fast, but not good. 
- There is no genuine middle of the road: It's just not profitable to make it and second hand gear is a nightmare to maintain (because it relies on having enough gear being available on the market at the right time).

We're finally in an era where the cheapest[^1] commodity hardware is finally fast enough such that - with clever arrangement of parts - **we can make it fast, cheap _and_ good**.

[^1]: "Cheap" is relative. The AI-pocalypse has driven up the price of RAM, SSDs and - by proxy - Raspberry Pis, but that's an entire industry problem and while some prices might haven't caught up: eventually they will until the supply chain settles. Even without price hikes, this is still about a thousand US dollars worth of hardware.

### The "How":
We achieve that by 
- Using hardware that is "fast enough" to get the job done
- Using hardware that has become (de facto) standardised and widely available
- Using some now-excellent open source software to orchestrate it all

And most importantly:
- Spreading the typical failure modes such that the **total risk is is divided** rather than additive or multiplicative.

## The BOM

> [!NOTE]
> The purchase links -
> - Aren't guaranteed to be the best price
> - Aren't affiliate links
> - Are so you have a springboard to find the right gear. 
> - You know how eCommerce works. Do your own eCom.

### Consumables
> "Cheaper than printer ink but will still lighten your wallet"

- lots of 16-64gb SD cards (at least one for every node including control planes, Use the best ones in the control planes)
  - Purchase Links: (I don't have any good recommendations at the moment)
- 6x128 or 256gb NVMe SSDs (two per control plane)
  - Purchase Links: (I don't have any good recommendations at the moment)
- 9x Short Ethernet cables (for the nodes)
  - Purchase Links: [AU 1](https://www.amazon.com.au/dp/B079583DDT), [AU 2](https://www.amazon.com.au/dp/B0FSX4DSZ3)
- A few Ethernet cables for uplinks
  - You have some in that basket you refuse to organise. Don't buy more.
- Access to a 3D Printer (for your custom rack mounts)

### Infrastructure

- 10" Mini rack ≥ 4u
  - Purchase Links: [AU 1](https://www.amazon.com.au/dp/B0FHD8V32R), [AU 2](https://www.amazon.com.au/dp/B0FHV9K5P2)
- PoE Switch (with at least 9 ports PoE+ and a power budget of at least 100w)
  - Purchase Links: [1GBe](https://www.aliexpress.com/item/1005007045849781.html), [2.5GBe](https://www.aliexpress.com/item/1005007635108859.html)
- Carrier Boards & Chassis for your workers 
  - 6x [Xerxes Pi](https://store.rapidanalysis.com/product/xerxes-pro-bundle/) (Recommended)
  - DeskPi Super6C [Board](https://deskpi.com/products/deskpi-super6c-raspberry-pi-cm4-cluster-mini-itx-board-6-rpi-cm4-supported?variant=44454335512732), [Case](https://deskpi.com/products/deskpi-itx-case-kit-for-deskpi-super6c-raspberry-pi-cm4-cluster-mini-itx-board)
- Adequate rack/chassis cooling
  - At least SOME fans to pull air past the CMs and out of the chassis

### Control Plane

3 each of
- Raspberry Pi 5 8gb
  - Purchase links: [AU](https://core-electronics.com.au/raspberry-pi-5-model-b-8gb.html)
- Pimoroni NVMe Duo
  - Purchase links: [AU](https://core-electronics.com.au/nvme-base-duo-raspberry-pi-5-ssd-expansion.html)
- Raspberry Pi Active Cooler
  - Purchase links: [AU](https://core-electronics.com.au/raspberry-pi-5-active-cooler.html)
- waveshare POE hat (G or H)
  - Purchase links: [AU](https://core-electronics.com.au/power-over-ethernet-hat-g-raspberry-pi-5-5v-5a.html)

### Worker Nodes

- 6 each of
  - CM5008000 (No wifi, 8gb, No flash)
    - Purchase links: [AU](https://core-electronics.com.au/raspberry-pi-compute-module-5-8gb-non-wireless-lite.html)
  - Compute Module 5 Passive Cooler 
    - Purchase links: [AU](https://core-electronics.com.au/raspberry-pi-compute-module-5-passive-cooler.html)


## The Design (and why it's a bit different)

### 1. Why Proxmox and Why Virtualized Control Planes?
Typically, running a Talos Control Plane uses the entire physical node. In this lab, the actual Kubernetes *workloads* will run on a separate fleet of bare-metal Raspberry Pi Compute Modules (CM4/CM5/Anything Talos can support). 

These three Pi 5 nodes exist *exclusively* to host the Control Plane.

By virtualizing the Control Planes inside Proxmox instead of flashing Talos directly to the metal:
- **Resiliency & Backup:** A virtualized Control Plane can be cleanly snapshotted by Proxmox. If an `etcd` database corrupts or a K8s upgrade spectacularly fails, you can roll back the entire Control Plane VM in seconds. 
- **Hardware Abstraction:** If a Pi hardware component fails, the Proxmox VM disk image can be seamlessly migrated or restored onto any other Pi in the cluster. It sits "above the hardware" as a drop-in replacement.
- **Resource Segmentation:** An 8GB Pi has plenty of overhead. Virtualization allows us to cleanly allocate 4GB to the Kubernetes API, leaving the remaining memory for robust ZFS caching on the host without risking out-of-memory kernel panics killing the K8s API server.

// todo: PVE can also serve PVCs for the worker nodes

### 2. Cattle and Pets

#### The cattle:

In this design, **every phyiscal node is cattle** (Including the control plane): If _anything_ fails we can[^2] cordon the node and treat it as dead.
The idea being that you can rip out the failed node, replace the failing component and **trivially reprovision it back into the cluster**

[^2]: can, not will: You don't want fleeticide because someone rebooted a switch or bumped a cable

#### The pets:

**Data!** This includes your operating data of course, but also your **configuration data** (and state)
- Control plane VMs store and own the state of the cluster
- Configuration data is versioned and stored in git (and pushed to a remote)
- Workers handle but _never_ own or store data
- Control Plane Host's ZFS pools _can_ serve the PVCs for the workers if absolutely required[^3]
- Snapshotting and backup solutions keep that data safe from disaster

[^3]: This does introduce more risk and is not a replacement for proper storage and handling

### 3. Why ZFS Mirroring?
NAND flash (SD cards in particular but SSDs as well) is notorious for failure under heavy I/O workloads (like Kubernetes databases).
While ZFS does contribute to write amplification, the functionality it provides is worth it for the net-increase in resiliency.
- **The Benefit:** ZFS provides real-time data scrubbing, bit-rot protection, and automatic failover if an NVMe drive dies. The VMs boot and run entirely off this redundant block storage.

### 4. The Backplane network / VLAN
A Proxmox cluster (Corosync) expects high availability and high throughput access to the other nodes in the cluster. If connectivity is lost Corosync will panic and start fencing and rebooting nodes.
The reliable connectivity is required for state to remain in sync (same principal as etcd in K8s), the high throughput is required for efficient VM migration and replication. 

We can achieve this via a dedicated network interface. Pi5 has USB3.0 ports and 2.5gbe adapters and switching is now cheap and plentiful.
In practice this means that we have the onboard interface dedicated to k8s cluster traffic and the higher speed USB interface for bursty Proxmox cluster traffic.

The playbook handles this by automatically discovering a (technically optional, but recommended) secondary USB Network Adapter and binding it into the Backplane subnet (eg `10.10.99.x`) which is physically conneceted to a different switch.
The cluster uses the backplane as its primary path for cluster communications, but both paths are active and available for use if something fails.

### 6. Gotchas and caveats (What you need to know)
- **ARM64 Translation:** Proxmox officially supports `x86_64`. The `arm64` port is operated by a third party and relies on specific kernel forks (PXVirt repository). While stable, it's not "official" and could introduce supply chain concerns. 
- **SD Card Fragility:** The base OS still boots from an SD card on all nodes. While we've mitigated the impact of card failure and worked to reduce the possibility, this will be the first thing to fail in this cluster so don't use the bottom-of-the-barrel cards.
- **TLS Certificates (The acme.sh Workaround):** Proxmox has a native ACME client, but Debian Trixie's updated security patches currently break the `pvenode` local REST deployment logic for some cryptographic operations (And the proxmox fix hasn't been backported to the PXVirt fork yet). I've ignored the native client for now and instead implemented `acme.sh` using the DNS challenge via Cloudflare. This is a workaround until PXVirt fully supports Trixie.

## Bringing it all together 

### The hardware build

> [!NOTE]
> Coming soon

### The software build - Control Plane

Ansible is used to automate the deployment of the Proxmox environment to a fresh Raspberry Pi node.

#### Bootstrapping the Control Plane nodes

1.  Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to flash 3 SD cards:
    *   **Device**: Raspberry Pi 5
    *   **Image**: Raspberry Pi OS Lite (64-bit)
    *   **Hostname**: something simple and not emotionally charged (cattle, not pets) e.g., `k8s-pve-port1`
    *   **Username and Password**: Make sure you set these since you'll be using them to SSH into the Pi and sudo respectively.
    *   **SSH**: Enable SSH (use key-based authentication, _don't_ use password authentication: it's not 1998 anymore).
3.  Boot the Pi, connect it to your network, and ensure you can SSH into it from your main computer (but don't do anything else).
4.  Ensure Ansible is installed on your main computer.
    *  I've used [mise](https://mise.jdx.dev/getting-started.html) to manage my ansible installation (see `mise.toml`)
    *  so you should be able to `mise install` and be good to go.
    *  You could use `brew`, you _should_ try `mise`.

#### Inventory Configuration
1.  Copy `hosts.example.ini` to `hosts.ini`.
2.  Open `hosts.ini` and add your new node under the `[pve_nodes]` group.
3.  Provide the `ansible_host` (the IP address the Pi currently received from DHCP)

    ```ini
    [pve_nodes]
    # left node
    k8sctl-pi5l ansible_host=10.10.80.50
    # right node
    k8sctl-pi5r ansible_host=10.10.80.51
    # external node
    k8sctl-pi5x ansible_host=10.10.80.52
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

#### Running the Playbook

To provision the node, you will run the `proxmox-pi-setup.yaml` playbook.

Ansible will rely on the user and password you already created during the Raspberry Pi Imager process (via cloud-init) to set up Proxmox Administrator access later. You just need to provide your `sudo` password to Ansible so it can run elevated commands.

**Read the [Playbook Notes](#playbook-notes) below first**, but the command to run is:

```bash
ansible-playbook proxmox-pi-setup.yaml -l k8sctl-pi5r -K -e tailscale_auth_key="tskey-auth-..."
```

The playbook will take about 25min all up, mostly during the ZFS and Proxmox installation phase. Once it completes, it will reboot the Pi.

1.  Wait a few minutes for the Pi to reboot.
2.  Navigate to `https://<node_ip>:8006` in your web browser.
3.  Log in using your username (`your_username`) and password, ensuring the Realm is set to **Linux PAM standard authentication**.

#### Playbook Notes

##### Anatomy of the command

Make sure you understand what the switches mean and adjust to your needs:
*   `-l k8sctl-pi5r`: Limits the execution to just the one node
    * prevents re-running against existing nodes
    * you **can omit this** if you're ready to roll out all your nodes at once.
*   `-K`: Prompts you for the BECOME password (aka the `sudo` password for the Pi user).
    * This one is essential.
*   `-e tailscale_auth_key="..."`: (Optional) Provides your Tailscale auth key. If omitted, the Tailscale installation step will be skipped.
    * Make sure you created a `Reusable` key


##### Proxmox Administrator Setup:

If the `Grant user Administrator rights in Proxmox` task fails during the playbook run (which can happen occasionally if the PVE services haven't fully started before the reboot), you can manually grant access once you SSH back into the node after it reboots:

```bash
sudo pveum acl modify / -user your_username@pam -role Administrator
```

##### Workaround Raspberry Pi OS first-boot initramfs corruption

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


### The software build - Kubernetes

> [!NOTE]
> Coming soon.
>
> Tl;DR:
> - Talos cluster and control planes provisioned with `proxmox-talos-vms`
> - Talos worker nodes directly imaged with a Talos metal image and provisioned into the cluster directly with `talosctl`
