# Proxmox Claude backup: topology

How the [Proxmox VM backup integration](README.md) is laid out on the Claude Code VM, and how a request flows through it.

## Client filesystem

```text
~/
├── .config/proxmox/backup.env          API address, node, storage, token (600)
├── .local/bin/proxmox-backup.sh        Validation, API calls, task monitoring (700)
└── .claude/skills/backup/SKILL.md      /backup command and confirmation workflow
```

| File | Role |
|---|---|
| `backup.env` | Connection settings and token. Stays on the VM only. |
| `proxmox-backup.sh` | The only component that talks to the Proxmox API. Supports `inspect` and `start ... --confirmed`. |
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
    User->>Claude: /backup <vmid> "comment"
    Claude->>Script: inspect <vmid> <comment>
    Script->>PVE: GET qemu/<vmid>/config, storage/<storage>/status
    PVE-->>Script: VM name, free space
    Script-->>Claude: JSON summary
    Claude->>User: Details + "Do you want me to start this backup?"
    User->>Claude: Yes
    Claude->>Script: start <vmid> <comment> --confirmed
    Script->>PVE: POST nodes/<node>/vzdump (mode=snapshot)
    PVE-->>Script: UPID
    loop every 5 s, up to 12 h
        Script->>PVE: GET tasks/<upid>/status
    end
    Script-->>Claude: BACKUP SUCCESSFUL / FAILED
    Claude->>User: Result with UPID
```

## API endpoints used

| Method | Path | Purpose |
|---|---|---|
| GET | `/nodes/<node>/qemu/<vmid>/config` | Check the VM exists, read its name |
| GET | `/nodes/<node>/storage/<storage>/status` | Check the storage is active, read free space |
| POST | `/nodes/<node>/vzdump` | Start the snapshot backup with a notes template |
| GET | `/nodes/<node>/tasks/<upid>/status` | Monitor the task |
