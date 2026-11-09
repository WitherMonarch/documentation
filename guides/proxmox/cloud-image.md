# Guide to setting up cloud images with Cloud-Init on Proxmox

## Basic info

I'll be using the Debian 13 generic image for this guide. 
> The image can be found at https://cloud.debian.org/images/cloud/trixie/latest/.

From my basic testing, a genericcloud image was only 10 MiB of RAM lighter than a generic one so it's not worth using it and losing the extra drivers that generic comes with.

I won't be explaining every single one of my decisions for this guide, I'll assume you can do your own research.

## Prerequisites

- Snippets are enabled in the **Datacenter** configuration.
> If they aren't, go to **Datacenter**, **Storage**, select the local storage of you node(s), **Edit**, add **Snippets** to Content.

- The cloud image has been downloaded
```
wget -P /root https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2
```

### Step 1: Create the VM 

Create a VM template with the cloud image
```
qm create 101 --name debian13-cloud-img --ostype l26 --memory 1024 --sockets 1 --cores 1 --cpu x86-64-v2-AES --net0 virtio,bridge=vmbr0,firewall=1 --ostype l26 --scsihw virtio-scsi-single --scsi1 local-lvm:cloudinit --agent 1

qm set 101 --scsi0 local-lvm:0,import-from=/root/debian-13-generic-amd64.qcow2,iothread=1 --boot order=scsi0 --template 1
```

## Step 2: Setup Cloud-Init

There are settings you can change on the Proxmox Web UI for the VM
- user + password
- DNS domain and servers
- SSH public keys
- Upgrade packages at boot
- Network config

These settings are applied on each boot so changing them and then rebooting applies.

To setup more advanced provisioning, a **cloud-init script** is required. My current one lives [here](../../homelab/proxmox/cloud-image.md)

Create a basic cloud-init script
```
cat <<'EOF' > /var/lib/vz/snippets/debian13-cloud.yaml
#cloud-config
timezone: America/Toronto

keyboard:
  layout: ca
  variant: multix

packages:
  - qemu-guest-agent
  - console-setup

write_files:
  # Disable ssh over VSOCK (UI spam)
  - path: /etc/default/grub.d/99-no-ssh-vsock.cfg
    permissions: '0644'
    content: |
      # Disable ssh over VSOCK
      GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT systemd.ssh_auto=no"

power_state:
  mode: reboot
  message: Rebooting to apply changes
  delay: now
  timeout: 30
  condition: true
EOF
```
> This sets the timezone, keyboard layout, installs the guest-agent, disables ssh VSOCK and reboots when done. Cloud-Init can do much more but this is an example.

Assign the script to the template VM, every guest cloned from it will inherit it. After this, the VM is ready to be cloned and used.
```
qm set 100 --cicustom vendor=local:snippets/debian13-cloud.yaml
```
> Using **vendor** instead of **user** will not overwrite the Proxmox Cloud-Init configuration.

## Step 3: Clean up

Delete the old cloud image unused disk
```
rm /root/debian-13-generic-amd64.qcow2
```

## Useful info and limitations

- Never resize the disk before cloning the VM, it will make the clone slower. You can, however, clone the VM, then go to **VM**, **Hardware**, **Hard Disk (scsi0)**, **Edit**, **Resize**, and enter a number of GiB to add (no downsizing). This change will be applied on VM reboot or if these commands are entered in the VM.
```
sudo cloud-init single --name growpart
sudo cloud-init single --name resizefs
```

- In the Proxmox VM Cloud-Init tab, some changes can bring unexpected changes:
	- Changing the password but not the user will change that user's password on next boot
	- Changing the user but not the password will create a new user on the next boot
		- If an SSH key is already set and this is done, it will give the SSH key to that new user too

- There can be multiple SSH keys for a user

- Setting a VM's **cicustom** again after it has already ran will make the referenced script run again on the next reboot

- The snippet must exist on the PVE node when trying to start a VM, it doesn't matter if its the first launch of subsequent ones, make sure every node in the cluster has the scripts.