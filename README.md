# Windows Server DC — NTP Time Sync Troubleshooting

**Environment:** ADForest.local · Windows Server 2025 DC · Topton N150 Rocky Linux (Suricata TAP)  
**Scope:** Diagnosing and resolving incorrect system time on a standalone/unactivated Windows Server DC  
**Outcome:** Full internal NTP hierarchy established — DC syncing from Rocky Linux Topton via chrony  

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Investigation — w32tm Query](#2-investigation--w32tm-query)
3. [Root Cause Chain](#3-root-cause-chain)
4. [Resolution Steps](#4-resolution-steps)
   - [Step 1 — Repair W32Time Service](#step-1--repair-w32time-service)
   - [Step 2 — Fix DNS Forwarders](#step-2--fix-dns-forwarders)
   - [Step 3 — Correct DC Timezone](#step-3--correct-dc-timezone)
   - [Step 4 — Configure Topton as NTP Server](#step-4--configure-topton-as-ntp-server)
   - [Step 5 — Point DC at Internal NTP Source](#step-5--point-dc-at-internal-ntp-source)
5. [Verification](#5-verification)
6. [Final NTP Hierarchy](#6-final-ntp-hierarchy)
7. [Skills Demonstrated](#7-skills-demonstrated)

---

## 1. Problem Statement

The Windows Server 2025 Domain Controller showed the correct date but incorrect time. The offset was significant enough to affect Splunk log timestamps and would eventually breach the 5-minute Kerberos tolerance, breaking domain authentication.

**Constraints:**
- Server is unactivated (evaluation licence) — activation not required for this fix
- DC has no direct internet access — isolated on internal virtual switch
- NTP must be resolved without external connectivity from the DC itself

---

## 2. Investigation — w32tm Query

Initial diagnostic run on the DC:

```powershell
w32tm /query /status
```

**Output (screenshot 1):**

```
Stratum:          1 (primary reference - syncd by radio clock)
ReferenceId:      0x4C4F434C  (source name: "LOCL")
Last Successful Sync Time: 25/05/2026 5:07:56
Source:           Free-running System Clock
Poll Interval:    6 (64s)
```

> ![w32tm query showing LOCL source](images/01_w32tm_query_locl.jpg)

**Key findings:**
- `Source: Free-running System Clock` — not syncing from any NTP server
- `ReferenceId: LOCL` — Windows fallback when no external source is reachable
- Last sync was the previous day — clock was drifting unchecked

Forced resync attempt:

```powershell
w32tm /resync /force
```

**Result:** `The computer did not resync because no time data was available.`

This confirmed the service could not reach any NTP peer, not just that the sync was stale.

---

## 3. Root Cause Chain

Three separate issues combined to produce the problem:

| # | Root Cause | Effect |
|---|-----------|--------|
| 1 | W32Time service corrupt/unregistered | `net stop w32tm` → "service name is invalid" |
| 2 | DC DNS had no forwarders configured | Could not resolve `pool.ntp.org` — name resolution failed |
| 3 | DC timezone set to Pacific Standard Time (UTC-8) | Displayed time was 8 hours behind BST even when UTC was correct |

Each issue independently would have caused time problems. Together they produced a situation where the service was broken, unreachable externally, and displaying the wrong local time regardless.

---

## 4. Resolution Steps

### Step 1 — Repair W32Time Service

The service registration was corrupt. Standard `net stop`/`net start` returned "service name is invalid":

> ![W32Time service invalid error](images/02_w32time_service_error.jpg)

**Fix — unregister, re-register, start via sc.exe:**

```powershell
w32tm /unregister
w32tm /register
Start-Service -Name W32Time
Get-Service W32Time
```

**Result:** `Status: Running · DisplayName: Windows Time`

> ![W32Time service running](images/03_w32time_running.jpg)

> **Note:** A reboot was required between the unregister/register cycle due to a "marked for deletion" state that persists in the SCM until the next boot.

---

### Step 2 — Fix DNS Forwarders

With the service running, `w32tm /resync /force` still failed. Connectivity test:

```powershell
Test-NetConnection -ComputerName pool.ntp.org -Port 123
```

**Result:** `WARNING: Name resolution of pool.ntp.org failed`

> ![DNS resolution failure](images/04_dns_resolution_failed.jpg)

The DC is authoritative DNS for `ADForest.local` but had no forwarders configured for external name resolution. This also meant domain clients could not resolve internet names.

**Fix — add DNS forwarders:**

```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
Add-DnsServerForwarder -IPAddress 1.1.1.1
```

**Verification:**

```powershell
Resolve-DnsName pool.ntp.org
```

**Result:** Four A records returned — DNS resolution restored.

> ![DNS resolution working](images/05_dns_resolution_working.jpg)

However, `w32tm /resync /force` still failed with "no time data was available." Further testing showed that ping to the resolved IP also returned "General failure" — the DC's virtual NIC has no internet routing. UDP 123 to external NTP servers was unreachable at the network level, not just the firewall level.

**Decision:** Rather than force internet access to the DC (which is architecturally undesirable for a DC anyway), configure the Topton Rocky Linux machine as an internal NTP source. The Topton has internet access via its bridged management interface.

---

### Step 3 — Correct DC Timezone

During investigation, the DC timezone was found to be incorrectly set:

```powershell
Get-TimeZone
```

**Output:**
```
Id                         : Pacific Standard Time
DisplayName                : (UTC-08:00) Pacific Time (US & Canada)
BaseUtcOffset              : -08:00:00
```

> ![DC set to Pacific timezone](images/06_dc_pacific_timezone.jpg)

This was the primary reason displayed time was wrong — the DC was operating as if in California (UTC-8) rather than the UK (UTC+1 BST).

**Fix:**

```powershell
Set-TimeZone -Id "GMT Standard Time"
```

---

### Step 4 — Configure Topton as NTP Server

**On the Topton (Rocky Linux, 192.168.1.217):**

Checked chrony status:

```bash
timedatectl
```

**Output confirmed:**
```
Local time:               Tue 2026-05-26 12:42:01 BST
Universal time:           Tue 2026-05-26 11:42:01 UTC
Time zone:                Europe/London (BST, +0100)
System clock synchronized: yes
NTP service:              active
```

Chrony was already synced and running. Enabled LAN client access by editing `/etc/chrony.conf`:

```bash
sudo nano /etc/chrony.conf
```

Uncommented the existing allow line:

```
# Before:
#allow 192.168.0.0/16

# After:
allow 192.168.0.0/16
```

> ![chrony.conf allow line uncommented](images/07_chrony_conf.jpg)

Applied changes and opened firewall:

```bash
sudo systemctl restart chronyd
sudo firewall-cmd --add-service=ntp --permanent
sudo firewall-cmd --reload
```

Verified chrony was synced and serving:

```bash
chronyc tracking
```

**Output:**
```
Reference ID    : 6FE14EF4 (2607:ff68:14:11::e539:b04a)
Stratum         : 3
Ref time (UTC)  : Tue May 26 11:31:16 2026
System time     : 0.000180183 seconds fast of NTP time
Last offset     : +0.000236521 seconds
```

> ![chronyc tracking output](images/08_chronyc_tracking.jpg)

Topton confirmed as Stratum 3, offset < 0.001 seconds — ready to serve as internal NTP source.

---

### Step 5 — Point DC at Internal NTP Source

**On the DC:**

```powershell
w32tm /config /syncfromflags:manual /manualpeerlist:"192.168.1.217" /reliable:YES /update
w32tm /resync /force
w32tm /query /status
```

---

## 5. Verification

**Final w32tm status on DC:**

```
Stratum:                  4 (secondary reference - syncd by (S)NTP)
ReferenceId:              0xC0A801D9  (source IP: 192.168.1.217)
Last Successful Sync Time: 26/05/2026 12:46:36
Source:                   192.168.1.217
Poll Interval:            6 (64s)
```

> ![Final w32tm status showing 192.168.1.217](images/09_final_sync_status.jpg)

All indicators confirmed:

| Check | Result |
|-------|--------|
| Source | `192.168.1.217` (Topton) ✅ |
| Stratum | `4` — correct position in hierarchy ✅ |
| ReferenceId | `0xC0A801D9` = 192.168.1.217 in hex ✅ |
| Resync command | `The command completed successfully` ✅ |
| Sync time | Current timestamp ✅ |

---

## 6. Final NTP Hierarchy

```
┌─────────────────────────────────────┐
│  Internet NTP Pool (Stratum 2)      │
│  2.rocky.pool.ntp.org               │
└──────────────────┬──────────────────┘
                   │ UDP 123
┌──────────────────▼──────────────────┐
│  Topton N150 — Rocky Linux          │
│  192.168.1.217  (Stratum 3)         │
│  chrony · Europe/London · BST+0100  │
└──────────────────┬──────────────────┘
                   │ UDP 123 (LAN)
┌──────────────────▼──────────────────┐
│  Windows Server 2025 DC             │
│  192.168.1.10   (Stratum 4)         │
│  W32Time · GMT Standard Time        │
└──────────────────┬──────────────────┘
                   │ Domain time sync
┌──────────────────▼──────────────────┐
│  Future domain-joined clients       │
│                 (Stratum 5)         │
└─────────────────────────────────────┘
```

The DC does not require internet access. The Topton provides a stable, accurate internal time source for the entire ADForest.local domain.

---

## 7. Skills Demonstrated

| Skill | Detail |
|-------|--------|
| Windows Time service repair | Unregister/register cycle, SCM state management |
| w32tm diagnostics | `/query /status`, `/stripchart`, `/resync` flags |
| Active Directory DNS | Identifying missing forwarders, adding via PowerShell |
| Network connectivity testing | `Test-NetConnection`, ping-to-IP to isolate DNS vs routing |
| Windows timezone management | `Get-TimeZone`, `Set-TimeZone` via PowerShell |
| Linux chrony administration | `chronyc tracking`, `chrony.conf`, `timedatectl` |
| firewalld management | `firewall-cmd --add-service`, `--permanent`, `--reload` |
| NTP architecture | Stratum hierarchy design, internal vs external source selection |
| Root cause analysis | Isolating three independent faults contributing to one symptom |

---

*Part of the ADForest.local homelab portfolio — github.com/arturskaufmanis*
