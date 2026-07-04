ADCS Mastery Guide for CPTS

Section 1 — Foundations (Read This Before Any ESC Attack)


Why this section exists: Every ESC attack is just a different way of abusing the same underlying machinery. If you memorize certipy req flags without understanding why the CA trusts the certificate it just issued, you'll pass HTB boxes by pattern-matching and fail the moment CPTS gives you a slightly different misconfiguration. This section builds the mental model. Everything after this is "which piece of this model is broken."




1.1 What Problem Does PKI Solve?

Passwords have a fundamental weakness: to prove you know one, you either send it (risking interception) or send a hash of it (risking replay/cracking). Public Key Infrastructure (PKI) solves authentication differently — using asymmetric cryptography.

ConceptExplanationPrivate KeySecret, never leaves the holder. Used to sign or decrypt.Public KeyShareable. Used to verify signatures or encrypt data for the owner.CertificateA public key + identity info (name, purpose), digitally signed by a trusted third party (a CA), binding "this public key belongs to this identity."Certificate Authority (CA)The trusted third party. Its signature is what makes a certificate meaningful — anyone can generate a keypair, but only the CA's signature makes AD trust it.

The core trust chain: Domain Controllers trust the CA → CA signs a certificate saying "this key belongs to jsmith" → jsmith presents that certificate → DC believes jsmith is authenticated, without ever seeing a password.

This is exactly why ADCS attacks matter offensively: if you can get the CA to sign a certificate claiming you are Administrator, the DC will believe it — no password needed, no hash needed. That single sentence is the entire premise of ESC1 through ESC8. Keep it in your head for the whole guide.


1.2 ADCS Architecture

┌─────────────────────────────────────────────────────────────────┐
│                        ACTIVE DIRECTORY FOREST                    │
│                                                                     │
│   ┌──────────────┐        ┌───────────────────────────────────┐  │
│   │ Domain        │◄──────►│         Enterprise CA              │  │
│   │ Controller(s) │  LDAP  │  (Integrated with AD, publishes    │  │
│   │  (KDC)        │        │   templates via AD, auto-enrolls)  │  │
│   └──────┬────────┘        └───────────────┬───────────────────┘  │
│          │                                   │                      │
│          │ Trusts certs signed by CAs        │ Issues certs based   │
│          │ in NTAuth store                   │ on Certificate       │
│          │                                   │ Templates            │
│          ▼                                   ▼                      │
│   ┌──────────────┐                 ┌───────────────────────┐      │
│   │  NTAuth Store │                 │  Certificate Templates │      │
│   │ (list of CAs  │                 │  (AD objects defining  │      │
│   │  trusted for  │                 │   what a cert can be   │      │
│   │  AD auth)     │                 │   used for & by whom)  │      │
│   └──────────────┘                 └───────────────────────┘      │
│                                                                     │
│   Clients (users/computers) ──CSR──► CA ──Certificate──► Client   │
│                                                                     │
└─────────────────────────────────────────────────────────────────┘

Key components explained

Enterprise CA vs Standalone CA

Enterprise CAStandalone CAAD-integratedYesNo (or optional)Uses Certificate TemplatesYesNo — uses request attributes onlyAuto-enrollment supportedYesNoPublishes to AD (LDAP)YesNoRelevant to ADCS attacksAlmost always the targetRarely the focus of ESC attacks

CPTS and real engagements are ~always about Enterprise CAs, because they're the ones integrated with AD authentication and templates — the whole attack surface this guide covers assumes Enterprise CA.

Certificate Templates
A Certificate Template is an AD object (lives in the Configuration partition, under CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=...) that acts as a policy — it defines:


Who is allowed to enroll (enrollment permissions — an ACL, just like any AD object)
What the certificate can be used for (EKUs)
Whether the requester can specify their own identity in the request (ENROLLEE_SUPPLIES_SUBJECT)
Whether a manager must approve the request
Whether signatures from an existing authorized cert are required
Cryptographic settings (key size, algorithm)
Validity period


This is the #1 thing to understand: a Certificate Template is just an ACL-protected object with settings. Almost every ESC attack (ESC1–ESC4) is "the ACL or the settings on this object are wrong."

CA Database & CA object
The CA itself is also an AD object with its own ACL (pKIEnrollmentService). Permissions here (like ManageCA, ManageCertificates) are what ESC7 abuses.


1.3 The Certificate Lifecycle (Step by Step)

Enterprise CAActive DirectoryUser/ComputerEnterprise CAActive DirectoryUser/Computer#mermaid-rvq-r5 { font-family: "Anthropic Sans", system-ui, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; font-size: 16px; fill: rgb(229, 229, 229); }
#mermaid-rvq-r5 .edge-animation-slow { stroke-dashoffset: 900; animation: 50s linear 0s infinite normal none running dash; stroke-linecap: round; stroke-dasharray: 9, 5 !important; }
#mermaid-rvq-r5 .edge-animation-fast { stroke-dashoffset: 900; animation: 20s linear 0s infinite normal none running dash; stroke-linecap: round; stroke-dasharray: 9, 5 !important; }
#mermaid-rvq-r5 .error-icon { fill: rgb(204, 120, 92); }
#mermaid-rvq-r5 .error-text { fill: rgb(51, 135, 163); stroke: rgb(51, 135, 163); }
#mermaid-rvq-r5 .edge-thickness-normal { stroke-width: 1px; }
#mermaid-rvq-r5 .edge-thickness-thick { stroke-width: 3.5px; }
#mermaid-rvq-r5 .edge-pattern-solid { stroke-dasharray: 0; }
#mermaid-rvq-r5 .edge-thickness-invisible { stroke-width: 0; fill: none; }
#mermaid-rvq-r5 .edge-pattern-dashed { stroke-dasharray: 3; }
#mermaid-rvq-r5 .edge-pattern-dotted { stroke-dasharray: 2; }
#mermaid-rvq-r5 .marker { fill: rgb(161, 161, 161); stroke: rgb(161, 161, 161); }
#mermaid-rvq-r5 .marker.cross { stroke: rgb(161, 161, 161); }
#mermaid-rvq-r5 svg { font-family: "Anthropic Sans", system-ui, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; font-size: 16px; }
#mermaid-rvq-r5 p { margin: 0px; }
#mermaid-rvq-r5 .actor { stroke: rgb(161, 161, 161); fill: transparent; stroke-width: 1; }
#mermaid-rvq-r5 rect.actor.outer-path[data-look="neo"] { filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); }
#mermaid-rvq-r5 rect.note[data-look="neo"] { stroke: rgb(161, 161, 161); fill: rgb(45, 45, 45); filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); }
#mermaid-rvq-r5 text.actor > tspan { fill: rgb(229, 229, 229); stroke: none; }
#mermaid-rvq-r5 .actor-line { stroke: rgb(161, 161, 161); }
#mermaid-rvq-r5 .innerArc { stroke-width: 1.5; stroke-dasharray: none; }
#mermaid-rvq-r5 .messageLine0 { stroke-width: 1.5; stroke-dasharray: none; stroke: rgb(229, 229, 229); }
#mermaid-rvq-r5 .messageLine1 { stroke-width: 1.5; stroke-dasharray: 2, 2; stroke: rgb(229, 229, 229); }
#mermaid-rvq-r5 [id$="-arrowhead"] path { fill: rgb(229, 229, 229); stroke: rgb(229, 229, 229); }
#mermaid-rvq-r5 .sequenceNumber { fill: rgb(94, 94, 94); }
#mermaid-rvq-r5 [id$="-sequencenumber"] { fill: rgb(229, 229, 229); }
#mermaid-rvq-r5 [id$="-crosshead"] path { fill: rgb(229, 229, 229); stroke: rgb(229, 229, 229); }
#mermaid-rvq-r5 .messageText { fill: rgb(229, 229, 229); stroke: none; }
#mermaid-rvq-r5 .labelBox { stroke: rgb(161, 161, 161); fill: transparent; filter: none; }
#mermaid-rvq-r5 .labelText, #mermaid-rvq-r5 .labelText > tspan { fill: rgb(229, 229, 229); stroke: none; }
#mermaid-rvq-r5 .loopText, #mermaid-rvq-r5 .loopText > tspan { fill: rgb(229, 229, 229); stroke: none; }
#mermaid-rvq-r5 .sectionTitle, #mermaid-rvq-r5 .sectionTitle > tspan { fill: rgb(229, 229, 229); stroke: none; }
#mermaid-rvq-r5 .loopLine { stroke-width: 2px; stroke-dasharray: 2, 2; stroke: rgb(161, 161, 161); fill: rgb(161, 161, 161); }
#mermaid-rvq-r5 .note { stroke: rgb(161, 161, 161); fill: rgb(45, 45, 45); }
#mermaid-rvq-r5 .noteText, #mermaid-rvq-r5 .noteText > tspan { fill: rgb(229, 229, 229); stroke: none; font-weight: normal; }
#mermaid-rvq-r5 .activation0 { fill: transparent; stroke: rgba(0, 0, 0, 0); }
#mermaid-rvq-r5 .activation1 { fill: transparent; stroke: rgba(0, 0, 0, 0); }
#mermaid-rvq-r5 .activation2 { fill: transparent; stroke: rgba(0, 0, 0, 0); }
#mermaid-rvq-r5 .actorPopupMenu { position: absolute; }
#mermaid-rvq-r5 .actorPopupMenuPanel { position: absolute; fill: transparent; box-shadow: rgba(0, 0, 0, 0.2) 0px 8px 16px 0px; filter: drop-shadow(rgba(0, 0, 0, 0.4) 3px 5px 2px); }
#mermaid-rvq-r5 .actor-man circle, #mermaid-rvq-r5 line { fill: transparent; stroke-width: 2px; }
#mermaid-rvq-r5 g rect.rect { filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); stroke: rgb(161, 161, 161); }
#mermaid-rvq-r5 .node .neo-node { stroke: rgb(161, 161, 161); }
#mermaid-rvq-r5 [data-look="neo"].node rect, #mermaid-rvq-r5 [data-look="neo"].cluster rect, #mermaid-rvq-r5 [data-look="neo"].node polygon { stroke: url("#mermaid-rvq-r5-gradient"); filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); }
#mermaid-rvq-r5 [data-look="neo"].node path { stroke: url("#mermaid-rvq-r5-gradient"); stroke-width: 1px; }
#mermaid-rvq-r5 [data-look="neo"].node .outer-path { filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); }
#mermaid-rvq-r5 [data-look="neo"].node .neo-line path { stroke: rgb(161, 161, 161); filter: none; }
#mermaid-rvq-r5 [data-look="neo"].node circle { stroke: url("#mermaid-rvq-r5-gradient"); filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); }
#mermaid-rvq-r5 [data-look="neo"].node circle .state-start { fill: rgb(0, 0, 0); }
#mermaid-rvq-r5 [data-look="neo"].icon-shape .icon { fill: url("#mermaid-rvq-r5-gradient"); filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); }
#mermaid-rvq-r5 [data-look="neo"].icon-shape .icon-neo path { stroke: url("#mermaid-rvq-r5-gradient"); filter: drop-shadow(rgb(185, 185, 185) 1px 2px 2px); }
#mermaid-rvq-r5 :root { --mermaid-font-family: "Anthropic Sans",system-ui,"Segoe UI",Roboto,Helvetica,Arial,sans-serif; }alt[Manager approval required][Auto-issue]1. Query available templates (LDAP)Templates + ACLs returned2. Build Certificate Signing Request (CSR)(generate keypair, fill Subject/SAN per template rules)3. Submit CSR via RPC (ICertPassage) orHTTP (Web Enrollment / CES) or DCOM4. Validate requester identity & permissionsagainst template ACL5. Validate CSR against template policy(check EKU, subject rules, approval requirements)Request pending6. Sign certificate with CA private key7. Issued certificate returned8. Authenticate later using cert (PKINIT or Schannel)

Step-by-step meaning:


Enumeration — the client (or attacker) looks up which templates exist and which it has Enroll rights on. This is identical to what certipy find or Certify.exe find automate.
CSR construction — the client generates an RSA/ECC keypair locally, keeps the private key, and builds a request containing the public key + identity claims (Subject Name, and possibly Subject Alternative Name if allowed).
Submission — sent to the CA over RPC (ICertPassage/ICertRequestD), HTTP Web Enrollment, or Certificate Enrollment Service (CES/CEP, used for HTTP-based enrollment — this is the surface ESC8 relays into).
Identity/permission validation — CA checks the requester (Kerberos/NTLM-authenticated identity) against the template's ACL. No Enroll permission = rejected here.
Policy validation — CA checks the CSR contents obey the template's rules. This is where misconfigurations like ENROLLEE_SUPPLIES_SUBJECT become dangerous — if the template allows the requester to specify arbitrary SAN, the CA will not correct it.
Signing — CA uses its own private key to sign the certificate, cryptographically binding the public key to the claimed identity.
Issuance — certificate handed back to requester.
Later use — the certificate is used to authenticate, via PKINIT (Kerberos) or Schannel (TLS-based, e.g., LDAPS).



1.4 Certificate Templates — The Fields That Matter

FieldWhat it ControlsWhy Attackers CareEnrollment Rights (ACL)Who can request a cert from this templateIf low-priv users have Enroll, they can request in the first place (baseline for every ESC)msPKI-Certificate-Name-FlagWhether requester can supply their own Subject/SAN (ENROLLEE_SUPPLIES_SUBJECT bit)ESC1 — if set, requester can claim to be anyone, including Domain AdminExtended Key Usage (EKU)What purposes the cert is valid for (Client Auth, Smart Card Logon, Any Purpose, Certificate Request Agent, etc.)Determines whether the cert can be used to authenticate to AD at all, or to sign requests on behalf of others (ESC3)Authorized Signatures RequiredRequires the CSR be co-signed by an already-issued authorized certificateMitigation for ESC1/ESC3 abuse when properly configuredManager ApprovalRequests sit pending until an admin approvesMitigation — blocks silent self-service abuseTemplate Object ACL (WriteOwner/WriteDacl/GenericWrite/etc.)Who can modify the template itselfESC4 — if you can write to the template, you can inject ENROLLEE_SUPPLIES_SUBJECT yourself, turning any template into ESC1pkiextendedkeyusage vs Client Authentication OID specifically(1.3.6.1.5.5.7.3.2 = Client Auth, 1.3.6.1.4.1.311.20.2.2 = Smart Card Logon, 2.5.29.37.0 = Any Purpose, no EKU = also treated as Any Purpose)Determines PKINIT eligibility

EKU cheat table (memorize this — it recurs in every ESC section)

EKUOIDAllows Domain Auth?Client Authentication1.3.6.1.5.5.7.3.2✅ YesSmart Card Logon1.3.6.1.4.1.311.20.2.2✅ YesAny Purpose2.5.29.37.0✅ Yes(No EKU present at all)—✅ Yes (treated as Any Purpose by Windows CA)Certificate Request Agent1.3.6.1.4.1.311.20.2.1❌ No — but lets holder request certs on behalf of others (ESC3)Server Authentication only1.3.6.1.5.5.7.3.1❌ NoCode Signing / Email Protection etc.(various)❌ No


1.5 Subject Name, SAN, and UPN — Don't Confuse These

This trio is where most beginners get confused, and where ESC1/ESC6/ESC9/ESC10 all live.


Subject Name (Subject DN): The "X.500 Distinguished Name" field of the cert, e.g. CN=jsmith,OU=Users,DC=corp,DC=local. Historically used for identity mapping but considered a weak mapping mechanism.
Subject Alternative Name (SAN): An extension field that can carry additional identity claims — critically, a userPrincipalName (UPN) like Administrator@corp.local, or an rfc822Name (email), or dNSName (for machine certs).
UPN mapping: When a DC receives a certificate during PKINIT/Schannel auth, it looks at the SAN's UPN field (if present) to determine which AD account this certificate claims to represent, then checks that account exists and the mapping is valid.


Why this matters offensively: if a template lets the requester choose the SAN (ENROLLEE_SUPPLIES_SUBJECT), you can request a certificate for the "User" template using your own low-privileged credentials to authenticate, but put Administrator@corp.local in the SAN's UPN field. The CA signs it (it only validated that you have Enroll rights, not that the SAN matches your account) → you now hold a certificate the DC will treat as Administrator's identity. This is ESC1, and you now understand why it works, before we even open Certipy.


1.6 Authentication: PKINIT vs Schannel

Certificates aren't only useful in Kerberos — but Kerberos (via PKINIT) is the primary path CPTS/HTB cares about because it directly yields a TGT.

PKINIT (Public Key Cryptography for Initial Authentication)

An extension to Kerberos (RFC 4556) that replaces the password-derived key in the AS-REQ/AS-REP exchange with certificate-based public key crypto.

Client                                    KDC (Domain Controller)
  │  AS-REQ (PA-PK-AS-REQ: cert + signed  │
  │  timestamp, signed with client's       │
  │  private key)                          │
  ├────────────────────────────────────────►
  │                                         │  1. Validate cert chain (is CA in NTAuth?)
  │                                         │  2. Validate cert not revoked/expired
  │                                         │  3. Map cert → AD account (via SAN UPN, 
  │                                         │     or explicit altSecurityIdentities mapping)
  │                                         │  4. Build reply encrypted partly with a key
  │                                         │     derived via Diffie-Hellman, partly with
  │                                         │     the client's public key
  │  AS-REP (PA-PK-AS-REP: TGT +           │
  │  session key material)                 │
  ◄────────────────────────────────────────┤
  │                                         │
  │  Client now holds a valid TGT for      │
  │  the mapped account — no password      │
  │  was ever used.                        │

Critical detail — NTAuth Store: The DC will only trust certificates issued by a CA whose certificate is published in the NTAuth Store (CN=NTAuthCertificates,CN=Public Key Services,CN=Services,CN=Configuration,DC=...). This is separate from a CA merely existing/being trusted at the OS trust-store level. A legitimate Enterprise CA is automatically added here during install. This matters for ESC5/forged CA scenarios and for understanding why a random self-signed cert can't just be used against a DC.

The NTLM fallback quirk (important for real engagements): When a service that only understands NTLM needs to validate a PKINIT-authenticated user, the KDC can embed the user's NTLM hash inside the PAC (PAC_CREDENTIAL_DATA), encrypted for the requesting service. Tools like PKINITtools (getnthash.py) abuse this legitimate feature to convert "I have a certificate" into "I have the NTLM hash," which is huge for lateral movement and pass-the-hash afterward.

Schannel (TLS client-cert auth)

Used for things like LDAPS mutual auth. Less central to CPTS's typical ESC path but relevant for ESC9/ESC10 (certificate mapping) discussions since Schannel mapping behavior changed after the Certifried/strong-mapping patches (more in Section 6).


1.7 Certificate Mapping — Strong vs Weak (Foundational for ESC9/ESC10/Certifried)

After CVE-2022-26923 ("Certifried") and the associated May 2022 patches, Microsoft tightened how certificates map to AD accounts. You need the vocabulary now, even though we won't fully exploit it until Section 6.

Mapping TypeMechanismStrengthExplicit strong mappingaltSecurityIdentities attribute on the account explicitly lists the trusted cert (by issuer+serial, SKI, or SHA1 hash)StrongUPN implicit mappingSAN's UPN matches the account's userPrincipalNameWeak by default (this is what ESC1 abuses) — but Microsoft added a szOID_NTDS_CA_SECURITY_EXT extension in patched CAs to strongly bind cert→SID, closing part of this gapSAN dNSName mapping (computer accounts)Machine cert's SAN DNS name matches computer objectHistorically weak — root of Certifried

Since Nov 2023 (and enforced by default since Feb 2025), DCs operate in Full Enforcement mode by default for strong certificate mapping — meaning weakly-mapped certs (no SID security extension, no explicit strong mapping) get rejected, not just warned. This is hugely relevant: some older ESC1 walkthroughs will differ slightly from current patched behavior. We'll flag this explicitly in every ESC section going forward.


1.8 Enrollment Agents (Foundational for ESC3)

Normally you can only enroll for a certificate representing yourself. An Enrollment Agent certificate (EKU: Certificate Request Agent) is a special exception — it lets its holder submit a CSR on behalf of another user, and the CA will issue a certificate for that other identity. This exists legitimately for smart card administrators provisioning cards for employees. Offensively: if you can obtain an Enrollment Agent certificate, and a template that allows enrollment-agent-signed requests without restricting which users can be impersonated, you can mint a certificate for Domain Admin without ever compromising their account directly. That's ESC3.


1.9 The Mental Model — Tie It All Together

Before moving to attacks, make sure you can answer these without looking back:


What makes a certificate trustworthy to a DC? → It was signed by a CA in the NTAuth store, and the DC can map it to a valid AD account.
What determines what identity a certificate claims? → The Subject Name / SAN, filled in either by the CA (safe) or by the requester (dangerous, if the template allows it).
What determines whether you're even allowed to request a certificate from a template? → The template's ACL (Enroll permission).
What determines whether you can request on behalf of someone else? → Holding an Enrollment Agent certificate + template allowing agent-based requests.
What determines whether the CA itself might sign something it shouldn't? → CA-level flags (like EDITF_ATTRIBUTESUBJECTALTNAME2) and CA object permissions (ManageCA).


Every ESC number is just "which of these five trust boundaries is broken." Keep this table mentally bookmarked — I'll reference it by number in every subsequent section:

ESCBroken Trust Boundary (from list above)ESC1#2 — requester controls their own claimed identityESC2#2 variant — Any Purpose/No EKU template misusedESC3#4 — enrollment agent abuseESC4#3 — template ACL itself is writableESC5#5 — broader PKI object ACL abuseESC6#5 — CA-wide flag lets #2 apply to every templateESC7#5 — CA object ACL (ManageCA/ManageCertificates)ESC8#1 — NTLM relay to HTTP enrollment forges "who is asking"ESC9/10#1 — certificate mapping/security extension weaknessesESC11#1 — relay to RPC enrollment (ICPR, no encryption)


1.10 What You Need Installed / Available (Prep for Enumeration Sections)

ToolPlatformPurposeCertipy (5.x)LinuxThe primary modern offensive ADCS tool — enumeration + exploitationCertifyWindows (.NET)GhostPack's original enumeration/request toolPKINITtoolsLinuxPKINIT auth from certs, NTLM hash extraction from PACRubeusWindowsKerberos ticket manipulation, asktgt /certificatePSPKIAuditWindows (PowerShell)Deep ADCS auditingBloodHound (with ADCS collection)BothAttack path graphing including ESC edgesopensslBothManual cert inspection/conversionNetExec / ImpacketLinuxLDAP enum, relay (ESC8/ESC11), auth

We'll introduce exact commands and every flag starting in the next section, per attack.


✅ Section 1 Complete — Self-Check Before Continuing

You should now be able to explain, in your own words:


Why a certificate is more powerful than a password for an attacker
What a Certificate Template actually is (an ACL-protected AD object with policy settings)
The full lifecycle from CSR to authenticated TGT
What ENROLLEE_SUPPLIES_SUBJECT does and why it's dangerous
The difference between weak and strong certificate mapping, and why it matters post-2022/2025 patches
What an Enrollment Agent certificate is
