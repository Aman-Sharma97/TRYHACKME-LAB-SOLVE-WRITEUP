# Road - TryHackMe CTF Writeup

**Platform:** TryHackMe
**Room Name:** Road
**Difficulty:** Medium
**Category:** Web Exploitation / Privilege Escalation

---

# Initial Enumeration

I kicked things off with an nmap scan to see what services were running on the target. Command used: `nmap -T4 -sV -v $IP`

The scan came back with two open ports: 22 (SSH) and 80 (HTTP).



# Web Enumeration

Browsing to the web server, I found it was hosting a full web application. Poking around the various links on the site, I came across a login page.



The site allowed new user registration, so I signed up for an account and checked out the profile page. Since the backend was built in PHP, my plan was to try uploading a reverse shell disguised as a profile picture. However, when I got to the upload section, I noticed something off — only admin accounts were permitted to upload a profile picture, and even more interesting, the admin's username was visible right there on the page. A pretty serious information leak.



Digging further into the site's functionality, I spotted a "ResetUser" feature. That immediately made me think of parameter tampering, so I fired up BurpSuite, turned on the intercept proxy, and submitted a password reset to see exactly what was being sent to the server.



Looking at the intercepted request, I could see it contained a username field along with the new password. So I tried swapping out the username value with the admin account name I'd discovered earlier.



Sure enough, submitting the request with the admin's username went through without any server-side validation catching it — I had just reset the admin account's password.



With the new password in hand, I tried logging in as admin.



It worked. From there I headed back to the profile section to upload my reverse shell through the profile picture upload function. Before doing that, I checked the page source to figure out where uploaded images actually get stored on the server.



Once I had my reverse shell crafted and uploaded, I started a netcat listener and then browsed directly to the file's hosted location to trigger it.



The shell connected back successfully — I was in.




# Initial Access

Now logged in as the `dub-data` user, it was time to enumerate other accounts on the box. Reading through `/etc/passwd`, I found a regular user account along with a system account that looked worth investigating further.



# User.txt Flag

Checking the user's home directory turned up the first flag.



# Privilege Escalation

After upgrading to a proper interactive shell, I moved into `/tmp` and confirmed I had write access there — a good staging spot for tools.



With write access confirmed, I transferred linpeas.sh onto the box to automate the hunt for privilege escalation paths.



I made the script executable with `chmod +x linpeas.sh` and ran it. The output flagged the system as vulnerable to CVE-2021-4034 (the Polkit "PwnKit" vulnerability).



I recalled using a solid exploit toolkit for this exact CVE in a previous CTF, built by Ly4k. I grabbed it with:
`curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit`

After uploading it the same way I had linpeas, making it executable, and running it, I landed a root shell.



# Final Thoughts

Overall this was an enjoyable box to work through. After getting root, I kept exploring the system a bit more out of curiosity and noticed MongoDB running — something I hadn't worked with before. Digging into it ended up revealing credentials for a user account I'd already identified.

# MongoDB

I worked through MongoDB's built-in help to figure out the commands needed to enumerate the databases present, and found the following:


Running further commands against the database uncovered a stored username and password pair.



Testing those credentials against SSH confirmed they worked — giving me a second way into the server.



# Wrapping Up

This room was a great learning experience, especially since it was my first real hands-on exposure to MongoDB. Figuring out the syntax on the fly was a good reminder that there's always something new to pick up. Looking forward to tackling more CTF writeups — feel free to drop any recommendations for rooms to try next!
