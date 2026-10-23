# Network Printers

Printers are the forgotten computers of every office: full network stacks, web admin panels, stored credentials, document memory, and historically wormable. Nobody patches them and everybody trusts them. The aim is credential theft, document capture, and a persistent, unmonitored network foothold.

## What it is and why it is a target (the aim)

- **Stored credentials**: printers hold scan-to-email SMTP creds, scan-to-folder SMB/FTP creds, and LDAP/AD bind credentials for the address book. Aim: extract these, often domain credentials, from the printer.
- **Document capture**: print/scan/fax jobs pass through and may be stored on disk. Aim: read spooled or stored documents.
- **Network foothold**: the printer is a Linux-ish host nobody monitors. Aim: persistence and a pivot, including NTLM-relay capture.
- **Admin takeover**: the web panel often has default/no credentials. Aim: full control of the device.

## Architecture (the four sides)

- **Device**: an embedded controller (often VxWorks or Linux) with a web server, printing languages (PostScript, PJL, PCL), and a hard drive/flash that may retain jobs.
- **Network**: many services (see below); printers are chatty and expose a lot.
- **Cloud**: for consumer/"smart" printers, a vendor cloud for remote print and supplies.
- **App/driver**: desktop drivers and mobile print apps.

## Step-by-step how-to

### 1. Network recon (aim: the exposed services)

```
nmap -sS -sV -p- PRINTER_IP
# classic printer ports:
#  9100  raw printing (JetDirect/PJL/PostScript) - the main control channel
#  515   LPD        631  IPP/CUPS
#  80/443 web admin  161 SNMP (community often "public", leaks lots)
#  21 FTP  23 telnet  445 SMB (scan-to-folder)
```

### 2. Web admin and SNMP (aim: config and stored creds)

```
# web admin: test default/blank credentials; the config pages often show SMTP/SMB/LDAP
# server settings and sometimes the stored credentials (or let you redirect them).
# SNMP: pull device info and config:
snmpwalk -v2c -c public PRINTER_IP
```
Aim: read or redirect stored scan-to-email/SMB/LDAP credentials. A classic trick: change the LDAP/SMB server to one you control so the printer authenticates to you, capturing the (often domain) credentials (a "pass-back" attack).

### 3. Printing-language abuse (aim: filesystem and memory access via PJL/PostScript)

The printer languages are powerful. PJL and PostScript can read/write the printer's filesystem, read stored jobs, and change settings:

```
# PJL filesystem access over port 9100 (printer-dependent):
@PJL FSDIR NAME="0:\"          # list files
@PJL FSUPLOAD NAME="0:\..\..\etc\passwd"   # read files on vulnerable models
# PRET (Printer Exploitation Toolkit) automates PJL/PostScript/PCL attacks:
#  file read/write, job capture, NVRAM access, and lockouts.
```
Aim: read stored documents and config, and on vulnerable models reach the OS filesystem.

### 4. The pass-back / NTLM-relay capture (aim: domain credentials)

```
# set the printer's scan-to-SMB or LDAP target to your host, trigger a scan/address-book
# lookup, and capture the NTLM authentication with Responder/impacket, then relay or crack.
```
This turns a printer into a domain-credential harvester, one of the highest-impact printer findings.

### 5. Cloud/app and firmware (aim: fleet and persistence)

For cloud printers, proxy the app and test the cloud API (cross-account print/supplies, see `../../web/IDOR-and-BOLA.md`). Extract firmware (see `../Firmware-Analysis-and-Extraction.md`) for stored secrets and the update path; unsigned firmware is persistence.

## Beyond the basics

- **The pass-back attack is the money finding**: redirecting the printer's stored scan/LDAP/SMB destination to capture credentials (often privileged AD service accounts) is a reliable path from "a printer" to "domain foothold." Always check what credentials the printer stores and whether you can change the destination.
- **PRET and the printing languages**: PJL/PostScript give file access, job retrieval, NVRAM manipulation, and physical-display/lockout control on many models. Learning PRET covers a broad range of devices and demonstrates document theft and DoS.
- **Stored-job retrieval**: printers with disks retain print/scan jobs; retrieving them leaks documents. Secure-print PINs are sometimes bypassable.
- **Printer worms and botnets**: printers have been wormed across the internet (to prove the point) and used in botnets; an unauthenticated 9100 plus a firmware bug is remote compromise.
- **SNMP over-sharing**: default community strings leak configuration, serials, and sometimes credentials; always SNMP-walk.
- **Fax and cellular surfaces**: some printers have fax/cellular modems; "faxploit"-class bugs reached the network via a malicious fax. Note non-IP entry points.

## Defensive framing / what to watch for

- The headline risks are credential pass-back, document capture, and an unmonitored foothold; defenses are strong admin auth, restricting/disabling raw printing and unused protocols, protecting stored credentials, SNMPv3, and segmenting/patching printers.
- Confirm you actually read stored creds/documents or captured an auth on your own device before claiming it.
- Frame impact by credential theft (often domain) and data exposure.
- To defend: set admin passwords, disable telnet/FTP/raw-9100 if unused, use SNMPv3, restrict scan destinations, enable stored-job encryption and secure print, keep firmware updated, and put printers on a segmented, monitored VLAN.

## Checklist

- Scan the printer; enumerate 9100/515/631/web/SNMP/FTP/telnet/SMB.
- Test web admin default creds; SNMP-walk for config and stored secrets.
- Use PRET/PJL/PostScript to test filesystem and stored-job access.
- Attempt the pass-back / NTLM-relay capture of scan/LDAP/SMB credentials (your own lab domain).
- Test the cloud API for cross-account access; check firmware signing.
- Frame by credential theft and document exposure; test only your own printer.
