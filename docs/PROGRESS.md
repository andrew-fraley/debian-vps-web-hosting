# Phase 1: Project preparation and Debian baseline.

**Objectives:**  

- Create a git repo (locally, push to Github after project completion) [X]
- Establish documentation structure. [X]  
- Confirm the VPS plan and operation system. [X]  
    - Platform: IONOS  
    - OS: Debian 13  
    - DNS: Cloudflare  
    - Compute: 2vCPU, 4GB, 120 GB NVMe  
- Establish secure administrative access. [X]  
    Documentation below:  
    - Generate SSH key pair  
    - Setup non-root admin user  
- Apply system updates. [X]  
- Configure and verify the firewall. [X]  
    Documentation below:
    - IONOS Firewall
    - Debian Firewall
- Automatic security updates setup. [X]
- Record the initial server configuration. [X]

---

## Process Documenntation 
## Phase 1: SSH Configure and Hardening

### SSH key pair setup

**Generate local SSH key pair**  
The SSH keys replace passwords for server login. Generate the key pair on the local machine, the private key stays with me as proof of identity, the public key is copied to the server and is used to check that proof.   
This Setup is a secure attempt to keep unauthorized access to the server. Bots constantly try passwords against any internet facing server, so this setup bypasses the need for password access.  

    bash
    ssh-keygen -t ed25519 -C "andrew-ionos-vps" -f ~/.ssh/ionos_vps
    - -t ed25519 selects a modern, fast, short key type.  
    - -C sets a comment label for identifying the key later.  
    - -f sets the filename. Using a dedicated key per server limits the damage if one is lost.  

After running the command it asks for a passphrase, set one and store it appropriately. This encrypts the private key file, so a stolen key file is still useless.

    Verification command:
    bash
    ls -l ~/.ssh/ionos_vps*
    ssh-keygen -lf ~/.ssh/ionos_vps.pub

This should show two files: ionos_vps (private, permissions -rw-------) and ionos_vps.pub (public)  
**The key pair is now generated and verified.**

### Server login and user setup

*Note: to avoid lockout of server keep the root session open until final test passes.* 

**Step A: Log in as root** (On local machine)  

    bash
    ssh root@<VPS-IP>

This prompts for initial connection and shows the server's host fingerprint with a prompt to continue. This is a Verification method that the server you are connecting to is the right server, verify the fingerprint if the provider provides it. It stores the answer in ~/.ssh/known_hosts to detect impersonation later.  
It will then prompt for the password (provided by the hosting provider in your control panel or somewhere).  

Verify the machine is the right server:

    bash
    head -3 /etc/os-release
    command -v sudo

This verifies the operating system and if sudo is installed. 

**Step 2: Create the admin user** (On VPS)  

    bash 
    adduser <adminuser>
    usermod -aG sudo <adminuser>
    id <adminuser>


adduser creates the account and home directory and prompts for a strong password.  
usermod -aG sudo appends the user to the sudo group.  
-a keeps the user from being removed from all other groups.

*Note: This user is the main user that will be used going forward for daily work using sudo for root level commands to protect root access and also sudo leaves an auditable trail for investigation.*

**Step C: Install the public key** (On local machine)  

    bash
    ssh-copy-id -i ~/.ssh/ionos_vps.pub <adminuser>@<VPS-IP>

This logs in with the new user's password and adds your public key to ~/.ssh/authorized_keys on the server.  
Test the key login.  

    bash 
    ssh -i ~/.ssh/ionos_vps <adminuser>@<VPS-IP>
    sudo whoami

This should prompt for the key passphrase, not the account password. sudo whoami should print root.  

*Note: if anything fails, terminal 1 should still be logged in as root to the VPS.*

---

### System update

Log in as the admin user and run:  
    
    bash 
    sudo apt updates
    sudo apt full-upgrade

Then reboot to load any new kernel:  
    
    bash 
    sudo apt reboot

Wait a moment, reconnect, and verify.

    bash
    uptime -p 
    uname -r
    sudo apt update && apt list --upgradable

---

### SSH Hardening

This process is to ensure unauthorized access to the VPS is blocked through user/password credential login attempts to combat bots pursuing brute-force password logins to users and root on the public facing VPS.  

*Note: this is the step with risk of lock out. To stay safe while hardening the SSH process, keep the current admin session open the whole time. Test from a second new terminal before closing anything.*

**Step A: Check for existing overrides** (On VPS session)  

    bash
    ls /etc/ssh/sshd_config.d/
    sudo grep -riE "PasswordAuthentication|PermitRootLogin" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/

Debian’s sshd_config reads every file in sshd_config.d/, and for each setting the first value found wins. A provider file such as 50-cloud-init.conf may set PasswordAuthentication yes. We’ll name ours 00-hardening.conf so it’s read first and wins.

**Step B: Create the drop-in file** (On VPS session)  

    bash
    sudo nvim /etc/ssh/sshd_config.d/00-hardening.conf

    PermitRootLogin no
    PasswordAuthentication no
    KbdInteractiveAuthentication no
    PubkeyAuthentication yes

- PermitRootLogin no blocks logging in as root over SSH.  
- PasswordAuthentication no allows keys only.  
- KbdInteractiveAuthentication no closes a second password-style path that also bypasses the previous setting.  
- PubkeyAuthentication yes makes explicit the one method we depend on.  

**Step C: Validate, then reload**  

    bash
    sudo sshd -t
    sudo systemctl reload ssh

sshd -t checks syntax and prints nothing if the config is valid. Don't reload if it prints an error.

After reloading the system ssh config, confirm that the config from above is in effect.

    bash
    sudo sshd -T | grep -iE "permitrootlogin|passwordauthentication|kbdinteractive|pubkeyauthentication"

This should output the current configurations and they should be what was added above. 

**Step D: Test from a new terminal on local machine**   

*Note: Keep the initial terminal connected to the server for backup access in case the ssh connection is locked.*
    
    bash
    1. Key login should work:
    ssh -i ~/.ssh/ionos_vps <adminuser>@<VPS-IP>

    2. Password login should be refused:
    ssh -o PubkeyAuthentication=no <adminuser>@<VPS-IP>

    3. Root login should be refused:
    ssh -i ~/.ssh/ionos_vps root@<VPS-IP>

Expected for test 1 to succeed, and tests 2 and 3 to return Permission denied (publickey)
If this test fails don't close the old session. Remove the 00-hardening.conf file from the VPS session and run sudo systemctl reload ssh. Then from a new terminal reconnect and repeat Steps above and test. If persistant issues persist check with the hosting provider for known or intended issues for ssh access.  

---

## Firewall Setup


### IONOS Firewall 

**Step A: Baseline** (On local machine)  

    bash
    nc -vz -w 5 <VPS-IP> 22
    nc -vz -w 5 <VPS-IP> 80
    nc -vz -w 5 <VPS-IP> 8080

Expect 22 to succeed and ports 80 and 8080 should have connection refused, meaning the packet reached the server and nothing is listening.   

**Step B: Create the policy** (IONOS control panel)

    Protocol Port Description
    TCP	22	SSH
    TCP	80	HTTP
    TCP	443	HTTPS
    UDP	443	Caddy HTTP/3


These are configured in the respective provider control panel or platform. IONOS is what this project uses, and these are the baseline rules needed.

**Step C: Assign Rules to VPS**

Verify the rules are set and assigned to the VPS.  

**Step D: Verify** 

Rerun the above three nc commands adding one more check for port 443, then open a new terminal and verify SSH connection is still available.  

Expect 22 to succeed, 80 and 443 to show connection refused, 8080 now times out instead of being refused giving evidence that the firewall rules are working.  

---

Note on the two firewalls, IONOS firewall sites in front of the server, the Debian one runs on the VPS machine itself. This gives redundancy to our Firewall rules across two layers in case of compromise on the provider firewall layer or the host firewall layer.

---

### Debian Firewall

Tool choice: ufw "uncomplicated firewall", this is a simple frontend for the kernel's packet filter fitting the project philosophy of preferring simple rule. 


**Step A: Install and set defaults** (On VPS)

    bash
    sudo apt install ufw
    sudo ufw default deny incoming
    sudo ufw default allow outgoing

The firewall is not active yet, these commands just set the policy: drop all inbound traffic that is not allowed, and let the server make its own outbound conntections (updates, certificate renewal).  

**Step B: Add rules, then review** (On VPS)  

    bash
    sudo ufw allow 22/tcp comment 'SSH'
    sudo ufw allow 80/tcp comment 'HTTP'
    sudo ufw allow 443/tcp comment 'HTTPS'
    sudo ufw allow 443/udp comment 'Caddy HTTP/3'
    sudo ufw show added

show added lists the rules enetered without enabling anything. First check 22/tcp is in the list before continuing.  

**Step C: Enable and verify** 

    bash
    sudo ufw enable  
    sudo ufw status verbose

ufw warns about disruption to SSH connections, answer y only if the 22 rule was verified in the above verification.  

The status should show Status: active, Default: deny (incoming), allow (outgoing), and the four rules set above.  

Now open a new terminal and SSH in, rerun the above nc tests from the local machine and verify that 22 connects, 80 and 443 are refused, and 8080 time out.  

---

## Automatic Security Updates Setup

*Note: Tool pick, unattended-upgrades installs security updates atomatically and is simple to setup.*

### Setup

**Step A: Install and enable** (On VPS)  

    bash
    sudo apt install unattended-upgrades
    sudo dpkg-reconfigure -plow unattended-upgrades

Answer Yes at the prompt. This writes /etc/apt/apt.conf.d/20auto-upgrades turning on the daily update check and install.

**Step B: Verify it's configured**

    bash
    cat /etc/apt/apt.conf.d/20auto-upgrades
    systemctl list-timers 'apt-daily*'
    sudo unattended-upgrade --dry-run --debug 

Expect the file contains APT::Periodic::Update-Package-Lists "1"; and APT::Periodic::Unattended-Upgrade "1";.  
Two timers are listed, apt-daily.timer and apt-daily-upgrade.timer, each with a next run time.  
The dry run ends without errors. Look for "Allowed orgins" naming the Debian security repository.  

---
