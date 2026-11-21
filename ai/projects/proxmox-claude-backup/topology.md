# Proxmox Claude backup: topology

How the [Proxmox VM backup integration](README.md) is laid out on the Claude Code VM, and how a request flows through it.

## Client filesystem

```text
~/
├── .config/proxmox/backup.env          API address, node, storage, token (600, directory 700)
├── .local/bin/proxmox-backup.sh        Validation, API calls, task monitoring (700)
└── .claude/skills/backup/SKILL.md      /backup command and confirmation workflow
```

The script also uses a lock file, `$XDG_RUNTIME_DIR/proxmox-backup.lock` (or `~/.cache/proxmox-backup.lock`), so only one backup run is active at a time.

| File | Role |
|---|---|
| `backup.env` | Connection settings and token. Stays on the VM only. |
| `proxmox-backup.sh` | The only component that talks to the Proxmox API. Supports `inspect`, `start ... --confirmed` and `status`. |
| `SKILL.md` | Tells Claude Code how and when to run the script. Never calls the API itself. |

## Network

```mermaid
flowchart LR
    U[User] --> CC[Claude Code VM]
    CC -- "HTTPS :8006<br/>API token" --> PVE[Proxmox VE node]
    PVE --> ST[(Backup storage)]
```

## Execution flow

```mermaid
sequenceDiagram
    actor User
    participant Claude as Claude Code + skill
    participant Script as proxmox-backup.sh
    participant PVE as Proxmox VE API
    User->>Claude: /backup <vmid>... "comment" [--confirmed]
    Claude->>Script: inspect --comment <comment> <vmid>...
    Script->>PVE: GET qemu/<vmid>/config (each VM), storage/<storage>/status
    PVE-->>Script: VM names, storage size and free space
    Script-->>Claude: JSON summary
    alt without --confirmed
        Claude->>User: Summary + "Do you want me to start this backup?"
        User->>Claude: Yes
    else with --confirmed
        Claude->>User: Summary + 🛑 warning
    end
    Claude->>Script: start --comment <comment> --confirmed <vmid>...
    loop each VM, stop at first non-SUCCESS
        Script->>PVE: POST nodes/<node>/vzdump (mode=snapshot, compress=zstd)
        PVE-->>Script: UPID
        loop every 5 s, up to 12 h
            Script->>PVE: GET tasks/<upid>/status
        end
    end
    Script-->>Claude: BACKUP SUCCESS / FAILED / UNKNOWN, per-VM results and UPIDs
    Claude->>User: Result
```

## API endpoints used

| Method | Path | Purpose |
|---|---|---|
| GET | `/nodes/<node>/qemu/<vmid>/config` | Check the VM exists, read its name |
| GET | `/nodes/<node>/storage/<storage>/status` | Check the storage is active, read size and free space |
| POST | `/nodes/<node>/vzdump` | Start a zstd-compressed snapshot backup with a notes template |
| GET | `/nodes/<node>/tasks/<upid>/status` | Monitor the task, and `status` checks |
| GET | `/nodes/<node>/tasks/<upid>/log` | Task log for `status --log` |
