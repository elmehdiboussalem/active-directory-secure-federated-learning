Active Directory Hardening
Overview

The Active Directory environment was intentionally deployed with controlled weaknesses so that representative attack techniques could be executed and measured.

After each attack, a corresponding hardening measure was applied and the attack was replayed to verify that the original attack path was no longer effective.

The hardening strategy therefore follows:

Identify Weakness
      │
      ▼
Execute Attack
      │
      ▼
Apply Mitigation
      │
      ▼
Replay Attack
      │
      ▼
Verify Failure

The main hardening domains are:

Kerberos
NTLM
LSASS protection
Local administrator credentials
LLMNR / NBT-NS
AD CS
krbtgt
DNS
Administrative tiering
Active Directory permissions
1. Kerberos Hardening
AS-REP Roasting Protection

AS-REP Roasting was initially enabled through a controlled account configured with:

DONT_REQ_PREAUTH

The remediation consisted of:

re-enabling Kerberos pre-authentication;
assigning a strong password to the affected account;
auditing the domain for remaining accounts with disabled pre-authentication.

The final enumeration returned no roastable accounts.

Before:
DONT_REQ_PREAUTH → AS-REP material available

After:
Kerberos pre-authentication enabled
→ No roastable account

This was verified by replaying the AS-REP enumeration after remediation.

Kerberoasting Protection

The initial environment contained service accounts with SPNs that could be targeted for Kerberoasting.

The hardening strategy was:

Prefer managed service accounts where possible.
Use strong service-account secrets.
Prefer AES Kerberos encryption.
Remove unnecessary SPNs.

The project deployed:

AES128 / AES256
msDS-SupportedEncryptionTypes = 24
KDS root key
gMSA migration
SPN removal

The hardened ticket was observed using AES-256 (etype 18).

The offline cracking test did not recover the AES-protected secret during the reported test period.

The exposed SPN was subsequently removed and GetUserSPNs returned:

No entries found

The former service-account credentials also failed authentication after remediation.

2. Pass-the-Hash Hardening

Pass-the-Hash protection combines two defensive layers:

Protect the credential
        +
Block credential replay

The project implemented six complementary measures.

2.1 RunAsPPL

LSASS was configured as a Protected Process Light:

HKLM\...\Lsa\RunAsPPL = 1

This protects LSASS against direct credential extraction by ordinary user-mode processes.

2.2 Credential Guard

Credential Guard was enabled using virtualization-based security.

The project configured:

LsaCfgFlags = 1

and verified the VBS / Credential Guard state using:

Win32_DeviceGuard

The objective is to isolate sensitive authentication material from the normal operating-system environment.

2.3 NTLMv2 Only

The LAN Manager authentication level was configured to:

Send NTLMv2 response only.
Refuse LM & NTLM.

This removes legacy LM and NTLM authentication from the accepted authentication mechanisms.

2.4 NTLM Restriction

The project also configured the NTLM restriction policy so that incoming NTLM authentication was refused.

This provides a second defensive layer against replay of NTLM credentials.

2.5 RestrictRemoteSAM

Remote SAM access was restricted so that only administrators could query the SAM database remotely.

The implemented security descriptor limits access to the Administrators group.

2.6 Persistent VBS / Credential Guard

The VBS / Credential Guard configuration was enforced through Group Policy with UEFI protection so that the security configuration could not simply be removed through a normal user-level change.

Verification

After all six controls were deployed, Pass-the-Hash was replayed using the three tested mechanisms:

impacket-psexec
crackmapexec
impacket-smbexec

All three failed with:

STATUS_NOT_SUPPORTED

This provided direct evidence that the original NTLM replay path was no longer functional.

3. LLMNR / NBT-NS Hardening

The initial environment allowed LLMNR/NBT-NS poisoning to be demonstrated.

The remediation removed the vulnerable name-resolution mechanisms and strengthened SMB authentication.

LLMNR

LLMNR was disabled through Group Policy:

Network
└── DNS Client
    └── Turn off multicast name resolution
        = Enabled
NBT-NS

NBT-NS was disabled on the relevant network interfaces.

This removed the NetBIOS fallback mechanism that could otherwise provide another poisoning path.

SMB Signing

SMB signing was enforced.

This reduces the possibility of relaying captured NTLM authentication material to SMB services.

Strong Passwords

Strong passwords were also applied to user accounts so that captured NTLMv2 material would be significantly harder to recover through offline cracking.

The defensive model is therefore:

Disable poisoning
       +
Block relay
       +
Strengthen credentials
       │
       ▼
Reduced LLMNR/NBT-NS attack surface

The project verified the resulting configuration during the replay phase.

4. Local Administrator Protection with LAPS

Local administrator password reuse creates a major lateral-movement risk.

The project therefore deployed LAPS across the Windows clients.

The configured policy included:

Password length:       16 characters
Uppercase:             Enabled
Lowercase:             Enabled
Digits:                Enabled
Special characters:    Enabled
Rotation period:       30 days

Each client receives its own randomized local administrator password.

This prevents the same local administrator credential from being reused across multiple Windows hosts.

The project also verified that a previously captured local administrator hash was rejected after the LAPS password rotation.

5. krbtgt Protection

The krbtgt account is a critical Tier 0 security asset because its secret is used to validate Kerberos ticket material.

The project therefore used a double rotation strategy.

First rotation
      │
      ▼
Old secret becomes previous secret
      │
      ▼
Second rotation
      │
      ▼
Previous compromised secret removed

The previously generated Golden Ticket was replayed after the two rotations.

The domain controller rejected it with:

KRB_AP_ERR_BAD_INTEGRITY

This confirmed that the forged ticket was no longer accepted.

6. AD CS / ESC8 Hardening

The initial AD CS environment exposed the web enrollment endpoint:

/certsrv/

This configuration was deliberately used to demonstrate the ESC8 relay path.

The remediation was to remove the unnecessary Web Enrollment component from the Certificate Authority.

The vulnerable certificate template was also removed.

After remediation:

/certsrv/

returned:

HTTP/1.1 404 Not Found

Certificate-template enumeration also confirmed that the vulnerable template was no longer available.

This removed the tested ESC8 attack entry point.

7. DNSSEC

DNSSEC was deployed as a cross-cutting DNS security measure.

The nord.corp DNS zone was signed using:

KSK
ZSK
RSA/SHA-256
NSEC3

The objective was to provide authenticity and integrity for DNS responses and reduce the risk associated with DNS cache poisoning.

DNSSEC therefore complements the endpoint-level controls applied to LLMNR and NBT-NS.

8. Active Directory Tier Model

A Tier 0 / Tier 1 / Tier 2 administrative model was deployed to reduce privilege exposure.

The Active Directory structure was reorganized into dedicated organizational units:

Tier 0
├── Accounts
├── Computers
├── Groups
└── ServiceAccounts

Tier 1
├── Accounts
├── Computers
├── Groups
└── ServiceAccounts

Tier 2
├── Accounts
├── Computers
├── Groups
└── ServiceAccounts

Dedicated administrative identities were created for each tier:

a0-admin
a1-admin
a2-admin

The Tier 0 administrator was placed in Protected Users and configured as non-delegable.

Domain controllers were moved into the Tier 0 computer scope.

The project verified that an administrative account belonging to one tier could not simply authenticate to a machine belonging to another tier.

This provides a strong architectural boundary between:

Tier 0 → Domain / Identity infrastructure
Tier 1 → Server administration
Tier 2 → Workstation administration

The tier model was therefore used as a defense against credential exposure and lateral movement rather than merely as an organizational naming convention.

9. Active Directory Enumeration Hardening

The project also reduced unnecessary information available to unprivileged users.

RestrictRemoteSAM

Remote SAM enumeration was restricted to administrators.

Session Enumeration

Remote session enumeration was restricted using the SrvsvcSessionInfo security configuration.

ACL Cleanup

Unnecessary privileged relationships such as excessive GenericAll and WriteDACL permissions were removed.

These measures reduce the information available to reconnaissance tools such as BloodHound and make privilege-escalation paths harder to discover.

10. Security Policy Baseline

The laboratory password and account policy included:

Policy	Configuration
Minimum password length	14 characters
Password complexity	Enabled
Password history	24 passwords
Maximum password age	90 days
Account lockout threshold	5 failed attempts

These settings were part of the broader credential-hardening strategy used during the attack validation.

11. Hardening Matrix
Security Area	Control	Verification
AS-REP Roasting	Kerberos pre-authentication	No roastable accounts
Kerberoasting	AES + gMSA + SPN removal	No vulnerable SPN
Pass-the-Hash	RunAsPPL	LSASS protected
Pass-the-Hash	Credential Guard / VBS	VBS state verified
Pass-the-Hash	NTLMv2 only	Legacy NTLM rejected
Pass-the-Hash	NTLM restriction	Replay blocked
Pass-the-Hash	RestrictRemoteSAM	Remote SAM restricted
LLMNR	GPO disablement	Poisoning path removed
NBT-NS	Interface disablement	NetBIOS fallback removed
SMB relay	SMB signing	Signing enforced
Local admin	LAPS	Old hash rejected
Golden Ticket	Double krbtgt rotation	KRB_AP_ERR_BAD_INTEGRITY
AD CS / ESC8	Remove Web Enrollment	/certsrv/ → 404
DNS	DNSSEC	Signed zone
Privilege escalation	Tier 0/1/2	Cross-tier login blocked
AD reconnaissance	RestrictRemoteSAM / session restrictions	Reduced enumeration
12. Defense-in-Depth Model

The final security architecture combines several layers rather than relying on a single mitigation:

                    Active Directory
                           │
          ┌────────────────┼────────────────┐
          │                │                │
       Identity          Network          Endpoint
          │                │                │
   Kerberos hardening   NTLM limits     RunAsPPL
   gMSA / AES           SMB signing     Credential Guard
   krbtgt rotation      LLMNR off       LAPS
   Tier Model            NBT-NS off      Windows auditing
          │                │                │
          └────────────────┼────────────────┘
                           │
                           ▼
                  Windows Telemetry
                           │
                           ▼
                  Federated Detection
                           │
                           ▼
                     Security Alert

This architecture connects preventive controls with the project's detection capability.

13. Verification Philosophy

A key characteristic of the project is that hardening was not considered complete merely because a configuration option was enabled.

Each major control was followed by an attack replay or an explicit configuration check.

Examples include:

AS-REP Roasting
→ No roastable accounts

Kerberoasting
→ No vulnerable SPN

Pass-the-Hash
→ STATUS_NOT_SUPPORTED

LLMNR Poisoning
→ Vulnerable protocols disabled

LAPS
→ Previous local-admin hash rejected

Golden Ticket
→ KRB_AP_ERR_BAD_INTEGRITY

ESC8
→ /certsrv/ returns 404

Tier Model
→ Cross-tier authentication rejected

This attack → harden → replay methodology provides stronger evidence than configuration screenshots alone.

14. Relationship with Federated Detection

The hardening phase is directly connected to the federated learning component.

The offensive phase generates realistic Windows telemetry.

The detector learns from this type of telemetry.

The hardening phase then reduces the attack surface and allows the same scenarios to be replayed to determine whether:

the attack still succeeds;
the telemetry changes;
the detector still identifies the resulting behavior.

The complete defensive lifecycle is:

Attack
  │
  ▼
Windows Telemetry
  │
  ▼
Federated Learning
  │
  ▼
Detection
  │
  ▼
Hardening
  │
  ▼
Attack Replay
  │
  ▼
Verification

This creates a continuous relationship between Active Directory security engineering and privacy-preserving machine-learning-based detection.

Conclusion

The Active Directory hardening phase transformed the initial intentionally vulnerable laboratory into a significantly more resilient environment.

The main controls covered:

Kerberos pre-authentication;
AES and gMSA for service accounts;
LSASS protection;
Credential Guard;
NTLM restrictions;
LLMNR/NBT-NS removal;
SMB signing;
LAPS;
double krbtgt rotation;
AD CS Web Enrollment removal;
DNSSEC;
Active Directory tiering;
reconnaissance restrictions.

The important result is not only that these controls were configured, but that the project systematically replayed the corresponding attacks and verified their failure.

This provides an end-to-end security engineering workflow:

Offensive Assessment
        ↓
Telemetry Collection
        ↓
Federated Detection
        ↓
Security Hardening
        ↓
Attack Replay
        ↓
Measured Verification
