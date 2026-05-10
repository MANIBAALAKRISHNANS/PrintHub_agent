# PrintHub Agent

The PrintHub Agent is the software that runs on every printer workstation in the hospital. It connects to the PrintHub backend server over the network, receives print jobs, and sends them to the USB printer plugged into that PC — automatically, silently, in the background.

**You install this once per PC. After that it starts automatically every time Windows logs in.**

---

## How It Works

```
  PrintHub Backend Server (192.168.1.14:8000)
              │
              │  WebSocket (real-time)
              │  HTTP poll (fallback every 30s)
              ▼
    ┌────────────────────┐
    │   PrintHub Agent   │  ← runs silently on this PC
    │   (this software)  │
    └────────────────────┘
              │
              │  win32print / CUPS
              ▼
    USB Printer (plugged into this PC)
```

When a staff member sends a print job from the dashboard, the backend pushes it to the correct agent over WebSocket. The agent prints it instantly and reports success back.

---

## Files in This Repo

| File | What it does |
|---|---|
| `agent.py` | Main agent — WebSocket listener + print job processor |
| `agent_config.py` | Reads and writes the local config file |
| `agent_setup.py` | One-time registration with the server using an activation code |
| `agent_service.py` | Legacy Windows Service wrapper (kept for reference) |
| `agent_macos.py` | macOS CUPS printing helpers |
| `requirements.txt` | Python dependencies |
| `install_agent.bat` | **Windows installer** — right-click → Run as administrator |
| `install_agent.sh` | **Mac / Linux installer** — run with `bash install_agent.sh` |
| `debug_wmi.py` | Diagnostic script to list printers visible to Windows |

---

## Before You Start

You need:
1. **The PrintHub backend running** on the server PC
2. **An activation code** — generated from the dashboard (Dashboard → Activation Codes → Generate Code)
   - Codes expire in 10 minutes. Generate one fresh right before installing.
3. **Python 3.11 or newer** installed on the printer PC (the installer will check)

---

## Windows — Install Step by Step

### Step 1 — Download the agent files

Download or copy this entire folder to the printer PC.  
You can use a USB stick, a network share, or download the zip from GitHub.

The folder must contain these files:
```
PrintHub_Agent\
├── agent.py
├── agent_config.py
├── agent_setup.py
├── agent_service.py
├── requirements.txt
└── install_agent.bat   ← the installer
```

### Step 2 — Right-click the installer → Run as administrator

In File Explorer, find `install_agent.bat`.

**Right-click** on it → click **"Run as administrator"**.

> Do NOT just double-click it. It MUST say "Run as administrator". If Windows asks "Do you want to allow this app to make changes to your device?" click **Yes**.

A black window will open and stay open throughout the installation.

### Step 3 — Answer the questions (first time only)

The installer asks 4 questions. Type your answer and press **Enter** after each:

```
Server IP address (e.g. 192.168.1.14):
```
→ Type the IP address of the server running the PrintHub backend.  
  Find it by running `ipconfig` on the server and looking for the IPv4 Address.

```
Server port (press Enter for 8000):
```
→ Just press **Enter** — 8000 is the default.

```
Use HTTPS? (y/N):
```
→ Just press **Enter** (choose No unless your IT team set up HTTPS).

```
Enter the 8-character activation code:
```
→ Type the code you generated from the dashboard. Example: `6F166AC4`

> **Already installed before?** If `agent_config.json` already exists, the installer skips the questions and goes straight to creating the scheduled task. No setup needed again.

### Step 4 — Watch it complete

You will see these lines appear:

```
[OK] Running as Administrator
[OK] Python 3.11.x found
[STEP 1] Preparing C:\PrintHubAgent...
[OK] Files copied
[STEP 2] Creating virtual environment...
[OK] Virtual environment ready
[STEP 3] Installing dependencies...
[OK] Dependencies installed.
[STEP 4] Checking configuration...
[OK] Configuration saved.
[STEP 5] Creating Task Scheduler task...
[OK] Task Scheduler task created (auto-starts at every login).
[STEP 6] Starting agent now...
[OK] Agent started in background.

============================================================
  SUCCESS! PrintHub Agent is installed and running.
  It will start automatically every time you log in.
============================================================
```

Installation is complete. Press **Enter** to close the window.

### Step 5 — Verify the agent is running correctly (Windows)

**Method 1 — Check the dashboard (easiest):**
Open the PrintHub Dashboard → click **Agents** in the left menu.
This PC should appear with a green **Online** badge within 15–30 seconds.

**Method 2 — Check the log in Command Prompt:**

If the installer window is still open (it shows `C:\Windows\System32>`), run:
```cmd
type C:\PrintHubAgent\agent.log
```

**Method 3 — Check the log in PowerShell:**

Press `Win + X` → click **Terminal** or **PowerShell**, then run:
```powershell
Get-Content C:\PrintHubAgent\agent.log -Tail 20
```

**Method 4 — Open the log in Notepad:**
```cmd
notepad C:\PrintHubAgent\agent.log
```

In the log, look for this line — it confirms the agent is fully connected:
```
[WS] Connected to server
```

**Method 5 — Check Task Manager:**
Press `Ctrl + Shift + Esc` → **Details** tab → look for `python.exe`.
If it is there, the agent is running.

---

## Mac — Install Step by Step

### Step 1 — Copy the agent folder to the Mac

Copy this folder to the Mac (USB, AirDrop, or network share).

### Step 2 — Open Terminal

Press `Cmd + Space` → type **Terminal** → press Enter.

### Step 3 — Go to the folder and run the installer

```bash
cd ~/Desktop/PrintHub_Agent
bash install_agent.sh
```

The installer will ask for:
- Server IP address and port
- Whether to use HTTPS
- An 8-character activation code from the dashboard

### Step 4 — Verify the agent is running correctly (Mac)

**Method 1 — Check the dashboard (easiest):**
Open the PrintHub Dashboard → click **Agents** in the left menu.
This Mac should appear with a green **Online** badge within 15–30 seconds.

**Method 2 — Check the log in Terminal:**

Open Terminal (`Cmd + Space` → Terminal → Enter) and run:
```bash
tail -20 ~/Library/Logs/PrintHubAgent/agent.log
```

Look for this line — it confirms the agent is fully connected:
```
[WS] Connected to server
```

**Method 3 — Watch live log output:**
```bash
tail -f ~/Library/Logs/PrintHubAgent/agent.log
```
This keeps scrolling in real time. Press `Ctrl + C` to stop.

**Method 4 — Check if the service is registered:**
```bash
launchctl list | grep printhub
```
If a line appears, the service is loaded and running.

---

## Managing the Agent

### Windows

The agent starts automatically at every Windows login. You normally never need to touch it.

**View the agent log (Command Prompt):**
```cmd
type C:\PrintHubAgent\agent.log
```

**View the agent log (PowerShell):**
```powershell
Get-Content C:\PrintHubAgent\agent.log -Tail 20
```

**Start the agent manually:**
```cmd
C:\PrintHubAgent\venv\Scripts\python.exe C:\PrintHubAgent\agent.py
```

**Stop the agent:**
Open Task Manager → Details tab → find `python.exe` → End Task.

**Check the scheduled task:**
Open Task Scheduler → find `PrintHubAgent` → it runs at every login.

**Reinstall / re-register (if server IP changes):**
```cmd
del C:\PrintHubAgent\agent_config.json
```
Then run `install_agent.bat` again as Administrator.

**Uninstall completely (run Command Prompt as Administrator):**
```cmd
schtasks /delete /tn "PrintHubAgent" /f
rmdir /s /q C:\PrintHubAgent
```

**List printers visible to Windows (diagnostic):**
```cmd
C:\PrintHubAgent\venv\Scripts\python.exe C:\PrintHubAgent\debug_wmi.py
```

---

### Mac

The agent starts automatically at every login via launchd. You normally never need to touch it.

**View the last 20 lines of the log:**
```bash
tail -20 ~/Library/Logs/PrintHubAgent/agent.log
```

**Watch live log output:**
```bash
tail -f ~/Library/Logs/PrintHubAgent/agent.log
```
Press `Ctrl + C` to stop.

**Stop the agent:**
```bash
launchctl unload ~/Library/LaunchAgents/com.printhub.agent.plist
```

**Start the agent:**
```bash
launchctl load ~/Library/LaunchAgents/com.printhub.agent.plist
```

**Uninstall completely:**
```bash
launchctl unload ~/Library/LaunchAgents/com.printhub.agent.plist
rm ~/Library/LaunchAgents/com.printhub.agent.plist
```

---

## Troubleshooting

### The install window closes instantly with no error

**Cause:** The self-elevation failed or you ran it without administrator rights.

**Fix:** Right-click `install_agent.bat` → **Run as administrator**. Do not just double-click.

---

### `[ERROR] Python 3.11+ not found in PATH`

**Cause:** Python is not installed, or was installed without "Add to PATH".

**Fix:**
1. Go to [python.org/downloads](https://www.python.org/downloads/)
2. Download and run the installer
3. On the first screen, **tick "Add Python to PATH"** before clicking Install Now
4. Run `install_agent.bat` again

---

### Agent shows **Offline** in the dashboard

**Cause 1 — Agent not running.** Open PowerShell and run:
```powershell
C:\PrintHubAgent\venv\Scripts\python.exe C:\PrintHubAgent\agent.py
```
When you see `[WS] Connected to server`, it is working.

**Cause 2 — Can't reach the server.** Open a browser on the printer PC and go to:
```
http://SERVER_IP:8000/health
```
If this page doesn't load, port 8000 is blocked by the server's firewall. Ask your IT admin to open it.

**Cause 3 — Wrong server IP saved.** Delete the config and reinstall:
```powershell
Remove-Item C:\PrintHubAgent\agent_config.json
```
Then run `install_agent.bat` again with the correct server IP.

---

### `[ERROR] Setup failed. Check the activation code and server URL.`

**Cause:** The activation code expired (codes expire in 10 minutes) or the server IP is wrong.

**Fix:**
1. Go to Dashboard → **Activation Codes** → Generate Code
2. Use the new code immediately in the installer

---

### Print job stays "Pending" and never prints

Checklist:
1. Dashboard → **Agents** — is this PC showing as **Online**?
2. Dashboard → **Mapping** — does this location have a printer assigned?
3. Is the USB printer physically plugged in and turned on?
4. View the agent log: `notepad C:\PrintHubAgent\agent.log` — look for errors near the bottom.

---

### `install_agent.bat` shows `: was unexpected at this time.`

**Cause:** Old installer version that used `schtasks /rl HIGHEST` which breaks on Windows 11.

**Fix:** Download the latest `install_agent.bat` from this repository. The updated version uses PowerShell's `Register-ScheduledTask` internally and does not have this error.

---

## Requirements

- Windows 10 / 11 or macOS 12+ or Ubuntu 20.04+
- Python 3.11 or newer
- USB printer plugged in and recognised by Windows (visible in Control Panel → Printers)
- Network access to the PrintHub backend server (port 8000)

Python packages installed automatically by the installer:
```
requests>=2.31.0
urllib3>=2.0.0
websocket-client>=1.6.0
pywin32>=306          (Windows only)
wmi>=1.5.1            (Windows only)
```

---

## Part of PrintHub

This agent is one component of the PrintHub hospital print management system.

- **Backend + Frontend (server):** https://github.com/MANIBAALAKRISHNANS/print_centre
- **Agent (this repo):** https://github.com/MANIBAALAKRISHNANS/PrintHub_agent

*PrintHub — Savetha Hospital IT Engineering*
