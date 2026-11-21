# Proxmox VM backups from Claude Code


## Purpose

Changes made from Claude Code, such as package upgrades, configuration edits or scripts run inside a guest, are easier to undo when a fresh backup of the VM exists. Without this integration, the user has to leave the session, open the Proxmox web interface and start a backup by hand before every change.

This integration moves that step into the Claude Code workflow. The user types `/backup` with one or more VMIDs in the session, Claude Code checks the VMs and the storage, starts the backups, follows them to completion and reports the result. The work then continues with a known rollback point. Restoring is still done by the user in Proxmox.

## How it works

- A Claude Code skill provides the `/backup <vmid> [<vmid>...] ["comment"] [--confirmed]` command.
- The skill runs a local Bash script, the only component that talks to the Proxmox VE API.
- The script uses a dedicated, restricted API token and can only read VM and storage details, start zstd-compressed snapshot backups and follow their tasks.
- Claude Code shows a summary and asks for confirmation before any backup starts, unless the user added `--confirmed` to the command.

Related pages:

- [Topology and execution flow](topology.md)
- [Backup script](proxmox-backup-script.md)
- [Claude Code skill](backup-skill.md)
- [Example client configuration](backup.env.example)

## Design

- No SSH. The integration uses only the Proxmox VE HTTPS API, through a dedicated API token.
- Least privilege: a non-admin user with a custom role and a privilege-separated token.
- The node, storage, mode (`snapshot`) and compression (`zstd`) are fixed in the configuration and the script. Claude Code can't choose them.
- Claude Code never calls the API directly. The skill may only run the local script.
- Every VM and the storage are checked before any backup starts. VMs are then backed up one at a time, one Proxmox task each, and the batch stops at the first VM that isn't `SUCCESS`.
- Every backup needs an explicit confirmation: a reply after the inspection summary, or `--confirmed` typed by the user on the `/backup` command.
- Backups are never retried automatically. A result that can't be verified is reported as `UNKNOWN`, never as a failure or a success.
- One backup run at a time from the Claude Code VM (`flock` lock file).

| Component | Configuration |
|---|---|
| Hypervisor | Proxmox VE 9 |
| Backup mode | `snapshot` |
| Compression | `zstd` |
| Backup storage | `<backup-storage>`, for example `local` |
| API authentication | API token, privilege separation enabled |
| Client | Dedicated Claude Code VM |
| Implementation | Bash, curl, jq, flock |

## Proxmox permissions

Custom role `ClaudeBackup` with:

- `VM.Backup`
- `VM.Audit`: reads the VM configuration during inspection
- `Datastore.AllocateSpace`
- `Datastore.Audit`: reads storage status during inspection

The account can't change VM power state or configuration, and can't delete or restore backups. It can read the status and log of the tasks its token started, which the `status` operation uses.

## Proxmox setup

Run these as root on a Proxmox node.

```bash
pveum role add ClaudeBackup --privs "VM.Backup VM.Audit Datastore.AllocateSpace Datastore.Audit"
pveum user add <api-user>@pve --comment "Claude Code backup integration"
pveum user token add <api-user>@pve <token-name> --privsep 1
```

The token secret is shown only once, when the token is created. Store it straight in the client configuration file (see below).

With privilege separation, the token's effective permissions are the intersection of the user's ACLs and the token's ACLs, so give both the role. `/vms` covers every VM. To limit backups to specific VMs, use `/vms/<vmid>` instead.

```bash
pveum acl modify /vms --roles ClaudeBackup --users <api-user>@pve
pveum acl modify /vms --roles ClaudeBackup --tokens '<api-user>@pve!<token-name>'
pveum acl modify /storage/<backup-storage> --roles ClaudeBackup --users <api-user>@pve
pveum acl modify /storage/<backup-storage> --roles ClaudeBackup --tokens '<api-user>@pve!<token-name>'
```

## Client setup (Claude Code VM)

Dependencies: `bash`, `curl`, `jq`, `ca-certificates`, plus `flock`, `sed`, `dirname`, `stat`, `id` and `mkdir`, which a standard Debian install already provides (util-linux, sed, coreutils).

```bash
sudo apt install curl jq ca-certificates
mkdir -p ~/.config/proxmox ~/.local/bin ~/.claude/skills/backup
chmod 700 ~/.config/proxmox
```

Copy [`backup.env.example`](backup.env.example) to `~/.config/proxmox/backup.env`, set its mode to `600`, and replace the placeholders. The script refuses to run if the file isn't owned by the current user with mode `600` or stricter, or if its directory is writable by group or others. Read the comments on `PVE_VERIFY_TLS` and `PVE_CA_FILE` before you choose a value.

Install the [backup script](proxmox-backup-script.md) as `~/.local/bin/proxmox-backup.sh` (mode `700`), and the [skill](backup-skill.md) as `~/.claude/skills/backup/SKILL.md`. A running Claude Code session picks up the new skill only after a restart.

Check it with a read-only inspection:

```bash
~/.local/bin/proxmox-backup.sh inspect --comment "Connectivity test" <vmid>
```

## Usage

```text
/backup <vmid> "Before upgrading Docker"
/backup <vmid1> <vmid2> <vmid3> "Before upgrading Docker"
/backup <vmid>
/backup <vmid1> <vmid2> "Before upgrading Docker" --confirmed
```

- If you leave out the comment, Claude Code suggests one.
- Claude Code shows each VM's name, the node, storage, mode, compression, storage space (available and total in MB, percentage used), the TLS verification mode and the exact comment. It then asks "Do you want me to start this backup?" and starts only after a clear yes.
- With `--confirmed` as the last argument, Claude Code still runs the inspection and stops if it fails, then shows a 🛑 `AUTOMATIC RUN WITHOUT CONFIRMATION — USE AT YOUR OWN RISK` warning and starts without asking. Without a comment, it generates one and says so.

### Outcomes

| Outcome | Meaning | Script exit code |
|---|---|---|
| `SUCCESS` | Proxmox reported the task stopped with exit status `OK` | 0 |
| `FAILED` | Rejected by Proxmox (HTTP 4xx or 500), not sent (connection or TLS failure before submission), or the task stopped with any exit status other than `OK`, including warnings | 1 |
| `UNKNOWN` | The request may have been accepted, but the result couldn't be verified: submission timeout, lost response, HTTP 5xx other than 500, no task ID returned, task status unreachable, or monitoring timeout | 3 |
| `NOT STARTED` | Not attempted because an earlier VM in the batch wasn't `SUCCESS` | — |

Other exit codes: 2 for a usage error, 4 when `status` finds the task still running, and 5 when the API is unreachable (a network problem) before any backup was submitted. TLS, authentication (HTTP 401) and permission (HTTP 403) errors are reported separately from network errors.

Before retrying an `UNKNOWN` backup, check the task:

```bash
~/.local/bin/proxmox-backup.sh status --upid '<upid>' --log
```

`status` reports the task status, exit status and outcome, and with `--log` up to 1000 task log lines. It accepts only `vzdump` UPIDs on the configured node. When no UPID was received, check the node's task list in the Proxmox web interface.

### Comments and template variables

The comment is sent as the vzdump `notes-template`. Proxmox replaces `{{cluster}}`, `{{guestname}}`, `{{node}}` and `{{vmid}}` with their values. The template must be a single line with backslashes escaped; the script doubles each backslash so it's stored literally, and rejects newlines and other control characters.

### Storage space

Inspection reports available space and total size in MB (1 MB = 1,000,000 bytes) and the percentage used, and warns when free space is below `PVE_LOW_SPACE_PERCENT` (default 20). Free space doesn't guarantee a backup will fit: the archive size depends on disk usage and compression.

## Security considerations

- **TLS verification:** on by default. A node that still uses the default self-signed Proxmox certificate fails verification unless the client trusts the node's CA. Set `PVE_CA_FILE` to a copy of the node's `/etc/pve/pve-root-ca.pem` to verify it; the certificate must also be valid for the name or address in `PVE_HOST`. If you set `PVE_VERIFY_TLS="false"` instead, curl runs with `--insecure` and doesn't check the server's identity. Anyone who can intercept traffic between the VM and the node could impersonate the API and capture the token. Every run then prints a warning and reports `TLS verification: DISABLED`. Keep the two on a trusted network segment.
- The token is passed to curl through a process-substitution config file, not as a command-line argument, so it doesn't show up in the process list.
- The script sources `backup.env` as shell code, so it enforces ownership and permissions on the file and its directory, and rejects a symlinked configuration file.
- The script rejects newlines in configuration values, invalid VMIDs (digits only, 100 to 999999999), more than 20 VMIDs per run, and comments with control characters or longer than 1024 bytes.
- Never commit the real `backup.env`. Keep only a template with placeholders under version control.

## Known limitations

- The lock only prevents overlapping runs from the same Claude Code VM.
- Backup retention and storage capacity are managed outside this integration.
- Task monitoring stops after 12 hours per VM and reports `UNKNOWN`. The backup task may still be running after that.
- Each backup triggers the node's notification settings. With the default matcher, that's one email to root per VM.

## Sources

- [Proxmox VE User Management](https://pve.proxmox.com/wiki/User_Management)
- [pveum(1) manual](https://pve.proxmox.com/pve-docs/pveum.1.html)
- [Proxmox VE Backup and Restore](https://pve.proxmox.com/wiki/Backup_and_Restore)
- [vzdump(1) manual](https://pve.proxmox.com/pve-docs/vzdump.1.html) (notes-template variables and escaping)
- [Proxmox VE API viewer](https://pve.proxmox.com/pve-docs/api-viewer/)
- [curl exit codes](https://curl.se/libcurl/c/libcurl-errors.html)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
