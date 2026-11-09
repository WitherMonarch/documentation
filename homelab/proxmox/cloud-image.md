# Current Cloud-Init setup

## Features

Version 2 - Debian 13

- xtermjs default console (serial 0)
- CPU hotplug (numa)
- ipv6 disabled
- serial console colored and resize on the fly
- kernel logging to warning and not debug

## Script
```
#cloud-config
timezone: America/Toronto
locale: en_CA.UTF-8

keyboard:
  layout: ca
  variant: multix

packages:
  - qemu-guest-agent
  - console-setup
  - net-tools
  - nmap
  - htop

write_files:
  # CPU hotplug
  - path: /etc/udev/rules.d/80-hotplug-cpu.rules
    permissions: '0644'
    content: |
      # CPU hotplug
      SUBSYSTEM=="cpu", ACTION=="add", TEST=="online", ATTR{online}=="0", ATTR{online}="1"

  # Memory hotplug
  - path: /etc/default/grub.d/98-memory-hotplug.cfg
    permissions: '0644'
    content: |
      # Memory hotplug
      GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT movable_node memhp_default_state=online_movable"

  # Disable ssh over VSOCK (UI spam)
  - path: /etc/default/grub.d/99-no-ssh-vsock.cfg
    permissions: '0644'
    content: |
      # Disable ssh over VSOCK
      GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT systemd.ssh_auto=no"

  # Show kernel warnings, not debug output, on the console
  - path: /etc/sysctl.d/99-kernel-printk.conf
    permissions: '0644'
    content: |
      # Disable kernel warnings/debug on console, only warnings
      kernel.printk = 4 4 1 7

  # Colors on the serial console
  - path: /etc/systemd/system/serial-getty@ttyS0.service.d/override.conf
    permissions: '0644'
    content: |
      # Colors on serial console
      [Service]
      Environment="TERM=xterm-256color"

  # Serial console resize on login and on demand
  - path: /etc/profile.d/serial-resize.sh
    permissions: '0644'
    content: |
      # Sync the guest's tty size with the real terminal on serial consoles
      # (xterm.js in Proxmox, etc.). Serial lines can't report window size.
      # - runs once at login on /dev/ttyS*
      # - on demand: type `resize` or `rs`

      [ -n "$BASH_VERSION" ] || return 0
      case $- in *i*) ;; *) return 0 ;; esac

      resize() {
          local old rows cols
          old=$(stty -g) || return 1
          stty raw -echo min 0 time 5
          printf '\0337\033[r\033[999;999H\033[6n\0338' > /dev/tty
          IFS='[;R' read -r _ rows cols _ < /dev/tty
          stty "$old"
          [ -n "$rows" ] && [ -n "$cols" ] && stty cols "$cols" rows "$rows"
      }

      alias rs=resize

      case "$(tty 2>/dev/null)" in
          /dev/ttyS*)
              resize
              printf '\n\033[2mSerial console: if the screen looks broken at any time, run "rs" or "resize" to resize.\033[0m\n\n'
              ;;
      esac

runcmd:
  - [ sed, -i, 's/^GRUB_CMDLINE_LINUX_DEFAULT=""/GRUB_CMDLINE_LINUX_DEFAULT="ipv6.disable=1"/', /etc/default/grub ]
  - [ update-grub ]
  - [ udevadm, control, --reload ]

power_state:
  mode: reboot
  message: Rebooting to apply GRUB and sysctl changes
  delay: now
  timeout: 30
  condition: true
```