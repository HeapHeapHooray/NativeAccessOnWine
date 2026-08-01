# Native Access & NTK Daemon under Wine Guide

This guide explains how Native Access and its background service (**NTK Daemon**) interact, why Native Access fails under Wine, and how to reliably run them in any Wine prefix.

---

## Overview & Architecture

**Native Access** is an Electron-based application from Native Instruments used for registering, downloading, and updating software and sound libraries.

Native Access relies entirely on a background Windows service:
* **NTK Daemon**: `NTKDaemonService`, binary at `C:\Program Files\Common Files\Native Instruments\NTK\NTKDaemon.exe`
* **Communication protocol**: ZeroMQ sockets over TCP loopback:
  * **Send / Request**: `tcp://127.0.0.1:5146`
  * **Subscribe / Events**: `tcp://127.0.0.1:5563`
  * **Auxiliary**: `tcp://127.0.0.1:7865`

---

## How Native Access Startup Works

On launch, Native Access follows this sequence:

1. **Ping Daemon**: Sends a `daemonVersionRequest` over ZeroMQ and compares the response with its bundled daemon version (e.g. `1.31.1.0`).
2. **Stop Service on Failure**: If the daemon does not respond, Native Access issues `sc stop NTKDaemonService`.
3. **Version & Installation Check**: Reads `C:\Users\Public\Documents\Native Instruments\NTK\install.json`.
   * If missing or mismatched, it tries to **reinstall the daemon elevated** using the bundled `resources\daemon\win\NTKDaemon <version> Setup PC.exe`, via the `@vscode/sudo-prompt` npm module.
4. **Service Elevation**: If installed but stopped, it tries to start the service with:
   ```cmd
   powershell -command "start-process cmd -verb runas -argumentlist '/c sc start NTKDaemonService'"
   ```

On real Windows, steps 3–4 trigger a UAC prompt. **Under Wine, both elevation paths are non-functional.**

---

## Why Native Access Fails under Wine

### 1. Wine's PowerShell stub (the dialog trigger)
Wine's `powershell.exe` is a stub that exits `0` without executing anything. `sudo-prompt` elevates via `powershell Start-Process -Verb runAs` and then polls for temp files the elevated helper was supposed to create. They never appear, so it reports `Error: User did not grant permission.` Native Access maps this to:

> **"Please grant permission to Native Access to install dependencies. Please restart Native Access and try again."**

The user is blamed for declining a UAC prompt that never existed. This is structural: **Native Access can never install or start the daemon by itself under Wine — the daemon must already be installed and running before launch.**

### 2. Missing `install.json`
If the silent daemon install (`/S`) does not complete properly, `install.json` is missing. Native Access then concludes "dependencies are not installed" and retries the (always failing) elevated reinstall on every launch.

### 3. Zombie daemon port squatting
If a previous Wine session dies abruptly, an orphaned `NTKDaemon.exe` may keep ports `5146`/`5563`/`7865` bound. It accepts TCP connections but never answers ZeroMQ requests, and any new daemon instance dies with:
`Exception thrown instantiating the Daemon: Address in use`

### 4. Cold-boot service start timeout
On a fresh `wineserver` boot, the **first** start of `NTKDaemonService` can die silently: Wine's service-control handshake between the daemon and `services.exe` times out (`failed to create control pipe`, `ERROR_SEM_TIMEOUT`) while `services.exe` is still initializing. `daemon.log` stops right after `Running Daemon as an Service` and `sc query` reports `STOPPED` with exit code `1077`. A retry a minute or two later (warm server) succeeds within seconds — but by then Native Access has already given up and shown the dialog. The likelihood increases with boot noise (other services crash-looping, many stale Wine sessions on the machine).

---

## Requirements for Running Native Access

1. **`NTKDaemonService` registered and RUNNING** before `Native Access.exe` starts.
2. **Valid `install.json`** at `C:\Users\Public\Documents\Native Instruments\NTK\install.json`:
   ```json
   {"path":"C:\\Program Files\\Common Files\\Native Instruments\\NTK\\NTKDaemon.exe"}
   ```
3. **No stale `NTKDaemon.exe`** processes holding ports `5146`/`5563`/`7865`.

Known-good winetrick set for the prefix (Electron app): `corefonts webview2 vcrun2015 tahoma nocrashdialog`.

---

## Step-by-Step Setup Guide

Replace `/path/to/prefix` with your actual prefix path.

### Step 1: Clean up zombie processes
```bash
pkill -f NTKDaemon.exe
ss -tln | grep -E ':(5146|5563|7865)'   # should print nothing afterwards
```

### Step 2: Install the NTK Daemon silently
Run the installer bundled inside the Native Access installation (adjust version):
```bash
WINEPREFIX=/path/to/prefix wine \
  "C:/Program Files/Native Instruments/Native Access/resources/daemon/win/NTKDaemon 1.31.1 Setup PC.exe" /S
```
Verify:
```bash
WINEPREFIX=/path/to/prefix wine sc query NTKDaemonService     # service exists
cat "/path/to/prefix/drive_c/users/Public/Documents/Native Instruments/NTK/install.json"
```

### Step 3: Create a launcher batch script
Because Native Access (a) cannot elevate to start the service and (b) actively **stops** the service whenever it can't reach it, always launch through a wrapper that guarantees the daemon is up — including retrying the start, which covers the cold-boot timeout (issue #4):

`C:\launch_native_access.bat`
```bat
@echo off
setlocal
set /a tries=0
:retry
net start NTKDaemonService >nul 2>&1
set /a waits=0
:waitrun
sc query NTKDaemonService | find "RUNNING" >nul && goto running
ping -n 3 127.0.0.1 >nul
set /a waits+=1
if %waits% lss 25 goto waitrun
set /a tries+=1
if %tries% lss 4 goto retry
:running
ping -n 4 127.0.0.1 >nul
start "" "C:\Program Files\Native Instruments\Native Access\Native Access.exe" %*
endlocal
```

Notes:
* `%*` forwards arguments so `nativeaccess://` deep-links (used by the login flow) still reach the app.
* `ping` is used as a sleep because Wine's `timeout.exe` does not work in plain batch contexts.
* Put the file inside the prefix (`drive_c\`), since `C:\` maps there.

### Step 4: Launch through the wrapper
Always start Native Access via the batch file instead of the exe directly:
```bash
WINEPREFIX=/path/to/prefix wine cmd /c "C:\launch_native_access.bat"
```
If you use a launcher (desktop entry, Bottles, Lutris, mozapp, …), point its *executable* at `C:\launch_native_access.bat`, keep argument forwarding, and keep the `nativeaccess` URI scheme registered to it.

---

## Troubleshooting & Maintenance

| Diagnostic step | Command / file path | Purpose |
|---|---|---|
| Native Access log | `C:\Users\Public\Documents\Native Instruments\Logs\Native Access\native-access.log` | Which startup step failed (ZMQ timeout, `install.json` ENOENT, `sudo-prompt` permission error). |
| Daemon log | `C:\Users\Public\Documents\Native Instruments\Logs\NTK\daemon.log` | `Address in use` → stale daemon; stops after `Running Daemon as an Service` → cold-boot timeout; `DaemonServer Started` → healthy. |
| Service status | `WINEPREFIX=/path/to/prefix wine sc query NTKDaemonService` | Want `STATE: 4 RUNNING`. Exit `1077` after a start attempt = cold-boot failure, just retry. |
| Stale daemons | `ps aux \| grep NTKDaemon.exe` | Kill orphans holding the ZMQ ports. |
| Port check | `ss -tln \| grep -E ':(5146\|5563\|7865)'` | Exactly one listening daemon should own these. |
| Reset a wedged prefix | `WINEPREFIX=/path/to/prefix wineserver -k` | Kills all Wine processes in the prefix; next launch is a clean boot. |

> **After a Native Access update:** if the required daemon version changed, Native Access will show the permission dialog again. Re-run the newly bundled `NTKDaemon <version> Setup PC.exe /S` manually (Step 2).

> **Housekeeping:** stale Wine sessions from deleted prefixes (old `wineserver`/`services.exe` processes) waste resources and can hold shared ports (e.g. Bonjour/mDNS on UDP 5353, which other software like PACE needs). Reboot or kill them occasionally.
