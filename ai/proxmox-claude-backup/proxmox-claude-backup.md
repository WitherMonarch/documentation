# Proxmox VM backups from Claude Code

Tested with: Debian 13 (trixie) Claude Code VM, GNU bash 5.2.37, curl 8.14.1, jq 1.7, against a Proxmox VE 9 node (exact minor version not recorded).

A restricted Proxmox VE API integration that lets Claude Code, running in its own VM, start and monitor snapshot backups of individual QEMU VMs. The user requests a backup with `/backup <vmid> [comment]`. Claude Code inspects the VM and storage, then asks for explicit confirmation before it starts anything.

Related pages:

- [Topology and execution flow](topology.md)
- [Backup script](proxmox-backup-script.md)
- [Claude Code skill](backup-skill.md)

## Testing status

- **Tested:** `inspect` and `start` on the Claude Code VM against one QEMU VM, with snapshot mode, the `local` storage, the backup comment (notes) and UPID task monitoring through to `BACKUP SUCCESSFUL`.
- **Unverified:** the Proxmox-side setup commands below (user, role, token, ACLs). They were checked against the Proxmox documentation but not run as written here. Check them with `pveum help` on your PVE version.
- **Unverified:** the ACL paths shown. Scope them to your own environment.

## Design

- No SSH. The integration uses only the Proxmox VE HTTPS API, through a dedicated API token.
- Least privilege: a non-admin user with a custom role and a privilege-separated token.
- The node, storage and mode are fixed in the configuration and the script. Claude Code can't choose them.
- Claude Code never calls the API directly. The skill may only run the local script.
- Every backup needs an explicit confirmation from the user after the inspection result is shown.
- Failed or timed-out backups are reported and never retried automatically.

| Component | Configuration |
|---|---|
| Hypervisor | Proxmox VE 9 |
| Backup mode | `snapshot` |
| Backup storage | `<backup-storage>` (tested with `local`) |
| API authentication | API token, privilege separation enabled |
| Client | Dedicated Claude Code VM |
| Implementation | Bash, curl, jq |

## Proxmox permissions

Custom role `ClaudeBackup` with:

- `VM.Backup`
- `VM.Audit`: reads the VM configuration during inspection
- `Datastore.AllocateSpace`
- `Datastore.Audit`: reads storage status during inspection

The account can't change VM power state or configuration, and can't delete or restore backups. In the tested setup, `Sys.Audit` wasn't needed to monitor the status of the token's own backup task.

## Proxmox setup

Unverified: syntax checked against the `pveum` man page for PVE 9.2, but not run as written. Run these as root on a Proxmox node.

```bash
pveum role add ClaudeBackup --privs "VM.Backup VM.Audit Datastore.AllocateSpace Datastore.Audit"
pveum user add <api-user>@pve --comment "Claude Code backup integration"
pveum user token add <api-user>@pve <token-name> --privsep 1
```

The token secret is shown only once, when the token is created. Store it straight in the client configuration file (see below).

With privilege separation, the token's effective permissions are the intersection of the user's ACLs and the token's ACLs, so give both the role. Example scoping (adjust the paths; `/vms` covers every VM):

```bash
pveum acl modify /vms --roles ClaudeBackup --users <api-user>@pve
pveum acl modify /vms --roles ClaudeBackup --tokens '<api-user>@pve!<token-name>'
pveum acl modify /storage/<backup-storage> --roles ClaudeBackup --users <api-user>@pve
pveum acl modify /storage/<backup-storage> --roles ClaudeBackup --tokens '<api-user>@pve!<token-name>'
```

## Client setup (Claude Code VM)

Dependencies: `bash`, `curl`, `jq`, `ca-certificates`.

```bash
sudo apt install curl jq ca-certificates
mkdir -p ~/.config/proxmox ~/.local/bin ~/.claude/skills/backup
chmod 700 ~/.config/proxmox
```

Create `~/.config/proxmox/backup.env` with mode `600`:

```bash
PVE_HOST="https://<pve-address>:8006"
PVE_NODE="<pve-node>"
PVE_STORAGE="<backup-storage>"
PVE_TOKEN_ID="<api-user>@pve!<token-name>"
PVE_TOKEN_SECRET="<token-secret>"
PVE_VERIFY_TLS="true"
```

Install the [backup script](proxmox-backup-script.md) as `~/.local/bin/proxmox-backup.sh` (mode `700`), and the [skill](backup-skill.md) as `~/.claude/skills/backup/SKILL.md`.

Check it with a read-only inspection:

```bash
~/.local/bin/proxmox-backup.sh inspect <vmid> "Connectivity test"
```

## Usage

```text
/backup <vmid> "Before upgrading Docker"
/backup <vmid>
```

If you leave out the comment, Claude Code suggests one. Claude Code then shows the VM name, node, storage, mode, free space and the exact comment, asks "Do you want me to start this backup?", and starts only after a clear yes.

## Security considerations

- **TLS verification:** keep `PVE_VERIFY_TLS="true"`. Setting it to `false` makes curl use `--insecure`. Then anyone who can intercept traffic between the VM and the node can impersonate the API and capture the token. Install a trusted certificate on the node instead, or add the node's CA to the client's trust store. In the tested setup, verification was temporarily turned off. That's a known gap to fix.
- The token is passed to curl through a process-substitution config file, not as a command-line argument, so it doesn't show up in the process list.
- The script rejects symlinked configuration files, newlines in values, non-numeric VMIDs, and comments that span more than one line or are longer than 1024 characters.
- Never commit the real `backup.env`. Keep only a template with placeholders under version control.
- The script sources `backup.env` as shell code, so it must be owned by, and writable only by, the Claude Code user.

## Known limitations

- There's no protection against two backups being requested at the same time.
- Backup retention and storage capacity are managed outside this integration.
- Task monitoring stops after 12 hours. The backup task may still be running after that.

## Sources

- [Proxmox VE User Management](https://pve.proxmox.com/wiki/User_Management)
- [pveum(1) manual](https://pve.proxmox.com/pve-docs/pveum.1.html)
- [Proxmox VE Backup and Restore](https://pve.proxmox.com/wiki/Backup_and_Restore)
- [Proxmox VE API viewer](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
