# Proxmox Virtual Environment

So we begin the journey of setting up a Purple Team Lab. This will be based on Proxmox Virtual Environment (PVE), which will itself be hosted on Hetzner Robot.
You can of course use any other cloud provider or even your own hardware, but I dont have my own hardware, already use Hetzner, and the auctions are actually well priced if you get a good deal.


Personally, I had a good deal for the following spec:

| Type | Value | 
|:----:|:-----:|
| CPU  | Intel Xeon E3-1275v5 |
| RAM  | 4x RAM 16384 MB DDR4 ECC |
| DISK | 2x SSD SATA 480 GB Datacenter |

I use software RAID 1, so I only get 480GB storage, which might be tight since I might be collecting a lot of data, but I can make do.
In any case, its worth it for the 64GB ECC memory and the Xeon CPU. The GOAD machines will mostly be idle so the small number of cores doesnt really concern me, and the clock speed is decently high so itll be fast for things I want to get done.

## Setting up PVE

Following [this guide from hetzner](https://community.hetzner.com/tutorials/install-and-configure-proxmox_ve?title=Proxmox_VE/en), one must first boot into the rescue system, then run:
```bash
installimage
```

This will drop you in a TUI from where we can select `Debian 13 (Trixie)`. I only configured the hostname (to `ares`, the god of war, as thats what this system is for), and left the other settings at default. The installer will now install the base system.
Once install completes, we reboot the server and log in using our SSH key.

We now move to setting up `Proxmox`. To do this, we can follow the [official guide from Proxmox](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_13_Trixie). 

Importantly, we need to ensure the hostname resolves. We can do this by running
```bash
hostname --ip-address
```

If this prints the publicly assigned IP address, youre good. If not, adjust that entry in `/etc/hosts`, as follows:
```text
(...snip...)
<YOUR_PUBLIC_IP> <YOUR_HOSTNAME>
(...snip...)
```

### Proxmox Repository

We can now move to the actual installation. First, we must add the required APT repositories:
```bash
cat > /etc/apt/sources.list.d/pve-install-repo.sources << EOL
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOL
```

Yoink the key:
```bash
wget https://enterprise.proxmox.com/debian/proxmox-archive-keyring-trixie.gpg -O /usr/share/keyrings/proxmox-archive-keyring.gpg
```

Add all this to APT:
```bash
apt update && apt -y full-upgrade
```

### Proxmox Kernel

Now we install the kernel:
```bash
apt -y install proxmox-default-kernel
```

and reboot:

```bash
reboot now
```

After rebooting, we install the required Proxmox packages, remove the old debian kernel, update grub, and remove os-prober:
```bash
apt -y install proxmox-ve postfix open-iscsi chrony && \
    apt -y remove linux-image-amd64 'linux-image-6.12*' && \
    update-grub && \
    apt -y remove os-prober
```

### Proxmox Web UI

As Proxmox has now been installed, we can access the web UI at `https://<YOUR_IP>:8006/`. 
First, however, I lock down the proxmox API to be accessible only from localhost and `192.168.1.3`, which will be our IAC Provisioning LXC container:
```bash
cat << EOF > /etc/default/pveproxy
LISTEN_IP="0.0.0.0"
ALLOW_FROM="127.0.0.1,192.168.1.3"
DENY_FROM="all"
POLICY="allow"
EOF

systemctl restart pveproxy.service
```

### Disable Enterprise Repo

We now disable the enterprise repo, as we dont wanna pay for it and dont need it:

```bash
echo "Enabled: false" >> /etc/apt/sources.list.d/pve-enterprise.sources
```

We can add the no-subsciption repo:

```bash
cat << EOF > /etc/apt/sources/list.d/pve-no-subscription.sources
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

Now, that we have the correct repo installed, we upgrade:
```bash
apt update && apt -y full-upgrade && apt -y autoremove
```

### SSH Hardening

Since SSH will be one of the only ports exposed, its important to harden it accordingly.
This can include installing Fail2Ban as well as restricting Password login and cipher suites.
This is beyond the scope of this blog, thus I will point to [this very good resource here](https://www.digitalocean.com/community/tutorials/how-to-harden-openssh-on-ubuntu-20-04).



