# Packing the Host

First, we must generate an Ubuntu template on our Proxmox instance that we can use with terraform.
This allows us to customize the image, and keep it up to date using CI later on. 
To achieve this, we will use [HashiCorp Packer](https://developer.hashicorp.com/packer/integrations/hashicorp/proxmox).
We will continue using the provisioning LXC we set up for instantiating GOAD, as this is already in the correct network segment and has all prerequisites installed. Also, this reuse allows us to be efficient with our limited compute resources.
The source code can be found [in the dedicated `ares-detection repository`](https://github.com/manfred6/ares-detection).

## Obtaining the Image

First, we must obtain the required ubuntu image to use for out template. To do this, we can ssh to our Proxmox host and exeute the following command:

```bash
wget https://releases.ubuntu.com/26.04.1/ubuntu-26.04.1-live-server-amd64.iso -O \
    /var/lib/vz/template/iso/ubuntu-26.04.1-live-server-amd64.iso
```

This is functionally equivalent to downloading it from or (uploading it using) the Web UI.
Now that we have the image, we can move to building our template with packer.

## Packer Configuration

Below is an extract from the `proxmox-iso` source for our ubuntu template, which defines the settings our template will have:

```hcl
source "proxmox-iso" "ubuntu" {
  proxmox_url              = var.proxmox_url
  username                 = var.proxmox_iac_username
  password                 = var.proxmox_iac_password
  node                     = var.proxmox_node
  insecure_skip_tls_verify = true

  vm_name       = "packer-ubuntu-resolute-ephemeral"
  template_name = "packer-ubuntu-resolute-base"

  cores  = 2
  memory = 2048

  disks {
    type         = "scsi"
    disk_size    = "20G"
    storage_pool = "local"
  }

  network_adapters {
    model  = "virtio"
    bridge = "vmbr2"
  }

  boot_wait = "5s"

  boot_command = [
    "c<wait2>",
    "linux /casper/vmlinuz ipv6.disable=1 --- autoinstall ds=nocloud<enter><wait5>",
    "initrd /casper/initrd<enter><wait5>",
    "boot<enter><wait30>"
  ]

  boot = "order=scsi0;ide2"

  boot_iso {
    type     = "ide"
    index    = "2"
    iso_file = "local:iso/ubuntu-26.04.1-live-server-amd64.iso"
    unmount  = true
  }

  additional_iso_files {
    type             = "ide"
    iso_storage_pool = "local"
    cd_label         = "cidata"
    index            = "3"
    unmount          = true

    cd_content = {
      "user-data" = templatefile(
        "${path.root}/http/user-data.yaml",
        {
          ssh_public_key = local.ssh_public_key
        }
      )
      "meta-data" = file("${path.root}/http/meta-data.yaml")
    }
  }

  ssh_username         = "packer"
  ssh_private_key_file = var.ssh_private_key_file
  ssh_timeout          = "20m"

  qemu_agent = true
}
```

This will generate a single ubuntu template (`packer-ubuntu-resolute-base`). Importantly, this will be provisioned using DHCP in the same subnet as our provisioning LXC.
All settings and OS-level operations are contained in (1) the `http/user-data.yaml` file (such as installed packages) and (2) the shell provisioner for this `proxmox-iso` resource:

```hcl
build {
  sources = [
    "source.proxmox-iso.ubuntu"
  ]
  
  provisioner "shell" {
    inline = [
      "sudo systemctl is-active qemu-guest-agent",
      "sudo apt-get clean",
      "sudo rm -rf /var/lib/apt/lists/*",
      "sudo rm -f /etc/ssh/ssh_host_*",
      "sudo cloud-init clean --logs --seed --machine-id"
    ]
  }
}
```

Here, it can be seen that we check that the `qemu-guest-agent` is installed and active (this gets installed using `http/user-data.yaml`, as well as some sysprep steps such as ensuring individual ssh keys get generated upon cloning, logs are cleared, machine uuid is reset, etc.


## Packer Build

First, however, we must generate some SSH keys, as packer does not provide a native plugin for this such as Terraform would. We can do this using [the following script](https://github.com/manfred6/ares-detection/blob/main/packer/scripts/secrets.sh):

```bash
#!/bin/bash

set -euo pipefail

mkdir -p .secrets
chmod 700 .secrets

if [[ ! -f .secrets/resolute-ed25519 ]]; then
    ssh-keygen \
        -t ed25519 \
        -N "" \
        -C "packer-resolute-temp" \
        -f .secrets/resolute-ed25519
fi
```

For automating the packer build process, I created a small bash script including all necessary commands:

```bash
#!/bin/bash

set -euo pipefail

packer init .
packer fmt .
packer validate .
packer build .
```

This can also be found in the repo mentioned above, in [`scripts/packer.sh`](https://github.com/manfred6/ares-detection/blob/main/packer/scripts/packer.sh).

If everything runs smoothly, we now have a new template in proxmox called `packer-ubuntu-resolute-base` that we can use with terraform in our next steps.

