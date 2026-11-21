# Proxmox backup script

Script for the [Proxmox VM backup integration](README.md). Install it as `~/.local/bin/proxmox-backup.sh` with mode `700`. It reads its settings from `~/.config/proxmox/backup.env` and contains no site-specific values.


Operations:

| Operation | Purpose |
|---|---|
| `inspect --comment <comment> <vmid>...` | Check every VM and the storage; print a JSON summary. Starts nothing. |
| `start --comment <comment> --confirmed <vmid>...` | Back up the VMs one at a time and print the final report. |
| `status --upid <upid> [--log]` | Read-only check of one vzdump task, optionally with its log. |

```bash
#!/usr/bin/env bash
#
# Proxmox VE QEMU VM backup helper
#
# Operations:
#   inspect --comment <comment> <VMID> [<VMID>...]
#   start   --comment <comment> --confirmed <VMID> [<VMID>...]
#   status  --upid <UPID> [--log]
#
# Backups use snapshot mode and zstd compression.
#
# Multiple VMIDs are backed up one at a time, one Proxmox task each.
# The batch stops at the first VM whose outcome is not SUCCESS; the
# remaining VMs are reported as NOT STARTED.
#
# Backup outcomes:
#   SUCCESS  Proxmox reported the task stopped with exit status OK.
#   FAILED   Proxmox rejected the request, the request could not be sent,
#            or the task stopped with any exit status other than OK.
#   UNKNOWN  The request may have been accepted, but the result could not
#            be verified (timeout, lost response, task status unreachable).
#            Check the task in Proxmox (status --upid, or the node's task
#            list when no UPID was received) before retrying.
# This script never retries a backup submission.
#
# Exit codes:
#   0  inspect passed; start: all backups SUCCESS; status: SUCCESS
#   1  FAILED, or an error before any backup was submitted
#   2  usage error
#   3  UNKNOWN
#   4  status: task still running
#   5  Proxmox API unreachable (network) before any backup was submitted
#
# Backup comment:
#   Sent to Proxmox as the vzdump notes-template. Proxmox expands template
#   variables in it, such as {{guestname}}, {{vmid}}, {{node}} and
#   {{cluster}} (Proxmox VE 9 vzdump documentation). The template must be a
#   single line, with backslash escaped as \\; this script doubles each
#   backslash so it is stored literally. Newlines and other control
#   characters are rejected.
#
# Configuration:
#   ~/.config/proxmox/backup.env, owned by the current user, mode 0600 or
#   stricter, in a directory not writable by group or others.
#
#   PVE_HOST                HTTPS URL of the Proxmox API.
#   PVE_NODE                Fixed node name.
#   PVE_STORAGE             Fixed backup storage ID.
#   PVE_TOKEN_ID            API token ID (user@realm!token).
#   PVE_TOKEN_SECRET        API token secret.
#   PVE_CA_FILE             Optional CA certificate used to verify the node
#                           (for example a copy of the node's pve-root-ca.pem).
#   PVE_VERIFY_TLS          true (default) or false. false skips certificate
#                           verification (common with self-signed Proxmox
#                           certificates); every run then prints a warning
#                           and reports TLS verification as DISABLED.
#   PVE_LOW_SPACE_PERCENT   Warn when free space on the backup storage is
#                           below this percentage of its size (default 20).
#
# Dependencies:
#   bash, curl, jq, flock, sed, dirname, stat, id, mkdir
#
# Security:
#   - Fixed Proxmox host, node and storage.
#   - API token never passed as a command-line argument.
#   - TLS verification on by default; disabling it is reported on every run.
#   - No VM power/configuration operations.
#   - No backup deletion or restoration.
#   - One backup run at a time from this VM (lock file).

set -Eeuo pipefail
shopt -s inherit_errexit
umask 077
export LC_ALL=C

readonly ENV_FILE="${HOME}/.config/proxmox/backup.env"
readonly SCRIPT_NAME="proxmox-backup.sh"
readonly TASK_TIMEOUT=43200
readonly POLL_INTERVAL=5
readonly STATUS_MAX_FAILURES=12
readonly MAX_VMIDS=20
readonly LOG_LIMIT=1000
readonly BACKUP_MODE="snapshot"
readonly BACKUP_COMPRESS="zstd"
readonly LOCK_FILE="${XDG_RUNTIME_DIR:-${HOME}/.cache}/proxmox-backup.lock"

BASE_URL=""
API_BASE=""
PVE_NODE=""
PVE_STORAGE=""
PVE_TOKEN_ID=""
PVE_TOKEN_SECRET=""
PVE_CA_FILE=""
PVE_VERIFY_TLS="true"
PVE_LOW_SPACE_PERCENT="20"
TLS_MODE=""

die() {
    printf 'ERROR: %s\n' "$*" >&2
    exit 1
}

die_unreachable() {
    printf 'ERROR: UNREACHABLE: %s\n' "$*" >&2
    exit 5
}

log() {
    printf '%s\n' "$*" >&2
}

usage() {
    cat <<EOF
Usage:

  ${SCRIPT_NAME} inspect --comment <comment> <VMID> [<VMID>...]

  ${SCRIPT_NAME} start --comment <comment> --confirmed <VMID> [<VMID>...]

  ${SCRIPT_NAME} status --upid <UPID> [--log]

Examples:

  ${SCRIPT_NAME} inspect --comment "Before upgrading Docker" 105

  ${SCRIPT_NAME} start --comment "Before upgrading Docker" --confirmed 105 106

  ${SCRIPT_NAME} status --upid 'UPID:node:0001A2B3:00C4D5E6:6B01AB6E:vzdump:105:user@pve!token:' --log

EOF
    exit 2
}

# Collapses a value to one line of at most 300 characters for reporting.
one_line() {
    local value="${1//$'\r'/ }"
    value="${value//$'\n'/ }"
    printf '%s' "${value:0:300}"
}

# ---------------------------------------------------------------------
# Dependency checks
# ---------------------------------------------------------------------

for cmd in curl jq flock sed dirname stat id mkdir; do
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

env_dir="$(dirname -- "$ENV_FILE")"
env_mode="$(stat -c '%a' -- "$ENV_FILE")" ||
    die "Cannot read configuration file permissions."
env_owner="$(stat -c '%u' -- "$ENV_FILE")" ||
    die "Cannot read configuration file owner."
dir_mode="$(stat -c '%a' -- "$env_dir")" ||
    die "Cannot read configuration directory permissions."
dir_owner="$(stat -c '%u' -- "$env_dir")" ||
    die "Cannot read configuration directory owner."

[[ "$env_owner" == "$(id -u)" && "$dir_owner" == "$(id -u)" ]] ||
    die "Configuration file and its directory must be owned by the current user."

(( (8#$env_mode & 8#077) == 0 )) ||
    die "Configuration file permissions are ${env_mode}; they must be 0600 or stricter (chmod 600 ${ENV_FILE})."

(( (8#$dir_mode & 8#022) == 0 )) ||
    die "Configuration directory permissions are ${dir_mode}; it must not be writable by group or others (chmod 700 ${env_dir})."

# Configuration is a trusted, user-owned shell file.
# shellcheck disable=SC1090
source "$ENV_FILE"

: "${PVE_HOST:?Missing PVE_HOST}"
: "${PVE_NODE:?Missing PVE_NODE}"
: "${PVE_STORAGE:?Missing PVE_STORAGE}"
: "${PVE_TOKEN_ID:?Missing PVE_TOKEN_ID}"
: "${PVE_TOKEN_SECRET:?Missing PVE_TOKEN_SECRET}"

PVE_CA_FILE="${PVE_CA_FILE:-}"
PVE_VERIFY_TLS="${PVE_VERIFY_TLS:-true}"
PVE_LOW_SPACE_PERCENT="${PVE_LOW_SPACE_PERCENT:-20}"

[[ "$PVE_HOST" == https://* ]] ||
    die "HTTPS is required."

[[ "$PVE_VERIFY_TLS" == "true" || "$PVE_VERIFY_TLS" == "false" ]] ||
    die "PVE_VERIFY_TLS must be true or false."

[[ "$PVE_LOW_SPACE_PERCENT" =~ ^[0-9]{1,2}$ ]] ||
    die "PVE_LOW_SPACE_PERCENT must be a whole number from 0 to 99."

# Reject unexpected characters in configuration values.
# The generated Proxmox token secret is a UUID.
for value in \
    "$PVE_HOST" \
    "$PVE_NODE" \
    "$PVE_STORAGE" \
    "$PVE_TOKEN_ID" \
    "$PVE_TOKEN_SECRET" \
    "$PVE_CA_FILE"
do
    [[ "$value" != *$'\n'* && "$value" != *$'\r'* ]] ||
        die "Invalid newline in configuration."
done

[[ "$PVE_TOKEN_SECRET" =~ ^[a-zA-Z0-9_-]+$ ]] ||
    die "Unexpected characters in API token secret."

[[ "$PVE_TOKEN_ID" =~ ^[a-zA-Z0-9@._-]+![a-zA-Z0-9_-]+$ ]] ||
    die "Invalid API token ID."

[[ "$PVE_NODE" =~ ^[a-zA-Z0-9._-]+$ ]] ||
    die "Invalid node name."

[[ "$PVE_STORAGE" =~ ^[a-zA-Z0-9._-]+$ ]] ||
    die "Invalid storage ID."

if [[ "$PVE_VERIFY_TLS" == "false" ]]; then
    [[ -z "$PVE_CA_FILE" ]] ||
        die "PVE_CA_FILE is set but PVE_VERIFY_TLS is false."
    TLS_MODE="DISABLED"
    log "WARNING: TLS certificate verification is disabled (PVE_VERIFY_TLS=false). The API token could be intercepted on the network path. Setting PVE_CA_FILE to the node's pve-root-ca.pem enables verification with a self-signed certificate."
elif [[ -n "$PVE_CA_FILE" ]]; then
    [[ -f "$PVE_CA_FILE" && -r "$PVE_CA_FILE" ]] ||
        die "PVE_CA_FILE is not a readable file: ${PVE_CA_FILE}"
    TLS_MODE="verified (PVE_CA_FILE)"
else
    TLS_MODE="verified (system CA store)"
fi

BASE_URL="${PVE_HOST%/}"
API_BASE="${BASE_URL}/api2/json"

# ---------------------------------------------------------------------
# API helpers
# ---------------------------------------------------------------------

api_curl() {
    local tls_options=()

    if [[ "$PVE_VERIFY_TLS" == "false" ]]; then
        tls_options+=(--insecure)
    elif [[ -n "$PVE_CA_FILE" ]]; then
        tls_options+=(--cacert "$PVE_CA_FILE")
    fi

    curl \
        --silent \
        --show-error \
        --fail-with-body \
        --proto '=https' \
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

# Prints a short, single-line error message from a Proxmox error body.
pve_error() {
    local body="$1"
    local message

    message="$(
        jq -r '
            [ (.message // empty),
              (.errors // empty | tostring) ]
            | map(select(. != "")) | join(" ")
        ' <<< "$body" 2>/dev/null
    )" || message="$body"

    one_line "$message"
}

# Short description of a curl exit code that means the network path failed.
curl_reason() {
    case "$1" in
        6)  printf 'host name could not be resolved' ;;
        7)  printf 'connection refused or host unreachable' ;;
        28) printf 'connection or response timed out' ;;
        52) printf 'empty reply from server' ;;
        55|56) printf 'connection reset or broken' ;;
        *)  printf 'curl exit %s' "$1" ;;
    esac
}

# Exits with a message that separates network, TLS, authentication,
# permission and other API errors. Network failures exit with code 5.
api_failure_exit() {
    local rc="$1"
    local http_code="$2"
    local body="$3"
    local what="$4"

    case "$rc" in
        22)
            case "$http_code" in
                401) die "Authentication failed while ${what} (HTTP 401). Check PVE_TOKEN_ID and PVE_TOKEN_SECRET." ;;
                403) die "Permission denied while ${what} (HTTP 403): $(pve_error "$body")" ;;
                *)   die "Proxmox returned HTTP ${http_code} while ${what}: $(pve_error "$body")" ;;
            esac
            ;;
        35|58|60|77|83|90|91)
            die "TLS error connecting to Proxmox at ${BASE_URL} while ${what} (curl exit ${rc}). Check the certificate settings (PVE_VERIFY_TLS, PVE_CA_FILE)."
            ;;
        *)
            die_unreachable "Proxmox API at ${BASE_URL} is unreachable while ${what} ($(curl_reason "$rc")). This is a network problem, not a permission problem. No backup was submitted."
            ;;
    esac
}

# Runs a GET request and prints the response body. On failure, exits
# through api_failure_exit.
api_get_or_die() {
    local url="$1"
    local what="$2"
    local response
    local rc=0
    local http_code
    local body

    response="$(api_curl --write-out '\n%{http_code}' "$url")" || rc=$?

    if [[ "$response" == *$'\n'* ]]; then
        http_code="${response##*$'\n'}"
        body="${response%$'\n'*}"
    else
        http_code="000"
        body="$response"
    fi

    if (( rc == 0 )); then
        printf '%s\n' "$body"
        return 0
    fi

    api_failure_exit "$rc" "$http_code" "$body" "$what"
}

# ---------------------------------------------------------------------
# Input validation
# ---------------------------------------------------------------------

validate_comment() {
    local comment="$1"

    [[ -n "$comment" ]] ||
        die "A backup comment is required."

    [[ ! "$comment" =~ [[:cntrl:]] ]] ||
        die "Backup comment must be a single line without control characters."

    (( ${#comment} <= 1024 )) ||
        die "Backup comment exceeds 1024 bytes."
}

validate_vmid() {
    [[ "$1" =~ ^[1-9][0-9]{2,8}$ ]] ||
        die "Invalid VMID: $1 (expected 100-999999999, digits only)."
}

# Accepts only vzdump task UPIDs on the configured node.
validate_upid() {
    local upid="$1"
    local node

    [[ "$upid" =~ ^UPID:([a-zA-Z0-9._-]+):[0-9A-Fa-f]{8,}:[0-9A-Fa-f]{8,}:[0-9A-Fa-f]{8,}:vzdump:[0-9]*:[a-zA-Z0-9@._!-]+:$ ]] ||
        die "Invalid UPID: expected a vzdump task UPID."

    node="${BASH_REMATCH[1]}"

    [[ "$node" == "$PVE_NODE" ]] ||
        die "UPID belongs to node ${node}, not the configured node ${PVE_NODE}."
}

# Escapes a comment for use as a vzdump notes-template. Proxmox treats
# backslash as an escape character in the template, so each backslash is
# doubled to be stored literally.
notes_template_for() {
    printf '%s' "${1//\\/\\\\}"
}

# ---------------------------------------------------------------------
# Preflight checks
# ---------------------------------------------------------------------

# Prints the VM name for one VMID, or exits if the VM cannot be read.
vm_name_for() {
    local vmid="$1"
    local vm_json
    local vm_name

    log "Checking VM ${vmid}..."

    vm_json="$(
        api_get_or_die \
            "${API_BASE}/nodes/${PVE_NODE}/qemu/${vmid}/config" \
            "reading the configuration of VM ${vmid} (needs VM.Audit; the VM must exist on ${PVE_NODE})"
    )" || exit $?

    vm_name="$(jq -r '.data.name // empty' <<< "$vm_json")"
    printf '%s\n' "${vm_name:-VM-${vmid}}"
}

# Checks every VM and the backup storage before any backup starts.
# Prints a JSON summary on stdout.
preflight() {
    local comment="$1"
    shift

    local vms_json='[]'
    local storage_json
    local storage_report
    local vmid
    local vm_name

    validate_comment "$comment"

    for vmid in "$@"; do
        validate_vmid "$vmid"
    done

    for vmid in "$@"; do
        vm_name="$(vm_name_for "$vmid")" || exit $?
        vms_json="$(
            jq -c \
                --arg vmid "$vmid" \
                --arg name "$vm_name" \
                '. + [{vmid: $vmid, name: $name}]' \
                <<< "$vms_json"
        )"
    done

    log "Checking backup storage ${PVE_STORAGE}..."

    storage_json="$(
        api_get_or_die \
            "${API_BASE}/nodes/${PVE_NODE}/storage/${PVE_STORAGE}/status" \
            "reading the status of storage ${PVE_STORAGE} (needs Datastore.Audit)"
    )" || exit $?

    jq -e \
        '.data.active == 1 or .data.active == true' \
        <<< "$storage_json" >/dev/null ||
        die "Backup storage is not active."

    storage_report="$(
        jq -ce \
            --argjson threshold "$PVE_LOW_SPACE_PERCENT" '
            .data
            | select((.avail | type) == "number" and (.total | type) == "number" and .total > 0)
            | {
                available_mb: ((.avail / 1000000) | floor),
                total_mb: ((.total / 1000000) | floor),
                used_percent: ((((.total - .avail) * 1000 / .total) | floor) / 10),
                low_space_threshold_percent: $threshold,
                low_space_warning: ((.avail * 100) < ($threshold * .total))
              }
            ' <<< "$storage_json"
    )" || die "Unable to determine storage size and available space."

    if jq -e '.low_space_warning' <<< "$storage_report" >/dev/null; then
        log "WARNING: free space on ${PVE_STORAGE} is below ${PVE_LOW_SPACE_PERCENT}% of its size."
    fi

    jq -n \
        --arg node "$PVE_NODE" \
        --arg storage "$PVE_STORAGE" \
        --arg mode "$BACKUP_MODE" \
        --arg compress "$BACKUP_COMPRESS" \
        --arg comment "$comment" \
        --arg tls "$TLS_MODE" \
        --argjson space "$storage_report" \
        --argjson vms "$vms_json" \
        '{
            node: $node,
            storage: $storage,
            mode: $mode,
            compression: $compress,
            comment: $comment,
            tls_verification: $tls,
            storage_space: ($space + {
                unit: "MB (1 MB = 1,000,000 bytes)",
                note: "Free space does not guarantee the backup will fit. Backup size depends on VM disk usage and compression."
            }),
            vms: $vms
        }'
}

# ---------------------------------------------------------------------
# Backup one VM and monitor its task
# ---------------------------------------------------------------------

# Prints three lines: outcome (SUCCESS, FAILED or UNKNOWN), UPID (empty
# if none was received) and a one-line detail.
emit_result() {
    printf '%s\n%s\n%s\n' "$1" "$2" "$(one_line "$3")"
}

# Submits one backup and monitors its task. Never resubmits.
# Returns 0 for SUCCESS, 1 for FAILED, 3 for UNKNOWN.
backup_one() {
    local vmid="$1"
    local vm_name="$2"
    local comment="$3"

    local notes
    local response
    local rc=0
    local http_code
    local body
    local upid
    local encoded_upid
    local status_json
    local task_status
    local exit_status
    local elapsed=0
    local failures=0
    local status_rc=0

    notes="$(notes_template_for "$comment")"

    log "Starting snapshot backup for VM ${vmid} (${vm_name})..."

    response="$(
        api_curl \
            --request POST \
            --write-out '\n%{http_code}' \
            --data-urlencode "vmid=${vmid}" \
            --data-urlencode "mode=${BACKUP_MODE}" \
            --data-urlencode "compress=${BACKUP_COMPRESS}" \
            --data-urlencode "storage=${PVE_STORAGE}" \
            --data-urlencode "notes-template=${notes}" \
            "${API_BASE}/nodes/${PVE_NODE}/vzdump"
    )" || rc=$?

    if [[ "$response" == *$'\n'* ]]; then
        http_code="${response##*$'\n'}"
        body="${response%$'\n'*}"
    else
        http_code="000"
        body="$response"
    fi

    case "$rc" in
        0)
            upid="$(
                jq -er '.data | strings | select(startswith("UPID:"))' \
                    <<< "$body" 2>/dev/null
            )" || {
                emit_result UNKNOWN "" "Proxmox answered HTTP ${http_code} without a task ID. A backup task may have been created; check the node's task list before retrying."
                return 3
            }
            ;;
        22)
            if [[ "$http_code" =~ ^(4[0-9][0-9]|500)$ ]]; then
                emit_result FAILED "" "Proxmox rejected the backup request (HTTP ${http_code}): $(pve_error "$body")"
                return 1
            fi
            emit_result UNKNOWN "" "Backup submission outcome unknown (HTTP ${http_code}). The task may have been accepted; check the node's task list before retrying."
            return 3
            ;;
        3|6|7)
            emit_result FAILED "" "Backup request not sent: Proxmox API unreachable ($(curl_reason "$rc")). Network problem."
            return 1
            ;;
        35|58|60|77|83|90|91)
            emit_result FAILED "" "Backup request not sent: TLS error connecting to Proxmox (curl exit ${rc})."
            return 1
            ;;
        *)
            emit_result UNKNOWN "" "Backup submission outcome unknown (curl exit ${rc}, HTTP ${http_code}). The task may have been accepted; check the node's task list before retrying."
            return 3
            ;;
    esac

    log "Backup task started: ${upid}"
    log "Monitoring task status..."

    encoded_upid="$(jq -rn --arg value "$upid" '$value | @uri')"

    while (( elapsed < TASK_TIMEOUT )); do

        status_rc=0
        status_json="$(
            api_get \
                "${API_BASE}/nodes/${PVE_NODE}/tasks/${encoded_upid}/status"
        )" || status_rc=$?

        if (( status_rc == 0 )); then
            failures=0
            task_status="$(jq -r '.data.status // empty' <<< "$status_json" 2>/dev/null || true)"

            case "$task_status" in
                stopped)
                    exit_status="$(jq -r '.data.exitstatus // empty' <<< "$status_json" 2>/dev/null || true)"
                    log "VM ${vmid}: task stopped, exit status ${exit_status:-unknown}"

                    if [[ "$exit_status" == "OK" ]]; then
                        emit_result SUCCESS "$upid" "Task exit status OK"
                        return 0
                    fi

                    emit_result FAILED "$upid" "Task exit status: ${exit_status:-unknown}"
                    return 1
                    ;;
                running)
                    ;;
                *)
                    emit_result UNKNOWN "$upid" "Unexpected task status: ${task_status:-empty}"
                    return 3
                    ;;
            esac
        else
            failures=$((failures + 1))
            log "Task status check failed (${failures}/${STATUS_MAX_FAILURES}): $(curl_reason "$status_rc")."

            if (( failures >= STATUS_MAX_FAILURES )); then
                emit_result UNKNOWN "$upid" "Task status could not be retrieved after ${STATUS_MAX_FAILURES} attempts (last error: $(curl_reason "$status_rc")); the task may still be running"
                return 3
            fi
        fi

        sleep "$POLL_INTERVAL"
        elapsed=$((elapsed + POLL_INTERVAL))
    done

    emit_result UNKNOWN "$upid" "Monitoring timed out after ${TASK_TIMEOUT} seconds; the task may still be running"
    return 3
}

# ---------------------------------------------------------------------
# Start backups for all requested VMs, one at a time
# ---------------------------------------------------------------------

R_VMIDS=()
R_NAMES=()
R_RESULTS=()
R_UPIDS=()
R_DETAILS=()

start_backups() {
    local comment="$1"
    shift

    local preflight_json
    local count
    local i
    local output
    local rc
    local overall="SUCCESS"

    mkdir -p -- "$(dirname -- "$LOCK_FILE")"
    exec 9>"$LOCK_FILE"
    flock --nonblock 9 ||
        die "Another backup run from this VM is in progress."

    log "Running preflight checks..."

    preflight_json="$(preflight "$comment" "$@")" || exit $?
    count="$(jq '.vms | length' <<< "$preflight_json")"

    for (( i = 0; i < count; i++ )); do
        R_VMIDS[i]="$(jq -r --argjson i "$i" '.vms[$i].vmid' <<< "$preflight_json")"
        R_NAMES[i]="$(jq -r --argjson i "$i" '.vms[$i].name' <<< "$preflight_json")"
        R_RESULTS[i]="NOT STARTED"
        R_UPIDS[i]=""
        R_DETAILS[i]=""
    done

    for (( i = 0; i < count; i++ )); do
        rc=0
        output="$(backup_one "${R_VMIDS[i]}" "${R_NAMES[i]}" "$comment")" || rc=$?

        R_RESULTS[i]="$(sed -n '1p' <<< "$output")"
        R_UPIDS[i]="$(sed -n '2p' <<< "$output")"
        R_DETAILS[i]="$(sed -n '3p' <<< "$output")"

        case "$rc" in
            0) ;;
            3) overall="UNKNOWN"; break ;;
            *) overall="FAILED"; R_RESULTS[i]="FAILED"; break ;;
        esac
    done

    print_report "$comment" "$overall" "$count"

    case "$overall" in
        SUCCESS) return 0 ;;
        UNKNOWN) return 3 ;;
        *) return 1 ;;
    esac
}

# ---------------------------------------------------------------------
# Final report
# ---------------------------------------------------------------------

# Prints the overall result, then one detailed group per distinct
# node/storage/mode/compression/comment combination, each followed by its
# VMs. Every VM in a run currently shares the configured node, storage,
# mode, compression and comment, so a run produces a single group.
print_report() {
    local comment="$1"
    local overall="$2"
    local count="$3"

    local -a group_keys=()
    local -A group_members=()
    local key
    local i
    local j
    local g_node
    local g_storage
    local g_mode
    local g_compress
    local g_comment

    for (( i = 0; i < count; i++ )); do
        key="${PVE_NODE}"$'\x1f'"${PVE_STORAGE}"$'\x1f'"${BACKUP_MODE}"$'\x1f'"${BACKUP_COMPRESS}"$'\x1f'"${comment}"
        if [[ -z "${group_members[$key]+x}" ]]; then
            group_keys+=("$key")
            group_members[$key]="$i"
        else
            group_members[$key]+=" $i"
        fi
    done

    printf '\n'
    printf 'BACKUP %s\n' "$overall"

    if [[ "$overall" == "UNKNOWN" ]]; then
        printf 'WARNING: A backup task may have been accepted but its result is unverified. Do not retry until the task has been checked in Proxmox.\n'
    fi

    for key in "${group_keys[@]}"; do
        IFS=$'\x1f' read -r g_node g_storage g_mode g_compress g_comment <<< "$key"

        printf '\n'
        printf 'Node: %s\n' "$g_node"
        printf 'Storage: %s\n' "$g_storage"
        printf 'Mode: %s\n' "$g_mode"
        printf 'Compression: %s\n' "$g_compress"
        printf 'Comment: %s\n' "$g_comment"
        printf 'TLS verification: %s\n' "$TLS_MODE"

        for j in ${group_members[$key]}; do
            printf '\n'
            printf 'VMID: %s\n' "${R_VMIDS[j]}"
            printf 'VM name: %s\n' "${R_NAMES[j]}"
            printf 'Result: %s\n' "${R_RESULTS[j]}"

            case "${R_RESULTS[j]}" in
                FAILED|UNKNOWN)
                    printf 'Detail: %s\n' "${R_DETAILS[j]:-none}"
                    ;;
            esac

            if [[ -n "${R_UPIDS[j]}" ]]; then
                printf 'UPID: %s\n' "${R_UPIDS[j]}"
            fi

            if [[ "${R_RESULTS[j]}" == "UNKNOWN" ]]; then
                if [[ -n "${R_UPIDS[j]}" ]]; then
                    printf "Check: %s status --upid '%s' --log\n" "$SCRIPT_NAME" "${R_UPIDS[j]}"
                else
                    printf 'Check: no UPID was received; review the task list of node %s in Proxmox before retrying.\n' "$PVE_NODE"
                fi
            fi
        done
    done
}

# ---------------------------------------------------------------------
# Task status lookup
# ---------------------------------------------------------------------

# Prints a JSON report for one vzdump task. Returns 0 for SUCCESS,
# 1 for FAILED, 3 for UNKNOWN, 4 while the task is running.
task_status_report() {
    local upid="$1"
    local want_log="$2"

    local encoded_upid
    local status_json
    local task_status=""
    local exit_status=""
    local outcome
    local rc
    local log_raw
    local log_json='null'
    local log_error=""
    local detail=""

    validate_upid "$upid"

    encoded_upid="$(jq -rn --arg value "$upid" '$value | @uri')"

    local status_rc=0
    status_json="$(
        api_get "${API_BASE}/nodes/${PVE_NODE}/tasks/${encoded_upid}/status"
    )" || status_rc=$?

    if (( status_rc == 0 )); then
        task_status="$(jq -r '.data.status // empty' <<< "$status_json" 2>/dev/null || true)"
        exit_status="$(jq -r '.data.exitstatus // empty' <<< "$status_json" 2>/dev/null || true)"

        case "$task_status" in
            stopped)
                if [[ "$exit_status" == "OK" ]]; then
                    outcome="SUCCESS"; rc=0
                else
                    outcome="FAILED"; rc=1
                fi
                ;;
            running)
                outcome="RUNNING"; rc=4
                ;;
            *)
                outcome="UNKNOWN"; rc=3
                ;;
        esac
    else
        outcome="UNKNOWN"; rc=3
        case "$status_rc" in
            6|7|28|52|55|56)
                detail="Proxmox API unreachable ($(curl_reason "$status_rc")); task status not retrieved." ;;
            *)
                detail="Task status could not be retrieved from Proxmox (curl exit ${status_rc})." ;;
        esac
    fi

    if [[ "$want_log" == "true" ]]; then
        if log_raw="$(
            api_get "${API_BASE}/nodes/${PVE_NODE}/tasks/${encoded_upid}/log?start=0&limit=${LOG_LIMIT}"
        )" && log_json="$(jq -c '[.data[]? | .t]' <<< "$log_raw" 2>/dev/null)"; then
            :
        else
            log_json='null'
            log_error="Task log could not be retrieved."
        fi
    fi

    jq -n \
        --arg upid "$upid" \
        --arg node "$PVE_NODE" \
        --arg task_status "$task_status" \
        --arg exit_status "$exit_status" \
        --arg outcome "$outcome" \
        --argjson log "$log_json" \
        --arg log_error "$log_error" \
        --arg detail "$detail" \
        --argjson log_limit "$LOG_LIMIT" \
        '{
            upid: $upid,
            node: $node,
            task_status: (if $task_status == "" then null else $task_status end),
            exit_status: (if $exit_status == "" then null else $exit_status end),
            outcome: $outcome
        }
        + (if $log != null then {log: $log, log_line_limit: $log_limit} else {} end)
        + (if $detail != "" then {detail: $detail} else {} end)
        + (if $log_error != "" then {log_error: $log_error} else {} end)'

    return "$rc"
}

# ---------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------

[[ $# -ge 1 ]] || usage

action="$1"
shift

comment=""
comment_set="false"
confirmed="false"
upid=""
upid_set="false"
want_log="false"
vmids=()

while [[ $# -gt 0 ]]; do
    case "$1" in
        --comment)
            [[ $# -ge 2 && "$comment_set" == "false" ]] || usage
            comment="$2"
            comment_set="true"
            shift 2
            ;;
        --confirmed)
            confirmed="true"
            shift
            ;;
        --upid)
            [[ $# -ge 2 && "$upid_set" == "false" ]] || usage
            upid="$2"
            upid_set="true"
            shift 2
            ;;
        --log)
            want_log="true"
            shift
            ;;
        -*)
            usage
            ;;
        *)
            validate_vmid "$1"
            vmids+=("$1")
            shift
            ;;
    esac
done

case "$action" in

    inspect|start)
        [[ "$comment_set" == "true" && ${#vmids[@]} -ge 1 ]] || usage
        [[ "$upid_set" == "false" && "$want_log" == "false" ]] || usage

        # Remove duplicate VMIDs, keeping the first occurrence.
        declare -A seen=()
        unique_vmids=()
        for vmid in "${vmids[@]}"; do
            if [[ -z "${seen[$vmid]+x}" ]]; then
                seen[$vmid]=1
                unique_vmids+=("$vmid")
            fi
        done

        (( ${#unique_vmids[@]} <= MAX_VMIDS )) ||
            die "Too many VMIDs (maximum ${MAX_VMIDS})."

        if [[ "$action" == "inspect" ]]; then
            [[ "$confirmed" == "false" ]] || usage
            preflight "$comment" "${unique_vmids[@]}"
        else
            [[ "$confirmed" == "true" ]] || usage
            status=0
            start_backups "$comment" "${unique_vmids[@]}" || status=$?
            exit "$status"
        fi
        ;;

    status)
        [[ "$upid_set" == "true" && ${#vmids[@]} -eq 0 ]] || usage
        [[ "$comment_set" == "false" && "$confirmed" == "false" ]] || usage
        status=0
        task_status_report "$upid" "$want_log" || status=$?
        exit "$status"
        ;;

    *)
        usage
        ;;

esac
```
