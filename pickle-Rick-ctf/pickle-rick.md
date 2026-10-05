# Pickle Rick

*Web enumeration, command execution, and ingredient discovery*

**Platform:** TryHackMe | **Difficulty:** Easy | **Room:** Picklerick

---

## Scope and environment

This write-up presents the approach used to complete the authorized TryHackMe Pickle Rick lab. The target is shown as `<MACHINE_IP>` because the room provides a temporary IP address whenever the machine is started.

## Reconnaissance

I started by running a simple Nmap scan to determine which TCP services were exposed by the target.

```bash
nmap <MACHINE_IP>
```

The scan revealed two accessible ports: 22/tcp running SSH and 80/tcp serving HTTP. Since the challenge is centered around a web application, I concentrated on the HTTP service first.

![Initial Nmap scan displaying ports 22 and 80.]
<img width="1016" height="298" alt="01-nmap-scan" src="https://github.com/user-attachments/assets/be5bbf9c-be73-43c8-bb8c-313f0f70ba3d" />

*Evidence - Initial Nmap scan displaying ports 22 and 80.*


## Web enumeration

### HTML source inspection

I visited the web application and reviewed its HTML source. A developer comment in the source code revealed a username that could be useful for logging in.

**Username:** `R1ckRul3s`

![Username found inside an HTML comment.]c:\tryhackme-pickle-rick-writeup\images\02-html-source-username.png
*Evidence - Username found inside an HTML comment.*


### Directory enumeration

I ran Gobuster with PHP, HTML, and TXT extensions to discover files and directories that were not directly exposed through the main page.

```bash
gobuster dir -u http://<MACHINE_IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

The enumeration returned `login.php`, `robots.txt`, and `portal.php` among other resources. The 302 redirect from `portal.php` to `login.php` suggested that the command interface was protected by authentication.

![Results returned by the Gobuster enumeration.]
c:\tryhackme-pickle-rick-writeup\images\03-gobuster-results.png
*Evidence - Results returned by the Gobuster enumeration.*


### Credential discovery in robots.txt

Before trying the login form, I checked `robots.txt`. Rather than containing only standard crawler rules, it exposed a single string. I considered that value a possible password for the username obtained from the source code.

**Password candidate:** `Wubbalubbadubdub`

![Password candidate exposed through robots.txt.]
c:\tryhackme-pickle-rick-writeup\images\04-robots-txt.png
*Evidence - Password candidate exposed through robots.txt.*


## Authentication

I entered the discovered username and password into `login.php`. The credentials were accepted, and the application redirected to the command panel at `portal.php`.

**Credentials:** `R1ckRul3s` / `Wubbalubbadubdub`

![Discovered credentials submitted through the portal login page.]c:\tryhackme-pickle-rick-writeup\images\05-login-page.png
*Evidence - Discovered credentials submitted through the portal login page.*


## Command execution and the first ingredient

Once authenticated, the portal allowed operating-system commands to be executed. I began by listing the files in the current web directory.

```bash
ls
```

![Files displayed from the current web directory.]c:\tryhackme-pickle-rick-writeup\images\06-web-directory-listing.png
*Evidence - Files displayed from the current web directory.*


The directory listing contained `Sup3rS3cretPickl3Ingred.txt`. I used `less` to read the contents of this file.

```bash
less Sup3rS3cretPickl3Ingred.txt
```

![Contents returned from Sup3rS3cretPickl3Ingred.txt.]

c:\tryhackme-pickle-rick-writeup\images\07-first-ingredient.png
*Evidence - Contents returned from Sup3rS3cretPickl3Ingred.txt.*


**First ingredient:** 

## Locating the second ingredient

After locating the first ingredient, I continued with filesystem enumeration by checking the root directory and then examining the directories inside `/home`.

```bash
ls /
ls /home
ls /home/rick
```

![Listing of the root filesystem.]c:\tryhackme-pickle-rick-writeup\images\08-root-filesystem.png
*Evidence - Listing of the root filesystem.*

![User directories discovered under /home.]c:\tryhackme-pickle-rick-writeup\images\09-home-directory.png
*Evidence - User directories discovered under /home.*

![The file containing the second ingredient under /home/rick.]c:\tryhackme-pickle-rick-writeup\images\10-rick-home.png
*Evidence - The file containing the second ingredient under /home/rick.*


Because the filename includes a space, I placed the complete filename inside double quotes so the shell would interpret it as a single argument.

```bash
less /home/rick/"second ingredients"
```

![Contents retrieved from the second ingredients file.]c:\tryhackme-pickle-rick-writeup\images\11-second-ingredient.png
*Evidence - Contents retrieved from the second ingredients file.*


**Second ingredient:** 

## Privilege context and the final ingredient

Before attempting to inspect `/root`, I first verified which operating-system account was being used to execute commands through the web portal.

```bash
whoami
```

The result was `www-data`, confirming that the command panel was operating under the web-service account rather than a normal user account.

![whoami output showing that portal commands execute as www-data.]c:\tryhackme-pickle-rick-writeup\images\12-whoami.png
*Evidence - whoami output showing that portal commands execute as www-data.*


I then checked whether this web-service account had permission to access privileged resources by running `sudo ls /root`. The command worked without asking for a password, indicating that `www-data` could use sudo for privileged access. The original evidence does not include a full `sudo -l` result, so the precise sudo policy was not established.

```bash
sudo ls /root
```

![/root successfully listed with elevated privileges.]c:\tryhackme-pickle-rick-writeup\images\13-root-directory.png
*Evidence - /root successfully listed with elevated privileges.*


The privileged directory listing showed `3rd.txt`. I used the available sudo access to read the file.

```bash
sudo less /root/3rd.txt
```

![Contents of /root/3rd.txt.]c:\tryhackme-pickle-rick-writeup\images\14-third-ingredient.png
*Evidence - Contents of /root/3rd.txt.*


**Final ingredient:** 

## Answer summary

The three room questions were completed in the same sequence as the three ingredients were located.



| First   `Sup3rS3cretPickl3Ingred.txt`   (web directory) 
| Second  `/home/rick/second ingredients` 
| Final   `/root/3rd.txt` 

The final answers remain intentionally redacted in this public version.

## Security findings

- Username exposed through an HTML source comment
- Sensitive data revealed through `robots.txt`
- Authenticated operating-system command execution was available
- The web-service account had excessive sudo permissions
- The web application was not adequately isolated from privileged system resources

## Remediation

- Remove usernames and other sensitive details from HTML comments.
- Do not place passwords, credentials, or secrets inside `robots.txt`.
- Avoid sending user-controlled input directly to operating-system commands.
- Use narrowly defined server-side functionality instead of unrestricted shell execution.
- Run the web application with only the privileges it actually requires.
- Remove unnecessary passwordless sudo permissions from `www-data`.

## Lessons learned

This room shows how several small security weaknesses can be combined into a complete attack path. Information disclosure revealed usable credentials, the authenticated command panel allowed filesystem discovery, and excessive sudo permissions ultimately exposed a file owned by root.
