# Phase 1 Project Preparation, SSH Hardening, Firewalls, Security Updates

**Objectives:**

- Create a git repo (locally, push to Github after project completion). [X]
- Establish documentation structure. [X]
- Confirm the VPS plan and operation system. [X]
    - Platform: IONOS
    - OS: Debian 13
    - DNS: Cloudflare
    - Compute: 2vCPU, 4GB, 120 GB NVMe
- Establish secure administrative access. [X]
    - Documentation below:
        - Generate SSH key pair
        - Setup non-root admin user
- Apply system updates. [X]
- Configure and verify the firewall. [X]
    - Documentation below:
        - IONOS Firewall
        - Debian Firewall
- Automatic security updates setup. [X]
- Record the initial server configuration. [X]

---

## Process Documentation Phase 1

### SSH Key Pair Setup

**Generate local SSH key pair**

The SSH keys replace passwords for server login. Generate the key pair on the local machine, the private key stays with me as proof of identity, the public key is copied to the server and is used to check that proof.

This Setup is a secure attempt to keep unauthorized access to the server. Bots constantly try passwords against any internet facing server, so this setup bypasses the need for password access.

```bash
ssh-keygen -t ed25519 -C "example-ionos-vps" -f ~/.ssh/ionos_vps
```

- `-t ed25519` selects a modern, fast, short key type.
- `-C` sets a comment label for identifying the key later.
- `-f` sets the filename. Using a dedicated key per server limits the damage if one is lost.

After running the command it asks for a passphrase, set one and store it appropriately. This encrypts the private key file, so a stolen key file is still useless.

Verification command:

```bash
ls -l ~/.ssh/ionos_vps*
ssh-keygen -lf ~/.ssh/ionos_vps.pub
```

This should show two files: ionos_vps (private, permissions -rw-------) and ionos_vps.pub (public)

**The key pair is now generated and verified.**

### Server Login and User Setup

*Note: to avoid lockout of server keep the root session open until final test passes.*

**Step A: Log in as root** (On local machine)

```bash
ssh root@<VPS-IP>
```

This prompts for initial connection and shows the server's host fingerprint with a prompt to continue. This is a Verification method that the server you are connecting to is the right server, verify the fingerprint if the provider provides it. It stores the answer in `~/.ssh/known_hosts` to detect impersonation later.

It will then prompt for the password (provided by the hosting provider in your control panel or somewhere).

Verify the machine is the right server:

```bash
head -3 /etc/os-release
command -v sudo
```

This verifies the operating system and if sudo is installed.

**Step B: Create the admin user** (On VPS)

```bash
adduser <adminuser>
usermod -aG sudo <adminuser>
id <adminuser>
```

adduser creates the account and home directory and prompts for a strong password.

usermod -aG sudo appends the user to the sudo group.

-a keeps the user from being removed from all other groups.

*Note: This user is the main user that will be used going forward for daily work using sudo for root level commands to protect root access and also sudo leaves an auditable trail for investigation.*

**Step C: Install the public key** (On local machine)

```bash
ssh-copy-id -i ~/.ssh/ionos_vps.pub <adminuser>@<VPS-IP>
```

This logs in with the new user's password and adds your public key to `~/.ssh/authorized_keys` on the server.

Test the key login.

```bash
ssh -i ~/.ssh/ionos_vps <adminuser>@<VPS-IP>
sudo whoami
```

This should prompt for the key passphrase, not the account password. sudo whoami should print root.

*Note: if anything fails, terminal 1 should still be logged in as root to the VPS.*

---

### System Update

Log in as the admin user and run:

```bash
sudo apt update
sudo apt full-upgrade
```

Then reboot to load any new kernel:

```bash
sudo reboot
```

Wait a moment, reconnect, and verify.

```bash
uptime -p
uname -r
sudo apt update && apt list --upgradable
```

---

### SSH Hardening

This process is to ensure unauthorized access to the VPS is blocked through user/password credential login attempts to combat bots pursuing brute-force password logins to users and root on the public facing VPS.

*Note: this is the step with risk of lock out. To stay safe while hardening the SSH process, keep the current admin session open the whole time. Test from a second new terminal before closing anything.*

**Step A: Check for existing overrides** (On VPS session)

```bash
ls /etc/ssh/sshd_config.d/
sudo grep -riE "PasswordAuthentication|PermitRootLogin" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Debian's sshd_config reads every file in `sshd_config.d/`, and for each setting the first value found wins. A provider file such as `50-cloud-init.conf` may set PasswordAuthentication yes. We'll name ours `00-hardening.conf` so it's read first and wins.

**Step B: Create the drop-in file** (On VPS session)

```bash
sudo nvim /etc/ssh/sshd_config.d/00-hardening.conf
```

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

- PermitRootLogin no blocks logging in as root over SSH.
- PasswordAuthentication no allows keys only.
- KbdInteractiveAuthentication no closes a second password-style path that also bypasses the previous setting.
- PubkeyAuthentication yes makes explicit the one method we depend on.

**Step C: Validate, then reload**

```bash
sudo sshd -t
sudo systemctl reload ssh
```

sshd -t checks syntax and prints nothing if the config is valid. Don't reload if it prints an error.

After reloading the system ssh config, confirm that the config from above is in effect.

```bash
sudo sshd -T | grep -iE "permitrootlogin|passwordauthentication|kbdinteractive|pubkeyauthentication"
```

This should output the current configurations and they should be what was added above.

**Step D: Test from a new terminal on local machine**

*Note: Keep the initial terminal connected to the server for backup access in case the ssh connection is locked.*

```bash
# 1. Key login should work
ssh -i ~/.ssh/ionos_vps <adminuser>@<VPS-IP>

# 2. Password login should be refused
ssh -o PubkeyAuthentication=no <adminuser>@<VPS-IP>

# 3. Root login should be refused
ssh -i ~/.ssh/ionos_vps root@<VPS-IP>
```

Expected for test 1 to succeed, and tests 2 and 3 to return Permission denied (publickey).
If this test fails don't close the old session. Remove the `00-hardening.conf` file from the VPS session and run sudo systemctl reload ssh. Then from a new terminal reconnect and repeat Steps above and test. If persistent issues persist check with the hosting provider for known or intended issues for ssh access.

---

### Firewall Setup

#### IONOS Firewall

**Step A: Baseline** (On local machine)

```bash
nc -vz -w 5 <VPS-IP> 22
nc -vz -w 5 <VPS-IP> 80
nc -vz -w 5 <VPS-IP> 8080
```

Expect 22 to succeed and ports 80 and 8080 should have connection refused, meaning the packet reached the server and nothing is listening.

**Step B: Create the policy** (IONOS control panel)

| Protocol | Port | Description |
|---|---|---|
| TCP | 22 | SSH |
| TCP | 80 | HTTP |
| TCP | 443 | HTTPS |
| UDP | 443 | Caddy HTTP/3 |

These are configured in the respective provider control panel or platform. IONOS is what this project uses, and these are the baseline rules needed.

**Step C: Assign Rules to VPS**

Verify the rules are set and assigned to the VPS.

**Step D: Verify**

Rerun the above three nc commands adding one more check for port 443, then open a new terminal and verify SSH connection is still available.

Expect 22 to succeed, 80 and 443 to show connection refused, 8080 now times out instead of being refused giving evidence that the firewall rules are working.

---

Note on the two firewalls, IONOS firewall sits in front of the server, the Debian one runs on the VPS machine itself. This gives redundancy to our Firewall rules across two layers in case of compromise on the provider firewall layer or the host firewall layer.

---

#### Debian Firewall

Tool choice: ufw "uncomplicated firewall", this is a simple frontend for the kernel's packet filter fitting the project philosophy of preferring simple rule.

**Step A: Install and set defaults** (On VPS)

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

The firewall is not active yet, these commands just set the policy: drop all inbound traffic that is not allowed, and let the server make its own outbound connections (updates, certificate renewal).

**Step B: Add rules, then review** (On VPS)

```bash
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw allow 443/udp comment 'Caddy HTTP/3'
sudo ufw show added
```

show added lists the rules entered without enabling anything. First check 22/tcp is in the list before continuing.

**Step C: Enable and verify**

```bash
sudo ufw enable
sudo ufw status verbose
```

ufw warns about disruption to SSH connections, answer y only if the 22 rule was verified in the above verification.

The status should show Status: active, Default: deny (incoming), allow (outgoing), and the four rules set above.

Now open a new terminal and SSH in, rerun the above nc tests from the local machine and verify that 22 connects, 80 and 443 are refused, and 8080 time out.

---

### Automatic Security Updates Setup

*Note: Tool pick, unattended-upgrades installs security updates automatically and is simple to setup.*

#### Setup

**Step A: Install and enable** (On VPS)

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

Answer Yes at the prompt. This writes `/etc/apt/apt.conf.d/20auto-upgrades` turning on the daily update check and install.

**Step B: Verify it's configured**

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
systemctl list-timers 'apt-daily*'
sudo unattended-upgrade --dry-run --debug
```

Expect the file contains APT::Periodic::Update-Package-Lists "1"; and APT::Periodic::Unattended-Upgrade "1";.

Two timers are listed, `apt-daily.timer` and `apt-daily-upgrade.timer`, each with a next run time.

The dry run ends without errors. Look for "Allowed origins" naming the Debian security repository.

---

# Phase 2 First Hugo Website

**Objectives:**

- Install Caddy. [X]
- Establish the website directory convention. [X]
- Deploy a Hugo-generated site. [X]
- Configure domain routing. [X]
    - Documentation below:
        - Domain routing part 1 (DNS records) (first step of Phase 2)
        - Domain routing part 2 (Caddyfile setup)
- Verify external access and HTTPS. [X]
- Inspect service status and logs. [X]

---

## Process Documentation Phase 2

### Domain Routing Part 1: DNS Records in Cloudflare

**Step A: Look what's there**

Inspect the cloudflare domains dashboard for the website to host on the VPS.

Select the domain, got to DNS > Records. Note what exists in the records.

**Step B: Add the records**

| Type | Name | Content | Proxy status |
|---|---|---|---|
| A | @ | VPS IPv4 address | DNS only |
| CNAME | www | domain name | DNS only |

- @ means the bare domain.
- the www CNAME makes www follow the apex, so you only maintain one address.
- leave TTL on Auto.

DNS is only set for now, no proxy for setup with Caddy, proxy can be setup later for added CDN and DDoS protection.

**Step C: Verify from local machine**

```bash
dig +short example.com @1.1.1.1
dig +short www.example.com @1.1.1.1
```

Both commands should return the VPS server's IPv4 address.

---

### Install Caddy

Caddy is the web server. It serves the site files, redirects HTTP to HTTPS, and requests and renews Let's Encrypt certificates on its own.

**Step A: Install Caddy** (On VPS)

```bash
sudo apt install caddy
```

Inspect the package list and confirm, the package creates a caddy user and a systemd service and starts it.

**Step B: Verify** (On VPS)

```bash
caddy version
systemctl status caddy --no-pager
sudo ss -tlnp | grep caddy
id caddy
ls -l /etc/caddy
```

Expected output

- Current Debian version.
- active (running) and enabled in the status. (enabled means it starts on boot)
- Caddy listening on ports 80 and 443. It also listens on `127.0.0.1:2019`, this is the local admin API that stays reachable only on the server itself.
- a caddy user and a Caddyfile in `/etc/caddy/`.

Then verify on local machine:

```bash
curl -I http://<VPS-IP>
```

This should get an HTTP/1.1 200 OK from Caddy's placeholder page. This confirms the whole path: IONOS firewall, ufw, Caddy.

---

### Directory Convention and Permissions

This convention allows for multiple sites to share the same pattern with directory locations, and file permissions. Caddy needs to be able to read the site files but never write to them, this keeps site files secure in the event of the web server being compromised.

**The convention**

```text
/var/www/<domain>/  - this is one directory per site, named after its domain.
```

- `/var/www` is Debian's traditional location for web content.
- naming each directory after the domain name makes it obvious where each site is.
- the content of `/public` in the Hugo site project goes into this directory.

**Permission Model**

- Owner: the admin user. Deploy the sites as this user, so no root is needed for routine updates.
- Group: www-data. Caddy's user is already in this group, this is verified through the "id caddy" command output.
- Mode: 755 for directories and 644 for files. Anyone can read them and only the owner can write. This is the correct permissions for public website content, Caddy gets read access and nothing more.

**Step A: Create the Directory** (On VPS)

```bash
sudo mkdir -p /var/www/example.com
sudo chown <adminuser>:www-data /var/www/example.com
sudo chmod 755 /var/www/example.com
```

- mkdir -p creates the directory and parent directories as needed.
- chown user:group sets the owner and group.
- chmod 755 gives permission for the owner to read, write, and enter, while group and others permissions are read and enter. For directories, "execute" means the right to enter.

**Step B: Add a test page**

```bash
echo '<h1>example.com: server test</h1>' > /var/www/example.com/index.html
```

This is a simple placeholder to test serving before putting a real Hugo project on the server.

**Step C: Verify**

```bash
ls -ld /var/www /var/www/example.com
ls -l /var/www/example.com
namei -l /var/www/example.com/index.html
sudo -u caddy cat /var/www/example.com/index.html
sudo -u caddy touch /var/www/example.com/should-fail
```

Expected output:

- Directory owned by `<adminuser>:www-data` with drwxr-xr-x.
- namei -l lists the path component by component showing the perms for each directory, every directory in the chain needs to allow others to enter or Caddy cannot reach the file.
- cat as caddy prints the page, verifying Caddy can read it.
- touch as caddy fails with Permission denied, verifying Caddy is read-only. This is a critical check to make sure the public facing websites on the server will be read only.

*Troubleshooting note, "403" usually means Caddy is not able to traverse a parent directory in the path or read a file, "namei -l" command is a good tool to find the culprit.*

---

### Deploy the Hugo Site

Hugo only generates files, so the server never runs Hugo, it just hosts the rendered output. Deploying a Hugo site requires a local project build and copy the `public/` results into the VPS domain directory (`/var/www/example.com`).

This is the procedure for deploying and applying updates to each site, local build then VPS deployment of the projects public files.

**Step A: Make SSH convenient** (On local machine)

Add this to `~/.ssh/config` creating the file if needed:

```text
Host ionos-vps
    HostName <VPS-IP>
    User <adminuser>
    IdentityFile ~/.ssh/ionos_vps
```

This lets ssh ionos-vps command replace the longer ssh command to connect to the IONOS VPS. Use chmod 600 `~/.ssh/config` if SSH complains about perms.

**Step B: Make sure rsync exists on both ends**

```bash
rsync --version | head -1
ssh ionos-vps 'rsync --version | head -1'
```

This is verifying rsync tool is installed on both machines, if it is not run the sudo apt install rsync command where it is missing. rsync copies only what changed when doing transfers, making it a better choice than scp for repeated deployments and synchronization tasks.

*rsync is a file transfer and synchronization utility used between local and remote hosts. Similar to scp (secure copy) which is a simpler file transfer utility, but it copy full files every transfer, not checking for partials. This is why rsync is a better choice for this project since it will be used for synchronization and file transfer sometimes not requiring full file transfers and just partial synchronization.*

**Step C: Build the site locally**

In the Hugo project's directory on the local machine:

```bash
hugo --minify
ls public/
```

- --minify shrinks HTML, CSS, and JS.

Before copying anything over, verify `public/` has no drafts, secrets, or private notes since this directory will be fully exposed to the public facing internet.

**Step D: Dry run, then deploy** (On local machine)

```bash
rsync -avzn --delete --chmod=D755,F644 public/ ionos-vps:/var/www/example.com/
```

The n makes it a dry run, it lists what would happen without doing it. This should output the site files and delete the old `index.html` test page made earlier. If it looks right run it without the n.

```bash
rsync -avz --delete --chmod=D755,F644 public/ ionos-vps:/var/www/example.com/
```

- a preserves structure and timestamps, -v is verbose, -z compresses in transit.
- --chmod=D755,F644 forces the permission model from the above directory and permissions conventions.
- Note that the trailing slash on the `public/` directory is important, it allows for the transfer of the `public/` directory contents not the directory plus the contents.

**Step E: Verify**

```bash
ssh ionos-vps 'ls -l /var/www/example.com | head'
ssh ionos-vps 'find /var/www/example.com -type f | wc -l'
find public -type f | wc -l
ssh ionos-vps 'sudo -u caddy cat /var/www/example.com/index.html | head -5'
```

Expect the two file counts match, `index.html` and the assets are present with owner adminuser:www-data and Caddy's user can read the page. Verify the transfer and test rsync by running the rsync command again, it should transfer almost nothing, this is proof that this deployment procedure is working and repeatable across new sites, and updating existing sites.

The site is not viewable yet in the browser, this will be setup in configuring the domain step below.

---

### Domain Routing Part 2: Write the Caddyfile

The Caddyfile tells Caddy which domain maps to which directory. So when Caddy sees a site block with a real domain name it automatically requests a Let's Encrypt certificate, serves the site over HTTPS, and redirects HTTP to HTTPS.

Note: Let's Encrypt limits failed validation attempts to a handful per hour per domain, so if during setup it is not working, do not spam reloads while attempting to fix. Read the logs and troubleshoot the cause deliberately, then retry. The DNS is verified from the process above, and both firewalls are open and verified, so if the Caddyfile and setup is correct it should work the first try.

**Step A: Back up the default** (On VPS)

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.orig
```

This keeps the packaged default for reference and recovery.

**Step B: Write the Config**

```bash
sudo vim /etc/caddy/Caddyfile
```

Replace the entire contents with:

```caddy
example.com {
    root * /var/www/example.com
    file_server
    encode zstd gzip
    log
}

www.example.com {
    redir https://example.com{uri} permanent
}
```

Explained:

- example.com {...} is a site block. The address is what triggers automatic HTTPS.
- root * `<path>` sets the directory to serve for all request paths (*).
- file_server makes Caddy serve static files from that root.
- encode zstd gzip compresses responses to browsers that support it.
- log turns on access logging, by default it goes to the systemd journal. Access logs contain visitor IPs, so keep this private for documentation or other uses.
- the www block permanently redirects (HTTP 301) to the bare domain, keeping the path via {uri}. This gives one canonical address, which is better for SEO and avoids duplicate sites.

**Step C: Format, validate, reload**

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

fmt cleans-up the indentation. validate checks the config without applying it, and should end with Valid configuration. Do not reload if it reports an error, fix the error and validate again.

Start a live log view in a second SSH terminal:

```bash
sudo journalctl -u caddy -f
```

Then in the first terminal reload caddy:

```bash
sudo systemctl reload caddy
```

reload applies the new config cleanly without dropping connections. Watch for lines in the log terminal about obtaining a certificate for the domain. Success shows as certificate obtained successfully. Stop the live log view terminal when successful.

**Step D: Verify**

```bash
sudo ss -tlnp | grep caddy
sudo journalctl -u caddy --since "10 minutes ago" --no-pager | grep -iE "obtain|certificate|error|warn"
systemctl status caddy --no-pager | head -5
```

Expect that Caddy is now listening on *:443 as well as *:80. The log shows certificate obtained for both names with no errors. And the service is still active (running).

Quick external check, either run the command in the local terminal or visit your domain in a browser. If the above was correctly implemented there should be a live site.

```bash
curl -I https://example.com
```

Expect HTTP/2 200 with server: Caddy.

Quick Troubleshooting tips:

- Validation error: it will name the line in the config, this can often be an indentation or missing brace issue.
- Certificate errors in the log usually mean DNS does not point at the server or the firewall is blocking ports 80/443. These should have been verified above, but go back and check and verify everything is in order from above, with the DNS records and the firewall policies.

---

### Verify External Access and HTTPS

**Step A: Test with curl** (On local machine)

-s silences progress output and -I fetches headers only.

```bash
# 1. HTTP should redirect to HTTPS
curl -sI http://example.com

# 2. HTTPS should serve the site
curl -sI https://example.com

# 3. www should redirect to the bare domain, keeping the path
curl -sI "https://www.example.com/about?x=1"

# 4. Follow the whole chain from http://www (-L follows redirects)
curl -sIL http://www.example.com

# 5. A missing page should return 404
curl -sI https://example.com/does-not-exist
```

Expect:

1. redirect status (308 for Caddy's automatic redirect) with Location: https://example.com/
2. HTTP/2 200, server: Caddy and content-type: text/html
3. 301 with Location: https://example.com/about?x=1 the path and query string are preserved.
4. a chain of redirects ending at 200 on the bare HTTPS address.
5. 404

**Step B: A host-header check**

This test confirms Caddy serves the site only for the site's domain:

```bash
curl -skI https://<VPS-IP>
curl -sI http://<VPS-IP>
```

-k skips certificate checking since the certificate isn't valid for bare IP.

**Step C: Real browsers on real networks**

1. Open the site on a phone or other device on a separate network, mobile or something, testing the path from a different network.
2. Check for the certificate details.
3. In browser DevTools, open the Network tab and reload, every request should be 200 or 304 with no red entries and no "mixed content" warnings in the Console.
4. Try the www address and the http:// address and verify they redirect.

---

### Inspect Service Status and Logs

This process shows how to inspect what is happening on the server and its logs to check what happened in the past on the server. Also, this will test a reboot for continuity verification.

**Step A: Learn the three views constantly used** (On VPS)

```bash
systemctl status caddy --no-pager
sudo journalctl -u caddy -n 30 --no-pager -o cat
sudo journalctl -u caddy -p warning --since today --no-pager -o cat | cut -c1-250
```

- systemctl status shows whether it is running, when it started and the last few log lines.
- journalctl -u caddy -n 30 shows the last 30 lines of Caddy's log. -u selects the service and -o cat strips the prefix.
- -p warning filters to warnings and errors only. This is a fast way to find problems in noisy logs.

Next, look how systemd handles supervising Caddy:

```bash
systemctl cat caddy | grep -iE "^(user|group|restart|execstart|execreload)"
```

This shows the user it runs as, the command, and the Restart= policy which decides whether systemd revives the service after a crash.

*Note: after checking this I was missing the restart policy so I had to add it to my systemd config.*

**Step A.5: Config systemd restart policy for Caddy**

```bash
sudo mkdir -p /etc/systemd/system/caddy.service.d
sudo tee /etc/systemd/system/caddy.service.d/override.conf > /dev/null <<'EOF'
[Service]
Restart=on-failure
RestartSec=5s
EOF
sudo systemctl daemon-reload
```

The system is now ready to do the restart test and crash recovery test. Note, without the Restart config the crash test will not recover and run a restart of Caddy service.

**Step B: Restart test** (On VPS)

```bash
sudo systemctl restart caddy
systemctl is-active caddy
```

Then on local machine:

```bash
curl -sI https://example.com | head -30
```

This should show active and HTTP/2 200, after a restart Caddy should reuse the stored certificate and not request a new one. Check logs for this to verify this.

```bash
sudo journalctl -u caddy --since "2 minutes ago" --no-pager -o cat | grep -E '"logger":"tls' | cut -c1-250
```

**Step C: Crash recovery test**

Simulate Caddy being killed abruptly:

```bash
sudo kill -9 $(pidof caddy)
sleep 5
systemctl status caddy --no-pager | head -5
```

If the Restart= value was correct above, systemd should restart Caddy service. If it does not restart or something is wrong, check the results and troubleshoot the issue.

**Step D: Reboot test**

```bash
sudo reboot
```

wait a minute, then reconnect and run:

```bash
uptime -p
systemctl is-active caddy
sudo ufw status | head -1
```

and on local machine:

```bash
curl -sI https://example.com | head -3
```

Expect, a short uptime, active for Caddy, Status: active for ufw and HTTP/2 200. Nothing was started manually proving the site, the firewall and certificates all returned by themselves.

---

# Phase 3 FastAPI Backend Deployment

**Objectives:**

- Setup service account and directories. [X]
- Clone, Python environment, manual run. [X]
- Establish an isolated Python environment. [X]
- Configure the systemd service. [X]
- Set up site-key and run `create-site.py` script. [X]
- Configure Caddy reverse proxying. [X]
- Test application functionality, error handling and reboot testing. [X]

---

## Process Documentation Phase 3

### Setup Service Account and Directories

The backend runs as its own unprivileged user, so a problem in the app cannot reach the admin account.

- The app's code directory is read-only to that user.
- Only its data directory is writable.

**Step A: Create the service account** (On VPS)

```bash
sudo adduser --system --group --home /var/lib/lead-form-backend --no-create-home --shell /usr/sbin/nologin leadform
```

This creates a system user with no password and no login and a matching group. The --home flag names the data directory as its home without creating it because it gets created manually below.

**Step B: Create the directories**

```bash
sudo mkdir -p /opt/lead-form-backend /var/lib/lead-form-backend
sudo chown <adminuser>:<adminuser> /opt/lead-form-backend
sudo chmod 755 /opt/lead-form-backend
sudo chown leadform:leadform /var/lib/lead-form-backend
sudo chmod 750 /var/lib/lead-form-backend
```

- `/opt/lead-form-backend/` is where the app code and virtual environment will be. Adminuser owns it, so it can run updates without root. The service can read it but not write to it.
- `/var/lib/lead-form-backend` holds the database, logs, and backups, all writable data. Only the service can enter it, because this is where lead data will live.

**Step C: Verify**

```bash
getent passwd leadform
sudo -u leadform touch /var/lib/lead-form-backend/probe && echo "data directory: write OK"
sudo -u leadform touch /opt/lead-form-backend/probe
ls /var/lib/lead-form-backend
sudo rm /var/lib/lead-form-backend/probe
```

Expect:

- the shell shows `/usr/sbin/nologin`
- the directory show the owners and modes above.
- the data directory write succeeds.
- the code directory write fails with Permission denied.
- ls of the data directory fails with Permission denied.

If the code directory touch succeeds, stop and troubleshoot what went wrong, check the permissions and the command tests, this directory should not allow writes.

---

### Clone, Python Environment, Manual Run

This is to put the app on the server, build an isolated Python environment (a virtual environment for isolation of the app's packages from the system's Python), and then prove the app works before setting it up in systemd as a service.

**Step A: Tooling** (On VPS)

```bash
python3 --version
git --version
sudo apt install git python3-venv
```

Debian 13 should be on Python 3.13, python3-venv provides the virtual environment tooling, which Debian splits into its own package. This is checking if git is installed and if not installs and we check the version.

**Step B: Clone the repo**

```bash
git clone https://github.com/andrew-fraley/lead-form-backend.git /opt/lead-form-backend
cd /opt/lead-form-backend
git log --oneline -3
grep -n DATA_DIR app/config.py app/logging_config.py scripts/test_database_restore.py
```

The directory exists and is empty, which git clone accepts, this then copies only tracked files, so no .env, database, or venv. The grep confirms the server has your DATA_DIR support.

**Step C: Build the environment**

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/pip check
```

The first command creates `.venv` inside the code directory owned by adminuser. The install uses the pinned versions in `requirements.txt`, using Python 3.10 from the development environment, so this step may hit a snag. Then pip check should end with No broken requirements found.

**Step D: Manual run as the service user**

In the VPS terminal using dummy values run:

```bash
sudo -u leadform env DATA_DIR=/var/lib/lead-form-backend \
  RESEND_API_KEY=re_dummy EMAIL_FROM=test@example.com \
  CORS_ORIGINS=https://example.com \
  /opt/lead-form-backend/.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
```

The env command sets the variables for this process, `--host 127.0.0.1` binds it to the loopback address only, RESEND_API_KEY and EMAIL_FROM need to exist for the app to start, but nothing sends email yet.

Next, run in a second SSH session:

```bash
curl -s http://127.0.0.1:8000/
ls -la /var/lib/lead-form-backend /var/lib/lead-form-backend/logs
ls /opt/lead-form-backend
```

Expect the health check returns a JSON response, the data directory should now hold a database `lead-form-database.db` and `logs/` directory, all owned by leadform user. The app's code directory listing shows no .db file and no logs directory. This proves DATA_DIR works on the server.

Stop the app in the app running terminal before proceeding.

---

### Lead Form Systemd Service Setup

To move the service to run as a systemd service as opposed to the terminal manual running of the app. The goal is to set it up as a systemd service that starts on boot, restarts after crashes, runs as leadform user, and logs to the system journal.

**Step A: Clean up the VPS and confirm nothing is running** (On VPS)

```bash
rm -r /opt/lead-form-backend/logs
ls /opt/lead-form-backend
sudo ss -tlnp | grep 8000
```

The listing should show README.md app `requirements.txt` scripts, and ss line should print nothing. If something still listens on 8000, stop the manual run from above with Ctrl+C in the terminal first.

**Step B: The secrets file**

```bash
sudo mkdir -p /etc/lead-form-backend
sudo install -m 600 -o root -g root /dev/null /etc/lead-form-backend/env
sudo vim /etc/lead-form-backend/env
```

install creates an empty file that is already 600 and root owned, so it has no moment where it is readable by others. Put the env lines in with the necessary values and the server only Resend key.

```text
RESEND_API_KEY=<your server-only key>
EMAIL_FROM=leads@example.com
CORS_ORIGINS=https://example.com
```

Verify without exposing the secret.

```bash
sudo ls -l /etc/lead-form-backend/env
sudo cut -d= -f1 /etc/lead-form-backend/env
```

Expect to see -rw------- 1 root root, then only the three variable names. systemd reads this file as root and passes the values to the service, so leadform can't read this file.

**Step C: The unit file**

```bash
sudo tee /etc/systemd/system/lead-form-backend.service > /dev/null <<'EOF'
[Unit]
Description=Lead form backend (FastAPI)
After=network.target

[Service]
User=leadform
Group=leadform
WorkingDirectory=/opt/lead-form-backend
Environment=DATA_DIR=/var/lib/lead-form-backend
EnvironmentFile=/etc/lead-form-backend/env
ExecStart=/opt/lead-form-backend/.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF
sudo systemd-analyze verify /etc/systemd/system/lead-form-backend.service
```

- `Restart=on-failure` same as we set up with Caddy earlier to ensure crash restarts on Debian.
- systemd-analyze verify checks the syntax and should print nothing.

**Step D: Start and verify**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now lead-form-backend
systemctl status lead-form-backend --no-pager
sudo ss -tlnp | grep 8000
curl -sSi http://127.0.0.1:8000/ | head -1
ps -o user= -p $(systemctl show -p MainPID --value lead-form-backend)
sudo journalctl -u lead-form-backend -n 20 --no-pager -o cat | cut -c1-200
```

Expect:

- active (running) and enabled.
- a listener on `127.0.0.1:8000` only.
- HTTP/1.1 200 OK.
- leadform printed as the process owner.
- Uvicorn startup lines in the journal with no errors.

**Step E: Crash test**

```bash
sudo kill -9 $(systemctl show -p MainPID --value lead-form-backend)
sleep 8
systemctl status lead-form-backend --no-pager | head -5
systemctl show -p MainPID --value lead-form-backend
```

It should be active (running) after the test with a different PID than before. That gives evidence the restart policy works.

---

### Set Up Site-key and Run create-site.py Script

This will setup a new site in the database that will accept lead form submissions. First verify the site-key is configured in the Hugo site's config file `hugo.toml` and then run the script to create-site in the app, this prompts for setup info to populate the database with the site data.

**Step A: Verify the site key in the Hugo project** (On local machine)

Verify the site-key value exists and is appropriate for the website. Run the following in the project folder.

```bash
grep -n -i "key" hugo.toml
```

If it exists take note for when the script is run, if it does not it needs to be added and update the site project locally and rsync synchronization with the VPS.

**Step B: Run the create site script** (On VPS)

```bash
cd /opt/lead-form-backend
sudo -u leadform env DATA_DIR=/var/lib/lead-form-backend \
  RESEND_API_KEY=unused EMAIL_FROM=unused@example.com \
  .venv/bin/python -m scripts.create_site
```

The script prompts for the site key from Step A, a site name, the domain (example.com), and the notification email. Enter a real email for the site owner, where they wish to receive lead form submissions.

---

### Configure Caddy Reverse Proxying

The backend only listens on loopback so nothing outside can reach it, Caddy is the one public door, it terminates HTTPS on api.example.com and forwards the request to the `127.0.0.1:8000` loopback and port setup. First setup the DNS record in Cloudflare then setup Caddy with the Caddyfile.

**Step A: DNS Record** Cloudflare

in example.com > DNS > Records, add:

| Type | Name | Content | Proxy status |
|---|---|---|---|
| A | api | VPS IPv4 address | DNS only |

After adding the record verify from local machine:

```bash
dig +short api.example.com @1.1.1.1
```

It should return the VPS IPv4 Address. It may take some minutes so if it is non responsive wait and retry. Do not go forward in the Steps til this works.

**Step B: Caddy site block** (On VPS)

```bash
sudo vim /etc/caddy/Caddyfile
```

Leave the existing block alone and add this to the end:

```caddy
api.example.com {
    @allowed path / /leads
    handle @allowed {
        reverse_proxy 127.0.0.1:8000
    }
    handle {
        respond 404
    }
    log
}
```

- reverse_proxy `127.0.0.1:8000` forwards requests to the app.
- `@allowed path / /leads` limits what the public can reach to the health check (/) and the form endpoint (`/leads`), everything else gets plain 404. This keeps default FastAPI `/docs` and other endpoints from being reachable.
- handle blocks are mutually exclusive, a request takes the first one that matches.

**Step C: Validate, watch the logs, reload** (On VPS)

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

This should output Valid configuration. In a second SSH session, start a live log view. (Similar to the launch of the example.com site with Caddy above.)

```bash
sudo journalctl -u caddy -f -o cat | cut -c1-200
```

Watch the live view for a certificate being obtained for api.example.com, stop the log read when it is done.

**Step D: Verify** (On local machine)

```bash
curl -sSi https://api.example.com/ | head -12
curl -sSI https://api.example.com/docs | head -1
```

This show the health request logged by uvicorn as coming from `127.0.0.1`, which is Caddy talking to it, and a recent certificate lines from the API name without errors.

---

### Test Application Functionality, Error Handling and System Reboot

This is testing to prove full functionality as expected for the deployed Hugo site + lead form backend. This verifies if submissions are validated and processed, errors are handled, data persists, logs record what happened, and the app is unreachable except through Caddy.

**Test A: The four baseline requests** (On local machine)

Switch out placeholders for real site-key and website domains.

```bash
# 1. Valid: expect 200
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://api.example.com/leads \
  -H 'Content-Type: application/json' \
  -d '{"site_key":"<your-site-key>","email":"deploy-test@example.com","name":"Deployment Test","message":"server test"}'

# 2. Missing email: expect 422
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://api.example.com/leads \
  -H 'Content-Type: application/json' \
  -d '{"site_key":"<your-site-key>"}'

# 3. Unknown site key: expect 403
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://api.example.com/leads \
  -H 'Content-Type: application/json' \
  -d '{"site_key":"does-not-exist","email":"deploy-test@example.com"}'

# 4. Honeypot filled: expect 200, nothing saved
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://api.example.com/leads \
  -H 'Content-Type: application/json' \
  -d '{"site_key":"<your-site-key>","email":"bot@example.com","honeypot":"gotcha"}'
```

Expected output: 200, 422, 403, 200, and also check the notification was sent to the email.

**Test B: What was stored and persistence** (On VPS)

```bash
sudo -u leadform /opt/lead-form-backend/.venv/bin/python -c "import sqlite3; c=sqlite3.connect('/var/lib/lead-form-backend/lead-form-database.db'); print(c.execute('select * from lead_submissions order by rowid desc limit 5').fetchall())"
sudo systemctl restart lead-form-backend
sleep 5
sudo -u leadform /opt/lead-form-backend/.venv/bin/python -c "import sqlite3; c=sqlite3.connect('/var/lib/lead-form-backend/lead-form-database.db'); print(c.execute('select count(*) from lead_submissions').fetchall())"
```

Should see one row, the valid test, and an accepted status. The honeypot submission must not appear. After the restart, the row count should be unchanged proving persistence.

**Test C: Logs record what happened** (On VPS)

```bash
sudo journalctl -u lead-form-backend -n 15 --no-pager -o cat | cut -c1-160
sudo tail -n 8 /var/lib/lead-form-backend/logs/lead-form-backend.log | cut -c1-200
```

Both outputs should show the requests and outcomes, including the 200, 422, 403, and the honeypot event in the app's own log. Note, these logs will contain real IPs, so be mindful of exposure.

**Test D: The backend port is not public** (On local machine)

Put the real VPS IP in the command;

```bash
nc -vz -w 5 <VPS-IP> 8000
```

Expect this to time-out, same as the closed port tests in Phase 1 above. On the VPS, sudo ufw status | grep 8000 should print nothing.

**Test E: CORS behaves as intended** (On local machine)

This simulates the browsers pre-flight check:

```bash
curl -sSi -X OPTIONS https://api.example.com/leads \
  -H 'Origin: https://example.com' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type' | head -12

curl -sSi -X OPTIONS https://api.example.com/leads \
  -H 'Origin: https://evil.example' \
  -H 'Access-Control-Request-Method: POST' | head -12
```

The first should answer 200 with access-control-allow-origin: https://example.com. The second command should not include an access-control-allow-origin header.

Note: CORS is a browser rule so curl can still post from anywhere, it stops other websites from submitting through a visitor's browser and it does not replace authentication.

**Test F: Reboot test**

(On VPS)

Simulate a server reboot and verify lead form service starts on boot.

```bash
sudo reboot
```

Wait a minute, reconnect, and verify:

```bash
uptime -p
systemctl is-active caddy lead-form-backend
systemctl --failed --no-legend
sudo ufw status | head -1
```

(On local machine)

```bash
curl -sSi https://api.example.com/ | head -1
curl -sSI https://example.com | head -1
```

Expect a short uptime, active for both services, no failed units, ufw active, and 200 for both sites.

---

### Quick Secrets Check on Server

```bash
sudo ls -l /etc/lead-form-backend/env
git -C /opt/lead-form-backend status --short
```

The first needs to read -rw------- 1 root root. The second should print nothing, meaning the code folder has no local changes and no stray files.

---

# Phase 4 Maintenance and Recovery

**Objectives:**

- Establish an independent backup procedure. [ ]
- Perform a restore test. [ ]
- Document update procedures. [ ]
- Test service recovery. [ ]
- Review firewall rules, file permissions and service exposure. [ ]
- Prepare a maintenance checklist. [ ]

---

## Process Documentation Phase 4

### Establish an Independent Backup Procedure

Irreplaceable: the lead database and the server's config files.

Rebuildable: site files (Hugo projects), app code (Local repo or GH), Caddy certificates (reissued automatically).

**Step A: Back up the database on the server manually**

```bash
cd /opt/lead-form-backend
sudo -u leadform env DATA_DIR=/var/lib/lead-form-backend .venv/bin/python -m scripts.backup_database
sudo ls -l /var/lib/lead-form-backend/backups
```

Expect a timestamped `lead-form-database_<timestamp>.db`.

**Step B: Schedule it with a systemd timer**

```bash
sudo tee /etc/systemd/system/lead-form-backup.service > /dev/null <<'EOF'
[Unit]
Description=Lead form database backup

[Service]
Type=oneshot
User=leadform
Group=leadform
WorkingDirectory=/opt/lead-form-backend
Environment=DATA_DIR=/var/lib/lead-form-backend
ExecStart=/opt/lead-form-backend/.venv/bin/python -m scripts.backup_database
EOF
sudo tee /etc/systemd/system/lead-form-backup.timer > /dev/null <<'EOF'
[Unit]
Description=Daily lead form database backup

[Timer]
OnCalendar=*-*-* 03:30:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now lead-form-backup.timer
sudo systemctl start lead-form-backup.service
systemctl list-timers lead-form-backup.timer --no-pager
sudo journalctl -u lead-form-backup.service -n 5 --no-pager -o cat
```

Expect a next-run time and a new backup file. `Persistent=true` catches up a missed run after downtime. The script does not prune old backups. This will be adjusted as the project grows to need it.

**Step C: Pull an independent copy to local backup**

First step is to add the adminuser to be able to read the backups directory, then restart a new SSH session so the group applies:

```bash
sudo usermod -aG leadform <adminuser>
```

On local machine run:

```bash
mkdir -p ~/backups/vps/db ~/backups/vps/config && chmod 700 ~/backups/vps
rsync -av ionos-vps:/var/lib/lead-form-backend/backups/ ~/backups/vps/db/
rsync -av --relative ionos-vps:/etc/caddy/Caddyfile ionos-vps:/etc/systemd/system/lead-form-backend.service ionos-vps:/etc/systemd/system/lead-form-backup.service ionos-vps:/etc/systemd/system/lead-form-backup.timer ionos-vps:/etc/systemd/system/caddy.service.d/override.conf ionos-vps:/etc/ssh/sshd_config.d/00-hardening.conf ~/backups/vps/config/
sha256sum ~/backups/vps/db/*.db
ssh ionos-vps 'sha256sum /var/lib/lead-form-backend/backups/*.db'
```

Expect matching checksums.

---

### Perform a Restore Test

**Step A: Test-restore on the server with the script** (On VPS)

```bash
cd /opt/lead-form-backend
sudo -u leadform env DATA_DIR=/var/lib/lead-form-backend .venv/bin/python -m scripts.test_database_restore /var/lib/lead-form-backend/backups/<backup-file>
sudo -u leadform rm /var/lib/lead-form-backend/restore-test.db
```

Expect the script's integrity check to pass, it should refuse to run if `restore-test.db` already exists, so remove it afterward.

**Step B: Verify the off-server copy locally**

```bash
sqlite3 ~/backups/vps/db/<backup-file> "pragma integrity_check; select count(*) from lead_submissions;"
```

Expect ok and the row count you expect, this proves the independent copy is usable.

**Step C: Real restore drill on the live service**

```bash
sudo systemctl stop lead-form-backend
sudo -u leadform cp /var/lib/lead-form-backend/lead-form-database.db /var/lib/lead-form-backend/pre-restore.db
sudo -u leadform cp /var/lib/lead-form-backend/backups/<backup-file> /var/lib/lead-form-backend/lead-form-database.db
sudo systemctl start lead-form-backend
sleep 5
curl -sS https://api.example.com/
```

Check the row count with the earlier select, then delete `pre-restore.db`, if anything is wrong, copy `pre-restore.db` back.

---

### Document Update Procedures

**Step A: System updates** (On VPS)

```bash
sudo apt update
apt list --upgradable
sudo apt full-upgrade
uname -r
ls /boot/vmlinuz*
sudo tail -n 20 /var/log/unattended-upgrades/unattended-upgrades.log
systemctl list-timers 'apt-daily*' --no-pager
```

See whether a newer kernel was installed (compare uname -r to the files) and, if so, reboot afterward. This also gives evidence that unattended-upgrades has actually run.

---

### Test Service Recovery

**Step A: Crash test for both services**

```bash
sudo kill -9 $(systemctl show -p MainPID --value caddy)
sudo kill -9 $(systemctl show -p MainPID --value lead-form-backend)
sleep 8
systemctl is-active caddy lead-form-backend
```

Expect both active with new PIDs.

**Step B: Reboot test**

```bash
sudo reboot
```

Reconnect, then:

```bash
uptime -p
systemctl is-active caddy lead-form-backend
systemctl list-timers lead-form-backup.timer --no-pager
systemctl --failed --no-legend
sudo ufw status | head -1
```

From local machine, check curl -sSI https://example.com | head -1 and the API root.

---

### Review Firewall Rules, File Permissions and Service Exposure

**Step A: Network exposure**

```bash
sudo ss -tulnp
sudo ufw status verbose
```

Expected listeners: sshd on 22, Caddy on 80, 443/tcp and 443/udp, Caddy's admin on `127.0.0.1:2019`, uvicorn on `127.0.0.1:8000`, plus the system resolver on a loopback address.

Also on local machine scan with `nmap -Pn -p 1-10000 <VPS-IP>`, only 22, 80 and 443 should answer.

**Step B: Permissions**

```bash
ls -ld /opt/lead-form-backend /var/lib/lead-form-backend /etc/lead-form-backend /var/www/example.com
sudo ls -l /etc/lead-form-backend/env
sudo find /var/www -type f ! -perm 644
sudo find /var/lib/lead-form-backend ! -user leadform
sudo find / -xdev -type f -perm -0002 2>/dev/null | head
```

Compare to the above design setup: code owned by admin 755, data leadform 750, secrets root 600, site files 644. The last three commands should print nothing, or only explainable permission settings.

**Step C: SSH and update posture**

```bash
sudo sshd -T | grep -iE "permitrootlogin|passwordauthentication|kbdinteractive|pubkeyauthentication"
sudo journalctl -u ssh --since "7 days ago" --no-pager -o cat | grep -ciE "invalid user|failed password|authentication failure"
systemctl list-timers 'apt-daily*' --no-pager
```

Expect root and password logins off, the count shows how many login attempts bots made.

---
