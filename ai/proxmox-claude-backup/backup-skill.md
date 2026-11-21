# Claude Code backup skill

Skill for the [Proxmox VM backup integration](README.md). Install it as `~/.claude/skills/backup/SKILL.md` and replace `<pve-node>` with your node name. `disable-model-invocation: true` means the skill runs only when the user types `/backup`.

````markdown
---
name: backup
description: Create and monitor a Proxmox QEMU virtual machine backup. Use when the user explicitly invokes /backup.
argument-hint: "<vmid> [comment]"
disable-model-invocation: true
---

# Proxmox VM Backup

Create snapshot-mode backups using the local Proxmox backup script.

## Configuration

The integration is fixed to:

- Proxmox node: `<pve-node>`
- Backup storage: `local`
- Backup mode: `snapshot`
- Script: `~/.local/bin/proxmox-backup.sh`

Never construct arbitrary Proxmox API requests.

Never access the Proxmox API directly. Use only the local backup script.

## Procedure

1. Extract the VMID and optional comment from `$ARGUMENTS`.

2. If the VMID is missing, ask the user for it.

3. Validate the VMID contains digits only.

4. If a comment is supplied, preserve its exact intended text.

5. If no comment is supplied:
   - Suggest a concise comment based on the current conversation.
   - If there is no relevant context, suggest `Manual backup requested via Claude Code`.
   - Present the proposed comment to the user.

6. Run the inspection command:

   `~/.local/bin/proxmox-backup.sh inspect VMID COMMENT`

   Pass VMID and COMMENT as separate, safely quoted shell arguments. Never interpolate untrusted text into executable shell syntax.

7. Review the inspection result.

8. Present a confirmation summary containing:
   - VMID and VM name
   - Proxmox node
   - Backup storage
   - Backup mode
   - Available storage space
   - Exact backup comment

9. Ask explicitly:

   "Do you want me to start this backup?"

10. Wait for an affirmative response to that specific confirmation request.

11. The original `/backup` invocation, a suggested comment, or an ambiguous response is never confirmation.

12. If the user changes the VMID or comment, repeat inspection and obtain confirmation for the updated request.

13. Only after confirmation, run:

   `~/.local/bin/proxmox-backup.sh start VMID COMMENT --confirmed`

   Pass all arguments separately and safely quoted.

14. Wait for the script to finish.

15. Report the actual result, including:
   - VMID and VM name
   - Backup storage
   - Backup mode
   - Comment
   - UPID
   - Success or failure

16. Only report success if the script exits successfully and reports `BACKUP SUCCESSFUL`.

## Restrictions

- Never start a backup without explicit user confirmation.
- Never treat the original `/backup` invocation as confirmation.
- Never invent a VMID.
- Never change the configured Proxmox node or storage.
- Never modify VM configuration or power state.
- Never delete or restore backups.
- Never bypass script validation.
- Never expose the API token or credential file.
- Never retry a failed backup automatically.
- If an operation fails, report the error and stop.
- If monitoring times out, report that the task may still be running.
- Never start another backup automatically after a timeout.
- Never claim success based solely on the task being started.

The local Bash script is the authority for backup execution and task monitoring.
````
