# Debian VPS for Web Hosting 

## Project Overview:

### Purpose:

Setup a Debian 13 based Virtual Private Server used as a practical hosting platform for websites (personal and client), along with API backends to power certain functionality within the websites.

### Primary Goal:

Deploy, operate, and maintain Hugo websites using Caddy on a hosted Debian server, with secure remote administration and HTTPS. Additional goals are to expand the to support multiple websites and Python/FastAPI backend(s) on the same server.

### Guiding Principles:

- Prefer free and open-source software.  
- Keep architecture simple and understandable.  
- Documentation of procedures and verified results.  
- Add complexity only if a real requirement justifies it.  
- Maintain control over project files, configurations, and data.  
- Ensure security and availability are prioritized in the environment.  

### Compute Specifications:

- Platform: IONOS  
- OS: Debian 13  
- DNS: Cloudflare  
- Compute: 2vCPU, 4GB, 120 GB NVMe

# Project Plan

## Phase 1 Project Preparation, SSH Hardening, Firewalls, Security Updates

### Overview: 

This was the initial VPS and configuation of the IONOS virtual private server and the Debian host machine.  

This phase established a secure admin access through SSH key pairs between the client machine and the VPS Debian host. Also disallowed access for root user and any user using a normal SSH user/password login, this restricted strict SSH access between the client and server using only the keypair setup. 

Also, configured and verified firewall rules for the IONOS server layer firewall and the Debian host layer firewall. This adds a redundancy to the firewall for the setup in case of one firewall layer being compromised.  

Next, setup automatic security upgrades for the Debian host to ensure security updates and upgrades are checked and installed automatically thorugh the automated tool.  

All of the above is documented and verified through the process of this phase.  

### Technical tools, systems, and software used:

- IONOS as the hosting provider for the VPS.  
- Debian 13 as the operating system on the server host.  
- SSH for secure client access to the server.   
- IONOS Firewall for the server layer firewall.  
- ufw "uncomplicated firewall" for the Debian host layer firewall.
- unattended-upgrades for the automated security upgrades.

---

## Phase 2 First Hugo Site Deployed and Server Tests

### Overview

This phase deployed the first Hugo site live on the Caddy web server, which created a live website at the domain. First step was to configure the Cloudflare DNS records to point the domain to the IONOS VPS. 

Hugo process defined as such, hugo projects build locally using hugo --minify, then rsync transfers the file using our command to the desired IONOS VPS Debian host directory. The rsync utility was the tool of choice to allow for initial deployments of the public/ site files to the server as well as site updates using the same command that transfers only the changes updated in the sites public directory, rsync was picked because of its use to transfer complete files as well as optimal synchronization transfers, proving to be an effective choice for the project. 

Inital Caddy setup starts with creating and configuring the Caddyfile for the domain for the hugo site. When successfull configured Caddy serves the site live over HTTPS on the VPS.  The directory conventions were setup on the IONOS VPS Debian host using traditional Debain web directory conventions /var/www/example.com. Caddy automates Let's Encrypt certificate requests and renewals and the redirect the www. and HTTP-to-HTTPS.  

During systemd caddy service testing, I had to create a override.conf file containing the Restart rules for systemd for caddy service. This allows systemd to supervise the service and restart caddy on crashes, service restarts, and server reboots. The tests then verified that systemd properly restarts caddy services after crashes, restarts, and reboots which should achieve optimal uptime for the websites Caddy servers to the web. All tests verifed the desired results.  

Tests included systemd server restarts, mock caddy service crashes, and server reboots, all tests were accompanied by local host, external network, and interneral VPS checks for the service and Hugo website. 

### Technical tools, systems, and software used:  

- Cloudflare for domain registry.  
- Caddy for the webserver, picked for simple setup, config, and automated certificate process.  
- rsync as the file transfer utility, picked for simple, secure, and efficient file trasnfers and continuous synchronization.  
- systemd for caddy service handling, this was a familiar enough native Debian utility for handling and configuring the service.  
- journalctl as the native utility for querying and displaying relevant systemd service logs.  
- Let's Encrypt as the obvious choice for certificate procedures as Caddy automates and handles the process for the domain.  

