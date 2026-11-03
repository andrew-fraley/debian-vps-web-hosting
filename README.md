# Debian VPS for Web Hosting

## Project Overview

### Purpose

Setup a Debian 13 based Virtual Private Server used as a practical hosting platform for websites (personal and client), along with API backends to power certain functionality within the websites.

### Primary Goal

Deploy, operate, and maintain Hugo websites using Caddy on a hosted Debian server, with secure remote administration and HTTPS. Additional goals are to expand the setup to support multiple websites and Python/FastAPI backend(s) on the same server.

### Guiding Principles

- Prefer free and open-source software.
- Keep architecture simple and understandable.
- Documentation of procedures and verified results.
- Add complexity only if a real requirement justifies it.
- Maintain control over project files, configurations, and data.
- Ensure security and availability are prioritized in the environment.

### Compute Specifications

- Platform: IONOS
- OS: Debian 13
- DNS: Cloudflare
- Compute: 2vCPU, 4GB, 120 GB NVMe

## Project Plan

### Phase 1 Project Preparation, SSH Hardening, Firewalls, Security Updates

#### Overview

This was the initial VPS and configuration of the IONOS virtual private server and the Debian host machine.

This phase established a secure admin access through SSH key pairs between the client machine and the VPS Debian host. Also disallowed access for root user and any user using a normal SSH user/password login, this restricted strict SSH access between the client and server using only the keypair setup.

Also, configured and verified firewall rules for the IONOS server layer firewall and the Debian host layer firewall. This adds a redundancy to the firewall for the setup in case of one firewall layer being compromised.

Next, setup automatic security upgrades for the Debian host to ensure security updates and upgrades are checked and installed automatically through the automated tool.

All of the above is documented and verified through the process of this phase.

#### Technical Tools, Systems, and Software Used

- IONOS as the hosting provider for the VPS.
- Debian 13 as the operating system on the server host.
- SSH for secure client access to the server.
- IONOS Firewall for the server layer firewall.
- ufw "uncomplicated firewall" for the Debian host layer firewall.
- unattended-upgrades for the automated security upgrades.

---

### Phase 2 First Hugo Site Deployed and Server Tests

#### Overview

This phase deployed the first Hugo site live on the Caddy web server, which created a live website at the domain. First step was to configure the Cloudflare DNS records to point the domain to the IONOS VPS.

Hugo process defined as such, hugo projects build locally using hugo --minify, then rsync transfers the file using our command to the desired IONOS VPS Debian host directory. The rsync utility was the tool of choice to allow for initial deployments of the `public/` site files to the server as well as site updates using the same command that transfers only the changes updated in the sites public directory, rsync was picked because of its use to transfer complete files as well as optimal synchronization transfers, proving to be an effective choice for the project.

Initial Caddy setup starts with creating and configuring the Caddyfile for the domain for the hugo site. When successfully configured Caddy serves the site live over HTTPS on the VPS. The directory conventions were setup on the IONOS VPS Debian host using traditional Debian web directory conventions `/var/www/example.com`. Caddy automates Let's Encrypt certificate requests and renewals and the redirect the www. and HTTP-to-HTTPS.

During systemd caddy service testing, I had to create a `override.conf` file containing the Restart rules for systemd for caddy service. This allows systemd to supervise the service and restart caddy on crashes, service restarts, and server reboots. The tests then verified that systemd properly restarts caddy services after crashes, restarts, and reboots which should achieve optimal uptime for the websites Caddy servers to the web. All tests verified the desired results.

Tests included systemd server restarts, mock caddy service crashes, and server reboots, all tests were accompanied by local host, external network, and internal VPS checks for the service and Hugo website.

#### Technical Tools, Systems, and Software Used

- Cloudflare for domain registry.
- Caddy for the webserver, picked for simple setup, config, and automated certificate process.
- rsync as the file transfer utility, picked for simple, secure, and efficient file transfers and continuous synchronization.
- systemd for caddy service handling, this was a familiar enough native Debian utility for handling and configuring the service.
- journalctl as the native utility for querying and displaying relevant systemd service logs.
- Let's Encrypt as the obvious choice for certificate procedures as Caddy automates and handles the process for the domain.

---

### Phase 3 FastAPI Backend Deployment

#### Overview

This phase deployed the existing FastAPI lead-form backend to the IONOS VPS, making it accessible through Caddy at api.example.com and running it as a systemd service. The backend receives submissions from static Hugo websites, validates the data, stores leads persistently in SQLite, and sends email notifications to the relevant site owner through Resend.

The application was cloned from the public GitHub repository into `/opt` and installed in an isolated Python 3.13 virtual environment using the pinned `requirements.txt`. A dedicated, non-login leadform service account runs the application with read-only access to the code directory and write access to a separate data directory for the database, logs, and backups, while the environment secrets file remains accessible only to root. The backend's DATA_DIR setting enables this separation between application code and writable data, reducing unnecessary permissions and limiting the potential impact of a compromised service.

A systemd service runs the application as leadform, bound to `127.0.0.1:8000`, with systemd loading the protected environment variables and `Restart=on-failure` configured for recovery from application crashes. A Cloudflare DNS-only A record directs the API hostname to the VPS, while Caddy handles HTTPS through an automatically provisioned Let's Encrypt certificate and reverse-proxies requests to the application. The API uses a dedicated hostname and restricts accessible paths to / and `/leads`, returning 404 for other paths, while port 8000 remains inaccessible externally.

Testing verified expected request responses (200, 422, 403, and 200), persistent data across service restarts, request and error logging, Resend email delivery, external port isolation, and CORS acceptance of the authorized site origin and rejection of an unauthorized origin.

#### Technical Tools, Systems, and Software Used

- FastAPI, the framework for the existing backend.
- uvicorn, the server that runs the app.
- Python 3.13 and a virtual environment, from Debian's own packages and isolated from the system.
- SQLite, a file-based database in the data folder, with no database server to run.
- Resend, the email service for notifications.
- systemd, supervising the app with a unit file, an environment file and a restart policy.
- Caddy as a reverse proxy, the same server as Phase 2 forwarding HTTPS requests to the app.
- Cloudflare DNS, for the api record.
- Let's Encrypt, through Caddy, for the API hostname's certificate.
- Git and GitHub, deploying by git clone and updating by git pull, with the server holding a read-only copy.

---

### Phase 4 Maintenance and Recovery

#### Overview

This phase made the setup recoverable and reviewable. The goal was to prove that the data could be backed up and restored, that the services come back on their own, and that the server exposes only what it should.

The lead database and the server's configuration files are the only irreplaceable parts of the setup. Site files, application code, and Caddy's certificates can all be rebuilt from Git, the Hugo projects, or automatic reissue. The backup procedure therefore covers only the database and configs. A systemd timer runs the application's own backup script daily as the leadform service account, with `Persistent=true` so a run missed during downtime happens at the next boot. An independent copy of the backups and key config files is pulled over SSH with rsync to the local machine, and checksums are compared against the server to confirm the copies match. Secrets are deliberately excluded from automated backups and stay in a password manager.

The restore procedure was tested three ways: the application's restore-test script on the server, an integrity check and row count on the off-server copy, and a restore drill on the live service.

Update procedures were run and documented for the system (apt, kernel check, and a check of the unattended-upgrades log for proof that automatic updates actually run). Service recovery was tested with kill -9 crashes of Caddy and the backend and with a full server reboot, to confirm that systemd brings everything back with no manual steps.

The exposure review checked listening ports, firewall rules, file permissions, and SSH settings against the design. Listeners and firewall rules matched, with one finding: systemd-resolved was listening for LLMNR on port 5355 on all interfaces. Both firewall layers block it externally, but it is an unused service, so it was disabled. File permissions matched the design (code 755 admin-owned, data 750 leadform, secrets 600 root, site files 644), and the checks for unexpected ownership and world-writable files returned nothing. Root and password SSH logins are confirmed off. The journal showed roughly 29,500 failed login lines in 7 days from bots, none of which can succeed with key-only authentication.

The remaining weak spots are recorded in the risk list below with risk and priority levels, which set the order for later improvements.

#### Technical Tools, Systems, and Software Used

- systemd timers for scheduled backups, chosen over cron because they log to the journal and catch up after downtime.
- The backend's own backup and restore-test scripts, so the procedure tests the same tooling used in production.
- rsync over SSH for the independent copy, with sha256sum to verify it.
- sqlite3 for integrity checks and row counts on the copy.
- ss and ufw for checking listeners and firewall rules.
- nmap for an external scan of open ports.
- journalctl for backup runs, service logs, and SSH attempt counts.

---

### Risk List

(Risk description, Risk level: High Medium Low, and Priority level: High Medium Low. Priority level determines the order of improvement implementations.)

- Single server with no active monitoring or alerting.
    - Explained: Monitoring and alerting implementation planned as later improvement additions to the project.
    - Risk Level High
    - Priority: High
- No rate limiting on form.
    - Explained: Rate limits will be a future update on the lead form app.
    - Risk Level: Medium
    - Priority: High
- No maintenance routine or checklist developed.
    - Explained: Later improvement will implement this.
    - Risk Level: High
    - Priority: High
- No full rebuild drill from scratch.
    - Explained: Later improvement will implement this.
    - Risk Level: Medium
    - Priority: Low
- Dependency on third-party solutions: IONOS, Cloudflare, and Resend.
    - Explained: Service dependencies deferred until later solutions are needed, these solutions are trusted and reliable for the project scope.
    - Risk Level: Low
    - Priority: Low
- No automatic backup pruning.
    - Explained: Planned for later improvement implementation.
    - Risk Level: Low
    - Priority: Low

### Deferred

- Maintenance routine and checklist.
- Monitoring and alerting.
- Backup pruning.
- Rate limiting (backend repo).
- Multi-site hosting, to be added as its own phase when done live.

---

## Architecture

```text
Visitor browser
      |  HTTPS (443)
      v
Cloudflare DNS (DNS only) -> VPS public IP
      |
IONOS firewall -> ufw -> Caddy (automatic HTTPS)
      |
      +-- example.com      -> /var/www/example.com (static Hugo files)
      |
      +-- api.example.com  -> 127.0.0.1:8000 (uvicorn and FastAPI, runs as leadform)
                                 |
                                 +-- SQLite database (/var/lib/lead-form-backend)
                                 +-- Resend (email notification to the site owner)
```

Server layout:

| Location | Purpose | Owner and mode |
|---|---|---|
| `/opt/lead-form-backend/` | Application code and virtual environment | admin user, 755, read-only to the service |
| `/var/lib/lead-form-backend/` | Database, logs, and backups | leadform, 750 |
| `/etc/lead-form-backend/env` | Secrets file read by systemd | root, 600 |
| `/var/www/<domain>/` | Static site files, one directory per site | admin user and www-data, 755 and 644 |

## Repository Map

| Path | Contents |
|---|---|
| `README.md` | Project overview, phase summaries, and risk list |
| `docs/PROGRESS.md` | Step by step process documentation for each phase, with verification commands |
| `docs/DECISIONS.md` | Decisions and reasons for each phase |
| `docs/DEPLOYMENT.md` | Procedure for deploying a Hugo site to the server |
| `screenshots/` | Evidence of verified results |
| `.gitignore` | Excludes secrets and build output |

## Project Summary

This project showcases how to setup a website, webserver, backend app, and basic security configurations on a hosted IONOS Virtual Private Server. The website is a static Hugo site frontend with a FastAPI python lead form backend app, both are securely configured to be delivered to the public internet while not exposing them to unnecessary risks.

The webserver is built using Caddy which automatically requests HTTPS certificates, reroutes HTTP and www to HTTPS based on the simple configuration Caddyfile.

The backend app, as mentioned above, is a simple lead form app built using FastAPI in python, it takes lead submissions on the front end websites, sends it to the `leads/` endpoint on the app, which then processes, stores, and sends the email notification to the website owner's email. The email process is handled using Resend, a mail sender platform, the lead is stored in a local SQLite database, and the submission handling is processed by vanilla JavaScript on the frontend and then through the FastAPI app on the backend. The backend is setup on the VPS to separate duties and permissions as described above to keep secrets from being exposed, (systemd gets root to load the secrets directly into the service so the app does not process raw secrets), the leadform user setup to handle the app running in its code directory (as read-only) and the writable data being writable only by the leadform user (data directory, separate from code directory, for database, logs, and backups), and the directories being owned by the adminuser for routine updates and synchronization processes.

Beyond the above, the VPS was setup with security and thorough testing of all processes to maintain a secure environment for hosting websites and simple backend apps used by those websites, the project shows a single website setup but the infrastructure and tool decisions were picked for the end goal of hosting multiple websites on the single server.

## Related Repos

FastAPI Lead form backend app: https://github.com/andrew-fraley/lead-form-backend.git
