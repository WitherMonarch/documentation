# Claude Code backup skill

Skill for the [Proxmox VM backup integration](README.md). Install it as `~/.claude/skills/backup/SKILL.md` and replace `<pve-node>` with your node name. `disable-model-invocation: true` means the skill runs only when the user types `/backup`.

The skill accepts several VMIDs and an optional `--confirmed` final argument. With `--confirmed`, it still inspects first, shows a 🛑 warning and starts without asking.

````markdown
---
name: backup
description: Create and monitor Proxmox QEMU virtual machine backups. Use when the user explicitly invokes /backup.
argument-hint: "<vmid> [<vmid>...] [\"comment\"] [--confirmed]"
disable-model-invocation: true
---

# Proxmox VM Backup

Create snapshot-mode backups of one or more VMs using the local Proxmox backup script.

## Configuration

The integration is fixed to:

- Proxmox node: `<pve-node>`
- Backup storage: `local`
- Backup mode: `snapshot`
- Compression: `zstd`
- Script: `~/.local/bin/proxmox-backup.sh`

Never construct arbitrary Proxmox API requests.

Never access the Proxmox API directly. Use only the local backup script, with
its `inspect`, `start` and `status` operations.

## Arguments

Arguments: `$ARGUMENTS`

Form: `<vmid> [<vmid>...] ["comment"] [--confirmed]`

- VMIDs: one or more unquoted numbers, before the comment.
- Comment: optional, one quoted string. The same comment applies to every VM.
  Proxmox expands template variables in it: `{{guestname}}`, `{{vmid}}`,
  `{{node}}` and `{{cluster}}`.
- `--confirmed`: optional. Counts only as a bare, unquoted final token. Text
  `--confirmed` inside the quoted comment is part of the comment.

## Script outcomes

| Outcome | Meaning | Exit code |
|---|---|---|
| `SUCCESS` | Proxmox reported the task stopped with exit status `OK` | 0 |
| `FAILED` | Rejected by Proxmox, not sent, or task stopped with a non-OK exit status | 1 |
| `UNKNOWN` | The request may have been accepted; the result is unverified | 3 |
| `NOT STARTED` | Not attempted because an earlier VM in the batch was not `SUCCESS` | — |

Exit code 2 is a usage error. For `status`, exit code 4 means the task is still running.
Exit code 5 means the Proxmox API was unreachable (network problem) before any
backup was submitted; report it as unreachable, not as a permission problem.

## Procedure

1. Parse the VMIDs, the comment and whether `--confirmed` is present.

2. If no VMID is given, ask the user for one or more VMIDs.

3. Validate every VMID contains digits only. Remove duplicates, keeping the first.

4. If a comment is supplied, preserve its exact intended text.

5. If no comment is supplied:
   - Suggest a concise comment based on the current conversation.
   - If there is no relevant context, suggest `Manual backup requested via Claude Code`.
   - Present the proposed comment to the user.
   - If `--confirmed` was given, use the suggested comment without asking and
     state in the summary that the comment was generated.

6. Inspect all requested VMs in one command:

   `~/.local/bin/proxmox-backup.sh inspect --comment COMMENT VMID [VMID...]`

   Pass COMMENT and each VMID as separate, safely quoted shell arguments. Never
   interpolate untrusted text into executable shell syntax.

7. If inspection fails, report the error and stop. `--confirmed` does not
   override a failed inspection. Distinguish the cause from the script's message:
   `UNREACHABLE` (network, exit 5), TLS error, authentication failure (HTTP
   401), permission denied (HTTP 403) or another Proxmox error.

8. Present a summary containing:
   - Each VMID and VM name
   - Proxmox node
   - Backup storage
   - Backup mode and compression
   - Storage space: available MB, total MB and percentage used, plus a
     warning if `low_space_warning` is true (free space below the threshold,
     20% by default). State that free space does not
     guarantee the backup will fit.
   - TLS verification mode, with a warning if it is `DISABLED`
   - Exact backup comment

9. If `--confirmed` is present:
   - Show this warning directly above the summary:

     `## 🛑 AUTOMATIC RUN WITHOUT CONFIRMATION — USE AT YOUR OWN RISK`

   - Go straight to step 13 without asking.

10. Otherwise, ask explicitly:

    "Do you want me to start this backup?"

11. Wait for an affirmative response to that specific confirmation request. An
    invocation without `--confirmed`, a suggested comment, or an ambiguous
    response is never confirmation.

12. If the user changes the VMIDs or comment, repeat inspection and obtain
    confirmation for the updated request.

13. Run, with exactly the VMIDs and comment that were inspected and approved:

    `~/.local/bin/proxmox-backup.sh start --comment COMMENT --confirmed VMID [VMID...]`

    Pass all arguments separately and safely quoted. The script backs up the VMs
    one at a time and stops at the first VM that is not `SUCCESS`.

14. Wait for the script to finish.

15. Report the actual result from the script's final report:
    - Overall result: `BACKUP SUCCESS`, `BACKUP FAILED` or `BACKUP UNKNOWN`
    - Backup storage, mode and comment
    - For each VM: VMID, name, result, UPID when one was received, and the
      detail for `FAILED` or `UNKNOWN`

16. Report success only if the script exits with code 0 and reports
    `BACKUP SUCCESS`.

17. If any VM is `UNKNOWN`:
    - Say clearly that the backup may have been accepted and its result is
      unverified. Do not describe it as failed.
    - If a UPID was received, offer to check it with the `status` operation.
    - If no UPID was received, tell the user to check the node's task list in
      Proxmox before retrying.

## Checking a task

Run only when the user asks, or after offering it for an `UNKNOWN` result:

`~/.local/bin/proxmox-backup.sh status --upid UPID [--log]`

Pass the UPID as a single quoted argument. Report `outcome`, `task_status`,
`exit_status` and, with `--log`, the relevant log lines. Checking a task is
read-only and never starts a backup.

## Restrictions

- Never start a backup without either a `--confirmed` invocation from the user
  or an explicit confirmation reply.
- Never add `--confirmed` to the user's request yourself.
- Never back up VMIDs other than those inspected and approved.
- Never invent a VMID.
- Never change the configured Proxmox node or storage.
- Never modify VM configuration or power state.
- Never delete or restore backups.
- Never bypass script validation, including the configuration permission and
  TLS checks.
- Never expose the API token or credential file.
- Never retry a `FAILED` or `UNKNOWN` backup automatically.
- Never start VMs the script reported as `NOT STARTED` automatically.
- Never treat `UNKNOWN` as a definite failure or as success.
- If an operation fails, report the error and stop.
- Never claim success based solely on the task being started.

The local Bash script is the authority for backup execution and task monitoring.
````
