# HotspotGate

A small, readable alternative to MyHotspot-style software: turn a Windows PC into a **voucher-based Wi-Fi hotspot**.
Guests join the Wi-Fi, get sent to a login page, enter a one-time access code, and are online for the time the
code is worth. When time runs out they are cut off again.

> **Status:** the portal, database, admin console and DNS logic are covered by automated tests (`pytest`).
> The Windows-only parts (the PowerShell scripts, the WinDivert enforcer, and the Windows exe/installer build)
> were written but **not run on a real Windows machine** by the author; the frozen-app bundling was verified
> only with a Linux PyInstaller build. Test with your own phone first, and read "Known limits" below.

## How it works

```
 phone ──Wi-Fi──► Windows Mobile Hotspot (192.168.137.1, shares your internet via ICS)
                         │
        ┌────────────────┴──────────────────────────────┐
        │ HotspotGate (one Python process, run as admin)│
        │                                               │
        │  enforcer.py  WinDivert packet rules          │
        │  portal.py    login page + admin console      │
        │  db.py        vouchers & sessions (SQLite)    │
        └───────────────────────────────────────────────┘
```

1. **Windows provides the hotspot.** `scripts/hotspot.ps1` switches on the built-in Mobile Hotspot (it handles
   Wi-Fi, DHCP, DNS proxy and internet sharing). No custom driver is written.
2. **WinDivert enforces access.** Through the signed WinDivert driver (installed by the `pydivert` package)
   the app applies the rules below to every guest.
3. **The portal authorises.** A guest enters a code; the session is stored; within about a second that guest's
   IP is exempted from the drop rule and traffic flows at full speed.

### Enforcement rules

| # | Traffic | Action |
|---|---------|--------|
| R1 | DNS query to the gateway from a **not-yet-authorised** client | Answer locally with the portal's IP (5 s TTL), so any plain-HTTP page or OS connectivity check lands on the login page |
| R2 | DNS query from an **authorised** client | Pass through to Windows' normal DNS |
| R3 | Any forwarded packet from an **unauthorised** client | Dropped (no internet, no outside DNS, no DNS-over-HTTPS) |
| R4 | Any forwarded packet from an **authorised** client | Not captured at all, so no speed penalty |
| R5 | Traffic to the portal (gateway, TCP 80) | Untouched; served by the web app |

The packet filter is rebuilt whenever the authorised set changes. The new filter is opened **before** the old one
is closed, so there is no gap where an unauthorised guest could slip through.

## Install as a normal Windows app (recommended)

You get a real installer, `HotspotGate-Setup.exe`, built from this project. Two ways to produce it:

**A. On GitHub (no tools needed).** Create a private GitHub repository from this folder, open the
*Actions* tab, run **Build Windows installer**, and download `HotspotGate-Setup` from the finished run.
The workflow runs the tests, builds the exe with PyInstaller, and wraps it with Inno Setup.

**B. On your own Windows PC.**
```powershell
winget install JRSoftware.InnoSetup          # once
powershell -ExecutionPolicy Bypass -File .\scripts\build.ps1
# -> installer\Output\HotspotGate-Setup.exe   (and dist\HotspotGate\HotspotGate.exe)
```

Run `HotspotGate-Setup.exe` (it asks for Administrator rights). It installs to *Program Files*, adds a Start Menu
entry (plus an optional desktop shortcut and a "Reset admin password" shortcut), and creates the firewall rule
for you. Launch **HotspotGate**: on first run it generates your passwords (**shown once in the console**),
starts the hotspot, and opens the admin page. Closing the window stops the hotspot. Your config and code
database live in `C:\ProgramData\HotspotGate` and are kept when you uninstall.

The installer is unsigned, so Windows SmartScreen will warn you. That is expected for a self-built app; code
signing requires a paid certificate.

## Run from source (development)

Requirements: Windows 10/11 with a Wi-Fi adapter that supports Mobile Hotspot, an internet connection to share,
Python 3.10+, and Administrator rights.

```powershell
# 1. one-time, from an elevated PowerShell in this folder
powershell -ExecutionPolicy Bypass -File .\scripts\setup.ps1

# 2. every time you want to run the hotspot
.\scripts\run.ps1
```
(The installed app does all of this itself; use these scripts only if you are working on the code.)

`setup.ps1` creates a virtual environment, installs dependencies, generates `config.json` with random secrets
(**it prints the admin password and Wi-Fi password once**), and adds a firewall rule that lets guests reach only
the portal port on this PC.

Then open `http://192.168.137.1/admin` **on the host PC** (the admin console refuses every other device),
sign in, and create access codes. Guests join the Wi-Fi using the Wi-Fi password, open any website, and enter
a code.

Try the portal without touching the network or any driver: `python -m app.main --dry-run --bind 127.0.0.1`
(set `portal_port` in `config.json` to something above 1024 first).

`.\scripts\teardown.ps1` removes the firewall rule and turns the hotspot off.

## Files

| Path | Purpose |
|------|---------|
| `app/main.py` | Starts enforcer, then hotspot, portal and the one-second expiry loop |
| `app/hotspotctl.py` | Starts/stops Mobile Hotspot through the bundled `hotspot.ps1` |
| `run_app.py` | PyInstaller entry point for `HotspotGate.exe` |
| `installer/HotspotGate.iss` | Inno Setup script for `HotspotGate-Setup.exe` |
| `.github/workflows/build.yml` | Builds the installer on a Windows runner |
| `app/enforcer.py` | WinDivert rules R1-R4 |
| `app/portal.py` | Guest login/status pages, admin console, throttling, captive-portal probe handling |
| `app/db.py` | Vouchers, sessions, expiry |
| `app/dnsforge.py` | Builds the forged DNS answer |
| `scripts/*.ps1` | `build` (exe + installer), `hotspot` (Mobile Hotspot control), dev `setup`/`run`/`teardown` |
| `tests/` | `python -m pytest tests` |

## Known limits (please read)

- **Fail-open on a hard crash.** The app switches the hotspot off when it exits normally, on error, or when you
  close its console window. If the process is *killed* (power loss, Task Manager), the packet filter disappears
  while the hotspot may still be on. After any unclean exit, turn off *Mobile hotspot* in Windows Settings
  (or run `scripts\hotspot.ps1 -Action stop`).
- **Spoofing.** Authorisation is by IP address. A guest who copies an authorised device's IP and MAC address can
  piggy-back on that session. The MAC is recorded (visible in admin) but not enforced. Windows' hotspot also
  always requires a Wi-Fi password, so guests need two things: the Wi-Fi password and a code.
- **HTTPS-only first visits.** Because unauthorised DNS points at a plain-HTTP portal, modern phones usually pop
  up the login page automatically. If a browser first opens an https site, it shows a certificate error until
  the guest signs in. This is inherent to captive portals.
- **IPv4 only.** Windows internet sharing is IPv4, which this design assumes.
- **Port 80 must be free** (stop IIS, Skype-era web servers, etc.), or change `portal_port` and the firewall rule.
- **Antivirus** products sometimes flag WinDivert-based tools. WinDivert is a legitimate, signed packet-filter
  driver, but review the source and the download before allowing it.
- **Not included (on purpose):** per-user URL logging (a privacy and legal burden), PayPal/social-login billing,
  data-volume caps, bandwidth shaping. The code is small enough to add them if you need them.

## Responsible use

Only run this on a network you own or are authorised to operate. Tell guests that a gateway controls access, and
check the legal and privacy rules in your country before you log anything beyond what this project stores
(code, device IP, MAC address, start and end time).
