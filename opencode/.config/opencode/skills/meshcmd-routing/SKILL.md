---
name: meshcmd-routing
description: Manage local TCP port forwarding to remote machines via MeshCmd and MeshCentral. Use when asked to connect to, route to, or tunnel to a remote machine managed by MeshCentral.
compatibility: opencode
---

# MeshCmd Port Forwarding Skill

This skill manages local TCP port forwarding to remote machines using MeshCmd, tunnelled through a MeshCentral server.

## Overview

MeshCmd (`meshcmd route`) maps a local TCP port to a port on a remote machine that has the MeshAgent installed. This works over the Internet through firewalls and proxies. Multiple simultaneous connections can be active — each one runs in its own temp directory with its own `meshaction.txt` and is tracked by PID.

## Directory Layout

All action files and the connection state file live in `~/meshcmd/`:

```
~/meshcmd/
  meshaction_<machinename>.txt   # one file per remote machine
  connections.json               # tracks active connections (PID, ports, etc.)
```

`meshcmd` binary must be on `$PATH` (confirm with `which meshcmd`).

## Action File Naming Convention

Action files are downloaded from the MeshCentral web UI (Router page for a device). Save them with the machine name appended:

```
meshaction_<machinename>.txt
```

Examples: `meshaction_48210b2d46ee.txt`, `meshaction_myserver.txt`

A minimal action file looks like:

```json
{
  "action": "route",
  "localPort": 1234,
  "remoteName": "<machinename>",
  "remoteNodeId": "node//...",
  "remoteTarget": null,
  "remotePort": 3389,
  "username": "<meshcentral_username>",
  "password": "<meshcentral_password>",
  "serverHttpsHash": "<hash>",
  "debugLevel": 0,
  "serverUrl": "wss://<server>:443/meshrelay.ashx"
}
```

The `localPort` and `remotePort` fields in the file are overridden at runtime via CLI flags — do not rely on their stored values for routing.

## Troubleshooting: stale `serverId` / `serverHttpsHash`

If a connection fails immediately after starting, the cause is often a stale `serverId` in the action file. To fix:

1. Open the machine's `~/meshcmd/meshaction_<machinename>.txt`.
2. **Delete the entire `"serverId": "..."` line** from the JSON.
3. Run `meshcmd route` again (see below). MeshCmd will contact the server, receive the current certificate, and display the new `serverHttpsHash` value.
4. **Copy the displayed hash into the file** as `"serverHttpsHash": "<newhash>"` and save it.
5. Future connections to this machine will succeed.

The `serverHttpsHash` acts as TOFU (trust-on-first-use) certificate pinning. Once saved, a mismatch (e.g. after a server cert renewal) will also cause failure — apply the same fix.

## Troubleshooting: tunnel connects but no data flows

If `meshcmd route` reports "Redirecting..." but SSH (or any service) hangs at banner exchange with no response, the most likely cause is a stale `remoteNodeId`. This happens when the MeshAgent on the remote machine is reinstalled or re-registers with the server — it receives a new node ID, making the old one in the action file invalid. The relay opens a websocket channel but has no matching agent to route traffic to.

To fix: re-download a fresh `meshaction.txt` from the MeshCentral web UI for that device (Router link on the device page) and save it as `~/meshcmd/meshaction_<machinename>.txt`. The new file will have the correct `remoteNodeId`. Note that freshly downloaded files typically have an empty `password` field and a stale `serverHttpsHash` — repopulate both before connecting.

## Port Assignment Strategy

Local ports are assigned dynamically starting from **20001**, incrementing by 1 for each new connection. Before assigning a port, check that it is not already in use:

```bash
lsof -i :<port>   # empty output = port is free
```

Read `~/meshcmd/connections.json` to find the highest currently assigned port, then use `max + 1`. If no connections exist, start at 20001.

Suggested port conventions (optional, for readability):
- SSH: `200xx` range (remote port 22)
- RDP: `210xx` range (remote port 3389)

These are conventions only — use the next free port regardless.

## SSH Credentials

When connecting via SSH, always use username **`dev`** unless the user specifies otherwise. Never use a stored or hardcoded password — always ask the user for the SSH password before attempting to connect.

## Starting a Connection

### Steps

1. **Identify the action file** for the target machine: `~/meshcmd/meshaction_<machinename>.txt`.
2. **Choose local port**: read `~/meshcmd/connections.json`, find the next available port ≥ 20001.
3. **Verify port is free**: `lsof -i :<localport>` — retry with next port if occupied.
4. **Create a temp working directory** for this connection:
   ```bash
   mkdir -p /tmp/meshcmd_<machinename>_<localport>
   ```
5. **Copy the action file** into the temp dir as `meshaction.txt`:
   ```bash
   cp ~/meshcmd/meshaction_<machinename>.txt /tmp/meshcmd_<machinename>_<localport>/meshaction.txt
   ```
6. **Launch meshcmd in the background** from the temp dir:
   ```bash
   cd /tmp/meshcmd_<machinename>_<localport> && \
   meshcmd route --localport <localport> --remoteport <remoteport> \
     >> /tmp/meshcmd_<machinename>_<localport>/output.log 2>&1 &
   PID=$!
   ```
7. **Wait briefly** (`sleep 2`) then check the log:
   - `Redirecting local port...` — tunnel is up, proceed to step 8
   - `Login token required, use --token [token]` — ask the user for their current TOTP code, kill the process, and relaunch immediately with `--token <code>` (codes expire in ~30s, so do not delay)
   - `Invalid TLS certificate` — see stale `serverId`/`serverHttpsHash` troubleshooting below
   - No output / connection refused — see stale `remoteNodeId` troubleshooting below
8. **Record the connection** in `~/meshcmd/connections.json` (see below).

### Example (SSH to machine `48210b2d46ee` on local port 20001)

```bash
mkdir -p /tmp/meshcmd_48210b2d46ee_20001
cp ~/meshcmd/meshaction_48210b2d46ee.txt /tmp/meshcmd_48210b2d46ee_20001/meshaction.txt
cd /tmp/meshcmd_48210b2d46ee_20001
meshcmd route --localport 20001 --remoteport 22 >> output.log 2>&1 &
echo "PID: $!"
```

Then connect via (ask the user for their SSH password first):
```bash
ssh -p 20001 dev@localhost
```

## `connections.json` Format

Maintained at `~/meshcmd/connections.json`. Create it if absent.

```json
{
  "connections": [
    {
      "machineName": "48210b2d46ee",
      "localPort": 20001,
      "remotePort": 22,
      "pid": 12345,
      "tempDir": "/tmp/meshcmd_48210b2d46ee_20001",
      "startedAt": "2026-04-09T10:00:00Z"
    }
  ]
}
```

## Listing Active Connections

1. Read `~/meshcmd/connections.json`.
2. For each entry, verify the process is still running: `kill -0 <pid> 2>/dev/null` (exit 0 = alive).
3. Mark dead connections as stale (process exited unexpectedly).
4. Display a table: machine name, local port, remote port, PID, status, start time.

## Stopping a Connection

1. Find the connection in `connections.json` by machine name or local port.
2. Kill the process: `kill <pid>`.
3. Remove the temp directory: `rm -rf <tempDir>`.
4. Remove the entry from `connections.json`.

## Workflow Summary

| Goal | Action |
|---|---|
| Connect to a machine | Start a connection (steps above) |
| SSH to remote machine | `ssh -p <localport> dev@localhost` (ask user for password) |
| RDP to remote machine | Connect RDP client to `localhost:<localport>` |
| List connections | Read + validate `connections.json` |
| Stop a connection | Kill PID, clean up temp dir, update JSON |
| Fix TLS cert error | Delete `serverId` from action file, restart |
| Rotate server cert hash | Delete `serverId`, run once, copy new hash |
| 2FA token required | Kill, relaunch immediately with `--token <totp_code>` |
| Tunnel up but nothing flows | Re-download action file from MeshCentral web UI |
