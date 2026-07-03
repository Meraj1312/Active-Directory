# BloodHound-python - Collect AD Data using Kerberos

## Purpose

Collect Active Directory objects and attack paths for BloodHound.

### Requirements

- [ ] Valid credentials
- [ ] LDAP access
- [ ] DNS working
- [ ] BloodHound-python installed

### Command

```bash
bloodhound-python \
-d <DOMAIN> \
-u <USERNAME> \
-p '<PASSWORD>' \
-k \
-ns <DNS_SERVER> \
-dc <DC_HOSTNAME> \
-c all \
--zip \
--auth-method kerberos
```

### Output

Creates a BloodHound ZIP archive ready for import.

---
