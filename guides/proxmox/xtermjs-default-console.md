# Default to xterm.js for chosen Proxmox VMs

Make the Proxmox web UI open the **xterm.js** serial console, instead of noVNC, only for the VMs you pick. Other VMs keep noVNC.

Tested on: Proxmox VE `<your version>`

> **Heads up:** this patches a file shipped by Proxmox (`pvemanagerlib.js`). It is not supported by Proxmox, and a package upgrade can replace the file. An apt hook re-applies the patch, and the script refuses to patch if the upstream code has changed.

## How it works

The console tab uses xterm.js for a VM only when **both** are true:

1. The VM has a serial socket (`serial0: socket`).
2. The VM's notes (description) contain the word `xtermjs`.

The check runs once when you open a VM in the web UI. The Console tab then uses the stored result when you click it.

## Prerequisites

Add a serial port to each VM you want to use this on, then fully stop and start it (a reboot from inside the guest is not enough):

```
qm set <vmid> --serial0 socket
```

The guest also needs a login prompt on the serial port. Many cloud images already have one. On Debian or Ubuntu guests that don't:

```
systemctl enable --now serial-getty@ttyS0
```

## Step 1: Create the patch script

Run this in the PVE shell. It creates the script and makes it executable:

```
cat <<'EOF' > /usr/local/bin/pve-xtermjs-console-patch.sh
#!/bin/bash
# Console tab: xterm.js when the VM has a serial socket AND its notes contain "xtermjs"; noVNC otherwise.
# The decision is kept in window.pveXtermVms[vmid] so it survives PVE rebuilding the panel.
FILE=/usr/share/pve-manager/js/pvemanagerlib.js
MARK='pve-serial-xterm'
A1='queryDict[type] = 1;'
A2='consoleBtn.setEnableXtermJS(xtermjs);'
P1="/*pve-serial-xterm*/ queryDict[(me.consoleType === 'kvm' && (window.pveXtermVms || {})[me.vmid]) ? 'xtermjs' : type] = 1;"
P2="/*pve-serial-xterm*/ if (success && !me.pveXtermChecked) { me.pveXtermChecked = true; Proxmox.Utils.API2Request({ url: base_url + '/config', method: 'GET', success: function (response) { try { let d = (response.result.data.description || ''); let want = !!xtermjs && /xtermjs/i.test(d); window.pveXtermVms = window.pveXtermVms || {}; let changed = !!window.pveXtermVms[vmid] !== want; window.pveXtermVms[vmid] = want; let con = me.down('#console'); if (want && changed && con && con.rendered && con.isVisible(true)) { con.fireEvent('activate', con); } } catch (e) { console.warn('pve-serial-xterm', e); } } }); }"

log() { logger -t pve-xtermjs-console-patch "$*"; echo "$*"; }

[ -f "$FILE" ] || { log "missing $FILE"; exit 1; }
grep -q "$MARK" "$FILE" && { log "already patched"; exit 0; }

for a in "$A1" "$A2"; do
    n=$(grep -cF -- "$a" "$FILE")
    [ "$n" -eq 1 ] || { log "anchor '$a' found $n times (expected 1); NOT patching, upstream code changed"; exit 1; }
done

cp -a "$FILE" "$FILE.orig"

TMP=$(mktemp) || exit 1
trap 'rm -f "$TMP"' EXIT
cp "$FILE" "$TMP"

A1="$A1" P1="$P1" A2="$A2" P2="$P2" perl -0pi -e '
    my ($a1,$p1,$a2,$p2) = @ENV{qw(A1 P1 A2 P2)};
    s/\Q$a1\E/$p1/;
    s/\Q$a2\E/$a2\n$p2/;
' "$TMP" || { log "perl failed"; exit 1; }

[ "$(grep -c "$MARK" "$TMP")" -eq 2 ] || { log "patch did not apply cleanly; file untouched"; exit 1; }
if command -v node >/dev/null; then
    node --check "$TMP" || { log "JS syntax check failed; file untouched"; exit 1; }
fi

cat "$TMP" > "$FILE"
log "patched $FILE"
curl -sk --max-time 5 https://127.0.0.1:8006/pve2/js/pvemanagerlib.js 2>/dev/null | grep -q "pve-serial-xterm" || systemctl reload pveproxy || true
EOF
chmod +x /usr/local/bin/pve-xtermjs-console-patch.sh
```

What the script does for safety:

- Patches only if both expected code anchors appear exactly once. Otherwise it logs a message and changes nothing.
- Saves a backup of the original as `/usr/share/pve-manager/js/pvemanagerlib.js.orig`.
- Applies the change to a temporary copy, checks it (and runs a JavaScript syntax check if `node` is installed), and only then replaces the real file.
- Does nothing if the file is already patched.

## Step 2: Create an apt hook

This re-applies the patch after package upgrades. Because the script exits early when the file is already patched, the extra runs are harmless.

```
cat <<'EOF' > /etc/apt/apt.conf.d/99-pve-xtermjs-console-patch
DPkg::Post-Invoke { "/usr/local/bin/pve-xtermjs-console-patch.sh || true"; };
EOF
```

## Step 3: Run it and reload the browser

Run the script once:

```
/usr/local/bin/pve-xtermjs-console-patch.sh
```

Then hard-reload the web UI (Ctrl+Shift+R in Firefox and Chrome).

## Step 4: Choose which VMs use xterm.js

For each VM, put `xtermjs` in its notes (VM, Summary, Notes). To keep it out of sight in the UI, use an HTML comment, which doesn't render:

```
<!-- do not remove, for xtermjs console override -->
```

Open the VM and click Console. A VM with a serial socket and `xtermjs` in its notes opens xterm.js. Every other VM still opens noVNC.

## Troubleshooting

- **Still noVNC:** confirm `serial0: socket` appears in `qm config <vmid>`, that the notes contain `xtermjs`, and that you **hard-reloaded the page**.
- **Blank xterm.js console:** the guest has no login prompt on the serial port. See Prerequisites.
- **Script says "anchor ... found N times":** a Proxmox update changed the code, so the script refused to patch. Check `journalctl -t pve-xtermjs-console-patch`. The script has to be updated for the new version.

## Undo

Reinstall the original package file:

```
apt install --reinstall pve-manager
```

To remove it completely, also delete the apt hook (`/etc/apt/apt.conf.d/99-pve-xtermjs-console-patch`) and the script.