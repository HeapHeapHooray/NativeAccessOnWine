# Native Access & NTK Daemon under Wine Guide

This guide explains how Native Access and its background service (**NTK Daemon**) interact, why Native Access fails under standard Wine setups, and how to properly configure and run them.

---

## Overview & Architecture

**Native Access** is an Electron-based application from Native Instruments used for registering, downloading, and updating software and sound libraries.

Native Access relies entirely on a background service:
* **NTK Daemon**: (`NTKDaemonService` registered at `C:\Program Files\Common Files\Native Instruments\NTK\NTKDaemon.exe`)
* **Communication Protocol**: Native Access communicates with the daemon via **ZeroMQ** sockets over TCP:
  * **Send / Request**: `tcp://127.0.0.1:5146`
  * **Subscribe / Events**: `tcp://127.0.0.1:5563`
  * **Auxiliary Sockets**: Port `7865`

---

## How Native Access Startup Works

On launch, Native Access follows this sequence:

1. **Ping Daemon**: Sends a `daemonVersionRequest` over ZeroMQ and compares the response with the bundled version requirement (e.g., `1.31.1.0`).
2. **Stop Service on Failure**: If the daemon does not respond over ZeroMQ, Native Access issues `sc stop NTKDaemonService`.
3. **Version & Installation Check**: Checks for `C:\Users\Public\Documents\Native Instruments\NTK\install.json`.
   * If missing or version mismatched, it attempts to reinstall the daemon elevated using the bundled installer: `resources\daemon\win\NTKDaemon <version> Setup PC.exe /S` via the `@vscode/sudo-prompt` npm module.
4. **Service Elevation**: If installed but stopped, it attempts to restart the service using PowerShell:
   ```cmd
   powershell -command "start-process cmd -verb runas -argumentlist '/c sc start NTKDaemonService'"
   ```

---

## Why Native Access Fails under Wine

Under standard Windows, steps 3 & 4 trigger a UAC elevation prompt. Under Wine, this mechanism fails due to three primary issues:

### 1. Wine PowerShell Stub Limitations
Wine's `powershell.exe` is a lightweight stub that exits with status `0` without executing `Start-Process -Verb runAs`. Because `@vscode/sudo-prompt` relies on PowerShell creating temporary status files in the elevated context, the missing files trigger `Error: User did not grant permission.`
Native Access catches this error and presents the user with:
> **"Please grant permission to Native Access to install dependencies. Please restart Native Access and try again."**

### 2. Missing `install.json`
Silent installation (`/S`) of `NTKDaemon Setup PC.exe` might not automatically generate `C:\Users\Public\Documents\Native Instruments\NTK\install.json`. Without this file, Native Access concludes that dependencies are missing and continuously attempts (and fails) auto-elevation on every launch.

### 3. Zombie Daemon Port Squatting
If a previous Wine session dies abruptly, an orphan `NTKDaemon.exe` process may remain alive in the background holding ports `5146`, `5563`, and `7865`. Any newly spawned daemon process will immediately crash with:
`Exception thrown instantiating the Daemon: Address in use`

---

## Requirements for Running Native Access

To run Native Access successfully under Wine, the following conditions **must** be met:

1. **Active `NTKDaemonService` Service**: The NTK Daemon service must be registered in the Wine prefix and running **before** `Native Access.exe` starts.
2. **Valid `install.json`**: The metadata file must exist at `C:\Users\Public\Documents\Native Instruments\NTK\install.json` with the correct path to `NTKDaemon.exe`:
   ```json
   {"path":"C:\\Program Files\\Common Files\\Native Instruments\\NTK\\NTKDaemon.exe"}
   ```
3. **Clean Sockets**: No stale `NTKDaemon.exe` processes squatting on ZeroMQ ports `5146`/`5563`/`7865`.

---

## Step-by-Step Setup & Running Guide

### Step 1: Clean Up Zombie Processes
Kill any lingering daemon processes from previous sessions:
```bash
pkill -f NTKDaemon
```

### Step 2: Install/Reinstall the NTK Daemon Silently
Run the bundled daemon installer in your Wine prefix:
```bash
WINEPREFIX=~/.mozapp/cheapwine/.cheapwine wine "C:/Program Files/Native Instruments/Native Access/resources/daemon/win/NTKDaemon 1.31.1 Setup PC.exe" /S
```
*Verify that `C:\Users\Public\Documents\Native Instruments\NTK\install.json` exists with valid JSON contents.*

### Step 3: Create a Launcher Batch Script
Because Native Access stops the service if ZeroMQ connection fails and cannot elevate to restart it, use a launcher wrapper script (`C:\launch_nativeaccess.bat`) to ensure `NTKDaemonService` is always started prior to opening Native Access:

```bat
@echo off
net start NTKDaemonService >nul 2>&1
start "" "C:\Program Files\Native Instruments\Native Access\Native Access.exe" %*
```

> **Note**: `%*` ensures URL scheme deep-links (e.g. `nativeaccess://` used during login flow) are forwarded properly.

### Step 4: Configure App Launcher / `distillery.json`
If using `mozapp` or a launcher system, target `C:\launch_nativeaccess.bat` instead of `Native Access.exe` directly:

```json
"Native Access": {
  "exe": "C:\\launch_nativeaccess.bat",
  "args": [],
  "env": {},
  "uri_schemes": ["nativeaccess"]
}
```

---

## Troubleshooting & Maintenance

| Diagnostic Step | Command / File Path | Purpose |
|---|---|---|
| **Check Native Access Logs** | `C:\Users\Public\Documents\Native Instruments\Logs\Native Access\native-access.log` | Check for ZeroMQ timeouts or `sudo-prompt` errors. |
| **Check Daemon Logs** | `C:\Users\Public\Documents\Native Instruments\Logs\NTK\daemon.log` | Look for `Address in use` or `DaemonServer Started`. |
| **Check Service Status** | `wine sc query NTKDaemonService` | Verify status is `STATE: 4 RUNNING`. |
| **Kill Stale Daemon** | `ps aux \| grep NTKDaemon` | Identify and terminate orphan background processes. |

> **Note on Native Access Updates**:
> If a Native Access update updates the required daemon version, Native Access will fail to start and ask for permissions again. Manually run the newly bundled `NTKDaemon Setup PC.exe /S` in the Wine prefix to update the daemon.
