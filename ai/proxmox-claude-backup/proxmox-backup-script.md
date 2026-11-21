# Proxmox backup script

Script for the [Proxmox VM backup integration](proxmox-claude-backup.md). Install it as `~/.local/bin/proxmox-backup.sh` with mode `700`. It reads its settings from `~/.config/proxmox/backup.env` and contains no site-specific values.

Tested: `inspect` and `start` on Debian 13 with bash 5.2.37, curl 8.14.1 and jq 1.7.

```bash
#!/usr/bin/env bash
#
# Proxmox VE QEMU VM backup helper
#
# Operations:
#   inspect <VMID> <comment>
#   start <VMID> <comment> --confirmed
#
# Configuration:
#   ~/.config/proxmox/backup.env
#
# Dependencies:
#   bash, curl, jq
#
# Security:
#   - Fixed Proxmox host, node and storage.
#   - API token never passed as a command-line argument.
#   - TLS verification configurable.
#   - No VM power/configuration operations.
#   - No backup deletion or restoration.

set -Eeuo pipefail
umask 077

readonly ENV_FILE="${HOME}/.config/proxmox/backup.env"
readonly SCRIPT_NAME="proxmox-backup.sh"
readonly TASK_TIMEOUT=43200
readonly POLL_INTERVAL=5

BASE_URL=""
API_BASE=""
PVE_NODE=""
PVE_STORAGE=""
PVE_TOKEN_ID=""
PVE_TOKEN_SECRET=""
PVE_VERIFY_TLS="true"

die() {
    printf 'ERROR: %s\n' "$*" >&2
    exit 1
}

log() {
    printf '%s\n' "$*" >&2
}

usage() {
    cat <<EOF
Usage:

  ${SCRIPT_NAME} inspect <VMID> <comment>

  ${SCRIPT_NAME} start <VMID> <comment> --confirmed

Examples:

  ${SCRIPT_NAME} inspect 105 "Before upgrading Docker"

  ${SCRIPT_NAME} start 105 "Before upgrading Docker" --confirmed

EOF
    exit 2
}

# ---------------------------------------------------------------------
# Dependency checks
# ---------------------------------------------------------------------

for cmd in curl jq; do
    command -v "$cmd" >/dev/null 2>&1 ||
        die "Required command not found: $cmd"
done

# ---------------------------------------------------------------------
# Load configuration
# ---------------------------------------------------------------------

[[ -f "$ENV_FILE" && ! -L "$ENV_FILE" ]] ||
    die "Missing configuration file or symbolic link: $ENV_FILE"

[[ -r "$ENV_FILE" ]] ||
    die "Configuration file is not readable."

# Configuration is a trusted, user-owned shell file.
# shellcheck disable=SC1090
source "$ENV_FILE"

: "${PVE_HOST:?Missing PVE_HOST}"
: "${PVE_NODE:?Missing PVE_NODE}"
: "${PVE_STORAGE:?Missing PVE_STORAGE}"
: "${PVE_TOKEN_ID:?Missing PVE_TOKEN_ID}"
: "${PVE_TOKEN_SECRET:?Missing PVE_TOKEN_SECRET}"

PVE_VERIFY_TLS="${PVE_VERIFY_TLS:-true}"

[[ "$PVE_HOST" == https://* ]] ||
    die "HTTPS is required."

[[ "$PVE_VERIFY_TLS" == "true" || "$PVE_VERIFY_TLS" == "false" ]] ||
    die "PVE_VERIFY_TLS must be true or false."

# Reject unexpected characters in configuration values.
# The generated Proxmox token secret is a UUID.
for value in \
    "$PVE_HOST" \
    "$PVE_NODE" \
    "$PVE_STORAGE" \
    "$PVE_TOKEN_ID" \
    "$PVE_TOKEN_SECRET"
do
    [[ "$value" != *$'\n'* && "$value" != *$'\r'* ]] ||
        die "Invalid newline in configuration."
done

[[ "$PVE_TOKEN_SECRET" =~ ^[a-zA-Z0-9_-]+$ ]] ||
    die "Unexpected characters in API token secret."

[[ "$PVE_TOKEN_ID" =~ ^[a-zA-Z0-9@._-]+![a-zA-Z0-9_-]+$ ]] ||
    die "Invalid API token ID."

BASE_URL="${PVE_HOST%/}"
API_BASE="${BASE_URL}/api2/json"

if [[ "$PVE_VERIFY_TLS" == "false" ]]; then
    log "WARNING: TLS certificate verification is disabled."
fi

# ---------------------------------------------------------------------
# API helpers
# ---------------------------------------------------------------------

api_curl() {
    local tls_options=()

    if [[ "$PVE_VERIFY_TLS" == "false" ]]; then
        tls_options+=(--insecure)
    fi

    curl \
        --silent \
        --show-error \
        --fail-with-body \
        --connect-timeout 10 \
        --max-time 60 \
        "${tls_options[@]}" \
        --config <(
            printf 'header = "Authorization: PVEAPIToken=%s=%s"\n' \
                "$PVE_TOKEN_ID" "$PVE_TOKEN_SECRET"
        ) \
        "$@"
}

api_get() {
    api_curl "$1"
}

# ---------------------------------------------------------------------
# Input validation
# ---------------------------------------------------------------------

validate_input() {
    local vmid="$1"
    local comment="$2"

    [[ "$vmid" =~ ^[0-9]+$ ]] ||
        die "VMID must contain digits only."

    [[ -n "$comment" ]] ||
        die "A backup comment is required."

    [[ "$comment" != *$'\n'* && "$comment" != *$'\r'* ]] ||
        die "Backup comment must be a single line."

    (( ${#comment} <= 1024 )) ||
        die "Backup comment exceeds 1024 characters."
}

# ---------------------------------------------------------------------
# Preflight checks
# ---------------------------------------------------------------------

preflight() {
    local vmid="$1"
    local comment="$2"

    local vm_json
    local storage_json
    local vm_name
    local available

    validate_input "$vmid" "$comment"

    log "Checking VM ${vmid}..."

    vm_json="$(
        api_get \
            "${API_BASE}/nodes/${PVE_NODE}/qemu/${vmid}/config"
    )" || die "Cannot read VM configuration. Check VM.Audit permissions."

    vm_name="$(jq -r '.data.name // empty' <<< "$vm_json")"
    vm_name="${vm_name:-VM-${vmid}}"

    log "Checking backup storage ${PVE_STORAGE}..."

    storage_json="$(
        api_get \
            "${API_BASE}/nodes/${PVE_NODE}/storage/${PVE_STORAGE}/status"
    )" || die "Cannot inspect storage. Check storage permissions."

    jq -e \
        '.data.active == 1 or .data.active == true' \
        <<< "$storage_json" >/dev/null ||
        die "Backup storage is not active."

    available="$(
        jq -er '.data.avail | numbers' <<< "$storage_json"
    )" || die "Unable to determine available storage space."

    jq -n \
        --arg vmid "$vmid" \
        --arg name "$vm_name" \
        --arg node "$PVE_NODE" \
        --arg storage "$PVE_STORAGE" \
        --arg mode "snapshot" \
        --arg comment "$comment" \
        --argjson available "$available" \
        '{
            vmid: $vmid,
            name: $name,
            node: $node,
            storage: $storage,
            mode: $mode,
            comment: $comment,
            available_bytes: $available
        }'
}

# ---------------------------------------------------------------------
# Start backup and monitor its own task
# ---------------------------------------------------------------------

start_backup() {
    local vmid="$1"
    local comment="$2"

    local preflight_json
    local vm_name
    local response
    local upid
    local encoded_upid
    local status_json
    local task_status
    local exit_status
    local elapsed=0

    log "Running preflight checks..."

    preflight_json="$(preflight "$vmid" "$comment")"
    vm_name="$(jq -r '.name' <<< "$preflight_json")"

    log "Starting snapshot backup for VM ${vmid} (${vm_name})..."

    response="$(
        api_curl \
            --request POST \
            --data-urlencode "vmid=${vmid}" \
            --data-urlencode "mode=snapshot" \
            --data-urlencode "storage=${PVE_STORAGE}" \
            --data-urlencode "notes-template=${comment}" \
            "${API_BASE}/nodes/${PVE_NODE}/vzdump"
    )" || die "Proxmox rejected the backup request."

    upid="$(jq -er '.data' <<< "$response")" ||
        die "Proxmox did not return a task ID."

    log "Backup task started."
    log "Monitoring task status..."

    encoded_upid="$(jq -rn --arg value "$upid" '$value | @uri')"

    while (( elapsed < TASK_TIMEOUT )); do

        status_json="$(
            api_get \
                "${API_BASE}/nodes/${PVE_NODE}/tasks/${encoded_upid}/status"
        )" || die "Unable to retrieve backup task status. Task may still be running: ${upid}"

        task_status="$(jq -r '.data.status // empty' <<< "$status_json")"

        case "$task_status" in

            stopped)
                exit_status="$(
                    jq -r '.data.exitstatus // empty' <<< "$status_json"
                )"

                if [[ "$exit_status" == "OK" ]]; then
                    printf '\n'
                    printf 'BACKUP SUCCESSFUL\n'
                    printf 'VMID: %s\n' "$vmid"
                    printf 'VM name: %s\n' "$vm_name"
                    printf 'Node: %s\n' "$PVE_NODE"
                    printf 'Storage: %s\n' "$PVE_STORAGE"
                    printf 'Mode: snapshot\n'
                    printf 'Comment: %s\n' "$comment"
                    printf 'UPID: %s\n' "$upid"
                    return 0
                fi

                log "BACKUP FAILED"
                log "Exit status: ${exit_status:-unknown}"
                log "UPID: ${upid}"

                return 1
                ;;

            running)
                sleep "$POLL_INTERVAL"
                elapsed=$((elapsed + POLL_INTERVAL))
                ;;

            *)
                die "Unexpected task status: ${task_status:-unknown}. UPID: ${upid}"
                ;;

        esac
    done

    die "Monitoring timed out after ${TASK_TIMEOUT} seconds. The task may still be running. UPID: ${upid}"
}

# ---------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------

[[ $# -ge 1 ]] || usage

action="$1"
shift

case "$action" in

    inspect)
        [[ $# -eq 2 ]] || usage
        preflight "$1" "$2"
        ;;

    start)
        [[ $# -eq 3 && "$3" == "--confirmed" ]] || usage
        start_backup "$1" "$2"
        ;;

    *)
        usage
        ;;

esac
```
