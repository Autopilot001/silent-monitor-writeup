# Silent Monitor — Notes to Myself

**Mohamed Nabil Algamoudi**  
**TryHackMe room:** Silent Monitor — Junior Penetration Testing

These are my notes from completing the room. I replaced the instance address with TARGET_IP because TryHackMe assigns a different address each time. I have left out every flag and password.

## 1. Start the target and connect

I started the target machine and AttackBox, connected to the TryHackMe VPN, and copied the assigned address into my notes. Before running a command, replace TARGET_IP below with the address for the active instance.

## 2. Enumerate all TCP ports

I ran a full TCP scan with service detection and the default scripts:

    sudo nmap -sC -sV -p- -Pn TARGET_IP

The scan showed SSH on port 22 and a Werkzeug web server on port 5050. The HTTP title identified the site as the CorpNet Network Operations Centre. My next step was to understand the web service before trying credentials elsewhere.

## 3. Find the internal page

I used Gobuster against port 5050:

    gobuster dir -u http://TARGET_IP:5050 -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt

The scan found /internal with a 200 response. I opened the page and checked the login behavior. Invalid credentials returned an “Invalid username or password” message.

I started down a Hydra route, but that was unnecessary. Note to myself: test the application behavior before spending time on password enumeration. I tried this username instead:

    admin' OR 1=1--

The login accepted it. That was the SQL injection foothold.

## 4. Follow the health-check clue

After logging in, I checked /internal/health and the application logs. One log entry contained an encoded newline followed by whoami:

    127.0.0.1%0awhoami

The input did not behave the way I expected through the page. I intercepted and replayed the request in Burp Suite, where I got the diagnostic response I was looking for. I did not pin down why the browser request differed. Note to myself: compare the raw requests next time instead of guessing.

## 5. Confirm the diagnostic behavior

I tested the ping command with a version check:

    ping -c 2 -W 1 127.0.0.1 -V

The response showed iputils version 20211215. This confirmed that the health-check input reached command execution.

## 6. Inspect the application files

I listed the files and found app.py and secret.config. The Python file was the backend; secret.config exposed the backup-agent account and showed that it ran as sysadmin. The configuration also contained a password. I am not copying that password into this write-up.

## 7. Use the credentials over SSH

I tried the recovered sysadmin credentials against the SSH service:

    ssh sysadmin@TARGET_IP

SSH accepted them and gave me a shell on the target. I used the address and host-key prompt shown by my own active room instance.

## 8. Check the account and backup directory

I checked sudo permissions:

    sudo -l

sysadmin could not run sudo. I listed the home directory and found backups and user.txt. Inside backups were README.txt and infrastructure.kdbx.

## 9. Work through the KeePass database

I used keepass4crack.py (credited to unix-ninja) to recover the KeePass database password. I am leaving the recovered password out of these notes. Using the database information, I completed the final escalation, obtained root access, and checked /root/root.txt to confirm completion. I have not included any flag values.

## What I would remember next time

- Skip Hydra for this room; the login input test exposed the useful weakness sooner.
- Keep the Burp request and response when the browser behavior is unclear.
- Treat configuration files and backup databases as sensitive; do not copy recovered secrets into a public write-up.
- Use TARGET_IP only as a placeholder. Get the real address from the current TryHackMe instance.

## Remediation notes

The login should use parameterized queries. The health-check should not pass user input to a shell. Service credentials should be kept out of configuration files, rotated if exposed, and restricted to the access the account needs. Backup databases should be protected with strong, unique credentials.

## Screenshots

I have not added screenshots to this copy. I will add captures from my own run when available, with the target address, passwords, and flags removed.

---

Mohamed Nabil Algamoudi
