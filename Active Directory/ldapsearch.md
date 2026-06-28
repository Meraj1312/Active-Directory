---

# ldapsearch - Enumerate Active Directory

## Purpose

Query an LDAP server directly to enumerate Active Directory objects, attributes, users, groups, computers, and domain information.

Useful when:

- BloodHound is unavailable
- You need raw LDAP data
- Troubleshooting permissions
- Performing manual enumeration

---

# ldapsearch - Authenticated LDAP Enumeration

## Purpose

Perform an authenticated LDAP search against the domain.

### Requirements

- [ ] Valid domain credentials
- [ ] LDAP (389) or LDAPS (636) accessible

### Command

```bash
ldapsearch -x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-D "<USERNAME>@<DOMAIN>" \
-w '<PASSWORD>' \
-b "<BASE_DN>"
```

### Example

```bash
ldapsearch -x \
-H ldap://dc01.example.com \
-D "alice@example.com" \
-w 'Password123!' \
-b "DC=example,DC=com"
```

### Common Uses

*Retrieve*

- Users
- Groups
- Computers
- Organizational Units (OUs)
- Service Accounts
- Group Memberships

---

# ldapsearch - Discover Naming Contexts

## Purpose

Retrieve the naming contexts (base Distinguished Names) of the LDAP directory.

Useful when you do **not** know the domain's Base DN.

### Requirements

- [ ] LDAP accessible
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

### Why?

The returned `DC=example,DC=com` becomes your Base DN for future LDAP queries.

---

# ldapsearch - Anonymous LDAP Enumeration

## Purpose

Attempt anonymous LDAP enumeration.

Some environments permit anonymous reads, exposing valuable information without credentials.

### Requirements

- [ ] Anonymous LDAP enabled

### Command

```bash
ldapsearch \
-x \
-H ldap://<DC_HOSTNAME_OR_IP> \
-b "<BASE_DN>"
```

### Example

```bash
ldapsearch \
-x \
-H ldap://dc01.example.com \
-b "DC=example,DC=com"
```

### Possible Information Retrieved

- Users
- Groups
- Computers
- OUs
- Domain information
- Password policies (if exposed)

---

# Useful LDAP Filters

## Enumerate all users

```bash
ldapsearch -x -H ldap://<DC> \
-b "<BASE_DN>" \
"(objectClass=user)"
```

---

## Enumerate all groups

```bash
ldapsearch -x -H ldap://<DC> \
-b "<BASE_DN>" \
"(objectClass=group)"
```

---

## Enumerate all computers

```bash
ldapsearch -x -H ldap://<DC> \
-b "<BASE_DN>" \
"(objectClass=computer)"
```

---

## Find Service Accounts (SPNs)

```bash
ldapsearch -x -H ldap://<DC> \
-b "<BASE_DN>" \
"(servicePrincipalName=*)"
```

Useful for Kerberoasting.

---

## Find Domain Admins Group

```bash
ldapsearch -x -H ldap://<DC> \
-b "<BASE_DN>" \
"(cn=Domain Admins)"
```

---

## Find a Specific User

```bash
ldapsearch -x -H ldap://<DC> \
-b "<BASE_DN>" \
"(sAMAccountName=<USERNAME>)"
```

---

## Display Specific Attributes

```bash
ldapsearch -x -H ldap://<DC> \
-b "<BASE_DN>" \
"(sAMAccountName=<USERNAME>)" \
memberOf pwdLastSet userAccountControl servicePrincipalName
```

Useful attributes include:

- memberOf
- userAccountControl
- servicePrincipalName
- pwdLastSet
- lastLogon
- description
- msDS-AllowedToDelegateTo
- msDS-KeyCredentialLink

---

# Typical LDAP Enumeration Workflow

```text
Discover Naming Context
        │
        ▼
Find Base DN
        │
        ▼
Anonymous LDAP? (Try)
        │
        ▼
Authenticated LDAP
        │
        ▼
Enumerate
 ├── Users
 ├── Groups
 ├── Computers
 ├── OUs
 ├── SPNs
 ├── Descriptions
 └── Interesting Attributes
        │
        ▼
Identify Attack Paths
```
