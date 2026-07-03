# LDAP Enumeration Cheat Sheet

> **Purpose:** Query an LDAP server to enumerate Active Directory objects, users, groups, computers, and domain information.

---

# Authenticated LDAP Enumeration

## Description

Perform an authenticated LDAP search against the domain.

### Requirements

- [ ] Valid domain credentials
- [ ] LDAP (389) or LDAPS (636) reachable

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-D "<USERNAME>@<DOMAIN>" \
-w '<PASSWORD>' \
-b "<BASE_DN>"
```

### Example

```bash
ldapsearch \
-x \
-H ldap://dc01.example.com \
-D "alice@example.com" \
-w 'Password123!' \
-b "DC=example,DC=com"
```

### Common Enumeration Targets

- Users
- Groups
- Computers
- Organizational Units (OUs)
- Service Accounts
- Group Membership
- Domain Information

---

# Discover the Base DN (Naming Context)

## Description

If you don't know the domain's Base DN (e.g. `DC=example,DC=com`), query the RootDSE.

### Requirements

- [ ] LDAP reachable
- [ ] Anonymous access may work (depends on configuration)

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-s base \
-b "" \
namingContexts
```

### Example Output

```text
namingContexts:
DC=example,DC=com
CN=Configuration,DC=example,DC=com
CN=Schema,CN=Configuration,DC=example,DC=com
DC=DomainDnsZones,DC=example,DC=com
DC=ForestDnsZones,DC=example,DC=com
```

### Why Use This?

The first `DC=...` value is usually your Base DN for future LDAP queries.

---

# Anonymous LDAP Enumeration

## Description

Attempt LDAP enumeration without credentials.

Some Active Directory environments allow anonymous LDAP reads.

### Requirements

- [ ] Anonymous LDAP enabled

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>"
```

### Possible Information

- Users
- Groups
- Computers
- Organizational Units
- Domain Information
- Password Policy (if exposed)

---

# Enumerate All Users

## Purpose

Retrieve every user object in the domain.

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>" \
"(objectClass=user)"
```

---

# Enumerate All Groups

## Purpose

Retrieve every group object.

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>" \
"(objectClass=group)"
```

---

# Enumerate All Computers

## Purpose

Retrieve every computer object.

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>" \
"(objectClass=computer)"
```

---

# Find Accounts with SPNs (Kerberoast Targets)

## Purpose

Locate accounts configured with Service Principal Names.

### Why?

These accounts are potential Kerberoasting targets.

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>" \
"(servicePrincipalName=*)"
```

---

# Find Domain Admins Group

## Purpose

Locate the Domain Admins group.

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>" \
"(cn=Domain Admins)"
```

---

# Find a Specific User

## Purpose

Retrieve information about a single user.

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>" \
"(sAMAccountName=<USERNAME>)"
```

---

# Retrieve Specific LDAP Attributes

## Purpose

Display only the attributes you care about.

### Common Useful Attributes

- memberOf
- userAccountControl
- servicePrincipalName
- pwdLastSet
- lastLogon
- lastLogonTimestamp
- badPwdCount
- lockoutTime
- description
- msDS-AllowedToDelegateTo
- msDS-KeyCredentialLink

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>" \
"(sAMAccountName=<USERNAME>)" \
memberOf \
userAccountControl \
servicePrincipalName \
pwdLastSet
```

---

# LDAP Enumeration Workflow

```text
                 Start
                   │
                   ▼
        Discover Base DN (RootDSE)
                   │
                   ▼
     Try Anonymous LDAP Enumeration
                   │
          ┌────────┴────────┐
          │                 │
       Success           Failed
          │                 │
          ▼                 ▼
 Continue Enum      Authenticate
          │                 │
          └────────┬────────┘
                   ▼
      Enumerate Users / Groups / Computers
                   │
                   ▼
         Search for Interesting Objects
                   │
         ├── SPNs
         ├── Descriptions
         ├── Privileged Groups
         ├── Delegation
         ├── Service Accounts
         ├── Key Credentials
         └── Sensitive Attributes
                   │
                   ▼
          Identify Attack Paths
```
