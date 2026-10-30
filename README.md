# NetScanner

Port scanner + network device discovery with a local REST API and dashboard.
Pure Windows PowerShell 5.1 — nothing to install.

## Desktop app

1. Unzip `NetScanner.zip` anywhere. If Windows blocks it: right-click the zip → Properties → **Unblock** before unzipping.
2. Double-click **`NetScanner.bat`**.
3. Click **Scan network** (or enter a subnet such as `192.168.1.0/24` first).

The **Map** view draws your router in the centre with every device around it, with an icon and colour per device type.
Click a device to see its vendor, MAC, OS guess and open ports, or to run a full 1-1024 port scan on it.
**List** shows the same devices as a table. **Save** exports the scan as JSON (re-open it later with **Open**) or CSV.

## Files

| File | Purpose |
|---|---|
| `NetScanner.bat` | Double-click launcher for the desktop app |
| `NetScannerApp.ps1` | Desktop app (network map, device details, list, save/open) |
| `NetScanner.psm1` | Core: ping sweep, ARP lookup, TCP port scan, vendor lookup, device-type detection |
| `portscan.ps1` | Command-line tool |
| `api.ps1` | REST API + dashboard on `http://localhost:8080/` |
| `dashboard.html` | Web UI served by the API |

> Only scan networks you own or have permission to scan. Scans are capped at 1,024 addresses
> unless you pass `-AllowLarge`; big subnets (campus, office, hotel Wi-Fi) are usually shared.

## Command line

```powershell
powershell -ExecutionPolicy Bypass -File .\portscan.ps1 192.168.1.10                  # ports 1-1024
powershell -ExecutionPolicy Bypass -File .\portscan.ps1 192.168.1.10 -Ports 22,80,8000-8100
powershell -ExecutionPolicy Bypass -File .\portscan.ps1 -Discover                      # all devices on current network
powershell -ExecutionPolicy Bypass -File .\portscan.ps1 -Discover -Subnet 192.168.1.0/24 -Json
powershell -ExecutionPolicy Bypass -File .\portscan.ps1 -UpdateOui                     # download IEEE vendor list (~6 MB, once)
```

## API

```powershell
powershell -ExecutionPolicy Bypass -File .\api.ps1 -ScanOnStart            # optional: -Port 9000 -Subnet 192.168.1.0/24
```

Open http://localhost:8080/ for the dashboard, or call the endpoints:

| Method | Path | Description |
|---|---|---|
| GET | `/api` | Endpoint list |
| GET | `/api/networks` | Local interfaces, subnets, gateways |
| POST | `/api/scan` | Start a device scan. Query or JSON body: `subnet`, `ports` (`22,80,1-1024` or `common`), `skipPorts` |
| GET | `/api/scan` | Status: `idle` / `running` (with progress) / `completed` / `failed` |
| GET | `/api/devices` | All devices from the last scan; `?type=printer` filters by device type |
| GET | `/api/devices/{ip}` | One device |
| GET | `/api/ports/{ip}` | Port-scan any host right now; `?ports=1-1024` (default) |

```powershell
Invoke-RestMethod -Method Post 'http://localhost:8080/api/scan?subnet=192.168.1.0/24'
Invoke-RestMethod http://localhost:8080/api/scan
(Invoke-RestMethod http://localhost:8080/api/devices).devices | Format-Table Ip, HostName, DeviceType, Vendor
```

The server only listens on `localhost`. A second `POST /api/scan` while one is running returns `409`;
invalid subnets/ports and oversized ranges return `400`.

## How devices are identified

1. **Discovery** — pings every address in the subnet, then reads the ARP table (catches devices that ignore ping).
2. **Names** — reverse DNS / NetBIOS lookups.
3. **Vendor** — MAC prefix lookup (built-in short list; run `-UpdateOui` for the full IEEE database).
   Randomized "private" MACs (phones/laptops) are flagged.
4. **Ports** — 36 well-known ports by default (printer, AirPlay, Chromecast, RTSP, SMB, RDP, SSH, …).
5. **Type** — rules combine the above, e.g. 9100/631 → printer, 62078 → iPhone/iPad, 8008/8009 → Chromecast,
   554 → camera, 135+445 or 3389 → Windows PC, gateway IP → router. Each device includes a `TypeReason`.
   Ping TTL gives a rough OS family guess (64 = Linux/macOS/iOS/Android, 128 = Windows, 255 = network gear).

Device typing is a best guess — firewalled devices with no open ports and a private MAC can only be
labelled generically.
