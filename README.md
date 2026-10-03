# Task 1: Scan Your Local Network for Open Ports

Elevate Labs Cyber Security Internship, Task 1.

## Objective
Discover open ports on devices in my local network using Nmap, identify the services behind them, and assess the security risks. Wireshark was used to observe how a SYN scan looks at packet level.

## Environment
- OS: Windows 11
- Tools: Nmap 7.991 (with Npcap 1.88), Wireshark 4.6.9
- Network: `10.41.53.0/24` (my own Wi-Fi network; scanned with permission as the owner)

## What I did
1. Found my IP range with `ipconfig`: IPv4 `10.41.53.1`, mask `255.255.255.0`, so the range is `10.41.53.0/24`.
2. Ran a TCP SYN scan on the whole range:
   ```
   nmap -sS 10.41.53.0/24 -oN scans/syn_scan.txt
   ```
3. Ran service/version detection on the open ports found:
   ```
   nmap -sV -p 80,135,139,445,1801,2103,2105,2107,5432 10.41.53.1 -oN scans/service_scan.txt
   ```
4. Captured a small scan in Wireshark (loopback adapter) to see the packets:
   ```
   nmap -sS -p 80,81 10.41.53.1
   ```

## Results
256 addresses scanned, **1 host up** (`10.41.53.1`, my own PC). Other addresses, including the gateway, did not respond to the scan. 991 of the top 1000 ports were closed.

| Port | Service | Notes |
|------|---------|-------|
| 80 | HTTP | Microsoft IIS 10.0 (web server) |
| 135 | MSRPC | Windows RPC |
| 139 | NetBIOS-SSN | Legacy Windows file/printer sharing |
| 445 | Microsoft-DS (SMB) | Windows file sharing (not fully verified by `-sV`) |
| 1801 | MSMQ | Microsoft Message Queuing (unverified) |
| 2103, 2105, 2107 | MSMQ-related RPC ports | Nmap labels (zephyr-clt, eklogin, msmq-mgmt) are guesses from port numbers |
| 5432 | PostgreSQL | Database server |

Ports marked unverified returned no banner, so their names are based on Nmap's port lookup table.

## Wireshark observation
Filter used: `tcp.port == 80 || tcp.port == 81`

- **Port 80 (open):** `SYN` → `SYN, ACK` → `RST`. Nmap never completes the handshake (half-open scan).
- **Port 81 (closed):** `SYN` → `RST, ACK`.

## Security risks
- **445 / 139 (SMB, NetBIOS):** history of serious exploits (e.g. EternalBlue); risky on shared networks if unpatched.
- **5432 (PostgreSQL):** a database should not be reachable from the network unless needed.
- **1801, 2103, 2105, 2107 (MSMQ):** optional Windows feature; unused services add attack surface.
- **135 (RPC):** required by Windows but must never be exposed to the internet.
- **80 (HTTP):** unencrypted traffic; IIS should be patched and only run if needed.

## How to reduce exposure
- Disable unused Windows features (IIS, MSMQ) via `optionalfeatures`.
- Turn off file sharing / SMBv1 if not needed and keep Windows updated.
- Bind PostgreSQL to `localhost` (`listen_addresses` in `postgresql.conf`).
- Use the Windows Firewall to block inbound connections to ports that don't need to be reachable.

## Files
- `scans/syn_scan.txt`: SYN scan of the full range
- `scans/service_scan.txt`: service/version detection
- `scans/syn_capture.pcapng`: Wireshark capture
- `screenshots/`: terminal and Wireshark screenshots
