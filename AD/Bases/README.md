_____

# Basic AD TryHackMe

- For this exercise, we are going to install **xfreerdp**

```
sudo apt install freerdp3-x11 -y 
```

    The command **`xfreerdp3-x11 -y`** is used on Linux systems to connect to a remote desktop using the **RDP (Remote Desktop Protocol)**.

    The **`-y`** option tells FreeRDP to **automatically accept any security certificate** presented by the remote server, without asking for user confirmation. This is useful in controlled or lab environments, but **not recommended in insecure connections**, since it skips certificate validation and could expose you to **man-in-the-middle attacks**.

### **Systems compatible with FreeRDP**

- **Windows**: Any version with Remote Desktop (RDP) enabled
- **Linux**: Servers with **xrdp** installed
- **macOS**: With third-party RDP servers
- **Others**: Embedded devices or virtualization solutions supporting RDP

- Command to use

```
xfreerdp3 /u:Administrator /p:Password321 /v:10.65.163.103:3389 
```
     Sometimes, due to protocol parsing issues, it is recommended to wrap the password in single quotes.

![[Captura de pantalla 2026-02-23 210212.png]]
    Port **3389** is the default port for RDP.

- We complete the questions from **Task 2**

![[Captura de pantalla 2026-02-23 215358.png]]

- Moving into **Task 3**, we work with objects inside **AD DS (Active Directory Domain Services)**. The **TryHackMe** platform provides an introduction.

![[Captura de pantalla 2026-02-23 222659.png]]
    Once inside, with **left click**, we can create new objects and even new **Organizational Units (OUs)** and configure them.

- We are now dealing with **Organizational Units (OU)**, which store **objects** such as:

     Users, Groups, Machines

-  This differs from **Security Groups**. **Security Groups** can be defined as the **privileges assigned to users**, for example:

```
Domain Admins: Users of this group have administrative privileges over the entire domain.
```

or

```
Account Operators: Users in this group can create or modify accounts in the domain.
```

- Another important concept is **machine accounts**

    Each time a computer joins the domain, a machine account is created. These are identified by adding a **$** at the end:

```
A machine named DC01 will have an account called DC01$
```

![[Captura de pantalla 2026-02-23 230345.png]]


- In **Task 4**, we learn about:

    Assigning privileges
    Deleting users and OUs

- First, we delete an **OU**, By default, OUs are protected against accidental deletion. We must disable this setting first.

![[Captura de pantalla 2026-02-24 204650 2.png]]

- Check if an OU is protected:

```
Get-ADOrganizationalUnit -Identity "OU=Sales,DC=thm,DC=local" -Properties ProtectedFromAccidentalDeletion | Select Name, ProtectedFromAccidentalDeletion
```

- Disable protection:

```
Set-ADOrganizationalUnit -Identity "OU=Sales,DC=thm,DC=local" -ProtectedFromAccidentalDeletion $false
```
### Key point

- If deletion still fails, it's because there are objects inside. Use:

```
Remove-ADOrganizationalUnit -Identity "..." -Recursive
```

- The `-Recursive` flag deletes everything inside the OU.



- Now we assign privileges to **phillip** over the **Sales OU** to reset passwords

![[Captura de pantalla 2026-02-24 211552 2.png]]
    Left click → **Delegate Control**

- Alternative (more granular method):

![[Captura de pantalla 2026-02-24 233027 3.png]]
    Left click on the **user to be controlled** → **Properties → Advanced**

- Now we log in as **phillip** and reset **sophie’s** password:

```
Set-ADAccountPassword sophie -Reset -NewPassword (Read-Host -AsSecureString -Prompt 'New Password') -Verbose
```

- This method is better for **pentesting OPSEC**,  The password is not stored in plain text. It is handled in **RAM**, making it harder for Blue Team to detect.

- If you use:

```
Set-ADAccountPassword sophie -NewPassword "Password123!"
```

- The password is stored in command history, With `Read-Host`, only the command is logged, not the password.

```
Hacking123*
```
    We log in as Sophie and get the flag:

```
THM{thanks_for_contacting_support}
```

- Force password change at next login:

```
Set-ADUser -ChangePasswordAtLogon $true -Identity sophie -Verbose
```
# Task 5 – Machines organization

- **Workstations**  
Used by users for daily tasks. Should NOT have privileged sessions.

- **Servers**  
Provide services.

- **Domain Controllers**  
Most sensitive. Store password hashes.

# Task 6 – GPO (Group Policy Objects)

Good organization depends on correct privilege assignment via **GPOs**

Tool: **Group Policy Management**

![[Captura de pantalla 2026-02-26 073244.png]]

- Two key parts:

![[Captura de pantalla 2026-02-26 074112.png]]

    Where GPOs are applied
    Where GPOs are created

- Edit a GPO:

![[Captura de pantalla 2026-02-26 082739.png]]

![[Captura de pantalla 2026-02-26 093924.png]]

- Apply changes immediately:

```
gpupdate /force
```

- Default refresh is 90 minutes → `/force` applies instantly

![[Captura de pantalla 2026-02-26 100358.png]]

- View policy explanation:

![[Captura de pantalla 2026-02-26 100233.png]]

- The way we just made this **domain policy** change is one of two methods. The other can be done through the terminal, but to understand it, we need to understand that **GPM** works with the **SYSVOL** share, which allows the distribution of **GPOs**. Essentially, it's the network through which these changes are executed. As a security best practice, it's recommended that only specific **users** have access to this share. Let's do an exercise:

     SYSVOL path:

```
  \NombreDelDC\SYSVOL\NombreDelDominio\Policies
```

- Example: Restrict Control Panel GPO

![[Captura de pantalla 2026-02-26 105059.png]]

![[Captura de pantalla 2026-02-26 105204 1.png]]

![[Captura de pantalla 2026-02-26 105808.png]]
     We are going to link this to **GPO** to the** Ou's** that intered to us.

With these two GPOs established, we will test them by logging in as a standard user without privileges. First, if we wait five minutes, the session must lock automatically. 

![[Captura de pantalla 2026-02-26 211320.png|190]]

![[Captura de pantalla 2026-02-26 211858.png]]
    The lock is successful.


 ![[Captura de pantalla 2026-02-26 211923.png]] 
     We see that the time range is met

Second, we verify access to the GPM (Group Policy Management). 

![[Captura de pantalla 2026-02-26 204055.png]] Access is successfully blocked as well.

### Task 7: Authentication Methods

**Kerberos (Step-by-Step):** It is the most secure current method and works with a ticket system.

1. **AS-REQ:** The client sends a request to the KDC (AS service) with a timestamp encrypted using their NT Hash.
2. **Validation:** The KDC verifies the hash against its database and generates the **TGT**.
3. **AS-REP:** The KDC sends the TGT back. It includes a **Login Session Key** and the TGT is encrypted with the **KRBTGT** account key.
4. **TGS-REQ:** The client uses the TGT to request a **TGS** (Service Ticket) from the TGS service.
5. **TGS-REP:** The client receives the Service Ticket (encrypted with the target service's hash) and a **Service Session Key**.
6. **AP-REQ:** The client presents the Service Ticket to the target server (e.g., File Server) for final validation.

**Attack Vectors:**

- **Kerberoasting:** Extracting TGS tickets to crack service account passwords (e.g., `sql_svc`). '''The success of this attack depends on password strength. Use long, complex passwords or gMSA.'''
- **KRBTGT (Golden Ticket):** Stealing the KRBTGT hash allows for full domain compromise. '''Domain SID is required to "cook" the perfect Golden Ticket.'''

**NETNTLM (Challenge/Response):** An older, less secure protocol. It lacks timestamps, making it vulnerable to replay attacks.

- **Attack:** **LLMNR/NBT-NS Poisoning** using tools like **Responder** to capture the NetNTLM hash.

### Task 8: Trees, Forests and Trust Relationships

- **Tree:** Two domains sharing the same namespace (e.g., `thm.local`).
- **Forest:** A collection of multiple domains with different namespaces.
- **Trust Relationships:** Connections that allow users from one domain to access resources in another.
    - **One-way:** Domain AAA trusts BBB; BBB users can access AAA resources.
    - **Two-way:** Both domains mutually authorize each other's users.

### Key Points: AD Troubleshooting & OPSEC

Managing active connections correctly is vital to avoid detection by the **Blue Team**.

- **Route Conflict:** Occurs when a new VPN route is created without closing the previous one (**File exists** error).
- **Zombie Interfaces:** When `tun0` hangs in the Kernel, forcing the creation of `tun1` and causing network failure. '''A network admin will see strange tunX interfaces and identify unauthorized VPN activity. "Cleaning the house" is part of stealth.'''

**Cleanup Commands:**

1. Check routes: `ip route`
2. Kill processes: `sudo killall openvpn`
3. Manual delete: `sudo ip link delete tun0`
4. Flush orphan routes: `sudo ip route flush dev tun0`
### Pentesters Tools

**Enumeration:**

- **SMBMap:** Used to audit shared resources and permissions.

```
sbmmap -H IP -u 'user' -p 'password'
```

![[Captura de pantalla 2026-02-25 085756.png]]


- **NetExec (RID Brute Forcing):** To enumerate users and groups without a prior list.

```
netexec smb <IP> -u 'user' -p 'password' --rid brute
```

![[Captura de pantalla 2026-02-25 093215.png]]

**Password Changes & Hash Extraction:**

- **BloodyAD:** To reset passwords from the attacker's terminal.

     We're going to use a tool called **Bloodyad**

```
apt install bloodyad
```

     Usage sintaxis

```
bloodyAD --host <IP> -u <user> -p <pass> set password <new_pass>
```

- **Impacket-secretsdump:** Extracts credentials and hashes from the DC.

```
impacket-secretsdump -just-dc backup@<IP> 
```

- **`impacket-secretsdump`**: This is the tool in the Impacket suite that extracts credentials and hashes from Active Directory.
- **just-dc** : This indicates that only information from the domain controller will be extracted, not from other computers in the domain.
- **`backup@<IP>`**: Here, `backup` is the username (or account) being used to authenticate to the domain controller, and `<IP>` is the IP address of the domain controller you want to connect to.

A valid domain username and password are required.

**Administrator privileges are not required**:

- Although having high privileges facilitates the attack, in many cases, a low-privilege user may be sufficient if the domain controller is misconfigured or if there are vulnerabilities that allow privilege escalation.
